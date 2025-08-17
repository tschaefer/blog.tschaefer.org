---
title: "Finch - A Minimal Logging Stack"
description: "Debugging without logs is painful. Finch and finchctl let you
spin up a complete Grafana/Loki logging stack in minutes and enroll agents
with a single command. Out of the box, you get pre-populated dashboards for
Docker and systemd journal logs — so you can go from \"no logs\" to
\"actionable insights\" fast."
date: 2025-08-17T13:21:57+02:00
draft: false
toc: false
images:
tags:
  - logging
  - grafana
  - loki
  - alloy
  - finch
---

One of the recurring challenges I face as a software engineer is debugging issues with little to no logs. Often, bug reports or production issues land on my desk containing only vague error descriptions — and crucially, missing the logs that would make the problem reproducible.

This situation becomes even harder when third-party services are involved. In those cases, I can’t simply “turn up” the logging. I need a way to get useful, structured logs quickly, with minimal setup overhead, so that the next time an incident happens, I’m not left blind.

That’s the motivation behind **finch**: a minimal, reliable, and fast way to deploy a logging stack and register log agents.


## Why Finch?

I didn’t want to create a new logging tool. In fact, there are already great solutions out there. The problem I ran into again and again wasn’t *how* to collect logs, but how to quickly get a working logging stack running when I needed it most — often in the middle of debugging an issue with minimal information available.

That’s why Finch exists: it automates the deployment of a minimal logging stack and makes enrolling agents effortless.

I chose the **Grafana ecosystem** because it provides a consistent set of tools:

- **Loki** for log aggregation,
- **Grafana** for visualization,
- **Alloy** as a unified agent that integrates existing log sources (systemd, Docker, files).

With `finchctl`, you don’t need to manually wire these pieces together. One command gets you a full stack, and adding new agents is just as simple.


## The Stack

When you deploy finch, you get a complete logging stack built from well-known tools:

- **Grafana** – for visualization.
- **Loki** – for log aggregation.
- **Alloy** – the log shipping agent.
- **Traefik** – reverse proxy, TLS termination and access gateway.
- **Finch** – the agent manager.

Everything runs as Docker containers. No extra glue required.


## Walkthrough

### Deploy the Logging Service

The easiest way to get started is with a blank Linux VM and SSH access. From
your local machine, run:

```bash
finchctl service deploy root@10.19.80.100
```

This installs the stack remotely and makes it available under `https://10.19.80.100`.
Admin credentials are written to `~/.finch/config.json`.

If you have a public hostname and want HTTPS with Let’s Encrypt:

```bash
finchctl service deploy \
    --service.letsencrypt --service.letsencrypt.email acme@example.com \
    --service.user admin --service.password secret \
    --service.host finch.example.com root@cloud.machine
```

Now your stack is up at `https://finch.example.com`, secured with a valid TLS certificate.

Out of the box, Grafana comes with **two pre-populated dashboards**: one for
**Docker logs** and one for **systemd journal logs**. That means as soon as
your agents are enrolled, you can start exploring logs visually without any
additional setup.

### Enroll an Agent

With the service running, the next step is to add agents. For example, to ship
systemd logs:

```bash
finchctl agent register \
    --agent.hostname sparrow.example.com \
    --agent.log.journal finch.example.com
```

This creates an agent configuration (`finch-agent.cfg`) prefilled with Loki
endpoints and credentials. To deploy it to the target machine:

```bash
finchctl agent deploy --config finch-agent.cfg root@sparrow.example.com
```

Done. Logs from `sparrow.example.com` will start streaming into Grafana/Loki.

Agents support multiple sources:

- Journald: `--agent.log.journal`
- Docker: `--agent.log.docker`
- Files: `--agent.log.file /var/log/*.log`


## Beyond the Basics

Finchctl also includes commands to update or tear down your stack, and Finch
exposes an HTTP API if you prefer managing agents programmatically.

For example, registering an agent via curl:

```bash
curl -u admin:secret -X POST \
    -H "Content-Type: application/json" \
    -d '{"hostname": "app.example.com", "log-sources": ["journal://"] }' \
    https://finch.example.com/api/v1/agent
```

## Closing Thoughts

Finch isn’t a new logging system — it’s an enabler. By automating deployment
and agent enrollment, it removes the friction of standing up a logging stack
when you need it most.

The heavy lifting is still done by **Grafana, Loki, and Alloy**. Finch just
ties them together in a way that makes sense for quick troubleshooting and
small-scale setups: one command for the stack, one command per agent.

The goal is simple: when the next bug report arrives with missing context,
I can spin up a logging stack in minutes, enroll the relevant machines, and
finally get the logs I need — without re-inventing the wheel.

Try it out here:
- [finchctl on GitHub](https://github.com/tschaefer/finchctl)
- [finch on GitHub](https://github.com/tschaefer/finch)
