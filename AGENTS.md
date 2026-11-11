# Investigation

Find the cause of the request surge on `easyweddings.com.au` that began around 2026-08-31 02:00 Australia/Melbourne.

Candidate explanations live in `investigation/hypotheses.md`. Test them; do not assume one. More than one can be true. Do not pick a cause until evidence rules the others out.

## Layout test

```
investigation/
├── facts.md          measured observations only
├── hypotheses.md     H1–H4 and current status
├── timeline.md       ordered events
├── data/waf/         WAF HAR + extracted samples
├── data/varnish/     empty until logs arrive
├── data/origin/      empty until logs arrive
└── analysis/         ips, uris, fingerprints, cache-cost
```

## How to work

- Read `investigation/facts.md` first. Treat it as the only accepted fact base.
- Put new observations in `investigation/facts.md`, with source and time. Do not mix interpretation into that file.
- Hypotheses and status go in `investigation/hypotheses.md`.
- Event order goes in `investigation/timeline.md`.
- Working notes and derived groupings go under `investigation/analysis/`.
- Do not upgrade an operator statement into a measured fact. Label it.
- WAF `GetSampledRequests` rows are per-rule samples. Do not treat country or User-Agent counts in `sampled_requests.*` as the mix of all traffic.
- Do not invent traffic, IPs, or User-Agents that are not in a source.

## Known context

- Site: `easyweddings.com.au` (also `www`, `directory`, `travel`, `api` hosts in the WAF samples).
- Operator: normal traffic is mostly AU; the extra traffic during the surge was mostly US.
- Mitigation: AWS WAF rule `BlockChrome125UserAgent` on Web ACL `waf-prod` (`ap-southeast-2`). After the block, the dashboard request rate fell at about 2026-09-01 10:41 Melbourne. Operator also stated (~2026-09-01) that the match is not a hard deny: about 100 requests / 5-minute window still pass. Rule JSON is not in the repo.
- Evidence on disk: `investigation/data/waf/` (HAR, extracted samples). Varnish and origin logs are not in the repo yet.

## Open questions

1. What share of the 1.1M–1.4M plateau was Chrome/125, vs other User-Agents?
2. Is the US geo on Chrome/125 a real client location, a proxy/VPN, or a cloud egress?
3. Why do Chrome/125 samples have an empty `Accept-Language`, and why do BLOCK samples have no Referer?
4. Why were some Chrome/125 requests still ALLOW after the block? Operator: ~100 req / 5 min still pass. Measured ALLOW slice is mostly a different mix (API/static, no US, origin IP `52.65.126.156`), not a visible leak of the US HTML stream. Confirm the rule statement.
5. Is Chrome/120 on `3.24.246.204` related, or a separate stream?
6. Did the same clients hit many distinct URLs (directory pages, enquiry, reviews), and at what rate per IP?
