
# Kernel Hardening with AppArmor and seccomp

Goal: Restrict a container's ability to make outbound network calls (and reduce its syscall attack surface) using AppArmor mandatory access control and seccomp syscall filtering.


## 0. Kubernetes prerequisites

- A working Kubernetes cluster.
- Linux worker nodes — both Seccomp and AppArmor are Linux-node security features.
- kubectl configured with sufficient permissions.
- A CRI-compatible runtime such as containerd or CRI-O. Kubernetes documents containerd as supporting AppArmor.
- Kubelet running normally.
  
### Initial sanity checks for a CKS AppArmor + Seccomp lab.
```bash
# 1. Check AppArmor
sudo aa-status

# 2. Check Linux kernel
uname -r

# 3. Check Kubernetes nodes
kubectl get nodes -o wide

# 4. Check AppArmor kernel support
cat /sys/module/apparmor/parameters/enabled

# 5. Check Seccomp kernel support
grep -E 'CONFIG_SECCOMP|CONFIG_SECCOMP_FILTER' /boot/config-$(uname -r)

# 6. Check container runtime
containerd --version
runc --version

# 7. Check kubelet
systemctl is-active kubelet

# 8. Check AppArmor profiles currently loaded
sudo cat /sys/kernel/security/apparmor/profiles
```
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


**Step 1 — Create the profile on every node Custom profiles let you Deny specific syscalls — e.g., block `unshare`, `setns` etc  while allowing everything else.**

```bash
sudo mkdir -p /var/lib/kubelet/seccomp/
```
```
sudo vi /var/lib/kubelet/seccomp/deny-cis-based.json

{
  "defaultAction": "SCMP_ACT_ALLOW",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "comment": "Kernel module manipulation — container breakout / persistence",
      "names": ["init_module", "finit_module", "delete_module", "create_module"],
      "action": "SCMP_ACT_ERRNO"
    },
    {
      "comment": "Namespace manipulation — privilege escalation, breakout",
      "names": ["unshare", "setns"],
      "action": "SCMP_ACT_ERRNO"
    },
    {
      "comment": "Mount namespace tampering",
      "names": ["mount", "umount2", "pivot_root", "chroot"],
      "action": "SCMP_ACT_ERRNO"
    },
    {
      "comment": "Process tracing / debugging — credential theft, injection",
      "names": ["ptrace", "process_vm_readv", "process_vm_writev"],
      "action": "SCMP_ACT_ERRNO"
    },
    {
      "comment": "Kernel keyring — credential/secret exfiltration vector",
      "names": ["add_key", "request_key", "keyctl"],
      "action": "SCMP_ACT_ERRNO"
    },
    {
      "comment": "System state / host control — never needed by app containers",
      "names": ["reboot", "kexec_load", "kexec_file_load", "swapon", "swapoff"],
      "action": "SCMP_ACT_ERRNO"
    },
    {
      "comment": "BPF — kernel exploitation surface, container escape research",
      "names": ["bpf"],
      "action": "SCMP_ACT_ERRNO"
    },
    {
      "comment": "Kernel/CPU tunables — side-channel or perf-based attacks",
      "names": ["perf_event_open"],
      "action": "SCMP_ACT_ERRNO"
    },
    {
      "comment": "Time/clock tampering — can break logging, auth tokens, TLS validation",
      "names": ["clock_settime", "clock_adjtime", "adjtimex", "settimeofday", "stime"],
      "action": "SCMP_ACT_ERRNO"
    },
    {
      "comment": "Legacy/uncommon syscalls with history of kernel vulnerabilities",
      "names": ["acct", "vm86", "vm86old", "modify_ldt", "iopl", "ioperm", "syslog"],
      "action": "SCMP_ACT_ERRNO"
    }
  ]
}


```
### Create the yaml file for the above profile

```
pod-seccomp-deny-cis.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sc-deny-cis-based
  namespace: seccomp-lab
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: deny-cis-based.json
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]

```
```
# kubectl apply -f /var/lib/kubelet/seccomp/deny-cis-based.json
# kubectl get pods -n seccop-lab
```

### Create the pod using the following yaml file
```
# vi  pod-seccomp-deny-cis.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sc-deny-cis-based
  namespace: seccomp-lab
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: deny-cis-based.json
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
```
```
# kubectl apply -f /var/lib/kubelet/seccomp/deny-cis-based.json
# kubectl get pods -n seccop-lab
```

**Step 2 — Create the profile on every node Custom profiles let you Dobserve syscall activity before creating a restrictive profile.**

### Seccomp audit/logging profile

```json
# sudo vi /var/lib/kubelet/seccomp/audit.json
{
  "defaultAction": "SCMP_ACT_LOG"
}
```

Yaml file for the audit profile

```
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-audit
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: audit.json

  containers:
  - name: app
    image: busybox:1.36
    command:
    - sh
    - -c
    - sleep 3600
```

```
# kubectl apply -f seccomp-audit.yaml
# kubectl get pod seccomp-audit -n seccomp-lab
```

For your CKS practice, a good progression is:

```
SCMP_ACT_LOG
     ↓
Observe syscalls
     ↓
Identify unnecessary/dangerous syscalls
     ↓
Create restrictive profile
     ↓
SCMP_ACT_ERRNO
     ↓
Test violation
```
**Step 3 — Create the Custom Seccomp Profile on Every Kubernetes Node (Deny by Default).**

### Deny-by-default Seccomp profile

```json
# sudo vi /var/lib/kubelet/seccomp/deny-default.json
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

Create the yaml file for the deny default profile

```
# sudo vi pod-seccomp-deny-default.yaml
apiVersion: v1
kind: Pod
metadata:
  name: sc-violation
  namespace: seccomp-lab
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: deny-default.json
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]

```
```
# kubectl apply -f seccomp-deny-default.yaml
# kubectl get pod seccomp-deny-default -n seccomp-lab

```
**Step 4 — Prove the restriction works:**

```bash
kubectl exec -it sc-violation -n seccomp-lab -- sh
# Inside the pod, try a syscall NOT in the allow-list, e.g.:
unshare --map-root-user sh
# Expect: "unshare: unshare failed: Operation not permitted"

or

kubectl exec -it seccomp-net-test -- wget -T 3 http://example.com
# Expect: bad address / network unreachable — socket() blocked by seccomp

kubectl exec -it seccomp-net-test -- nslookup kubernetes.default
# Expect: failure — DNS resolution needs socket() too
```

### 1.4 Quick verification commands (exam speed tips)

```bash
# Confirm which profile a running pod is using
kubectl get pod <pod> -o=jsonpath='{.spec.securityContext.seccompProfile.type}{"\n"}'

# Check node-level default seccomp profile config for kubelet (if set)
grep -i seccomp /var/lib/kubelet/config.yaml 2>/dev/null

# List profiles staged on the node
ls -l /var/lib/kubelet/seccomp/profiles/

# seccomp
ls /var/lib/kubelet/seccomp/profiles/
kubectl exec <pod> -- cat /proc/1/status | grep Seccomp   # 2 = filter active

# General verification
kubectl describe pod <pod> | grep -A5 "Security Context"
sudo journalctl -k | grep -Ei "apparmor|seccomp|audit"

```
---

## Part 2 — AppArmor

### 2.1 Concepts to remember for the exam

- AppArmor is a **Linux Security Module (LSM)** — confines what a process (and thus a container) can do: file read/write, network, capabilities, mount, ptrace, etc. It's **path-based**, unlike SELinux (label-based).
- Only works on nodes with AppArmor support (Ubuntu/Debian ship it by default; check with `sudo aa-status` or `cat /sys/module/apparmor/parameters/enabled`).
- As of Kubernetes **1.30**, AppArmor is set via **`securityContext.appArmorProfile`** field (GA). On older clusters (pre-1.30 / pre-1.4 beta), it used the annotation `container.apparmor.security.beta.kubernetes.io/<container-name>: <profile-ref>` — **know both for the exam**, since exam cluster versions vary.
- Profile must be **loaded into the kernel on the node first** (`apparmor_parser`) — Kubernetes does not distribute AppArmor profiles for you.
- `type` values (1.30+): `RuntimeDefault`, `Localhost` (with `localhostProfile: <name>`), `Unconfined`.

### 2.2 Lab: Write, load, and verify an AppArmor profile

**Step 1 — Confirm AppArmor is active on the node:**

```bash
# Quick sanity checks
sudo aa-status                 # AppArmor loaded profiles / enforce mode
uname -r                       # kernel version (seccomp needs kernel >= 3.5, standard today)
kubectl get nodes -o wide
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
### OR
## Create the profile on every node that will run the workload:

sudo tee /etc/apparmor.d/k8s-deny-network <<'EOF'
#include <tunables/global>

profile k8s-deny-network flags=(attach_disconnected) {
  #include <abstractions/base>

  # Deny all network activity (address-family level block)
  deny network,

  # Still allow basic file reads so the app can start
  file,

  # Explicitly deny access to sensitive host paths
  deny /etc/shadow r,
  deny /root/** rwklx,

  # Deny mount/ptrace-style container breakout primitives
  deny mount,
  deny ptrace,
}
EOF

```


**Step 3 — Load it into the kernel:**

```bash
sudo apparmor_parser /etc/apparmor.d/k8s-deny-write
sudo aa-status | grep k8s-deny-write

or

sudo apparmor_parser -r -W /etc/apparmor.d/k8s-deny-network

# Verify it loaded
sudo aa-status | grep k8s-deny-network

```
Repeat Step 1.1–1.2 on every node in the cluster (or use a DaemonSet / config-management tool in real life).

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


or

kubectl exec -it apparmor-net-test -- wget -T 3 http://example.com
# Expect: connection refused / permission denied — network denied by AppArmor

kubectl exec -it apparmor-net-test -- ping -c1 8.8.8.8
# Expect: Operation not permitted

Check the kernel audit log on the node to confirm AppArmor is the one blocking it:

sudo dmesg | grep -i apparmor | tail -5
# or
sudo journalctl -k | grep DENIED

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

## Part 3 — Combine Both (Defense in Depth)
A realistic CKS-style task: "Harden pod restricted-app so it cannot make any outbound network calls, using both AppArmor and seccomp."

---

```
apiVersion: v1
kind: Pod
metadata:
  name: restricted-app
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/deny-network.json
    appArmorProfile:
      type: Localhost
      localhostProfile: k8s-deny-network
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

This layers:
- **seccomp** ? blocks the syscalls needed to open sockets
- **AppArmor** ? blocks network mediation + filesystem/mount/ptrace paths
- **capabilities: drop ALL** + **allowPrivilegeEscalation: false** ? standard
  CKS baseline hardening you should reflexively add to every restricted pod

 ---



## Part 4 — Cluster-wide Enforcement (Bonus, PSA-adjacent)

To make these profiles mandatory rather than opt-in per pod, CKS may test
**Pod Security Admission (PSA)** or an admission webhook / OPA Gatekeeper
constraint that rejects pods missing a seccomp profile. Quick PSA reminder:

```bash
kubectl label ns restricted-ns pod-security.kubernetes.io/enforce=restricted
```

The `restricted` PSS profile **requires** `seccompProfile.type` to be
`RuntimeDefault` or `Localhost` — pods without one are rejected. This is a
fast way to fail-closed at the namespace level instead of hoping every pod
spec remembers to set it.

```bash
# Test: this should be rejected because it has no seccompProfile
kubectl run no-seccomp --image=busybox -n restricted-ns -- sleep 3600
kubectl get events -n restricted-ns --field-selector reason=FailedCreate
```

---


## Part 5 — Cleanup

```bash
kubectl delete ns seccomp-lab
sudo apparmor_parser -R /etc/apparmor.d/k8s-deny-write
sudo rm /etc/apparmor.d/k8s-deny-write
sudo rm -f /var/lib/kubelet/seccomp/profiles/audit.json /var/lib/kubelet/seccomp/profiles/violation.json
```


# Part B — Restrict Service Exposure

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

---
