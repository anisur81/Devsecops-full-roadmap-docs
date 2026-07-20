
# Kernel Hardening with AppArmor & seccomp


Goal: Practice restricting container syscalls and file/capability access using seccomp and AppArmor.


> Prerequisites: A running Kubernetes cluster where you control the **nodes** 
(kubeadm cluster, kind is NOT sufficient for AppArmor since it needs host kernel support). 
Nodes must run a Linux distro with AppArmor enabled (Ubuntu/Debian). 
For seccomp, any Linux node with a kernel >= 3.5 works (seccomp is enabled via kubelet, no special kernel module needed like AppArmor).

---

## Part 1 — seccomp

### 1.1 Concepts to remember for the exam

- Seccomp = **sec**ure **comp**uting mode. Filters **syscalls** a container can make.
- As of Kubernetes 1.19+, seccomp profiles are set via **`securityContext.seccompProfile`** (the old `seccomp.security.alpha.kubernetes.io` annotation is deprecated/removed).
- Three `type` values:
  - `RuntimeDefault` — use the container runtime's default profile (recommended baseline).
  - `Localhost` — use a custom JSON profile stored on the **node** at `/var/lib/kubelet/seccomp/profiles/<name>.json` (path depends on `--root-dir`; default kubelet root is `/var/lib/kubelet`, so profiles go under `/var/lib/kubelet/seccomp/`).
  - `Unconfined` — no filtering (default if unset, for legacy reasons — **exam gotcha**).
- Can be set at **Pod** level (`spec.securityContext.seccompProfile`) or **container** level (`spec.containers[].securityContext.seccompProfile`), container level overrides pod level.

### 1.2 Lab: Apply `RuntimeDefault`

```bash
kubectl create ns seccomp-lab
```

```yaml
# pod-seccomp-runtimedefault.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sc-runtimedefault
  namespace: seccomp-lab
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
```

```bash
kubectl apply -f pod-seccomp-runtimedefault.yaml
kubectl get pod sc-runtimedefault -n seccomp-lab -o jsonpath='{.spec.securityContext.seccompProfile}'
```

### 1.3 Lab: Build and apply a custom (`Localhost`) profile

Custom profiles let you **deny specific syscalls** — e.g., block `unshare` (used for namespace-based container escapes) while allowing everything else.

**Step 1 — Create the profile on every node** (in the exam, this is usually just the single node, but note it must exist on whichever node the pod schedules to):

```bash
sudo mkdir -p /var/lib/kubelet/seccomp/profiles
```

```json
// /var/lib/kubelet/seccomp/profiles/audit.json
{
  "defaultAction": "SCMP_ACT_LOG"
}
```

```json
// /var/lib/kubelet/seccomp/profiles/violation.json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": [
        "accept4","access","arch_prctl","bind","brk","clone","close",
        "connect","epoll_create1","epoll_ctl","epoll_pwait","execve",
        "exit_group","fcntl","fstat","futex","getcwd","getdents64",
        "getpid","getppid","listen","mmap","mprotect","munmap",
        "nanosleep","openat","poll","prctl","read","readlink",
        "rt_sigaction","rt_sigprocmask","rt_sigreturn","sched_yield",
        "set_robust_list","set_tid_address","setsockopt","sigaltstack",
        "socket","stat","write","writev"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

> `violation.json` is deliberately restrictive: it will **block `unshare`**, `mount`, `ptrace`, etc. because they're not in the allow-list, forcing `SCMP_ACT_ERRNO` (syscall fails with an error) instead of `SCMP_ACT_LOG` (just logs).

**Step 2 — Reference it from a Pod:**

```yaml
# pod-seccomp-violation.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sc-violation
  namespace: seccomp-lab
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: violation.json
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
```

```bash
kubectl apply -f pod-seccomp-violation.yaml
kubectl get pod sc-violation -n seccomp-lab
```

**Step 3 — Prove the restriction works:**

```bash
kubectl exec -it sc-violation -n seccomp-lab -- sh
# Inside the pod, try a syscall NOT in the allow-list, e.g.:
unshare --map-root-user sh
# Expect: "unshare: unshare failed: Operation not permitted"
```

Compare with the `audit.json` profile (logs to node's `dmesg`/audit log but doesn't block) to see the difference between **audit** and **enforce** modes — a common exam distinction.

### 1.4 Quick verification commands (exam speed tips)

```bash
# Confirm which profile a running pod is using
kubectl get pod <pod> -o=jsonpath='{.spec.securityContext.seccompProfile.type}{"\n"}'

# Check node-level default seccomp profile config for kubelet (if set)
grep -i seccomp /var/lib/kubelet/config.yaml 2>/dev/null

# List profiles staged on the node
ls -l /var/lib/kubelet/seccomp/profiles/
```

---

## Part 2 — AppArmor

### 2.1 Concepts to remember for the exam

- AppArmor is a **Linux Security Module (LSM)** — confines what a process (and thus a container) can do: file read/write, network, capabilities, mount, ptrace, etc. It's **path-based**, unlike SELinux (label-based).
- Only works on nodes with AppArmor support (Ubuntu/Debian ship it by default; check with `sudo aa-status` or `cat /sys/module/apparmor/parameters/enabled`).
- As of Kubernetes **1.30**, AppArmor is set via **`securityContext.appArmorProfile`** field (GA). On older clusters (pre-1.30 / pre-1.4 beta), it used the annotation `container.apparmor.security.beta.kubernetes.io/<container-name>: <profile-ref>` — **know both for the exam**, since exam cluster versions vary.
- Profile must be **loaded into the kernel on the node first** (`apparmor_parser`) — Kubernetes does not distribute AppArmor profiles for you.
- `type` values (1.30+ field): `RuntimeDefault`, `Localhost` (with `localhostProfile: <name>`), `Unconfined`.

### 2.2 Lab: Write, load, and verify an AppArmor profile

**Step 1 — Confirm AppArmor is active on the node:**

```bash
sudo aa-status
```

**Step 2 — Write a profile that denies all file writes:**

```bash
sudo tee /etc/apparmor.d/k8s-deny-write <<'EOF'
#include <tunables/global>

profile k8s-deny-write flags=(attach_disconnected) {
  #include <abstractions/base>

  file,
  # Explicitly deny write to any file
  deny /** w,
}
EOF
```

**Step 3 — Load it into the kernel:**

```bash
sudo apparmor_parser /etc/apparmor.d/k8s-deny-write
sudo aa-status | grep k8s-deny-write
```

**Step 4a — Reference it from a Pod (Kubernetes 1.30+, native field):**

```yaml
# pod-apparmor-native.yaml
apiVersion: v1
kind: Pod
metadata:
  name: aa-deny-write
  namespace: seccomp-lab
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    securityContext:
      appArmorProfile:
        type: Localhost
        localhostProfile: k8s-deny-write
```

**Step 4b — Reference it from a Pod (pre-1.30, annotation-based — still commonly tested):**

```yaml
# pod-apparmor-annotation.yaml
apiVersion: v1
kind: Pod
metadata:
  name: aa-deny-write-legacy
  namespace: seccomp-lab
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: localhost/k8s-deny-write
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
```

```bash
kubectl apply -f pod-apparmor-native.yaml
kubectl get pod aa-deny-write -n seccomp-lab
```

**Step 5 — Prove the restriction works:**

```bash
kubectl exec -it aa-deny-write -n seccomp-lab -- sh
# Inside the pod:
echo "test" > /tmp/testfile
# Expect: "sh: can't create /tmp/testfile: Permission denied"
```

### 2.3 Troubleshooting checklist (this is where exam time is lost)

1. **Pod stuck in `Blocked` / `CreateContainerError`** → profile not loaded on the node the pod scheduled to. Check with `kubectl describe pod <pod>` for the event message, then `sudo aa-status` on that specific node.
2. **Profile name mismatch** → the `localhostProfile` value must exactly match the `profile <name> {...}` name inside the file, not the filename (they're often the same by convention but don't have to be).
3. **Multi-node clusters** → AppArmor profiles are **node-local**; you must load the profile on **every node** the pod could schedule to, or use a `nodeSelector`/`DaemonSet`-based provisioning approach.
4. **Wrong K8s version assumption** → run `kubectl version` first; use annotation syntax for <1.30, field syntax for >=1.30.

### 2.4 Quick verification commands

```bash
# See which profile a pod is confined by, live on the node
sudo aa-status | grep -A2 <container-id-or-profile-name>

# From inside the container (if cat is available)
cat /proc/1/attr/current

# Kubernetes-side check (1.30+)
kubectl get pod <pod> -o jsonpath='{.spec.containers[0].securityContext.appArmorProfile}'
```

---

## Part 3 — Combined lab exercise (exam-style task)

**Task:** Create a Pod named `hardened-pod` in namespace `seccomp-lab` that:
1. Uses image `nginx:1.25`.
2. Applies the `RuntimeDefault` seccomp profile at the pod level.
3. Applies the custom AppArmor profile `k8s-deny-write` (deny all writes) at the container level.
4. Drops **all** Linux capabilities and adds back only `NET_BIND_SERVICE`.
5. Runs as non-root.

<details>
<summary>Solution (try it yourself first)</summary>

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
  namespace: seccomp-lab
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
    runAsNonRoot: true
    runAsUser: 101
  containers:
  - name: nginx
    image: nginx:1.25
    securityContext:
      appArmorProfile:
        type: Localhost
        localhostProfile: k8s-deny-write
      capabilities:
        drop: ["ALL"]
        add: ["NET_BIND_SERVICE"]
    ports:
    - containerPort: 80
```

> Note: `k8s-deny-write` as written above will actually break nginx (it needs to write logs/pid files), so in a real answer you'd scope the `deny` rule to specific paths rather than `/** w`. This is intentional — the exam often expects you to **debug why a hardened pod won't start** and loosen the profile appropriately, e.g.:
> ```
> deny /etc/** w,
> allow /var/log/nginx/** w,
> allow /var/run/nginx.pid w,
> ```

</details>

---

## Part 4 — Cleanup

```bash
kubectl delete ns seccomp-lab
sudo apparmor_parser -R /etc/apparmor.d/k8s-deny-write
sudo rm /etc/apparmor.d/k8s-deny-write
sudo rm -f /var/lib/kubelet/seccomp/profiles/audit.json /var/lib/kubelet/seccomp/profiles/violation.json
```

---
