---
title: Kubernetes Deployment
description: Deploy EmailEngine on Kubernetes with production configurations
sidebar_position: 2
---

# Kubernetes Deployment

Deploy EmailEngine on Kubernetes for container orchestration and cloud-native deployments.

:::tip Docker Basics
For Docker and Docker Compose setup, see the [Docker Installation Guide](/docs/installation/docker). This page covers Kubernetes-specific deployment.
:::

:::warning Single Instance Only
**EmailEngine does NOT support horizontal scaling.** You must run exactly **one replica** of EmailEngine per Redis database. Two instances on the same Redis database each open their own IMAP connections for every account, deliver every webhook twice, and overwrite each other's sync state. Scale vertically (more CPU/RAM) instead of horizontally (more replicas).
:::

## Overview

Kubernetes deployment provides:

- **Self-healing** - Automatic pod replacement on failure
- **Controlled updates** - A `Recreate` rollout that never overlaps two instances (see [Update Strategy](#update-strategy))
- **Service discovery** - A stable in-cluster address for Redis and for EmailEngine
- **Secret management** - Credentials mounted from Secrets rather than baked into images
- **Resource management** - CPU and memory limits

## Prerequisites

- A Kubernetes cluster whose API serves `networking.k8s.io/v1` Ingress (1.19 or later)
- `kubectl` configured
- Redis accessible from cluster (or deploy Redis in cluster)

## Basic Deployment

### EmailEngine Deployment

Create `emailengine-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: emailengine
  labels:
    app: emailengine
spec:
  replicas: 1  # IMPORTANT: Do not increase - EmailEngine doesn't support horizontal scaling
  selector:
    matchLabels:
      app: emailengine
  template:
    metadata:
      labels:
        app: emailengine
    spec:
      containers:
      - name: emailengine
        image: postalsys/emailengine:v2
        ports:
        - containerPort: 3000
          name: http
        env:
        - name: EENGINE_REDIS
          valueFrom:
            secretKeyRef:
              name: emailengine-secrets
              key: redis-url
        - name: EENGINE_SECRET
          valueFrom:
            secretKeyRef:
              name: emailengine-secrets
              key: secret
        # The image already sets EENGINE_HOST=0.0.0.0 and EENGINE_API_PROXY=true.
        # Name the peers allowed to set X-Forwarded-For (the ingress controller's pods):
        - name: EENGINE_API_PROXY_ADDRESSES
          value: "10.244.0.0/16"
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: emailengine
spec:
  selector:
    app: emailengine
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: LoadBalancer
```

### Secrets Management

Create `emailengine-secrets.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: emailengine-secrets
type: Opaque
stringData:
  redis-url: "redis://redis:6379/8"
  secret: "your-secret-key-at-least-32-characters"
```

**Create from the command line:**

```bash
kubectl create secret generic emailengine-secrets \
  --from-literal=redis-url="redis://redis:6379/8" \
  --from-literal=secret="$(openssl rand -hex 32)"
```

The keys of the Secret (`redis-url`, `secret`) are what the Deployment's `secretKeyRef` entries name; they do not have to match the environment variable names.

**Prepared configuration** goes through the same Secret. Add the values as keys and map them to [`EENGINE_PREPARED_PASSWORD`](/docs/configuration/reset-password), [`EENGINE_PREPARED_TOKEN`](/docs/configuration/prepared-settings/tokens) and [`EENGINE_PREPARED_LICENSE`](/docs/configuration/prepared-settings/license) with `secretKeyRef`, or mount the Secret as files and point the `_FILE` variants at them:

```yaml
        - name: EENGINE_PREPARED_LICENSE_FILE
          value: /run/secrets/emailengine/license
        volumeMounts:
        - name: prepared
          mountPath: /run/secrets/emailengine
          readOnly: true
      volumes:
      - name: prepared
        secret:
          secretName: emailengine-secrets
```

Most `EENGINE_*` variables have a `_FILE` form; the exceptions are listed under [Loading values from files](/docs/configuration/environment-variables#loading-values-from-files).

### Redis StatefulSet

If running Redis in the cluster:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        command: ["redis-server", "--save", "60 10000", "--save", "300 10", "--save", "900 1", "--maxmemory-policy", "noeviction"]
        ports:
        - containerPort: 6379
        volumeMounts:
        - name: redis-data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
  - port: 6379
  clusterIP: None
```

## Deploy to Kubernetes

```bash
# Create namespace
kubectl create namespace emailengine

# Apply configurations
kubectl apply -f emailengine-secrets.yaml -n emailengine
kubectl apply -f redis-statefulset.yaml -n emailengine
kubectl apply -f emailengine-deployment.yaml -n emailengine

# Check status
kubectl get pods -n emailengine
kubectl get services -n emailengine

# View logs
kubectl logs -f deployment/emailengine -n emailengine

# Note: Do NOT scale to multiple replicas - EmailEngine doesn't support horizontal scaling
# kubectl scale deployment/emailengine --replicas=1 -n emailengine
```

## Production Configuration

### Resource Tuning

Adjust resources based on account count. Since EmailEngine runs as a single instance, scale **vertically** by increasing CPU and memory:

| Accounts | Memory | CPU | Notes |
|----------|--------|-----|-------|
| < 100 | 1Gi | 500m | Minimum for small deployments |
| 100-1000 | 2Gi | 1 | Typical small business usage |
| 1000-5000 | 4Gi | 2 | Medium deployments |
| 5000+ | 8Gi | 4 | Large deployments, consider dedicated Redis |

:::note Vertical Scaling Only
EmailEngine does not support horizontal scaling. For very large deployments (10,000+ accounts), consider:
- Running multiple EmailEngine instances with **separate Redis databases** (each handling different accounts)
- Using a dedicated high-performance Redis instance
- Optimizing worker count with `EENGINE_WORKERS` environment variable
:::

### Ingress Configuration

Two of EmailEngine's endpoints stream responses: `/v1/changes` (the [change stream](/docs/api/get-v-1-changes), also used by the admin dashboard as `/admin/changes`) and `/mcp` when an agent subscribes. An ingress that buffers responses holds those events until the connection closes, so turn buffering off and allow long reads. With ingress-nginx:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: emailengine-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-buffering: "off"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - emailengine.example.com
    secretName: emailengine-tls
  rules:
  - host: emailengine.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: emailengine
            port:
              number: 80
```

### Update Strategy

Use the `Recreate` strategy so the old pod is fully terminated before the new one starts:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: emailengine
spec:
  replicas: 1
  strategy:
    type: Recreate
```

Do not use `RollingUpdate` with `maxSurge` - that briefly runs two EmailEngine instances against the same Redis database, which EmailEngine does not support. The `Recreate` strategy means a short downtime during each deployment, but this is the safe trade-off for a single-instance application.

Roll out a new release by changing the image tag, for example `kubectl set image deployment/emailengine emailengine=postalsys/emailengine:v2.79.4`. A floating tag such as `v2` does not change the pod template, so nothing is rolled out.

## Monitoring

### Prometheus ServiceMonitor

The `/metrics` endpoint needs an access token with the `metrics` scope. Issue one with `emailengine tokens issue -s metrics` (see [Prepared Tokens](/docs/configuration/prepared-settings/tokens)) and store it in the Secret under `metrics-token`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: emailengine
  labels:
    app: emailengine
spec:
  selector:
    matchLabels:
      app: emailengine
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
    bearerTokenSecret:
      name: emailengine-secrets
      key: metrics-token
```

### Health Check Endpoints

`/health` needs no authentication. It returns `{"success": true}` once the API worker is serving, every configured IMAP worker thread is running, and a Redis write-read-delete round trip succeeds; otherwise it returns a 500 whose message is `Not all IMAP workers available` or `Database check failed`. Nothing answers on the port until the API worker is up, which is what makes it usable for both probes:

- **Liveness:** `/health` - a pod that keeps failing after `initialDelaySeconds` is restarted
- **Readiness:** `/health` - the Service does not route to the pod until the API answers

The check says nothing about individual accounts: an instance whose accounts are all in `authenticationError` is still healthy. Watch the `imap_connections` metric or the account list for that.

## Troubleshooting

### Check Pod Status

```bash
# Get pod status
kubectl get pods -n emailengine

# Describe pod for events
kubectl describe pod <pod-name> -n emailengine

# Get logs
kubectl logs <pod-name> -n emailengine
kubectl logs <pod-name> -n emailengine --previous  # Previous container
```

### Common Issues

**Pods not starting:**
- Check secrets are created: `kubectl get secrets -n emailengine`
- Verify Redis connectivity from pod
- Check resource limits

**Redis connection errors:**
- Verify Redis service is running
- Check Redis URL in secret
- Test connectivity: `kubectl exec -it <pod> -- nc -zv redis 6379`

**High memory usage:**
- Increase memory limits
- Check account count vs resources
- Review worker configuration

## See Also

- [Docker Installation](/docs/installation/docker) - Docker and Docker Compose setup
- [Nginx Reverse Proxy](/docs/deployment/nginx-proxy) - The same streaming and forwarded-header rules for a proxy outside the cluster
- [Prepared Settings](/docs/configuration/prepared-settings) - Provisioning settings, tokens, the license and the password from Secrets
- [Performance Tuning](/docs/advanced/performance-tuning) - Optimize for large deployments
- [Monitoring](/docs/advanced/monitoring) - Set up Prometheus and Grafana
