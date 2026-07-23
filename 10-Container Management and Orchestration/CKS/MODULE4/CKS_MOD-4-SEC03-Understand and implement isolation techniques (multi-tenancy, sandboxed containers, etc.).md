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

#### Install and Configure kata container

Install zstd
```
sudo apt update
sudo apt install -y zstd
```
Download the correct package
```
export VERSION=3.32.0

wget https://github.com/kata-containers/kata-containers/releases/download/${VERSION}/kata-static-${VERSION}-amd64.tar.zst

Extract it
sudo tar --zstd -xvf kata-static-${VERSION}-amd64.tar.zst -C /
or
zstd -d kata-static-${VERSION}-amd64.tar.zst
sudo tar -xvf kata-static-${VERSION}-amd64.tar -C /
```
Expose the binaries to your default system runtime path by creating symbolic links

```
sudo ln -s /opt/kata/bin/kata-runtime /usr/local/bin/kata-runtime
sudo ln -s /opt/kata/bin/containerd-shim-kata-v2 /usr/local/bin/containerd-shim-kata-v2
```
Confirm the installation is completed
```
$ find / -name "containerd-shim-kata-v2" 2>/dev/null
/opt/kata/runtime-rs/bin/containerd-shim-kata-v2
/opt/kata/bin/containerd-shim-kata-v2
/usr/local/bin/containerd-shim-kata-v2

```

Check the following parameters
```
oracle@dockertest01:~/ISOLATION$ containerd --version
containerd containerd v2.2.6 11ce9d5f3c68c941867e82890e93e815c1304f1b
oracle@dockertest01:~/ISOLATION$ lscpu | grep Virtualization
Virtualization:                          VT-x
Virtualization type:                     full
```
Your environment looks suitable for Kata Containers:
```
containerd: v2.2.6 
CPU Virtualization: VT-x 
Virtualization type: full 
```

Now configure containerd

Since your config.toml contains:
```
version = 3
imports = ['/etc/containerd/conf.d/*.toml']
```
This is containerd 2.x configuration. Do not add the Kata runtime directly to /etc/containerd/config.toml. Instead, create a separate configuration file under /etc/containerd/conf.d/. That's what the imports directive is for.

Create a Kata runtime configuration
```
sudo mkdir -p /etc/containerd/conf.d

sudo tee /etc/containerd/conf.d/kata.toml >/dev/null <<'EOF'
[plugins."io.containerd.cri.v1.runtime".containerd.runtimes.kata]
  runtime_type = "io.containerd.kata.v2"
EOF

Restart containerd:

sudo systemctl restart containerd

Verify the restart succeeded:

sudo systemctl status containerd --no-pager
```
Create the Runtime Class
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

## LAB 8 — Install and configure the gVisor in on premises k8s 

### Install gVisor on Nodes 
 
 ```
 ARCH=$(uname -m)
 wget https://storage.googleapis.com/gvisor/releases/release/latest/${ARCH}/runsc
 wget https://storage.googleapis.com/gvisor/releases/release/latest/${ARCH}/containerd-shim-runsc-v1
 chmod +x runsc
 chmod +x containerd-shim-runsc-v1
 sudo mv runsc /usr/local/bin/
 sudo mv containerd-shim-runsc-v1 /usr/local/bin/
 hash -r
 runsc --version
 
 sudo tee /etc/containerd/conf.d/99-runsc.toml >/dev/null <<'EOF'
[plugins."io.containerd.cri.v1.runtime".containerd.runtimes.runsc]
  runtime_type = "io.containerd.runsc.v1"
EOF

 sudo systemctl restart containerd
 
 Verify installation
 runsc --version

 ``` 

### Create the RuntimeClass for the gvisor

```
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

``` 
kubectl get runtimeclass
```
Create sample pod using the gvisor runtimeclass

$ cat runtimeclass-gvisor.yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc

$ kubectl apply -f runtimeclass-gvisor.yaml
$ kubectl describe pod
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
 
