# BlueBubbles (iMessage)

Connect Hermes to Apple iMessage via [BlueBubbles](https://bluebubbles.app/) — a free, open-source macOS server that bridges iMessage to any device.

## Prerequisites

- A **Mac** (always on) running [BlueBubbles Server](https://bluebubbles.app/)
- Apple ID signed into Messages.app on that Mac
- BlueBubbles Server v1.0.0+ (webhooks require this version)
- Network connectivity between Hermes and the BlueBubbles server

## Setup

### 1. Install BlueBubbles Server

Download and install from [bluebubbles.app](https://bluebubbles.app/). Complete the setup wizard — sign in with your Apple ID and configure a connection method (local network, Ngrok, Cloudflare, or Dynamic DNS).

### 2. Get your Server URL and Password

In BlueBubbles Server → **Settings → API**, note:
- **Server URL** (e.g., `http://192.168.1.10:1234`)
- **Server Password**

### 3. Configure Hermes

Run the setup wizard:

```bash
hermes gateway setup
```

Select **BlueBubbles (iMessage)** and enter your server URL and password.

Or set environment variables directly in `~/.hermes/.env`:

```bash
BLUEBUBBLES_SERVER_URL=http://192.168.1.10:1234
BLUEBUBBLES_PASSWORD=your-server-password
```

#### Optional: Require mentions in group chats

By default, Hermes responds to every authorized BlueBubbles/iMessage DM or group message. To make group chats opt-in, enable mention gating:

```yaml
platforms:
  bluebubbles:
    enabled: true
    extra:
      require_mention: true
```

With `require_mention: true`, DMs still work normally, but group-chat messages are ignored unless they match a mention pattern. If you do not configure custom patterns, Hermes uses conservative defaults for `Hermes` and `@Hermes agent` variants.

For a custom agent name, set regex patterns:

```yaml
platforms:
  bluebubbles:
    extra:
      require_mention: true
      mention_patterns:
        - '(?<![\w@])@?amos\b[,:\-]?'
```

### 4. Authorize Users

Choose one approach:

**DM Pairing (recommended):**
When someone messages your iMessage, Hermes automatically sends them a pairing code. Approve it with:
```bash
hermes pairing approve bluebubbles <CODE>
```
Use `hermes pairing list` to see pending codes and approved users.

**Pre-authorize specific users** (in `~/.hermes/.env`):
```bash
BLUEBUBBLES_ALLOWED_USERS=user@icloud.com,+15551234567
```

**Open access** (in `~/.hermes/.env`):
```bash
BLUEBUBBLES_ALLOW_ALL_USERS=true
```

### 5. Start the Gateway

```bash
hermes gateway run
```

Hermes will connect to your BlueBubbles server, register a webhook, and start listening for iMessage messages.

The webhook Hermes registers is:

```text
http://localhost:8645/bluebubbles-webhook?password=<BLUEBUBBLES_PASSWORD>
```

(`BLUEBUBBLES_WEBHOOK_HOST` / `PORT` / `PATH` change the bind; default host `127.0.0.1` is advertised as `localhost`.) You do **not** normally paste this into the BlueBubbles UI — the adapter POSTs it to `/api/v1/webhook` on connect. Confirm in BlueBubbles → Settings → API → Webhooks that **one** Hermes URL is listed.

Then verify the adapter actually started:

```bash
hermes config get platforms.bluebubbles.enabled   # must be true
hermes logs gateway | grep bluebubbles
```

You want `connected to http://…`, `webhook listening on …`, and `webhook registered with server`. If you see `explicitly disabled by platforms.bluebubbles.enabled: false`, setup saved `.env` credentials but did **not** enable the platform — run `hermes config set platforms.bluebubbles.enabled true` and restart the gateway.

`GET /api/v1/server/info` on the BlueBubbles API (`computer_id`, `detected_imessage`) is the Apple ID that will send. Two BlueBubbles apps on one Mac (two macOS users) are two Apple IDs and need **two API ports**. See [Multiple Users on the Same Mac](https://docs.bluebubbles.app/server/basic-guides/multiple-users-on-the-same-mac) (menu-bar Fast User Switching, do not log out the server user).

## How It Works

```
iMessage → Messages.app → BlueBubbles Server → Webhook → Hermes
Hermes → BlueBubbles REST API → Messages.app → iMessage
```

- **Inbound:** BlueBubbles sends webhook events to a local listener when new messages arrive. No polling — instant delivery.
- **Outbound:** Hermes sends messages via the BlueBubbles REST API.
- **Media:** Images, voice messages, videos, and documents are supported in both directions. Inbound attachments are downloaded and cached locally for the agent to process.

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `BLUEBUBBLES_SERVER_URL` | Yes | — | BlueBubbles server URL |
| `BLUEBUBBLES_PASSWORD` | Yes | — | Server password |
| `BLUEBUBBLES_WEBHOOK_HOST` | No | `127.0.0.1` | Webhook listener bind address |
| `BLUEBUBBLES_WEBHOOK_PORT` | No | `8645` | Webhook listener port |
| `BLUEBUBBLES_WEBHOOK_PATH` | No | `/bluebubbles-webhook` | Webhook URL path |
| `BLUEBUBBLES_HOME_CHANNEL` | No | — | Phone/email for cron delivery |
| `BLUEBUBBLES_ALLOWED_USERS` | No | — | Comma-separated authorized users |
| `BLUEBUBBLES_ALLOW_ALL_USERS` | No | `false` | Allow all users |
| `BLUEBUBBLES_REQUIRE_MENTION` | No | `false` | Require a mention pattern before responding in group chats |
| `BLUEBUBBLES_MENTION_PATTERNS` | No | Hermes wake words | JSON array, newline-separated, or comma-separated regex patterns for group mention matching |

Auto-marking messages as read is controlled by the `send_read_receipts` key under `platforms.bluebubbles.extra` in `~/.hermes/config.yaml` (default: `true`). There is no corresponding environment variable.

## Features

### Text Messaging
Send and receive iMessages. Markdown is automatically stripped for clean plain-text delivery.

### Rich Media
- **Images:** Photos appear natively in the iMessage conversation
- **Voice messages:** Audio files sent as iMessage voice messages
- **Videos:** Video attachments
- **Documents:** Files sent as iMessage attachments

### Tapback Reactions
Love, like, dislike, laugh, emphasize, and question reactions. Requires the BlueBubbles [Private API helper](https://docs.bluebubbles.app/helper-bundle/installation).

### Typing Indicators
Shows "typing..." in the iMessage conversation while the agent is processing. Requires Private API.

### Read Receipts
Automatically marks messages as read after processing. Requires Private API.

### Chat Addressing
You can address chats by email or phone number — Hermes resolves them to BlueBubbles chat GUIDs automatically. No need to use raw GUID format.

## Private API

Some features require the BlueBubbles [Private API helper](https://docs.bluebubbles.app/helper-bundle/installation):
- Tapback reactions
- Typing indicators
- Read receipts
- Creating new chats by address

Without the Private API, basic text messaging and media still work.

## Troubleshooting

### "Cannot reach server"
- Verify the server URL is correct and the Mac is on
- Check that BlueBubbles Server is running
- Ensure network connectivity (firewall, port forwarding)

### Messages not arriving
- Check that **exactly one** Hermes webhook is registered in BlueBubbles Server → Settings → API → Webhooks (`http://localhost:8645/bluebubbles-webhook?password=…`). Delete leftover hooks (dead ports, missing `http://`, events `*`).
- A row in that list is not proof of delivery. Proof is `hermes logs gateway` showing `inbound message: platform=bluebubbles`.
- If the message is in BlueBubbles (chat.db / API `message/count`) but Hermes has no inbound line, BlueBubbles is not POSTing or the send queue is wedged (see home-channel spam below).
- `curl http://127.0.0.1:8645/health` should return `ok`. `localhost` vs IPv6 `::1` can fail if the listener is IPv4-only.

### Setup saved credentials but nothing listens on 8645
- `hermes gateway setup` writes `BLUEBUBBLES_SERVER_URL` / `BLUEBUBBLES_PASSWORD` only. It does not set `platforms.bluebubbles.enabled: true`.
- If that key is `false` (for example Photon was the iMessage path), env credentials **do not** start the adapter. Set `platforms.bluebubbles.enabled: true` and restart.
- Do not run Photon and BlueBubbles both enabled.

### Wrong Apple ID / two BlueBubbles on one Mac
- Hermes talks to whatever `BLUEBUBBLES_SERVER_URL` answers (`:1234` vs `:1235`). That process's Messages.app is the from-address (`detected_imessage` on `/api/v1/server/info`).
- Give the second macOS user a **different API port**. Fast User Switch from the menu bar so that session stays logged in.
- A stale Server URL in the BlueBubbles UI (old DHCP IP) is not what Hermes uses.

### Two replies per iMessage
- Gateway log shows the same text twice with `chat=any;-;+…` and `chat=+…` (two session keys). That is `new-message` plus `updated-message` (delivery/read) without chat-id canonicalization — see #30708 / #34372.
- Also check you do not have two webhooks or Photon + BlueBubbles both on.

### Home-channel recovered-reply spam / AppleScript -1743
- `BLUEBUBBLES_HOME_CHANNEL` / `platforms.bluebubbles.home_channel` makes the gateway text that chat on restart. Failed AppleScript sends (`Not authorized to send Apple events to Messages. (-1743)`, 150s timeouts) retry as `♻️ Recovered reply` and can starve webhooks.
- Leave home channel empty until a `method: private-api` send works on **that** BlueBubbles user. Use Telegram/etc. for home if needed.
- Private API is required for reliable send when BlueBubbles runs in a background macOS user; AppleScript often does not.

### "Private API helper not connected"
- Install the Private API helper: [docs.bluebubbles.app](https://docs.bluebubbles.app/helper-bundle/installation)
- Basic messaging works without it — only reactions, typing, and read receipts require it

