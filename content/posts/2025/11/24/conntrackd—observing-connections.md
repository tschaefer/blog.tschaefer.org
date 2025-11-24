---
title: "conntrackd - Observing Connections at the Kernel Edge"
description: "conntrackd listens to conntrack/netlink events, enriches them
with GeoIP data, and sends structured connection events to multiple sinks
(Loki, journald, syslog, or stream). It’s a tiny, focused tool for turning
low-level connection state into actionable, searchable logs."
date: 2025-11-24T20:16:54+01:00
draft: false
toc: false
images:
    - "https://blog.tschaefer.org/images/netlink-logging-conntrackd.png"
tags:
  - conntrack
  - netlink
  - observability
  - loki
  - logging
---

When I'm investigating weird connection patterns - unexplained SYN floods,
flaky peer connections, or a service that suddenly stops answering - the kernel
already knows a lot about what’s happening. conntrack holds the live state for
every tracked connection; conntrackd makes those events visible, searchable,
and enriches them with context so you can act.

![conntrackd at the Kernel Edge](/images/netlink-logging-conntrackd.png)

I wrote conntrackd to bridge that gap: a small, dependable daemon that **listens
for conntrack/netlink events, enriches IPs with GeoIP data, applies a concise
filter DSL, and fans events out to whatever sink suits your workflow**
(journald, Loki, syslog, or stream).

## How it works

- Listens for conntrack/netlink events (NEW, UPDATE, DESTROY).
- Emits structured JSON with fields: type, flow, src/dst, sport/dport, prot,
  and TCP state (when applicable).
- Optionally enriches IP addresses with a MaxMind GeoIP2/GeoLite2 database
  (city/country/lat/lon).
- Applies an expressive filter DSL so you log only what matters.
- Fans events to multiple sinks simultaneously: journald, syslog, Loki, or a
  stream (stdout/stderr/discard).

A focused feature set keeps the tool small and auditable - a single binary that
does one job well.

## Quick start

Prerequisites
- Linux with netlink / conntrack support
- Root privileges (listening to conntrack events requires elevated capabilities)
- Optional: MaxMind GeoIP2/GeoLite2 City DB for geo enrichment

Run and write to the system journal:

```bash
sudo conntrackd run --sink.journal.enable
```

Ship to Loki:

```bash
sudo conntrackd run \
  --sink.loki.enable \
  --sink.loki.address http://loki.local:3100 \
  --sink.loki.labels "env=prod,team=platform"
```

Stream to stdout (containers / pipelines):

```bash
conntrackd run --sink.stream.enable --sink.stream.writer stdout
```

## Configuration: files and environment variables

You don't have to pass every option on the command line - conntrackd supports
config files and environment variables.

Where it looks
- Default path: `/etc/conntrackd`
- Filenames: `conntrackd.yaml`, `conntrackd.yml`, `conntrackd.json`,
  `conntrackd.toml`
- Override with `--config /path/to/config.yaml`

Formats
- YAML, JSON, and TOML are supported. See
[contrib/config.yaml](https://github.com/tschaefer/conntrackd/blob/main/contrib/config.yaml)
for a full example.

Environment variables
- Use the `CONNTRACKD_` prefix; underscores represent nested keys:
  - sink.stream.writer → CONNTRACKD_SINK_STREAM_WRITER
  - log.level → CONNTRACKD_LOG_LEVEL

Priority (highest → lowest)
1. Command-line flags
2. Environment variables
3. Configuration file
4. Internal defaults

This lets you keep reusable configs for deployments, override runtime values
via environment variables, and still tweak with flags for ad-hoc runs.

Example configuration (YAML):

```yaml
---
log:
  level: info

sink:
  journal:
    enable: true
  loki:
    enable: true
    address: "http://loki.local:3100"
    labels:
      - "env=prod"
      - "team=platform"
  stream:
    enable: false
    writer: stdout

geoip:
  database: /usr/share/GeoIP/GeoLite2-City.mmdb

filter:
  - "drop destination address 10.0.0.1"
  - "log protocol TCP"
  - "drop ANY"
```

Run with a config file:

```bash
sudo conntrackd run --config /etc/conntrackd/conntrackd.yaml
```

Or override with environment variables:

```bash
export CONNTRACKD_LOG_LEVEL=debug
export CONNTRACKD_SINK_STREAM_WRITER=discard
sudo -E conntrackd run --config /etc/conntrackd/conntrackd.yaml
```

Notes
- Use config files for repeatable deployments (systemd, containers).
- Use env vars for CI or container overrides.
- Use flags for quick debugging.

## Filtering: see less, find more

conntrackd includes a small DSL to control which events are logged. Filters are
evaluated in order (first-match wins), and events are logged by default unless
a rule matches.

Examples:

- Log every new TCP connection:
  ```
  log type NEW and protocol TCP
  drop ANY
  ```

- Log only NEW TCP connections to port 22:
  ```
  log type NEW and protocol TCP and destination port 22
  drop ANY
  ```

- Drop a specific destination, otherwise log TCP:
  ```
  drop destination address 8.8.8.8
  log protocol TCP
  drop ANY
  ```

The DSL is case-insensitive and supports abbreviations (src/dst).
See [docs/filter.md](https://github.com/tschaefer/conntrackd/blob/main/docs/filter.md)
for full grammar, examples, and best practices.

## Structured logs you can query

conntrackd emits JSON suitable for log pipelines or Loki streams.

Loki-style example:

```json
{
  "stream": {
    "city": "Nuremberg",
    "country": "Germany",
    "detected_level": "INFO",
    "dport": "443",
    "dst": "2a01:4f8:1c1c:b751::1",
    "flow": "574674164",
    "host": "core.example.com",
    "lat": "49.4527",
    "level": "INFO",
    "lon": "11.0783",
    "prot": "TCP",
    "service_name": "conntrackd",
    "sport": "44950",
    "src": "2003:cf:1716:7b64:d6e9:8aff:fe4f:7a59",
    "state": "TIME_WAIT",
    "type": "UPDATE"
  },
  "values": [
    [
      "1763537351540294198",
      "UPDATE TCP connection from 2003:cf:1716:7b64:d6e9:8aff:fe4f:7a59/44..."
    ]
  ]
}
```

Stream sink example:

```json
{
  "time": "2025-11-22T11:34:43.181432081+01:00",
  "level": "INFO",
  "msg": "NEW TCP connection from 2003:cf:1716:7b64:da80:83ff:fecd:da51/4...",
  "type": "NEW",
  "flow": 2899284024,
  "prot": "TCP",
  "src": "2003:cf:1716:7b64:da80:83ff:fecd:da51",
  "dst": "2a01:4f8:160:5372::2",
  "sport": 41220,
  "dport": 80,
  "state": "SYN_SENT"
}
```

## Sinks and integration

- journald - local/systemd integration.
- syslog - forward to classic collectors.
- loki - label streams for Grafana searches. conntrackd sets
  `service_name=conntrackd` and `host=<hostname>`; add labels with
  `--sink.loki.labels`.
- stream - stdout/stderr/discard for containers and pipelines.

The Loki sink performs readiness checks to avoid silently dropping events when
Loki is unavailable.

## Operational notes & best practices

- Run as a dedicated, minimally-privileged service (AmbientCapabilities or
  equivalent).
- Keep GeoIP DBs updated if you use geo enrichment.
- Be mindful of privacy and compliance — connection events can contain
  sensitive metadata.
- Use the filter DSL to reduce noise before logs hit long-term storage.
- When shipping to Loki, add meaningful labels (env, team) for better queries.

Minimal systemd unit:

```ini
[Unit]
Description=conntrackd - connection event logger
After=network.target

[Service]
ExecStart=/usr/local/bin/conntrackd run --config /etc/conntrackd/conntrackd.yaml
Restart=on-failure
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_RAW

[Install]
WantedBy=multi-user.target
```

## When to use conntrackd

- **Incident triage**: surface which hosts/services are making or receiving
  connections.
- **Baseline/telemetry**: monitor SYNs, resets, and unusual ports across hosts.
- **Security monitoring**: feed structured connection events into SIEMs or Loki
  for correlation.
- **Ad-hoc debugging**: surface connection state without running packet captures.

## Closing thoughts

conntrackd doesn’t replace full packet captures or complex network
observability suites; it complements them by surfacing kernel-level connection
state as structured, enrichable events. If you want quick visibility into what
the kernel is tracking and ship that into your existing log pipeline,
conntrackd keeps the surface area small and the data useful.

**Try it out**:
- Repository: https://github.com/tschaefer/conntrackd
- Releases: https://github.com/tschaefer/conntrackd/releases
- Docs: [docs/filter.md](https://github.com/tschaefer/conntrackd/blob/main/docs/filter.md)
  (filter DSL and examples)

If you have ideas - tighter filtering predicates, new sinks, or dashboards -
open an issue or a PR. I’d love to see how you use kernel conntrack events in
the wild.
