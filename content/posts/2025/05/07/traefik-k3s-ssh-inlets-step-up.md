---
title: "Traefik, K3s, SSH and Inlets - Step Up"
date: 2025-05-07T18:40:05+02:00
draft: false
toc: false
images:
tags:
  - inlets
  - k3s
  - ssh
  - traefik
---

I learned my first lessons from [K3s](https://k3s.io) and
[inlets-pro](https://inlets.dev) on my Raspberry Pi 5 devices. And came up with
a new tunnel architecture.

![Architecture](/images/inlets-ingress-snimux.png)

Two [inlets cloud](https://inlets.dev/cloud/) tunnels passing through
any TCP traefik and preserving client IP address with proxy protocol v2.
The first tunnel is used to route ingress traffic to the K3s cluster.
The second tunnel is used to route management traffic into the home network.

## Tunnel setup

The ingress inlets-pro client is deployed in the K3s namespace `kube-system`
and upstreams to traefik. The tunnel authentication token is stored in a
Kubernetes secret and mounted into the respective pods.

```bash
kubectl --namespace kube-system \
    create secret generic inlets-token \
    --from-file inlets.token
```

```bash
kubectl apply -f i-coresec-zone-ingress.yaml
```

<details>
<summary>i-coresec-zone-ingress.yaml</summary>

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: i-coresec-zone-inlets-client
  namespace: kube-system
spec:
  replicas: 3
  selector:
    matchLabels:
      app: i-coresec-zone-inlets-client
  template:
    metadata:
      labels:
        app: i-coresec-zone-inlets-client
    spec:
      volumes:
        - name: inlets-token
          secret:
            secretName: inlets-token
      containers:
        - name: i-coresec-zone-inlets-client
          image: ghcr.io/inlets/inlets-pro:0.10.3
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: inlets-token
              mountPath: /etc/inlets.token
              subPath: inlets.token
              readOnly: true
          command: ["inlets-pro"]
          args:
            - "uplink"
            - "client"
            - "--url=wss://cambs1.uplink.inlets.dev/tobias-scha-fer/i-coresec-zone"
            - "--token-file=/etc/inlets.token"
            - "--upstream=80=traefik:80"
            - "--upstream=443=traefik:443"
          resources:
            requests:
              memory: "25Mi"
              cpu: "100m"
          securityContext:
            runAsUser: 1000
            runAsGroup: 1000
            runAsNonRoot: true
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
```

</details>

The management inlets-pro client is deployed in a new K3s namespace
`snimux` and upstreams to an inlets-pro snimux server, deployed in the same
namespace. Beside the tunnel authentication token, the snimux server
configuration is stored in a Kubernetes secret as well. Both secrets are mounted
into the respective pods.

```bash
kubectl --namespace snimux \
    create secret generic inlets-token \
    --from-file inlets.token
```

```bash
kubectl --namespace snimux \
    create secret generic snimux-yaml \
    --from-file snimux.yaml
```

```bash
kubectl apply -f k-coresec-zone-ingress.yaml
```

<details>
<summary>k-coresec-zone-ingress.yaml</summary>

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k-coresec-zone-inlets-client
  namespace: snimux
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k-coresec-zone-inlets-client
  template:
    metadata:
      labels:
        app: k-coresec-zone-inlets-client
    spec:
      volumes:
        - name: inlets-token
          secret:
            secretName: inlets-token
      containers:
        - name: k-coresec-zone-inlets-client
          image: ghcr.io/inlets/inlets-pro:0.10.3
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: inlets-token
              mountPath: /etc/inlets.token
              subPath: inlets.token
              readOnly: true
          command: ["inlets-pro"]
          args:
            - "uplink"
            - "client"
            - "--url=wss://cambs1.uplink.inlets.dev/tobias-scha-fer/k-coresec-zone"
            - "--token-file=/etc/inlets.token"
            - "--upstream=443=k-coresec-zone-inlets-snimux:8443"
          resources:
            requests:
              memory: "25Mi"
              cpu: "100m"
          securityContext:
            runAsUser: 1000
            runAsGroup: 1000
            runAsNonRoot: true
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k-coresec-zone-inlets-snimux
  namespace: snimux
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k-coresec-zone-inlets-snimux
  template:
    metadata:
      labels:
        app: k-coresec-zone-inlets-snimux
    spec:
      volumes:
        - name: snimux-yaml
          secret:
            secretName: snimux-yaml
      containers:
        - name: k-coresec-zone-inlets-snimux
          image: ghcr.io/inlets/inlets-pro:0.10.3
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8443
          livenessProbe:
            tcpSocket:
              port: 8443
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            tcpSocket:
              port: 8443
            initialDelaySeconds: 3
            periodSeconds: 5
          volumeMounts:
            - name: snimux-yaml
              mountPath: /etc/snimux.yaml
              subPath: snimux.yaml
              readOnly: true
          command: ["inlets-pro"]
          args:
            - "snimux"
            - "server"
            - "--data-addr=0.0.0.0:"
            - "--port=8443"
            - "--proxy-protocol=v2"
            - "/etc/snimux.yaml"
          resources:
            requests:
              memory: "25Mi"
              cpu: "100m"
          securityContext:
            runAsUser: 1000
            runAsGroup: 1000
            runAsNonRoot: true
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
---
apiVersion: v1
kind: Service
metadata:
  name: k-coresec-zone-inlets-snimux
  namespace: snimux
spec:
  selector:
    app: k-coresec-zone-inlets-snimux
  ports:
    - protocol: TCP
      port: 8443
      targetPort: 8443
  type: ClusterIP
```

</details>

### Traffic redirection and proxy protocol

Traefik must be configured to allow proxy protocol v2 for http and https
entrypoints. Additional any HTTP traffic is redirected to HTTPS.
Both is done by adding arguments to the Traefik deployment.

```bash
kubectl --namespace kube-system edit deployment/traefik
```

```diff
containers:
- args:
+  - --entrypoints.web.proxyProtocol.insecure=true
+  - --entrypoints.web.proxyProtocol.trustedIPs=165.227.238.95/32
+  - --entrypoints.websecure.proxyProtocol.insecure=true
+  - --entrypoints.websecure.proxyProtocol.trustedIPs=165.227.238.95/32
+  - --entrypoints.web.http.redirections.entryPoint.to=:443
+  - --entrypoints.web.http.redirections.entryPoint.scheme=https
+  - --entrypoints.web.http.redirections.entryPoint.permanent=true
```

That's it! 

## TLS with Cert-Manager and Let's Encrypt

But with this any TLS termination must be done on the K3s cluster. I decided to
use [cert-manager](https://cert-manager.io) to controll certificate handling
and followed the instructions from
[K3S Rocks](https://k3s.rocks/https-cert-manager-letsencrypt/) to set it up.

Finally, I extended the `whoami` Helm chart to assign a certificate and serve
over HTTPS.

```bash
helm update whoami . --namespace whoami
```

```yaml
---
# templates/certificate.yaml
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: whoami-cert
  namespace: whoami
spec:
  secretName: whoami-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: whoami.i.coresec.zone
  dnsNames:
    - whoami.i.coresec.zone
```

```diff
---
# templates/ingress.yaml
  routes:
    - match: Host(`whoami.i.coresec.zone`)
      kind: Rule
      services:
        - name: whoami
          port: 80
+  tls:
+    secretName: whoami-tls
```

## Conclusion

This setup is much easier to maintain and extend than the previous one.
Deployment takes two steps. Adding a new domain to the tunnel exitpoint and
deploying the service in the K3s cluster.
