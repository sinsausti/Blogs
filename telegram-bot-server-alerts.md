---
title: "Send Server Alerts to Telegram with a Bot"
description: "Create a Telegram bot and send practical server and monitoring alerts with curl and a reusable Bash script."
author: "Sebastian Insausti"
date: "2026-09-02"
tags: ["Infrastructure", "Monitoring"]
canonical_url: "https://insaustis.com/blog/telegram-bot-server-alerts.html"
---

# Send Server Alerts to Telegram with a Bot

Telegram bots are a simple way to deliver infrastructure alerts without maintaining a mail server or a custom mobile application. A shell script can notify you about backup failures, low disk space, service outages, or completed maintenance jobs.

## 1. Create the Bot

1. Open Telegram and start a chat with **@BotFather**.
2. Send `/newbot`.
3. Choose a display name and a unique username ending in `bot`.
4. Copy the API token returned by BotFather.

> Treat the token like a password. Anyone who has it can control the bot. If it is exposed, revoke it through BotFather and generate a new one.

## 2. Start the Conversation

Open your new bot in Telegram and press **Start**, or send it a message such as `/start`. Bots cannot initiate private conversations, so this step is required before the bot can alert you.

## 3. Get Your Chat ID

Temporarily export the token and verify it:

```
export TELEGRAM_BOT_TOKEN='replace-with-your-token'

curl --silent --show-error --fail \
  "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getMe" | jq
```

After sending a message to the bot, retrieve the pending updates:

```
curl --silent --show-error --fail \
  "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/getUpdates" \
  | jq '.result[-1].message.chat | {id, type, username}'
```

Save the value of `id`. Group IDs are normally negative. If `result` is empty, send another message and retry. A configured webhook must be removed before `getUpdates` can be used.

## 4. Send a Test Alert

```
export TELEGRAM_CHAT_ID='replace-with-your-chat-id'

curl --silent --show-error --fail \
  --request POST \
  "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  --data-urlencode "chat_id=${TELEGRAM_CHAT_ID}" \
  --data-urlencode "text=Test alert from $(hostname)"
```

A successful response contains `"ok":true`, and the message should appear immediately in Telegram.

## 5. Create a Reusable Alert Script

Store the credentials outside the script in `~/.config/telegram-alerts.env`:

```
mkdir -p ~/.config ~/bin
```

```
TELEGRAM_BOT_TOKEN='replace-with-your-token'
TELEGRAM_CHAT_ID='replace-with-your-chat-id'
```

```
chmod 600 ~/.config/telegram-alerts.env
```

After saving the file, remove the temporary interactive-shell variables:

```
unset TELEGRAM_BOT_TOKEN TELEGRAM_CHAT_ID
```

Create `~/bin/telegram-alert`:

```
#!/usr/bin/env bash
set -euo pipefail

source "${HOME}/.config/telegram-alerts.env"

if [[ $# -eq 0 ]]; then
  echo "Usage: telegram-alert MESSAGE" >&2
  exit 2
fi

message="[$(hostname)] $*"
# Telegram text messages are limited to 4096 characters.
message="${message:0:4096}"

curl --silent --show-error --fail \
  --retry 3 \
  --connect-timeout 5 \
  --max-time 15 \
  --request POST \
  "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  --data-urlencode "chat_id=${TELEGRAM_CHAT_ID}" \
  --data-urlencode "text=${message}" \
  >/dev/null
```

```
chmod 700 ~/bin/telegram-alert
~/bin/telegram-alert "Monitoring notifications are working"
```

## 6. Connect It to a Check

This example sends an alert when root filesystem usage reaches 90 percent:

```
usage=$(df -P / | awk 'NR==2 {gsub(/%/, "", $5); print $5}')

if (( usage >= 90 )); then
  ~/bin/telegram-alert "CRITICAL: / filesystem is ${usage}% full"
fi
```

The same script can be called from cron, systemd timers, backup jobs, database maintenance scripts, or monitoring systems that support custom notification commands. Always test both the alert condition and the recovery path.

## Operational Notes

- Use a dedicated bot and private chat or restricted group for operational alerts.
- Never send passwords, connection strings, customer data, or full logs.
- The API token is part of the request URL; protect process inspection, proxy logs, shell history, and debug output on shared systems.
- Set timeouts and retries so a Telegram problem does not block the monitored job.
- Include hostname, severity, service, timestamp, and a short action in each message.
- Keep another alert channel for critical systems; Telegram should not be the only path.

---

This small integration is enough for home labs and lightweight operations. Larger environments should connect Telegram to the central alert manager rather than configuring every server independently.
