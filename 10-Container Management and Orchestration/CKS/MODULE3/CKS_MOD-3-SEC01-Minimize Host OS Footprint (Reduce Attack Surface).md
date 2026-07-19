# Minimize Host OS Footprint (Reduce Attack Surface)

**Domain:** Cluster Hardening / System Hardening

---

##  

| Topic in a general hardening doc | CKS relevance | Action |
|---|---|---|
| Service disable/mask (`cups`, `avahi-daemon`) | ✅ Directly tested | Practice below |
| Kernel module blacklisting | ✅ Directly tested, often missing from general guides | Practice below |
| Open port / firewall auditing | ✅ Tested, but expect **iptables**, not ufw, on exam VMs | Practice below |
| Unnecessary package removal | ✅ Tested lightly | Practice below |
| SSH hardening (`PermitRootLogin no`, etc.) | ✅ Tested lightly | Practice below |
| Container runtime socket permissions | ✅ Tested (CKS-specific, not in general guides) | Practice below |
| Select sysctl kernel params (`ip_forward`, ASLR, SYN cookies) | ⚠️ Possible, lower priority | Included below, trimmed to relevant subset |
| `/tmp` `noexec,nosuid,nodev` mount options | ⚠️ Plausible but rare | Nice-to-know, not drilled here |
| Password policy / `chage` / `pam_pwquality` | ❌ Not tested | Skip for CKS prep |
| Fail2Ban | ❌ Not tested | Skip for CKS prep |
| Time sync (chrony/NTP) | ❌ Not tested | Skip for CKS prep |
| Lynis / debsums / compliance scanning | ❌ Not tested | Skip for CKS prep |
| NIST/ISO control mapping | ❌ Not tested | Skip for CKS prep |

---

## 1. What "Minimize Host OS Footprint" Actually Means

The exam tests whether you can:
1. Remove/disable unnecessary OS packages, services, and daemons on nodes
2. Restrict which kernel modules can load
3. Keep only required open ports/services listening
4. Reduce the attack surface of the container runtime itself
5. Use minimal base images/OS distros for nodes (conceptual, rarely hands-on)

Think: **"If a node were compromised, what could an attacker abuse?"** Anything unused is a liability.

---

## 2. Lab Setup

Any of these work — use whatever you have:
- `kubeadm` cluster (3+ VMs, real practice closest to exam)
- A plain Ubuntu VM if you just want to practice OS-level tasks without a cluster

```
# Example: check what's listening right now (baseline)
sudo ss -tulnp
sudo systemctl list-units --type=service --state=running
```

---

## Exercise 1: Audit and Disable Unnecessary Services
 
Goal: Any listening daemon is a potential entry point. Kubernetes nodes typically only need kubelet, container runtime, and CNI-related processes — not things like cups, avahi-daemon, rpcbind, nfs-server.

**Task:** On a worker node, identify and disable any non-essential services.

```
# List all enabled services
systemctl list-unit-files --type=service --state=enabled

# List active services
systemctl list-units --type=service --state=running



# List everything listening on the network
sudo ss -tulpn
# or
sudo netstat -tulpn

Example output might show rpcbind or avahi-daemon listening — these are almost never needed on a k8s node.

# Common exam-bait services to check for and disable if not needed:
# - rpcbind, nfs-common, avahi-daemon, cups, bluetooth, telnet, rsh

sudo systemctl stop avahi-daemon
sudo systemctl disable avahi-daemon
sudo systemctl mask avahi-daemon   # prevents accidental restart

# Verify
systemctl status avahi-daemon

sudo systemctl stop rpcbind
sudo systemctl disable rpcbind

Verify:

$ sudo ss -tulpn | grep -E 'avahi|rpcbind'
# should return nothing

```

**Why `mask` matters:** `disable` prevents auto-start at boot, but another unit or dependency could still start it. `mask` symlinks the unit to `/dev/null`, so it can't be started at all — even manually — until unmasked. Exam graders often check for `masked`, not just `disabled`.

---

## Exercise 2: Disable unused/unsafe kernel modules

Goal: Reduce kernel attack surface by blacklisting modules not needed for container workloads (rare protocols like DCCP, SCTP, RDS, TIPC are common CVE targets and rarely used).

```
# Check currently loaded modules
lsmod | grep -E 'dccp|sctp|rds|tipc'

# If loaded, unload it
sudo modprobe -r dccp

# Blacklist them so they can't be loaded (even by an attacker via container escape)
cat <<EOF | sudo tee /etc/modprobe.d/cks-blacklist.conf
install dccp /bin/false
install sctp /bin/false
install rds /bin/false
install tipc /bin/false
EOF

# Unload if currently loaded
sudo modprobe -r dccp sctp rds tipc 2>/dev/null || true

# Verify it can't load
sudo modprobe dccp
lsmod | grep dccp   # should show nothing

```

**Exam tip: Know that CKS may present this as "prevent module X from being loaded on the node" — the answer is a /etc/modprobe.d/*.conf blacklist file, not just rmmod, because rmmod alone doesn't survive reboot or reload attempts.

---

## Exercise 3: Minimize Open Ports on a Node

Goal: Use ufw/iptables to only permit the ports Kubernetes actually needs, blocking everything else.

Key ports to allow (control plane example, adjust for worker nodes):


6443 (API server)
2379-2380 (etcd, control plane only)
10250 (kubelet)
10259, 10257 (scheduler/controller-manager, control plane only)
Your CNI's ports (e.g., 8285/8472 for flannel VXLAN)
```
# Baseline
sudo ss -tulnp

 sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 22/tcp        # SSH (already hardened above)
sudo ufw allow 6443/tcp      # API server
sudo ufw allow 10250/tcp     # kubelet API
sudo ufw allow 2379:2380/tcp # etcd (control plane node only)

sudo ufw enable
sudo ufw status verbose
```
Verify from another node:
```
nc -zv <node-ip> 6443     # should succeed
nc -zv <node-ip> 23       # should fail/timeout (telnet port, closed)
```


Know the required Kubernetes control-plane/node ports so you don't accidentally block the cluster:

| Port | Component |
|---|---|
| 6443 | kube-apiserver |
| 2379-2380 | etcd |
| 10250 | kubelet API |
| 10257 | kube-controller-manager |
| 10259 | kube-scheduler |
| 30000-32767 | NodePort services |

---

## Exercise 4: Audit installed packages and remove unnecessary ones

Goal: Identify and remove packages that aren't required for the node to function (compilers, unused clients, legacy protocols).
 
```
# Check what's installed that shouldn't be there
dpkg -l | grep -E 'gcc|make|telnet|tcpdump'

Audit installed packages and remove unnecessary ones

Goal: Identify and remove packages that aren't required for the node to function (compilers, unused clients, legacy protocols).

# See what's installed
dpkg -l | less                     # Debian/Ubuntu
rpm -qa | less                     # RHEL/CentOS

# Common attack-surface packages to check for and remove if unused:
sudo apt list --installed | grep -E 'telnet|ftp|rsh-client|talk|nis|tftp'

# Remove them
sudo apt purge -y telnet ftp rsh-client talk nis tftp
sudo apt autoremove -y
```

**Why it matters:** A compromised pod that escapes to the host shouldn't find a compiler to build exploits with, or `telnet`/`nc` to pivot further.

---

## Exercise 5: Harden SSH on Nodes

***Goal: SSH is the most common node entry point. Restrict it hard.

```
sudo vi /etc/ssh/sshd_config
```
Set:
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowTcpForwarding no
MaxAuthTries 3
```
``` 
sudo systemctl restart sshd
```

---

## Exercise 6: Trim Kernel Network Parameters (sysctl)

**Task:** Apply the small subset of sysctl hardening that's plausible on CKS — don't try to memorize a full CIS sysctl list, just these.

```
sudo nano /etc/sysctl.d/99-cks-hardening.conf
```
Add:
```
# Disable IP forwarding unless this node is meant to route traffic
net.ipv4.ip_forward = 0

# Reject source-routed packets
net.ipv4.conf.all.accept_source_route = 0

# Ignore ICMP redirects (prevents MITM route injection)
net.ipv4.conf.all.accept_redirects = 0

# SYN flood protection
net.ipv4.tcp_syncookies = 1
```
```bash
sudo sysctl -p /etc/sysctl.d/99-cks-hardening.conf

# Verify
sudo sysctl net.ipv4.ip_forward net.ipv4.tcp_syncookies
```

**Careful:** if this node is a Kubernetes node using an overlay CNI (Calico, Flannel, etc.), `ip_forward` is usually required to be `1` for pod networking to work. Don't blindly disable it on a real cluster node — this exercise is best practiced on a standalone VM. On the exam, only change what the task explicitly asks for.

---

## Exercise 7: Container Runtime Footprint

**Task:** Ensure the container runtime socket is only accessible as needed, and that no unnecessary runtime is installed alongside the active one.

```
# Check permissions on the runtime socket
ls -l /run/containerd/containerd.sock

# Should not be world-writable
sudo chmod 660 /run/containerd/containerd.sock
```
---

## 8:  Run an automated OS hardening audit with Lynis

Goal: CKS expects familiarity with using a scanning tool to find hardening gaps (same pattern as kube-bench for k8s components, but for the OS itself).
```
bashsudo apt install -y lynis
sudo lynis audit system
```
Review the output — it flags things like:


Unnecessary running services
```
Weak SSH config
World-writable files
Missing kernel hardening (sysctl) settings
```

At the end it gives a hardening index score. Pick a few "suggestions" and remediate them as extra practice (e.g., set umask, disable core dumps).

---

## Exercise 9: Put It Together — Simulated Exam Task

> *"SSH into node `worker1`. The `cramfs` and `freevxfs` kernel modules must never be loaded. Additionally, the `xinetd` service must be stopped, disabled, and unable to be started again. Verify all changes persist after checking module load attempts and service status."*

**Solution:**
```
ssh worker1

# Kernel modules
for mod in cramfs freevxfs; do
  sudo modprobe -r $mod 2>/dev/null
  echo "install $mod /bin/true" | sudo tee /etc/modprobe.d/${mod}-blacklist.conf
done

# Verify
sudo modprobe cramfs; lsmod | grep cramfs   # empty = success

# Service
sudo systemctl stop xinetd
sudo systemctl disable xinetd
sudo systemctl mask xinetd

# Verify
systemctl is-active xinetd     # inactive
systemctl is-enabled xinetd    # masked
```

---
  
