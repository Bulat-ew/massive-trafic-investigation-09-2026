# Easy Weddings — Preliminary Traffic Incident Investigation

**Incident:** abnormal request surge beginning 31 Aug 2026  
**Status:** preliminary investigation  
**Confidence:** high on technical mechanism; low on actor and motivation

## Executive summary

Beginning at approximately **02:00 Melbourne time on 31 Aug 2026**, request volume to Easy Weddings increased by roughly an order of magnitude and remained elevated for more than a day.

A large share of the anomalous traffic used the same User-Agent:

```text
Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)
AppleWebKit/537.36 (KHTML, like Gecko)
Chrome/125.0.0.0 Safari/537.36
```

A WAF rule blocking this User-Agent was enabled at approximately **10:41 Melbourne time on 1 Sep 2026**. Immediately afterwards, the overall request rate dropped sharply.

Analysis of AWS WAF full logs indicates that the most likely source is a **large distributed browser-based crawler/scraper operating through a rotating proxy infrastructure**.

The traffic behaves like a real browser runtime: it performs top-level document navigations, loads JavaScript and CSS, starts workers, and executes frontend API calls. This creates significant request amplification: one crawler page navigation can trigger many additional application requests.

The crawler does not appear to navigate the site in the same way as a normal user. Evidence suggests that URL discovery and browser rendering are separate stages: URLs are discovered or queued first, then browser workers open them directly.

At this stage, the observed behaviour is best described as **aggressive automated crawling that caused an L7 denial-of-service effect**. The operator and motivation are not established.

---

## 1. Traffic characteristics

During the incident, request volume increased from the normal baseline to approximately **1.1M–1.4M requests on the dashboard scale**, compared with roughly **~100K** before the surge.

In a one-hour window before mitigation, Chrome/125 traffic alone included approximately:

```text
661,683 requests
46,759 unique IPs
48,304 unique paths
```

The traffic was therefore both high-volume and highly distributed.

---

## 2. Rotating proxy infrastructure

The source IP distribution is inconsistent with a small number of hosts repeatedly crawling the site.

In WAF samples:

- 197 blocked requests came from 197 different client IPs.
- Those IPs were spread across 142 different `/24` networks.
- The IPs mapped to multiple hosting, transit, and proxy-oriented ASNs.
- Major observed networks included Web2Objects, M247, Datacamp/CDNEXT, Ace Data Centers, Leaseweb, Oxylabs, and others.

In the full logs, tens of thousands of distinct IPs were observed in a single hour.

The most likely architecture is therefore:

```text
crawler / controller
        ↓
rotating proxy pool
        ↓
browser workers
        ↓
easyweddings.com.au
```

This is more consistent with scraping infrastructure than with ordinary users or a conventional compromised-device botnet.

---

## 3. A single automation/network fingerprint

JA4 analysis provides one of the strongest indicators that the traffic comes from a single implementation.

For Chrome/125 traffic with JA4 present:

```text
JA4                                           requests    unique IPs
t13d1516h2_8daaf6152771_d8a2da3f94cd         387,914     46,507
other fingerprints                                  4          2
```

Almost all observed requests with a JA4 fingerprint therefore share the **same TLS/network fingerprint across more than 46,000 IP addresses**.

This strongly suggests:

```text
one browser / automation stack
            ↓
large rotating proxy pool
```

rather than tens of thousands of independent clients.

---

## 4. The traffic uses a real browser-like execution pipeline

The anomalous client does substantially more than send HTTP requests with a fake Chrome User-Agent.

Fetch Metadata headers show internally consistent browser resource semantics:

```text
Sec-Fetch-Dest

empty       427,976
document    113,979
script       55,383
style        26,666
worker        4,990
```

Corresponding modes:

```text
Sec-Fetch-Mode

cors         427,976
navigate     113,981
no-cors       82,049
same-origin    4,990
```

The counts align exactly in several places:

```text
empty                         = 427,976
cors                          = 427,976

script + style                = 82,049
no-cors                       = 82,049

worker                        = 4,990
same-origin                   = 4,990
```

This is consistent with a real browser resource-loading pipeline:

```text
document navigation
        ↓
HTML parsing
        ↓
CSS / JS / workers
        ↓
frontend JavaScript execution
        ↓
XHR / fetch API calls
```

A plain HTTP crawler could theoretically reproduce these headers, but correctly reproducing them across hundreds of thousands of document, script, stylesheet, worker, and API requests would require recreating significant browser behaviour. The simpler explanation is a Chromium-like browser runtime or browser automation platform.

---

## 5. The browser is automated rather than a normal Chrome user

The traffic does not look like normal desktop Chrome usage.

Observed characteristics include:

- a fixed `Chrome/125.0.0.0` User-Agent across the workload;
- `Accept-Language` absent in all 433 Chrome/125 sampled requests;
- Referer present in only 48 of those 433 samples;
- `Sec-CH-UA-Platform` and `Sec-CH-UA-Mobile` almost completely absent in the full logs;
- a single dominant JA4 fingerprint across tens of thousands of IPs.

At the same time, browser-controlled `Sec-Fetch-*` behaviour is highly consistent.

The most likely interpretation is therefore **browser automation with a modified or fixed HTTP fingerprint**, rather than ordinary Chrome users.

---

## 6. The crawler reaches URLs that are not visible in the desktop UI

A particularly useful behavioural clue is:

```text
/wedding/enquiry/mobile
```

On supplier listing pages, this URL is present in an anchor intended only for mobile layouts:

```html
<a
  href="/wedding/enquiry/mobile?..."
  class="btn btn-primary d-block d-sm-none"
>
```

The desktop UI shows a modal instead.

On supplier profile pages, the mobile URL may exist only inside an `onclick` handler:

```html
<a
  class="... d-lg-none mobile-btn"
  onclick="...;window.location='/wedding/enquiry/mobile?...'"
>
```

Despite presenting a desktop macOS Chrome User-Agent, the crawler loads the mobile enquiry route at very high volume.

Observed `fromProfile` distribution:

```text
fromProfile=False    28,918
fromProfile=True      5,480
```

The `False` version is directly available in `href` attributes on listing pages, while the `True` version is harder to discover because it is embedded in JavaScript on profile pages.

This strongly suggests that the system is **extracting or collecting URLs from page structure rather than interacting only with elements visible to a desktop user**.

---

## 7. URL discovery and browser rendering appear to be separate stages

Top-level document navigations usually contain:

```text
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
```

For the heavily repeated `/WeddingAccommodation/` URL:

- the query string is empty;
- no Referer was found;
- it is repeatedly opened as a direct document navigation.

When the incident was analysed from the beginning of the anomalous traffic, the first observed document URL for many IPs showed a highly unusual distribution:

```text
first URL                    IPs
/WeddingAccommodation/      10,632
/wedding/enquiry/mobile      7,915
next most common URLs           tens or fewer
```

When enquiry query strings were included, `/wedding/enquiry/mobile` split into thousands of different concrete URLs.

In a 40-minute window:

```text
IPs                 11,571
unique first URLs   10,837
```

This suggests an architecture in which URLs are discovered or prepared elsewhere and then supplied to browser workers as independent tasks:

```text
URL discovery / corpus
          ↓
shared frontier or task queue
          ↓
browser worker
          ↓
direct page navigation
```

rather than a browser continuously clicking through the site like a user.

---

## 8. `/WeddingAccommodation/` appears to be a special seed or bootstrap URL

`/WeddingAccommodation/` is particularly unusual:

- exact same URL;
- no query parameters;
- no Referer;
- direct document navigation;
- thousands of requests;
- almost one unique IP per request in short time buckets;
- first observed document for more than 10,000 IPs across the incident window.

This page is not proportionally dominant in the Easy Weddings site structure, so its dominance is unlikely to be explained by normal site popularity.

The most likely explanation is that it has a special role in the crawler configuration, for example:

- a seed URL;
- a bootstrap URL;
- a repeatedly replicated task;
- a starting point assigned to many workers.

The exact reason is not yet known.

---

## 9. Browser jobs appear to have a bounded crawl/session size

Across the full incident window, IPs starting from very different URLs commonly generated a similar number of top-level document navigations.

Examples:

```text
first URL                                      avg documents/IP
/WeddingAccommodation/                        ~43.0
/destination-weddings/                         ~45.5
MarriageCelebrant supplier page                ~43.6
WeddingFlowers supplier page                   ~43.0
WeddingVenues supplier page                    ~45–50
```

This suggests that browser/proxy sessions may operate with a bounded job size or crawl budget, roughly in the range of **40–50 document navigations per session/IP**.

This should be treated as an architectural inference rather than a proven fixed limit.

---

## 10. Frontend execution creates substantial request amplification

One top-level page navigation triggers normal Easy Weddings frontend behaviour.

A typical observed sequence includes:

```text
document
↓
CSS
JavaScript
workers
↓
login-status
get-categories
get-regions-grouped
chat-messages
load-shortlisted-suppliers
supplier stats
...
```

Several API counts show near-identical execution frequency:

```text
/chat-messages          64,594
/get-categories         64,589
/get-regions-grouped    64,581

/login-status          129,178
```

`login-status` is approximately twice the other bootstrap calls, which is consistent with repeated execution of the same frontend initialization lifecycle.

This is operationally important:

> A crawler navigation is not a single backend request. Rendering the page causes multiple additional requests and can therefore amplify crawler traffic significantly.

This amplification is a likely reason why the crawling workload caused visible service degradation.

---

## 11. Why the Chrome/125 WAF block was so effective

Before mitigation, Chrome/125 traffic was producing roughly **15K–20K+ allowed requests per minute**.

After the rule was enabled, the crawler continued sending approximately **850–930 blocked requests per minute** for an extended period.

The most likely mechanism is:

```text
Before block

GET document
    ↓
ALLOW
    ↓
browser receives page
    ↓
loads CSS / JS
    ↓
executes frontend
    ↓
many API/resource requests


After block

GET document
    ↓
BLOCK
    ↓
browser receives no page
    ↓
no CSS / JS execution
    ↓
no downstream API amplification
```

This explains why blocking the initial browser navigation reduced total traffic by much more than the number of directly blocked requests alone would suggest.

The crawler itself did **not immediately stop** after receiving blocks. It continued attempting navigations at a relatively stable rate.

No immediate behaviour was observed where the crawler changed User-Agent and restored the previous load.

This suggests a workload more similar to a queued automated crawling job than an actively adapting attacker.

---

# Working hypotheses

| Hypothesis | Current assessment | Rationale |
|---|---|---|
| Distributed browser crawler/scraper | **Very high** | Browser pipeline, URL traversal, JA4 consistency, proxy rotation |
| Central/shared URL frontier or task queue | **High** | Direct navigations, first-URL distribution, missing Referers |
| Automated data harvesting/indexing | **Medium–high** | Structured traversal of supplier, review, brochure and enquiry content |
| Accidental L7 DoS caused by aggressive crawling | **High** | Frontend request amplification explains degradation |
| Intentional L7 DDoS | **Low–medium** | Technically possible, but observed behaviour is inefficient for a pure DoS goal |
| Adaptive attacker | **Low** | No immediate fingerprint/UA adaptation after mitigation |
| Conventional compromised-device botnet | **Very low** | One dominant JA4 plus hosting/proxy-heavy infrastructure |
| AI/agent-driven crawler | **Unknown** | Architecture is compatible, but there is no evidence identifying AI specifically |
| Competitor activity | **Unknown** | No evidence currently identifies the operator or motivation |

---

# Recommended next actions

## P0 — Production protection

### Keep the current Chrome/125 block temporarily

The rule has a demonstrated operational benefit and immediately reduced load.

### Do not rely on User-Agent as the long-term control

The User-Agent can be changed trivially.

The dominant JA4 fingerprint is currently a much stronger behavioural identifier:

```text
t13d1516h2_8daaf6152771_d8a2da3f94cd
```

Evaluate whether AWS WAF can safely block, challenge, rate-limit, or otherwise constrain this traffic based on stronger fingerprint and behavioural signals.

### Monitor for adaptation

Track:

- new User-Agents;
- new JA4 fingerprints;
- return of the previous request volume;
- changes in ASN/country distribution;
- equivalent browser behaviour under a new fingerprint.

A change in fingerprint immediately after mitigation would materially change the assessment of attacker intent and adaptability.

---

## P1 — Preserve incident evidence

Retain or export:

- full WAF logs for the complete incident window;
- JA4 statistics;
- IP and ASN distributions;
- URI and first-URL statistics;
- request-rate timeline before and after mitigation;
- representative raw requests;
- current `facts.md` investigation notes.

The raw factual record should remain separate from hypotheses and attribution.

---

## P1 — Reduce browser-triggered backend amplification

The highest-value engineering follow-up is to identify which frontend bootstrap requests are expensive or unnecessary for anonymous crawlers.

Review at least:

```text
login-status
chat-messages
get-categories
get-regions-grouped
load-shortlisted-suppliers
supplier stats
other uncached API endpoints
```

For each endpoint, measure:

```text
request count
cache HIT/MISS
origin latency
database/service cost
anonymous-user necessity
```

Potential mitigations may include:

- caching;
- request coalescing;
- lazy loading;
- avoiding user-specific bootstrap calls for anonymous sessions;
- endpoint-specific rate limits;
- challenge rules for suspicious browser fingerprints;
- reducing work done before a session has demonstrated legitimate interaction.

The goal is to ensure that a future crawler cannot turn one page view into many expensive backend operations.

---

## P2 — Attribution and provider escalation

If further attribution is useful:

- map the largest proxy ranges to network providers;
- preserve representative offending IPs and timestamps;
- prepare abuse reports for major providers;
- investigate the origin and age of stale/invalid URLs;
- compare crawler URL patterns with known external indexes or scraping services.

This is lower priority than improving production resilience because identifying the operator does not remove the underlying amplification mechanism.

---

# Current technical model

```text
                 crawler controller
                        │
                URL corpus / frontier
                        │
          ┌─────────────┴─────────────┐
          │                           │
 repeated bootstrap seed       specific target URLs
/WeddingAccommodation/         enquiry?supplier=...
          │                           │
          └─────────────┬─────────────┘
                        ↓
                  browser worker
               Chromium-like runtime
                        │
                 rotating proxy
                        │
                 direct navigation
                        │
                full page execution
                        │
            CSS / JS / workers / XHR
                        │
                API amplification
                        │
                        ↓
              Easy Weddings backend
```
