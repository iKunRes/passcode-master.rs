# Ingress Passcode Forwarder

A Telegram bot that collects Ingress passcodes from authorized users, auto-redeems them across multiple accounts, and reports results back via Telegram. Optionally exposes a WebSocket endpoint so external clients can subscribe to new passcode events in real time.

## Features

- **Passcode forwarding** — send a passcode to the bot and it is posted to a configured target Telegram channel.
- **Access levels** — three-tier permission model (`Cookie`, `Send`, `All`) managed entirely through Telegram inline keyboards. Admins approve users without any external tooling.
- **TOTP invite flow** — new users send a `/auth <totp-code>` to request access; admins receive a prompt and choose which access level to grant.
- **WebSocket API** — optional HTTP server (Axum) with a `/ws` endpoint that pushes new passcodes to authenticated subscribers in real time.
- **Custom Telegram server support** — the `[platform] server` field lets you point the bot at a self-hosted Bot API server.

## Architecture

```
Telegram Bot (teloxide)
  ├── /auth      — TOTP-gated user registration
  ├── /cookie    — manage stored cookies (add / modify / toggle / query)
  ├── /log       — query history (admin only)
  ├── /resent    — re-broadcast a code (admin only)
  ├── /invite    — generate a one-time TOTP auth code (admin only)
  ├── /ping      — show chat ID, access level, and bot version
  └── <text>     — passcode lines forwarded to the target channel

Web server (Axum, optional)
  ├── GET /      → JSON version response
  └── GET /ws    → WebSocket; push new passcodes to authenticated subscribers
```

All components communicate through a `tokio::sync::broadcast` channel. The database layer (SQLite via sqlx) runs in its own task; all mutations go through a `DatabaseHelper` channel handle.

## Configuration

Copy the example below to `config.toml` (the default path) and fill in your values.

```toml
# Telegram user IDs that have full admin rights
admin = [123456789]

# TOTP secret (Base32-encoded) used for the /auth invite flow
totp = "JBSWY3DPEHPK3PXP"

# SQLite database path (created automatically on first run)
database = "data.db"

[platform]
# Telegram Bot API token
key = "1234567890:AAF..."
# Target channel/group chat ID where passcodes are posted
target = -1001234567890
# Optional: self-hosted Bot API server URL
# server = "http://localhost:8081"

[web]
enabled = false
bind    = "0.0.0.0:26511"
# Optional URL prefix (e.g. "/passcode")
# prefix = "/passcode"
# Argon2 hash of the WebSocket client password
access_key = "$argon2id$v=19$..."
```

### Running

```bash
# Default config path (config.toml)
cargo run --release

# Custom config path
cargo run --release -- /path/to/config.toml

# Systemd mode (omits timestamps from log lines)
cargo run --release -- --systemd
```

### WebSocket API

When `[web] enabled = true`, clients can connect to `ws://<bind>/ws` and authenticate by sending a JSON message:

```json
{ "codename": "<your-name>", "hash": "<argon2-hash-of-access-key>" }
```

After a successful authentication the server pushes each new passcode as a plain-text WebSocket message. Send `"close"` to disconnect gracefully.

## Requirements

- Rust 2024 edition (stable)
- A Telegram bot token (from [@BotFather](https://t.me/BotFather))
- An Ingress Intel account with valid cookies for each account you want to redeem on

## License

[![](https://www.gnu.org/graphics/agplv3-155x51.png)](https://www.gnu.org/licenses/agpl-3.0.txt)

Copyright (C) 2024 KunoiSayami

This program is free software: you can redistribute it and/or modify it under the terms of the GNU Affero General Public License as published by the Free Software Foundation, either version 3 of the License, or any later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU Affero General Public License for more details.

You should have received a copy of the GNU Affero General Public License along with this program. If not, see <https://www.gnu.org/licenses/>.
