---
title: "Finch - From Logging to Observability"
description: "With the latest releases, Finch and Finchctl are no longer just
about logs—they now deliver a full observability stack: logs, metrics, and
application profiling, all deployable in minutes, with agent enrollment just
as easy as before."
date: 2025-10-26T14:18:30+01:00
draft: false
toc: false
images:
    - "https://blog.tschaefer.org/images/finch-logging-profiling.png"
tags:
  - observability
  - grafana
  - loki
  - alloy
  - finch
  - mimir
  - pyroscope
---

When I started Finch, my primary pain point was simple: debugging without logs is like flying blind. A minimal logging stack made troubleshooting easier, but as my own systems (and yours) have grown, I realized that **logs alone aren’t enough**.

![Finch Observability Stack](/images/finch-logging-profiling.png)

Modern applications need:
- **Metrics** for performance and health.
- **Profiling** for deep-dive analysis (CPU, memory, etc.).
- **Unified dashboards** for actionable insight.

Observability isn’t just a buzzword—it’s about connecting the dots between logs, metrics, and profiling to answer the tough questions.

## The Stack: Minimal, Mature, Complete

The new Finch observability stack includes:

- **Grafana** – Unified visualization for all signals.
- **Loki** – Log aggregation (as before).
- **Mimir** – Scalable, Prometheus-compatible metrics backend.
- **Pyroscope** – Application profiling (CPU, memory).
- **Alloy** – Single agent for logs, metrics, and profiling data.
- **Traefik** – Reverse proxy and TLS termination.
- **Finch** – Agent manager and API.

With just a few commands, you get a deployable stack that covers all the basics of modern observability.

## What's New: Metrics & Profiling

**Metrics via Mimir:**  
Now you can collect, store, and visualize application and system metrics. This means tracking performance, alerting on thresholds, and correlating metrics with log spikes.

**Profiling via Pyroscope:**  
Profiling lets you see *how* your code is running—CPU bottlenecks, memory leaks, and more. With Alloy’s Pyroscope integration, any app can push profiling data for instant flamegraph analysis.

**Unified Agent (Alloy):**  
Alloy now handles logs, metrics, and profiling data—all configurable with a single agent config. No more stitching together multiple collectors.

## Workflow: Deploy, Enroll, Observe

**Deploy the stack:**  
```bash
finchctl service deploy root@your.machine
```
**Enroll an agent:**  
```bash
finchctl agent register --agent.hostname app.example.com --agent.log.journal your.machine
```
**Send metrics and profiling data:**  
Your app can push metrics to `http://localhost:9091` and profiling data to `http://localhost:4040` (via Alloy).

**Explore in Grafana:**  
Dashboards for logs, metrics, and profiles are ready out of the box.

## Real-World Impact

- **Faster troubleshooting:** Correlate logs, metrics, and profiles in one place.
- **Performance tuning:** Spot bottlenecks with metrics and flamegraphs.
- **Easy onboarding:** New agents and services take minutes, not hours.
- **Minimal ops:** No manual config file wrangling; everything can be managed via Finch’s API or finchctl CLI.

## Lessons Learned

- You don’t need a “big enterprise” stack to get real observability.
- Open source tools (Grafana, Mimir, Pyroscope, Alloy) are mature and integrate seamlessly.
- The hardest part is making deployment and agent enrollment intuitive—that’s what Finch and Finchctl focus on.

## Try It Out

Ready to go beyond logs?  
- [finchctl on GitHub](https://github.com/tschaefer/finchctl)
- [finch on GitHub](https://github.com/tschaefer/finch)

Feedback and contributions always welcome!

**What’s next?**  
Stay tuned for guides on custom dashboards, alerting, and advanced profiling integrations!

