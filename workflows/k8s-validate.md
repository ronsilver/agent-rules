---
name: k8s-validate
description: Validate Kubernetes manifests
---

# Workflow: K8s Validate

Validate Kubernetes manifests for correctness, security, and best practices.

## Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| `kubectl` | K8s CLI | `brew install kubernetes-cli` |
| `helm` | Helm charts | `brew install helm` |
| `kubeval` | Schema validation | `brew install kubeval` |
| `kubeconform` | Schema validation (faster) | `brew install kubeconform` |
| `kube-linter` | Best practices | `brew install kube-linter` |
| `pluto` | Deprecated API detection | `brew install pluto` |

## Steps

### 1. Detect Manifest Type

| Marker File | Type | Tool Chain |
|-------------|------|------------|
| `Chart.yaml` | Helm | helm lint → helm template → kubeval |
| `kustomization.yaml` | Kustomize | kubectl kustomize → kubeval |
| `*.yaml` with `kind:` | Plain YAML | kubeval → kubectl dry-run |

### 2. Schema Validation

**Helm Charts:**
```bash
# // turbo
helm lint ./chart
# STOP if lint fails

# Template and validate
helm template ./chart > /tmp/rendered.yaml
kubeconform -strict -summary /tmp/rendered.yaml
# STOP if schema validation fails
```

**Kustomize:**
```bash
# // turbo
kubectl kustomize . > /tmp/rendered.yaml
# STOP if kustomize build fails

kubeconform -strict -summary /tmp/rendered.yaml
# STOP if schema validation fails
```

**Plain YAML:**
```bash
# // turbo
kubeconform -strict -summary *.yaml
# Or legacy: kubeval *.yaml
# STOP if schema validation fails
```

### 3. Dry Run Against Cluster (Optional)

```bash
# Client-side dry run (no cluster needed)
kubectl apply --dry-run=client -f manifests/

# Server-side dry run (requires cluster access)
kubectl apply --dry-run=server -f manifests/
# STOP if dry run fails
```

### 4. Security & Best Practices

```bash
# Kube-linter (comprehensive)
kube-linter lint manifests/
# STOP if HIGH severity issues found

# Check for deprecated APIs
pluto detect-files -d manifests/
# STOP if deprecated APIs for target K8s version
```

**Security Checklist - MANDATORY:**

| Check | Required Value | Why |
|-------|---------------|-----|
| `runAsNonRoot` | `true` | Prevent root container exploits |
| `readOnlyRootFilesystem` | `true` | Prevent filesystem tampering |
| `allowPrivilegeEscalation` | `false` | Block privilege escalation |
| `capabilities.drop` | `["ALL"]` | Minimize kernel capabilities |
| `resources.requests` | Defined | Enable proper scheduling |
| `resources.limits` | Defined | Prevent resource exhaustion |

**Example Secure SecurityContext:**
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  runAsGroup: 10001
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

### 5. Resource Validation

```bash
# Check for missing resources
grep -rL "resources:" manifests/*.yaml
# Should return empty (all files should have resources)
```

**Resource Guidelines:**
| Workload Type | CPU Request | Memory Request | CPU Limit | Memory Limit |
|--------------|-------------|----------------|-----------|--------------|
| Small service | 50m | 64Mi | 200m | 256Mi |
| Medium service | 100m | 128Mi | 500m | 512Mi |
| Large service | 250m | 256Mi | 1000m | 1Gi |

### 6. Probes Validation

```bash
# Check for missing probes (Deployments/StatefulSets)
grep -rL "livenessProbe\|readinessProbe" manifests/*.yaml
# Review any files without probes
```

**Probe Guidelines:**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

## Common Errors & Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `unknown field` | Wrong API version | Check K8s version compatibility |
| `missing required field` | Incomplete manifest | Add required fields |
| `deprecated API` | Old apiVersion | Update to new API (use `pluto`) |
| `invalid YAML` | Syntax error | Check indentation, use `yamllint` |
| `namespace not found` | Missing namespace | Create namespace first or use `-n` |

## Report Format

### Success
```
✅ K8s validation passed

Manifests: 12 files
Type: Helm chart
K8s Version: 1.29

Checks:
- Schema validation: ✓
- Security context: ✓
- Resources defined: ✓
- Probes configured: ✓
- No deprecated APIs: ✓
```

### Failure
```
❌ K8s validation failed

File: manifests/deployment.yaml
Error: spec.template.spec.containers[0].securityContext.runAsNonRoot is required

Fix: Add securityContext with runAsNonRoot: true
```
