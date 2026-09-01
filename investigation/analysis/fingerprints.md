# Fingerprints

HTTP headers only. No TLS, JA3/JA4, or browser automation artifacts are in the sources.

## Chrome/125 stream (433 rows, all rules)

Exact User-Agent on every row:

```
Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0.0.0 Safari/537.36
```

| Signal | Observation |
|---|---|
| Accept-Language | empty on all 433 |
| Origin | empty on 432; one `https://www.easyweddings.com.au` |
| Referer | 48 / 433; **0 / 197** on `BlockChrome125UserAgent` |
| Method | GET 432, OPTIONS 1 |
| HTTP version | HTTP/2.0 399, HTTP/1.0 22, HTTP/1.1 12 |

BLOCK samples after the UA rule: GET only, no Referer, no Accept-Language, no `/api/`, no `.js`/`.css`.

ALLOW Chrome/125 after 10:41 Melbourne (rule `waf`, 51 rows): 40 have a Referer; 35 API + 7 static + 9 other; countries AU/RO/GB/DE/…; no US. That slice does not match the blocked US-heavy stream. Operator rate-window claim vs origin-IP pile: `analysis/allow-after-block.md`.

HTML document rows across all Chrome/125 have Referer on 0 rows. Referer appears on API and static follow-ups only.

## Shared enquiry canary `ugzWtPtK`

Three US client IPs (`107.174.117.26`, `31.76.160.73`, `172.121.212.11`) send the same `name=ugzWtPtK` on `/wedding/enquiry/mobile`. That string is not a plausible personal name. It is an 8-character mixed-case token.

The name is not the only shared field. All three also send `message=555`, `venue.haveVenue=Yes`, `location=Melbourne`, the same Chrome/125 UA, empty `Accept-Language`, and a time-based SQL probe in `phone`. `107.174.117.26` and `172.121.212.11` (~1h52 later) reuse the same phone nonce (`xdWi3EgB`) and the same `returnUrl` email placeholder; only the supplier slug changes. `31.76.160.73` uses a different nonce (`FqVICQGW`) and adds `email=testing@example.com`.

What that can mean:

- Same script or scanner, not three independent people. A real couple does not reuse `ugzWtPtK`.
- The IPs are egress, not identities. Two of the three sit in `/16`s that also appear on the later UA-block list (`31.76.0.0/16` Ace Data Centers, `172.121.0.0/16` GTT). That is compatible with H4 (rented proxies) carrying a form-probe module. It does not prove those three IPs are the 197 blocked catalog clients (0 overlap).
- The 120 other Chrome/125 enquiry rows do **not** carry `name` / `phone` / `message`. The main catalog walk hits the enquiry URL as a page GET. This canary set is a form-fill / injection probe sampled by `SQLiRuleSet` (`PopulationSize` 9). It is not a volume explanation for the plateau.

A weaker alternative is a public scanner default string copied by unrelated operators. The joint reuse of name + `555` + the same SQL delay + `testing@example.com` makes one tool more likely than independent copy-paste.

## Other fingerprints in the extract

- Chrome/120 (`Windows NT 10.0`) — 328 rows, 11 IPs, hosts `3.24.246.204` / `mainsite.3.24.246.204`. Separate unless a join appears.
- Named bots (AhrefsBot, MJ12bot, Applebot, Bytespider, others) appear. Share of surge volume is unknown.

## Relation to H1–H4

- One frozen UA plus empty `Accept-Language` is not a normal browser mix. It fits scripted clients (H1, H2, H3) on a fleet (H4).
- Missing Referer on the blocked slice fits direct/scripted hits, not click-through sessions.
- Page-boot API + JS/CSS with Referer exist, but they are the ALLOW leftover (and two pre-drop US IPs on SQLi-shaped enquiry), not the `BlockChrome125UserAgent` list.
- H2 is not helped by a spoofed Chrome/125 string; a real accidental crawler more often sends its own UA.

## Next measurements

- TLS fingerprint / HTTP2 settings from the load balancer or Varnish.
- Cookie / session reuse across IPs (would tighten H4 vs independent scripts).
- Whether Chrome/125 before 10:41 already lacked Referer (needs logs outside this HAR window).
