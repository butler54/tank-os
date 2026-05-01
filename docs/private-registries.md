# Private Container Registry Authentication

This guide covers authenticating to private container registries for both **tank-os image updates** (via bootc) and **container images** pulled by Podman (OpenClaw, service-gator).

## When You Need This

You need registry authentication in these scenarios:

1. **Private tank-os image** — You publish tank-os to a private registry and want VMs to pull updates via `bootc upgrade`
2. **Private OpenClaw image** — You use a private fork or build of OpenClaw instead of the public `ghcr.io/openclaw/openclaw:latest`
3. **Private service-gator image** — You use a private version of service-gator

## Two Authentication Scopes

tank-os has two separate container runtimes that need authentication:

1. **bootc** (OS-level image updates) — uses `/etc/ostree/auth.json`
2. **Podman** (rootless, runs as `openclaw` user) — uses `~openclaw/.config/containers/auth.json`

You must configure both if you're using private registries for both the OS image and the container workloads.

---

## 1. Authenticating bootc for OS Updates

bootc reads container registry credentials from `/etc/ostree/auth.json`. This file must exist **before** the first `bootc upgrade` attempt.

### Manual Configuration on a Running VM

SSH into the VM as the `openclaw` user (or another user with sudo), then:

```bash
sudo mkdir -p /etc/ostree
sudo bash -c 'cat > /etc/ostree/auth.json' <<'EOF'
{
  "auths": {
    "quay.io": {
      "auth": "BASE64_ENCODED_USERNAME:PASSWORD"
    }
  }
}
EOF
sudo chmod 600 /etc/ostree/auth.json
```

Replace `BASE64_ENCODED_USERNAME:PASSWORD` with the base64-encoded string `username:password`. Generate it:

```bash
echo -n "myuser:mypassword" | base64
```

Alternatively, use `podman login` to generate the auth file, then copy it:

```bash
# On your local machine
podman login quay.io
cat ~/.config/containers/auth.json
# Copy the JSON to /etc/ostree/auth.json on the VM
```

### Cloud-init Injection

To provision the auth file during VM creation, use cloud-init `write_files`:

```yaml
#cloud-config
write_files:
  - path: /etc/ostree/auth.json
    owner: root:root
    permissions: '0600'
    content: |
      {
        "auths": {
          "quay.io": {
            "auth": "BASE64_ENCODED_USERNAME:PASSWORD"
          }
        }
      }
```

Place this in your `user-data.yaml` before booting the VM.

---

## 2. Authenticating Podman for Container Images

The OpenClaw and service-gator containers run as **rootless Podman** under the `openclaw` user. Podman reads credentials from `~openclaw/.config/containers/auth.json` (or `~/.config/containers/auth.json` when logged in as `openclaw`).

### Manual Configuration on a Running VM

SSH into the VM as `openclaw`, then run `podman login`:

```bash
podman login ghcr.io
# Enter username and password/token when prompted
```

This writes to `~/.config/containers/auth.json` automatically. Verify:

```bash
cat ~/.config/containers/auth.json
```

When the Quadlet units restart (`systemctl --user restart openclaw.service`), Podman will use this auth file to pull private images thanks to the `Pull=newer` directive.

### Cloud-init Injection

To inject Podman credentials at provision time:

```yaml
#cloud-config
write_files:
  - path: /var/home/openclaw/.config/containers/auth.json
    owner: openclaw:openclaw
    permissions: '0600'
    content: |
      {
        "auths": {
          "ghcr.io": {
            "auth": "BASE64_ENCODED_USERNAME:PASSWORD"
          }
        }
      }

runcmd:
  - mkdir -p /var/home/openclaw/.config/containers
  - chown openclaw:openclaw /var/home/openclaw/.config/containers
  - chown openclaw:openclaw /var/home/openclaw/.config/containers/auth.json
  - chmod 600 /var/home/openclaw/.config/containers/auth.json
```

Ensure the `runcmd` block runs to set correct ownership and permissions.

---

## 3. GitHub Container Registry (GHCR) Example

OpenClaw images are published to `ghcr.io/openclaw/openclaw:latest`. If you're using a private fork or a private GHCR image, you need a **Personal Access Token (PAT)** with `read:packages` scope.

### Creating a GitHub PAT

1. Go to https://github.com/settings/tokens (classic tokens) or https://github.com/settings/personal-access-tokens/new (fine-grained)
2. For classic tokens: select `read:packages` scope
3. For fine-grained tokens: grant `Packages: Read-only` permission
4. Copy the token (starts with `ghp_` for classic, `github_pat_` for fine-grained)

### Logging in to GHCR

```bash
# As the openclaw user on the VM
echo "GITHUB_PAT" | podman login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

Or generate the auth JSON manually:

```bash
echo -n "YOUR_GITHUB_USERNAME:GITHUB_PAT" | base64
# Use the output in auth.json:
{
  "auths": {
    "ghcr.io": {
      "auth": "BASE64_OUTPUT_HERE"
    }
  }
}
```

### Full Example: Private OpenClaw Image

If you've published a private OpenClaw build to `ghcr.io/myorg/openclaw:latest`:

1. Update the Quadlet to reference your image:
   ```bash
   sudo mkdir -p /var/home/openclaw/.config/containers/systemd/openclaw.container.d
   sudo bash -c 'cat > /var/home/openclaw/.config/containers/systemd/openclaw.container.d/99-private-image.conf' <<EOF
   [Container]
   Image=ghcr.io/myorg/openclaw:latest
   EOF
   sudo chown -R openclaw:openclaw /var/home/openclaw/.config
   ```

2. Authenticate Podman as `openclaw`:
   ```bash
   sudo -iu openclaw
   podman login ghcr.io
   # Enter your GitHub username and PAT
   ```

3. Restart the service:
   ```bash
   systemctl --user daemon-reload
   systemctl --user restart openclaw.service
   ```

---

## 4. Quay.io Example

Quay.io is commonly used for bootc images. For automated access, use **robot accounts**.

### Creating a Quay.io Robot Account

1. Go to your Quay.io organization settings
2. Create a new robot account (e.g., `myorg+bootc-puller`)
3. Grant it `Read` permission on the `tank-os` repository
4. Copy the username (e.g., `myorg+bootc-puller`) and token

### Authenticating with Quay.io

```bash
echo "QUAY_TOKEN" | podman login quay.io -u "myorg+bootc-puller" --password-stdin
```

Or in `/etc/ostree/auth.json`:

```json
{
  "auths": {
    "quay.io": {
      "auth": "BASE64_ENCODED_myorg+bootc-puller:QUAY_TOKEN"
    }
  }
}
```

---

## 5. Complete Cloud-init Example

This example provisions both bootc and Podman auth for a VM that uses:
- Private tank-os image from `quay.io/myorg/tank-os:latest`
- Private OpenClaw image from `ghcr.io/myorg/openclaw:latest`

```yaml
#cloud-config
users:
  - name: openclaw
    groups:
      - wheel
    shell: /bin/bash
    lock_passwd: true
    ssh_authorized_keys:
      - ssh-ed25519 AAAAC3... your-key@example.com

write_files:
  # bootc auth for OS updates
  - path: /etc/ostree/auth.json
    owner: root:root
    permissions: '0600'
    content: |
      {
        "auths": {
          "quay.io": {
            "auth": "BASE64_QUAY_USER:TOKEN"
          }
        }
      }

  # Podman auth for container images
  - path: /var/home/openclaw/.config/containers/auth.json
    owner: openclaw:openclaw
    permissions: '0600'
    content: |
      {
        "auths": {
          "ghcr.io": {
            "auth": "BASE64_GITHUB_USER:PAT"
          }
        }
      }

runcmd:
  - mkdir -p /var/home/openclaw/.config/containers
  - chown -R openclaw:openclaw /var/home/openclaw/.config
  - chmod 600 /etc/ostree/auth.json
  - chmod 600 /var/home/openclaw/.config/containers/auth.json
  - loginctl enable-linger openclaw
```

Replace `BASE64_QUAY_USER:TOKEN` and `BASE64_GITHUB_USER:PAT` with your actual base64-encoded credentials.

---

## Summary

| Scope | Auth File | Owner | Used By |
|-------|-----------|-------|---------|
| bootc OS updates | `/etc/ostree/auth.json` | root:root (0600) | `bootc upgrade` |
| Podman containers | `~openclaw/.config/containers/auth.json` | openclaw:openclaw (0600) | Quadlet + Podman |

- Both auth files use the same JSON structure (Docker credential store format)
- Both can be injected via cloud-init `write_files` or created manually with `podman login`
- Private image refs go in Quadlet drop-ins or `openclaw.container` / `service-gator.container` Image= lines
