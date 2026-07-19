# Restrict Access to the Kubernetes API Server
 
This lab covers the layers CKS actually tests:
1. Network-level restriction (firewall on port 6443,10250,10257,10259)
2. Disabling anonymous authentication
3. Disabling the insecure port
4. RBAC least-privilege (users, service accounts)
5. Verifying with `kubectl auth can-i`
6. NodeRestriction admission plugin

---

## 0. Recon — see what you're working with

```
# Find the API server static pod manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Confirm the port it listens on
ss -tlnp | grep 6443

# Check current cluster-info
kubectl cluster-info
```

Every change to `kube-apiserver.yaml` is picked up automatically by kubelet (static pod) — no `kubectl apply` needed, just edit and save. Watch it restart:

```
watch crictl ps   # or docker ps, depending on runtime
```

---

## 1. Restrict network access to the API server (Control Plane Isolation & Firewall Rules)

Goal: only allow specific CIDRs (e.g. your bastion/admin subnet + worker nodes) to reach port 6443.


```
# Install UFW (Uncomplicated Firewall)
sudo apt update && sudo apt install ufw -y

# Set default deny policy for incoming traffic
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow essential Kubernetes control plane ports
sudo ufw allow 6443/tcp    # Kubernetes API server
sudo ufw allow 2379:2380/tcp  # etcd
sudo ufw allow 10250/tcp   # Kubelet API
sudo ufw allow 10259/tcp   # kube-scheduler
sudo ufw allow 10257/tcp   # kube-controller-manager

# Enable UFW
sudo ufw enable

# Check status
sudo ufw status verbose

```


**Test:**
``` 
# From an allowed host
curl -k https://<control-plane-ip>:6443/version

# From a disallowed host — should time out / connection refused
curl -k https://<control-plane-ip>:6443/version
```

> On the exam, this may instead be phrased as "use a security group / cloud firewall" — same idea, just enforced by the cloud provider instead of iptables. Know both.

---

## 2. Disable anonymous authentication

By default `--anonymous-auth` defaults to `true`. Anonymous requests get the `system:anonymous` user, group `system:unauthenticated` — dangerous if RBAC is loose.

**Check current behavior (before fix):**

``` 
curl -k https://<control-plane-ip>:6443/api/ -s | head

{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/api/\"",
  "reason": "Forbidden",
  "details": {},
  "code": 403
}

If you provide valid credentials

For example:

TOKEN=$(kubectl create token default)

curl -k \
-H "Authorization: Bearer $TOKEN" \
https://172.18.0.18:6443/api/

Then the API server authenticates you as the service account, and RBAC determines what that account can access.

```

**Fix — edit the static pod manifest:**

``` 
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```
Add/confirm this flag under `command:`:

``` 
    - --anonymous-auth=false
```

Save and wait for the apiserver pod to restart:

``` 
watch crictl ps
```

**Verify the fix:**

```
curl -k https://<control-plane-ip>:6443/api/ -s
# Expect: 401 Unauthorized instead of a served response
```

---
## 3. Ensure the insecure port is off (legacy but still asked)

Older API servers exposed `--insecure-port=8080` with **no auth at all**. On modern kubeadm clusters this is already removed/defaulted to 0, but verify:

```
grep insecure-port /etc/kubernetes/manifests/kube-apiserver.yaml
```
```
If present and non-zero, remove the flag or set it to `0`. There is no reason for this flag to exist in a hardened cluster.
--insecure-port=0              (Disable insecure port)
```
---
## Step 4. RBAC least-privilege (users, service accounts) We can follow the following section.

CKS_MOD-03-SEC-02-Using least-privilege identity and access management
---
## Step 5. Verify with `kubectl auth can-i` (the CKS bread-and-butter command)

```
# Should be yes
kubectl auth can-i get pods --as=system:serviceaccount:default:dev-user -n default

# Should be no — no create permission granted
kubectl auth can-i create pods --as=system:serviceaccount:default:dev-user -n default

# Should be no — no access outside its namespace
kubectl auth can-i get pods --as=system:serviceaccount:default:dev-user -n kube-system
```

---


## Step 6. Enable the NodeRestriction admission plugin

Ensures kubelets can only modify their **own** Node/Pod objects — prevents a compromised node from tampering with others.

```
grep enable-admission-plugins /etc/kubernetes/manifests/kube-apiserver.yaml
 - --enable-admission-plugins=NodeRestriction

```

Confirm it includes `NodeRestriction`:
```
    - --enable-admission-plugins=NodeRestriction,...
```

If missing, add it to the comma-separated list and save the manifest.

---



