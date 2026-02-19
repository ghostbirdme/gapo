# Configuration

Gapo supports configuration via CLI flags and config files. CLI flags always take priority over config file values.

## Config File Format

Config files use a simple `KEY=VALUE` format:

```bash
# Comments start with #
GAPO_TOKEN=my-secret-token
GAPO_SERVER=tunnel.example.com:19443

# Quotes are optional (stripped automatically)
GAPO_DOMAIN="example.com"
```

- Blank lines and lines starting with `#` are ignored
- Surrounding quotes (single or double) are stripped from values
- Keys and values are trimmed of whitespace

## Client Config

**Location:** `~/.gapo/config`

| Key | Description | Default |
|-----|-------------|---------|
| `GAPO_SERVER` | Server tunnel address | `localhost:19443` |
| `GAPO_TOKEN` | Auth token | — |
| `GAPO_TLS` | Connect using TLS | `false` |
| `GAPO_INSECURE` | Skip TLS certificate verification | `false` |

Example:

```bash
# ~/.gapo/config
GAPO_SERVER=tunnel.example.com:19443
GAPO_TOKEN=your-secret-token
GAPO_TLS=true
```

## Server Config

**Location:** `/etc/gapo/config` (primary), `~/.gapo/server-config` (fallback)

The server checks `/etc/gapo/config` first. If that file is missing or empty, it falls back to `~/.gapo/server-config`.

| Key | Description | Default |
|-----|-------------|---------|
| `GAPO_HTTP` | HTTP listen address | `:8443` |
| `GAPO_TUNNEL` | Tunnel listen address | `:19443` |
| `GAPO_DOMAIN` | Base domain for subdomains | `localhost` |
| `GAPO_TOKEN` | Auth token | — |
| `GAPO_TLS` | Enable TLS on tunnel listener | `false` |
| `GAPO_AUTOCERT` | Enable Let's Encrypt HTTPS | `false` |
| `GAPO_CERT_DIR` | Directory for autocert cache | `/var/lib/gapo/certs` |
| `GAPO_EMAIL` | Email for Let's Encrypt registration | — |
| `GAPO_TCP_PORTS` | TCP tunnel port range (e.g. `30000-39999`) | — |

Example:

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

## Boolean Values

Boolean config keys accept the following truthy values (case-insensitive):
- `true`
- `yes`
- `1`

Any other value (or missing key) is treated as `false`.

## Priority Order

Settings are resolved in this order (highest priority first):

1. CLI flags
2. Config file
3. Built-in defaults
