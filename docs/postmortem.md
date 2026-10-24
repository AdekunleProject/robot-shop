# Postmortem: Robot Shop catalogue outage (MongoDB scaled to zero)

**Status:** complete &nbsp;|&nbsp; **Date:** Saturday 24 October 2026 &nbsp;|&nbsp; **Author:** _your name_
**Type:** self-run incident drill (the failure was injected by a script that picks one service at random and scales it to zero)
**Severity:** partial outage, catalogue and payment health failing for about 12 minutes
**Blameless:** this document describes what the system and our process allowed to happen. It is not about who is at fault.

All times are **Lagos time (WAT, UTC+1)** unless marked UTC. Sources: Kubernetes event log, Slack `#alerts`, the drill start file, terminal output.

## 1. Summary

At 22:22 WAT the MongoDB deployment was scaled to zero copies. MongoDB holds the product catalogue and user data, so the catalogue endpoints began failing, and so did the payment service's health check. Our alerts reached Slack within about two minutes. MongoDB was restored at 22:33 and all alerts resolved by about 22:35. The shop was partly unavailable for roughly **12 minutes**, which used about **1.3% of the monthly error budget**.

## 2. Impact

- **Failing probes:** 3 of the 6 monitored URLs: `/api/catalogue/products`, `/api/catalogue/categories` and `/api/payment/health`. In a real shop, customers could not browse products.
- **Duration:** MongoDB was down for **11 min 38 s** (21:22:14 to 21:33:52 UTC). Alerts resolved by about 22:35 WAT.
- **Error budget:** with 3 of 6 URLs failing (50%) for about 11.6 minutes, the loss is about 5.8 minutes of full outage. The 30-day budget is 432 minutes, so this used about **1.3%**. This is below the 5% threshold in `docs/slo.md` that makes a postmortem mandatory, so this one was written for the drill and for learning.
- **Data:** MongoDB stores its data inside the pod (temporary storage). The pod was deleted and a new one was started from the image, so any user accounts or order history created since the shop was installed were lost. The product catalogue came back, because it is part of the image.

## 3. Timeline (WAT)

| Time | Event | Source |
|---|---|---|
| 22:22:01 | Drill script started | `~/.drill-start` (21:22:01 UTC) |
| **22:22:14** | **MongoDB scaled from 1 to 0, its pod deleted** | Kubernetes events (21:22:14 UTC) |
| ~22:23 | Probes for `catalogue/products` and `catalogue/categories` have failed for 1 minute. **`RobotShopEndpointDown` fires in Slack**, then the same alert for `payment/health` | Slack, 10:23 PM |
| 22:23 to 22:31 | Alerts are open and the cause is not yet known. Time was spent reading the runbook and working out which command to run first | Slack, terminal |
| **22:31:35** | `kubectl get deployments -n robot-shop` shows `mongodb 0/0`. **Cause identified.** | Terminal |
| **22:33:52** | `kubectl scale deployment/mongodb --replicas=1` takes effect. New pod scheduled and started within one second | Kubernetes events (21:33:52 UTC) |
| 22:35 | `RobotShopEndpointDown` for `payment/health` resolves | Slack, 10:35 PM |
| 22:35 | `RobotShopAvailabilityFastBurn` fires (late, see section 5) | Slack, 10:35 PM |
| 22:43 | All deployments confirmed `1/1` | Terminal |

**Key durations:** break to first alert about 1 to 2 minutes. Break to mitigation 11 min 38 s. Alert to cause identified about 8 minutes.

## 4. Root cause

The `mongodb` deployment was set to zero replicas, so no database pod existed. `catalogue` and `user` need MongoDB for every request, so the catalogue endpoints returned errors.

**Why it could happen and stay unnoticed by cause:**
- Every data store runs as a **single copy** with no redundancy, so removing it is a full outage for everything that depends on it.
- No alert watches for a deployment having **no available pods**. Our alerts watch the symptoms (URLs failing) but not the cause, so the alert text said `catalogue/products failing` and never mentioned MongoDB.
- Nothing prevents a manual `kubectl scale` on a live system.

**Open question, not confirmed:** `payment/health` also failed, although payment does not use MongoDB directly. A likely explanation is a knock-on effect: payment calls the user service during checkout, the user service waits on MongoDB, and payment runs a single worker process (`processes = 1`), so its health check queued behind the stuck requests. We did **not** verify this, for example with a long `payment: POST /pay/<id>` trace in Jaeger during the window. Until it is verified it remains a hypothesis.

## 5. What went well

- **Detection was fast.** The alert reached Slack within about 1 to 2 minutes of the failure, with the failing URL and a link to the runbook.
- **The fix was simple and quick.** Scaling back up restored service within seconds, because the MongoDB image was already on the node.
- **Several tools agreed.** Slack, the Kubernetes event log and the dashboard gave a consistent picture, which made the timeline easy to rebuild.
- **The runbook's "mitigate first" order worked.** Scaling back up came before any deep investigation.

## 6. What went badly or was lucky

- **About 8 minutes passed between the alert and finding the cause.** The runbook said what to restore but did not say what to look at first, so the cause had to be found by trial.
- **The alerts named symptoms, not the cause.** Three URLs alerting at once did not point at MongoDB.
- **`RobotShopAvailabilityFastBurn` fired late.** It needs the error ratio above 14.4% in both a 5-minute and a 1-hour window. The one-hour window needs time to fill up, so it fired at 22:35, after the fix. The alert named "fast burn" was slower than the basic endpoint alert.
- **Lucky:** the product catalogue lives in the MongoDB image, so it returned on its own. Real customer data would have been lost.

## 7. Action items

| # | Action | Why | Status |
|---|---|---|---|
| 1 | Add a first diagnostic step to the runbook: `kubectl get deployments -n robot-shop`, look for `0/0` or `0/1` | Cuts the time to find the cause | **Done** (`docs/runbook.md`) |
| 2 | Add an alert `RobotShopDeploymentUnavailable` when a Robot Shop deployment has zero available pods for 1 minute, so the message names the failed service | Alerts should show the cause, not only the symptom | To do |
| 3 | Shorten the long window of the fast-burn alert (1 hour to 30 minutes) and re-test | The alert fired after the problem was fixed | To do |
| 4 | Verify the payment health hypothesis (Jaeger trace for the window) and consider more than one payment worker | The payment failure is still unexplained | To do |
| 5 | Give MongoDB persistent storage and, longer term, redundancy | Deleting its pod lost its data | Later (needs the EBS storage driver) |

## 8. Lessons

1. Alert on both symptoms (customers see errors) and causes (a component has no healthy copy). The first tells you something is wrong, the second tells you where.
2. A runbook should start with the fastest way to see what is broken, not only with fixes.
3. Measuring an alert's real behaviour in a drill, such as the late fast-burn alert, finds problems that reading the rule never would.
