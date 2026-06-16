# Lab 4 — OS & Networking: Trace, Debug, and Read the Substrate

## 1. Annotated Packet Capture (lab4-trace.txt)

### TCP Three-Way Handshake
- SYN:
  `Flags [S]` — client initiates connection to :8080
- SYN/ACK:
  `Flags [S.]` — server acknowledges and accepts connection
- ACK:
  `Flags [.]` — client confirms connection established

---

### HTTP Request
- Request line:
  `POST /notes HTTP/1.1`

- Headers:
  Content-Type: application/json
  Content-Length: 39

- Body:
```json
{"title":"trace me","body":"in flight"}

The capture shows a complete TCP connection lifecycle. The client established a connection using the TCP three-way handshake (SYN → SYN/ACK → ACK), sent an HTTP POST request containing JSON data, received an HTTP/1.1 201 Created response with the created note in JSON format, and then both sides closed the connection gracefully using TCP FIN packets. The capture demonstrates how application-layer HTTP traffic is carried inside a TCP connection.

1.3

Debug Command Outputs (1.3)
Listening on port 8080

```text
COMMAND    PID USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
quicknote 2376 i      5u   IPv6 0x8a522e14d49aca29      0t0  TCP *:http-alt (LISTEN)

Routing tables

Internet:
Destination        Gateway            Flags       Netif
default            172.20.10.1        UGScIg      en0
127                127.0.0.1          UCS         lo0
127.0.0.1          127.0.0.1          UH          lo0

Start: 2026-06-16T17:22:11+0100
HOST: is-MacBook-Air.local

HOST                 Loss%   Snt   Last   Avg  Best  Wrst StDev
localhost            0.0%     5    0.2   0.2   0.2   0.2   0.0

104.20.23.154
172.66.147.243

zsh: command not found: journalctl
On macOS, journalctl is not available because systemd is not used. QuickNotes runs directly via go run . instead of a system service.

If QuickNotes returned a 502 Bad Gateway error, I would first verify that the application is actually running and listening on port 8080 using ss -tlnp (or lsof on macOS). A 502 usually means a reverse proxy or gateway cannot successfully communicate with the backend service. Next, I would check application logs to see whether QuickNotes crashed or returned errors. I would then test the application directly with curl -v http://localhost:8080/notes to determine whether the issue is with the application itself or the proxy layer. Finally, I would inspect network connectivity and routing to ensure requests can reach the service and responses can be returned correctly.

Task 2

## 2.1 Broken Deployment Output

Second instance failed with:

listen tcp :8080: bind: address already in use
exit status 1

Task 2.1

Is QuickNotes running? (process check)?
501  2376  2108   0 Sun12PM ttys000  0:00.43 /var/folders/.../quicknotes
501 32623 25700   0  5:32PM ttys008  0:00.01 grep quicknotes

Decision: QuickNotes is running successfully (PID 2376). The second line is just the grep command itself.

Is it listening on port 8080?
COMMAND    PID USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
quicknote 2376 i      5u   IPv6 ...        0t0  TCP *:http-alt (LISTEN)

Decision:
Port 8080 is actively bound and listening. The service is reachable at the socket layer.

Is HTTP reachable?
200

Decision:
HTTP health endpoint is reachable and responding correctly. The service is functional at L7 (application layer).

Firewall check
Nil
Decision:
No firewall rules are blocking local traffic. On macOS, iptables/nft are not used in this context.

DNS resolution
Nil
Decision:
dig +short localhost returned empty, which is expected on some macOS configurations where localhost resolution is handled internally via /etc/hosts rather than DNS queries.

Task 2.4
Full Outside-In Debugging Chain (Commands Run)

ps -ef | grep quicknotes
ss -tlnp | grep 8080
# (macOS equivalent used: lsof -iTCP:8080 -sTCP:LISTEN)
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/health
sudo iptables -L -n -v 2>/dev/null || sudo nft list ruleset 2>/dev/null || true
dig +short localhost


Root Cause

The failure was caused by:
bind: address already in use
This occurred when two instances of QuickNotes attempted to bind to the same port (:8080) simultaneously.

Mini Postmortem
This incident was caused by a port binding conflict where multiple instances of QuickNotes attempted to listen on the same TCP port (:8080). This is a common class of failure in local development and distributed systems where resource contention is not coordinated.

From a systems perspective, the issue is not tied to a single developer mistake but to the lack of safeguards preventing duplicate process launches. This type of failure often emerges in environments where manual process management is used instead of orchestrated service control.

Preventative tooling could include process managers (e.g., systemd, supervisord, or Docker Compose), which enforce single-instance guarantees and manage port allocation. Additionally, health checks and startup guards could detect existing listeners before attempting to bind. Observability tooling such as structured logs and startup diagnostics would also make these conflicts faster to identify.

Overall, this is a systemic coordination problem between processes competing for shared network resources rather than an application logic failure.


Bonus:

## Bonus — TLS Handshake via Reverse Proxy

A Caddy reverse proxy was placed in front of QuickNotes to terminate TLS on port 8443 while forwarding traffic to the HTTP service on port 8080.

### Architecture
Client → TLS (8443) → Caddy → HTTP (8080) → QuickNotes

### Observation
Packet capture on port 8443 showed TLS handshake messages instead of plaintext HTTP. The captured sequence included:

- Client Hello
- Server Hello
- Certificate exchange
- Encrypted application data

This demonstrates how TLS encrypts application traffic on the wire, making HTTP content invisible at the packet level once encryption is introduced.

### Insight
Unlike the earlier HTTP-only capture (Task 1), application payloads are no longer readable in tcpdump. Only the handshake metadata is visible, showing the separation between transport security (TLS) and application logic (HTTP).