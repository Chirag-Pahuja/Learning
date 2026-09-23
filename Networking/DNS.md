# DNS Fundamentals — Stage 1

> Stages 1–3 comprehensive coverage in progress. See `OSI & TCP-IP.md:10` — DNS is L7.

## Structure
- Part 1: Core Theory (phonebook, tree, recursive vs iterative, records A/AAAA/CNAME/TXT)
- Part 2: TTL, migration, GTM
- Part 3: Failure modes + runbook
- Labs 1-9

## 7. DNS Basics (Recap)
DNS translates domain names to IP addresses.

Key Points:
- DNS uses UDP port 53 (512B) and TCP 53 for large responses
- Resolution is hierarchical: Root → TLD → Authoritative
- Caching happens at every layer (browser, OS, resolver)
- TTL determines cache duration
- DNS is L7 (see ./OSI\ \&\ TCP-IP.md:10)

## 8. DNS Record Types

| Type | Purpose | Example |
|------|---------|---------|
| A | IPv4 address | example.com → 93.184.216.34 |
| AAAA | IPv6 address | example.com → 2606:2800::1 |
| CNAME | Alias | www.example.com → example.com |
| MX | Mail server | example.com → mail.example.com |
| TXT | Text (SPF, DKIM, verification) | v=spf1 include:_spf.google.com |
| NS | Nameserver | example.com → ns1.example.com |
| SOA | Start of Authority | Zone metadata |
| PTR | Reverse DNS | 34.216.184.93 → example.com |
| SRV | Service location | _https._tcp.example.com |

## 9. DNS Resolution Process

```
1. Browser checks its cache
2. OS checks /etc/hosts and its cache
3. Query sent to recursive resolver (e.g., 8.8.8.8, ISP)
4. Resolver checks its cache
5. If not cached, resolver queries:
   a. Root server (.) → referral to com. (13 logical roots)
   b. TLD server (.com) → referral to example.com's NS
   c. Authoritative server → returns A record
6. Resolver caches and returns to OS
7. OS caches and returns to browser
8. Browser connects to IP
```

## Notes
- Resolver cache hit ratio matters (95% vs 50%)
