# Kubernetes Cluster Clean Reset Guide

## Overview

This document provides a complete and clean procedure to reset a Kubernetes cluster node and remove all Kubernetes-related configurations, runtime data, networking components, and cached state.

This process is useful for:

- Rebuilding Kubernetes clusters
- Resetting lab or testing environments
- Reinitializing failed clusters
- Cleaning old Kubernetes installations
- Preparing a fresh node installation

---

## ⚠️ Warning

This process will permanently remove:

- Kubernetes cluster configuration
- etcd data
- Pod networking configuration
- Container runtime data
- kubeconfig files
- Cluster certificates
- Running workloads

**Use carefully on production systems.**

---

## Step 1: Reset Kubernetes Cluster

Run the `kubeadm reset` process:

```bash
sudo kubeadm reset -f
```

This removes:

- Kubernetes control plane configuration
- kubelet configuration
- cluster certificates
- iptables rules created by kubeadm

---

## Step 2: Remove Kubernetes Directories

Delete Kubernetes configuration and state directories:

```bash
sudo rm -rf /etc/kubernetes
sudo rm -rf /var/lib/etcd
sudo rm -rf /var/lib/kubelet
sudo rm -rf /var/lib/cni
sudo rm -rf /etc/cni
```

### Directory Explanation

| Directory | Purpose |
|-----------|---------|
| `/etc/kubernetes` | Kubernetes configuration files |
| `/var/lib/etcd` | etcd database storage |
| `/var/lib/kubelet` | kubelet runtime state |
| `/var/lib/cni` | Container Network Interface state |
| `/etc/cni` | CNI plugin configuration |

---

## Step 3: Remove Kubernetes Packages

Purge Kubernetes packages completely:

```bash
sudo apt-get purge -y kubeadm kubelet kubectl kubernetes-cni
sudo apt-get autoremove -y
```

Optional cleanup:

```bash
sudo apt-get autoclean -y
```

---

## Step 4: Clean Container Runtime Data

### Option A: Containerd Runtime Cleanup

Restart containerd service:

```bash
sudo systemctl restart containerd
```

Remove container runtime data:

```bash
sudo rm -rf /var/lib/containerd
```

Optional socket cleanup:

```bash
sudo rm -rf /run/containerd
```

### Option B: Docker Runtime Cleanup

Restart Docker service:

```bash
sudo systemctl restart docker
```

Remove Docker runtime data:

```bash
sudo rm -rf /var/lib/docker
```

Optional cleanup:

```bash
sudo rm -rf /var/run/docker*
```

---

## Step 5: Remove kubeconfig Files

Delete user kubeconfig files:

```bash
rm -rf $HOME/.kube
```

Remove Kubernetes admin and kubelet configuration:

```bash
sudo rm -f /etc/kubernetes/admin.conf
sudo rm -f /etc/kubernetes/kubelet.conf
```

---

## Step 6: Clean Network Configuration (Recommended)

Flush iptables rules:

```bash
sudo iptables -F
sudo iptables -t nat -F
sudo iptables -t mangle -F
sudo iptables -X
```

Clear IPVS tables (if IPVS mode was used):

```bash
sudo ipvsadm --clear
```

Remove network interfaces created by CNI plugins:

```bash
sudo ip link delete cni0
sudo ip link delete flannel.1
```

> **Note:** Some interfaces may not exist depending on the CNI plugin used.

---

## Step 7: Clean Residual Files

Remove temporary Kubernetes files:

```bash
sudo rm -rf /tmp/*
sudo rm -rf /var/tmp/*
```

Clear journal logs (optional):

```bash
sudo journalctl --vacuum-time=1s
```

---

## Step 8: Reboot the System

Reboot is highly recommended after cleanup:

```bash
sudo reboot
```

This ensures:

- stale mounts are removed
- network interfaces are refreshed
- container runtime state is cleared
- kubelet processes are terminated completely

---

## Step 9: Verify Clean State

After reboot, verify Kubernetes components are removed.

### Verify kubeadm

```bash
which kubeadm
```

### Verify kubelet

```bash
which kubelet
```

### Verify kubectl

```bash
which kubectl
```

### Verify no Kubernetes pods exist

For containerd:

```bash
sudo crictl ps -a
```

For Docker:

```bash
docker ps -a
```

---

## Optional: Full System Cleanup

Remove unused packages:

```bash
sudo apt autoremove -y
sudo apt autoclean -y
```

Clear cache memory:

```bash
sync && echo 3 | sudo tee /proc/sys/vm/drop_caches
```

Check disk usage:

```bash
df -h
```

---

## Recommended Verification Commands

### Check ports

```bash
sudo ss -tulpn
```

### Check running services

```bash
systemctl list-units --type=service
```

### Check mounted Kubernetes directories

```bash
mount | grep kube
```

---

## Summary

This guide performs:

- Kubernetes reset
- Package removal
- etcd cleanup
- CNI cleanup
- Runtime cleanup
- Network cleanup
- kubeconfig cleanup
- Optional system cleanup
- Preparation for fresh installation

**After completion, the node should be ready for a clean Kubernetes installation.**
