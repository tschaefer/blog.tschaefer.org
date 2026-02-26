---
title: "Finch - The Technical Evolution"
description: "Finch started as a fast way to get logs when debugging hurts.
Today, it deploys a complete observability stack - logs, metrics, and profiling
- while keeping the original goal: minimal setup, fast enrollment, and good
defaults."
author: "Tobias Schäfer"
date: 2026-02-26T19:25:00+01:00
draft: false
toc: false
images:
  - "https://blog.tschaefer.org/images/finch-dashboard.png"
tags:
  - finch
  - observability
  - logging
  - grafana
  - loki
  - mimir
  - pyroscope
  - alloy
  - traefik
---

Finch began with a very pragmatic problem: debugging without logs is painful,
and turning on the right logs is often impossible once you depend on
third-party systems or a production environment you can’t easily change.

The original idea was deliberately small
**get a working logging stack up quickly**, then make enrolling machines
trivial.

![Finch Dashboard](/images/finch-dashboard.png)

Since then, Finch has grown without abandoning the minimal part. The project
evolved from *logging infrastructure* into what it now calls
**"The Minimal Observability Infrastructure"**. Logs, metrics, and continuous
profiling, deployed in minutes and managed through a single control plane.

This post is about that evolution, what changed, and what you get today.

## Then: A Minimal Logging Stack

The first iterations focused on two things,
**a repeatable Docker-based deployment** with no hand-wiring components, and
**an enrollment workflow** that turns into a couple commands.

That goal still exists. What changed is that logs alone stopped being enough
once systems grew in complexity.

## Now: The Minimal Observability Infrastructure

The stack has expanded from "Grafana + Loki + agent" into a complete
three-signal setup:

- **Grafana** – dashboards and exploration
- **Loki** – log streaming and aggregation
- **Mimir** – metrics backend (Prometheus-compatible)
- **Pyroscope** – continous profiling backend
- **Alloy** – single agent for logs/metrics/profiles
- **Traefik** – reverse proxy + TLS termination
- **Finch** – agent manager, API and dashboard

This isn’t add everything and hope it works. The theme is still sensible
defaults, minimal operational burden, and an enrollment experience that makes
it realistic to use in day-to-day debugging.

And importantly the stack isn’t just running containers, it also comes with
**provisioned alert rules** so the system can tell you when *itself* is
unhealthy.

This fits the project’s philosophy: **minimal setup, strong defaults**
including guardrails.

## What changed

### Finch is now a proper control plane

Finch isn’t just part of the docker-compose. It’s a management service with
**a gRPC API** with multiple services and a well-defined schema.
**A lightweight web dashboard** with real-time updates and agent-centric
views making it easy to see the state of your fleet at a glance. The dashboard
is intentionally pragmatic: it focuses on *what you need while troubleshooting*,
enrolled agents, endpoints, credentials, config download, and quick filtering.

### Passwordless security model

The API is protected via **mTLS** with finchctl cli certificates deployed
during stack deployment, and explicit while registering any further cli.
Registered agents use **JWT tokens** with long expiration (365 days) and
regeneration on config requests. This keeps enrollment simple, while still
giving you rotation primitives. The dashboard authenticates via JWTs with a
role-based access control model.

### Profiling became a first-class concern

Finch now includes a Pyroscope profiler integration in-process. This is a
subtle but important maturity step. As soon as you ship profiling in your
stack, the natural next step is to profile your own control-plane components
too.

### A single agent can accept logs, metrics, and profiling input

A key shift, instead of ship logs only, the stack is now prepared for
*applications* to forward signals to Alloy locally:

- **Logs Listen:** `http://localhost:3100`
- **Metrics Listen:** `http://localhost:9091`
- **Profiling Listen:** `http://localhost:4040`

This effectively makes "instrumentation onboarding" easier, applications can
push logs, metrics and profiles to local endpoints. Alloy forwards to Loki,
Mimir and Pyroscope, you explore everything in Grafana.

### More lifecycle commands

The CLI also evolved from "deploy stack + register agent" into a proper
operator tool with full lifecycle coverage.

- deploy, update, teardown for both service and agent
- list, describe, edit commands for agents
- dashboard token retrieval, opening
- server secret, mTLS certificates rotation

Even if you only use a small subset day-to-day, these are the pieces that make
the project viable beyond demos.

### Installation as a self-contained installer

[`contrib/install.sh`](https://github.com/tschaefer/finchctl/blob/3ed57a8502778ac584d1a0e1ec4d29509873e319/contrib/install.sh)
provides a "download latest release binary for my OS/arch" workflow. That
matters for adoption, a minimal stack that takes minutes must also have a CLI
that installs in seconds.

## The real evolution: from "get logs" to "connect the dots"

The most meaningful change is that Finch now lets you answer questions like:

- *Did latency increase when these errors started?* (metrics + logs)
- *Which code path is burning CPU during this incident?* (profiling)
- *Can I correlate a deploy to changes in error rate and resource usage?* (dashboards across signals)

While still keep the original promise,
**a minimal, reproducible deployment and fast enrollment.**

## Closing thoughts

Finch’s evolution is the path many teams eventually take, from start with logs
because they’re the fastest "time to value", over add metrics because you need
trends, rates, and alerting, to profiling because you eventually hit performance
questions that logs and metrics can’t explain.

The difference is that Finch tries to make that journey **fast and realistic**,
without turning into "another platform you need to operate".

If you want to dive deeper, check the repos:

- https://github.com/tschaefer/finch
- https://github.com/tschaefer/finchctl

And for the original motivation and early walkthrough, see the earlier posts
on the blog.
