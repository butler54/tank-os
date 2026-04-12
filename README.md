# tank-os

Fedora bootc image for running OpenClaw as a rootless Podman workload.

bootc turns a container image into a bootable, updateable Linux OS image. tank-os
uses it to package Fedora plus a rootless OpenClaw service into one VM, cloud, or
device image.

## Why This Is Useful

tank-os turns OpenClaw into a bootable Linux appliance. You publish one bootc
container image, build it into a cloud image, VM disk, or device image, and boot
machines that come up with the same rootless OpenClaw service every time.

The interesting part is that the OpenClaw runtime, host OS, Quadlet units, CLI
shim, and upgrade path all travel together as one OCI container image.
The mutable parts stay where users expect them: OpenClaw state under
`~openclaw/.openclaw`, API keys in the `openclaw` user's rootless Podman secret
store, and SSH access configured per instance.

That makes tank-os a good fit for:

- local demos that behave like the cloud target
- lab or device fleets where every machine gets its own OpenClaw interface
- sandboxed OpenClaw hosts with a mostly read-only, image-managed OS
- transactional updates through bootc instead of ad hoc host package changes
- per-machine secrets through rootless Podman secrets instead of baked-in API keys
- fast rollback and rebuild loops while developing OpenClaw host integrations

The result is still a normal Fedora system. Users SSH in as `openclaw`, edit
OpenClaw files in `~/.openclaw`, use the host `openclaw` CLI wrapper, and let
systemd/Podman keep the service running.

## Direction

- Base the OS image on Fedora bootc.
- Run OpenClaw through a rootless Podman Quadlet owned by an `openclaw` login user.
- Keep editable state in `~openclaw/.openclaw` so users can manage OpenClaw with the CLI and manually edit workspace files.
- Use Podman secrets in the `openclaw` user's rootless secret store instead of baking secrets into the image.
- Use cloud-init or VM-local provisioning to inject SSH keys and instance-specific access.

## Initial Scope

- Single OpenClaw gateway container.
- Rootless user Quadlet.
- Host-editable OpenClaw state under `/var/home/openclaw/.openclaw`.
- Optional service-gator sidecar for scoped GitHub/GitLab/Forgejo/JIRA access.
- Documentation for EC2, local macOS VM, and libvirt bootstrapping.

Sidecars such as LiteLLM and OTEL can come later once the single-container shape is working.

## Start Here

- Build the image: [docs/build.md](docs/build.md)
- Configure login access: [docs/provisioning.md](docs/provisioning.md)
- Use the OpenClaw CLI: [docs/cli.md](docs/cli.md)
- Configure model provider keys: [docs/model-providers.md](docs/model-providers.md)
- Configure service-gator: [docs/service-gator.md](docs/service-gator.md)

For bootc concepts and day-2 operations, see the upstream [bootc documentation](https://bootc-dev.github.io/bootc/).
For disk image builds, see the [Podman Desktop BootC extension](https://github.com/podman-desktop/extension-bootc)
and [bootc-image-builder docs](https://osbuild.org/docs/bootc/).

The host `openclaw` command delegates into the running OpenClaw container. Log in as `openclaw` when manually editing files under `~/.openclaw`.

For a Podman Desktop/macOS VM, see the [local macOS VM access notes](docs/provisioning.md#local-macos-vm) for finding the SSH port or guest IP.

## Demo Flow

For a short recording, a useful narration path is:

1. tank-os is a Fedora bootc image that starts OpenClaw as a rootless Podman
   Quadlet.
2. The same image can be turned into a local QCOW2, a cloud image, or a device
   image.
3. Podman Desktop can build and boot the local Linux VM on macOS.
4. The `openclaw` user owns the running service and editable state in
   `~/.openclaw`.
5. Secrets are added after boot with rootless Podman secrets, then synced with
   `tank-openclaw-secrets`.
6. The dashboard URL comes from `openclaw dashboard --no-open`.
7. Updates are image based: publish a new image and run `bootc switch` or
   `bootc upgrade`.

## Agent Prompt

Use this prompt with a coding agent to get oriented and run the local VM flow:

```text
I am working in the tank-os repo. This repo builds a Fedora bootc image that runs OpenClaw as a rootless Podman Quadlet owned by the `openclaw` user. Please help me get a local smoke test running.

Goals:
- Build or use the published bootc image `quay.io/sallyom/tank-os:latest` for arm64 or amd64.
- Build a QCOW2 disk image with the Podman Desktop BootC extension or manual bootc-image-builder flow.
- Start the disk image as a Linux VM on macOS/Podman Desktop.
- SSH in as `openclaw`, verify `sudo -n true`, `sudo bootc status`, `systemctl --user status openclaw.service`, and `podman ps`.
- If the VM is running under Podman Desktop/macadam, find the forwarded SSH port from the `gvproxy` process and use `ssh -i ~/.ssh/id_ed25519 -p <port> openclaw@localhost`.
- Use an SSH tunnel to open the UI from the host browser: `ssh -N -i ~/.ssh/id_ed25519 -p <port> -L 18789:127.0.0.1:18789 -L 18790:127.0.0.1:18790 openclaw@localhost`, then browse to `http://127.0.0.1:18789`.
- Print the dashboard URL from the VM with `openclaw dashboard --no-open`.
- Configure model provider and service-gator credentials using rootless Podman secrets as the `openclaw` user, then run `tank-openclaw-secrets`.

Constraints:
- Do not bake private keys or API keys into the image.
- Keep OpenClaw state editable under `~openclaw/.openclaw`.
- Prefer rootless Podman for the OpenClaw and service-gator services.
- Use `bootc switch --apply quay.io/sallyom/tank-os:latest` to test image upgrades after the VM is running.

Relevant docs in this repo:
- `docs/build.md`
- `docs/provisioning.md`
- `docs/cli.md`
- `docs/model-providers.md`
- `docs/service-gator.md`
```
