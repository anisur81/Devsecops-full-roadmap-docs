# Use Appropriate Pod Security Standards Lab Practice Guide  
 
 
- ✅ Pod Security Admission (PSA)
- ✅ Security Context
- ✅ OPA Gatekeeper (policy enforcement)

---

## Lab Objectives

1. Understand the three Pod Security Standards
2. Configure Pod Security Admission on namespaces
3. Test privileged / non-compliant pods
4. Configure Security Context correctly
5. Enforce cluster-wide defaults via AdmissionConfiguration
6. Enforce custom policy using OPA Gatekeeper
7. Practice CKS-style exam scenarios

---

## Prerequisites

```bash
kubectl version --client
kubectl get nodes
kubectl get ns

# Create working namespaces used throughout this guide
kubectl create ns secure
kubectl create ns psa-lab
```

---

## 1. Pod Security Standards Overview

| Profile | Purpose |
|---|---|
| **Privileged** | No restrictions. Allows known privilege escalations. Used for system/infra pods. |
| **Baseline** | Prevents common privilege escalations; stays permissive for typical workloads. |
| **Restricted** | Strongest built-in protection; follows pod hardening best practices. |

### How it's enforced — Pod Security Admission (PSA)

PSA is applied at the **namespace** level via labels, with three independent modes:

| Mode | Behavior |
|---|---|
| `enforce` | Violating pods are **rejected** |
| `audit` | Violating pods are **allowed** but logged in the audit log |
| `warn` | Violating pods are **allowed** but a warning is returned to the client |

Label format:
```
pod-security.kubernetes.io/<MODE>: <LEVEL>
pod-security.kubernetes.io/<MODE>-version: <VERSION>   # optional, defaults to "latest"
```

### Admission flow

```
                 User
                  │
          kubectl apply pod.yaml
                  │
          Pod Security Admission
          ┌──────────────────┐
          │ enforce=restricted│
          └──────────────────┘
                  │
             Allow / Deny
                  │
              Kubernetes API
```

**CKS Exam tip:** Expect tasks that (a) label a namespace to enforce a given standard, and (b) fix a pod spec that violates `baseline`/`restricted`. Speed at editing YAML under time pressure matters more than reciting theory.

---

## 2. Lab 1 — Run a Privileged Pod (baseline check)

`privileged.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true
```

```bash
kubectl apply -f privileged.yaml -n secure
kubectl get pod -n secure
```

With no PSA label on the namespace yet, this pod runs fine — there's nothing to block it.

---

## 3. Lab 2 — Enforce `restricted` on a Namespace

```bash
kubectl label namespace secure \
  pod-security.kubernetes.io/enforce=restricted \
  --overwrite

kubectl get ns secure --show-labels
# pod-security.kubernetes.io/enforce=restricted
```

### Retry the privileged pod (should now be REJECTED)

```bash
kubectl delete pod privileged -n secure
kubectl apply -f privileged.yaml -n secure
```

Expected:
```
Error from server (Forbidden): pods "privileged" is forbidden:
violates PodSecurity "restricted:latest": privileged (container "nginx" must not
set securityContext.privileged=true), ...
```

The Pod Security Admission controller successfully blocked the pod.

---

## 4. Lab 3 — Enforce `baseline` on a Second Namespace

```bash
kubectl label namespace psa-lab \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/enforce-version=latest
```

```bash
kubectl apply -f privileged.yaml -n psa-lab
# Rejected — baseline also blocks privileged=true
```

This shows `baseline` catches the obvious violations even though it's more permissive than `restricted` overall (e.g., baseline doesn't require non-root or seccomp).

---

## 5. Lab 4 — Build and Deploy a Compliant `restricted` Pod

`secure-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-nginx
  labels:
    owner: admin
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault

  volumes:
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}

  containers:
  - name: nginx
    image: nginx:stable

    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL

    volumeMounts:
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /run

```

```bash
kubectl apply -f secure-pod.yaml -n secure
kubectl get pod -n secure
```

### Verify the security context took effect

```bash
kubectl describe pod secure-nginx -n secure
```

Look for:
```
RunAsNonRoot=True
AllowPrivilegeEscalation=False
ReadOnlyRootFilesystem=True
Capabilities: Drop = ALL
```

### Security Context field reference

| Setting | Meaning |
|---|---|
| `runAsNonRoot` | Container must not run as root |
| `runAsUser` | Runs with the specified UID |
| `readOnlyRootFilesystem` | Root filesystem cannot be modified |
| `allowPrivilegeEscalation` | Prevents a process from gaining more privileges than its parent |
| `privileged` | Grants full host access |
| `capabilities.drop` | Removes Linux capabilities |
| `seccompProfile` | Restricts allowed Linux syscalls |

---

## 6. Lab 5 — Targeted Violation Tests

These isolate exactly *which* field trips `restricted`, which is useful for building exam-speed intuition.

### 6a. Running as root explicitly

```yaml
# rootpod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: rootpod
spec:
  containers:
  - image: nginx
    name: nginx
    securityContext:
      runAsUser: 0
```
```bash
kubectl apply -f rootpod.yaml -n secure
# Forbidden: runAsUser=0 not allowed under restricted
```

### 6b. Privilege escalation allowed

```yaml
# escalation.yaml
apiVersion: v1
kind: Pod
metadata:
  name: escalation
spec:
  containers:
  - image: nginx
    name: nginx
    securityContext:
      allowPrivilegeEscalation: true
```
```bash
kubectl apply -f escalation.yaml -n secure
# Forbidden
```

### 6c. Host networking

```yaml
# hostnetwork.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostnetwork
spec:
  hostNetwork: true
  containers:
  - image: nginx
    name: nginx
```
```bash
kubectl apply -f hostnetwork.yaml -n secure
# Forbidden — blocked under both baseline and restricted
```

---

## 7. Lab 6 — Non-blocking Modes: `warn` and `audit`

Useful for migrating existing workloads without breaking them immediately.

```bash
kubectl label namespace psa-lab \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted \
  --overwrite
```

```bash
kubectl apply -f secure-pod.yaml -n psa-lab
# Allowed (baseline passes), but client shows:
# Warning: would violate PodSecurity "restricted:latest": ...
```

Enable audit log if disable


At first create the direcotory 
```
sudo mkdir -p /var/log/kubernetes
```
Create the auidit policy yaml file inside the /etc/kubernetes
```
$ vi /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata

```

Change the file kube-api yaml file /etc/kubernetes/manifests/kube-apiserver.yaml
```
   - command
     - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
     - --audit-log-path=/var/log/kubernetes/audit.log
     - --audit-log-maxage=30
     - --audit-log-maxbackup=10
     - --audit-log-maxsize=100

 volumeMounts:
 - mountPath: /etc/kubernetes/audit-policy.yaml
   name: audit-policy
   readOnly: true
 - mountPath: /var/log/kubernetes
   name: audit-log

  voulumes:
  - hostPath:
      path: /etc/kubernetes/audit-policy.yaml
      type: File
    name: audit-policy
  - hostPath:
      path: /var/log/kubernetes
      type: DirectoryOrCreate
    name: audit-log

```
The static pod (kube-apiserver) will automatically restarted.

Check audit log entries (path depends on cluster setup; commonly `/var/log/kubernetes/audit.log` on the control-plane node in kubeadm clusters):

```bash
grep "restricted" /var/log/kubernetes/audit.log | tail -5

log returned if the above labels matches for creating the pod

```

---

## 8. Lab 7 — Cluster-wide Default via AdmissionConfiguration

For clusters where you want a **default PSA level applied to every namespace** instead of labeling each one individually, configure the API server's admission plugin.

`/etc/kubernetes/psa-config.yaml` (on the control-plane node):
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "baseline"
      enforce-version: "latest"
      audit: "restricted"
      audit-version: "latest"
      warn: "restricted"
      warn-version: "latest"
    exemptions:
      usernames: []
      runtimeClasses: []
      namespaces: [kube-system]
```

Wire it into the API server (kubeadm static pod manifest, `/etc/kubernetes/manifests/kube-apiserver.yaml`):

```yaml
# add flag:
#   --admission-control-config-file=/etc/kubernetes/psa-config.yaml
# and a hostPath volume + volumeMount pointing at /etc/kubernetes/psa-config.yaml
```

```bash
# kubelet auto-restarts the static pod after the manifest change
watch kubectl get pod -n kube-system -l component=kube-apiserver
```

**CKS Exam tip:** This exact task (editing the static API server manifest to add an `AdmissionConfiguration` file + volume mount) has appeared in exam-style scenarios. A single YAML indentation error breaks the API server, so practice it under time pressure and always verify apiserver comes back healthy afterward.

---

## 9. Lab 8 — OPA Gatekeeper (Custom Policy Layer)

PSA covers the built-in standards; **OPA Gatekeeper** lets you enforce custom organizational policy on top (e.g., "every pod must have an `owner` label").

### 9a. Install Gatekeeper

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/v3.22.2/deploy/gatekeeper.yaml

kubectl get pods -n gatekeeper-system
# gatekeeper-controller-manager...
# gatekeeper-audit...
```

### 9b. Create a ConstraintTemplate

`k8srequiredlabels.yaml`:
```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels

      violation[{"msg": msg}] {
        not input.review.object.metadata.labels.owner
        msg := "owner label is required"
      }
```
```bash
kubectl apply -f k8srequiredlabels.yaml
```

### 9c. Create the Constraint

`constraint.yaml`:
```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: must-have-owner
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds:
      - Pod
```
```bash
kubectl apply -f constraint.yaml
```

### 9d. Test — pod without the required label (denied)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: testpod
spec:
  containers:
  - image: nginx
    name: nginx
```
```bash
kubectl apply -f testpod.yaml
# Denied: owner label is required
```

### 9e. Fix the pod

```yaml
metadata:
  labels:
    owner: anis
```
```bash
kubectl apply -f testpod.yaml
# Success
```

---

## 10. Quick Diagnosis Commands (exam speed-run)

```bash
# See why a pod was rejected (dry-run first!)
kubectl apply --dry-run=server -f pod.yaml

# Check current PSA labels on all namespaces
kubectl get ns -o custom-columns=NAME:.metadata.name,ENFORCE:.metadata.labels."pod-security\.kubernetes\.io/enforce"

# Remove PSA enforcement from a namespace (troubleshooting)
kubectl label namespace psa-lab pod-security.kubernetes.io/enforce-
```

---

## 11. Cleanup

```bash
kubectl delete ns secure psa-lab
kubectl delete -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.20/deploy/gatekeeper.yaml
```

---

## 12. CKS Exam Practice Questions

**Q1. Namespace `production` must only allow Restricted pods.**
```bash
kubectl label ns production pod-security.kubernetes.io/enforce=restricted
```

**Q2. Prevent root containers.**
```yaml
securityContext:
  runAsNonRoot: true
```

**Q3. Drop all Linux capabilities.**
```yaml
capabilities:
  drop:
  - ALL
```

**Q4. Disable privilege escalation.**
```yaml
allowPrivilegeEscalation: false
```

**Q5. Use the default seccomp profile.**
```yaml
seccompProfile:
  type: RuntimeDefault
```

---

## 13. Complete Reference Pod (copy-paste template)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
  labels:
    owner: admin
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginx:stable
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

---

## 14. CKS Exam Checklist

| Requirement | Configuration |
|---|---|
| Enforce Pod Security Standard | `pod-security.kubernetes.io/enforce=restricted` |
| Prevent root containers | `runAsNonRoot: true` |
| Run as specific UID | `runAsUser: 1000` |
| Disable privilege escalation | `allowPrivilegeEscalation: false` |
| Drop Linux capabilities | `capabilities.drop: ["ALL"]` |
| Enable seccomp | `seccompProfile.type: RuntimeDefault` |
| Read-only root filesystem | `readOnlyRootFilesystem: true` |
| Block privileged containers | `privileged: false` (or omit entirely) |
| Block host namespaces | no `hostNetwork` / `hostPID` / `hostIPC` |
| Block hostPath volumes | omit `hostPath` volumes entirely |
| Enforce organization-specific policy | OPA Gatekeeper `ConstraintTemplate` + `Constraint` |

### Baseline vs Restricted — side-by-side

| Requirement | baseline | restricted |
|---|---|---|
| `privileged: true` | ❌ blocked | ❌ blocked |
| `hostNetwork/hostPID/hostIPC` | ❌ blocked | ❌ blocked |
| `hostPath` volumes | ❌ blocked | ❌ blocked |
| Non-root enforced | not required | ✅ `runAsNonRoot: true` required |
| `allowPrivilegeEscalation` | not required | ✅ must be `false` |
| Capabilities | can't ADD dangerous ones | ✅ must `drop: ["ALL"]` |
| Seccomp profile | not required | ✅ `RuntimeDefault`/`Localhost` required |

### Important Notes

- **PSP was deprecated in v1.21 and fully removed in v1.25+.** Focus entirely on Pod Security Admission for modern clusters — don't waste study time on PSP syntax.
- The three profiles again: **Privileged** (no restrictions), **Baseline** (blocks common privilege escalation), **Restricted** (strongest built-in protection, most commonly tested in CKS labs).
- Combine **Pod Security Admission**, **Security Contexts**, and **OPA Gatekeeper** for layered defense — the exam expects you to know when each layer applies rather than relying on just one mechanism.
- **Practice goal:** be able to write a `restricted`-compliant pod spec from memory in under 90 seconds. 
