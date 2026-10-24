# Robot Shop: availability SLO and error budget policy

**Service:** Robot Shop (the customer-facing store, reached through the `web` service)
**Owner:** Alimi Lateef Adekunle &nbsp;|&nbsp; **Review:** once per week, and after every incident

## 1. The SLI (what we measure)

An external check (Prometheus blackbox exporter) requests six real URLs every 15 seconds:
the home page, the catalogue categories and products, and the health pages of the cart, user and payment services.
A request is **good** if it returns HTTP 2xx within 10 seconds, and **bad** otherwise.

> **Availability SLI = good requests / all requests**

Prometheus query: `avg(avg_over_time(probe_success{job="robot-shop-blackbox"}[5m]))`
(recorded as `robotshop:availability:ratio_5m`). It is shown on the Grafana dashboard
"Robot Shop: requests, errors, latency".

## 2. The SLO (the target)

**99% of requests succeed, measured over a rolling 30 days.**

## 3. The error budget

| Item | Value |
|---|---|
| Allowed failure | 1% (100% minus 99%) |
| In time, over 30 days | 30 x 24 x 60 x 1% = **432 minutes (7.2 hours)** of total outage |
| In requests | 1 in every 100 probe requests may fail |

The budget is the amount of failure we can afford. While budget remains, we may ship changes and run experiments.
When it runs out, reliability work comes before new features.

## 4. Alerts

| Alert | Fires when | Why |
|---|---|---|
| `RobotShopEndpointDown` | Any probed URL fails for 1 minute | Fast signal that customers see errors |
| `RobotShopAvailabilityFastBurn` | Over 14.4% of requests fail across both the last 5 minutes and the last hour | At this rate the 30-day budget is gone in about 2 days |

Alerts go to Slack and contain a link to the [runbook](runbook.md).

## 5. Error budget policy

| Budget remaining | What we do |
|---|---|
| More than 50% | Normal work. Releases go out as usual. |
| 25% to 50% | Releases need a rollback plan written down first. Review the causes of recent failures. |
| Less than 25% | Only reliability fixes and security fixes are released. Feature work waits. |
| 0% (SLO missed) | Release freeze until the budget recovers. A written postmortem is required, and fixes from it come first. |

Other rules:
- Every page that reaches a person gets a postmortem if it burned more than 5% of the monthly budget (about 22 minutes of outage).
- Planned maintenance is announced in advance and counts against the budget, like any other outage.
- If the SLO is repeatedly met with a lot of budget left over, tighten it. If it is repeatedly missed for reasons we accept, loosen it or fix the system. Never ignore a missed SLO.

## 6. Limits of this measurement

The probes are synthetic: they show what a simple automated client sees, not what every real customer sees.
Probes measure the front door, so they do not identify which internal service is at fault. Use logs (Loki) and traces (Jaeger) for that.
