# Website Uptime Monitoring and Availability

**Description:** Deploy Uptime Kuma for website uptime monitoring and alerting. Covers Docker and Kubernetes deployments.  
**Tags:** monitoring, uptime, alerting, availability, uptime-kuma, kubernetes, docker

## Goals

Deploy and configure Uptime Kuma for website uptime monitoring with automatic alerting.

## Quick Start

### Docker (Simple)

```bash
mkdir -p ~/uptime-kuma && cd ~/uptime-kuma
curl -o compose.yaml https://raw.githubusercontent.com/louislam/uptime-kuma/master/compose.yaml
docker compose up -d
# Access: http://localhost:3001
```

### Kubernetes (Production)

```bash
helm repo add uptime-kuma https://dirsigler.github.io/uptime-kuma-helm
helm repo update
helm install uptime-kuma uptime-kuma/uptime-kuma \
  --namespace monitoring --create-namespace \
  -f values.yaml
```

---

## Docker Deployment

### 1. Deploy

```bash
mkdir -p ~/uptime-kuma && cd ~/uptime-kuma
curl -o compose.yaml https://raw.githubusercontent.com/louislam/uptime-kuma/master/compose.yaml

# If port 3001 is in use, edit compose.yaml: change "3001:3001" to "3002:3001"
docker compose up -d
docker ps | grep uptime-kuma
```

### 2. Initial Setup

1. Open http://localhost:3001
2. Create admin account (no password recovery - store securely)
3. Enable 2FA: Settings → Security

### 3. Add Monitor

1. Click "Add New Monitor"
2. Configure:
   - **Type:** HTTP(s)
   - **URL:** https://your-site.com
   - **Interval:** 60 seconds
   - **Retries:** 3

### 4. Setup Notifications

Settings → Notifications → Setup Notification

**Slack:**
```yaml
Type: Slack
Webhook URL: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

**Email:**
```yaml
Type: Email (SMTP)
Host: smtp.gmail.com
Port: 587
```

```

### 5. Update

```bash
cd ~/uptime-kuma
docker compose pull
docker compose up -d
```

---

## Kubernetes Deployment

### 1. Create Values File

```yaml
# values.yaml
image:
  tag: "2"

volume:
  enabled: true
  size: 5Gi

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
  hosts:
    - host: status.yourdomain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: uptime-kuma-tls
      hosts:
        - status.yourdomain.com

serviceMonitor:
  enabled: false  # Enable if using Prometheus
```

### 2. Install

```bash
kubectl create namespace monitoring

helm install uptime-kuma uptime-kuma/uptime-kuma \
  --namespace monitoring \
  -f values.yaml

# Verify
kubectl get pods -n monitoring
kubectl get ingress -n monitoring
```

### 3. Access

```bash
# Via ingress (production)
open https://status.yourdomain.com

# Via port-forward (testing)
kubectl port-forward -n monitoring svc/uptime-kuma 3001:3001
open http://localhost:3001
```

### 4. Upgrade

```bash
helm repo update
helm upgrade uptime-kuma uptime-kuma/uptime-kuma \
  --namespace monitoring \
  -f values.yaml

# Rollback if needed
helm rollback uptime-kuma -n monitoring
```

---

## Production Checklist

- [ ] Enable 2FA for admin account
- [ ] Configure persistent storage
- [ ] Set up at least one notification channel
- [ ] Test notifications work
- [ ] Configure automated backups
- [ ] Set up external monitoring for Uptime Kuma itself
- [ ] Use HTTPS (reverse proxy or ingress with TLS)

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Port conflict | Change host port in compose.yaml |
| False positives | Increase retries to 3-5 |
| No alerts | Test notification channel, check monitor settings |
| High memory | Reduce monitors or increase check intervals |
| Container won't start | Check logs: `docker compose logs` |

## References

- [Uptime Kuma GitHub](https://github.com/louislam/uptime-kuma)
- [Helm Chart](https://artifacthub.io/packages/helm/uptime-kuma/uptime-kuma)
- [Notification Services](https://github.com/louislam/uptime-kuma/wiki/Notification-Services)