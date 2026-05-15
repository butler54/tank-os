# Signal Integration

tank-os includes [signal-cli](https://github.com/AsamK/signal-cli), which lets OpenClaw send and receive Signal messages. signal-cli runs as a local JSON-RPC HTTP daemon; OpenClaw connects to it over `http://127.0.0.1:8181`.

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

## Step 2: Create the daemon drop-in

The base `signal-cli.service` unit uses port 8181 and `--receive-mode=on-start`. You must create a drop-in to supply the account number, since that is deployment-specific:

```bash
mkdir -p ~/.config/systemd/user/signal-cli.service.d
cat > ~/.config/systemd/user/signal-cli.service.d/10-account.conf << 'EOF'
[Service]
ExecStart=
ExecStart=/usr/local/bin/signal-cli -a +15551234567 --output=json daemon --http=0.0.0.0:8181 --receive-mode=on-start
EOF
```

Replace `+15551234567` with your registered number.

> **Port note**: The base unit uses port 8181. Do not change this to 8080 — that port is occupied by service-gator.
>
> **`--receive-mode=on-start`**: Required. Without it, signal-cli defaults to `on-connection` mode in multi-account configurations, which silently drops SSE events and prevents OpenClaw from receiving messages.
>
> **`0.0.0.0` bind**: Required when the OpenClaw container uses `Network=host` (which it does in tank-os). Binding to `127.0.0.1` would make the daemon unreachable from the container's network namespace.

## Step 3: Enable the daemon

```bash
systemctl --user daemon-reload
systemctl --user enable --now signal-cli.service
systemctl --user status signal-cli.service
```

Verify the JSON-RPC interface is up:

```bash
curl -s -X POST http://127.0.0.1:8181/api/v1/rpc \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"version","id":1}'
```

You should see a response containing the signal-cli version.

## Step 4: Configure OpenClaw

Use `openclaw config set` to configure the Signal channel (this updates both the running gateway and the config file):

```bash
openclaw config set channels.signal.enabled true
openclaw config set channels.signal.account "+15551234567"
openclaw config set channels.signal.httpUrl "http://127.0.0.1:8181"
openclaw config set channels.signal.autoStart false
openclaw config set channels.signal.dmPolicy "allowlist"
openclaw config set channels.signal.allowFrom '["your-uuid-here", "+15559990100"]'
```

Then restart OpenClaw to start the channel:

```bash
systemctl --user restart openclaw.service
```

> **`autoStart: false`**: Required. OpenClaw must not try to spawn signal-cli itself — the binary lives on the host, not inside the OpenClaw container. signal-cli is managed by its own systemd user service.

## Step 5: Verify

```bash
openclaw status --deep
```

The Signal channel should show `ON · OK · configured`. Send a direct message to the bot's Signal number. OpenClaw should reply within seconds (when using a fast local model) or up to several minutes (CPU-only inference).

## Access control

| `dmPolicy` | Behaviour |
|------------|-----------|
| `"pairing"` | Unknown senders get a 1-hour pairing code (recommended for open access) |
| `"allowlist"` | Only entries in `allowFrom` can message the bot |
| `"open"` | Anyone can message the bot with no approval |
| `"disabled"` | DMs disabled; group messages only |

### Using UUIDs in the allowlist

Signal's privacy model no longer shares sender phone numbers by default — incoming messages arrive with `"sourceNumber": null` and only a UUID. **Use UUIDs in `allowFrom`**, not phone numbers, unless the sender has explicitly enabled phone number sharing.

To find sender UUIDs, check the signal-cli journal after receiving a message:

```bash
journalctl --user -u signal-cli --since "1 hour ago" --no-pager \
  | grep dataMessage \
  | grep -o '"sourceUuid":"[^"]*"' \
  | sort -u
```

Example allowlist config with UUIDs:

```bash
openclaw config set channels.signal.dmPolicy "allowlist"
openclaw config set channels.signal.allowFrom \
  '["22b556a4-b8ad-4d9c-a8c2-6175d1b25047", "+15559990100"]'
```

Phone numbers in `allowFrom` still work for senders who have enabled phone number visibility.

## Signal data storage

signal-cli stores account keys and message history at:

```
~/.local/share/signal-cli/
```

This directory is on the mutable root filesystem and persists across reboots. Back it up before migrating the VM — losing these files requires re-registration.

## Upgrading signal-cli

The signal-cli version is pinned in the Containerfile via `ARG SIGNAL_CLI_VERSION`. To upgrade, update that value and rebuild the image. After a `bootc upgrade` the new binary will be at `/usr/local/bin/signal-cli`; existing account data in `~/.local/share/signal-cli/` is preserved.
