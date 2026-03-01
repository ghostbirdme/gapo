# Frequently Asked Questions

## General

### What is Gapo?

Gapo is a tunnel tool that exposes local services to the internet. It gives your local web app, SSH server, or database a public address that anyone can reach.

### How is Gapo different from other tunnel services?

Gapo is fully self-hosted. You run your own server, so you control the domain, the data, and the cost. There are no usage limits, no account signups, and no third-party dependencies.

### What platforms are supported?

**Client (`gapo`):** Linux, macOS, Windows, FreeBSD (amd64 and arm64).

**Server (`gapo-server`):** Linux, FreeBSD (amd64 and arm64).

### Do I need to install anything else?

No. Both binaries are static with no runtime dependencies. Just download and run.

### Can I run multiple tunnels at the same time?

Yes. Run one `gapo` command per service, each with a different name:

```bash
gapo --http frontend 3000
gapo --http api 8080
gapo --tcp ssh 22
```

Each tunnel gets its own connection to the server.

---

## Server

### What kind of server do I need?

Any VPS with a public IP. Gapo is lightweight and I/O-bound — a $4-5/mo VPS with 1 vCPU and 64 MB RAM is enough. ARM instances work fine too.

### Do I need a domain name?

For HTTP tunnels, yes — you need a domain with a wildcard DNS record so that subdomains like `myapp.yourdomain.com` reach your server.

For TCP-only tunnels, the base domain is enough (no wildcard needed). You can also use an IP address directly.

### Can I use Gapo without HTTPS?

Yes. By default, the server runs plain HTTP. Use `--autocert` or `--tls-cert` only when you want HTTPS.

### How do HTTPS certificates work?

With `--autocert`, the server gets free certificates from Let's Encrypt automatically. Certificates are cached in `--cert-dir` and renewed before they expire. No manual steps needed.

You can also provide your own certificate with `--tls-cert` and `--tls-key`.

### Can I run other services on the same VPS?

Yes, as long as they use different ports. Gapo only binds the ports you configure. See [Server Setup — Running Other Services](SERVER-SETUP.md#running-other-services-on-the-same-vps) for details.

### What happens if the server restarts?

All active tunnels disconnect. Clients with automatic reconnection will reconnect within a few seconds. The server starts fresh with no remembered tunnels.

### Can multiple users share the same server?

Yes. All clients use the same token to connect. Each client picks a different subdomain name. There is currently no per-user access control — anyone with the token can create tunnels.

### Can I use multiple domains or tokens?

Not at the moment. The server supports one domain and one token. All tunnels share the same base domain (e.g., `*.tunnel.example.com`) and the same authentication token.

---

## Client

### What does the name argument do?

The name (e.g., `myapp` in `gapo --http myapp 3000`) serves two purposes:

- **HTTP tunnels:** It becomes the subdomain. Your service is available at `myapp.yourdomain.com`.
- **TCP tunnels:** It is just a label for identification. You can use any name you want.

### Should I use my HTTP or HTTPS port?

Always use the **HTTP** port. Gapo forwards plain HTTP to your local service. HTTPS is handled by `gapo-server` on the public side.

For example, if your web server runs on port 80 (HTTP) and 443 (HTTPS), use port 80:

```bash
gapo --http myapp 80
```

If you use the HTTPS port (443), you will see an error like: "Bad Request — You're speaking plain HTTP to an SSL-enabled server port."

The traffic flow is:

```
Browser → HTTPS → gapo-server → plain HTTP → your local service
```

### Do I need `--tls` to connect?

Only if the server uses `--tls` on the **tunnel listener**. The `--autocert` and `--tls-cert` flags are separate — they only affect HTTPS for end users, not the tunnel connection.

| Server setup | Tunnel listener | Client flags |
|-------------|----------------|-------------|
| No TLS flags (default) | Plain TCP | No extra flags |
| `--tls` (self-signed) | Self-signed TLS | `--tls --insecure` |
| `--autocert` only | Plain TCP | No extra flags |
| `--tls-cert` only | Plain TCP | No extra flags |

**Common mistake:** Do not use `--tls` on the client when the server only uses `--autocert`. The `--autocert` flag handles HTTPS on the public side (port 443). The tunnel listener (port 19443) stays plain TCP unless you also add `--tls` to the server.

### What happens if my internet connection drops?

The client detects the disconnect and reconnects automatically with exponential backoff (1s → 2s → 4s → ... up to 30s). Once reconnected, your tunnel resumes with the same subdomain.

### Can I use Gapo in a script or CI pipeline?

Yes. Use `--no-tui` for plain log output:

```bash
gapo --http --no-tui myapp 3000
```

Or run it in the background:

```bash
gapo --http --no-tui myapp 3000 &
```

### Can I save my settings so I don't type them every time?

Yes. Create `~/.gapo/config`:

```bash
GAPO_SERVER=tunnel.example.com:19443
GAPO_TOKEN=your-secret-token
```

Then just run:

```bash
gapo --http myapp 3000
```

See [Configuration](CONFIGURATION.md) for all available keys.

---

## Security

### Is the tunnel connection encrypted?

Only if you enable `--tls` on **both** the server and client. By default, the tunnel connection between client and server is plain TCP.

Note: `--autocert` and `--tls-cert` only encrypt HTTPS traffic for end users (port 443). They do not encrypt the tunnel connection (port 19443). To encrypt the tunnel, add `--tls` to the server and use `--tls --insecure` on the client (the tunnel uses a self-signed certificate).

### How is the token protected?

The token is compared using constant-time comparison, which prevents timing attacks. The server also rate-limits failed attempts: after 5 wrong tokens from the same IP within 1 minute, further connections are rejected.

### Can someone guess my token?

If you use a strong random token, it is practically impossible. Generate one with:

```bash
openssl rand -hex 32
```

This produces a 64-character hex string (256 bits of entropy).

### Can someone hijack my subdomain?

No. If another client tries to register the same subdomain while yours is active, the old tunnel is replaced by the new one. Only clients with a valid token can register.

### Should I put the token on the command line?

Prefer the config file. Command-line arguments may appear in process listings (`ps`) and shell history. The config file can be restricted with file permissions:

```bash
chmod 600 ~/.gapo/config
```

---

## Limits and Performance

### How large can uploads be?

The server limits request bodies (uploads) to **100 MB**. Uploads larger than 100 MB will fail.

### How large can downloads be?

There is **no size limit** on downloads. Large file downloads are streamed through the tunnel without buffering the entire file in memory.

### How many concurrent requests can a tunnel handle?

Many. Gapo uses yamux stream multiplexing — hundreds of requests can flow through a single tunnel connection at the same time.

### How many tunnels can the server handle?

The server limits concurrent tunnel connections to 1000 and concurrent TCP tunnel connections to 1000. Each tunnel uses one TCP connection with multiplexed streams, so 1000 tunnels can serve many thousands of concurrent requests.

### Does Gapo add latency?

Gapo adds a small amount of latency per request (typically a few milliseconds). The main factors are:

- Network round-trip between server and client
- yamux stream setup overhead
- Local service response time

For most use cases, the added latency is not noticeable.

### Does WebSocket work through the tunnel?

Yes. WebSocket connections are detected automatically and handled as a raw TCP bridge. No special configuration needed.

---

## Problems

### "connection refused"

- The server is not running or not reachable.
- Check the `--server` address and port.
- Make sure the tunnel port (default 19443) is open in the server's firewall.

### "authentication failed"

- The `--token` does not match the server's token.
- Check for extra spaces or quotes in the token.

### "too many failed attempts"

- The server blocked your IP after 5 failed login attempts.
- Wait 1 minute and try again with the correct token.

### "subdomain already in use"

- Another client is using that name. Choose a different one.
- If your previous session did not disconnect cleanly, wait about 30 seconds for the server to detect it.

### TLS certificate errors

- For self-signed server certs (development), add `--insecure` on the client.
- For production servers with valid certs, use `--tls` without `--insecure`.
- Make sure the server's domain matches the certificate.

### "Bad Request — You're speaking plain HTTP to an SSL-enabled server port"

- You are pointing the tunnel at a local HTTPS port (e.g., 443). Use the HTTP port instead (e.g., 80 or 8080).
- Gapo forwards plain HTTP to your local service. HTTPS is handled by `gapo-server` on the public side.
- See [Should I use my HTTP or HTTPS port?](#should-i-use-my-http-or-https-port) above.

### Tunnel connects but the service is not reachable

- Verify your local service is running on the specified port.
- Check that it listens on `localhost`, not just a specific interface.
- Test locally: `curl http://localhost:<port>`

### Large uploads fail

- The server has a 100 MB upload limit. Files larger than 100 MB are rejected.

### Server appears to start but nothing works

- Check if another process is using the same port. The server now exits with a clear error if a port is already in use.
- Check the server logs: `journalctl -u gapo-server -f`

---

## Updating

### How do I update Gapo?

Both binaries can update themselves:

```bash
gapo --update
gapo-server --update
```

You will be asked to confirm before any download starts.

### Is the update safe?

Yes. The downloaded file is verified with a SHA-256 checksum (constant-time comparison) before replacing the binary. The new binary is flushed to disk before replacing the old one, so even a system crash during update cannot corrupt the binary. If the checksum does not match, the update is aborted.

### Do I need to restart the server after updating?

Yes. After updating `gapo-server`, restart the service:

```bash
sudo systemctl restart gapo-server
```

Active tunnels will disconnect and reconnect automatically.

---

**Last Updated:** 2026-03-02 01:45:00 UTC
