---
trigger: glob
globs: ["*.yaml", "*.yml", "**/k8s/**", "Chart.yaml", "values.yaml"]
---

# Kubernetes Best Practices

## Validation - MANDATORY

```bash
# Schema validation
kubeconform -strict manifests/
# Or legacy: kubeval manifests/

# Policy validation
kube-linter lint manifests/
datree test manifests/

# Dry run against cluster
kubectl apply --dry-run=server -f manifests/
```

## Security - NON-NEGOTIABLE

### Strictly Forbidden

| Setting | Why Forbidden |
|---------|--------------|
| `privileged: true` | Full host access |
| `allowPrivilegeEscalation: true` | Can gain root |
| `runAsUser: 0` | Running as root |
| `hostNetwork: true` | Bypasses network policies |
| `hostPID: true` | Can see all host processes |
| `hostIPC: true` | Can access host IPC |

### Required SecurityContext - MANDATORY

```yaml
apiVersion: v1
kind: Pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
```

### Pod Security Standards

Use namespace labels to enforce security:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

| Level | Description |
|-------|-------------|
| `privileged` | No restrictions (avoid) |
| `baseline` | Minimal restrictions |
| `restricted` | Hardened (recommended for prod) |

## Resources - MANDATORY

Always define requests and limits:

```yaml
resources:
  requests:
    memory: "128Mi"    # Scheduling guarantee
    cpu: "100m"        # 0.1 CPU cores
  limits:
    memory: "256Mi"    # OOMKilled if exceeded
    cpu: "500m"        # Throttled if exceeded (optional)
```

### Resource Guidelines

| Workload Type | Memory Request | Memory Limit | CPU Request | CPU Limit |
|--------------|----------------|--------------|-------------|-----------|
| Small API | 64-128Mi | 256Mi | 50-100m | 500m |
| Standard API | 256-512Mi | 1Gi | 100-250m | 1000m |
| Worker/Batch | 512Mi-1Gi | 2Gi | 250-500m | 2000m |
| Memory-intensive | 1-4Gi | 8Gi | 500m | 2000m |

**Best Practice:** Set memory limit = 2x memory request to handle spikes.

## Probes - MANDATORY for Production

### Probe Types

| Probe | Purpose | Failure Action |
|-------|---------|----------------|
| **Liveness** | Is container alive? | Restart container |
| **Readiness** | Can receive traffic? | Remove from Service |
| **Startup** | Has app started? | Delay other probes |

### Probe Configuration

```yaml
containers:
- name: app
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    initialDelaySeconds: 15
    periodSeconds: 10
    timeoutSeconds: 3
    failureThreshold: 3

  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 3
    failureThreshold: 3

  # For slow-starting apps
  startupProbe:
    httpGet:
      path: /healthz
      port: 8080
    initialDelaySeconds: 10
    periodSeconds: 10
    failureThreshold: 30  # 30 * 10s = 5 min max startup
```

### Health Endpoint Implementation

```go
// Go example
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    // Check critical dependencies
    if err := db.Ping(); err != nil {
        w.WriteHeader(http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
})

http.HandleFunc("/ready", func(w http.ResponseWriter, r *http.Request) {
    // Check if ready to receive traffic
    if !app.IsWarmedUp() {
        w.WriteHeader(http.StatusServiceUnavailable)
        return
    }
    w.WriteHeader(http.StatusOK)
})
```

## Labels & Annotations

### Recommended Labels

```yaml
metadata:
  labels:
    # Required
    app.kubernetes.io/name: my-app
    app.kubernetes.io/instance: prod-us-east-1
    app.kubernetes.io/version: "1.2.3"

    # Recommended
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: ecommerce
    app.kubernetes.io/managed-by: helm

    # Custom
    team: platform
    environment: production
```

### Useful Annotations

```yaml
metadata:
  annotations:
    # Documentation
    description: "User authentication service"
    owner: "platform-team@example.com"

    # Operational
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
```

## Service Accounts

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  annotations:
    # AWS IRSA
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-app-role
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: my-app
      automountServiceAccountToken: false  # Unless K8s API access needed
```

## Network Policies

Restrict pod-to-pod traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - port: 5432
```

## Pod Disruption Budget

Ensure availability during updates:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2        # Or: maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

## Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

## ConfigMaps & Secrets

```yaml
# ConfigMap for non-sensitive config
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-config
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
---
# Secret for sensitive data (use external secrets manager in prod)
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgres://user:pass@host/db"
```

**Best Practice:** Use external secrets (AWS Secrets Manager, Vault) via External Secrets Operator.

## Deployment Checklist

- [ ] SecurityContext configured (runAsNonRoot, drop ALL)
- [ ] Resources (requests/limits) defined
- [ ] Liveness and readiness probes configured
- [ ] Recommended labels applied
- [ ] ServiceAccount created (automountToken: false)
- [ ] NetworkPolicy restricting traffic
- [ ] PodDisruptionBudget for availability
- [ ] HPA for autoscaling
- [ ] Secrets managed externally (not in manifests)
- [ ] Pod Security Standards enforced on namespace
