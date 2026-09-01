# Hypotheses

Open set. More than one can be true. Status is against `facts.md` only.
None of these is selected as the cause.

| ID | Claim | Status |
|---|---|---|
| H1 | Targeted crawler / scraper of wedding catalog content | Consistent with URI mix; volume and client spread still open |
| H2 | Accidental aggressive crawler (misconfigured bot / AI agent) | Weak as a single-egress crawler; possible if it rented a proxy fleet |
| H3 | Intentional L7 DDoS | Volume and drop-after-block fit; URI mix does not look like a single-URL flood |
| H4 | Botnet or rented-proxy fleet as the delivery layer | IP/ASN spread fits; does not by itself pick intent (H1 vs H3) |

---

## H1 targeted crawler / scraper

**Claim.** Clients are harvesting public catalog data: supplier profiles, brochures, reviews, enquiry forms, travel listings.

**Would support.** Many distinct catalog URIs; Australian location/supplier parameters; little or no static-asset flood; persistence across hours; content-bearing GET paths.

**Would weaken.** One hot URL; POST/login/checkout smash; no catalog coverage; identified search-engine bots as the main volume.

**Against current evidence.**

- Fits: `BlockChrome125UserAgent` samples are 197 unique URIs in a few site templates (enquiry, directory, brochure, reviews, travel). No `/api/`, no static assets, no POST, no Referer in that slice. Same HTML-only, no-Referer shape is in `Block-CN-SG-Scrapers` Chrome/125 before the UA block. Six SG `195.40.*` IPs in that rule are sampled on different catalog families 5–66 minutes apart. Cross-IP, instances are almost unique (123 enquiry URIs on 123 IPs). See `analysis/uris.md`.
- Chrome/125 API rows are page-boot XHR (`login-status`, `get-categories`, `chat-messages`, …) with Referer, not a separate catalog API crawl. They do not share IPs with the 197 block-rule samples. After 10:41 Melbourne they sit in `waf` ALLOW (no US). That is consistent with “list walk of HTML URLs; follow-ups only if the document is allowed.”
- Open: share of the 1.1M–1.4M plateau that is Chrome/125 is not measured. Per-IP rate is not measured (one sample per IP in that rule). Whether blocked clients requested API/static at all is not in full logs.

---

## H2 accidental aggressive crawler

**Claim.** A single misconfigured crawler or AI agent over-fetched the site without a harvest or attack goal.

**Would support.** One ASN / small IP set; bot or tool User-Agent; sitemap/robots or shallow URL walk; short burst then stop or retry-after-error.

**Would weaken.** Many unrelated proxy/hosting ASNs; one spoofed browser UA; 30+ hour plateau; deep parameterized catalog URLs.

**Against current evidence.**

- Weak as “one crawler from one place”: 197 IPs, 20 ASNs, 142 /24s in the block slice. See `analysis/ips.md`.
- Named bots in the extract (AhrefsBot, MJ12bot, Applebot, Bytespider) are a separate slice; their share of surge volume is not measured.
- Empty `Accept-Language` on all Chrome/125 samples can fit a headless client, but that is also compatible with H1/H3/H4.

---

## H3 intentional L7 DDoS

**Claim.** The goal is to exhaust edge/origin capacity, not to collect listings.

**Would support.** Sudden step-up to a high plateau; traffic falls when a cheap block is applied; uncacheable or cache-busting URLs; little useful content diversity; repeats of expensive endpoints.

**Would weaken.** Low request rate per client; cacheable GETs that Varnish absorbs; crawl-like unique-URL coverage with business parameters.

**Against current evidence.**

- Fits: dashboard TOTAL jumps ~100K → ~1.4M near 2026-08-31 02:00 Melbourne and stays on a plateau ~32 hours. After `BlockChrome125UserAgent`, the one-hour chart drops ~23K → ~3K–3.5K at 10:41 on 2026-09-01. See `timeline.md`.
- Does not look like a classic single-URL flood in the block samples: 197 raw URIs, 143 paths.
- No second User-Agent or new flood is in the sources after the UA block (HAR ends ~30 minutes later). Adaptation is not observed in that window; it is also not ruled out later.
- Cache vs origin cost is not measured (no Varnish/origin logs). See `analysis/cache-cost.md`.

---

## H4 botnet / proxy fleet

**Claim.** Many distinct clients (compromised hosts or rented residential/datacenter proxies) send the Chrome/125 stream.

**Would support.** Many IPs, many /24s, hosting/proxy/VPN ASNs; one shared fingerprint; geo that does not match the site’s normal audience.

**Would weaken.** One or two egress IPs; a single legitimate ISP; browser diversity that matches real users.

**Against current evidence.**

- Fits a rented/hosting proxy fleet: 197 unique IPs in the block slice; /24s mostly singletons; RIPEstat holders include Web2Objects, M247, Datacamp CDNEXT, Ace Data Centers, Oxylabs, Leaseweb. Same exact Chrome/125 UA. Empty `Accept-Language`. See `analysis/ips.md` and `analysis/fingerprints.md`.
- Weaker as a conventional compromised-device botnet: holders in this slice are mostly datacenter/proxy, not many consumer ISPs (Comcast is 5 IPs).
- Does not choose intent. A proxy fleet can carry H1 or H3.
- WAF `Country=US` is not proven as a real user location.

---

## Mitigation mechanism (not a cause)

Operator (2026-09-01): Chrome/125 is not a hard deny of every match; about 100 requests / 5-minute window still pass. That would explain leftover ALLOW without a second stream.

Against current evidence:

- Fits the rule-sample shape: `BlockChrome125UserAgent` rows are all BLOCK. A rate-based rule only samples keys that already exceeded the cap.
- Does not fit the 51 `waf` ALLOW Chrome/125 rows as “the same US HTML flood under the cap”: 0 IP overlap with the 197 block-rule IPs, no US, 35/51 are API/`/v5/`, 17/51 are origin IP `52.65.126.156`. See `analysis/allow-after-block.md`.
- Named `Ratelimit-*` rules in the ACL produced no samples in this HAR. Rule JSON is not in the repo.

This does not pick H1–H4.

## Not a current hypothesis

Chrome/120 on hosts `3.24.246.204` / `mainsite.3.24.246.204` is a separate WAF slice. It is not assigned to H1–H4 until overlap is shown.
