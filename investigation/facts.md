# Facts

Investigation record for the traffic surge on `easyweddings.com.au`.
This file lists observations only. It does not assign a cause.

Timezone used on the traffic dashboards: `Australia/Melbourne` (AEST, UTC+10 in these dates).
AWS WAF sample timestamps are stored in UTC.

---

## Sources

| Source | What it is | Time coverage |
|---|---|---|
| Operator statements (2026-09-01) | Direct report from the team running the site | Qualitative; not independently measured here |
| Operator-provided traffic dashboard screenshots | Request series labeled TOTAL / ALLOW / BLOCK | 2026-08-29 12:00 – 2026-09-01 ~11:00 Melbourne; plus 2026-09-01 09:55–10:55 Melbourne |
| `data/waf/ap-southeast-2.console.aws.amazon.com.har` | Chrome HAR of AWS WAF console `GetSampledRequests` calls | Query window 2026-08-31 22:12:39 UTC – 2026-09-01 01:12:39 UTC (2026-09-01 08:12–11:12 Melbourne) |
| `data/waf/sampled_requests.csv` / `data/waf/sampled_requests.jsonl` | Deduplicated rows extracted from that HAR by `data/waf/extract_sampled_requests.py` | Same window; 1720 rows |
| RIPEstat `prefix-overview` (queried 2026-09-01 ~02:01 UTC) | BGP-aligned ASN and announced prefix for each of the 197 `BlockChrome125UserAgent` client IPs | Lookup ~1 hour after that rule’s sample timestamps |

The HAR targets regional Web ACL `waf-prod` in `ap-southeast-2`:

`arn:aws:wafv2:ap-southeast-2:018844529345:regional/webacl/waf-prod/55deb26c-464c-4992-9e92-cdb265e78ee9`

WAF sampled requests are drawn per rule. They are not a random sample of all traffic. Counts below must not be read as overall traffic share.

The WAF sample window starts after the surge had already been running for more than 30 hours. It does not cover the start of the surge.

---

## Operator-reported

The following were stated by the operator on 2026-09-01. They are not derived from the files in this repo.

- The site `easyweddings.com.au` normally receives predominantly local Australian traffic.
- The increased traffic during this incident was mainly from the United States.
- The team blocked User-Agent `Chrome 125.0.0.0`.
- After that block, request volume dropped and returned to previous levels.
- On 2026-09-01 the operator also stated that matching requests are not all denied: there is an allowance of about **100 requests per 5-minute window**, and that is why some Chrome/125 requests still pass. The operator did not name the rule, the aggregate key (client IP vs `X-Forwarded-For`), or the evaluation window type. This was not read from a Web ACL export.

---

## Traffic volume (dashboards)

### Multi-day chart (2026-08-29 ~12:00 – 2026-09-01 ~11:00 Melbourne)

- Series: TOTAL, ALLOW, BLOCK.
- From 2026-08-29 through 2026-08-31 ~02:00, TOTAL stays near the 100K mark on the chart scale.
- Tooltip at **2026-08-31 01:53** Melbourne: ALLOW 84.48K, TOTAL 84.79K, BLOCK 309.
- Around **2026-08-31 02:00** Melbourne, TOTAL and ALLOW rise sharply to about **1.4M** on the chart scale.
- From then until 2026-09-01 morning, TOTAL stays on a plateau of about **1.1M–1.4M**. ALLOW overlaps TOTAL. BLOCK stays near the baseline relative to that scale.
- On 2026-09-01, near the right edge of this chart, TOTAL/ALLOW turn downward (still well above the 29–30 August level on this screenshot).

The dashboard does not label the bucket size on this chart. The 1.1M–1.4M plateau and the ~20K readings on the one-hour chart are consistent with hourly vs per-minute aggregation of the same volume, but the bucket size is not printed on the screenshots.

### One-hour chart (2026-09-01 09:55–10:55 Melbourne)

- From 09:55 to 10:41, TOTAL fluctuates mostly between about **18K and 24K**, with a peak near **28K** around 10:27. One brief dip near 10:02 falls to about 4K and recovers.
- ALLOW (or the equivalent non-block series) tracks TOTAL during this period. BLOCK stays near 0.
- At about **10:41**, TOTAL falls from about **23K to about 3K–3.5K**.
- From 10:42 to 10:55, TOTAL stays about **3K–4K**. The non-block series is about **2K**. BLOCK rises from near 0 to about **1K**.

---

## Mitigation that is in the WAF data

Web ACL `waf-prod` contains a rule named `BlockChrome125UserAgent`.

The HAR in this repo contains only `GetSampledRequests` calls. It does not contain `GetWebACL` (or any other call) with the rule statement, so whether `BlockChrome125UserAgent` is a hard BLOCK or a rate-based rule is not in the files. `MaxItems: 100` on those API calls is the sample-page size, not a rate limit.

`GetSampledRequests` for that rule in the HAR window:

- `PopulationSize`: 5000 (same value is also returned for several other rules in this HAR; it is the API field, not an independently counted total).
- 197 sampled requests, all `BLOCK`.
- Sample timestamps: **2026-09-01 00:47:11 – 00:54:24 UTC** (10:47–10:54 Melbourne).
- That is after the 10:41 Melbourne drop on the one-hour chart.

Every one of those 197 samples has this exact User-Agent:

```
Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36
```

---

## Chrome/125.0.0.0 in WAF samples

Across all extracted rules, 433 rows have `Chrome/125.0.0.0` in User-Agent. All 433 use the same string as above.

| Field | Value in those 433 rows |
|---|---|
| Unique client IPs | 396 |
| Methods | GET 432, OPTIONS 1 |
| HTTP versions | HTTP/2.0 399, HTTP/1.0 22, HTTP/1.1 12 |
| `Accept-Language` | empty on all 433 |
| `Origin` | empty on 432; one `https://www.easyweddings.com.au` |
| `Referer` present | 48 / 433 |
| Hosts | `www.easyweddings.com.au` 356; `easyweddings.com.au` 30; `directory.easyweddings.com.au` 22; `travel.easyweddings.com.au` 22; `api.easyweddings.com.au` 3 |
| First sample | 2026-08-31 22:51:17 UTC (09:51 Melbourne) |
| Last sample | 2026-09-01 01:12:24 UTC (11:12 Melbourne) |

Country field on those 433 rows (WAF `Country`, rule-biased):

| Country | Rows |
|---|---|
| US | 189 |
| SG | 139 |
| CN | 36 |
| AU | 26 |
| RO | 17 |
| GB | 10 |
| DE | 7 |
| BR | 5 |
| HR | 2 |
| BE | 1 |
| NL | 1 |

Most frequent URI paths among those 433 rows:

| Path (query stripped) | Rows |
|---|---|
| `/wedding/enquiry/mobile` | 125 |
| `/reviews/user/...` (prefix) | 58 |
| `/api/EasyDirectoryAPI` and `/api/easydirectoryapi` | 37 |
| `/activities` | 15 |

`/wedding/enquiry/mobile` query parameters in samples include supplier categories and Australian locations (examples: Melbourne, Sydney, Daylesford, HunterValley, Ballarat, Canberra).

### Chrome/125 API and page-boot follow-ups

Source: same `sampled_requests.jsonl` extract. Counts are rule samples, not traffic share.

43 of 433 Chrome/125 rows are API-shaped (`/api/` in the path, or host `api.easyweddings.com.au`). 41 of those 43 have a Referer. None of those 43 appear under `BlockChrome125UserAgent`. No IP from that rule’s 197 samples appears on a Chrome/125 API row.

| Path (query stripped) | Rows |
|---|---:|
| `/api/EasyDirectoryAPI/login-status` | 13 |
| `/api/easydirectoryapi/get-regions-grouped` | 7 |
| `/api/EasyDirectoryAPI/chat-messages` | 6 |
| `/api/EasyDirectoryAPI/get-categories` | 5 |
| `/api/easydirectoryapi/get-logged-in-user-details` | 5 |
| `/api/dashboardapi/load-shortlisted-suppliers` | 3 |
| `/v5/ew.Couples.Api/ai/agent/chat` | 2 |
| `/api/EasyDirectoryAPI/is-logged-in` | 1 |
| `/v5/ew.directory.api/supplier/stats` | 1 |

Hosts on those 43: `www.easyweddings.com.au` 20, `directory.easyweddings.com.au` 20, `api.easyweddings.com.au` 3. Methods: GET 42, OPTIONS 1. Rules: `waf` ALLOW 35, `AWS-AWSManagedRulesWindowsRuleSet` COUNT 8.

Referer bases (query stripped) on the 41 rows that have one: `/wedding/enquiry/mobile` 24; remaining 17 are directory / reviews / articles / homepage paths on `www.easyweddings.com.au`.

Country on the 43: AU 24, RO 7, GB 6, US 4, DE 1, HR 1. The 4 US rows are two client IPs, both before the 10:41 Melbourne drop, both `WindowsRuleSet` COUNT:

| Time (UTC) | IP | Path | Referer page |
|---|---|---|---|
| 2026-08-31 22:51:19 | `107.174.117.26` | `login-status`, `get-categories`, `chat-messages` (3 rows, same second) | `/wedding/enquiry/mobile?...` with `name`, `phone`, `waitfor delay` in query |
| 2026-08-31 23:28:25 | `31.76.160.73` | `get-categories` | same enquiry shape, different `category` / `supplier` |

The same two IPs have `SQLiRuleSet` COUNT rows on `/wedding/enquiry/mobile` 1–2 seconds earlier. `107.174.117.26` also has a parallel `directory` + `52.65.126.156` copy of that enquiry at 22:51:17.

The other 39 API rows are after 10:41 Melbourne, all `waf` ALLOW except that they sit in the 51-row ALLOW slice already listed below. 20 of those 39 are client IP `52.65.126.156` on `directory` / `api`.

7 Chrome/125 rows are static assets (`.js` / `.css`). All 7 are `waf` ALLOW after 10:41 Melbourne, all have a Referer to a catalogue or article page, countries RO/GB/BE/BR/NL. None are US. None are under `BlockChrome125UserAgent` or `Block-CN-SG-Scrapers`.

`Block-CN-SG-Scrapers` Chrome/125: 172 rows, all BLOCK, hosts `www` only, 0 API, 0 static, 0 Referer (147 before 10:41 Melbourne, 25 after).

HTML document rows (not API, not static) among Chrome/125 have Referer on 0 rows.

`/wedding/enquiry/mobile` Chrome/125 query keys: `category` / `location` / `supplier` / `fromProfile` on all 125. Extra keys `name`, `phone`, `message`, `venue.haveVenue` appear on 5 rows only — those are the `SQLiRuleSet` COUNT rows. The 55 enquiry rows in `BlockChrome125UserAgent` do not have `name` / `phone` / `message`.

Enquiry `returnUrl` is present on 106 / 125 Chrome/125 enquiry rows. Those decode to 95 distinct paths. Path-segment counts on the decoded `returnUrl` (query stripped): 3 segments 61, 2 segments 35, 1 segment 10. Values are directory listing or `/wedding/{suburb}/{n}/` paths (examples: `/WeddingCars/Bronte/`, `/wedding/Thomastown/10/`).

### Chrome/125 per-IP URL repeats

Source: same `sampled_requests.jsonl` extract. A unique request is `(timestamp, ip, method, host, uri)`. Chrome/125: 433 unique requests, 396 client IPs.

385 of 396 Chrome/125 IPs appear on exactly one unique request. 11 IPs appear on more than one.

`BlockChrome125UserAgent`: 197 IPs, 197 unique raw URIs, 0 IP repeats, 0 overlap with the other 199 Chrome/125 IPs in this extract.

`Block-CN-SG-Scrapers` Chrome/125: 172 rows, 165 IPs, 6 IPs with more than one row. Those 6 are all `195.40.*` / country SG. Gaps between that IP’s sampled rows: 306–3984 seconds. Families on those repeats: enquiry, review user, review supplier, directory profile, article, gallery. Referer empty on all 6 IPs.

Cross-IP uniqueness among all Chrome/125 rows (path or ID reused across IPs is rare except the enquiry path itself):

| Family | Rows | Unique IPs | Unique instance key |
|---|---:|---:|---|
| `/wedding/enquiry/mobile` | 125 | 123 | 123 raw URIs |
| Directory profile (`/{Category}/{Location}/{Slug}`) | 75 | 75 | 75 paths |
| `DownloadBrochure` | 62 | 62 | 61 numeric IDs |
| `/reviews/user/...` | 58 | 58 | 58 IDs (no ID reuse) |

The 11 multi-request Chrome/125 IPs:

| IP | Unique requests | Unique paths | Country | Rules | Time span |
|---|---:|---:|---|---|---|
| `52.65.126.156` | 23 | 9 | AU | `waf`, `SQLiRuleSet`, `WindowsRuleSet` | 22:51–01:12 UTC |
| `107.174.117.26` | 4 | 4 | US | `SQLiRuleSet`, `WindowsRuleSet` | 22:51:17–22:51:19 UTC |
| `195.40.173.252` | 3 | 3 | SG | `Block-CN-SG-Scrapers` | 23:32–00:32 UTC |
| `205.188.110.196` | 3 | 3 | RO | `waf` | 01:10:31 UTC (same second) |
| `159.148.248.172` | 3 | 3 | GB | `waf` | 01:11:37–01:11:41 UTC |
| `195.40.81.52` | 2 | 2 | SG | `Block-CN-SG-Scrapers` | 23:20–23:52 UTC |
| `195.40.64.101` | 2 | 2 | SG | `Block-CN-SG-Scrapers` | 23:24–00:30 UTC |
| `31.76.160.73` | 2 | 2 | US | `SQLiRuleSet`, `WindowsRuleSet` | 23:28:24–23:28:25 UTC |
| `195.40.172.190` | 2 | 2 | SG | `Block-CN-SG-Scrapers` | 00:20–00:36 UTC |
| `195.40.172.72` | 2 | 1 | SG | `Block-CN-SG-Scrapers` | 00:33–01:01 UTC |
| `195.40.90.231` | 2 | 2 | SG | `Block-CN-SG-Scrapers` | 01:01–01:06 UTC |

On the 23 Chrome/125 rows with client IP `52.65.126.156` that have `X-Forwarded-For`, the first hop is a public IP. 18 distinct first hops. None of those first hops are in the 197 `BlockChrome125UserAgent` IPs.

SQLi-shaped enquiry (`name=ugzWtPtK`, `message=555`, `phone` contains `waitfor delay '0:0:15'`) appears on 5 Chrome/125 rows: client IPs `107.174.117.26` (US), `31.76.160.73` (US), `172.121.212.11` (US), plus two origin copies on `52.65.126.156` whose XFF first hop matches the first two. Same `name` on all five. Suppliers differ: `BrowniesNow`, `HenkermanWeddingHangers`, `DisposableCameraCompany`. `172.121.212.11` is not in the 197 block-rule IPs.

### Rule `BlockChrome125UserAgent` only (197 BLOCK samples)

This rule is not a geo filter. It is the least geo-skewed Chrome/125 slice in the HAR.

| Country | Rows |
|---|---|
| US | 182 |
| RO | 7 |
| BR | 3 |
| DE | 2 |
| GB | 1 |
| HR | 1 |
| SG | 1 |

- Unique IPs: 197 (one sample per IP in this slice).
- Unique /8 prefixes: 25. Unique /16 prefixes: 35. Unique /24 prefixes: 142.
- /24 occupancy: 96 /24s with 1 IP, 39 with 2, 5 with 3, 2 with 4. Largest /24s: `176.20.194.0/24` (4) and `9.142.183.0/24` (4).
- Adjacent IPs (difference of 1): one pair, `9.151.192.204` and `9.151.192.205`.
- Top eight /16s hold 113 / 197 IPs. Same /16 does not map to one URI family.
- None of these 197 IPs appear in other Chrome/125 rows in this extract.

/16 counts in this slice (RIPEstat holder on that /16; some /16s mix holders):

| /16 | IPs | RIPEstat holder(s) in this slice |
|---|---:|---|
| `31.76.0.0/16` | 22 | Ace Data Centers (`AS11798`) |
| `9.151.0.0/16` | 15 | NET3-AI / Contact Consumers (`AS46635`) |
| `176.20.0.0/16` | 15 | Web2Objects (`AS62874`) |
| `87.56.0.0/16` | 13 | Web2Objects (`AS62874`) |
| `192.177.0.0/16` | 13 | Datacamp CDNEXT 11, EGIHosting 2 |
| `152.163.0.0/16` | 12 | Leaseweb USA PHX 7, Leaseweb USA SFO 4, M247 1 |
| `189.81.0.0/16` | 12 | M247 7, Datacamp CDNEXT 5 |
| `9.142.0.0/16` | 11 | RCN (`AS6079`) |
| `23.230.0.0/16` | 8 | Datacamp CDNEXT (`AS212238`) |
| `189.104.0.0/16` | 8 | Datacamp CDNEXT 5, M247 3 |
| `154.17.0.0/16` | 8 | M247 (`AS9009`) |
| `172.121.0.0/16` | 6 | GTT (`AS3257`) |
| `205.188.0.0/16` | 5 | M247 (`AS9009`) |
| `142.147.0.0/16` | 5 | Web2Objects (`AS62874`) |
| `23.239.0.0/16` | 5 | Oxylabs (`AS396319`) |
| 20 other /16s | 39 | see ASN table |

RIPEstat `prefix-overview` on 2026-09-01 ~02:01 UTC assigned all 197 IPs to 20 ASNs. For some addresses RIPEstat reported the exact resource was not announced and aligned the result to a less-specific prefix. Holder strings below are RIPEstat’s `holder` field.

| ASN | Holder (RIPEstat) | IPs | WAF Country on those rows |
|---|---|---:|---|
| `AS62874` | WEB2OBJECTS - Web2Objects LLC | 37 | US 35, DE 2 |
| `AS9009` | M247 M247 Europe SRL | 35 | US 26, RO 7, GB 1, HR 1 |
| `AS212238` | CDNEXT Datacamp Limited | 30 | US 30 |
| `AS11798` | ACEDATACENTERS-AS-1 - Ace Data Centers, Inc. | 22 | US 22 |
| `AS46635` | NET3-AI - Contact Consumers | 15 | US 15 |
| `AS6079` | RCN-AS - RCN | 11 | US 11 |
| `AS19148` | LEASEWEB-USA-PHX - Leaseweb USA, Inc. | 7 | US 7 |
| `AS3257` | GTT-BACKBONE GTT Communications Inc. | 6 | US 6 |
| `AS18779` | EGIHOSTING - EGIHosting | 5 | US 5 |
| `AS396319` | US-INTERNET-396319 - Oxylabs | 5 | US 5 |
| `AS7203` | LEASEWEB-USA-SFO - Leaseweb USA, Inc. | 4 | US 4 |
| `AS33667` | CMCS - Comcast Cable Communications, LLC | 4 | US 4 |
| `AS203020` | HostRoyale HostRoyale Technologies Pvt Ltd | 3 | BR 3 |
| `AS393886` | LEASEWEB-USA-MIA - Leaseweb USA, Inc. | 3 | US 3 |
| `AS7979` | SERVERS-COM - Servers.com, Inc. | 2 | US 2 |
| `AS263740` | Corporacion Laceibanetsociety | 2 | US 2 |
| `AS46261` | QUICKPACKET - QuickPacket, LLC | 2 | US 2 |
| `AS263821` | Soluciones Favorables | 2 | US 2 |
| `AS33651` | CMCS - Comcast Cable Communications, LLC | 1 | US 1 |
| `AS30058` | FDCSERVERS - FDCservers.net | 1 | SG 1 |

Three Leaseweb USA ASNs together: 14 IPs. Two Comcast ASNs together: 5 IPs. Unique RIPEstat-aligned prefixes: 80.

All 7 RO-labeled rows in this rule are `AS9009` M247. The one SG-labeled row is `195.40.64.164` / `AS30058`.

- Hosts: `www.easyweddings.com.au` 148, `easyweddings.com.au` 28, `travel.easyweddings.com.au` 21.
- Method: GET on all 197.
- Referer: empty on all 197.
- Accept-Language: empty on all 197.
- Unique raw URIs (query included): 197. Unique paths (query stripped): 143.
- No `/api/` paths in this rule’s samples.
- Query string present on 59 / 197 rows. 55 of those are `/wedding/enquiry/mobile`; the other 4 are travel host paths with `currency=` (`AUD`, `USD`, or `SGD`).

URI families in this rule’s 197 samples (path template, not traffic share):

| Family | Rows | Hosts | Path template |
|---|---:|---|---|
| Enquiry form | 55 | www 44, apex 11 | `/wedding/enquiry/mobile` |
| Directory profile | 45 | www 36, apex 9 | `/{Category}/{Location}/{SupplierSlug}` (optional trailing slash) |
| Brochure download | 38 | www 33, apex 5 | `/{Category}/{Location}/{numericId}/DownloadBrochure/` |
| Review user page | 25 | www 23, apex 2 | `/reviews/user/{id}` |
| Travel activity | 15 | travel 15 | `/activities/{slug}-{id}` |
| Article | 7 | www 5, apex 1, travel 1 | `/articles/...` |
| Travel accommodation | 3 | travel 3 | `/accommodation/detail/{slug}-{id}` |
| Directory gallery | 3 | www 3 | `/{Category}/{Location}/{SupplierSlug}/Gallery/` |
| Travel suggestion | 2 | travel 2 | `/suggestion/{slug}-{id}` |
| Review supplier page | 2 | www 2 | `/reviews/supplier/{slug}/` |
| Directory QA | 1 | www 1 | `/{Category}/{Location}/{SupplierSlug}/QA` |
| Area listing | 1 | www 1 | `/wedding/Daglish/` |

`/wedding/enquiry/mobile` query keys in all 55 rows: `category`, `location`, `supplier`, `fromProfile`. Also present: `returnUrl` 47, `isBrochureRequest` 42 (value `false` on all 42), `style` 1. `fromProfile` values: `False` 47, `True` 8. Distinct `category` 21, `location` 25, `supplier` 54. Location values in this slice are Australian place names (most frequent: Melbourne 17, Sydney 9, Adelaide 5).

All 38 `DownloadBrochure` paths have exactly four segments and a trailing slash. All 38 numeric IDs are distinct.

`/reviews/user/{id}`: 24 UUID-shaped IDs, 1 numeric ID (`143447`). All 25 paths are distinct.

First path segment counts (case as stored): `wedding` 56, `reviews` 27, `activities` 15, `WeddingVenues` 14, `MarriageCelebrant` 12, `WeddingPhotography` 10, `articles` 7, then smaller directory categories (`WeddingMusic`, `WeddingHair`, `WeddingCakes`, and others).

### Chrome/125 still ALLOW after 10:41 Melbourne

51 Chrome/125 rows have action `ALLOW`, all under rule `waf`, timestamps **2026-09-01 01:09:49 – 01:12:24 UTC** (11:09–11:12 Melbourne), after the drop.

Countries on those 51: AU 20, RO 10, GB 9, DE 5, SG 2, BR 2, HR 1, BE 1, NL 1. No US in this ALLOW slice. Unique client IPs in this slice: 31. Overlap with the 197 `BlockChrome125UserAgent` IPs: 0.

Path mix in this ALLOW slice (query stripped): `/api/` or `/v5/` 35; JS/CSS 7; `/wedding/enquiry/mobile` 4; `DownloadBrochure` 2; articles 2; one travel `/details/...` path.

20 of the 26 AU-labeled Chrome/125 rows are these ALLOW `waf` samples. 23 of 26 AU-labeled Chrome/125 rows have client IP `52.65.126.156` (AU) and, where `X-Forwarded-For` is present, an upstream public IP plus internal `10.20.*` hops on hosts `directory.easyweddings.com.au` / `www.easyweddings.com.au` / `api.easyweddings.com.au`. Of the 51 ALLOW rows, 17 are that origin IP (16 `directory`, 1 `api`); 16 of those 17 have XFF. Unique first-XFF addresses on those origin rows: 16. Three of those first-XFF IPs also appear as direct client IPs on other ALLOW Chrome/125 rows (`205.188.16.90`, `205.188.110.147`, `205.188.110.196`).

40 of the 51 ALLOW Chrome/125 rows have a Referer.

### Other rules that also sampled Chrome/125

- `Block-CN-SG-Scrapers`: 172 Chrome/125 rows, all BLOCK, countries SG/CN only (the rule is a geo/scraper block). `PopulationSize` 5000.
- `AWS-AWSManagedRulesWindowsRuleSet`: 8 COUNT.
- `AWS-AWSManagedRulesSQLiRuleSet`: 5 COUNT.

---

## Other WAF sample notes (same HAR, not a traffic census)

Extracted rows: 1720. Actions: COUNT 947, BLOCK 391, ALLOW 382.

Rules present in the extract:

| Rule | Samples | PopulationSize | Dominant action |
|---|---|---|---|
| `BlockChrome125UserAgent` | 197 | 5000 | BLOCK |
| `waf` | 196 | 5000 | ALLOW |
| `SizeRestrictions_BODY` | 195 | 1752 | COUNT |
| `Block-CN-SG-Scrapers` | 194 | 5000 | BLOCK |
| `NoUserAgent_HEADER` | 194 | 5000 | COUNT |
| `AWS-AWSManagedRulesCommonRuleSet` | 193 | 1054 | COUNT |
| `AWS-AWSManagedRulesWindowsRuleSet` | 187 | 458 | COUNT |
| `clients_whitelist` | 186 | 905 | ALLOW |
| `AWS-AWSManagedRulesKnownBadInputsRuleSet` | 169 | 447 | COUNT |
| `AWS-AWSManagedRulesSQLiRuleSet` | 9 | 9 | COUNT |

The HAR also requested samples for rules that produced no extracted rows in this dump, including `GenericRFI_BODY`, `block_testsite_public_access`, `Ratelimit-bots`, `Ratelimit-couples-ai-agent`, `Ratelimit-couples-ai-agent-easydirectory`, `Ratelimit-xff`.

A second frequent User-Agent in the extract is `Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0` (328 rows; 11 unique IPs). Those rows are concentrated on hosts `3.24.246.204` and `mainsite.3.24.246.204`, not on `www.easyweddings.com.au`. This is a separate slice from Chrome/125.

Identified bot User-Agents also appear in the extract (AhrefsBot, MJ12bot, Applebot, Bytespider, and others). They are present; their share of total surge volume is not measured by these samples.

---

## Not established by the current evidence

- Cause of the surge (crawler, AI bot, DDoS, or something else).
- Independent measurement of the site’s normal AU/US mix (operator statement only).
- Share of all surge requests that used Chrome/125 (WAF samples are per-rule, and `PopulationSize` 5000 may be capped).
- Whether `BlockChrome125UserAgent` is a hard BLOCK or a rate-based rule, and the numeric limit / window / aggregate key (operator stated ~100 requests / 5 minutes; no Web ACL export is in the repo).
- Whether every Chrome/125 request was blocked after 10:41 Melbourne (ALLOW samples exist after that time; operator says some are expected to pass under a per-window allowance).
- Payload, session, or TLS fingerprints beyond the HTTP headers in the WAF samples.
- Whether the dashboard and the WAF ACL observe the exact same edge (dashboard screenshots are unlabeled beyond TOTAL/ALLOW/BLOCK).
