# Gapo Tunnel

Gapo exposes local services to the internet through secure tunnels. Share a website, SSH access, or database with a single command — no port forwarding or firewall changes needed.

**Features:**
- HTTP/HTTPS tunnels with automatic Let's Encrypt certificates
- TCP tunnels for SSH, databases, and any TCP service
- WebSocket support
- Interactive TUI dashboard
- Automatic reconnection on network failure
- Self-update built in
- Single static binary, no dependencies

---

## Install

Download from the [Releases](https://github.com/ghostbirdme/gapo/releases) page.

**Linux / macOS / FreeBSD:**

```bash
tar xzf gapo_*_linux_amd64.tar.gz
sudo mv gapo gapo-server /usr/local/bin/
```

**Windows** (client only):

Extract the zip file and add `gapo.exe` to your PATH.

---

## Share a website (HTTP/HTTPS)

### Step 1 — Set up the server

Run this on your VPS:

```bash
gapo-server \
  --domain tunnel.example.com \
  --token your-secret-token \
  --autocert \
  --cert-dir /var/lib/gapo/certs \
  --email admin@example.com \
  --http 443 \
  --tunnel 19443
```

Add two DNS records pointing to your server:

```
A    tunnel.example.com      →  your-server-ip
A    *.tunnel.example.com    →  your-server-ip
```

### Step 2 — Connect from your computer

```bash
gapo --server tunnel.example.com:19443 --token your-secret-token --http myapp 3000
```

Your site is now live at `https://myapp.tunnel.example.com`.

> **Note:** Use your local service's **HTTP** port (e.g., 80 or 3000), not HTTPS (443). Gapo handles HTTPS automatically on the public side.

---

## Share SSH access

### Step 1 — Set up the server

```bash
gapo-server \
  --domain tunnel.example.com \
  --token your-secret-token \
  --tunnel 19443 \
  --tcp-ports 30000-39999
```

Make sure ports 30000-39999 are open in your firewall.

### Step 2 — Connect from your computer

```bash
gapo --server tunnel.example.com:19443 --token your-secret-token --tcp ssh 22
```

Gapo will show the allocated public port (e.g., `tcp port 30000 -> localhost:22`).

### Step 3 — Share with others

```bash
ssh user@tunnel.example.com -p 30000
```

---

## Share a database

Same server setup as SSH, then connect:

```bash
gapo --server tunnel.example.com:19443 --token your-secret-token --tcp mysql 3306
```

Others connect using the port Gapo assigns:

```bash
mysql -h tunnel.example.com -P 30000 -u myuser -p
```

---

## Save your settings

Create `~/.gapo/config` to avoid repeating flags:

```bash
GAPO_SERVER=tunnel.example.com:19443
GAPO_TOKEN=your-secret-token
```

Now you only need:

```bash
gapo --http myapp 3000    # share a website
gapo --tcp ssh 22         # share SSH
```

See [Configuration](docs/CONFIGURATION.md) for server config and all available keys.

---

## Updating

```bash
gapo --update
gapo-server --update
```

---

## More help

- [Server Setup](docs/SERVER-SETUP.md) — server guide, systemd, firewall, requirements
- [Client Usage](docs/CLIENT-USAGE.md) — all client options, TUI dashboard, troubleshooting
- [Configuration](docs/CONFIGURATION.md) — config file format and all keys
- [FAQ](docs/FAQ.md) — common questions and problems

## License

Proprietary. See [LICENSE.md](LICENSE.md).

Copyright (c) 2026-present [Ghostbird Media](https://ghostbirdmedia.com)
