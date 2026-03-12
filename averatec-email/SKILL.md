---
name: averatec-email
description: Send, reply, draft, and read Gmail via gog CLI. Use this for all email operations. Always handles multi-line formatting correctly — never use --body inline for multi-line content.
---

# Email Actions (averatec)

Uses `gog` CLI with account `ayetek0773@gmail.com` (auto-configured via `GOG_ACCOUNT` env var).

## Send an email

ALWAYS use Python to write the body file — NEVER use `--body "...\n..."` or shell variables with `\n`. The `\n` in shell strings renders as literal characters in the sent email.

```bash
python3 -c "
body = '''Hi [Name],

[Body paragraph]

Best,
Scott'''
open('/tmp/email_body.txt', 'w').write(body)
"

gog gmail send \
  --to "recipient@example.com" \
  --subject "Subject here" \
  --body-file /tmp/email_body.txt

rm -f /tmp/email_body.txt
```

## Send HTML email

```bash
gog gmail send \
  --to "recipient@example.com" \
  --subject "Subject here" \
  --body-html "<p>Hi,</p><p>Body here.</p><p>Best,<br>Scott</p>"
```

## Reply to an email

```bash
# First find the message ID
gog gmail search "subject:\"Re: Topic\" from:sender@example.com" --max 5

# Then reply using Python for the body
python3 -c "
body = '''Hi [Name],

[Reply content]

Best,
Scott'''
open('/tmp/email_body.txt', 'w').write(body)
"

gog gmail send \
  --to "sender@example.com" \
  --subject "Re: Original Subject" \
  --body-file /tmp/email_body.txt \
  --reply-to-message-id <msgId>

rm -f /tmp/email_body.txt
```

## Save as draft (review before sending)

```bash
python3 -c "
body = '''Hi [Name],

[Body content]

Best,
Scott'''
open('/tmp/email_body.txt', 'w').write(body)
"

gog gmail drafts create \
  --to "recipient@example.com" \
  --subject "Subject" \
  --body-file /tmp/email_body.txt

rm -f /tmp/email_body.txt
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
- Signature: always end emails with `Best,\nScott` unless user specifies otherwise
- Multi-line body: ALWAYS use Python triple-quoted string → write to `/tmp/email_body.txt` → `--body-file`
- Do NOT use `--body "line1\nline2"` — `\n` will appear as literal characters in the sent email
- Do NOT use shell variables or heredoc for body content — only Python triple-quoted strings are reliable
- Always clean up temp files after sending: `rm -f /tmp/email_body.txt`
