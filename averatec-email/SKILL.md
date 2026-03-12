---
name: averatec-email
description: Send, reply, draft, and read Gmail via gog CLI. Use this for all email operations. Always handles multi-line formatting correctly — never use --body inline for multi-line content.
---

# Email Actions (averatec)

Uses `gog` CLI with account `ayetek0773@gmail.com` (auto-configured via `GOG_ACCOUNT` env var).

## Send an email

Always write the body to a temp file or use stdin — NEVER pass multi-line content via `--body` directly (backslash-n will not render as newlines).

```bash
# Write body to temp file, then send
cat > /tmp/email_body.txt << 'BODY'
Hi [Name],

[Body paragraph]

Best,
Scott
BODY

gog gmail send \
  --to "recipient@example.com" \
  --subject "Subject here" \
  --body-file /tmp/email_body.txt
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

# Then reply
cat > /tmp/reply_body.txt << 'BODY'
Hi [Name],

[Reply content]

Best,
Scott
BODY

gog gmail send \
  --to "sender@example.com" \
  --subject "Re: Original Subject" \
  --body-file /tmp/reply_body.txt \
  --reply-to-message-id <msgId>
```

## Save as draft (review before sending)

```bash
cat > /tmp/draft_body.txt << 'BODY'
[Body content]
BODY

gog gmail drafts create \
  --to "recipient@example.com" \
  --subject "Subject" \
  --body-file /tmp/draft_body.txt
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
- Multi-line body: use `cat > /tmp/email_body.txt << 'BODY' ... BODY` — heredoc correctly handles newlines
- Do NOT use `--body "line1\nline2"` — `\n` will appear as literal characters in the sent email
- Always clean up temp files after sending: `rm -f /tmp/email_body.txt`
