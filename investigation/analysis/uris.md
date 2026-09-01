# URIs

Slice: `BlockChrome125UserAgent`, 197 BLOCK samples.
Counts are rule samples, not traffic share. Tables: `facts.md`.

## Pattern

- 197 raw URIs, 143 paths, 0 repeats. Not one hot endpoint in this slice.
- Families: enquiry form 55, directory profile 45, brochure download 38, review user 25, travel activity 15, then articles / accommodation / gallery / QA / suggestion / area listing.
- Hosts: `www.easyweddings.com.au` 148, apex 28, `travel.easyweddings.com.au` 21. No `directory` or `api` in this rule’s samples.
- Enquiry query shape is fixed: `category`, `location`, `supplier`, `fromProfile` on all 55. Locations are Australian place names. 54 distinct suppliers in 55 rows.
- Brochure: four path segments, trailing slash, 38 distinct numeric IDs.
- Reviews: 24 UUID-shaped user IDs, one numeric (`143447`); all paths distinct.
- Absent in this slice: static assets, POST, `/api/`, sitemap, robots.

Across all Chrome/125 rows (433, several rules, geo-biased): `/wedding/enquiry/mobile` 125, `/reviews/user/...` 58, EasyDirectory API 37, `/activities` 15.

## API and follow-up requests (all Chrome/125 rules)

The 37 EasyDirectoryAPI rows plus 6 other API-shaped rows are a different slice from the block-rule URL list.

- Endpoints are page-boot XHR: `login-status`, `get-categories`, `chat-messages`, `get-regions-grouped`, `get-logged-in-user-details`, `load-shortlisted-suppliers`. Not catalog harvest APIs.
- 41 / 43 have a Referer to an HTML page (24 of those are enquiry). HTML document rows themselves have Referer on 0 / 390.
- 0 overlap with the 197 `BlockChrome125UserAgent` IPs. That rule’s samples stay HTML-only (0 API, 0 static).
- `Block-CN-SG-Scrapers` Chrome/125 (172) is also HTML-only, including 147 rows before 10:41 Melbourne. List-walk without follow-ups is not new after the UA block.
- After 10:41, Chrome/125 follow-ups that *are* sampled sit in `waf` ALLOW: 35 API + 7 JS/CSS, no US, many via origin IP `52.65.126.156`.
- Two US IPs before the drop (`107.174.117.26`, `31.76.160.73`) hit enquiry (SQLi-shaped `phone`) then EasyDirectoryAPI 1–2s later with that page as Referer. Not in the later block-rule IP set.

## Relation to H1–H4

- H1 is the best fit for this URI mix: public catalog walk (cards, brochures, reviews, enquiry, travel), not a random path spray.
- H3 as a single-URL L7 flood is a poor fit for this slice. H3 as “many expensive dynamic GETs” is still open.
- H2 would more often show sitemaps, shallow listing pages, or one spider UA. That is not what this rule sampled.

## Per-IP URL behaviour (all Chrome/125 rules)

The block rule still cannot answer “how many URLs per blocked US IP”: 197 samples, one each. Across all Chrome/125 rows the answer is almost the same: 385 / 396 IPs have one sampled request.

The pattern that *is* measurable is shared **templates**, not shared URLs. Enquiry / brochure / reviews / directory profiles are nearly 1:1 IP-to-instance. Same `/16` in the block slice still mixes families.

Where one IP *is* sampled twice or more, three shapes appear:

1. **Catalog walk, minutes apart (SG `195.40.*`, `Block-CN-SG-Scrapers`).** Six IPs. Gaps 5–66 minutes. Different families on the same IP (enquiry → review → venue; two WeddingCars enquiries with different suppliers; gallery → enquiry). No Referer, no API. This is the only HTML-only per-IP walk in the extract.
2. **Same-second page-boot.** `107.174.117.26` and `31.76.160.73`: enquiry then EasyDirectoryAPI 1–2s later with that page as Referer. After 10:41, `205.188.110.196` (3 API, same second) and `159.148.248.172` (brochure then API ~4s later). Browser-like burst if JS ran; also compatible with a script that copies Referer.
3. **`52.65.126.156` is not one client.** 23 Chrome/125 rows, 18 XFF first hops, 0 overlap with the 197 block-rule IPs. Origin/forwarder copies of other clients.

SQLi enquiry (`ugzWtPtK` + time-based SQL in `phone`) is a third client set (three US IPs, same fake name, different suppliers). Same name across IPs is a canary / hardcoded filler, not three people. See `analysis/fingerprints.md`. Sampled because of `SQLiRuleSet`, not because it is a volume driver. Not the 197-IP block list.

Chrome/120 on `103.215.74.185` / `103.215.75.66` repeats `/admin` `/login` `/signin` `/signup` `/account` `/dashboard` on IP-literal hosts. Separate stream.

`.env` / `.aws` path sprays exist on other UAs. Separate from Chrome/125.

## Open

- Same client hitting many URLs on the blocked US list: still not measurable (one sample per IP in that rule).
- Whether enquiry/brochure/reviews dominate the plateau, or only this extract’s sample.
- Whether blocked clients requested API/static and those rows were not sampled, or they never requested them. 0/197 in a `PopulationSize` 5000 rule is the current measurement, not a full log.
- Whether the US enquiry→API pairs are JS execution or a script that copies the page Referer.
- Per-IP rate and distinct-URI count over the full surge still need Varnish or origin logs.
