# Postal mail printed, mailed, and tracked - mailbox.bot

Mail a real letter from Cursor. Your agent prices it, previews it and submits it; mailbox.bot staff print, envelope and mail it by USPS or FedEx to US addresses.

Official plugin from [mailbox.bot](https://mailbox.bot).

## What you can ask

- "What does a 3-page certified letter to 90266 cost?"
- "Mail this notice to Jane Doe, 123 Main St, Manhattan Beach, CA 90266 by certified mail."
- "Print and mail `invoice.pdf` to the address on its first page."
- "Where is the letter I sent yesterday?"
- "Add certified-mail sending to our billing service, with a Sandbox test."

## Install

**From cursor.directory:** open the plugin page and choose **Add to Cursor** on the MCP server. Add the rule the same way.

**The whole plugin, from this repository:**

```bash
git clone https://github.com/arbengine/mailbox-bot-print-and-mail ~/.cursor/plugins/local/mailbox-bot-print-and-mail
```

Restart Cursor or run **Developer: Reload Window**, then look under **Customize**.

**The MCP server alone:** add this to `~/.cursor/mcp.json`.

```json
{
  "mcpServers": {
    "mailbox-bot": {
      "url": "https://mailbox.bot/api/mcp"
    }
  }
}
```

## Connect

Price checks work with no account. For everything else, Cursor asks you to sign in to mailbox.bot the first time a tool needs your account. On that page you choose the agent and **Sandbox** or **Live**.

- **Sandbox** prints, mails and charges nothing. Use it to see the whole flow.
- **Live** mails for real from prepaid credits. Each letter waits for your approval in the mailbox.bot dashboard.

No account yet? [Create one](https://mailbox.bot/signup). Signing up takes a US mobile number.

## Who does what

| Step | Your agent | You |
|---|---|---|
| Price a letter | Quotes it | |
| Preview | Runs a dry run and shows the address, mail class and exact cost | Say yes or no |
| Send | Submits it | Buy credits; approve the letter in the dashboard |
| Print and mail | | Done by mailbox.bot staff after your approval |
| Track | Reports status, tracking and photos | |
| Disconnect | | Revoke the connected app on the API Keys page |

First-Class letters have no carrier tracking; mailbox.bot records photos instead. Certified mail has USPS tracking and proof of delivery. Letters go to US addresses only.

## Mailing a file from your workspace

Sign-in covers letters written as text. To mail a PDF or DOCX from disk, the agent uploads it through the REST API, which reads an agent key from your environment:

```bash
export MAILBOX_BOT_API_KEY="sk_agent_test_..."   # a Sandbox key from mailbox.bot/dashboard/api-keys
```

The agent runs a dry run first and sends with approval required, so you check the stored document in the dashboard before anything is charged.

## What is inside

| File | What it does |
|---|---|
| `mcp.json` | Connects Cursor to `https://mailbox.bot/api/mcp` by sign-in |
| `rules/mailbox-bot-mail.mdc` | The order to work in: price, read the account's rules, preview, send, track |
| `skills/print-and-mail` | Mailing a letter or a file, step by step |
| `skills/print-mail-api` | Building mail sending into your own app |

No scripts, hooks or install steps. No prices either: the agent reads live prices from mailbox.bot.

## Links

- Prices: https://mailbox.bot/pricing
- Outbound mail: https://mailbox.bot/outbound-mail-api
- API docs: https://mailbox.bot/api-docs
- Support: support@mailbox.bot

MIT License.
