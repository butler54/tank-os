# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

tank-os builds a Fedora bootc image that packages OpenClaw as a rootless Podman workload. The result is a bootable Linux appliance where the OpenClaw runtime, host OS, Quadlet units, CLI shim, and upgrade path all travel together as one OCI container image.

bootc turns a container image into a bootable OS image (VM disk, cloud image, or device image). tank-os uses this to create machines that boot with an already-configured rootless OpenClaw service running as the `openclaw` user.

## Build Commands

### Quick build with Makefile

```bash
make build  # Auto-detects architecture (arm64 or amd64)
make help   # Show all targets
```

### Manual build

For arm64 (Apple Silicon):
```bash
podman build --platform linux/arm64 -t localhost/tank-os:latest -f bootc/Containerfile bootc
```

For x86_64:
```bash
podman build --platform linux/amd64 -t localhost/tank-os:latest -f bootc/Containerfile bootc
```

The final `bootc` argument is the build context directory (contains `Containerfile` and `rootfs/`).

### Build a disk image manually

From the repo root, create output directory:
```bash
mkdir -p out-tank-os
```

Build QCOW2 with bootc-image-builder (requires rootful Podman machine on macOS):
```bash
podman --connection podman-machine-default-root run \
  --rm --tty --privileged \
  --security-opt label=type:unconfined_t \
  -v "$PWD/out-tank-os:/output/" \
  -v /var/lib/containers/storage:/var/lib/containers/storage \
  quay.io/centos-bootc/bootc-image-builder:latest \
  localhost/tank-os:latest \
  --output /output/ \
  --local \
  --type qcow2 \
  --target-arch arm64 \
  --rootfs xfs
```

Output: `out-tank-os/qcow2/disk.qcow2`

For Podman Desktop BootC extension workflow, see `docs/build.md`.

### Upgrade a running VM

After booting a disk image, switch to a registry-tracked ref for future updates:
```bash
sudo bootc status
sudo bootc switch --apply quay.io/sallyom/tank-os:latest
```

Later updates against the same tracked tag:
```bash
sudo bootc upgrade --apply
```

## Architecture Overview

### User and Service Layout

- **User**: `openclaw` (UID/GID 1000) owns the rootless Podman services
- **State directory**: `/var/home/openclaw/.openclaw` (same as `~openclaw/.openclaw`)
- **Quadlet units**: `/etc/containers/systemd/users/1000/openclaw.container` and `service-gator.container`
  - These are systemd user service units managed by Podman's Quadlet generator
  - They run as rootless containers under the `openclaw` user
- **CLI wrapper**: `/usr/local/bin/openclaw` delegates into the running OpenClaw container
  - When run as non-openclaw user, it uses `sudo -iu openclaw` to delegate
  - When run as openclaw, it does `podman exec` into the container
- **Tailscale (optional)**: Pre-installed but not activated. Provides secure remote access without SSH tunnels
  - `tailscaled` service is enabled but does nothing until authenticated
  - See `docs/tailscale.md` for setup and Tailscale Serve configuration
- **signal-cli (optional)**: Pre-installed but not activated. Enables OpenClaw to send/receive Signal messages
  - Runs as a systemd user service (`signal-cli.service`) once registered
  - Communicates with OpenClaw via JSON-RPC on `http://127.0.0.1:8181` (port 8080 conflicts with service-gator)
  - Must run in single-account mode (`-a +NUMBER`) with `--receive-mode=on-start`; without these, SSE events are silently dropped
  - Account number goes in a drop-in: `~/.config/systemd/user/signal-cli.service.d/10-account.conf`
  - Set `autoStart: false` in OpenClaw config — signal-cli lives on the host, not inside the container
  - Use UUIDs in `allowFrom` — Signal privacy hides `sourceNumber` by default
  - See `docs/signal.md` for registration and OpenClaw channel configuration

### Bootstrap Flow

On first boot or service start:
1. `bootstrap-openclaw` creates `~/.openclaw` structure if missing, writes default `openclaw.json`
2. `bootstrap-service-gator` creates `~/.config/service-gator` if missing
3. Quadlet units pull images (if needed) and start containers
4. OpenClaw gateway binds to `127.0.0.1:18789` and `127.0.0.1:18790`
5. service-gator binds to `127.0.0.1:8080` (MCP server)

### Secrets Management

- Model provider API keys are stored as **rootless Podman secrets** owned by the `openclaw` user
- The `tank-openclaw-secrets` helper syncs secrets to:
  1. Quadlet drop-ins at `~/.config/containers/systemd/openclaw.container.d/`
  2. SecretRefs in `~/.openclaw/openclaw.json`
- Secrets never appear in the bootc image or checked-in config files

Standard secret names: `anthropic_api_key`, `openai_api_key`, `gemini_api_key`, `openrouter_api_key`, `telegram_bot_token`

See `docs/model-providers.md` for the full provider mapping and custom provider examples.

### File Structure

```
bootc/
├── Containerfile              # Main image definition
└── rootfs/                    # Overlay files (copied to / during build)
    ├── etc/
    │   ├── containers/systemd/users/1000/
    │   │   ├── openclaw.container      # OpenClaw Quadlet
    │   │   └── service-gator.container # service-gator Quadlet
    │   ├── sudoers.d/openclaw          # Passwordless sudo for openclaw user
    │   └── tank-os-release             # Version metadata
    ├── etc/systemd/user/
    │   └── signal-cli.service          # signal-cli JSON-RPC daemon (disabled until registered)
    ├── usr/libexec/tank-os/
    │   ├── bootstrap-openclaw          # First-run setup for OpenClaw
    │   ├── bootstrap-service-gator     # First-run setup for service-gator
    │   └── sync-podman-secrets         # Internal helper for tank-openclaw-secrets
    └── usr/local/bin/
        ├── openclaw                    # CLI wrapper (delegates to container)
        ├── signal-cli                  # Symlink → /opt/signal-cli-VERSION/bin/signal-cli
        ├── tank-openclaw-secrets       # Sync Podman secrets to OpenClaw config
        └── tank-os-version             # Print image version

docs/
├── build.md                   # Build instructions and CI/CD setup
├── provisioning.md            # SSH/cloud-init/VM access
├── cli.md                     # OpenClaw CLI wrapper details
├── model-providers.md         # Secrets and provider config
├── service-gator.md           # service-gator setup
├── tailscale.md               # Tailscale setup (optional feature)
├── signal.md                  # Signal messaging setup via signal-cli (optional feature)
└── private-registries.md      # Private registry authentication (bootc + Podman)

examples/cloud-init/
└── openclaw-user-data.yaml    # Cloud-init template for provisioning

.github/workflows/
├── pr.yaml                    # PR build validation
├── create-release.yml         # Semantic versioning and tag creation
├── build-release.yml          # Multi-arch build, signing, SBOM, attestation
├── commitlint.yml             # PR title validation
└── scorecard.yml              # OpenSSF Scorecard security analysis

Makefile                       # Local build helpers
commitlint.config.js           # Conventional commit rules
```

## Local Testing Workflow (macOS/Podman Desktop)

### Build and boot a VM

1. Build the bootc image (or use `quay.io/sallyom/tank-os:latest`)
2. Use Podman Desktop BootC extension to build a QCOW2 disk image
   - Set user to `openclaw`
   - Paste your SSH public key
   - Set groups to `wheel`
3. Start the VM from Podman Desktop

### SSH Access

Find the forwarded SSH port (gvproxy exposes it):
```bash
export PORT="$(ps aux | grep 'gvproxy' | grep 'bootc.*tank' | sed -nE 's/.*-ssh-port ([0-9]+).*/\1/p' | tail -1)"
echo "$PORT"
```

Connect:
```bash
ssh -i ~/.ssh/id_ed25519 -p "$PORT" openclaw@localhost
```

### UI Access via SSH Tunnel

Open an SSH tunnel to forward the OpenClaw UI ports:
```bash
ssh -N -i ~/.ssh/id_ed25519 -p "$PORT" \
  -L 18789:127.0.0.1:18789 \
  -L 18790:127.0.0.1:18790 \
  openclaw@localhost
```

Then browse to `http://127.0.0.1:18789` on the Mac.

### Verify Services

After SSH in as `openclaw`:
```bash
sudo -n true                              # Verify passwordless sudo
sudo bootc status                         # Show bootc deployment status
systemctl --user status openclaw.service  # Check OpenClaw service
systemctl --user status service-gator.service
podman ps                                 # List running containers
openclaw status --deep                    # OpenClaw CLI status
openclaw dashboard --no-open              # Print dashboard URL
```

### Configure Secrets

As the `openclaw` user on the VM:
```bash
sudo -iu openclaw
printf '%s' "$ANTHROPIC_API_KEY" | podman secret create anthropic_api_key -
printf '%s' "$OPENAI_API_KEY" | podman secret create openai_api_key -
tank-openclaw-secrets
systemctl --user restart openclaw.service
```

## Important Notes for Development

- **Do not bake secrets into the image**: Private SSH keys, API keys, and tokens must be added at provision time (cloud-init, Podman Desktop user form, or post-boot via Podman secrets)
- **Private registry authentication**: If using private images, configure auth files at provision time. See `docs/private-registries.md` for bootc (`/etc/ostree/auth.json`) and Podman (`~openclaw/.config/containers/auth.json`) authentication
- **State is mutable**: The `~openclaw/.openclaw` directory is writable and persists across reboots (it's on the root filesystem, not in the container)
- **Rootless by default**: Both OpenClaw and service-gator run as rootless Podman containers owned by the `openclaw` user, not root
- **CLI wrapper delegation**: The `/usr/local/bin/openclaw` command automatically delegates to the `openclaw` user and `podman exec`s into the running container
- **Quadlet units**: Changes to `.container` files in `bootc/rootfs/etc/containers/systemd/users/1000/` require rebuilding the bootc image. Runtime overrides go in `~/.config/containers/systemd/openclaw.container.d/` on the running VM.
- **Test image security**: The built image grants the `openclaw` user passwordless sudo for testing convenience. Production deployments should use a separate admin user or tightly scoped sudo policy.

## Testing Image Updates

After building a new bootc image and pushing to a registry:
```bash
# On the running VM:
sudo bootc switch --apply quay.io/sallyom/tank-os:latest
# Reboot to activate
sudo systemctl reboot
```

Or if already tracking a registry ref:
```bash
sudo bootc upgrade --apply
sudo systemctl reboot
```

## Relevant Documentation

- bootc upstream: https://bootc-dev.github.io/bootc/
- bootc-image-builder: https://osbuild.org/docs/bootc/
- Podman Desktop BootC extension: https://github.com/podman-desktop/extension-bootc
- Quadlet (systemd-managed Podman): https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html
