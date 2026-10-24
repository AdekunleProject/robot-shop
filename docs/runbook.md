# Runbook: Robot Shop availability alert

**Applies to alerts:** `RobotShopEndpointDown`, `RobotShopAvailabilityFastBurn`
**Impact:** customers cannot browse, add to cart or check out. The 30-day error budget is being used up.

## 1. Mitigate first (stop the damage)

Fix the symptom before finding the cause. Do the first item that applies.

1. **A change went out recently?** Roll it back:
   ```bash
   helm history robot-shop -n robot-shop
   helm rollback robot-shop <previous-revision> -n robot-shop --wait
   ```
2. **A pod is down or crashing?** Restart it:
   ```bash
   kubectl get pods -n robot-shop
   kubectl rollout restart deployment/<service> -n robot-shop
   ```
3. **A service was scaled to zero?** Scale it back (`1` is the normal value):
   ```bash
   kubectl get deployments -n robot-shop
   kubectl scale deployment/<service> -n robot-shop --replicas=1
   ```
4. **Nodes not Ready?** `kubectl get nodes`. If a node is down, wait for the node group to replace it, or ask for help.

## 2. Confirm recovery

- The alert in Slack shows **RESOLVED** within a few minutes.
- The Grafana dashboard "Robot Shop: requests, errors, latency" shows availability back near 100%.
- Open the shop and browse a product, add it to the cart.

## 3. Find the cause

| Question | Where to look |
|---|---|
| Which URL is failing? | The alert text names it. Dashboard panel "Errors (failed probes, by URL)". |
| What do the logs say? | Grafana, Explore, Loki: `{service="<name>"} \|~ "(?i)error\|fail"` |
| Where in the request path does it break? | Jaeger, service `payment`, find traces with the red error marker. |
| What changed? | `helm history robot-shop -n robot-shop` and `kubectl get events -n robot-shop --sort-by=.lastTimestamp` |
| Is a pod unhealthy? | `kubectl describe pod <pod> -n robot-shop` and `kubectl logs <pod> -n robot-shop` |

Service map: `web` (front door) calls `catalogue`, `cart`, `user`, `shipping`, `payment`, `ratings`.
`cart` uses Redis, `catalogue` and `user` use MongoDB, `shipping` and `ratings` use MySQL, `payment` publishes to RabbitMQ and `dispatch` consumes from it.

## 4. Escalate

If you cannot restore service within 15 minutes, tell your instructor or team lead and keep the Slack thread updated.

## 5. Afterwards

- Write a blameless postmortem (what happened, impact, timeline, root cause, what went well, what we will change).
- Record the minutes of downtime against the error budget in `docs/slo.md`.
