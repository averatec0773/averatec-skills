---
name: averatec-email
description: Send, reply, draft, and read Gmail via gog CLI. Use this for all email operations. Always use HTML format for multi-paragraph emails to avoid line-wrapping issues.
---

# Email Actions (averatec)

Uses `gog` CLI with account `ayetek0773@gmail.com` (auto-configured via `GOG_ACCOUNT` env var).

## Send an email (standard)

ALWAYS use `--body-html` for multi-paragraph emails. Plain text `--body-file` causes line-wrap issues because the email client preserves every newline in the file verbatim.

```bash
gog gmail send \
  --to "recipient@example.com" \
  --subject "Subject here" \
  --body-html "<p>Hi [Name],</p><p>[Body paragraph]</p><p>Best,<br>Scott</p>"
```

For longer bodies, build the HTML string in Python to avoid quoting issues:

```bash
python3 -c "
html = '<p>Hi [Name],</p><p>[Paragraph one. Write the full paragraph here without line breaks — the email client handles word wrap.]</p><p>[Paragraph two.]</p><p>Best,<br>Scott</p>'
import subprocess
subprocess.run(['gog', 'gmail', 'send', '--to', 'recipient@example.com', '--subject', 'Subject here', '--body-html', html])
"
```

## Reply to an email

```bash
# First find the message ID
gog gmail search "subject:\"Topic\" from:sender@example.com" --max 5

# Then reply
python3 -c "
html = '<p>Hi [Name],</p><p>[Reply content.]</p><p>Best,<br>Scott</p>'
import subprocess
subprocess.run(['gog', 'gmail', 'send', '--to', 'sender@example.com', '--subject', 'Re: Original Subject', '--body-html', html, '--reply-to-message-id', 'MSG_ID_HERE'])
"
```

## Save as draft

```bash
python3 -c "
html = '<p>Hi [Name],</p><p>[Body.]</p><p>Best,<br>Scott</p>'
import subprocess
subprocess.run(['gog', 'gmail', 'drafts', 'create', '--to', 'recipient@example.com', '--subject', 'Subject', '--body-html', html])
"
```

## Read / search inbox

```bash
# Recent unread
gog gmail search "is:unread newer_than:2d" --max 10

# From specific sender
gog gmail messages search "from:someone@example.com" --max 5
```

## Notes

- Default account: `ayetek0773@gmail.com` — no `--account` flag needed
- Signature: always end with `<br>Scott` inside the last `<p>` tag
- ALWAYS use `--body-html` — plain text file preserves every line break verbatim, causing broken wrapping
- Do NOT use `--body "...\n..."` — `\n` renders as literal characters
- Do NOT add line breaks inside paragraph text — write each paragraph as one continuous string, let the email client wrap it
- Use Python `subprocess.run` for long bodies to avoid shell quoting issues with special characters
