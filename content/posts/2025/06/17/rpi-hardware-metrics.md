---
title: "Visualize Raspberry Pi Hardware Metrics"
description: "A guide to visualizing Raspberry Pi hardware metrics using
RPInfo, Prometheus and Grafana."
images:
    - "https://blog.tschaefer.org/images/rpi-grafana.png"
author: "Tobias Schäfer"
date: 2025-06-18T20:11:45+02:00
draft: false
toc: false
images:
tags:
  - rpinfo
  - prometheus
  - grafana
  - k3s
  - metrics
  - monitoring
---

The Raspberry Pi is equipped with a VideoCore (VC) GPU, a low-power
mobile-multimedia processor, which can encode and decode a series of multimedia
codes. Additionally, the VC provides information about the hardware and its
peripheral devices.
These information or metrics are exposed via the Linux kernel and can be
accessed with the command line tool `vcgencmd`. `vcgencmd` is part of the
`raspi-utils-core` package, which is installed by default with the Raspberry Pi
OS or can be built and installed by
[source](https://github.com/raspberrypi/utils/tree/master/vcgencmd).

![Grafana Raspberry Pi Hardware Dashboard](/images/rpi-grafana.png)

[RPInfo](https://github.com/tschaefer/rpinfo) is a lightweight RESTful API
server written in Go utilizing `vcgencmd` to expose the gathered information in
a JSON format via HTTP on several endpoints. Further it supports an optional
endpoint named `/metrics` for exposing clock frequencies, CPU temperature,
and voltage usage as [Prometheus](https://prometheus.io) metrics.

## Prerequisites and Deployment

I'm running a [K3s cluster on three Raspberry Pi 5 devices](/posts/2025/05/07/traefik-k3s-ssh-and-inlets-step-up/).
For monitoring the several services I deployed the Prometheus community helm
chart [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
which provides a Prometheus Server, Prometheus Alertmanager and [Grafana](https://grafana.com)
setup specifically designed for Kubernetes clusters.

I installed the RPInfo as systemd service on each Raspberry Pi node, listening
on the cluster internal IP address on port `8081` with Bearer token
authentication. For that I copied the suitable binary to the `/usr/local/bin`
directory, created a systemd service file `/etc/systemd/system/rpinfo.service`

```ini
# /etc/systemd/system/rpinfo.service
[Unit]
Description=Raspberry Pi information service
After=network.target

[Service]
Type=simple
Environment=RPINFO_HOST=127.0.0.1 RPINFO_PORT=8080 RPINFO_ARGS=""
EnvironmentFile=-/etc/default/rpinfo
ExecStart=/usr/bin/rpinfo server --host $RPINFO_HOST --port $RPINFO_PORT $RPINFO_ARGS
DynamicUser=true
Group=video
ProtectSystem=strict
# CapabilityBoundingSet=CAP_NET_BIND_SERVICE
DevicePolicy=closed
DeviceAllow=/dev/vcio r
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

and a related environment file `/etc/default/rpinfo`.

```bash
# This is the configuration file /etc/default/rpinfo for the systemd service
# /etc/systemd/system/rpinfo.service

RPINFO_HOST="172.19.80.200"
RPINFO_PORT="8081"
RPINFO_ARGS="--auth --token <BEARER_TOKEN>"
```

## Collecting and Visualizing the Metrics

To collect the metrics from the nodes I added an additional scrape config to
the Prometheus configuration within the `kube-prometheus-stack` helm chart
values.

```yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs: # Prefer to use additionalScrapeConfigsSecret
      - job_name: 'rpinfo'
        metrics_path: /metrics
        scheme: http
        authorization:
          type: Bearer
          credentials: "<BEARER_TOKEN>"
        static_configs:
          - targets: ['172.19.80.200:8081', '172.19.80.201:8081', '172.19.80.202:8081']
```

And finally I created a [Grafana dashboard](https://github.com/tschaefer/rpinfo/blob/main/contrib/grafana.json)
to visualize the Raspberry Pi hardware metrics. The dashboards includes three
time series panels showing the CPU temperature, the CPU clock frequencies and
the voltage on a logarithmic scale per switchable node.

## Monitoring and Alerting

To monitor the Raspberry Pi hardware metrics and to get notified about unusual
high temperature, I added an alerting rule to the Prometheus configuration
within the `kube-prometheus-stack` helm chart values, which is firing when the
CPU temperature reaches 60 degrees Celsius or higher for at least 1 minute.

```yaml
additionalPrometheusRulesMap:
  rpi-temperature-rules:
    groups:
      - name: rpi.rules
        rules:
          - alert: RpiTemperatureHigh
            expr: rpi_temperature >= 60
            for: 1m
            labels:
              severity: warning
            annotations:
              summary: Raspberry Pi temperature is too high
              description: The current temperature is above 60°C
```

## Conclusion

With RPInfo, Prometheus and Grafana you can easily visualize the hardware
metrics of your Raspberry Pi devices. The setup is lightweight and the metrics
can be used for monitoring and alerting, allowing you to keep an eye on the
health of your Raspberry Pi devices. The provided Grafana dashboard gives you a
quick overview of the hardware metrics and can be customized to fit your needs.

## References

- [RPInfo](https://github.com/tschaefer/rpinfo)
- [Prometheus](https://prometheus.io)
- [Grafana](https://grafana.com)
