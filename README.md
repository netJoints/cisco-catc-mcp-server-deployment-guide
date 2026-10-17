# Cisco Catalyst Center MCP Server on Podman Desktop

Companion [project case study](https://netjoints.github.io/cisco-catc-mcp-server-deployment-guide/) and source: [index.html](index.html).

## Purpose

This guide installs Cisco's [Catalyst Center MCP server](https://github.com/cisco-en-programmability/catc-mcp-oss) on macOS with Podman Desktop. It is written for Apple Silicon Macs and uses the server's Streamable HTTP endpoint. The same container commands work on Intel Macs, though Podman Desktop's provider and binary path can differ.

The server exposes generated Catalyst Center API tools. Treat every tool as a network operation: start with read-only tools and use an account with only the permissions your intended tasks need.

## Tested setup and compatibility

- Host: macOS on Apple Silicon, Podman Desktop with Podman CLI 6.1.4, `libkrun` machine provider.
- Server source: Cisco `release/3.2.3` branch.
- Catalyst Center lab controller: 3.2.2. This branch was selected because no exact `release/3.2.2` branch was available during setup. A successful basic inventory read is a lab result, not a Cisco compatibility guarantee. Recheck Cisco's branches and release notes for your controller before deployment.
- The successful lab machine had 6 CPUs and 2 GiB RAM. Container image build and a basic inventory read worked with those settings; larger tool calls may need more resources.

Do not infer compatibility from the branch name alone. Check the controller's exact version, the branch's tool bundle and Cisco's release notes before using the server against production.

## Before you start

You need:

1. Podman Desktop and its `podman` CLI installed on the Mac.
2. Access to a Catalyst Center reachable from the Mac, plus a dedicated account authorized for the APIs you plan to call.
3. Git and network access to GitHub, Quay.io, and PyPI (the first build downloads a base image and Python packages).
4. A terminal using `zsh` (the commands below use zsh secret prompts).

The guide uses a local clone of Cisco's source. Commands run from that checkout unless stated otherwise.

## 1. Install and verify Podman

Install Podman Desktop through your organization's approved software catalog or [Podman's official installation instructions](https://podman-desktop.io/docs/installation). Open Podman Desktop once so it can finish setup.

In Terminal, check whether the CLI is on your PATH:

```zsh
command -v podman
podman --version
```

If `zsh` says `command not found` and your installation uses `/opt/podman/bin`, add it for this Terminal session:

```zsh
export PATH="/opt/podman/bin:$PATH"
podman --version
```

To keep that setting in future Terminal windows, add the same `export PATH=...` line to `~/.zshrc`, then open a new Terminal window. If Podman Desktop installed the CLI elsewhere, use its displayed CLI path instead of `/opt/podman/bin`.

## 2. Start the Podman Linux machine

macOS runs Linux containers inside a Podman virtual machine. First inspect existing machines:

```zsh
podman machine list
```

If a machine is already listed, start it and skip initialization. Do not run `machine init` again just because `podman info` cannot connect; first start the existing machine:

```zsh
podman machine start
```

If there is no machine, create and start one:

```zsh
podman machine init --cpus 4 --memory 4096 --disk-size 40 --now
```

The memory value is MiB (4096 = 4 GiB); disk size is GiB. Initial creation downloads a large Linux image and may take several minutes. Let that command finish before retrying it.

Verify the machine and connection:

```zsh
podman machine list
podman info
```

`podman info` should show a Linux host and an existing remote socket. Confirm the effective CPU and memory values in `podman machine list`; do not assume requested values took effect. Podman's current `machine set` resource changes are supported only for QEMU machines. With `libkrun`, use Podman Desktop's machine settings if available, or recreate the machine only after confirming it contains no data you need to preserve. See [Podman machine init](https://docs.podman.io/en/latest/markdown/podman-machine-init.1.html) and [machine set](https://docs.podman.io/en/latest/markdown/podman-machine-set.1.html).

## 3. Select the Cisco server branch

Choose the server branch that Cisco documents for your Catalyst Center release. Review the [Cisco repository branches](https://github.com/cisco-en-programmability/catc-mcp-oss/branches) and that branch's README/release notes.

For the lab setup recorded above, the closest available candidate was `release/3.2.3`:

```zsh
git clone --branch release/3.2.3 --single-branch \
  https://github.com/cisco-en-programmability/catc-mcp-oss.git
cd catc-mcp-oss
```

If you already cloned the repository, inspect its branch and working tree before switching:

```zsh
git status --short --branch
git branch -a
```

Switch only after saving any local changes. Do not use `git reset --hard` to resolve branch differences. If Cisco publishes an exact branch for your controller, prefer it over the lab's 3.2.3 candidate.

## 4. Build the image

From the root of `catc-mcp-oss`, build with Podman. This uses the repository's Dockerfile; Podman does not require Docker Desktop:

```zsh
podman build -t localhost/catalyst-center-mcp:3.2.3 .
```

The first build can take a while while Podman downloads `python:3.14-slim` and installs dependencies. Watch the output until the command completes, then confirm the local image exists:

```zsh
podman images localhost/catalyst-center-mcp
```

## 5. Configure credentials and start the container

Use a dedicated Catalyst Center account. These prompts keep the password out of shell command history. Replace the sample TLS value with `true` whenever the controller certificate chains to a CA trusted by the container.

```zsh
read "CATALYST_CENTER_HOST?Catalyst Center hostname or IP: "
read "CATALYST_CENTER_USERNAME?Username: "
read -s "CATALYST_CENTER_PASSWORD?Password: "; print
export CATALYST_CENTER_HOST CATALYST_CENTER_USERNAME CATALYST_CENTER_PASSWORD
```

Start the server, binding the port to this Mac only:

```zsh
podman run --detach \
  --name catalyst-center-mcp \
  --restart unless-stopped \
  --publish 127.0.0.1:7001:7001 \
  --env CATALYST_CENTER_HOST \
  --env CATALYST_CENTER_USERNAME \
  --env CATALYST_CENTER_PASSWORD \
  --env CATALYST_CENTER_VERIFY_SSL=true \
  localhost/catalyst-center-mcp:3.2.3
unset CATALYST_CENTER_PASSWORD
```

If the server logs show a certificate verification error for a lab controller with a self-signed or untrusted certificate, the preferred fix is to configure a trusted certificate chain. For an isolated lab only, you can recreate the container with `CATALYST_CENTER_VERIFY_SSL=false`; this disables TLS certificate verification and makes a man-in-the-middle attack easier. It does not encrypt or replace the HTTPS connection. Never use that setting as a general production recommendation.

To recreate it for an isolated lab after the verified-TLS attempt fails, repeat the credential prompts above (do not paste a password into a command), then run:

```zsh
podman rm --force catalyst-center-mcp
podman run --detach \
  --name catalyst-center-mcp \
  --restart unless-stopped \
  --publish 127.0.0.1:7001:7001 \
  --env CATALYST_CENTER_HOST \
  --env CATALYST_CENTER_USERNAME \
  --env CATALYST_CENTER_PASSWORD \
  --env CATALYST_CENTER_VERIFY_SSL=false \
  localhost/catalyst-center-mcp:3.2.3
unset CATALYST_CENTER_PASSWORD
```

Podman stores the configured environment in container metadata, which a local user with Podman access can inspect. Use a dedicated account, protect the Mac login, and remove the container when you no longer need the integration.

## 6. Verify the service

Check that the container is running and inspect its recent logs:

```zsh
podman ps --filter name=catalyst-center-mcp
podman logs --tail 100 catalyst-center-mcp
```

Check the local endpoints:

```zsh
curl --fail http://127.0.0.1:7001/v1/health
curl --fail http://127.0.0.1:7001/v1/readiness
```

Health returns `{"status":"ok"}`. Readiness returns `{"status":"ready"}` when the required environment variables are present. Readiness does not prove that the controller is reachable or that login succeeds; verify that by making a harmless read-only MCP call from your MCP client.

## 7. Connect an MCP client

Configure your MCP client to use **Streamable HTTP** at:

```text
http://127.0.0.1:7001/v1/mcp
```

Keep the URL on `127.0.0.1` for laptop-only access. The port is not published on other network interfaces. Client configuration formats differ; use the client's current instructions for adding a remote HTTP MCP server. After connecting, make a simple read-only inventory query and verify the result against Catalyst Center.

## Daily operations

- View status: `podman ps --filter name=catalyst-center-mcp`
- Follow logs: `podman logs --follow catalyst-center-mcp`
- Stop/start: `podman stop catalyst-center-mcp` / `podman start catalyst-center-mcp`
- Remove the container: `podman rm --force catalyst-center-mcp`
- Remove the local image when no longer needed: `podman image rm localhost/catalyst-center-mcp:3.2.3`
- Upgrade: pull the selected Cisco branch changes, rebuild the image, then recreate the container with the same run options and credentials.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| `zsh: command not found: podman` | Add the Podman CLI directory to `PATH` for this session; on some Podman Desktop installs it is `/opt/podman/bin`. |
| `Cannot connect to Podman` or missing `podman.sock` | Run `podman machine list`; start the listed VM with `podman machine start`. Initialize a VM only when the list is empty. |
| `podman machine init` appears stuck | First boot downloads a large VM image. Let it complete and check progress before starting another initialization. |
| Requested CPU/RAM differs from `podman machine list` | Verify the active provider and effective settings. `podman machine set` only changes CPU, memory, and disk for QEMU machines; libkrun does not support those changes through that command. |
| Container exits immediately | Read `podman logs catalyst-center-mcp`; check controller host, username, password, network reachability, and TLS trust. Do not paste credentials into an issue or chat. |
| Health works but Catalyst Center calls fail | Health only confirms the web process is alive. Check logs, controller reachability, account authorization, and certificate validation; test with a read-only MCP tool. |
| Port 7001 is already in use | Stop the other local service or publish another host port, for example `127.0.0.1:7002:7001`, and use port 7002 in the client URL. |

## Security notes

- Use a dedicated, least-privilege Catalyst Center account. The generated catalog contains operations that can change configuration; grant permissions based on intended use and review each tool call.
- Do not commit passwords, `.env` files, tokens, controller hostnames/IPs, or returned device data to a public repository.
- Keep the port bound to loopback (`127.0.0.1`). Do not change it to `0.0.0.0` unless you intentionally secure remote access.
- Prefer verified TLS. Disabling certificate validation is an insecure lab workaround only.
- The container is a local convenience, not a production deployment or security boundary.

## References

- [Cisco Catalyst Center MCP server source](https://github.com/cisco-en-programmability/catc-mcp-oss)
- [Podman Desktop installation](https://podman-desktop.io/docs/installation)
- [Podman machine init reference](https://docs.podman.io/en/latest/markdown/podman-machine-init.1.html)
- [Podman machine set reference](https://docs.podman.io/en/latest/markdown/podman-machine-set.1.html)

## Publish the GitHub Pages site

This repository includes a self-contained static site in `index.html`. After pushing it to `main`, open **Settings → Pages** and choose **Deploy from a branch**, then select `main` and `/(root)`. GitHub Pages uses `index.html` as the entry page. The expected site address is `https://netjoints.github.io/cisco-catc-mcp-server-deployment-guide/`; confirm the final URL under **Settings → Pages** after deployment completes. Everything in the published site is public, so review it for confidential material before publishing. See [GitHub's Pages setup documentation](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site).
