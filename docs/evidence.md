# Evidence: Capstone 2, Keep It Running

Screenshots taken on 24 October 2026 while the system was running. Each section matches a step in the [README](../README.md).

## Step 1: Robot Shop deployed and healthy

The shop running on the Kubernetes cluster, viewed through `kubectl port-forward`.

![Robot Shop in the browser](screenshots/01-shop-1.png)
![Robot Shop in the browser, second view](screenshots/01-shop-2.png)

## Step 2: Metrics dashboard

The Grafana dashboard "Robot Shop: requests, errors, latency", built from blackbox probes of six shop URLs.

![Grafana dashboard](screenshots/02-dashboard-1.png)
![Grafana dashboard, second view](screenshots/02-dashboard-2.png)

## Step 3: Logs searchable by service and error text

Fluent Bit sends container logs to Loki. These are searches in Grafana Explore, for example `{service="cart"}` and `{service="payment"} |~ "(?i)error|fail"`.

![Loki search 1](screenshots/03-logs-1.png)
![Loki search 2](screenshots/03-logs-2.png)
![Loki search 3](screenshots/03-logs-3.png)
![Loki search 4](screenshots/03-logs-4.png)
![Loki search 5](screenshots/03-logs-5.png)
![Loki search 6](screenshots/03-logs-6.png)
![Loki search 7](screenshots/03-logs-7.png)

## Step 6: Incident drill, alerts in Slack

During the drill, MongoDB was scaled to zero. The alerts reached Slack within about two minutes, with a link to the runbook. See the [postmortem](postmortem.md) for the full timeline.

![Slack alerts at the start of the drill](screenshots/06-drill-slack-1.png)
![Slack alerts, earlier cart test and drill](screenshots/06-drill-slack-2.png)
![Slack alerts during the drill](screenshots/06-drill-slack-3.png)
![Slack alerts, catalogue categories](screenshots/06-drill-slack-4.png)
![Slack alerts, fast burn and resolved](screenshots/06-drill-slack-5.png)

<!--
Add these when the screenshots exist, then delete this comment block:

## Step 4: Trace in Jaeger
![Jaeger trace list](screenshots/04-trace-list.png)
![Jaeger open trace](screenshots/04-trace-open.png)

## Step 6: Dashboard during the drill (Last 1 hour)
![Dashboard dip and recovery](screenshots/06-drill-dashboard.png)

## Step 6: Logs during the drill
![Catalogue errors in Loki](screenshots/06-drill-logs.png)
-->
