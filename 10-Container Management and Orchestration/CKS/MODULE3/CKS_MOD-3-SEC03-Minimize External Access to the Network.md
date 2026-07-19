# Minimize External Access to the Network

This lab covers the **CKS exam curriculum topic "Minimize external access to the network"**, which in the official domain breakdown spans three layers:

1. **Cluster-level exposure control** — stopping workloads from being exposed externally in the first place (`ResourceQuota`, `loadBalancerSourceRanges`, Service types).
2. **Pod-to-pod / namespace traffic control** — `NetworkPolicy`, which is the layer most CKS candidates under-practice and the one most heavily tested.
3. **Host-level firewalling** — `ufw` / `iptables` on the nodes themselves.

---

## Lab Objectives

After completing this lab, you will be able to:

* Prevent developers from creating external `LoadBalancer` Services
* Restrict external IP ranges allowed to reach a `LoadBalancer`
* Use `ResourceQuota` to block Service types cluster- or namespace-wide
* Write `NetworkPolicy` resources to default-deny and selectively allow traffic
* Configure host firewall (`ufw`) without breaking cluster networking
* Use `iptables` to inspect and block specific traffic
* Verify and troubleshoot every control above

---

## Lab Environment

| Component  | Version                                  |
| ---------- | ----------------------------------------- |
| Ubuntu     | 22.04                                     |
| Kubernetes | v1.30+                                    |
| kubectl    | latest                                    |
| CNI        | Calico or Cilium (**required for NetworkPolicy** — Flannel alone does not enforce it) |
| ufw        | installed                                 |
| iptables   | installed                                 |

Verify your environment before starting:

```bash
kubectl version --client
kubectl get nodes
kubectl get pods -n kube-system -o wide   # confirm your CNI is running

ufw version
iptables --version
```

> **CNI check matters.** `NetworkPolicy` objects are only *enforced* if your CNI plugin supports it (Calico, Cilium, Weave). If you're on plain Flannel, the API will accept the objects but nothing will actually be blocked — a classic CKS exam trap.

---

## Lab Architecture

```
                    Internet
                        |
                 203.0.113.20
                        |
        ---------------------------------
        Kubernetes Worker Node
        ---------------------------------
          ufw (host firewall)
          iptables
                    |
        ---------------------------------
        Kubernetes Cluster
        ---------------------------------
        Namespace: security
          ResourceQuota (blocks LB Services)
          NetworkPolicy (blocks pod traffic)
          Services (ClusterIP / NodePort / LB)
```

---

# Part A — Restrict Service Exposure

## Lab 1 — Create Namespace

```bash
kubectl create namespace security
```

Verify:

```bash
kubectl get ns
```

## Lab 2 — Deploy a Sample Application

```bash
kubectl create deployment nginx --image=nginx -n security
```

Expose it internally only:

```bash
kubectl expose deployment nginx --port=80 --target-port=80 --type=ClusterIP -n security
```

Verify:

```bash
kubectl get all -n security
```

## Lab 3 — Create a LoadBalancer Service (the risk)

By default, any developer with namespace access can expose an app externally:

```bash
kubectl expose deployment nginx --type=LoadBalancer --port=80 -n security
```

```bash
kubectl get svc -n security
```

This creates a public-facing `EXTERNAL-IP` (or `<pending>` on bare-metal without a cloud LB controller) — this is exactly the exposure we want to prevent.

## Lab 4 — Block LoadBalancer Services with ResourceQuota

Clean up first:

```bash
kubectl delete svc nginx -n security
```

`quota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: service-quota
  namespace: security
spec:
  hard:
    services.loadbalancers: "0"
    services.nodeports: "0"     # optional: also block NodePort if required
```

Apply and verify:

```bash
kubectl apply -f quota.yaml
kubectl get resourcequota -n security
kubectl describe resourcequota service-quota -n security
```

Expected output includes:

```
Resource                 Used  Hard
--------                 ----  ----
services.loadbalancers   0     0
```

## Lab 5 — Verify the Restriction

```bash
kubectl expose deployment nginx --type=LoadBalancer --port=80 -n security
```

Expected result:

```
Error from server (Forbidden): error when creating ...: services.loadbalancers "nginx" is forbidden:
exceeded quota: service-quota, requested: services.loadbalancers=1,
used: services.loadbalancers=0, limited: services.loadbalancers=0
```

## Lab 6 — Confirm ClusterIP Still Works

```bash
kubectl expose deployment nginx --type=ClusterIP --port=80 -n security
kubectl get svc -n security
```

`ClusterIP` is unaffected by the `services.loadbalancers` quota.

## Lab 7 — NodePort Behavior

If you set `services.nodeports: "0"` in Lab 4, this will also be blocked:

```bash
kubectl delete svc nginx -n security
kubectl expose deployment nginx --type=NodePort --port=80 -n security
```

Expected (if quota includes NodePort):

```
Error from server (Forbidden): exceeded quota: service-quota
```

If you only quota'd `services.loadbalancers`, NodePort creation will still succeed — decide deliberately based on the exam scenario's wording.

## Lab 8 — Restrict LoadBalancer Source IP Ranges

Where you can't fully block `LoadBalancer` Services but need to restrict *who* can reach them, use `loadBalancerSourceRanges` (enforced by the cloud provider's LB / security group, not by Kubernetes itself — so it only works on cloud providers that honor this field):

`service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: secure-service
  namespace: security
spec:
  selector:
    app: nginx
  type: LoadBalancer
  loadBalancerSourceRanges:
    - 192.168.10.0/24
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f service.yaml
kubectl describe svc secure-service -n security
```

Expected:

```
LoadBalancer Ingress:     <provider-assigned-ip>
LoadBalancer Source Ranges: 192.168.10.0/24
```

Only that subnet is allowed to reach the LB (per cloud provider enforcement — on bare metal / kind clusters this field is accepted but not enforced without MetalLB or similar).

## Confirm the Admission Plugin Is Active

`ResourceQuota` enforcement depends on the `ResourceQuota` admission controller being enabled on the API server (it's on by default in kubeadm, EKS, GKE, AKS):

```bash
ps -ef | grep kube-apiserver | grep -o 'enable-admission-plugins=[^ ]*'
```

Or, if the API server runs as a static pod:

```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep -A2 admission-plugins
```

---

# Part B — NetworkPolicy (pod-to-pod traffic control)

This is the section most CKS candidates skip, and the one the exam weights most heavily under "minimize external access."

## Lab 9 — Default-Deny All Ingress in a Namespace

`default-deny-ingress.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: security
spec:
  podSelector: {}        # applies to all pods in the namespace
  policyTypes:
    - Ingress
```

```bash
kubectl apply -f default-deny-ingress.yaml
```

Verify nothing can reach the nginx pod anymore, even from inside the cluster:

```bash
kubectl run tmp-shell --rm -it --image=busybox -n security -- wget -qO- --timeout=3 nginx
```

Expected: the request hangs and times out — all ingress is now denied by default.

## Lab 10 — Allow Only Specific Pods (label-based)

`allow-frontend.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-nginx
  namespace: security
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - protocol: TCP
          port: 80
```

```bash
kubectl apply -f allow-frontend.yaml
```

Test with a labeled pod (should succeed):

```bash
kubectl run frontend --rm -it --image=busybox -n security -l role=frontend -- wget -qO- --timeout=3 nginx
```

Test with an unlabeled pod (should still be blocked):

```bash
kubectl run other --rm -it --image=busybox -n security -- wget -qO- --timeout=3 nginx
```

## Lab 11 — Restrict Cross-Namespace Traffic

Only allow traffic from a specific namespace (e.g., `monitoring`):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
  namespace: security
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 80
```

```bash
kubectl label ns monitoring kubernetes.io/metadata.name=monitoring --overwrite   # auto-set on k8s 1.21+
kubectl apply -f allow-from-monitoring.yaml
```

## Lab 12 — Default-Deny All Egress (block outbound/exfiltration)

This is the direct control for "minimize *external* access" — stopping pods from reaching the internet or other namespaces unless explicitly allowed:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: security
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

```bash
kubectl apply -f default-deny-egress.yaml
```

Verify outbound is blocked:

```bash
kubectl exec -n security deploy/nginx -- curl -m 3 -sS https://example.com
```

Expected: timeout.

> ⚠️ Don't forget DNS. A default-deny egress policy also blocks DNS resolution unless you explicitly allow it — this breaks almost everything and is a very common exam gotcha.

## Lab 13 — Allow DNS Egress (required alongside Lab 12)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: security
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

```bash
kubectl apply -f allow-dns-egress.yaml
```

## Lab 14 — Allow Egress to a Specific External CIDR Only

To let pods reach one external service (e.g., a payment API at `203.0.113.50/32`) while blocking everything else:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-external-api
  namespace: security
spec:
  podSelector:
    matchLabels:
      app: nginx
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 203.0.113.50/32
      ports:
        - protocol: TCP
          port: 443
```

```bash
kubectl apply -f allow-egress-external-api.yaml
```

Combined with Lab 12 + Lab 13, this pod can now reach *only* DNS and that one external IP — everything else, in or out, is denied.

---

# Part C — Host-Level Firewalling (ufw / iptables)

> ⚠️ **Read before running:** node-level firewall rules operate below the CNI overlay. Getting this wrong can sever cluster networking (etcd, kubelet, CNI VXLAN/BGP traffic) and is very hard to recover from remotely. Always keep an out-of-band console session open, and always allow SSH **before** enabling `deny incoming`.

## Lab 15 — Install and Check ufw

```bash
sudo apt update
sudo apt install ufw
sudo ufw status
```

Expected: `Status: inactive`

## Lab 16 — Allow Required Ports Before Enabling

Set the default policy and allow rules *before* turning ufw on — order matters, because `ufw enable` applies the default-deny immediately:

```bash
# SSH — never forget this or you'll lock yourself out
sudo ufw allow 22/tcp

# Kubernetes API server
sudo ufw allow 6443/tcp

# NodePort range (only if NodePort Services are in use)
sudo ufw allow 30000:32767/tcp

# kubelet API (control plane -> node)
sudo ufw allow 10250/tcp
```

If this node runs CNI overlay traffic, also allow the CNI's own ports, or nothing will schedule correctly across nodes:

```bash
# Calico (VXLAN mode)
sudo ufw allow 4789/udp

# Calico BGP (non-overlay mode)
sudo ufw allow 179/tcp

# Cilium (VXLAN)
sudo ufw allow 8472/udp

# Flannel (VXLAN)
sudo ufw allow 8472/udp
```

> Only allow the ports that match the CNI you actually run — allowing all of the above unconditionally just widens your attack surface unnecessarily.

## Lab 17 — Set Default Policy and Enable

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
```

Verify:

```bash
sudo ufw status verbose
```

Expected (abbreviated):

```
22/tcp                     ALLOW IN
6443/tcp                   ALLOW IN
30000:32767/tcp            ALLOW IN
10250/tcp                  ALLOW IN
```

## Lab 18 — Test the Firewall

From a machine outside the allowed rules:

```bash
curl --max-time 3 http://<worker-node-ip>
```

Expected: connection refused / timed out (port 80 was never opened).

Confirm SSH still works:

```bash
ssh user@<worker-node-ip>
```

## Lab 19 — Inspect and Modify with iptables

View current rules:

```bash
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v
sudo iptables-save > /tmp/iptables-backup.rules   # always back up first
```

Block a specific port directly:

```bash
sudo iptables -A INPUT -p tcp --dport 80 -j DROP
sudo iptables -L -n | grep 80
```

Remove the rule when done:

```bash
sudo iptables -D INPUT -p tcp --dport 80 -j DROP
```

> On kubeadm clusters, `kube-proxy` and the CNI plugin manage large numbers of `iptables` chains (`KUBE-SERVICES`, `cali-*`, etc.). Avoid flushing chains (`iptables -F`) on a live node — it will break Service routing cluster-wide.

## Lab 20 — Verify NodePort Reachability Through the Firewall

```bash
kubectl get svc -n security
```

Note the NodePort (e.g., `30080:32001/TCP`), then test:

```bash
curl --max-time 3 http://<worker-node-ip>:32001
```

* If ufw/iptables block it → `Connection refused` or timeout.
* If allowed and the Service/NetworkPolicy chain permits it → nginx welcome page.

---

# Verification & Troubleshooting Cheat Sheet

```bash
# Service exposure
kubectl get svc -A
kubectl get quota -A
kubectl describe quota -n <ns>

# NetworkPolicy
kubectl get networkpolicy -A
kubectl describe networkpolicy <name> -n <ns>
kubectl get pods -n <ns> --show-labels     # confirm podSelector matches

# Confirm your CNI actually enforces NetworkPolicy
kubectl get pods -n kube-system -o wide | grep -Ei 'calico|cilium|weave'

# Host firewall
sudo ufw status numbered
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v
sudo iptables-save

# API server admission plugins
ps -ef | grep kube-apiserver | grep -o 'enable-admission-plugins=[^ ]*'
```

Troubleshooting order when "external access" isn't behaving as expected:

1. **Service type / ResourceQuota** — is the Service even the type you think it is?
2. **NetworkPolicy** — `kubectl describe networkpolicy` and check `podSelector` labels match exactly (label selectors are case-sensitive and exact-match).
3. **CNI enforcement** — is a policy-enforcing CNI actually installed and healthy?
4. **kube-proxy / iptables chains** — `KUBE-SERVICES` chain routing correctly?
5. **Host firewall** — is ufw/iptables silently dropping the same traffic you're trying to test?

---

# CKS Exam Practice Tasks

### Task 1 — Namespace setup
Create a namespace named **finance**.
```bash
kubectl create ns finance
```

### Task 2 — Block LoadBalancer Services
Prevent any `LoadBalancer` Service from being created in **finance**.
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: no-lb
  namespace: finance
spec:
  hard:
    services.loadbalancers: "0"
```

### Task 3 — Verify the block
Confirm developers cannot create a `LoadBalancer` Service in **finance**.
Expected error contains: `Forbidden ... exceeded quota`

### Task 4 — Default-deny all ingress
In namespace **finance**, deny all incoming pod traffic by default.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: finance
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
```

### Task 5 — Allow only same-namespace traffic on port 8080
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-ns-8080
  namespace: finance
spec:
  podSelector: {}
  policyTypes: ["Ingress"]
  ingress:
    - from:
        - podSelector: {}
      ports:
        - protocol: TCP
          port: 8080
```

### Task 6 — Restrict egress to DNS + one external CIDR
Combine a default-deny egress policy with an allow rule for DNS (port 53) and `203.0.113.0/24` on port 443 only (see Labs 12–14).

### Task 7 — Configure host firewall
Requirements: allow SSH, allow the API server port, deny all other incoming (see Labs 16–17).

### Task 8 — Block a port with iptables
Drop inbound TCP traffic on port 8080 using `iptables`, then confirm and revert:
```bash
sudo iptables -A INPUT -p tcp --dport 8080 -j DROP
sudo iptables -L -n | grep 8080
sudo iptables -D INPUT -p tcp --dport 8080 -j DROP
```

### Task 9 — List NAT rules
```bash
sudo iptables -t nat -L -n -v
```

---

# Quick Reference — Common CKS Exam Commands

```bash
# Services
kubectl get svc -A
kubectl expose deployment nginx --type=ClusterIP --port=80
kubectl expose deployment nginx --type=NodePort --port=80
kubectl expose deployment nginx --type=LoadBalancer --port=80

# Quotas
kubectl get quota -A
kubectl describe quota -n <ns>

# NetworkPolicy
kubectl get networkpolicy -A
kubectl describe networkpolicy <name> -n <ns>
kubectl apply -f netpol.yaml

# Host firewall
sudo ufw status verbose
sudo ufw allow 6443/tcp
sudo ufw default deny incoming
sudo ufw enable

# iptables
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v
sudo iptables-save
```

---

# CKS Exam Tips

* `services.loadbalancers: "0"` on a `ResourceQuota` is the fastest way to prevent external exposure via a namespace — memorize this pattern.
* The `ResourceQuota` admission plugin is enabled by default on kubeadm, EKS, GKE, and AKS; only verify it manually if a quota isn't being enforced.
* `NetworkPolicy` is almost always the primary answer when a task says "restrict/minimize external access to pods" — reach for it before firewall rules.
* `NetworkPolicy` is **enforced by the CNI plugin**, not the API server. If policies "don't work," check the CNI first (Flannel does not enforce them).
* A `podSelector: {}` with no rules means "select everything" for the `policyTypes` listed, and an empty `ingress`/`egress` array under a default-deny policy means "allow nothing."
* Default-deny egress policies block DNS too — always pair them with an explicit DNS-allow rule, or workloads will fail mysteriously.
* `loadBalancerSourceRanges` is enforced by the **cloud provider's** load balancer/security group, not by Kubernetes — it does nothing on bare metal without something like MetalLB implementing it.
* Prefer `ClusterIP` by default; only use `NodePort`/`LoadBalancer` when the task explicitly requires external reachability.
* When editing host firewall rules on a live node, always allow SSH and back up existing `iptables` rules first — a mistake here can strand you outside the cluster with no rollback path.
