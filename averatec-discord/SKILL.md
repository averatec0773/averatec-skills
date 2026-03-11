---
name: averatec-discord
description: Send Discord messages, DMs, and notifications using the bot token. Use this to proactively message the owner or post to channels via the Discord API.
---

# Discord Actions (averatec)

Use `bash` with `curl` to call the Discord REST API. The bot token is available as `$DISCORD_BOT_TOKEN`.

## Send a DM to the owner

Always use this two-step process:

```bash
# Step 1: Create or retrieve DM channel
CHANNEL=$(curl -s -X POST https://discord.com/api/v10/users/@me/channels \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"recipient_id": "360785034438901761"}' | python3 -c "import sys,json; print(json.load(sys.stdin)['id'])")

# Step 2: Send message
curl -s -X POST "https://discord.com/api/v10/channels/$CHANNEL/messages" \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"content\": \"YOUR MESSAGE HERE\"}"
```

Owner Discord user ID: `360785034438901761`
Owner timezone: America/New_York (Boston, EST UTC-5 / EDT UTC-4)
When showing time to owner, always convert: `TZ='America/New_York' date '+%H:%M %Z'`

## Send a message to a channel

```bash
curl -s -X POST "https://discord.com/api/v10/channels/CHANNEL_ID/messages" \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content": "YOUR MESSAGE HERE"}'
```

## Send a message with an embed

```bash
curl -s -X POST "https://discord.com/api/v10/channels/CHANNEL_ID/messages" \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "embeds": [{
      "title": "Title",
      "description": "Body text",
      "color": 5814783
    }]
  }'
```

## Reply to a message

```bash
curl -s -X POST "https://discord.com/api/v10/channels/CHANNEL_ID/messages" \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "content": "Reply content",
    "message_reference": {"message_id": "MESSAGE_ID"}
  }'
```

## Add a reaction

```bash
curl -s -X PUT \
  "https://discord.com/api/v10/channels/CHANNEL_ID/messages/MESSAGE_ID/reactions/✅/@me" \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN"
```

## Get recent messages from a channel

```bash
curl -s "https://discord.com/api/v10/channels/CHANNEL_ID/messages?limit=10" \
  -H "Authorization: Bot $DISCORD_BOT_TOKEN"
```

## Notes

- `$DISCORD_BOT_TOKEN` is always available — do NOT ask the user for it
- DM requires creating the channel first (Step 1 above) — the channel ID changes per user
- Guild ID for AVERATEC-WORKPLACE: `1480722310704009346`
- Always check the response for errors (look for `"code"` field in JSON response)
