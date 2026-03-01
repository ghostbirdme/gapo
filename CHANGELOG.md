## v1.0.0 (2026-03-02)

- **New:** TUI now shows remote IP addresses for HTTP requests and TCP connections.
- **New:** TCP tunnels have a dedicated TUI panel showing active connections, total count, and connection history with connect/disconnect events.
- **New:** `--http` flag on the client for symmetry with `--tcp` (HTTP is still the default).
- **Enh:** TUI now shows `https://` URL when the server uses `--autocert` or `--tls-cert`.
- **Enh:** Server now accepts port numbers without a colon prefix (e.g., `--http 443` instead of `--http :443`).
- **Enh:** Client automatically reconnects in both TUI and log mode. If the connection drops, it retries with backoff (1s up to 30s).
- **Enh:** TUI uses a dark background for better visibility on light terminals.
- **Enh:** Duration display uses human-readable format (e.g., `<1ms`, `42ms`, `1.23s` instead of `758.521µs`).
- **Fix:** HTTPS certificate issuance with Let's Encrypt now works correctly. A bug was preventing the ACME TLS-ALPN-01 challenge from completing.
- **Fix:** TUI no longer shows garbled output when log messages appear during display.
- **Fix:** TCP tunnel disconnect events are now detected and displayed correctly in the TUI.

## v0.3.2 (2026-03-01)

- **Enh:** TCP tunnel connections are now limited to prevent resource exhaustion from too many simultaneous connections.
- **Enh:** Server now properly handles concurrent shutdown without race conditions.
- **Enh:** Streaming responses, large file downloads, and server-sent events no longer time out on the client.
- **Enh:** Client now uses a timeout when connecting to your local service, preventing hangs if the service is unresponsive.
- **Enh:** Client now sets a read deadline on incoming requests, preventing idle streams from consuming resources.
- **Enh:** Server now has a request body read timeout to protect against slow-body attacks.
- **Enh:** Proxy now strips additional hop-by-hop headers declared in the Connection header (RFC 7230).
- **Enh:** TUI connection stats are now capped to prevent high memory usage under heavy traffic.
- **Fix:** Self-update now flushes data to disk before replacing the binary, preventing corruption on system crash.
- **Fix:** Self-update checksum verification now uses constant-time comparison for improved security.
- **Fix:** Server no longer leaks memory from repeated failed login attempts across many different IP addresses.
- **Fix:** Config file read errors are now logged instead of silently ignored.
- **Fix:** Control messages that exceed the size limit are now rejected on both send and receive.
- **Fix:** Dropped TUI events are now logged instead of silently discarded.
- **Fix:** Server shutdown now stops accepting new tunnel connections before closing existing ones.

## v0.3.1 (2026-02-26)

- **Enh:** Server now limits repeated failed login attempts from the same IP address (5 failures per minute).
- **Enh:** Slow clients can no longer hold server connections open indefinitely.
- **Fix:** WebSocket connections no longer lose data sent during the upgrade handshake.
- **Fix:** Server now exits immediately with an error message if a port is already in use, instead of appearing to run normally.
- **Fix:** Large file downloads through tunnels no longer time out after 60 seconds.

## v0.3.0 (2026-02-26)

- **New:** Self-update with `--update` flag for both client and server.
- **New:** Custom TLS certificates with `--tls-cert` and `--tls-key` for the server.
- **New:** Optional CA/intermediate certificate chain with `--tls-ca`.
- **New:** FreeBSD support for both client and server (amd64, arm64).
- **Enh:** Download progress bar with resume and automatic retries on network failure.
- **Enh:** User confirmation prompt before downloading updates.

## v0.2.0 (2026-02-26)

- **Enh:** Short flags for common options (e.g., `-s` for `--server`, `-t` for `--token`).
- **Enh:** Separate release packages for server and client.
- **Dev:** Server builds limited to Linux only (client still supports Linux, macOS, Windows).
- **Dev:** Switched to POSIX-style flags with `--long` and `-short` support.

## v0.1.0 (2026-02-18)

- **New:** Tunnel server and client with subdomain routing.
- **New:** Automatic HTTPS via Let's Encrypt.
- **New:** WebSocket support.
- **New:** Interactive TUI dashboard.
- **New:** Automatic reconnection.
- **New:** Config file support for server and client.
- **New:** Cross-platform: Linux, macOS, Windows (amd64, arm64).

## v0.0.0 (2026-02-15)

- **Dev:** Internal use, migrated from bitbucket.
