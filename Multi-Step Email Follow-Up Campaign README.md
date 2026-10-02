# Multi-Step Email Follow-Up Campaigns in n8n (Gmail + Google Sheets)

Send a sequence of emails to a list of contacts and let n8n do the chasing. The first email goes to everyone. Follow-ups are sent **in the same thread** only if nobody has replied yet, so you only deal with the people who respond.

## What it does

- Reads contacts from a Google Sheet
- Sends email #1 to every row where `first_emailed` is blank, then stamps today's date in that column
- Every hour, scans your Gmail threads for this campaign and sends the next email in the sequence when it is due
- Stops automatically for any thread where a human has replied
- Personalises every message with `{placeholders}` taken from your sheet columns
- Skips weekends

## How it works

Every email carries a hidden tag at the start of its HTML body:

```html
<span data-cam='CAMPAIGN_ID' data-seq='0' data-ph='{"name":"Sam","company":"Acme"}'></span>
```

| Attribute | Purpose |
|-----------|---------|
| `data-cam` | Campaign ID (`mail_id` in Settings), so threads can be recognised as part of this campaign |
| `data-seq` | Position in the sequence (0, 1, 2...) |
| `data-ph` | The placeholder values used, so later emails can reuse them without re-reading the sheet |

On each run the workflow fetches recent threads, decodes every message, and checks that **every** message in the thread was sent by you, belongs to this campaign, and has the expected sequence number. A reply from anyone else breaks that chain, so the thread is left alone. If the chain is intact and enough days have passed, the next template is sent as a reply in that thread.

### Flow

```
Every hour -> Skip weekends -> Settings -> Email sequence
                                              |
                  +---------------------------+---------------------------+
                  v                                                       v
        Branch 1: first emails                                Branch 2: follow-ups
        Get emails (Sheet)                                    Get previous message threads (Gmail)
        To email? (first_emailed empty)                       Get thread details
        Prepare first message params                          Decode messages
        Package placeholder values                            Classify threads
        Send via sub-workflow                                 Next message due?
        Update first_emailed (Sheet)                          Prepare reply params
                                                              Decode placeholder values
                                                              Send via sub-workflow

Sub-workflow (same workflow, via Execute Workflow Trigger):
Fill message placeholders -> Compose message (adds hidden tag) -> Replying?
   -> yes: Reply to message      -> no: Send new message
```

## Requirements

- n8n (cloud or self-hosted)
- A Gmail account (Google Workspace recommended for higher sending limits)
- A Google Sheet containing your contacts
- Credentials in n8n for **Google Sheets OAuth2** and **Gmail OAuth2**

## Setup

1. **Import** `email-campaign-workflow.json` into n8n (Workflows -> Import from file).
2. **Create your sheet** with these columns:
   - `email`
   - `first_emailed` (leave blank, filled automatically)
   - One column for every `{placeholder}` used in your templates (for example `name`, `company`)
3. **Edit the `Settings` node:**

   | Field | Description |
   |-------|-------------|
   | `sheet_url` | URL of your Google Sheet |
   | `subject` | Email subject line (also used to find threads) |
   | `sender_name` | Your display name. Must match the name on your Gmail `From` header |
   | `email_column_name` | Sheet column that holds the address (default `email`) |
   | `mail_id` | A unique ID for this campaign |

4. **Edit the `Email sequence` node** with your messages (HTML) and `send_on_day`, the number of days after the first email that each message should go out.
5. **Attach credentials** to the two Google Sheets nodes (`Get emails`, `Update last contacted time`) and the three Gmail nodes (`Get previous message threads`, `Get thread details`, plus the send nodes `Reply to message` and `Send new message`).
6. **Test first.** Put only your own address in the sheet, run the workflow manually, and check that the email arrives and the sheet updates.
7. **Activate** the workflow.

### Example sequence

```json
{
  "emails": [
    { "message": "Hi {name},<br /><br />Would {company} be open to a quick call?<br /><br />Regards,<br />Nathan", "send_on_day": 0 },
    { "message": "Hi {name},<br /><br />Just following up on this.<br /><br />Regards,<br />Nathan", "send_on_day": 3 },
    { "message": "Last try from me :)<br /><br />Nathan", "send_on_day": 8 }
  ]
}
```

Messages are HTML, so use `<br />` for line breaks.

## Running several campaigns

Duplicate the workflow for each campaign and change **both** the `subject` and the `mail_id`, and point it at its own sheet. Reusing a `mail_id` or subject across copies can make threads from different campaigns match each other.

## Known limitations

- **Multipart messages:** Some Gmail messages store their body in `payload.parts` instead of `payload.body.data`. The `Decode messages` node reads only the latter, so those threads are treated as non-campaign and follow-ups silently stop for them. Extending the decoder to walk `parts` is the main improvement to make.
- **Sheet is updated after sending.** If the update fails (quota, auth), the next run will email that contact again. Consider adding error handling or marking rows before sending.
- **Subject matching** relies on Gmail search. The workflow quotes the subject for an exact phrase match, but avoid very generic subjects.
- **Lookback window** is the last `send_on_day` + 1 days. Threads older than that are ignored.
- **Weekends:** Nothing sends on Saturday or Sunday; anything due then goes out the next weekday run.
- **Sender name matching:** Reply detection checks that `sender_name` appears in the `From` header. If your Gmail display name differs, no follow-ups will send.
- Gmail sending limits apply (about 500 per day for personal accounts, 2,000 for Workspace).

## Changes from the original export

- Credentials, instance ID, webhook IDs, and the sample sheet URL removed
- `Reply to message` now uses `sender_name` from Settings instead of a hardcoded name
- `Classify threads`: first-message time now comes from the current item rather than a node lookup, and the broken placeholder-restore block was removed (`Decode placeholder values` already handles it)
- Gmail subject search is now quoted for exact-phrase matching
- Added setup sticky notes and a placeholder `mail_id`

## Responsible use

This is for legitimate, relevant outreach. Follow anti-spam laws that apply to you (CAN-SPAM, GDPR, PECR, CASL and others), include a way to opt out, and honour opt-out requests. Do not use it to spam.