# Tailscale Support (Optional)

## Overview

Tailscale is **pre-installed but not activated** in tank-os. It's an optional convenience for users who want secure remote access without SSH tunnels or port forwarding. The `tailscaled` service is enabled and starts on boot, but Tailscale does nothing until you authenticate it. Users who don't want Tailscale can simply ignore it—it consumes no resources when not authenticated.

**SSH tunnel access continues to work regardless of Tailscale configuration.**

## What Tailscale Provides

- **Secure remote access** to your tank-os VM from anywhere without exposing ports to the internet
- **Zero-configuration networking** between your devices on a private mesh network (tailnet)
- **HTTPS endpoints** via Tailscale Serve for exposing OpenClaw's web UI
- **MagicDNS** for easy hostname-based access (`tank-os.tailnet.ts.net`)

## Authenticating Tailscale

Tailscale requires authentication before it does anything. You have two options:

### Interactive Authentication

SSH into the VM and run:

```bash
sudo tailscale up --hostname=$(hostname)
```

This will print a URL. Visit it in your browser, authenticate with your Tailscale account, and authorize the device.

### Auth Key (Automated Deployments)

For cloud-init or automated provisioning, use an auth key from [Tailscale admin console](https://login.tailscale.com/admin/settings/keys):

```bash
sudo tailscale up --authkey=tskey-auth-xxxxx --hostname=$(hostname)
```

#### Cloud-init Example

Add to your `user-data.yaml`:

```yaml
runcmd:
  - tailscale up --authkey=tskey-auth-xxxxx --hostname=tank-os
```

## Verifying Connectivity

After authentication, check status:

```bash
tailscale status
```

You should see your device listed with an IP address (e.g., `100.x.y.z`).

## Exposing OpenClaw via Tailscale Serve (Optional)

By default, OpenClaw binds to `127.0.0.1:18789` and `127.0.0.1:18790` (loopback only). You can expose it to your tailnet using **Tailscale Serve**, which proxies HTTPS traffic from your tailnet to OpenClaw's loopback ports.

### Setting Up Tailscale Serve

Run as root:

```bash
sudo tailscale serve --bg https / http://127.0.0.1:18789
sudo tailscale serve --bg https /ws http://127.0.0.1:18790
```

The `--bg` flag runs the serve configuration in the background (persists across reboots).

### Accessing OpenClaw

Once configured, access the OpenClaw UI from any device on your tailnet:

```
https://<hostname>.tailnet.ts.net/
```

Replace `<hostname>` with your VM's hostname (shown in `tailscale status`).

### Checking Serve Status

```bash
sudo tailscale serve status
```

### Removing Serve Configuration

To stop exposing OpenClaw via Tailscale:

```bash
sudo tailscale serve --bg off
```

## OpenClaw Tailscale Integration

OpenClaw has native Tailscale integration features documented at [https://docs.openclaw.ai/gateway/tailscale](https://docs.openclaw.ai/gateway/tailscale), including:

- **Tailscale identity-based authentication** (tokenless access for tailnet users)
- **Gateway mode with Tailscale Serve** (OpenClaw binds to loopback, Tailscale provides HTTPS)

The basic Tailscale Serve setup described above is sufficient for most use cases. For advanced integration (mounting the Tailscale socket into the container, enabling identity-based auth), consult the OpenClaw upstream documentation.

## Disabling Tailscale

If you don't want Tailscale at all:

```bash
sudo systemctl disable --now tailscaled
```

Or simply leave it unauthenticated—it won't consume resources or open any ports.
