# claude-box

`claude-box` runs [Claude Code](https://docs.anthropic.com/en/docs/claude-code) inside a rootless Podman container, isolated to the project directory from which it is launched.

It supports:

- Native Claude models
- SAIA models through an internal LiteLLM proxy
- A configurable Bifrost/OpenAI-compatible gateway through an internal LiteLLM proxy

## Why use claude-box?

Claude Code is an agent. It can read files, execute commands, install packages, modify code, and access the network. Running it directly on a workstation can expose more than the project you intended to share.

Potential risks include:

- A process running as your user can normally read many unrelated files.
- Files, dependencies, web content, and command output can contain prompt injections.
- Environment variables may contain API keys, cloud credentials, and tokens.
- SSH agents, GPG agents, and authenticated sockets can allow impersonation.
- An agent with network access could leak data it can read.
- Claude Code’s own permissions and sandbox features are not a complete security boundary.

`claude-box` reduces this exposure by putting Claude Code behind a rootless container boundary. By default, it shares only the current project and a dedicated per-project home, passes only selected environment variables, and blocks access to the host and private/local networks.

This makes permission-free agent operation more practical: Claude can work freely inside the project without receiving the same access as a normal process running directly on your workstation.

### Important limitation

`claude-box` reduces risk; it does not make untrusted code harmless.

Claude can read and modify the mounted project, access its persistent project-specific state, and communicate with permitted internet services. It could therefore leak information stored in those locations.

Keep secrets out of project directories. Carefully review extra mounts, environment variables, proxy settings, published ports, disabled firewall rules, and `ALLOW_PTRACE=1`.

## What it provides

- Claude Code in an Ubuntu 24.04 container
- Python 3.12, `uv`, Poetry, Git, and common command-line tools
- Access to the current project, but not the rest of the host filesystem
- Separate sessions, settings, and caches for each project and backend
- Internet access while blocking the host and private/local networks
- Optional SAIA and Bifrost support through LiteLLM
- Isolation of gateway keys from the Claude Code process
- An optional shell inside the same sandbox

Claude can create virtual environments and run commands such as:

```bash
pip install -e .
uv sync
poetry install
pytest
```

Changes to the project and its project-specific home persist. Other container changes disappear when the container exits.

---

## 1. Install Podman

### Linux

On Ubuntu or Debian:

```bash
sudo apt update
sudo apt install -y podman uidmap iproute2
```

On Fedora:

```bash
sudo dnf install -y podman
```

Verify the installation:

```bash
podman info
```

Run `claude-box` as your normal user, not with `sudo`.

### macOS

Install Podman and the `ip` compatibility command:

```bash
brew install podman iproute2mac
```

Create and start the Podman virtual machine:

```bash
podman machine init --cpus 4 --memory 8192 --disk-size 50
podman machine start
```

If a machine already exists:

```bash
podman machine start
```

Verify the installation:

```bash
podman info
ip -o addr show scope global
```

Internal gateways may require VPN routes and internal DNS to be available inside the Podman virtual machine.

---

## 2. Build the image

From the directory containing `Containerfile`:

```bash
podman build -t claude-box:latest -f Containerfile .
```

The first build may take several minutes because the image includes Python, Poetry, `uv`, and LiteLLM.

Confirm that the image exists:

```bash
podman image exists localhost/claude-box:latest
echo $?
```

A result of `0` means the image was found.

---

## 3. Install the launcher

Assuming the launcher is named `claude-box`:

```bash
sudo install -m 0755 claude-box /usr/local/bin/claude-box
```

Alternatively, install it for only your user:

```bash
mkdir -p ~/.local/bin
install -m 0755 claude-box ~/.local/bin/claude-box
```

Make sure `~/.local/bin` is in your `PATH`:

```bash
export PATH="$HOME/.local/bin:\$PATH"
```

Inspect the resolved setup:

```bash
claude-box --box-info
```

---

## 4. Configure native Claude authentication

Create a long-lived Claude setup token:

```bash
claude setup-token
```

If Claude Code is not installed on the host, run it from the image:

```bash
podman run --rm -it localhost/claude-box:latest setup-token
```

Create the configuration directory:

```bash
mkdir -p ~/.config/claude-box
chmod 700 ~/.config/claude-box
```

Save the token without putting it in shell history:

```bash
umask 077
cat > ~/.config/claude-box/token
```

Paste the token, press **Enter**, then press **Ctrl-D**:

```bash
chmod 600 ~/.config/claude-box/token
```

You may instead provide `CLAUDE_CODE_OAUTH_TOKEN` or `ANTHROPIC_API_KEY` through the environment.

---

## 5. Optional: configure SAIA

SAIA mode uses an internal LiteLLM proxy:

```text
Claude Code
    │ Anthropic Messages API and dummy token
    ▼
LiteLLM on container loopback
    │ OpenAI-compatible API and real key
    ▼
SAIA
```

### Store the SAIA key

```bash
mkdir -p ~/.config/claude-box
chmod 700 ~/.config/claude-box

umask 077
cat > ~/.config/claude-box/saia-key
```

Paste the key, press **Enter**, then press **Ctrl-D**:

```bash
chmod 600 ~/.config/claude-box/saia-key
```

An exported `SAIA_API_KEY` takes precedence over this file.

### List models

```bash
claude-box --saia-models
```

### Start a model

```bash
claude-box --saia MODEL_ID
```

To configure a default model, add it to:

```text
~/.config/claude-box/config
```

For example:

```bash
SAIA_MODEL='deepseek-v4-flash-0731'
```

Then run:

```bash
claude-box --saia
```

### SAIA context window

`claude-box` uses the following order to determine the context window:

1. An explicit `SAIA_CONTEXT_TOKENS` value
2. Metadata returned by SAIA’s `/models` endpoint
3. A conservative fallback of 32,768 tokens

Some SAIA models do not return context metadata. In that case, you will see:

```text
the gateway does not report a context window for MODEL; using the conservative 32768-token fallback
```

This is informational. The model can still run.

Using a value lower than the model’s real limit is safe, but Claude will compact conversations earlier. A value higher than the gateway’s actual limit can cause long requests to fail.

If you know the model’s effective context size, add it to `~/.config/claude-box/config`:

```bash
SAIA_MODEL='deepseek-v4-flash-0731'
SAIA_CONTEXT_TOKENS=65536
```

Replace `65536` with the value documented by the gateway administrator.

To change the general fallback instead:

```bash
SAIA_DEFAULT_CONTEXT_TOKENS=65536
```

The built-in fallback is:

```bash
SAIA_DEFAULT_CONTEXT_TOKENS=32768
```

An explicit `SAIA_CONTEXT_TOKENS` overrides both gateway metadata and the fallback.

---

## 6. Optional: configure Bifrost

Bifrost mode connects Claude Code to a configurable OpenAI-compatible gateway through LiteLLM:

```text
Claude Code
    │ Anthropic Messages API and dummy token
    ▼
LiteLLM on container loopback
    │ OpenAI-compatible API and real key
    ▼
Configured Bifrost gateway
```

No Bifrost hostname, endpoint, model, CA file, or quota endpoint is built into `claude-box`.

### Configure the gateway

Create or edit:

```text
~/.config/claude-box/config
```

For example:

```bash
mkdir -p ~/.config/claude-box
chmod 700 ~/.config/claude-box
nano ~/.config/claude-box/config
```

At minimum, add the gateway URL:

```bash
BIFROST_URL='https://gateway.example.org/v1'
```

Optionally add a default model:

```bash
BIFROST_MODEL='model-id'
```

Protect the configuration:

```bash
chmod 600 ~/.config/claude-box/config
```

The configuration is sourced as a shell fragment. Do not put untrusted shell code in it.

### Store the Bifrost key

```bash
umask 077
cat > ~/.config/claude-box/bifrost-key
```

Paste the key, press **Enter**, then press **Ctrl-D**:

```bash
chmod 600 ~/.config/claude-box/bifrost-key
```

An exported `BIFROST_KEY` takes precedence over this file.

### List models

```bash
claude-box --bifrost-models
```

By default, the model endpoint is:

```text
<BIFROST_URL>/models
```

If the deployment uses another endpoint, add it to `~/.config/claude-box/config`:

```bash
BIFROST_MODELS_URL='https://gateway.example.org/custom/models'
```

The model list may include embedding, image, or other non-chat models. Claude Code requires a conversational model with chat completion and tool-calling support.

### Start a model

Specify a model:

```bash
claude-box --bifrost MODEL_ID
```

Or configure a default:

```bash
BIFROST_MODEL='MODEL_ID'
```

Then run:

```bash
claude-box --bifrost
```

### Internal CA certificates

If the gateway’s TLS certificate is signed by an internal organizational CA, add the path to its PEM CA certificate or CA bundle to the **claude-box configuration file**:

```text
~/.config/claude-box/config
```

For example:

```bash
BIFROST_URL='https://gateway.example.org/v1'
BIFROST_MODEL='model-id'
BIFROST_CA_FILE='/path/to/organization-ca-bundle.pem'
```

`BIFROST_CA_FILE` must point to a readable certificate file on the host.

`claude-box` mounts the file read-only into the container and uses it for model-list requests and for the LiteLLM proxy’s TLS verification.

Check that the file is readable:

```bash
test -r /path/to/organization-ca-bundle.pem \
    && echo "CA file is readable"
```

If the gateway certificate is already trusted by the container’s standard CA bundle, omit `BIFROST_CA_FILE`.

TLS verification remains enabled. `claude-box` does not use `curl -k`.

### Bifrost context window

`claude-box` uses the following order:

1. An explicit `BIFROST_CONTEXT_TOKENS` value
2. Metadata returned by the model-list endpoint
3. A conservative fallback of 32,768 tokens

If the gateway does not return context metadata, the 32,768-token fallback is used automatically.

To configure the selected model’s known limit, add this to `~/.config/claude-box/config`:

```bash
BIFROST_CONTEXT_TOKENS=65536
```

Replace `65536` with the gateway’s effective limit.

To change the general fallback:

```bash
BIFROST_DEFAULT_CONTEXT_TOKENS=65536
```

The built-in fallback is:

```bash
BIFROST_DEFAULT_CONTEXT_TOKENS=32768
```

The maximum requested output defaults to 8,192 tokens. To change it:

```bash
BIFROST_MAX_OUTPUT_TOKENS=16000
```

### Optional quota endpoint

Quota APIs are deployment-specific, so there is no default quota URL.

If the gateway provides one, add it to `~/.config/claude-box/config`:

```bash
BIFROST_QUOTA_URL='https://gateway.example.org/api/quota'
BIFROST_QUOTA_HEADER='x-bf-vk'
```

Query it with:

```bash
claude-box --bifrost-quota
```

### Complete Bifrost example

Example `~/.config/claude-box/config`:

```bash
# Required Bifrost endpoint
BIFROST_URL='https://gateway.example.org/v1'

# Default model
BIFROST_MODEL='model-id'

# Optional internal CA file on the host
BIFROST_CA_FILE='/path/to/organization-ca-bundle.pem'

# Optional explicit context limit for the selected model
# BIFROST_CONTEXT_TOKENS=65536

# Fallback when the gateway reports no context metadata
BIFROST_DEFAULT_CONTEXT_TOKENS=32768

# Maximum requested output
BIFROST_MAX_OUTPUT_TOKENS=8192

# Optional endpoint overrides
# BIFROST_MODELS_URL='https://gateway.example.org/v1/models'
# BIFROST_QUOTA_URL='https://gateway.example.org/api/quota'
# BIFROST_QUOTA_HEADER='x-bf-vk'
```

### Gateway key isolation

The real SAIA or Bifrost key is held by a separate LiteLLM process running under a dedicated UID. Claude Code receives only a dummy token.

When the firewall is enabled, the proxy UID can contact the configured gateway but not arbitrary destinations.

Do not enable `ALLOW_PTRACE=1` in gateway mode unless required. It may allow the agent to inspect the proxy process and recover the key.

---

## 7. Use claude-box

Start native Claude from a project:

```bash
cd ~/projects/my-project
claude-box
```

Pass arguments to Claude Code:

```bash
claude-box -c
claude-box -p "Explain this project"
```

Use SAIA:

```bash
claude-box --saia MODEL_ID
claude-box --saia MODEL_ID -c
```

Use Bifrost:

```bash
claude-box --bifrost MODEL_ID
claude-box --bifrost MODEL_ID -c
```

Useful commands:

```bash
claude-box --box-info
claude-box --box-shell
claude-box --saia-models
claude-box --bifrost-models
claude-box --bifrost-quota
```

Wrapper options must appear before Claude Code options.

Native Claude, SAIA, and Bifrost sessions use separate project homes, so `-c` does not resume a session created with another backend.

---

## How isolation works

By default, the container can access only:

- The current project directory, read-write
- A project-specific home mounted at `/claude`, read-write
- `/etc/localtime`, read-only
- Explicit shared items and extra mounts
- A configured Bifrost CA file, read-only

The rest of the host filesystem is not mounted.

### Temporary container root

When the firewall is enabled, or a gateway is used, the container starts as root **inside rootless Podman’s user namespace**.

This limited container-root identity:

1. Installs the nftables firewall.
2. Starts the isolated LiteLLM proxy when needed.
3. Switches to your normal UID and GID.
4. Removes capabilities.
5. Starts Claude Code or the shell.

This is not host root.

### Installing dependencies

Claude can install dependencies into user-writable locations:

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install -e .
```

It can also use:

```bash
uv sync
poetry install
```

Claude cannot normally use `apt install` after privileges have been dropped.

### Persistence

The container is removed when Claude exits.

These locations persist:

- The current project
- Claude sessions and settings
- Project-specific caches
- Virtual environments in the project or `/claude`

System-level container changes do not persist.

### Network restrictions

By default, the container can reach the public internet but cannot connect to:

- The host
- Private IPv4 ranges
- Carrier-grade NAT ranges
- Link-local networks
- Private IPv6 networks
- Multicast ranges

In SAIA or Bifrost mode, the proxy UID is limited to the configured gateway and explicit exceptions.

---

## Configuration reference

The main configuration file is:

```text
~/.config/claude-box/config
```

Common settings:

```bash
# General
FIREWALL=1
READ_ONLY=0
MEMORY_LIMIT='8g'
TMP_SIZE='2g'
PERMISSION_MODE='bypassPermissions'

# Image
CLAUDE_BOX_IMAGE='localhost/claude-box:latest'

# SAIA
SAIA_MODEL='model-id'

# Optional explicit model context:
# SAIA_CONTEXT_TOKENS=65536

# Fallback when metadata is unavailable:
SAIA_DEFAULT_CONTEXT_TOKENS=32768

SAIA_MAX_OUTPUT_TOKENS=8192

# Bifrost
BIFROST_URL='https://gateway.example.org/v1'
BIFROST_MODEL='model-id'
BIFROST_CA_FILE='/path/to/organization-ca-bundle.pem'

# Optional explicit model context:
# BIFROST_CONTEXT_TOKENS=65536

# Fallback when metadata is unavailable:
BIFROST_DEFAULT_CONTEXT_TOKENS=32768

BIFROST_MAX_OUTPUT_TOKENS=8192
```

Only configure options you need.

---

## Common issues

### Image not found

```bash
podman build -t claude-box:latest -f Containerfile .
```

### Podman is not running on macOS

```bash
podman machine start
podman info
```

### Claude asks you to log in repeatedly

Check the token:

```bash
ls -l ~/.config/claude-box/token
chmod 600 ~/.config/claude-box/token
```

### SAIA returns HTTP 401

An old exported `SAIA_API_KEY` may override the key file:

```bash
env -u SAIA_API_KEY claude-box --saia-models
```

If that works:

```bash
unset SAIA_API_KEY
```

Otherwise, update `~/.config/claude-box/saia-key`.

### Bifrost URL is not configured

Add it to `~/.config/claude-box/config`:

```bash
BIFROST_URL='https://gateway.example.org/v1'
```

There is intentionally no built-in Bifrost URL.

### Bifrost TLS verification fails

Add the host CA file path to `~/.config/claude-box/config`:

```bash
BIFROST_CA_FILE='/path/to/organization-ca-bundle.pem'
```

Do not disable TLS verification.

### The gateway does not report a context size

This is supported. `claude-box` automatically uses 32,768 tokens.

To use a known value, add one of these to `~/.config/claude-box/config`:

```bash
SAIA_CONTEXT_TOKENS=65536
```

or:

```bash
BIFROST_CONTEXT_TOKENS=65536
```

Replace the value with the selected model’s documented effective limit.

### Claude Code says the model is unknown

Claude Code may warn that a third-party model is not in its internal catalog. This does not necessarily prevent the model from working.

`claude-box` supplies an explicit context limit from gateway metadata or the 32,768-token fallback. The catalog warning may still appear, but Claude Code should use the supplied limit.

### Internal gateway cannot be reached on macOS

The Podman virtual machine may not inherit internal VPN routes or DNS. Test connectivity inside it:

```bash
podman machine ssh
```

Check the organization’s VPN and Podman networking configuration.

### Inspect the effective setup

```bash
claude-box --box-info
```

This displays the image, project mount, backend, authentication source, context policy, network, and firewall status without printing secret values.
