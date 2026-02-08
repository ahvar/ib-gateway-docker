# Research FAQ: IB Gateway Docker, API choices, and secure networking

## 1) TWS API vs Client Portal Web API (CPAPI)

Interactive Brokers provides **two different API families**:

- **TWS API** (used by TWS and IB Gateway)
  - Transport: proprietary line-based protocol over a long-lived TCP socket.
  - Typical endpoints/ports: IB Gateway/TWS local socket (for example 4001/4002 for gateway, 7496/7497 for TWS), then optionally relayed by socat in this project.
  - Interface style: event-driven callbacks / async streaming over the same socket.
  - Typical Python client libraries: `ibapi`, `ib_insync`, `ib_async`.

- **Client Portal Web API**
  - Transport: HTTPS REST endpoints and websocket streaming endpoints exposed by Client Portal Gateway.
  - Interface style: HTTP request/response + websocket for streaming account/market updates.
  - Typical deployment: run IBKR Client Portal Gateway process and talk to it via REST/websocket.

In practical terms: they are separate stacks; you generally pick one as your primary integration path.

### Official references

- TWS API landing page: https://www.interactivebrokers.com/en/trading/ib-api.php
- IBKR Campus API docs hub: https://ibkrcampus.com/campus/ibkr-api-page/
- Client Portal API docs (IBKR Campus): https://ibkrcampus.com/campus/ibkr-api-page/web-api-introduction/
- IBKR API guides: https://interactivebrokers.github.io/

## 2) Secure networking patterns for IB Gateway

### Why this repo emphasizes localhost

The IB socket protocol itself is **not encrypted/authenticated by default** end-to-end at the API socket layer. That is why this repository defaults to localhost-only host port mappings and discusses extra security layers before exposing access broadly.

### SSH bastion + reverse/forward tunnel pattern (from README)

Pattern summary:
1. `ib-gateway-docker` opens a **reverse tunnel** (`ssh -R`) from the container to a bastion.
2. Your analysis client (for example a Jupyter container) opens a **local forward tunnel** (`ssh -L`) to that same bastion/port.
3. Net effect: your strategy process talks to local port, traffic is carried inside SSH tunnels, and only SSH is publicly reachable.

Security properties:
- Good when done with key-based auth, strict host key checking, locked-down bastion users, and least-privilege security groups.
- Operationally simple and robust for single-tenant or small-team setups.
- Bastion becomes critical security boundary and should be hardened/monitored.

### Compare to mTLS model you used for remote Docker daemon

Your prior pattern (Docker daemon over mutual TLS, narrow source IP allow-list) is conceptually similar: both add transport security and endpoint identity over TCP.

- **SSH tunneling**:
  - Identity/auth via SSH keys.
  - Encryption and forwarding built in.
  - Often easier to operate when you already have bastion workflows.

- **mTLS directly on service**:
  - Identity/auth via X.509 certs on both client and server.
  - Strong for service-to-service designs and policy automation.
  - Requires cert issuance, rotation, revocation process.

Could mTLS be used here? Yes, with a proxy layer (e.g., stunnel/envoy/nginx stream) in front of IB socket endpoints because IB Gateway itself does not natively expose an mTLS socket for TWS API in the same way Docker daemon does. This is feasible but usually more moving parts than SSH bastion tunnels.

### Other secure alternatives to compare

- Private networking only (same VPC/private subnet, no public API ports, SG-restricted sources).
- WireGuard or Tailscale overlay network + private ports.
- Cloud LB + TCP TLS termination + client cert auth (if architecture needs centralized ingress).
- Zero-trust access broker products (team/audit/compliance-driven environments).

## 3) Websockets relevance

Your intuition is right: websockets provide full-duplex communication over one long-lived TCP connection, useful for low-latency streaming updates.

In IBKR context:
- TWS API stream-like behavior is achieved over the TWS socket protocol (not websocket).
- Client Portal stack includes websocket-based streaming for some data flows.

Protocols:
- `ws://` = plaintext websocket (avoid on untrusted networks).
- `wss://` = websocket over TLS.

Core specs/references:
- RFC 6455 (The WebSocket Protocol): https://datatracker.ietf.org/doc/html/rfc6455
- MDN WebSocket docs: https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API

## 4) `run.sh` line 107 and 110: where are function definitions?

In `image-files/scripts/run.sh`, line 110 calls `run_scripts "$HOME/$START_SCRIPTS"`.
That function is defined in `image-files/scripts/common.sh` and loaded by `source "${SCRIPT_PATH}/common.sh"` near the top of `run.sh`.

## 5) What are `jts.ini` and `jts.ini.tmpl`?

- `jts.ini.tmpl` is the template (contains placeholders such as `${TIME_ZONE}`).
- `jts.ini` is the concrete default file (already rendered with `Etc/UTC` in this repo copy).

At container startup, `apply_settings` in `common.sh` writes a `jts.ini` from the template **only if one does not already exist** in the selected settings path.

## 6) Where are `IbLoginId` and `IbPassword` configured?

In `config.ini.tmpl`, these are templated from:
- `IbLoginId=${TWS_USERID}`
- `IbPassword=${TWS_PASSWORD}`

So yes: they come from environment variables supplied to the container (or corresponding `_FILE` secret-based variables where supported by scripts). `apply_settings` renders the template to runtime `config.ini` with `envsubst`.

## 7) If you will not use TWS GUI, what changes?

If you are not using desktop TWS GUI:
- Prefer the `ib-gateway` image/service instead of `tws-rdesktop`.
- You can keep VNC disabled by not setting `VNC_SERVER_PASSWORD`.
- Keep API read-only where possible (`READ_ONLY_API=yes` when suitable for your workflows).
- Keep localhost-only or tunnel-only exposure; avoid broad port publishing.
- You do not need xrdp/xfce-specific tuning from the TWS compose profile.

## 8) EC2 initial setup guidance (starting from sample compose)

Suggested baseline for first run:

1. Start from this repo’s `docker-compose.yml` (gateway profile).
2. Keep host port bindings on `127.0.0.1` initially.
3. Provide credentials via Docker secrets (`TWS_PASSWORD_FILE`) instead of plain env if possible.
4. Mount persistent volume for `TWS_SETTINGS_PATH` to preserve settings/session metadata.
5. Decide connectivity model before production:
   - same-host client only (localhost),
   - same-docker-network client,
   - or SSH tunnel to bastion.
6. On EC2 security groups:
   - expose only SSH (or SSM Session Manager and no SSH port),
   - do not expose 4001/4002 publicly.
7. Add basic ops hardening:
   - CloudWatch logs/metrics, restart policy, host patching, fail2ban/sshd hardening if public SSH.
8. Run in paper mode first (`TRADING_MODE=paper`) and verify 2FA operational flow.

Helpful docs in this repo:
- sample compose files,
- security section (`Leaving localhost`, `SSH Tunnel`),
- credentials section (`_FILE` secrets usage).
