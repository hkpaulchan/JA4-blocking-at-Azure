# JA4blockingatazure
blocking JA4 fingerprint at Azure which is not natively supported @2025
This repository documents a real-world case study from HKSTP on how to detect and mitigate an economic denial-of-service (EDoS) bot attack targeting Azure Front Door using JA4 TLS fingerprints, while cascading protection through Azure Front Door (AFD) and Azure Application Gateway WAF.
​
​

Unlike traditional DDoS attacks that try to take your site down, this attack was designed to inflate CDN and data transfer costs without overloading the origin, exploiting cached static assets and globally distributed IPs.

Scenario: JA4-Driven EDoS on CorpSite
HKSTP’s public corporate site (“CorpSite”) is fronted by Azure Front Door, with traffic then forwarded to an Azure Application Gateway and finally to a Web App backend.
​

Between October and November 2025:

Azure Front Door monthly cost jumped from 1,187 USD to 2,273 USD (a 91.5% increase), driven primarily by Zone 2 (Asia-Pacific) data transfer.
​

Total data transfer grew from 2.1 TB to 9.9 TB, with China increasing from 27.6% to 92.3% of traffic.
​

HTTP diagnostic logs showed tens of millions of GET requests to cached static assets under /assets/..., returning almost exclusively HTTP 200 with cacheStatus = HIT.
​

At the same time, empty HTTP referrer headers surged from 22.5% of traffic in October to 83.2% in November, and further up to 94.9% in December, indicating highly automated, non-browser traffic.
​

The net result: over 90% of the increased Front Door cost was attributable to automated bot traffic, not real users.
​

Why JA4 Matters in This Attack
JA4 is a TLS client fingerprint derived from the TLS Client Hello that helps uniquely identify client implementations (browsers, libraries, bots) across IP addresses and sessions. In this case:
​

Top 3 JA4 fingerprints accounted for ≈97% of events, even though IP addresses were spread across tens of thousands of endpoints, heavily concentrated in CN.
​

This pattern is characteristic of bot frameworks / automation tools reusing the same TLS stack behind a large proxy pool.
​
​

Azure Front Door can expose the client’s JA4 fingerprint through the X-Azure-JA4-Fingerprint HTTP header on forwarded requests, which we then consume at Application Gateway WAF to apply precision blocking.
​
​

Example JA4 fingerprints from the attack:

text
t12d8007h2_ee0b5a6c69b8_0a9c83bf8b96
t12d8007h2_ee0b5a6c69b8_b8cb6a4c3c45
t12d800600_ee0b5a6c69b8_0a9c83bf8b96
Architecture Overview
The target production architecture (what this repo documents) looks like this:
​

Azure Front Door (AFD)

Global edge entry point.

Caching for static assets (e.g. /assets/**).

Attacks primarily hit AFD, not the origin.

Emits X-Azure-JA4-Fingerprint header when forwarding traffic downstream.
​

Azure Application Gateway (App Gateway) with WAF

Regional L7 gateway.

Terminates TLS from AFD.

WAF custom rules inspect headers (including X-Azure-JA4-Fingerprint) and block malicious fingerprints.
​

Backend

Web App (CorpSite) in hkstp-corpsite-prod backend pool.
​

text
Internet
    ↓
Azure Front Door (fd-hkstp-hub-shared)
    - Caching
    - Adds X-Azure-JA4-Fingerprint
    ↓
Application Gateway (agw-hkstp-hub-shared) + WAF
    - Reads JA4 header
    - Blocks malicious fingerprints
    ↓
CorpSite Web App (hkstp-corpsite-prod)
You can find the diagrams in the diagrams/ folder (exported from the attached PowerPoint).
​

Detection: From Cost Spike to JA4 Fingerprints
This case followed a structured investigation that you can reuse:

Billing and usage anomaly

Identify abnormal jumps in Front Door cost (requests + data transfer) and correlate with specific SKUs and geographies.
​

Geographic traffic shift

Compare pre-attack and attack-month distributions (e.g. October vs November).

In this case: traffic flipped from diversified regions to >90% in China within a single month.
​

Referrer analysis

Measure proportion of empty Referer headers per month.

Here, empty referrers went from 22.5% → 83.2% → 94.9% while total requests climbed from 12.3M → 32.4M → 42.5M.
​

This level of empty referrer traffic is inconsistent with real browser journeys, indicating automation.

Path and cache behavior

Focus on paths like /assets/js/** and check cacheStatus.

Attack traffic was almost entirely GETs to static assets with cacheStatus = HIT, meaning the origin was shielded but AFD costs still accrued per request.
​

JA4 distribution

Use Splunk (or preferred SIEM) over diagnostic logs containing JA4 to compute top fingerprints by count.

Here, top 3 JA4 fingerprints ≈97% of events across months, showing a small number of TLS client stacks behind many IPs.
​
​

Constraints: Why Front Door WAF Alone Was Not Enough
As of early 2026:

Azure Front Door WAF custom rules support match variables like IPs, geo-location, headers, query string, and request path.
​

However, JA4 isn’t yet exposed as a first-class match variable in AFD WAF, so you cannot directly create a “JA4Fingerprint equals X” rule on Front Door.
​

We confirmed in testing that:

JA4 shows up as the X-Azure-JA4-Fingerprint header on the forwarded request from AFD to downstream services,
​
​

But AFD WAF policies cannot natively match the JA4 fingerprint field at the edge yet.
​

This is why the solution pushes the JA4-aware blocking logic down to Application Gateway WAF, using X-Azure-JA4-Fingerprint as a custom header condition.
​

Mitigation Strategy Overview
The mitigation is intentionally layered and risk-aware:

Front Door: cost-aware limiting and shaping

Rate limit and/or block suspicious patterns based on HTTP method, path, referrer, country, and per-IP request rates.

Aim: reduce bulk of EDoS while avoiding user impact.

Application Gateway WAF: precise JA4 blocking

Use X-Azure-JA4-Fingerprint header to block or rate limit specific JA4 fingerprints associated with the attack.
​

Aim: surgical removal of attacker TLS stacks even if they rotate IPs.

Continuous monitoring and rollback

Track blocked/limited traffic, cost trend, and business KPIs.

Keep rules easily reversible (e.g. via dedicated WAF policy or Infrastructure-as-Code).

Step-by-Step Implementation
1. Prerequisites
You should already have:

An Azure Front Door Standard/Premium profile fronting your CorpSite or similar web property.
​

An Azure Application Gateway (v2) with WAF in Prevention mode, configured as an origin for Front Door.
​

Diagnostic logging for:

Front Door (including X-Azure-JA4-Fingerprint and referrer if available),
​

Application Gateway WAF logs.

A log analytics or SIEM solution (e.g. Azure Monitor + Log Analytics or Splunk) connected to both.

2. Enable and Verify JA4 Header from Front Door
Azure Front Door emits the JA4 fingerprint in the X-Azure-JA4-Fingerprint HTTP header when it has sufficient TLS Client Hello data to compute the fingerprint.
​

Confirm that:

Your Front Door frontend terminates TLS from clients.

It forwards HTTP/HTTPS to Application Gateway without stripping custom headers.

Send a test request through Front Door, then inspect either:

App Gateway access logs, or

A simple echo endpoint behind App Gateway.

Verify that the downstream request contains a header similar to:

text
X-Azure-JA4-Fingerprint: t12d8007h2_ee0b5a6c69b8_0a9c83bf8b96
3. Forward JA4 Header to Backend via Application Gateway
Ensure Application Gateway is configured so that WAF rules can inspect request headers and that it does not remove X-Azure-JA4-Fingerprint before WAF evaluation.
​

If you use rewrite rules or custom probes, ensure they do not override or drop this header.

Confirm via WAF logs that X-Azure-JA4-Fingerprint is visible in the logged request data.

4. Create Application Gateway WAF JA4 Rules
Create a regional WAF policy for Application Gateway with custom rules that match on the JA4 header value.
​

4.1 Block top attacker fingerprints
Use the top attack fingerprints you identified from logs, for example:
​

text
t12d8007h2_ee0b5a6c69b8_0a9c83bf8b96
t12d8007h2_ee0b5a6c69b8_b8cb6a4c3c45
t12d800600_ee0b5a6c69b8_0a9c83bf8b96
Example custom rule (conceptual):

Name: Block-Top-JA4-Fingerprints

Priority: 10

Action: Block

Match: ALL of

Match variable: RequestHeaders["X-Azure-JA4-Fingerprint"]

Operator: Equals

Match values: the three JA4 fingerprints above (multi-value OR)

Optionally: restrict to RequestUri starting with /assets/ and RequestMethod = GET to be extra safe.
​

In pseudo-JSON:

json
{
  "name": "Block-Top-JA4-Fingerprints",
  "priority": 10,
  "action": "Block",
  "matchConditions": [
    {
      "matchVariables": [
        {
          "variableName": "RequestHeaders",
          "selector": "X-Azure-JA4-Fingerprint"
        }
      ],
      "operator": "Equal",
      "matchValues": [
        "t12d8007h2_ee0b5a6c69b8_0a9c83bf8b96",
        "t12d8007h2_ee0b5a6c69b8_b8cb6a4c3c45",
        "t12d800600_ee0b5a6c69b8_0a9c83bf8b96"
      ],
      "transforms": []
    }
  ]
}
Apply this WAF policy to the Application Gateway associated with CorpSite.

5. Front Door WAF Rules for Cost Containment
Even without native JA4 matching, Front Door WAF can still be used to throttle and shape the attack traffic near the edge.
​

Recommended rule set (adapt these to your environment):

Rate-limit empty-referrer static asset floods

Priority: 20

Action: RateLimit (e.g. 30–60 requests per minute per IP)

Conditions (AND):

RequestMethod = GET

RequestUri starts with /assets/

Referer header is empty

Optional: geo-based control

If your business does not heavily depend on a given country (e.g. CN), you can add:

Priority: 30

Action: Block or RateLimit

Conditions (AND):

Country = CN

RequestUri starts with /assets/

Referer header is empty

Global static asset rate limit

Priority: 40

Action: RateLimit (e.g. 200 requests per minute per IP)

Conditions (AND):

RequestMethod = GET

RequestUri starts with /assets/

These rules mirror the strategy described in the internal investigation: start conservative with rate limits, then progressively add stricter block rules as you gain confidence.
​

6. Rollout Phases
To minimize false positives:

Phase 1 (first 30–60 minutes)

Enable only rate-limit rules on Front Door (empty-referrer and baseline asset limits).

Monitor:

Rate-limited request counts.

Overall request and cost trends.

Phase 2

Enable Application Gateway WAF JA4 block rule targeting top attacker fingerprints.

Optionally enable country-specific block/rate-limit rules on Front Door.

Phase 3

Tune thresholds and fingerprints:

Add new malicious JA4 fingerprints as they appear.

Remove any that cause user impact (rare if scoped to static assets and empty referrer).

Monitoring and Verification
After deployment, continuously monitor:

Front Door

Total requests and cost trends.
​

WAF metrics: blocked and rate-limited counts by rule ID.

Geographic distribution of traffic.

Application Gateway WAF

Custom rule matches for JA4 block rules.

Trends in X-Azure-JA4-Fingerprint distribution before/after rollout.

Log analytics / SIEM

Referer = "" volume – should collapse sharply from previous attack levels.

Top JA4 fingerprints – attack fingerprints should drop dramatically, and distribution should diversify.

Business KPIs:

Real user traffic by region.

Page load times and error rates.

The goal is to see:

Cost reduction (e.g. >1,000 USD/month savings in this case) with

No significant decline in legitimate traffic or user engagement.

Repository Structure
You can structure this repo similarly to your Cloud Exit example, with room to add Terraform/Bicep later:

text
├── diagrams
│   ├── existing-flow-frontdoor-appgw.png
│   ├── future-flow-ja4-blocking.png
│   └── ja4-fingerprint-examples.png
├── docs
│   ├── ja4-concepts.md
│   ├── detection-playbook.md
│   └── rollout-runbook.md
├── infrastructure
│   ├── bicep
│   │   ├── frontdoor.bicep
│   │   ├── appgateway-waf-ja4.bicep
│   │   └── monitor.bicep
│   └── terraform
│       └── (optional IaC modules)
└── README.md
diagrams/ – exports from your PowerPoint showing existing and future flows.
​

docs/ – deeper-dive documentation: log queries, KQL/Splunk examples, and operational runbooks.

infrastructure/ – optional Bicep/Terraform templates for AFD, App Gateway WAF policies, and monitoring.

Who Is This For?
This example is useful for:

Cloud and security architects running Azure Front Door and Application Gateway in front of public sites.

FinOps teams investigating unexplained CDN cost spikes without corresponding business growth.

SOC and SecOps engineers who want to leverage JA4 fingerprints for more robust bot and EDoS detection on Azure.

By publishing this case, the goal is to provide a practical, end-to-end blueprint for using JA4 fingerprints plus layered Azure WAF to stop economic attacks before they materially impact your cloud bills.
