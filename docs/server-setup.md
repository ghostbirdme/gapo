# Server Setup

This guide covers how to install and configure `gapo-server` on your public-facing machine.

## Requirements

- A server with a public IP address
- A domain name with wildcard DNS (for subdomain routing)
- Ports 80/443 (production) or a custom port (development)

## Installation

Download the latest binary for your platform from the [releases page](https://github.com/ghostbirdme/gapo/releases), then extract it:

```bash
tar xzf gapo_*_linux_amd64.tar.gz
sudo mv gapo-server /usr/local/bin/
```

Verify the installation:

```bash
gapo-server --version
```

## DNS Setup

Gapo routes traffic based on the `Host` header. You need a wildcard DNS record pointing to your server.

For example, if your domain is `tunnel.example.com`:

```
A       tunnel.example.com      → <your-server-ip>
A       *.tunnel.example.com    → <your-server-ip>
```

This allows subdomains like `myapp.tunnel.example.com` to reach the server.

## Quick Start (Development)

For local testing, no DNS or TLS is needed:

```bash
gapo-server --token secret --domain localhost --http :8443 --tunnel :19443
```

This starts:
- HTTP proxy on port 8443
- Tunnel listener on port 19443

## Configuration

Settings can be provided via CLI flags, or a config file.

### CLI Flags

| Flag | Description | Default |
|------|-------------|---------|
| `--http` | HTTP listen address | `:8443` |
| `--tunnel` | Tunnel listen address | `:19443` |
| `--domain` | Base domain for subdomains | `localhost` |
| `--token` | Auth token (required) | — |
| `--tcp-ports` | TCP tunnel port range (e.g. `30000-39999`) | — |
| `--tls` | Enable TLS on tunnel listener (self-signed) | `false` |
| `--autocert` | Enable Let's Encrypt HTTPS | `false` |
| `--cert-dir` | Directory for autocert certificate cache | `/var/lib/gapo/certs` |
| `--email` | Email for Let's Encrypt registration | — |
| `--version` | Show version and exit | — |

### Config File

The server reads config from `/etc/gapo/config` first, then falls back to `~/.gapo/server-config`. The file uses `KEY=VALUE` format:

```bash
# /etc/gapo/config
GAPO_HTTP=:443
GAPO_TUNNEL=:19443
GAPO_DOMAIN=tunnel.example.com
GAPO_TOKEN=your-secret-token
GAPO_TCP_PORTS=30000-39999
GAPO_AUTOCERT=true
GAPO_CERT_DIR=/var/lib/gapo/certs
GAPO_EMAIL=admin@example.com
```

CLI flags override config file values.

## Production Setup (HTTPS with Let's Encrypt)

For production, use `--autocert` to get automatic HTTPS certificates from Let's Encrypt:

```bash
gapo-server \
  --domain tunnel.example.com \
  --token your-secret-token \
  --autocert \
  --cert-dir /var/lib/gapo/certs \
  --email admin@example.com \
  --http :443 \
  --tunnel :19443
```

This starts:
- HTTPS proxy on port 443 with automatic Let's Encrypt certificates
- HTTP server on port 80 for ACME challenges and HTTPS redirect
- Tunnel listener on port 19443

Certificates are cached in `--cert-dir` and renewed automatically. Certificates are only issued for subdomains with active tunnels.

### Prepare the certificate directory

```bash
sudo mkdir -p /var/lib/gapo/certs
sudo chown gapo:gapo /var/lib/gapo/certs
```

### Firewall

Ensure these ports are open:

| Port | Purpose |
|------|---------|
| 80 | ACME challenges + HTTPS redirect |
| 443 | HTTPS proxy (serves tunneled traffic) |
| 19443 | Tunnel connections from clients |
| 30000-39999 | TCP tunnel ports (if `--tcp-ports` is configured) |

## Development Setup (Self-Signed TLS)

For development with TLS on the tunnel connection:

```bash
gapo-server \
  --domain localhost \
  --token secret \
  --tls \
  --http :8443 \
  --tunnel :19443
```

The `--tls` flag generates an in-memory self-signed certificate for the tunnel listener. Clients must use `--insecure` to skip certificate verification.

## Running as a Systemd Service

Create `/etc/systemd/system/gapo-server.service`:

```ini
[Unit]
Description=Gapo Tunnel Server
After=network.target

[Service]
Type=simple
User=gapo
Group=gapo
ExecStart=/usr/local/bin/gapo-server
Restart=on-failure
RestartSec=5

# Allow binding to privileged ports
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

Set up the user and config:

```bash
# Create system user
sudo useradd --system --no-create-home gapo

# Create config
sudo mkdir -p /etc/gapo
sudo nano /etc/gapo/config

# Create cert directory
sudo mkdir -p /var/lib/gapo/certs
sudo chown gapo:gapo /var/lib/gapo/certs

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable gapo-server
sudo systemctl start gapo-server

# Check status
sudo systemctl status gapo-server
sudo journalctl -u gapo-server -f
```

## Token Security

The token authenticates clients connecting to the server. Choose a strong, random token:

```bash
openssl rand -hex 32
```

The same token must be used by both the server and all clients.
