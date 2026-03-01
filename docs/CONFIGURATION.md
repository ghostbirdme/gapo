# Configuration

Gapo can be configured via CLI flags, config files, or both. CLI flags always take priority.

## Priority Order

Settings are resolved in this order (highest priority first):

| Priority | Source | Example |
|----------|--------|---------|
| 1 | CLI flags | `--token secret` |
| 2 | Config file | `GAPO_TOKEN=secret` |
| 3 | Built-in defaults | `localhost:19443` |

This means you can set defaults in the config file and override specific values with flags when needed.

## File Format

Config files use `KEY=VALUE` format:

```bash
# Comments start with #
GAPO_TOKEN=my-secret-token
GAPO_SERVER=tunnel.example.com:19443

# Quotes are optional (stripped automatically)
GAPO_DOMAIN="example.com"

# Single quotes also work
GAPO_EMAIL='admin@example.com'
```

Rules:
- One key-value pair per line
- Blank lines and lines starting with `#` are ignored
- Surrounding quotes (single or double) are stripped from values
- Keys and values are trimmed of whitespace
- Unknown keys are silently ignored

## Boolean Values

Boolean keys accept the following truthy values (case-insensitive):

```
true    yes    1
```

Any other value (or missing key) is treated as `false`.

---

## Client Config

**Location:** `~/.gapo/config`

| Key | CLI flag | Description | Default |
|-----|----------|-------------|---------|
| `GAPO_SERVER` | `--server` | Server address | `localhost:19443` |
| `GAPO_TOKEN` | `--token` | Authentication token | — |
| `GAPO_TLS` | `--tls` | Encrypt tunnel connection (only when server uses `--tls`) | `false` |
| `GAPO_INSECURE` | `--insecure` | Allow self-signed certificates | `false` |

### Minimal example

```bash
# ~/.gapo/config
GAPO_SERVER=tunnel.example.com:19443
GAPO_TOKEN=your-secret-token
```

With this config, you only need:

```bash
gapo --http myapp 3000    # HTTP tunnel
gapo --tcp ssh 22         # TCP tunnel
```

### Development example

```bash
# ~/.gapo/config
GAPO_SERVER=localhost:19443
GAPO_TOKEN=secret
```

---

## Server Config

**Location:** `/etc/gapo/config` (primary), `~/.gapo/server-config` (fallback)

The server checks `/etc/gapo/config` first. If that file doesn't exist, it falls back to `~/.gapo/server-config`. Only one file is used — they are not merged.

| Key | CLI flag | Description | Default |
|-----|----------|-------------|---------|
| `GAPO_DOMAIN` | `--domain` | Base domain for subdomains | `localhost` |
| `GAPO_TOKEN` | `--token` | Authentication token | — |
| `GAPO_TUNNEL` | `--tunnel` | Tunnel listen port or address | `19443` |
| `GAPO_HTTP` | `--http` | HTTP listen port or address | `8443` |
| `GAPO_TCP_PORTS` | `--tcp-ports` | TCP tunnel port range | — |
| `GAPO_AUTOCERT` | `--autocert` | Automatic HTTPS via Let's Encrypt | `false` |
| `GAPO_CERT_DIR` | `--cert-dir` | Certificate cache directory | `/var/lib/gapo/certs` |
| `GAPO_EMAIL` | `--email` | Email for Let's Encrypt | — |
| `GAPO_TLS` | `--tls` | Self-signed TLS on tunnel listener (separate from `--autocert`) | `false` |
| `GAPO_TLS_CERT` | `--tls-cert` | Path to TLS certificate file | — |
| `GAPO_TLS_KEY` | `--tls-key` | Path to TLS private key file | — |
| `GAPO_TLS_CA` | `--tls-ca` | Path to CA/intermediate certificate | — |

### Production example (Let's Encrypt)

```bash
# /etc/gapo/config
GAPO_DOMAIN=tunnel.example.com
GAPO_TOKEN=your-secret-token
GAPO_HTTP=443
GAPO_TUNNEL=19443
GAPO_AUTOCERT=true
GAPO_CERT_DIR=/var/lib/gapo/certs
GAPO_EMAIL=admin@example.com
GAPO_TCP_PORTS=30000-39999
```

### Production example (custom certificate)

```bash
# /etc/gapo/config
GAPO_DOMAIN=tunnel.example.com
GAPO_TOKEN=your-secret-token
GAPO_HTTP=443
GAPO_TUNNEL=19443
GAPO_TLS_CERT=/etc/gapo/cert.pem
GAPO_TLS_KEY=/etc/gapo/key.pem
GAPO_TLS_CA=/etc/gapo/ca.pem
GAPO_TCP_PORTS=30000-39999
```

### File permissions

The config file contains your token. Restrict access so only the owner can read it:

```bash
chmod 600 /etc/gapo/config
```

---

## See also

- [Server Setup](SERVER-SETUP.md) — full server installation and deployment guide
- [Client Usage](CLIENT-USAGE.md) — all client options, examples, and troubleshooting

---

**Last Updated:** 2026-03-02 01:45:00 UTC
