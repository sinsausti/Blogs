---
title: "Blue-Green and Canary Deployments"
description: "A practical guide to blue-green and canary deployment strategies — how they work, when to use each, and what to watch out for with databases."
author: "Sebastian Insausti"
date: "2026-03-10"
tags: ["Infrastructure", "Linux"]
canonical_url: "https://insaustis.com/blog/blue-green-canary-deployments.html"
---

# Blue-Green and Canary Deployments

In production, the hardest constraint is availability. Traditional in-place deployments create a downtime window while the old version stops and the new one starts. Blue-green and canary deployments can greatly reduce that window, but connection draining, state, and database changes still require planning.

## The Core Problem

A standard in-place deployment has three failure modes:

- **Downtime during the switch** — the gap between stopping v1 and starting v2.
- **Failed rollout** — v2 starts but is broken; rollback takes as long as the deploy.
- **Silent regression** — v2 works technically but has a bug that only shows under real traffic.

Blue-green addresses the first two. Canary addresses the third.

## Blue-Green Deployments

The idea is simple: run two identical production environments — Blue (current) and Green (new). At any given time, only one is live. To deploy:

1. Deploy v2 to the idle environment (Green) while Blue continues serving all traffic.
2. Run smoke tests against Green in isolation.
3. Switch the load balancer to send traffic to Green.
4. Blue becomes the idle environment — kept running for an instant rollback.

The traffic switch is the key operation. With HAProxy, it's a config reload:

```
# haproxy.cfg — switch backend from blue to green
backend app_backend
    server green-1 10.0.1.20:8080 check
    server green-2 10.0.1.21:8080 check
    # server blue-1 10.0.1.10:8080 check  # disabled

# Reload without dropping connections:
haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -sf $(cat /run/haproxy.pid)
```

With Nginx it's a symlink swap or an upstream block reload. With cloud load balancers (ALB, GCP LB), it's a single API call that takes effect in seconds.

> The critical property is a fast, controlled traffic switch. Rollback can be quick while Blue remains compatible with current data and external state; long-lived connections, DNS caching, queues, and irreversible migrations can make it slower or incomplete.

## Database Considerations

Blue-green is straightforward for stateless services. Databases add complexity because both environments must be able to read and write the same data without conflicts.

The key constraint: **schema changes must be backward-compatible**. During the switchover window, Blue (v1) and Green (v2) may both be active briefly or you may need to roll back. If v2's migration drops a column that v1 still reads, rollback is impossible.

The safe pattern is **expand–contract**:

1. **Expand** — add the new column/table (v1 ignores it, v2 uses it). Deploy Green.
2. **Contract** — after Blue is fully retired, remove the old column in a follow-up migration.

Tools like [gh-ost](https://github.com/github/gh-ost) can reduce blocking during large MySQL schema changes, but they still require capacity planning, monitoring, and a tested cutover.

## Canary Deployments

A canary release sends a small percentage of real traffic to the new version before committing to a full rollout. Unlike blue-green, both versions run simultaneously and serve production traffic.

```
# HAProxy: send ~5% of traffic to the canary (5 out of 100 total weight)
backend app_backend
    server stable-1 10.0.1.10:8080 check weight 47
    server stable-2 10.0.1.11:8080 check weight 48
    server canary-1 10.0.1.20:8080 check weight 5
```

You monitor error rates, latency, and business metrics on the canary. If the numbers look good, gradually increase its weight. If they don't, remove it and the blast radius was limited to 5% of users.

Canary releases are most powerful when combined with observability. You need to be able to answer: *is the canary behaving differently from stable, and is that difference meaningful?* That requires tagging metrics and traces by version so you can compare them directly.

## Failover and Rollback

Both strategies enable fast rollback, but the mechanism differs:

- **Blue-green rollback** — flip the load balancer back to Blue. Takes seconds. No code change needed.
- **Canary rollback** — set canary weight to 0 and remove the canary server. Traffic immediately returns to stable.

In both cases, rollback is a configuration change, not a deployment. That distinction matters at 2 AM.

## Choosing Between Them

Use **blue-green** when:

- You want the cleanest possible cutover and rollback
- Your changes are significant enough that you don't want partial exposure
- You can afford to run two environments simultaneously (the main cost)

Use **canary** when:

- You want to validate behavior under real traffic before full rollout
- Your changes are incremental and you can tolerate a small error rate during validation
- You have the observability in place to actually detect regressions

They're not mutually exclusive. A common pattern: use blue-green for the infrastructure switch, then slowly shift traffic from green to make it a canary-style validation. Once confident, decommission blue entirely.

---

Questions about deployment strategies for your specific setup? [Get in touch.](https://insaustis.com/#contact)
