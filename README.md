# Gapo Tunnel

Gapo lets you share services running on your computer with anyone on the internet.

For example:
- You have a website running on your laptop. You want your client to see it. Gapo gives you a public URL.
- You want to let someone SSH into your machine behind a firewall. Gapo gives you a public port.

Gapo has two apps:
- **`gapo-server`** — runs on a server (VPS) with a public IP. It receives traffic from the internet and forwards it to your computer.
- **`gapo`** — runs on your computer. It connects to the server and forwards traffic to your local service.

Both are single binary files with no extra software needed. Works on Linux, macOS, and Windows.

## Install

Download from the [Releases](https://github.com/ghostbirdme/gapo/releases) page.

**Linux / macOS:**

```bash
tar xzf gapo_*_linux_amd64.tar.gz
sudo mv gapo gapo-server /usr/local/bin/
```

**Windows:** Extract the zip file and add `gapo.exe` to your PATH.

---

## I want to share a website (HTTP/HTTPS)

Use this when you have a web application, API, or any HTTP service running locally.

### 1. Set up the server

You need a server (VPS) with a public IP and a domain name. Run `gapo-server` on that server:

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

You also need two DNS records pointing to your server:

```
A    tunnel.example.com      →  your-server-ip
A    *.tunnel.example.com    →  your-server-ip
```

That's it for the server. It handles HTTPS certificates automatically.

### 2. Connect from your computer

On your computer, start your web app (for example, on port 3000), then run:

```bash
gapo --server tunnel.example.com:19443 --token your-secret-token --tls myapp 3000
```

Now anyone can open `https://myapp.tunnel.example.com` in their browser and see your website.

### Local testing (no real server needed)

You can also test everything on one machine:

```bash
# Terminal 1 — start the server
gapo-server --token secret --domain localhost --http :8443 --tunnel :19443

# Terminal 2 — start a local website
python3 -m http.server 3000

# Terminal 3 — connect the tunnel
gapo --server localhost:19443 --token secret myapp 3000

# Terminal 4 — open it
curl -H "Host: myapp.localhost" http://localhost:8443/
```

---

## I want to share SSH access

Use this when you want someone to SSH into your machine, but your machine is behind a firewall or NAT.

### 1. Set up the server

Run `gapo-server` with the `--tcp-ports` flag. This tells the server which ports it can use for TCP tunnels:

```bash
gapo-server \
  --domain tunnel.example.com \
  --token your-secret-token \
  --tunnel :19443 \
  --tcp-ports 30000-39999
```

> Note: You do not need `--http`, `--autocert`, or DNS wildcard records for TCP-only tunnels. You only need the domain to point to your server.

Make sure ports 30000-39999 are open in your firewall.

### 2. Connect from your computer

On the machine you want to share SSH access from:

```bash
gapo --server tunnel.example.com:19443 --token your-secret-token --tls --tcp ssh 22
```

Gapo will show you the public port, for example: `tcp port 30001 -> localhost:22`

### 3. Someone else connects

Now anyone can SSH into your machine using:

```bash
ssh user@tunnel.example.com -p 30001
```

---

## I want to share a database

Same setup as SSH. Run the server with `--tcp-ports`, then connect from your machine:

```bash
# PostgreSQL
gapo --server tunnel.example.com:19443 --token your-secret-token --tls --tcp postgres 5432

# MySQL
gapo --server tunnel.example.com:19443 --token your-secret-token --tls --tcp mysql 3306
```

Others connect using the public port Gapo gives you:

```bash
# PostgreSQL
psql -h tunnel.example.com -p 30001 -U myuser mydb

# MySQL
mysql -h tunnel.example.com -P 30001 -u myuser -p mydb
```

---

## I want both HTTP and TCP tunnels

Run the server with all flags:

```bash
gapo-server \
  --domain tunnel.example.com \
  --token your-secret-token \
  --autocert \
  --cert-dir /var/lib/gapo/certs \
  --email admin@example.com \
  --http :443 \
  --tunnel :19443 \
  --tcp-ports 30000-39999
```

Then on your computer, run as many tunnels as you need:

```bash
# Share a website
gapo --server tunnel.example.com:19443 --token your-secret-token --tls myapp 3000

# Share SSH (in another terminal)
gapo --server tunnel.example.com:19443 --token your-secret-token --tls --tcp ssh 22
```

---

## Save your settings

You can save settings so you don't type them every time.

**Client** — create `~/.gapo/config`:

```bash
GAPO_SERVER=tunnel.example.com:19443
GAPO_TOKEN=your-secret-token
GAPO_TLS=true
```

Now you only need:

```bash
gapo myapp 3000           # share a website
gapo --tcp ssh 22         # share SSH
```

**Server** — create `/etc/gapo/config`:

```bash
GAPO_DOMAIN=tunnel.example.com
GAPO_TOKEN=your-secret-token
GAPO_TUNNEL=:19443
GAPO_TCP_PORTS=30000-39999
GAPO_HTTP=:443
GAPO_AUTOCERT=true
GAPO_CERT_DIR=/var/lib/gapo/certs
GAPO_EMAIL=admin@example.com
```

Now just run `gapo-server` with no flags.

---

## All flags

**Client (`gapo`):**

| Flag | What it does | Default |
|------|--------------|---------|
| `--server` | Server address | `localhost:19443` |
| `--token` | Password to connect (required) | — |
| `--tcp` | TCP mode (for SSH, databases, etc.) | off |
| `--tls` | Use secure connection to server | off |
| `--insecure` | Allow self-signed certificates | off |
| `--no-tui` | Show plain text logs instead of dashboard | off |
| `--version` | Show version | — |

**Server (`gapo-server`):**

| Flag | What it does | Default |
|------|--------------|---------|
| `--domain` | Your domain name | `localhost` |
| `--token` | Password for clients (required) | — |
| `--tunnel` | Port for client connections | `:19443` |
| `--http` | Port for web traffic | `:8443` |
| `--tcp-ports` | Port range for TCP tunnels (e.g. `30000-39999`) | off |
| `--autocert` | Get free HTTPS certificates | off |
| `--cert-dir` | Where to save certificates | `/var/lib/gapo/certs` |
| `--email` | Email for HTTPS certificates | — |
| `--tls` | Use self-signed certificate for tunnel | off |
| `--version` | Show version | — |

## More help

- [Server Setup](docs/server-setup.md) — complete server guide
- [Client Usage](docs/client-usage.md) — all client options and examples
- [Configuration](docs/configuration.md) — config file details

## License

Proprietary. See [LICENSE.md](LICENSE.md).

Copyright (c) 2026-present [Ghostbird Media](https://ghostbirdmedia.com)
