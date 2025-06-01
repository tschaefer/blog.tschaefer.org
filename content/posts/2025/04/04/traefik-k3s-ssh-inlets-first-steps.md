---
title: "Traefik, K3s, SSH and Inlets - First Steps"
date: 2025-04-04T21:21:31+02:00
draft: false
toc: false
description: "In this blog post, I share my first steps with K3s, Traefik and Inlets Pro on Raspberry Pi 5 devices."
images:
    - "https://blog.tschaefer.org/images/traefik-inlets-snimux.png"
author: "Tobias Schäfer"
tags:
  - inlets
  - k3s
  - ssh
  - traefik
---

Recently, I created a [K3s](https://k3s.io) cluster on three Raspberry Pi 5
devices and started using [inlets-pro](https://inlets.dev) for tunneling
private services to the public. A lot of new things to learn. Finally I ended
up with the below architecture.

![Architecture](/images/traefik-inlets-snimux.png)

Here's a short summary of the steps I took to get there.

## Simple K3s Cluster

Setting up the RPIs and K3s was a breeze using Alex Ellis' [k3sup](
https://k3sup.dev) and following his blog post [Will it cluster? k3s on your
Raspberry Pi](https://blog.alexellis.io/test-drive-k3s-on-raspberry-pi/).

### First Helm Chart

As a Kubernetes novice, I read some stuff about K3s and Kubernetes in general
and started to play around with it. After a lot of trial and error, I managed
to create my first [helm chart](https://helm.sh) to deploy Traefiklabs' [whoami
server](https://github.com/traefik/whoami).

<details>
<summary>Helm chart assets</summary>

```yaml
---
# Chart.yaml
apiVersion: v2
name: whoami
version: 0.1.0
description: A helm chart for Traefiklabs whoami server
```

```yaml
---
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
  namespace: whoami
spec:
  replicas: 3
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: docker.io/traefik/whoami:v1.11
          ports:
            - containerPort: 80
```

```yaml
---
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: whoami
  namespace: whoami
  annotations:
    traefik.ingress.kubernetes.io/router.path: /whoami
spec:
  selector:
    app: whoami
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

```yaml
---
# templates/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami
  namespace: whoami
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik
  rules:
    - http:
        paths:
          - path: /whoami
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
```
</details>

I deployed the whoami server by creating a new namespace and installing the
chart.

```bash
kubectl create namespace whoami
helm install whoami . --namespace whoami
```
And finally, I checked the deployment.

```bash
curl http://10.19.80.200/whoami
Hostname: whoami-75f67fdff4-9v895
IP: 127.0.0.1
IP: ::1
IP: 10.42.1.6
IP: fe80::b8fa:58ff:fe91:c289
RemoteAddr: 10.42.0.8:36852
GET /whoami HTTP/1.1
Host: 10.19.80.200
User-Agent: curl/7.88.1
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 10.42.0.1
X-Forwarded-Host: 10.19.80.200
X-Forwarded-Port: 80
X-Forwarded-Proto: http
X-Forwarded-Server: traefik-67bfb46dcb-lzx5p
X-Real-Ip: 10.42.0.1
```

Success!

## Inlets tunnel

The missing piece of the puzzle was to expose the whoami server to the outside.
I had already set up a tunnel to SSH into some private hosts. For that I
deployed an [inlets-pro](https://inlets.dev) TCP server on my dedicated Hetzner
root server, the corresponding inlets-pro TCP client and an inlets-pro snimux
server on one of the RPIs. A good read about the inlets-pro snimux server is
yet another blog post by Alex Ellis, [The only tunnel you'll need for your
homelab](https://inlets.dev/blog/2024/02/09/the-homelab-tunnel-you-need).

<details>
<summary>Inlets-pro tunnel assets</summary>

```systemd
# /etc/systemd/system/inlets-pro-tcp-server.service
[Unit]
Description=inlets Pro TCP Server
After=network.target

[Service]
Type=simple
Restart=always
RestartSec=5
StartLimitInterval=0
ExecStart=/usr/local/bin/inlets-pro tcp server \
    --auto-tls --auto-tls-san=78.47.60.169 \
    --control-addr=0.0.0.0 --control-port=8123 \
    --token-file /etc/inlets-pro/token \
    --allow-ips=::1 --allow-ips=0.0.0.0/0 \
    --proxy-protocol v2

[Install]
WantedBy=multi-user.target
```

```systemd
# /etc/systemd/system/inlets-pro-tcp-client.service
[Unit]
Description=inlets TCP Client
After=network.target

[Service]
Type=simple
Restart=always
RestartSec=5
StartLimitInterval=0
ExecStart=/usr/local/bin/inlets-pro tcp client \
    --url=wss://78.47.60.169:8123/connect \
    --upstream=127.0.0.1 --ports=8443 --auto-tls \
    --license-file=/etc/inlets-pro/license --token-file=/etc/inlets-pro/token

[Install]
WantedBy=multi-user.target
```

```systemd
# /etc/systemd/system/inlets-pro-snimux.service
[Unit]
Description=inlets SNImux Server
After=network.target
Before=inlets-pro-tcp-client.service

[Service]
Type=simple
Restart=always
RestartSec=5
StartLimitInterval=0
ExecStart=/usr/local/bin/inlets-pro snimux server \
    /etc/inlets-pro/mux.yaml --proxy-protocol v2

[Install]
WantedBy=multi-user.target
```
</details>

The key part is the multiplexing configuration file setting up the upstreams.

```yaml
# /etc/inlets-pro/mux.yaml
upstreams:
- name: core.r.coresec.zone
  upstream: 10.19.80.5:22
- name: rpi-zero.r.coresec.zone
  upstream: 10.19.80.222:22
- name: rpi1.r.coresec.zone
  upstream: 10.19.80.200:22
- name: rpi2.r.coresec.zone
  upstream: 10.19.80.201:22
- name: rpi3.r.coresec.zone
  upstream: 10.19.80.202:22
```

Additionally, I exposed the K3s management port 6443 to the outside world.

```yaml
# /etc/inlets-pro/mux.yaml
- name: k3s.r.coresec.zone
  upstream: 172.19.80.200:6443
  passthrough: true
```

### Exposing the whoami server

Still lacking a lot of knowledge about Kubernetes, I decided to terminate the
TLS connection on the existing Traefik instance running in a Docker environment
on the Hetzner server and forward the traffic to the tunnel.

I created a new CNAME record `*.r.coresec.zone` pointing to the Hetzner
server, and added a new router, service and middleware to the Traefik
configuration.

```yaml
---
http:
  routers:
    whoami.r.coresec.zone:
      rule: Host(`whoami.r.coresec.zone`)
      middlewares: whoami.r.coresec.zone
      service: whoami.r.coresec.zone
      tls:
        certresolver: cloudflare
  services:
    whoami.r.coresec.zone:
      loadbalancer:
        servers:
          - url: https://node1.k3s.r.coresec.zone:8443
          - url: https://node2.k3s.r.coresec.zone:8443
          - url: https://node3.k3s.r.coresec.zone:8443
  middlewares:
    whoami.r.coresec.zone:
      addprefix:
        prefix: "/whoami"
```

The key part are the three servers in the loadbalancer section. They are
pointing to the upstream host names, registered in the inlets-pro snimux server
configuration, passing the traffic to the K3s cluster nodes.

```yaml
# /etc/inlets-pro/mux.yaml
- name: node1.k3s.r.coresec.zone
  upstream: 172.19.80.200:443
  passthrough: true
  allow:
    - 172.18.0.0/16
- name: node2.k3s.r.coresec.zone
  upstream: 172.19.80.201:443
  passthrough: true
  allow:
    - 172.18.0.0/16
- name: node3.k3s.r.coresec.zone
  upstream: 172.19.80.202:443
  passthrough: true
  allow:
    - 172.18.0.0/16
```

Further I limited the access to the K3s cluster nodes to the Docker network.

That settled the matter. I can now access the whoami server from the outside
world.

```bash
curl --location https://whoami.r.coresec.zone
Hostname: whoami-75f67fdff4-9v895
IP: 127.0.0.1
IP: ::1
IP: 10.42.1.6
IP: fe80::b8fa:58ff:fe91:c289
RemoteAddr: 10.42.0.8:55346
GET /whoami/ HTTP/1.1
Host: whoami.r.coresec.zone
User-Agent: curl/7.88.1
Accept: */*
Accept-Encoding: gzip
X-Forwarded-For: 10.42.0.14
X-Forwarded-Host: whoami.r.coresec.zone
X-Forwarded-Port: 443
X-Forwarded-Proto: https
X-Forwarded-Server: traefik-67bfb46dcb-lzx5p
X-Real-Ip: 10.42.0.14
```

## Conclusion

I am still learning a lot about Kubernetes, K3s, Traefik and inlets-pro. The
above is just a first step to get a better understanding of the whole
ecosystem. I am sure there are better ways to do this, but for now it works
for me. I will keep you updated about my progress.
