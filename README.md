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

## Screenshots

**HTTP tunnel — TUI dashboard:**

```
╭──────────────────────────────────────────────────────────────────────────╮
│ Gapo                                                    (Ctrl+C to quit) │
│                                                                          │
│  Session Status   ● online                                               │
│  Forwarding       https://myapp.tunnel.example.com                       │
│                                                                          │
│──────────────────────────────────────────────────────────────────────────│
│                                                                          │
│  Connections     ttl      rt1      rt5      p50      p90                 │
│                  47       0.82     0.64     12ms     89ms                │
│                                                                          │
│──────────────────────────────────────────────────────────────────────────│
│                                                                          │
│  HTTP Requests                                                           │
│  14:32:08  GET     /api/users                     200  12ms   203.0.113.5│
│  14:32:07  POST    /api/login                     200  89ms   203.0.113.5│
│  14:32:05  GET     /assets/style.css              200  <1ms   198.51.100.│
│  14:32:04  GET     /                              200  23ms   198.51.100.│
│  14:32:01  GET     /favicon.ico                   404  4ms    203.0.113.5│
╰──────────────────────────────────────────────────────────────────────────╯
```

**TCP tunnel — TUI dashboard:**

```
╭──────────────────────────────────────────────────────────────────────────╮
│ Gapo                                                    (Ctrl+C to quit) │
│                                                                          │
│  Session Status   ● online                                               │
│  Mode             TCP                                                    │
│  Forwarding       tcp port 30000                                         │
│                                                                          │
│──────────────────────────────────────────────────────────────────────────│
│                                                                          │
│  TCP Connections                                                         │
│  Active           2  (total 5)                                           │
│                                                                          │
│  14:35:12  203.0.113.5           connected                               │
│  14:35:10  198.51.100.22         connected                               │
│  14:34:58  203.0.113.5           disconnected                            │
│  14:34:45  198.51.100.22         disconnected                            │
│  14:34:30  203.0.113.5           connected                               │
╰──────────────────────────────────────────────────────────────────────────╯
```

**Server log output:**

```
$ gapo-server \
    --domain tunnel.example.com \
    --token your-secret-token \
    --autocert \
    --cert-dir /var/lib/gapo/certs \
    --email admin@example.com \
    --http 443 \
    --tunnel 19443 \
    --tcp-ports 30000-39999

2026/03/02 14:30:00 gapo-server v1.0.0 (built 2026-03-02T00:00:00Z)
2026/03/02 14:30:00 http server on :443 (autocert)
2026/03/02 14:30:00 tunnel listener on :19443
2026/03/02 14:30:00 tcp tunnel ports: 30000-39999
2026/03/02 14:30:05 tunnel: new connection from 203.0.113.5:48201
2026/03/02 14:30:05 tunnel: registered myapp.tunnel.example.com (http, v1.0.0)
2026/03/02 14:30:08 http: myapp.tunnel.example.com GET / -> 200 (23ms)
2026/03/02 14:30:09 http: myapp.tunnel.example.com GET /api/users -> 200 (12ms)
2026/03/02 14:31:00 tunnel: new connection from 198.51.100.22:52310
2026/03/02 14:31:00 tunnel: registered ssh.tunnel.example.com (tcp, v1.0.0)
2026/03/02 14:31:00 tunnel: TCP port 30000 allocated for ssh.tunnel.example.com
2026/03/02 14:31:15 tunnel: tcp connection 203.0.113.5 -> ssh.tunnel.example.com:30000
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
