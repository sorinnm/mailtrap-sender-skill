#  mailtrap-api-sender
test adrian
Send personalized email through [Mailtrap](https://mailtrap.io) — sandbox
(testing) or live delivery — driven entirely by `curl`. No build step, no
dependencies beyond `curl`.

## How it works

- A **skill** (`.claude/skills/mailtrap-send`) and an **agent**
  (`.claude/agents/mailtrap-mailer`) tell Claude Code how to send.
- Claude reads a JSON file of recipients, substitutes each recipient's
  `{{variables}}` into the subject/text/html template, writes one payload per
  recipient to a temp file, and POSTs it with `curl`.

## Setup

1. `cp .env.example .env` and fill in your Mailtrap token + inbox id.
2. `cp templates/recipients.example.json templates/recipients.json` and edit
   the template and recipient list.

## Sending

Ask Claude Code, e.g.:

- "Send the campaign in `templates/recipients.json`" → runs in **sandbox**.
- "Dry run the first recipient" → renders the payload without sending.
- "Send it live" → switches to live delivery (Claude confirms first).

Or invoke directly: `/mailtrap-send` (skill) or the `mailtrap-mailer` subagent.

## Modes

| `MAILTRAP_MODE` | Endpoint | Effect |
|---|---|---|
| `sandbox` (default) | `sandbox.api.mailtrap.io/api/send/$MAILTRAP_INBOX_ID` | Lands in your Mailtrap test inbox |
| `live` | `send.api.mailtrap.io/api/send` | Real delivery (needs a verified domain) |

## Recipients JSON

```json
{
  "from": { "email": "hello@yourdomain.com", "name": "Your Name" },
  "subject": "{{first_name}}, welcome aboard!",
  "text": "Hi {{first_name}},\n\nThanks for joining {{company}}.",
  "html": "<p>Hi {{first_name}},</p><p>Thanks for joining {{company}}.</p>",
  "category": "personalized-campaign",
  "recipients": [
    { "email": "alice@example.com", "first_name": "Alice", "company": "Acme" }
  ]
}
```

`email` is required per recipient; every other key is a merge variable. `from`
falls back to `MAILTRAP_FROM_*` env vars. `category` (or the `MAILTRAP_CATEGORY`
env var) is sent as the `X-MT-Category` header to tag the mail's category in
Mailtrap. Secrets and your real `recipients.json` are gitignored.
