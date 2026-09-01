# Timeline

Times below are Australia/Melbourne unless marked UTC.
Sources: operator dashboards and WAF samples in `facts.md`. The WAF window does not cover the start of the surge.

| When | Event | Source |
|---|---|---|
| 2026-08-29 12:00 → 2026-08-31 ~02:00 | TOTAL near the 100K mark on the multi-day chart | Dashboard screenshot |
| 2026-08-31 01:53 | ALLOW 84.48K, TOTAL 84.79K, BLOCK 309 | Dashboard tooltip |
| **2026-08-31 ~02:00** | **Surge starts.** TOTAL/ALLOW rise to ~1.4M. BLOCK stays near baseline on that scale | Dashboard screenshot |
| 2026-08-31 ~02:00 → 2026-09-01 morning | Plateau ~1.1M–1.4M. ALLOW overlaps TOTAL | Dashboard screenshot |
| 2026-08-31 22:12:39 UTC (08:12) | Start of HAR `GetSampledRequests` query window | `data/waf/` HAR |
| 2026-08-31 22:51:17 UTC (09:51) | First Chrome/125 row in the extract | `data/waf/sampled_requests.jsonl` |
| 2026-09-01 09:55–10:41 | One-hour chart: TOTAL mostly 18K–24K, peak ~28K near 10:27; brief dip ~4K near 10:02. BLOCK near 0 | Dashboard screenshot |
| **2026-09-01 ~10:41** | **Traffic reaction.** TOTAL falls ~23K → ~3K–3.5K | Dashboard screenshot |
| 2026-09-01 10:42–10:55 | TOTAL ~3K–4K; non-block series ~2K; BLOCK rises to ~1K | Dashboard screenshot |
| 2026-09-01 00:47:11–00:54:24 UTC (10:47–10:54) | `BlockChrome125UserAgent` samples: 197 BLOCK, all Chrome/125 | WAF `GetSampledRequests` |
| 2026-09-01 01:09:49–01:12:24 UTC (11:09–11:12) | 51 Chrome/125 rows still ALLOW under rule `waf` (no US in that slice; 23/26 AU Chrome/125 rows are IP `52.65.126.156`) | `data/waf/sampled_requests.jsonl` |
| 2026-09-01 01:12:24 UTC (11:12) | Last Chrome/125 row in the extract; HAR window ends 01:12:39 UTC | `data/waf/` HAR |
| 2026-09-01 ~12:01 Melbourne (02:01 UTC) | RIPEstat `prefix-overview` for the 197 block-rule IPs | RIPEstat |

## Block Chrome/125

Operator stated on 2026-09-01 that they blocked User-Agent `Chrome 125.0.0.0`. The WAF rule name in the HAR is `BlockChrome125UserAgent`.

The same day the operator added that matching requests are not all denied: about 100 requests per 5-minute window still pass. The HAR has no Web ACL rule statement, so that limit is operator-reported only.

The rule’s sampled BLOCKs start at 10:47 Melbourne — after the 10:41 drop. The exact rule-publish time is not in the files. The drop at 10:41 is the measured traffic change; the 10:47–10:54 samples are the measured block evidence.

## Further changes

No later mitigation, rule edit, or second surge is in the current sources.
Varnish and origin timelines are empty until `data/varnish/` and `data/origin/` have logs.
