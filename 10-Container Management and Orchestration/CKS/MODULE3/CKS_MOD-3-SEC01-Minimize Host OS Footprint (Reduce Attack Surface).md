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

**Task:** On a worker node, identify and disable any non-essential services.

```
# List all enabled services
systemctl list-unit-files --type=service --state=enabled

# Common exam-bait services to check for and disable if not needed:
# - rpcbind, nfs-common, avahi-daemon, cups, bluetooth, telnet, rsh

sudo systemctl stop avahi-daemon
sudo systemctl disable avahi-daemon
sudo systemctl mask avahi-daemon   # prevents accidental restart

# Verify
systemctl status avahi-daemon
```

**Why `mask` matters:** `disable` prevents auto-start at boot, but another unit or dependency could still start it. `mask` symlinks the unit to `/dev/null`, so it can't be started at all — even manually — until unmasked. Exam graders often check for `masked`, not just `disabled`.

---

## Exercise 2: Restrict Kernel Modules

**Task:** Prevent loading of an unneeded/risky kernel module (classic exam example: `dccp`, `sctp`, or `bluetooth`).

```
# Check if module is currently loaded
lsmod | grep dccp

# If loaded, unload it
sudo modprobe -r dccp

# Blacklist it so it can't be loaded again
echo "install dccp /bin/true" | sudo tee /etc/modprobe.d/dccp-blacklist.conf

# Alternative blacklist syntax (also acceptable)
echo "blacklist dccp" | sudo tee -a /etc/modprobe.d/dccp-blacklist.conf

# Verify it can't load
sudo modprobe dccp
lsmod | grep dccp   # should show nothing
```

**Exam tip:** `install <module> /bin/true` is the more bulletproof method — it overrides the module load with a no-op command instead of just hiding it from automatic loaders. Know both syntaxes.

---

## Exercise 3: Minimize Open Ports on a Node

**Task:** Identify unexpected listening ports and lock them down with a host firewall.

```
# Baseline
sudo ss -tulnp

# Example: a debug service accidentally left listening on 8888
sudo ufw status
sudo ufw deny 8888/tcp
sudo ufw enable

# Or with iptables directly (more common in exam VMs, ufw isn't always installed)
sudo iptables -A INPUT -p tcp --dport 8888 -j DROP
sudo iptables -L -n
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

## Exercise 4: Remove Unnecessary Packages

**Task:** Purge compilers, package managers, and debug tools that shouldn't exist on a production node.

```
# Check what's installed that shouldn't be there
dpkg -l | grep -E 'gcc|make|telnet|tcpdump'

# Remove
sudo apt remove --purge -y telnetd
sudo apt autoremove -y
```

**Why it matters:** A compromised pod that escapes to the host shouldn't find a compiler to build exploits with, or `telnet`/`nc` to pivot further.

---

## Exercise 5: Harden SSH on Nodes

**Task:** Reduce SSH as an attack vector (common CKS host-hardening sub-task).

```
sudo vi /etc/ssh/sshd_config
```
Set:
```
PermitRootLogin no
PasswordAuthentication no
X11Forwarding no
```
```bash
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

## Exercise 8: Put It Together — Simulated Exam Task

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

## 3. Quick Self-Check Checklist

- [ ] Can you list and disable a systemd service, then explain disable vs mask?
- [ ] Can you blacklist a kernel module using both `install ... /bin/true` and `blacklist` syntax, from memory, without docs?
- [ ] Can you identify unexpected listening ports with `ss -tulnp` and block with iptables?
- [ ] Do you know which K8s ports must stay open ?
- [ ] Can you check/fix permissions on the container runtime socket?
- [ ] Can you name the 3-4 sysctl parameters worth knowing, and explain why `ip_forward` is dangerous to blindly disable on a real cluster node?

---

 
