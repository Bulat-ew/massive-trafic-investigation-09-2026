# Chrome/125 ALLOW after the UA rule

Question: why do some Chrome/125 requests still ALLOW after ~10:41 Melbourne?

Operator (2026-09-01): matching requests are not all denied; about 100 requests per 5-minute window still pass. That statement is not a Web ACL export. See `facts.md`.

## Two different ALLOW piles in the samples

The 51 `waf` ALLOW Chrome/125 rows (11:09–11:12 Melbourne) are not the same mix as the 197 `BlockChrome125UserAgent` BLOCKs (10:47–10:54).

| Signal | Block-rule BLOCK | `waf` ALLOW after 10:41 |
|---|---|---|
| IPs | 197, one each | 31; overlap with the 197: **0** |
| Country | US 182 / 197 | no US; AU/RO/GB/DE/… |
| Referer | 0 / 197 | 40 / 51 |
| Paths | HTML catalog only (enquiry, profile, brochure, reviews, travel) | 35 API/`/v5/`, 7 JS/CSS, 8 HTML |
| Hosts | www, apex, travel | also `directory` and `api` |
| Client IP `52.65.126.156` | 0 | 17 / 51 (16 with XFF) |

A rate-based rule that only emits samples when the key is already over the limit would explain all-BLOCK samples on `BlockChrome125UserAgent`. It does **not** by itself make these 51 rows the under-limit remainder of the same US HTML stream: that remainder would still look like the block slice (US, no Referer, catalog HTML, some of the same IPs).

## What the operator rate window can explain

If `BlockChrome125UserAgent` is rate-based (~100 / 5 min per key):

- Requests under the cap never match that rule and can show up later as `waf` ALLOW. That is normal WAF behavior.
- `GetSampledRequests` for the rate rule then shows only BLOCKs. The HAR matches that shape.
- Low-rate Chrome/125 clients (or a new 5-minute window) can still pass after 10:41.

What is not measured: the rule JSON (limit, window, key = IP vs XFF vs JA3, host scope). Named `Ratelimit-*` rules in the same ACL produced no samples in this HAR; they are a different set of names.

Dashboard after 10:41: TOTAL ~3K–4K, non-block ~2K, BLOCK ~1K (one-hour chart). Pre-surge on the multi-day chart is near 100K if that chart is hourly (~1.7K/min). The leftover ~2K non-block is in the same order as a return toward baseline plus some leak. It is **not** a count of Chrome/125 under a 100/5 min cap.

## What the origin IP explains without a rate window

17 ALLOW rows use client IP `52.65.126.156` on `directory` / `api`, with public first-XFF + `10.20.*`. Those requests are seen by WAF as the origin, not as the 197 block-rule IPs.

If the UA rule keys on the WAF client IP:

- Origin-forwarded Chrome/125 shares one key (`52.65.126.156`). A 100/5 min cap on that key would still allow a burst; 17 samples in ~3 minutes do not exceed 100.
- Or the rule does not apply the same way on `directory` / `api`.

Three first-XFF addresses on those origin rows also appear as direct ALLOW client IPs (`205.188.16.90`, `205.188.110.147`, `205.188.110.196`). Same browsers can hit www directly and directory via the origin.

## Direct (non-origin) ALLOW

34 ALLOW rows are not `52.65.126.156`. Mostly `www`, countries RO/GB/DE/…, 24 / 34 have Referer, paths are mostly EasyDirectory XHR plus a few enquiry/brochure/article/static hits.

These can be under-quota Chrome/125, or follow-up XHR from pages that already loaded. They are not the US HTML flood sampled by the block rule. 0 IP overlap with that rule is expected if those IPs never crossed the cap **or** if they were never in that rule’s sample set.

## Status

- Operator rate window: **plausible mechanism** for *some* Chrome/125 still ALLOW. Not confirmed in files.
- Measured ALLOW slice: **mostly a different pile** (API/static + origin forwarder), not a visible leak of the US catalog GETs.
- Q4 stays open until a Web ACL export (or console screenshot of the rule statement) is in `data/waf/`.
