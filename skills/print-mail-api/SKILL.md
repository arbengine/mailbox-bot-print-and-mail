---
name: print-mail-api
description: Add postal mail sending to an app with the mailbox.bot REST API. Use when writing or reviewing code that sends letters, certified mail or batch mail, handles mailbox.bot webhooks, or tests mail flows with a Sandbox key.
---

# Build postal mail into an app with the mailbox.bot API

One request with a document and a US address becomes a printed, mailed letter with status events and proof. Base URL: `https://mailbox.bot/api/v1`.

## Read the live contract first

Fetch the current spec before writing code. Do not write field names from memory.

- OpenAPI: https://mailbox.bot/openapi.json
- Full reference for agents: https://mailbox.bot/llms-full.txt
- Docs for people: https://mailbox.bot/api-docs

## Rules for the code you write

1. **Key from the environment.** Read `MAILBOX_BOT_API_KEY` from the environment or the project's secret store. Never hardcode it, log it or put it in a URL.
2. **Sandbox in development and tests.** An `sk_agent_test_` key runs the same endpoints with full validation and real price previews, and nothing is printed, mailed or charged. Use an `sk_agent_` key in production only. Account keys (`sk_live_`) cannot send mail.
3. **Version header on every send.** `GET /agents/me/instructions` returns `version` (an agent key resolves its own agent). Send it as `X-Mailbox-MD-Version`. On 409 `MAILBOX_MD_VERSION_MISMATCH`, fetch it again and retry once.
4. **Price before sending.** `dry_run=true` on `POST /mail` validates the request and returns `cost_cents`, `cost_breakdown` and `human_review`. Nothing is stored or charged.
5. **Cap or hold live sends.** Send `X-Max-Cost-Cents` to refuse a piece above a cap, or `requires_approval=true` to hold it until the account owner approves it in the dashboard.
6. **Safe retries.** Give each request its own `Idempotency-Key` (the dry run and the real send get different keys) and reuse a key only to retry that exact request. A 409 `DUPLICATE_OUTBOUND_MAIL` means the piece already exists: read the existing record and do not resend.
7. **Money errors go to a person.** 402 `INSUFFICIENT_CREDITS` carries `billing_url`, the balance and the shortfall. Code cannot buy credits; show the link to the account owner.
8. **Trust the record, not the event.** Verify each webhook's signature, then read `GET /mail/{id}` before acting on it.

## Send a letter

```ts
const API = "https://mailbox.bot/api/v1";
const auth = { Authorization: `Bearer ${process.env.MAILBOX_BOT_API_KEY}` };

type Recipient = {
  recipient_name: string;
  recipient_line1: string;
  recipient_city: string;
  recipient_state: string; // 2 letters
  recipient_zip: string;
};

export async function mailLetter(
  pdf: Blob, // from disk in Node: await fs.openAsBlob("letter.pdf", { type: "application/pdf" })
  to: Recipient,
  opts: { mailClass: string; idempotencyKey: string; maxCostCents: number; dryRun?: boolean },
) {
  const { version } = await fetch(`${API}/agents/me/instructions`, { headers: auth }).then((r) => r.json());

  const form = new FormData();
  form.append("document", pdf, "letter.pdf");
  for (const [field, value] of Object.entries(to)) form.append(field, value);
  form.append("mail_class", opts.mailClass);
  if (opts.dryRun) form.append("dry_run", "true");

  const res = await fetch(`${API}/mail`, {
    method: "POST",
    headers: {
      ...auth,
      "X-Mailbox-MD-Version": String(version),
      "X-Max-Cost-Cents": String(opts.maxCostCents),
      "Idempotency-Key": opts.idempotencyKey,
    },
    body: form,
  });
  const body = await res.json();
  if (!res.ok) throw new Error(`mailbox.bot ${res.status}: ${body.code ?? body.error}`);
  return body; // dry run: cost_cents, human_review. Send: outbound_mail.id, outbound_mail.status
}
```

Documents can be PDF, DOCX, JPG, PNG, TXT or CSV. A PDF prints as it looks. Optional `return_*` fields override the account's saved return address.

## Follow a letter

Status moves `pending_approval` → `submitted` → `ready` → `mailed` → `delivered`, with `failed` and `cancelled` as end states. `GET /mail/{id}` returns status, tracking and fulfillment photos. `first_class` has no carrier tracking, so `tracking_number` can stay null. `DELETE /mail/{id}` cancels while the piece is still `submitted`.

## Webhooks

Set the URL with `PUT /webhooks/settings`. Events: `mail.pending_approval`, `mail.submitted`, `mail.ready`, `mail.mailed`, `mail.delivered`, `mail.failed`, `mail.cancelled`. The event name is in `event_type`, and `metadata` you sent with the letter comes back unchanged.

Each delivery is signed in `X-Mailbox-Signature` as `<key prefix>:t=<timestamp>,v1=<hex>`, an HMAC-SHA256 of `<timestamp>.<raw body>` keyed with the signing secret:

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

export function isMailboxWebhook(rawBody: string, header: string, secret: string): boolean {
  const timestamp = /t=(\d+)/.exec(header)?.[1];
  const sent = /v1=([0-9a-f]{64})/.exec(header)?.[1];
  if (!timestamp || !sent) return false;
  const expected = createHmac("sha256", secret).update(`${timestamp}.${rawBody}`).digest("hex");
  return timingSafeEqual(Buffer.from(sent, "hex"), Buffer.from(expected, "hex"));
}
```

Verify against the raw request body, before any JSON parsing. Keep the secret in the same store as the API key.

## Test without mailing

- A Sandbox key on `POST /mail` gives the real flow with `test_mode: true` and `cost_cents: 0`.
- `POST /test/mail` creates a test piece with no document; `POST /test/mail/{id}/advance` moves it one status forward and fires the matching webhook.
- `POST /test/webhook` sends a sample signed event to the configured URL.

## One document, many recipients

Batch mail takes one template and a recipient list: `POST /batch-mail/estimate` for the price, `POST /batch-mail` for a draft (nothing charged), `POST /batch-mail/{id}/confirm` to pay and queue it. Guide: https://mailbox.bot/api-docs/batch-and-postcards

## Who does what

The code prices, submits, tracks and cancels. A person creates the account and the key, buys credits and approves held mail. mailbox.bot staff print and mail each piece. No API call adds credits, so do not write code that tries to.
