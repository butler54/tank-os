# Signal Integration

tank-os includes [signal-cli](https://github.com/AsamK/signal-cli), which lets OpenClaw send and receive Signal messages. signal-cli runs as a local JSON-RPC daemon; OpenClaw connects to it over `http://127.0.0.1:8080`.

Signal support is **optional** — signal-cli is pre-installed but the daemon is not started until you register a phone number and enable the service.

## Prerequisites

- A **dedicated phone number** for the bot. Options:
  - A spare SIM/phone number you don't use personally
  - A VoIP number (e.g. [JMP.chat](https://jmp.chat), Google Voice, Twilio)
  - Link to an existing Signal account instead of registering a new number (see Option A below)
- SSH access to the VM as the `openclaw` user

> Do not use your personal Signal account as the bot account. Signal only allows one primary device per number; using your personal number will disrupt your own Signal access.

## Step 1: Register signal-cli

SSH in as `openclaw`:

```bash
ssh openclaw@<your-vm>
```

Choose one of the following registration paths.

### Option A — Link to an existing Signal account (no new number needed)

This adds the VM as a linked device on your existing Signal account. Your phone remains the primary device.

Install `qrencode` if needed (`sudo dnf install -y qrencode`), then:

```bash
signal-cli link -n "OpenClaw" | qrencode -t ANSIUTF8
```

Open **Signal** on your phone → Settings → Linked Devices → tap **+** → scan the QR code printed in the terminal.

After scanning, note the phone number associated with your account — you'll need it for the OpenClaw config.

### Option B — Register a new standalone number

1. Get a captcha token from [signalcaptchas.org](https://signalcaptchas.org/registration/generate.html)
2. Register:
   ```bash
   signal-cli -a +15551234567 register --captcha <token>
   ```
3. Enter the SMS verification code you receive:
   ```bash
   signal-cli -a +15551234567 verify <code>
   ```

Replace `+15551234567` with your actual phone number in E.164 format.

## Step 2: Enable the daemon

```bash
systemctl --user enable --now signal-cli.service
systemctl --user status signal-cli.service
```

Verify the JSON-RPC interface is up:

```bash
curl -s -X POST http://127.0.0.1:8080/api/v1/rpc \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"version","id":1}'
```

You should see a response containing the signal-cli version.

## Step 3: Configure OpenClaw

Edit `~/.openclaw/openclaw.json` and add a `channels` block (or merge into an existing one):

```json
{
  "channels": {
    "signal": {
      "enabled": true,
      "account": "+15551234567",
      "httpUrl": "http://127.0.0.1:8080",
      "autoStart": false,
      "apiMode": "jsonrpc",
      "dmPolicy": "pairing"
    }
  }
}
```

Replace `+15551234567` with the phone number from Step 1.

Restart OpenClaw to apply:

```bash
systemctl --user restart openclaw.service
```

## Step 4: Verify

Send a direct message to the bot's Signal number from your phone. OpenClaw will reply with a pairing code (because `dmPolicy` is `"pairing"` — unknown senders must authenticate first). Enter the code in Signal to complete pairing.

## Access control

| `dmPolicy` | Behaviour |
|------------|-----------|
| `"pairing"` | Unknown senders get a 1-hour pairing code (default, recommended) |
| `"allowlist"` | Only numbers in `allowFrom` can message the bot |
| `"open"` | Anyone can message the bot with no approval |
| `"disabled"` | DMs disabled; group messages only |

Example allowlist config:

```json
"signal": {
  "enabled": true,
  "account": "+15551234567",
  "httpUrl": "http://127.0.0.1:8080",
  "autoStart": false,
  "apiMode": "jsonrpc",
  "dmPolicy": "allowlist",
  "allowFrom": ["+19995550100"]
}
```

## Signal data storage

signal-cli stores account keys and message history at:

```
~/.local/share/signal-cli/
```

This directory is on the mutable root filesystem and persists across reboots. Back it up before migrating the VM — losing these files requires re-registration.

## Upgrading signal-cli

The signal-cli version is pinned in the Containerfile via `ARG SIGNAL_CLI_VERSION`. To upgrade, update that value and rebuild the image. After a `bootc upgrade` the new binary will be at `/usr/local/bin/signal-cli`; existing account data in `~/.local/share/signal-cli/` is preserved.
