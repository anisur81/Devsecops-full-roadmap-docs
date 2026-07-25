#  Ensure Immutability of Containers at Runtime

## Why This Matters

A running container should never be modified after deployment. If an attacker
compromises a container, the first thing they typically try to do is:

- install tools (`curl`, `wget`, `vi`, `nano`, package managers)
- write to the filesystem to drop malware or persistence mechanisms
- escalate privileges to root
- add Linux capabilities to expand what they can do

Enforcing runtime immutability removes these options **even if the container
is compromised**, by making the filesystem read-only, blocking privilege
escalation, forcing a non-root user, and dropping unnecessary Linux
capabilities.

---

## Core Controls

| Control | Field | Effect |
|---|---|---|
| Read-only filesystem | `readOnlyRootFilesystem: true` | Container root FS cannot be written to |
| No privilege escalation | `allowPrivilegeEscalation: false` | Blocks `setuid`/`sudo`-style escalation inside container |
| Non-root user | `runAsNonRoot: true` | Refuses to start if image would run as UID 0 |
| Drop capabilities | `capabilities.drop: ["ALL"]` | Removes all Linux capabilities by default |

These four are the ones most commonly tested on the CKS exam and should be
memorized.

---

## Step 1 — Create a Pod with Immutable Runtime Settings

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: immutable-pod
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
```

> **Note:** `nginx` normally needs to write to `/var/cache/nginx`,
> `/var/run`, and needs root to bind port 80. On the real exam, if a task
> asks you to make a *specific* image immutable, you may also need to:
> - add `emptyDir` volumes for directories the app must write to (e.g. `/tmp`, `/var/cache/nginx`)
> - change the listening port to a non-privileged port (>1024) if `runAsNonRoot` is enforced

Example with writable scratch space via `emptyDir`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: immutable-pod-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
```

Apply it:

```bash
kubectl apply -f immutable-pod.yaml
```

---

## Step 2 — Verify the Pod Is Running

```bash
kubectl get pod immutable-pod
kubectl describe pod immutable-pod
```

If `runAsNonRoot: true` is set but the image's default user is root and no
`runAsUser` is specified, the pod will fail to start with an error like:

```
Error: container has runAsNonRoot and image will run as root
```

This is expected — fix by specifying `runAsUser` with a non-zero UID.

---

## Step 3 — Prove the Filesystem Is Read-Only

```bash
kubectl exec -it immutable-pod -- sh
```

Inside the container:

```bash
touch /tmp/test
# touch: /tmp/test: Read-only file system

echo hello > /etc/passwd
# Read-only file system
```

If you mounted `emptyDir` volumes (like `/tmp` or `/var/cache/nginx`),
writes to *those specific paths* will succeed — that's expected, since only
those paths are writable. Everything else on the root filesystem stays
locked.

---

## Step 4 — Prove Package Installation Fails

```bash
apk add curl
# or
apt install curl
```

Expected output:

```
Read-only file system
```

or

```
Permission denied
```

---

## Step 5 — Confirm the Container Is Not Running as Root

```bash
id
```

Expected:

```
uid=1000(...) gid=1000(...)
```

**Not:**

```
uid=0(root)
```

---

## Step 6 — Confirm Privilege Escalation Is Blocked

Try to escalate inside the container (this will fail if the image or exec
attempts anything requiring `setuid` binaries or extra privileges):

```bash
sudo su
# permission denied / command not found — no privilege escalation path
```

`allowPrivilegeEscalation: false` also sets the `no_new_privs` flag on the
process, which prevents any child process from gaining more privileges than
its parent — this blocks a large class of container breakout techniques
even if a `setuid` binary exists in the image.

---

## Step 7 — Confirm Capabilities Are Dropped

```bash
kubectl exec -it immutable-pod -- sh -c "cat /proc/1/status | grep Cap"
```

With `capabilities.drop: ["ALL"]`, the effective capability set should be
`0000000000000000`.

If your workload genuinely needs one capability (e.g. `NET_BIND_SERVICE` to
bind to port 80 as non-root), add it back explicitly rather than allowing
the full default set:

```yaml
securityContext:
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE
```

---

## Enforcing This Cluster-Wide

On the exam, a single pod spec is often just step one. You may also be
asked to **enforce** these settings cluster-wide so no one can deploy a
mutable/root/privileged container. Two mechanisms to know:

### Pod Security Admission (PSA) — built-in, exam-relevant

Label a namespace to enforce the `restricted` Pod Security Standard, which
requires (among other things) `runAsNonRoot`, blocks privilege escalation,
and requires dropping `ALL` capabilities:

```bash
kubectl label ns my-namespace \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest
```

Verify:

```bash
kubectl get ns my-namespace --show-labels
```

Then try deploying a pod without `readOnlyRootFilesystem`,
`runAsNonRoot`, etc., in that namespace — it should be **rejected** at
admission time.

### Legacy: PodSecurityPolicy (deprecated, know it exists but PSA is current)

If referenced on older exam material, PSP is removed as of Kubernetes 1.25+.
The current mechanism is Pod Security Admission (above) or an admission
controller like **Kyverno** / **OPA Gatekeeper** for more advanced/custom
policy enforcement.

---

## Quick Command Reference

```bash
# Create/apply an immutable pod
kubectl apply -f immutable-pod.yaml

# Check status
kubectl get pod immutable-pod
kubectl describe pod immutable-pod

# Exec in to test
kubectl exec -it immutable-pod -- sh

# Inside container — all of these should fail
touch /tmp/test
echo hi > /etc/passwd
apk add curl
id            # should NOT show uid=0

# Check capabilities
kubectl exec -it immutable-pod -- sh -c "cat /proc/1/status | grep Cap"

# Enforce at namespace level
kubectl label ns <namespace> pod-security.kubernetes.io/enforce=restricted
```

---

## CKS Exam Checklist for This Topic

- [ ] Know all four `securityContext` fields by heart:
      `readOnlyRootFilesystem`, `allowPrivilegeEscalation`, `runAsNonRoot`,
      `capabilities.drop`
- [ ] Know these can be set at **pod-level** (`spec.securityContext`) or
      **container-level** (`spec.containers[].securityContext`) —
      container-level overrides pod-level
- [ ] Know how to add `emptyDir` volumes for apps that need scratch write
      space despite a read-only root filesystem
- [ ] Know how to troubleshoot a pod that fails to start due to
      `runAsNonRoot` + a root-default image (fix: set `runAsUser`)
- [ ] Know how to verify immutability at runtime via `kubectl exec`
- [ ] Know Pod Security Admission labels for enforcing `restricted` at the
      namespace level
- [ ] Be fast at editing existing YAML/manifests under time pressure —
      practice adding these fields to an already-running workload's spec
      and redeploying
