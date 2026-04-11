# tank-os

Fedora bootc image for running OpenClaw as a rootless Podman workload.

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

The host `openclaw` command delegates into the running OpenClaw container. Log in as `openclaw` when manually editing files under `~/.openclaw`.

For a Podman Desktop/macOS VM, see the [local macOS VM access notes](docs/provisioning.md#local-macos-vm) for finding the SSH port or guest IP.
