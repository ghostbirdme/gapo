# Client Usage

This guide covers how to install and use the `gapo` client to expose local services through a tunnel.

## Installation

Download the latest binary for your platform from the [releases page](https://github.com/ghostbirdme/gapo/releases), then extract it:

**Linux / macOS:**
```bash
tar xzf gapo_*_linux_amd64.tar.gz
sudo mv gapo /usr/local/bin/
```

**Windows:**

Extract the zip file and add `gapo.exe` to your PATH.

Verify the installation:

```bash
gapo --version
```

## Quick Start

Start a local web server and expose it through the tunnel:

```bash
# Start your local service (example)
python3 -m http.server 4000

# Expose it as myapp.<domain>
gapo --server tunnel.example.com:19443 --token your-token myapp 4000
```

Your service is now available at `https://myapp.tunnel.example.com`.

## Usage

```
gapo [flags] <subdomain> <local-port>
```

**Arguments:**

| Argument | Description |
|----------|-------------|
| `subdomain` | Subdomain name for your tunnel (e.g., `myapp`) |
| `local-port` | Port of the local service to expose (1-65535) |

**Flags:**

| Flag | Short | Description | Default |
|------|-------|-------------|---------|
| `--server` | `-s` | Server tunnel address | `localhost:19443` |
| `--token` | `-t` | Auth token (required) | — |
| `--tcp` | | TCP tunnel mode (for SSH, databases, etc.) | `false` |
| `--tls` | | Connect using TLS | `false` |
| `--insecure` | `-k` | Skip TLS certificate verification | `false` |
| `--no-tui` | `-n` | Disable TUI, use plain log output | `false` |
| `--version` | `-v` | Show version and exit | — |

## Configuration

Instead of passing flags every time, you can save settings in a config file at `~/.gapo/config`:

```bash
mkdir -p ~/.gapo

cat > ~/.gapo/config << 'EOF'
GAPO_SERVER=tunnel.example.com:19443
GAPO_TOKEN=your-secret-token
GAPO_TLS=true
EOF
```

Available config keys:

| Key | Description |
|-----|-------------|
| `GAPO_SERVER` | Server tunnel address |
| `GAPO_TOKEN` | Auth token |
| `GAPO_TLS` | Connect using TLS (`true`, `yes`, or `1`) |
| `GAPO_INSECURE` | Skip TLS verification (`true`, `yes`, or `1`) |

CLI flags override config file values. With a config file, you only need:

```bash
gapo myapp 4000
```

## TUI Dashboard

By default, gapo starts with an interactive TUI dashboard that shows:
- Session info (connection status, server address)
- Active connections
- Request log (method, path, status, timing)

Press `Ctrl+C` to disconnect.

## Log Mode

For scripts, CI, or headless environments, use `--no-tui` for plain log output:

```bash
gapo --no-tui myapp 4000
```

## TCP Tunnel Mode

Use `--tcp` to expose any TCP service — SSH, databases, VNC, and more:

```bash
# Expose local SSH
gapo --tcp ssh 22

# Expose local PostgreSQL
gapo --tcp postgres 5432

# Expose local VNC
gapo --tcp vnc 5900
```

The server allocates a public port (e.g., 30001) and displays it. Remote users connect to `server:30001` to reach your local service.

The server must be configured with `--tcp-ports` to enable TCP tunneling.

## Examples

**Expose a local web app:**
```bash
gapo myapp 3000
```

**Expose with TLS (server uses self-signed cert):**
```bash
gapo --tls -k myapp 3000
```

**Expose with TLS (server uses valid cert):**
```bash
gapo --tls myapp 3000
```

**Run in background (log mode):**
```bash
gapo -n myapp 3000 &
```

**Multiple tunnels (different subdomains, different terminals):**
```bash
# Terminal 1
gapo frontend 3000

# Terminal 2
gapo api 8080
```

## Subdomain Rules

- Lowercase letters, numbers, and hyphens only
- Must start and end with a letter or number
- Maximum 63 characters
- Examples: `myapp`, `my-app`, `staging-v2`

## Troubleshooting

**"connection refused"**
- Check that the server is running and reachable
- Verify the `--server` address and port

**"authentication failed"**
- Check that `--token` matches the server's token

**"subdomain already in use"**
- Another client is using that subdomain — choose a different name

**TLS errors**
- For self-signed server certs, use `--insecure`
- For production servers with valid certs, use `--tls` without `--insecure`
