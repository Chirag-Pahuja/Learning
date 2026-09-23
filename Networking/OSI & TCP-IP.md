# Networking for DevOps - OSI Notes

## 1. OSI 7 Layers (bottom-up: P-D-N-T-S-P-A)

| Layer | Name | Examples |
|-------|------|----------|
| 1 | Physical | cable, WiFi, hub |
| 2 | Data Link | MAC, Ethernet, switch, ARP |
| 3 | Network | IP, routing, ping, ICMP, MTU |
| 4 | Transport | TCP/UDP, ports, SYN/RST |
| 5 | Session | connections |
| 6 | Presentation | TLS/SSL, encryption |
| 7 | Application | HTTP, DNS, SSH, JSON, URL path |

> Trick: **P**lease **D**o **N**ot **T**hrow **S**ausage **P**izza **A**way

TCP/IP 4 layers: Application (7+6+5), Transport (4), Internet (3), Network Access (2+1)

Encapsulation for POST https://api.../order + JSON:
L7: POST /order + headers + JSON
L6: TLS encrypts whole L7 blob
L4: TCP src-port -> 443 + seq
L3: src-IP -> dst-IP
L2: src-MAC -> next-hop-MAC
L1: bits

## 2. L3 / L4 / L6 / L7 Debug Rule

L3: `ping 8.8.8.8`, `traceroute 8.8.8.8`
L4: `nc -vz <IP> 443`, `ss -tlnp` on server
L7: `curl -v http://<IP>:<port>/health`
TLS: `openssl s_client -connect host:443 -servername host`

```
ping fails -> L3
ping OK, nc fails -> L4
ping OK, nc OK, curl fails -> L7
dig fails, ping 8.8.8.8 works -> DNS only
```

504 Gateway Timeout = gateway OK (L4+L6 OK), upstream slow/dead (L7).
Isolate: `curl pod-IP directly`, compare gateway logs vs app logs.

## 3. L4 vs L7 Load Balancer

Same dst-IP (L3) + same dst-port 443 (L4), different URL path (L7):
- NLB L4: sees only IP+port. Encrypted payload opaque. Cannot route /images vs /api.
- ALB L7: terminates TLS (decrypts with cert), parses HTTP method/URL/headers, routes by path.

No cert + TLS passthrough = no decrypt = no L7 = no path routing.
Can still route by SNI hostname (clear in ClientHello), not by path.

## 4. TLS is L6

Order: L4 TCP connect -> L6 TLS handshake -> L7 HTTP
- `nc 443 OK` + `curl TLS handshake failure` = L4 OK, L6 fail. ALB never sees URL.
- `curl -k works, without -k fails verify` = L6 verification fail only. `-k` skips trust check, still encrypts. Never use in prod.
- `HTTP 80 works, HTTPS 443 reset` + `nc 443 connects then drops` = L4 OK, L6 reject (no cert, cipher/SNI mismatch).

## 5. Curl Timing

```bash
curl -o /dev/null -s -w "dns:%{time_namelookup} connect:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n" https://example.com
```

- dns = DNS. High = DNS/CoreDNS
- connect = TCP L4. High = network/firewall
- tls (appconnect-connect) = TLS L6. High = cert/cipher
- ttfb (starttransfer-appconnect) = server processing L7. High + connect low = app slow
- total = end to end

GOOD: dns:0.04 connect:0.06 tls:0.12 ttfb:0.20 total:0.21
APP SLOW: dns:0.02 connect:0.03 tls:0.08 ttfb:30.05 total:30.05

Fast: https://example.com
Slow 5s: https://httpbin.org/delay/5

## 6. MTU - L3 Size Issue

MTU = Maximum Transmission Unit, max one L3 packet, normal 1500.
L4 chops app data to ~1460 + 20 IP + 8 ICMP = fit.

Small GET fits, large POST needs fragment. With DF-bit set + ICMP blocked, large drops silently, retries forever = hang.

Prove:
```bash
ping -c 3 -D -s 1472 8.8.8.8  # 1472+28=1500, DF set
ping -c 3 -D -s 1372 8.8.8.8  # 1372+28=1400, fits MTU 1400
ping -c 3 -s 1472 8.8.8.8     # no DF, allows fragment
tracepath 8.8.8.8
```
Real case: path MTU 1400 (VPN). 1372 passes, 1472 fails with `frag needed and DF set (MTU 1400)`.
Fix: lower MTU to 1400, MSS clamping, allow ICMP fragmentation-needed, fix overlay MTU.

Rule: same L7, size decides pass/fail = L3 MTU.

## 7. ARP - L2 vs L3

ARP = L2 phonebook: who has IP? tell MAC.
- `arp -a | grep IP` shows `incomplete` = L2 fail, L3 never starts. IP missing/off/wrong subnet.
- ARP MAC OK + ping fail = L2 OK, L3 dropping (firewall/SG blocks ICMP). Test L4 with nc.
- Stale MAC after replace = L2 stale, not iptables. Fix: `sudo arp -d IP`, re-ping. Gratuitous ARP in prod.

Rule: No MAC = no L2. MAC OK + no ping = filtered, not missing.

## 8. Refused vs Timeout - L4

- ping OK + `connection refused` + `ss 127.0.0.1:8080` = L4 arrived, app only on localhost. Fix: listen 0.0.0.0:8080.
- ping OK + `timeout` + `ss 0.0.0.0:8080` = app open, SYN dropped by firewall. Check iptables/SG/NetworkPolicy.
- Refused = RST back, instant. Timeout = silence, hang.

Check:
```bash
ss -tlnp | grep 8080
iptables -L -n -v | grep 8080
kubectl describe networkpolicy
```

## 9. tcpdump - Read Packets

CCTV for L2/L3/L4 before firewall.
```bash
sudo tcpdump -i en0 -n -c 5
sudo tcpdump -i any -n port 8080
sudo tcpdump -i en0 -n tcp port 80
```

Read: `srcIP.port > dstIP.port: Flags [X]`
- [S] = SYN knock, [S.] = SYN-ACK open, [P.] = data push, [.] = ACK, [R] = reset/refused, [F] = finish
- UDP/QUIC has no flags, just length.

Example TCP+HTTP:
```
you > server.80: [SEW]       # knock L4
server > you: [S.E]          # open L4
you > server: [.]            # enter
you > server: [P.] GET /     # L7 request
server > you: [P.] 200 OK    # L7 reply
you > server: [F.]           # bye
```

SYN in, no SYN-ACK out = receiver filter. SYN + SYN-ACK = path OK.

## 10. DNS is L7 (not L3)

* `ping 8.8.8.8 OK` + `ping google.com cannot resolve` + `dig timeout` + `/etc/resolv.conf` empty = **L7 DNS**, not L3. L3 proven by IP.
* Don't run `nc/openssl/curl google.com` — all need IP, will fail same `cannot resolve`.
* Fix: `echo "nameserver 8.8.8.8" > /etc/resolv.conf` → `dig google.com` → `ping google.com`
* Rule: *IP works + name fails = L3 done, L7 DNS missing.*

MSS in dump: `1300` = clamped for MTU 1400 (1300+40=1340 fits), `1460` = no clamp (1460+40=1500).

## 11. L4 Port-Specific Filter (not L3 host down)

* Same IP, `nc 80 OK` + `nc 443 timeout` = L3 OK, **L4 per-port DROP** (not host down). L3 blocked → both fail.
* Prove: `nc -vz host 80` vs `443` + `sudo tcpdump -i any -n port 443` (SYN out, no SYN-ACK back).
* Check: egress firewall, SG, `iptables -L -n | grep 443`, corporate proxy :8080.

## 12. Captive Portal - L7 Hijack (Hotel WiFi)

* `curl http://example.com` → `302 captive.portal/login` + `nc 443 timeout` + DNS OK = **L7 intercept**, not DNS/firewall.
* Why: 80 plain HTTP hijacked to login page, 443 TLS can't be hijacked → silently dropped until auth.
* Fix L7: open browser `http://example.com` → login → portal allows 443 → retry `nc 443`/`curl https`.
* Line: *80 redirect + 443 drop = captive portal. Auth, don't edit firewall.*

## 13. MTU Trap - Don't Blame First Anomaly

Timeline breaks tell truth, not first low MTU:

| mss | tls | ttfb | Root | Why |
|-----|-----|------|------|-----|
| 1300 | 0.09s fast | 29.8s slow | **L7 app** | MSS already fits 1400, 4K cert passed fast → MTU mitigated, ttfb slow only |
| 1460 | 28.5s slow | 29s slow | **L3 MTU x L6** | MSS 1460+40=1500 >1400, large TLS certs drop, tls hangs |
| 1460 | 0.09s fast | 29s slow | **L7 app** | Despite 1460, cert small/passed → tls fast proves MTU not hit this path |

Real proof: `ping -D -s 1472` fails `frag needed and DF set (MTU 1400)` but `mss 1300` in SYN means fix active.
Rule: *fast tls + slow ttfb = app, even with low bridge around. tls slow = MTU/TLS, not app.*

## 14. L5 Session via L7 Cookie vs L4 IP-Hash

* Concept L5 (same conversation sticks) → real world via **L7 cookie** `Set-Cookie: sess=abc`. ALB reads L7 cookie → pin to backend A. With cookie → always A, without → round-robin A/B/C.
* L4-only (NLB, passthrough) can't see cookie (L6 encrypted + L7 hidden) → fallback **src-IP hash**: `hash(IP)%N = backend`. Breaks behind NAT (all same IP → same backend).
* TLS passthrough (no cert, no decrypt) = blind to L7 → **no cookie**, only L4 hash.
* Mobile CGNAT rotates IP every req (`1.1.1.1 → 1.1.1.9`) → L4 hash flips A→C→B → session lost. L7 cookie would fix (stable ID regardless of IP) → needs decrypt/termination.
* Line: *Cookie = L7 doing L5 job. No decrypt = no L7 cookie = L4 hash only. Wristband vs address.* **Cookie is L7 (HTTP header), IP hash is L4 — don't mix L3. L3 is host/IP down, not stickiness.*

## 15. Health Checks - Active L7 vs L4

Backend: `nc 8080 OK` + `ss 0.0.0.0:8080 LISTEN` + `curl /api 200` + `curl /health 500` → ALB `unhealthy` → `502 Bad Gateway` (no healthy targets).

* Active health probes **only `GET /health`** (synthetic), not `GET /api` (real). `health 200` + `api 500` → health stays healthy blind → needs outlier to watch real `500`. Path difference, not IP.
* L4 TCP check (SYN) → sees port open → marks **healthy** → routes traffic → hides L7 `500` → false healthy, dangerous. `nc OK ≠ app OK`.
* L7 HTTP check (`GET /health expect 200`) → sees 500 → correctly `unhealthy`.
* Recovery delay: `Interval 10s × HealthyThreshold 3 = 30s` to flip unhealthy→healthy. Fix `/health →200` at T0 → T10 1/3 → T20 2/3 → T30 3/3 → healthy, 502 clears. Debounce prevents flap. UnhealthyThreshold 2 → 20s to mark down.

## 16. Retry Storm

One slow backend C `ttfb 29s` + others A/B `0.05s`, client 3× retry no backoff:

* `req1→C` hangs 29s → 504 → retry1→A 0.05s OK but client waited 29s.
* At scale 33 of 100 req/s hit C → 33×29s held + 99 retries slammed on A/B → A/B thread pool spikes → cascade.
* Fix L7: `timeout 2s < ttfb`, exponential backoff + jitter, circuit breaker, outlier ejection, retry only idempotent + 503 not 504.

## 17. Outlier Ejection - Passive L7 Health

Active health = lab test (`GET /health` every 10s). Misses `1/5 /api 500` while `/health 200`.

Outlier = field test, watches **real traffic 5xx**:

* Config: `consecutive_5xx:5 interval:10s base_ejection_time:30s max_ejection_percent:50%`
* Interval is **sweep timer** (bucket), not instant. 5 fails at 0-2s → sit until t=10s sweep → eject. 6th req at 3s still hits C. One 200 resets consecutive.
* Eject 10s → 30s quarantine (no traffic) → t40s one trial req → 200→ clear, return full weight; 500→ re-eject 60s (×2). Gradual ramp only if `slowStartWindow` enabled.
* `max_ejection 50%` caps: 3 hosts → max 1 ejected, 20 hosts → max 10. Prevents ejecting sick hosts from overloading remaining: 10×50=500 rps left vs burst 100 → OK, but 1×50=50 left vs 100 → cascade to 0. Ejection protects pool, **not** scale — burst 4000 vs 1000 total needs autoscale/rate-limit, not ejection.

Istio:
```yaml
outlierDetection: { consecutive5xxErrors: 5, interval: 10s, baseEjectionTime: 30s, maxEjectionPercent: 50 }
```

## 18. Circuit Breaker vs Outlier

Both L7, different guard:

* **Breaker (bouncer)**: *Count now* — `max_connections:50`. 70/100 sticky to C → 50 in, 51-70/100 instant `503 circuit_breaker`, no queue → prevents OOM. Triggers on concurrency, before failure. Resets instantly when slot frees.
* **Outlier (manager)**: *Errors over time* — 5×500 in 10s → eject at 10s tick, quarantine 10-40s (30s), trial at 40s (10+30). If trial 200→back, 500→re-eject 60s (×2). Triggers on failures, after failure.
* Need both: breaker stops flood (now), outlier removes consistently bad host (10s). Breaker resets when slot frees; outlier resets after trial success.

Sticky = all 70/100 to same shop C due to cookie (L7) / hash (L4), otherwise even spread 33 each → no breaker. Cookie L7 ≠ IP L4 ≠ host L3.

## 19. Glossary

* **MTU** = Maximum Transmission Unit — max L3 packet (1500, 1400 with VPN)
* **MSS** = Maximum Segment Size — max TCP payload (1460 = 1500-20 IP-20 TCP, 1300 = clamped for 1400 MTU)
* **TLS** = Transport Layer Security — L6 encryption (ex-SSL), handshake after TCP
* **TTFB** = Time To First Byte — `curl time_starttransfer - time_appconnect`, server processing L7; `total` = end-to-end
* **TTL** = Time To Live — hop limit; **ICMP** = Internet Control Message Protocol — ping/frag-needed
* **ARP** = Address Resolution Protocol — IP→MAC, L2; `incomplete` = L2 fail
* **SNI** = Server Name Indication — hostname in ClientHello (clear before TLS)
* **CGNAT** = Carrier-Grade NAT — carrier rotates public IP per req (mobile)
* **ALB** = Application LB — L7 (HTTP path/cookie); **NLB** = Network LB — L4 (IP+port, SNI only)
* **RPS** = Requests Per Second; **503/429** = circuit breaker, **502** = no healthy host, **504** = gateway timeout (upstream slow)
