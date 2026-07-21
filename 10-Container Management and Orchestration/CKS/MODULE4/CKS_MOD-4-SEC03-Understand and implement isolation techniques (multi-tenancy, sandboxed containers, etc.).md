#  Understand and implement isolation techniques (multi-tenancy, sandboxed containers, etc.)

## Topics Covered

- Multi-tenancy
- Namespace Isolation
- RBAC
- NetworkPolicy
- ResourceQuota
- LimitRange
- Pod Anti-Affinity
- Node Isolation
- RuntimeClass
- Kata Containers
- gVisor
- Sandbox Containers
- Verifying Isolation

---

## Lab Architecture

```
                 Kubernetes Cluster
+----------------------------------------------------+

 Node-1
 +----------------------------------------------+
 | Tenant-A Namespace                            |
 |                                                |
 | app-a-1                                        |
 | app-a-2                                        |
 +----------------------------------------------+

 Node-2
 +----------------------------------------------+
 | Tenant-B Namespace                            |
 |                                                |
 | app-b-1                                        |
 | app-b-2                                        |
 +----------------------------------------------+

            RuntimeClass
         -------------------
          runc
          kata
          gvisor

NetworkPolicy | RBAC | Quota | LimitRange
+----------------------------------------------------+
```

---

## Prerequisites

```bash
kubectl version --client
kubectl get nodes
kubectl cluster-info
```

**Expected:** 2 worker nodes minimum.

---

## LAB 1 — Namespace Isolation (Basic Multi-tenancy)

### Create two tenants

```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b
```

**Verify**

```bash
kubectl get ns
```

Expected output includes `tenant-a` and `tenant-b`.

### Deploy an application per tenant

Tenant A:

```bash
kubectl create deployment nginx \
  --image=nginx \
  -n tenant-a
```

Tenant B:

```bash
kubectl create deployment apache \
  --image=httpd \
  -n tenant-b
```

**Verify**

```bash
kubectl get pods -A
```

---

## LAB 2 — Resource Isolation

### Create a ResourceQuota

`quota.yaml`

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-quota
  namespace: tenant-a
spec:
  hard:
    pods: "5"
    requests.cpu: "2"
    requests.memory: 2Gi
    limits.cpu: "4"
    limits.memory: 4Gi
```

Apply and verify:

```bash
kubectl apply -f quota.yaml
kubectl describe quota -n tenant-a
```

### Create a LimitRange

`limitrange.yaml`

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: tenant-limit
  namespace: tenant-a
spec:
  limits:
  - default:
      cpu: 500m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    type: Container
```

Apply and verify:

```bash
kubectl apply -f limitrange.yaml
kubectl describe limitrange -n tenant-a
```

---

## LAB 3 — RBAC Isolation

### Create a ServiceAccount

```bash
kubectl create sa tenant-user -n tenant-a
```

### Create a Role

`role.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: tenant-a
  name: tenant-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

```bash
kubectl apply -f role.yaml
```

### Create a RoleBinding

`binding.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: tenant-binding
  namespace: tenant-a
subjects:
- kind: ServiceAccount
  name: tenant-user
  namespace: tenant-a
roleRef:
  kind: Role
  name: tenant-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f binding.yaml
```

### Verify

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:tenant-a:tenant-user \
  -n tenant-a
```

Expected: `yes`

Check cross-namespace access is denied:

```bash
kubectl auth can-i list pods \
  --as=system:serviceaccount:tenant-a:tenant-user \
  -n tenant-b
```

Expected: `no`

---

## LAB 4 — Network Isolation

### Default-deny NetworkPolicy

`networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

```bash
kubectl apply -f networkpolicy.yaml -n tenant-a
kubectl describe networkpolicy -n tenant-a
```

Traffic into and out of Tenant A is now denied unless explicitly allowed by another policy.

> **Note:** This requires a CNI plugin that enforces `NetworkPolicy` (e.g., Calico or Cilium). Kind's default networking does not enforce policies on its own.

---

## LAB 5 — Pod Anti-Affinity

Prevent Tenant-A pods from sharing a node with Tenant-B pods.

`deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tenant-a-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tenant-a
  template:
    metadata:
      labels:
        app: tenant-a
        tenant: tenant-a
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: tenant
                operator: In
                values:
                - tenant-b
            topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods -o wide
```

Pods should avoid nodes already running pods labeled `tenant=tenant-b`.

---

## LAB 6 — RuntimeClass Isolation

### View available RuntimeClasses

```bash
kubectl get runtimeclass
```

Example output:

```
NAME
kata
gvisor
runc
```

### Deploy a pod using a RuntimeClass

`pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kata-pod
spec:
  runtimeClassName: kata
  containers:
  - image: nginx
    name: nginx
```

```bash
kubectl apply -f pod.yaml
kubectl get pod
kubectl describe pod kata-pod
```

Look for `Runtime Class Name: kata` in the pod description.

---

## LAB 7 — Kata Containers (Sandboxed Containers)

### High-level install steps

1. Install containerd.
2. Install Kata Containers on each node.
3. Configure containerd to recognize the Kata runtime.
4. Restart containerd.
5. Create a `RuntimeClass` named `kata`.
6. Deploy a pod using `runtimeClassName: kata`.

`runtimeclass-kata.yaml`

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
```

```bash
kubectl apply -f runtimeclass-kata.yaml
```

Pod spec:

```yaml
spec:
  runtimeClassName: kata
```

Verify:

```bash
kubectl get runtimeclass
kubectl describe pod
```

---

## LAB 8 — gVisor

### Check runsc

```bash
runsc --version
```

### Configure containerd

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
runtime_type = "io.containerd.runsc.v1"
```

Restart containerd:

```bash
systemctl restart containerd
```

### RuntimeClass

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc
```

Deploy:

```yaml
spec:
  runtimeClassName: gvisor
```

Verify:

```bash
kubectl get runtimeclass
kubectl describe pod
```

---

## LAB 9 — Verify Isolation (End-to-End Checklist)

```bash
# Namespace isolation
kubectl get pods -A

# Quotas
kubectl describe quota -A

# LimitRange
kubectl describe limitrange -A

# Runtime isolation
kubectl get runtimeclass

# RBAC
kubectl auth can-i list pods --as=system:serviceaccount:tenant-a:tenant-user -n tenant-a
kubectl auth can-i list pods --as=system:serviceaccount:tenant-a:tenant-user -n tenant-b

# NetworkPolicy
kubectl get networkpolicy -A

# Node placement (anti-affinity)
kubectl get pods -o wide
```

---

## CKS Exam Practice Tasks

| # | Task |
|---|------|
| 1 | Create namespaces `tenant-a` and `tenant-b`. |
| 2 | Apply a `ResourceQuota` to `tenant-a` limiting it to 5 pods. |
| 3 | Configure a `LimitRange` with default CPU and memory requests/limits. |
| 4 | Create a ServiceAccount that can list pods only in `tenant-a`. |
| 5 | Apply a default-deny `NetworkPolicy` in `tenant-a`. |
| 6 | Create a deployment with `podAntiAffinity` to avoid scheduling alongside pods labeled `tenant=tenant-b`. |
| 7 | Create and use a `RuntimeClass` named `kata` (or `gvisor` if available). |
| 8 | Verify that the pod is running with the specified runtime class. |

**Tip:** In the real exam, always confirm object names, namespaces, and label selectors match exactly what's asked — grading is typically automated and literal.

---

## CKS Quick Revision Table

| Feature | Purpose | Kubernetes Object |
|---|---|---|
| Namespace | Logical tenant separation | `Namespace` |
| RBAC | API access isolation | `Role`, `RoleBinding` (or `ClusterRole`/`ClusterRoleBinding`) |
| NetworkPolicy | Network traffic isolation | `NetworkPolicy` |
| ResourceQuota | Limit namespace resource usage | `ResourceQuota` |
| LimitRange | Default/min/max resource settings | `LimitRange` |
| Pod Anti-Affinity | Prevent co-location of different tenants | Pod Affinity / Anti-Affinity rules |
| Node Isolation | Reserve nodes for specific workloads | Node labels, Taints/Tolerations, NodeAffinity |
| RuntimeClass | Select container runtime | `RuntimeClass` |
| Kata Containers | VM-based sandboxed containers | `RuntimeClass` + Kata runtime |
| gVisor | User-space kernel (syscall) isolation | `RuntimeClass` + `runsc` |

---

## Study Flow Summary

This progression mirrors the CKS exam objectives:

1. Start with **namespace-based multi-tenancy**.
2. Add **API isolation** (RBAC), **network isolation** (NetworkPolicy), and **resource isolation** (ResourceQuota/LimitRange).
3. Enforce **scheduling isolation** with pod anti-affinity and node isolation (taints/labels).
4. Finish with **sandboxed runtimes** (Kata Containers or gVisor) for the strongest workload-level isolation.

---

*End of module — practice each lab in order on a multi-node cluster (e.g., `kind` with 2+ workers) for the most realistic exam simulation.*
