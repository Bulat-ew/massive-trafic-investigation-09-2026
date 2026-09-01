# Cache cost

No Varnish or origin logs are in the repo (`data/varnish/` and `data/origin/` are empty).
Nothing below is a measured hit/miss or origin QPS.

## What can be said without those logs

Dashboard TOTAL ≈ ALLOW through the plateau; BLOCK is near baseline until ~10:41 Melbourne on 2026-09-01. That does not say whether ALLOW was served from cache or from origin.

URI families in the `BlockChrome125UserAgent` sample are mostly parameterized HTML/form/download paths:

| Family | Why it may miss cache or hit origin |
|---|---|
| `/wedding/enquiry/mobile?...` | Unique query strings (`supplier`, `location`, `returnUrl`) |
| `.../DownloadBrochure/` | Per-supplier download; often uncacheable or large |
| `/{Category}/{Location}/{SupplierSlug}` | Many distinct slugs; cacheable in principle if TTL exists |
| `/reviews/user/{id}` | Per-user pages; 25 distinct IDs in the sample |
| Travel `/activities`, `/accommodation/detail` | Some `currency=` variants |

If Varnish keys on full URI including query, enquiry rows are poor cache candidates. That would raise origin cost even if H1 (scrape) is the intent. That is a mechanism question, not a verdict on H3.

## What would settle it

Drop into `data/varnish/` and `data/origin/`:

- Hit/miss and `TTL` by path family during 2026-08-31 02:00 → 2026-09-01 10:41 Melbourne.
- Origin status (200 vs 5xx vs timeout) and response time on enquiry / brochure / directory.
- Whether `DownloadBrochure` and enquiry were the expensive set, vs directory HTML that cache could absorb.

Until those files exist, cache-cost cannot distinguish “scrape that happened to be expensive” (H1) from “designed origin exhaustion” (H3).
