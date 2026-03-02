# Client Usage

Complete guide for using the `gapo` client to expose local services through a tunnel.

## Requirements

| Resource | Minimum |
|----------|---------|
| CPU | Any modern CPU |
| RAM | ~20 MB |
| Disk | ~10 MB |
| OS | Linux, macOS, Windows, or FreeBSD (amd64 or arm64) |

Single static binary, no runtime dependencies. Runs on anything from a Raspberry Pi to a laptop.

## Installation

Download the latest binary from the [releases page](https://github.com/ghostbirdme/gapo/releases).

**Linux / macOS / FreeBSD:**

```bash
tar xzf gapo_*_linux_amd64.tar.gz
sudo mv gapo /usr/local/bin/
gapo --version
```

**Windows:**

Extract the zip file and add `gapo.exe` to your PATH.

## Quick Start

Start a local web server and expose it:

```bash
# Start your local service
python3 -m http.server 3000

# Expose it through the tunnel
gapo --server tunnel.example.com:19443 --token your-token --http myapp 3000
```

Your service is now live at `https://myapp.tunnel.example.com`.

## Command Syntax

```
gapo [flags] <name> <local-port>

gapo --http myapp 3000      # HTTP tunnel
gapo --tcp ssh 22           # TCP tunnel
```

| Argument | Description | Example |
|----------|-------------|---------|
| `name` | A name for your tunnel — used as the subdomain for HTTP tunnels | `myapp` |
| `local-port` | Port of the local service to expose (1-65535) | `3000` |

The name can be anything you choose (lowercase letters, numbers, hyphens). For HTTP tunnels it becomes the subdomain (`myapp.tunnel.example.com`). For TCP tunnels it's just an identifier.

## CLI Flags

| Flag | Short | Description | Default |
|------|-------|-------------|---------|
| `--server` | `-s` | Server address | `localhost:19443` |
| `--token` | `-t` | Authentication token (required) | — |
| `--tcp` | | TCP tunnel mode (for SSH, databases, etc.) | `false` |
| `--http` | | HTTP tunnel mode (default) | `true` |
| `--tls` | | Encrypt tunnel connection (only when server uses `--tls`) | `false` |
| `--insecure` | `-k` | Allow self-signed server certificates | `false` |
| `--no-tui` | `-n` | Plain log output instead of TUI dashboard | `false` |
| `--update` | | Check for updates and self-update | — |
| `--version` | `-v` | Show version and exit | — |
| `--about` | | Show license and copyright info | — |

## Config File

Save settings to `~/.gapo/config` so you don't have to repeat them:

```bash
mkdir -p ~/.gapo
cat > ~/.gapo/config << 'EOF'
GAPO_SERVER=tunnel.example.com:19443
GAPO_TOKEN=your-secret-token
EOF
```

| Key | Maps to |
|-----|---------|
| `GAPO_SERVER` | `--server` |
| `GAPO_TOKEN` | `--token` |
| `GAPO_TLS` | `--tls` |
| `GAPO_INSECURE` | `--insecure` |

CLI flags override config file values. With a config file, you only need:

```bash
gapo --http myapp 3000
```

See [Configuration](CONFIGURATION.md) for file format details.

---

## HTTP Tunnels

Expose any web application, API, or HTTP service:

```bash
gapo --http myapp 3000
```

Your service is available at `https://myapp.tunnel.example.com` (or `http://` depending on server setup). WebSocket connections are supported automatically.

> **Important:** The local port must be an **HTTP** port, not an HTTPS port. Gapo handles HTTPS on the public side and forwards plain HTTP to your local service. For example, if your service runs on port 80 (HTTP) and 443 (HTTPS), use port 80.

**Multiple tunnels** — run each in a separate terminal:

```bash
gapo --http frontend 3000
gapo --http api 8080
gapo --http docs 4000
```

Each gets its own subdomain. No conflicts as long as the names are different.

## TCP Tunnels

Use `--tcp` to expose any TCP service — SSH, databases, VNC, and more:

```bash
gapo --tcp ssh 22           # SSH
gapo --tcp postgres 5432    # PostgreSQL
gapo --tcp mysql 3306       # MySQL
gapo --tcp vnc 5900         # VNC
```

The server allocates a public port (e.g., 30000) and displays it. Remote users connect to `server:30000` to reach your local service.

> The name (`ssh`, `postgres`, etc.) is just a label you choose — it can be anything. The server must be configured with `--tcp-ports` to enable TCP tunneling.

---

## TUI Dashboard

By default, gapo starts with an interactive dashboard showing:

**HTTP mode:**
- **Session** — connection status, server address, tunnel URL
- **Connections** — active streams between server and your local service
- **Requests** — live log of HTTP requests (method, path, status, timing)

**TCP mode:**
- **Session** — connection status, mode, remote port
- **TCP Connections** — active/total connections with duration and bytes transferred

**Keyboard shortcuts:**
| Key | Action |
|-----|--------|
| `↑` / `↓` or `k` / `j` | Navigate the request or connection list |
| `Enter` | Inspect the selected request or TCP connection |
| `Esc` | Return to the list from the inspect view |
| `Home` / `End` or `g` / `G` | Jump to top or bottom of the list |
| `Ctrl+C` or `q` | Quit |

**HTTP inspect view** shows request/response headers, body preview, status, timing, and remote address.

**TCP inspect view** shows remote address, connection status, open/close times, duration, and data transfer (received, sent, total). For active connections, byte counters update in real time.

## Log Mode

For scripts, CI, or headless environments, use `--no-tui` for plain log output:

```bash
gapo --http -n myapp 3000
```

Run in background:

```bash
gapo --http -n myapp 3000 &
```

## Automatic Reconnection

If the connection to the server drops (network interruption, server restart), the client automatically reconnects. No manual intervention needed — just leave it running.

## Updating

Check for updates and update the binary in-place:

```bash
gapo --update
```

You'll be prompted before downloading. The download supports resume on network failure with automatic retries.

---

## Examples

**Web app (production server with `--autocert`):**

```bash
gapo --http myapp 3000
```

**Web app (server uses `--tls` for tunnel listener):**

```bash
gapo --http --tls -k myapp 3000
```

**SSH access through a TCP tunnel:**

```bash
gapo --tcp ssh 22
# Others connect: ssh user@tunnel.example.com -p 30000
```

**Database through a TCP tunnel:**

```bash
gapo --tcp postgres 5432
# Others connect: psql -h tunnel.example.com -p 30000 -U myuser mydb
```

**Multiple services at once (separate terminals):**

```bash
gapo --http frontend 3000  # https://frontend.tunnel.example.com
gapo --http api 8080       # https://api.tunnel.example.com
gapo --tcp ssh 22          # tcp://tunnel.example.com:30000
```

---

## Subdomain Rules

- Lowercase letters, numbers, and hyphens only
- Must start and end with a letter or number
- Maximum 63 characters
- Examples: `myapp`, `my-app`, `staging-v2`

## Troubleshooting

**"connection refused"**
- Check that the server is running and reachable
- Verify the `--server` address and port
- Make sure the tunnel port (default 19443) is open in the server's firewall

**"authentication failed"**
- Check that `--token` matches the server's token exactly
- If you see "too many failed attempts", wait 1 minute — the server rate-limits after 5 failed logins from the same IP

**"subdomain already in use"**
- Another client is using that name — choose a different one
- If your previous session didn't disconnect cleanly, wait ~30 seconds for the server to detect it

**TLS errors**
- For self-signed server certs (development), use `--tls --insecure`
- For production servers with valid certs, use `--tls` only
- Make sure the server's domain matches the certificate

**"Bad Request — You're speaking plain HTTP to an SSL-enabled server port"**
- You are pointing the tunnel at an HTTPS port (e.g., 443). Use the HTTP port instead (e.g., 80 or 8080)
- Gapo forwards plain HTTP to your local service. HTTPS is handled by the server on the public side

**Tunnel connects but service is unreachable**
- Verify your local service is running on the specified port
- Check that the service is listening on `localhost` (not just a specific interface)
- Try `curl http://localhost:<port>` locally to confirm

---

**Last Updated:** 2026-03-03 12:00:00 UTC
