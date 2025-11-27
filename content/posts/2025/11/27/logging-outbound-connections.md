---
title: "Logging Outbound Connections with conntrackd"
description: "Using conntrackd to log outbound connections pushing them to Loki
for analysis and visualization in Grafana."
date: 2025-11-26T15:57:32+01:00
draft: false
toc: false
images:
    - "https://blog.tschaefer.org/images/conntrackd-logging-outbound.png"
tags:
  - conntrack
  - finch
  - grafana
  - loki
  - logging
---

Suspicious outbound connections can indicate malware activity, data
exfiltration, or unauthorized access attempts. By logging these connections
with [conntrackd](https://blog.tschaefer.org/conntrackd) and pushing them to Loki,
you can analyze and visualize outbound traffic patterns, identify anomalies,
and enhance your security posture in Grafana.

![Logging Outbound Connections with
conntrackd](/images/conntrackd-logging-outbound.png)

## A few steps to get you started

Let's assume you are running a [Finch Observability Stack](http://blog.tschaefer.org/posts/2025/11/26/logging-outbound-connections-with-conntrackd/)
and at least enrolled one Finch agent.

**Install conntrackd with GeoIP support** on the agent machine.

```sh
ssh sparrow.example.com 'curl -sSLf https://conntrackd.coresec.zone | sudo WITH_GEOIP=1 sh -'
```

```plaintext
======================================
  conntrackd Installation Script
======================================

[INFO] Detected architecture: amd64
[INFO] Downloading conntrackd binary for amd64...
[INFO] Installing binary to /usr/local/bin/conntrackd...
[INFO] Binary installed successfully
[INFO] Creating configuration directory /etc/conntrackd...
[INFO] Creating basic configuration file /etc/conntrackd/conntrackd.yaml...
[INFO] Configuration file created successfully
[INFO] Creating data directory /var/lib/conntrackd...
[INFO] Downloading GeoLite2-City.mmdb...
[INFO] GeoIP database installed to /var/lib/conntrackd/GeoLite2-City.mmdb
[INFO] Adding GeoIP configuration to /etc/conntrackd/conntrackd.yaml...
[INFO] GeoIP configuration added
[INFO] Creating systemd service /etc/systemd/system/conntrackd.service...
[INFO] Reloading systemd daemon...
[INFO] Enabling conntrackd service...
[INFO] Starting conntrackd service...
[INFO] Service created and started successfully

======================================
  Installation Complete!
======================================

[INFO] conntrackd has been installed
[INFO] Configuration file: /etc/conntrackd/conntrackd.yaml
[INFO] Service status: systemctl status conntrackd
[INFO] View logs: journalctl -u conntrackd -f
```

The script downloads and installs the latest conntrackd release, sets up a
basic configuration file, installs the GeoIP database, and creates and starts
the service with a systemd unit.

```yaml
log:
  level: "info"

filter:
  - "log protocol tcp and destination network public"
  - "drop any"

sink:
  stream:
    enable: true
```

The basic configuration logs service information with level info to stderr and
sends connection log events to stream sink stdout. Log events are limited to
TCP connections to *public* network destinations only.

**Update the conntrackd configuration** to enable Loki as a sink.

```sh
ssh sparrow.example.com 'sudo sed -i "s/stream/loki/" /etc/conntrackd/conntrackd.yaml \
    && sudo systemctl restart conntrackd'
```

```diff
 sink:
-  stream:
+  loki:
     enable: true
```

The updated configuration sends connection log events to the Finch agent
(Alloy) providing a Loki endpoint at `http://localhost:3100`.
The endpoint can be modified by adding an `address` field under `loki`.
Further custom Loki labels can be added under a `labels` field.

```diff
sink:
  loki:
    enable: true
+   address: "http://admin:password@finch.example.com/loki"
+   labels:
+     - "host=sparrow.example.com"
+     - "environment=production"
```

## The last step

**Import the Grafana dashboard** for outbound connection analysis and
visualization.

You can find the dashboard JSON file in the conntrackd repository
[contrib/grafana-dashboard.json](https://github.com/tschaefer/conntrackd/blob/main/contrib/grafana-dashboard.json).

The dashboards provides four panels:

- A geo map showing destination locations (country, city, latitude, longitude,
  counts)
- A time series graph of NEW, UPDATE, DESTROY connections over time
- A panel showing detailed log events
- A table of source, destination IPs, location information (country, city,
  latitude, longitude) and connection counts

## Closing thoughts

Logging outbound connections with conntrackd and pushing them to Loki enables
you to monitor and analyze your network traffic effectively. By visualizing
this data in Grafana, you can quickly identify suspicious activities and take
proactive measures to secure your network. Regularly reviewing outbound
connection logs helps in maintaining a robust security posture and ensures that
any anomalies are promptly addressed.
