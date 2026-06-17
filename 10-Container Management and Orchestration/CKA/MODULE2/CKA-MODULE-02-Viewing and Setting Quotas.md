# Viewing and Setting Quotas in Kubernetes

## Overview

Kubernetes ResourceQuotas are used to limit resource consumption within namespaces. This guide covers creating, updating, and viewing quotas using kubectl.

---

## 1. Creating a Namespace

First, create a namespace to isolate resources:

```bash
kubectl create namespace myspace
```

---

## 2. Creating Compute Resource Quotas

### Define Compute Resource Quota

Create a YAML file defining CPU, memory, and GPU limits:

```bash
cat <<EOF > compute-resources.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
    requests.nvidia.com/gpu: 4
EOF
```

### Apply the Quota

```bash
kubectl create -f compute-resources.yaml --namespace=myspace
```

---

## 3. Creating Object Count Quotas

### Define Object Count Quota

Limit the number of Kubernetes objects in the namespace:

```bash
cat <<EOF > object-counts.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
spec:
  hard:
    configmaps: "10"
    persistentvolumeclaims: "4"
    pods: "4"
    replicationcontrollers: "20"
    secrets: "10"
    services: "10"
    services.loadbalancers: "2"
EOF
```

### Apply the Quota

```bash
kubectl create -f object-counts.yaml --namespace=myspace
```

---

## 4. Viewing Quotas

### List All Quotas in Namespace

```bash
kubectl get quota --namespace=myspace
```

**Example Output:**
```
NAME                    AGE
compute-resources       30s
object-counts           32s
```

### Describe Specific Quotas

View detailed usage information:

```bash
kubectl describe quota compute-resources --namespace=myspace
```

```bash
kubectl describe quota object-counts --namespace=myspace
```

---

## 5. Priority-Based Quotas

### Create Priority Classes

Create multiple quotas for different priority levels:

```bash
vi quota.yaml
```

```yaml
apiVersion: v1
kind: List
items:
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-high
  spec:
    hard:
      cpu: "1000"
      memory: 200Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator: In
        scopeName: PriorityClass
        values: ["high"]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-medium
  spec:
    hard:
      cpu: "10"
      memory: 20Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator: In
        scopeName: PriorityClass
        values: ["medium"]
- apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: pods-low
  spec:
    hard:
      cpu: "5"
      memory: 10Gi
      pods: "10"
    scopeSelector:
      matchExpressions:
      - operator: In
        scopeName: PriorityClass
        values: ["low"]
```

### Apply Priority Quotas

```bash
kubectl create -f ./quota.yaml
```

### Verify Quota Status

```bash
kubectl describe quota
```

**Expected Output:**
```
Name:            pods-high
Namespace:       default
Resource         Used  Hard
--------         ----  ----
cpu              0     1000
memory           0     200Gi
pods             0     10

Name:            pods-medium
Namespace:       default
Resource         Used  Hard
--------         ----  ----
cpu              0     10
memory           0     20Gi
pods             0     10

Name:            pods-low
Namespace:       default
Resource         Used  Hard
--------         ----  ----
cpu              0     5
memory           0     10Gi
pods             0     10
```

---

## 6. Creating a High-Priority Pod

### Define High-Priority Pod

```bash
vi high-priority-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: high-priority
spec:
  containers:
  - name: high-priority
    image: ubuntu
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
    resources:
      requests:
        memory: "10Gi"
        cpu: "500m"
      limits:
        memory: "10Gi"
        cpu: "500m"
  priorityClassName: high
```

### Create the Pod

```bash
kubectl create -f ./high-priority-pod.yaml
```

### Verify Quota Usage Changes

Check that the "high" priority quota shows used resources:

```bash
kubectl describe quota
```

**Expected Output:**
```
Name:            pods-high
Namespace:       default
Resource         Used  Hard
--------         ----  ----
cpu              500m  1000
memory           10Gi  200Gi
pods             1     10

Name:            pods-medium
Namespace:       default
Resource         Used  Hard
--------         ----  ----
cpu              0     10
memory           0     20Gi
pods             0     10

Name:            pods-low
Namespace:       default
Resource         Used  Hard
--------         ----  ----
cpu              0     5
memory           0     10Gi
pods             0     10
```

---

## Key Points

| **Aspect** | **Details** |
|------------|-------------|
| **Scope** | Quotas are namespace-scoped |
| **Resource Types** | Compute (CPU, Memory, GPU) and Object Counts |
| **Priority Classes** | Support for different QoS levels |
| **Verification** | Use `describe quota` to see used vs hard limits |
| **Enforcement** | Pod creation fails if quota exceeded |

---

## Common Commands Summary

| **Command** | **Purpose** |
|-------------|-------------|
| `kubectl get quota --namespace=myspace` | List quotas in namespace |
| `kubectl describe quota <name>` | View detailed quota usage |
| `kubectl create -f quota.yaml` | Apply quota from file |
| `kubectl delete quota <name>` | Remove a quota |

---

## Troubleshooting Tips

1. **Pod Creation Fails**: Check if quota limits are exceeded
2. **Quota Not Applied**: Verify namespace and YAML syntax
3. **Used Resources Not Updating**: Check pod status (must be running)

---

*This guide demonstrates how to effectively manage resource quotas in Kubernetes for better resource governance and multi-tenant isolation.*
