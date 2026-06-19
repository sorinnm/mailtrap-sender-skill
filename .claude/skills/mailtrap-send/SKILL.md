---
name: mailtrap-send
description: Send personalized emails through Mailtrap (sandbox or live) from a JSON recipients file using curl. Use when asked to send mail, run an email campaign, blast/notify a list, or test email sending via Mailtrap.
---

# Mailtrap personalized sender

Send one personalized email per recipient through the Mailtrap API using `curl`.
You (the model) do the templating: read the JSON data file, substitute
`{{variables}}` per recipient, write each finished payload to a temp file, and
POST it with `curl --data @file`. No `jq`, no extra scripts.

## 1. Resolve configuration (env vars)

Read these from the environment (set in `.env` — load it first if present):

| Variable | Required | Purpose |
|---|---|---|
| `MAILTRAP_API_TOKEN` | yes | API token (Bearer auth) |
| `MAILTRAP_MODE` | no (default `sandbox`) | `sandbox` = test inbox, `live` = real delivery |
| `MAILTRAP_INBOX_ID` | when `sandbox` | Sandbox inbox id |
| `MAILTRAP_FROM_EMAIL` | no | Default sender if JSON omits `from.email` |
| `MAILTRAP_FROM_NAME` | no | Default sender name |

Load the file if it exists, then read values:

```bash
set -a; [ -f .env ] && . ./.env; set +a
: "${MAILTRAP_MODE:=sandbox}"
```

Pick the endpoint by mode:

- `sandbox` → `https://sandbox.api.mailtrap.io/api/send/$MAILTRAP_INBOX_ID`
- `live`    → `https://send.api.mailtrap.io/api/send`

**If `MAILTRAP_API_TOKEN` is unset, stop and tell the user.** If `sandbox` and
`MAILTRAP_INBOX_ID` is unset, stop. If `live`, **confirm with the user before
sending** — live mail reaches real inboxes and cannot be recalled.

## 2. Read the recipients file

Default path: `templates/recipients.json` (see `templates/recipients.example.json`).
Shape:

```json
{
  "from": { "email": "hello@yourdomain.com", "name": "Your Name" },
  "subject": "{{first_name}}, welcome aboard!",
  "text": "Hi {{first_name}},\n\nThanks for joining {{company}}.\n\n— The Team",
  "html": "<p>Hi {{first_name}},</p><p>Thanks for joining <strong>{{company}}</strong>.</p>",
  "category": "personalized-campaign",
  "recipients": [
    { "email": "alice@example.com", "first_name": "Alice", "company": "Acme" }
  ]
}
```

`from` falls back to `MAILTRAP_FROM_EMAIL` / `MAILTRAP_FROM_NAME`. `category` is
optional. Each recipient must have `email`; any other keys are merge variables.

## 3. Build + send one payload per recipient

For each recipient:

1. Substitute every `{{key}}` in `subject`, `text`, and `html` with that
   recipient's value. If a `{{key}}` has no value, leave it blank and warn.
2. Build the per-recipient JSON payload and **write it to a temp file** (use the
   Write tool) — this avoids all shell-quoting problems with apostrophes,
   newlines, and HTML:

   ```json
   {
     "from": { "email": "hello@yourdomain.com", "name": "Your Name" },
     "to": [ { "email": "alice@example.com", "name": "Alice" } ],
     "subject": "Alice, welcome aboard!",
     "text": "Hi Alice,\n\nThanks for joining Acme.\n\n— The Team",
     "html": "<p>Hi Alice,</p><p>Thanks for joining <strong>Acme</strong>.</p>",
     "category": "personalized-campaign"
   }
   ```

3. Send it:

   ```bash
   curl -sS -w '\n%{http_code}' -X POST "$ENDPOINT" \
     -H "Authorization: Bearer $MAILTRAP_API_TOKEN" \
     -H "Content-Type: application/json" \
     --data @/tmp/mailtrap_payload.json
   ```

   A `2xx` with `{"success":true,...}` is a send. On non-2xx, show the body and
   continue to the next recipient (don't abort the whole batch).

## 4. Report

Summarize: total recipients, succeeded, failed (with the API error for each),
and the mode/endpoint used. In `sandbox`, remind the user the mail landed in the
Mailtrap test inbox, not real inboxes.

## Safety rules

- Default to `sandbox`. Only send `live` after explicit user confirmation.
- Never print `MAILTRAP_API_TOKEN`.
- For large lists (>50), confirm the count with the user before blasting.
