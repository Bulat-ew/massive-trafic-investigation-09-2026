# IPs

Slice: `BlockChrome125UserAgent`, 197 BLOCK samples, one IP each.
Counts are rule samples, not the mix of all traffic. Raw numbers: `facts.md`.

## What the spread looks like

- 197 IPs, no repeats in this rule. None of them appear in other Chrome/125 rows in the extract.
- /24s are thin: 142 /24s; 96 with one IP; largest occupancy is 4 (`176.20.194.0/24`, `9.142.183.0/24`). One adjacent pair: `9.151.192.204` / `9.151.192.205`.
- Concentration is at /16 and ASN. Top eight /16s hold 113 / 197 IPs. Same /16 does not map to one URI family.
- RIPEstat (2026-09-01 ~02:01 UTC): 20 ASNs, 80 aligned prefixes. Top four ASNs hold 124 IPs: Web2Objects (`AS62874`) 37, M247 (`AS9009`) 35, Datacamp CDNEXT (`AS212238`) 30, Ace Data Centers (`AS11798`) 22. Also present: NET3-AI, RCN, Leaseweb USA (three ASNs, 14 IPs), Oxylabs (`AS396319`) 5, Comcast (two ASNs, 5 IPs).

## Relation to H1–H4

- H4 is consistent: many /24s, hosting/proxy-shaped holders, one shared client fingerprint. whatismyipaddress `Services` on the same 197 IPs: VPN Server 131, Data Center/Transit 59, Public Proxy Server 7.
- H2 as a single crawler egress is weak.
- H1 vs H3 is not decided by IP layout. A proxy fleet can scrape or flood.
- WAF `Country` on this rule is mostly US (182/197). That is a WAF field, not a proven residential location. Page cities are mostly hosting metros, not a residential US mix.

## Other IPs (not this rule)

- `52.65.126.156` (AU) appears on 23 of 26 AU-labeled Chrome/125 rows, with `X-Forwarded-For` hops on `directory` / `www` / `api`. Those 23 rows have 18 distinct XFF first hops; none are in the 197 block-rule IPs. Treat as origin or internal forwarder, not one client walking many URLs, until origin logs say otherwise.
- Six `195.40.*` IPs (SG, `Block-CN-SG-Scrapers`) are the only HTML-only Chrome/125 clients sampled on more than one catalog URL. Gaps are minutes, not same-second floods.
- Chrome/120 samples concentrate on `3.24.246.204` / `mainsite.3.24.246.204`. Separate stream until overlap is shown.

## Next measurements

- Rate per IP and distinct-URI count per IP over the full surge (WAF logs or Varnish, not `GetSampledRequests`).
- Confirm whether the 197 IPs appear before 10:41 and at what QPS.
- Keep RIPEstat/whois dated; prefixes move.
