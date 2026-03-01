# Server Setup

Complete guide for installing and running `gapo-server` on a public-facing machine.

## Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 1 vCPU | 1+ vCPU |
| RAM | 64 MB free | 128 MB+ |
| Disk | ~10 MB | 50 MB+ |
| OS | Linux or FreeBSD (amd64 or arm64) | Any modern Linux or FreeBSD |

You also need:
- A public IP address
- A domain name (with wildcard DNS for HTTP tunnels)
- Ports open in your firewall (see [Firewall](#firewall))

The server is a single static binary with no runtime dependencies. It is I/O-bound — a $4-5/mo VPS (Hetzner, DigitalOcean, Vultr, etc.) is more than enough. ARM instances work fine too.

## Installation

Download the latest binary from the [releases page](https://github.com/ghostbirdme/gapo/releases):

```bash
tar xzf gapo_*_linux_amd64.tar.gz
sudo mv gapo-server /usr/local/bin/
gapo-server --version
```

## DNS Setup

Gapo routes HTTP traffic based on the `Host` header. You need two DNS records pointing to your server:

```
A       tunnel.example.com      → <your-server-ip>
A       *.tunnel.example.com    → <your-server-ip>
```

This allows subdomains like `myapp.tunnel.example.com` to reach the server.

> For TCP-only tunnels, the wildcard record is not required — only the base domain needs to resolve.

## Quick Start (Local Development)

For local testing, no DNS or TLS is needed:

```bash
gapo-server --token secret --domain localhost --http 8443 --tunnel 19443
```

This starts:
- HTTP proxy on port 8443
- Tunnel listener on port 19443

Test it:

```bash
# Start a local web server
python3 -m http.server 3000

# Connect the tunnel
gapo --server localhost:19443 --token secret --http myapp 3000

# Verify
curl -H "Host: myapp.localhost" http://localhost:8443/
```

## Production Setup (Let's Encrypt)

Use `--autocert` for automatic HTTPS certificates from Let's Encrypt:

```bash
# Prepare certificate directory
sudo mkdir -p /var/lib/gapo/certs

# Start the server
gapo-server \
  --domain tunnel.example.com \
  --token your-secret-token \
  --autocert \
  --cert-dir /var/lib/gapo/certs \
  --email admin@example.com \
  --http 443 \
  --tunnel 19443
```

This starts:
- HTTPS proxy on port 443 with automatic certificates
- HTTP server on port 80 for ACME challenges and HTTPS redirect
- Tunnel listener on port 19443

Certificates are cached in `--cert-dir` and renewed automatically. Only subdomains with active tunnels get certificates.

> **Note:** The server handles HTTPS on the public side and forwards plain HTTP to clients. When connecting a tunnel, always use the local service's HTTP port (e.g., 80 or 3000), not its HTTPS port (e.g., 443).

To also enable TCP tunnels, add `--tcp-ports`:

```bash
gapo-server \
  --domain tunnel.example.com \
  --token your-secret-token \
  --autocert \
  --cert-dir /var/lib/gapo/certs \
  --email admin@example.com \
  --http 443 \
  --tunnel 19443 \
  --tcp-ports 30000-39999
```

### Certificate Lifecycle

Let's Encrypt certificates are valid for 90 days and renew automatically ~30 days before expiry. No manual intervention needed.

**Inactive subdomains:** When a tunnel disconnects, no new certificate is issued for that subdomain. The cached certificate expires naturally within 90 days.

**Compromised server:** Revoke affected certificates through Let's Encrypt directly and rotate your token. To remove a cached certificate:

```bash
rm /var/lib/gapo/certs/myapp.tunnel.example.com
```

## Custom TLS Certificate

If you have your own certificate (e.g., from Cloudflare, your CA, or a wildcard cert):

```bash
gapo-server \
  --domain tunnel.example.com \
  --token your-secret-token \
  --tls-cert /etc/gapo/cert.pem \
  --tls-key /etc/gapo/key.pem \
  --http 443 \
  --tunnel 19443
```

If your certificate is signed by an intermediate CA, include the chain:

```bash
gapo-server \
  --domain tunnel.example.com \
  --token your-secret-token \
  --tls-cert /etc/gapo/cert.pem \
  --tls-key /etc/gapo/key.pem \
  --tls-ca /etc/gapo/ca.pem \
  --http 443 \
  --tunnel 19443
```

Notes:
- `--tls-cert` and `--tls-key` must be provided together
- `--tls-ca` is optional — can contain one or more PEM-encoded certificates
- The certificate should cover your wildcard domain (e.g., `*.tunnel.example.com`)
- You are responsible for renewing the certificate before it expires
- To reload a renewed certificate, restart the server: `sudo systemctl restart gapo-server`

## Development Setup (Self-Signed TLS)

For development with TLS on the tunnel connection:

```bash
gapo-server \
  --domain localhost \
  --token secret \
  --tls \
  --http 8443 \
  --tunnel 19443
```

The `--tls` flag generates an in-memory self-signed certificate for the tunnel listener. Clients must use `--insecure` to skip certificate verification.

## Firewall

Open the ports your setup needs:

| Port | Purpose | When needed |
|------|---------|-------------|
| 80 | ACME challenges + HTTPS redirect | `--autocert` |
| 443 | HTTPS proxy | `--autocert` or `--tls-cert` |
| 8443 | HTTP proxy (default) | Development / plain HTTP |
| 19443 | Tunnel connections from clients | Always |
| 30000-39999 | TCP tunnel ports | `--tcp-ports` |

Example with `firewalld`:

```bash
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --permanent --add-port=19443/tcp
sudo firewall-cmd --permanent --add-port=30000-39999/tcp
sudo firewall-cmd --reload
```

Example with `ufw`:

```bash
sudo ufw allow 443/tcp
sudo ufw allow 19443/tcp
sudo ufw allow 30000:39999/tcp
```

## Running as a Systemd Service

### 1. Create the config file

```bash
mkdir -p /etc/gapo
mkdir -p /var/lib/gapo/certs
```

Write your settings to `/etc/gapo/config`:

```bash
GAPO_DOMAIN=tunnel.example.com
GAPO_TOKEN=your-secret-token
GAPO_TUNNEL=19443
GAPO_HTTP=443
GAPO_AUTOCERT=true
GAPO_CERT_DIR=/var/lib/gapo/certs
GAPO_EMAIL=admin@example.com
GAPO_TCP_PORTS=30000-39999
```

Restrict access to the config file:

```bash
chmod 600 /etc/gapo/config
```

### 2. Create the service file

Write to `/etc/systemd/system/gapo-server.service`:

```ini
[Unit]
Description=Gapo Tunnel Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/gapo-server
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 3. Enable and start

```bash
systemctl daemon-reload
systemctl enable gapo-server
systemctl start gapo-server
```

### 4. Verify

```bash
systemctl status gapo-server
journalctl -u gapo-server -f
```

## Running Other Services on the Same VPS

Gapo binds ports 80 and 443 (in autocert mode), but other services can run on different ports:

| Service | Port | Access |
|---------|------|--------|
| gapo-server | 80/443 | Public (tunnel traffic) |
| PostgreSQL | 5432 | Internal only |
| Redis | 6379 | Internal only |
| Your API app | 3000 | Via gapo tunnel or direct |
| Admin panel | 8080 | Via gapo tunnel or direct |

You can run a gapo client on the same VPS to give a local service a public subdomain with HTTPS:

```bash
gapo --server localhost:19443 --token your-secret-token --http --no-tui api 3000
gapo --server localhost:19443 --token your-secret-token --http --no-tui admin 8080
```

Each client gets its own route. No conflicts as long as subdomain names are different.

For internal services (databases, caches), keep their ports closed in the firewall and access them via SSH or VPN.

## Updating

Check for updates and update the binary in-place:

```bash
gapo-server --update
```

You'll be prompted before downloading. The download supports resume on network failure with automatic retries.

After updating, restart the service:

```bash
sudo systemctl restart gapo-server
```

## Security

### Token

The token authenticates clients connecting to the server. Generate a strong random token:

```bash
openssl rand -hex 32
```

The same token must be used by both the server and all clients. Store it in the config file with restricted permissions:

```bash
chmod 600 /etc/gapo/config
```

### Rate Limiting

The server automatically rate-limits failed authentication attempts: after 5 failures from the same IP within 1 minute, further connections from that IP are rejected. This protects against brute-force token guessing.

### Recommendations

- Use `--autocert` or `--tls-cert` for HTTPS on the public side
- Add `--tls` to encrypt the tunnel listener (clients need `--tls --insecure`)
- Keep the token out of shell history — use the config file instead of `--token` on the command line
- Restrict firewall access to only the ports you need

## CLI Flags

| Flag | Short | Description | Default |
|------|-------|-------------|---------|
| `--domain` | `-d` | Base domain for subdomains | `localhost` |
| `--token` | `-t` | Auth token (required) | — |
| `--tunnel` | `-T` | Tunnel listen port or address | `19443` |
| `--http` | `-H` | HTTP listen port or address | `8443` |
| `--tcp-ports` | `-p` | TCP tunnel port range (e.g. `30000-39999`) | — |
| `--autocert` | | Automatic HTTPS via Let's Encrypt | `false` |
| `--cert-dir` | `-c` | Certificate cache directory | `/var/lib/gapo/certs` |
| `--email` | `-e` | Email for Let's Encrypt registration | — |
| `--tls` | | Self-signed TLS on tunnel listener (separate from `--autocert`) | `false` |
| `--tls-cert` | | Path to TLS certificate file | — |
| `--tls-key` | | Path to TLS private key file | — |
| `--tls-ca` | | Path to CA/intermediate certificate file | — |
| `--update` | | Check for updates and self-update | — |
| `--version` | `-v` | Show version and exit | — |
| `--about` | | Show license and copyright info | — |

## Config File

The server reads config from `/etc/gapo/config` first, then falls back to `~/.gapo/server-config`. CLI flags override config file values.

| Key | Maps to |
|-----|---------|
| `GAPO_DOMAIN` | `--domain` |
| `GAPO_TOKEN` | `--token` |
| `GAPO_TUNNEL` | `--tunnel` |
| `GAPO_HTTP` | `--http` |
| `GAPO_TCP_PORTS` | `--tcp-ports` |
| `GAPO_AUTOCERT` | `--autocert` |
| `GAPO_CERT_DIR` | `--cert-dir` |
| `GAPO_EMAIL` | `--email` |
| `GAPO_TLS` | `--tls` |
| `GAPO_TLS_CERT` | `--tls-cert` |
| `GAPO_TLS_KEY` | `--tls-key` |
| `GAPO_TLS_CA` | `--tls-ca` |

See [Configuration](CONFIGURATION.md) for file format details.

---

**Last Updated:** 2026-03-02 01:45:00 UTC
