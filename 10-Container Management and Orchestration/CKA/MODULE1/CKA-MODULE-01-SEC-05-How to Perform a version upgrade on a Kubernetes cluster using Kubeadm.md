# Comprehensive Guide: Upgrading Kubernetes Cluster Using Kubeadm

## Table of Contents
- [Overview](#overview)
- [Pre-Upgrade Checklist](#pre-upgrade-checklist)
- [Upgrade Control Plane Nodes](#upgrade-control-plane-nodes)
- [Upgrade Worker Nodes](#upgrade-worker-nodes)
- [Post-Upgrade Verification](#post-upgrade-verification)
- [Rollback Procedure](#rollback-procedure)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

---

## Overview

### Upgrade Workflow

The upgrade workflow at a high level consists of the following steps:

```
┌─────────────────────────────────────────────────────────────┐
│                  Kubernetes Upgrade Workflow                │
├─────────────────────────────────────────────────────────────┤
│  1. Upgrade Primary Control Plane Node                      │
│  2. Upgrade Additional Control Plane Nodes                  │
│  3. Upgrade Worker Nodes                                    │
└─────────────────────────────────────────────────────────────┘
```

### Components to Upgrade

| Component | Control Plane | Worker Nodes |
|-----------|---------------|--------------|
| **Kubeadm** | ✅ Yes | ✅ Yes |
| **Kubelet** | ✅ Yes | ✅ Yes |
| **Kubectl** | ✅ Yes | Optional |

### Version Compatibility Matrix

| Kubernetes Version | etcd Version | CoreDNS Version | Kubernetes Version |
|--------------------|--------------|-----------------|-------------------|
| v1.30.x | 3.5.10+ | v1.11.0+ | Compatible |
| v1.29.x | 3.5.9+ | v1.11.0+ | Compatible |
| v1.28.x | 3.5.9+ | v1.11.0+ | Compatible |

---

## Pre-Upgrade Checklist

### 1. Verify Cluster Health

```bash
# Check all nodes are Ready
kubectl get nodes

# Check all system pods are Running
kubectl get pods -A

# Check cluster health
kubectl cluster-info

# Check API Server health
curl -k https://localhost:6443/livez?verbose
```

### 2. Backup Important Data

```bash
# Backup etcd
sudo ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key

# Backup certificates
sudo tar -czf /backup/k8s-certs-$(date +%Y%m%d).tar.gz /etc/kubernetes/pki/

# Backup manifests
sudo tar -czf /backup/k8s-manifests-$(date +%Y%m%d).tar.gz /etc/kubernetes/manifests/

# Backup cluster configuration
sudo cp -r /etc/kubernetes/admin.conf /backup/admin.conf.$(date +%Y%m%d)
```

### 3. Ensure Sufficient Disk Space

```bash
# Check disk space
df -h

# Check inodes
df -i

# Check Docker/containerd disk usage
du -sh /var/lib/docker
du -sh /var/lib/containerd
```

---

## Upgrade Control Plane Nodes

### Step 1: Login to Control Plane

```bash
# SSH to the control plane node
ssh username@control-plane-node

# Switch to root or use sudo
sudo -i
```

### Step 2: Check Existing Versions

```bash
# Check current kubeadm version
kubeadm version -o json

# Check current kubelet version
kubelet --version

# Check current kubectl version
kubectl version --client

# Check Kubernetes version
kubectl version --short
```

### Step 3: Determine Upgrade Version

```bash
# Update package list
apt update

# Check available kubeadm versions
apt-cache madison kubeadm

# Run upgrade plan to see upgrade suggestions
kubeadm upgrade plan
```

**Example Output:**

```bash
$ kubeadm upgrade plan
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       TARGET
kubelet     1.29.x        1.30.x

Upgrade to the latest version in the v1.30 series:

COMPONENT                 CURRENT   TARGET
kube-apiserver            v1.29.0   v1.30.0
kube-controller-manager   v1.29.0   v1.30.0
kube-scheduler            v1.29.0   v1.30.0
kube-proxy                v1.29.0   v1.30.0
CoreDNS                   v1.11.0   v1.11.0
etcd                      3.5.9-0   3.5.9-0

You can now apply the upgrade by executing the following command:

    kubeadm upgrade apply v1.30.0
```

### Step 4: Unhold and Install Kubeadm

```bash
# Unhold kubeadm package
apt-mark unhold kubeadm

# Install desired kubeadm version (replace x with patch version)
apt-get update && apt-get install -y kubeadm='1.30.7-1.1'

# Hold kubeadm package to prevent accidental updates
apt-mark hold kubeadm

# Verify installation
kubeadm version
```

### Step 5: Apply Kubeadm Upgrade

```bash
# For the FIRST control plane node only
kubeadm upgrade apply v1.30.7

# For ADDITIONAL control plane nodes, use:
kubeadm upgrade node
```

**Example Upgrade Apply Output:**

```bash
$ kubeadm upgrade apply v1.30.7
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade/version] You have chosen to change the cluster version to "v1.30.7"
[upgrade/versions] Cluster version: v1.29.0
[upgrade/versions] kubeadm version: v1.30.7
[upgrade] Are you sure you want to proceed? [y/N]: y
[upgrade/prepull] Pulling images required for upgrading...
[upgrade/apply] Upgrading your Static Pod-hosted control plane...
[upgrade/etcd] Upgrading to TLS for etcd...
[upgrade/staticpods] Preparing for "etcd" upgrade
[upgrade/staticpods] Current and new manifests of etcd are equal, skipping upgrade
[upgrade/staticpods] Preparing for "kube-apiserver" upgrade
[upgrade/staticpods] Preparing for "kube-controller-manager" upgrade
[upgrade/staticpods] Preparing for "kube-scheduler" upgrade
[upgrade/staticpods] The phase "addons" succeeded
[upgrade] Successfully applied upgrade to the cluster!
```

### Step 6: Drain the Control Plane Node

```bash
# Drain the node (evict all workloads)
kubectl drain controlplane --ignore-daemonsets

# If using local storage, include delete-local-data
kubectl drain controlplane --ignore-daemonsets --delete-local-data

# For force drain if pods don't evict
kubectl drain controlplane --ignore-daemonsets --force
```

### Step 7: Upgrade Kubelet and Kubectl

```bash
# Unhold kubelet and kubectl
apt-mark unhold kubelet kubectl

# Install new versions
apt-get update && apt-get install -y kubelet='1.30.7-1.1' kubectl='1.30.7-1.1'

# Hold packages to prevent accidental updates
apt-mark hold kubelet kubectl

# Verify versions
kubelet --version
kubectl version --client
```

### Step 8: Restart Kubelet

```bash
# Reload systemd daemon
systemctl daemon-reload

# Restart kubelet
systemctl restart kubelet

# Verify kubelet status
systemctl status kubelet

# Check kubelet logs
journalctl -u kubelet -f --since "10 minutes ago"
```

### Step 9: Uncordon and Verify

```bash
# Uncordon the node (make it schedulable)
kubectl uncordon controlplane

# Verify node status
kubectl get nodes

# Verify node details
kubectl describe node controlplane

# Check control plane pods
kubectl get pods -n kube-system
```

### Step 10: Verify Cluster Health

```bash
# Check overall cluster health
kubectl get --raw='/readyz?verbose'

# Check API Server health
curl -k https://localhost:6443/livez?verbose

# Check component status
kubectl get componentstatuses

# Verify all system pods
kubectl get pods -A
```

---

## Upgrade Worker Nodes

### Step 1: Unhold and Install Kubeadm

```bash
# On each worker node
# Unhold kubeadm
apt-mark unhold kubeadm

# Install required kubeadm version
apt-get update && apt-get install -y kubeadm='1.30.7-1.1'

# Hold kubeadm
apt-mark hold kubeadm

# Verify installation
kubeadm version
```

### Step 2: Upgrade Kubeadm

```bash
# Run kubeadm upgrade on worker node
kubeadm upgrade node

# Expected output:
# [upgrade] Reading configuration from the cluster...
# [upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
# [upgrade] Upgrading node...
# [upgrade] The node has been upgraded successfully!
```

### Step 3: Drain the Worker Node

```bash
# From the control plane node
# Drain the worker node
kubectl drain worker-node01 --ignore-daemonsets

# If using local storage
kubectl drain worker-node01 --ignore-daemonsets --delete-local-data

# Force drain if needed
kubectl drain worker-node01 --ignore-daemonsets --force
```

**Example Drain Command Output:**

```bash
$ kubectl drain worker-node01 --ignore-daemonsets
node/worker-node01 cordoned
WARNING: ignoring DaemonSet-managed Pods: calico-node-xxx, kube-proxy-xxx
evicting pod default/nginx-xxx
pod/nginx-xxx evicted
node/worker-node01 drained
```

### Step 4: Upgrade Kubelet and Kubectl

```bash
# On the worker node
# Unhold kubelet and kubectl
apt-mark unhold kubelet kubectl

# Install new versions
apt-get update && apt-get install -y kubelet='1.30.7-1.1' kubectl='1.30.7-1.1'

# Hold packages
apt-mark hold kubelet kubectl

# Reload and restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# Verify kubelet status
systemctl status kubelet
```

### Step 5: Uncordon Worker Node

```bash
# From the control plane node
# Uncordon the worker node
kubectl uncordon worker-node01

# Verify node status
kubectl get nodes
```

---

## Post-Upgrade Verification

### Complete Cluster Verification

```bash
# Create verification script
cat << 'EOF' > verify-upgrade.sh
#!/bin/bash

echo "======================================"
echo "Kubernetes Upgrade Verification"
echo "======================================"

echo -e "\n1. Node Status..."
kubectl get nodes -o wide

echo -e "\n2. Node Versions..."
kubectl get nodes -o json | jq '.items[].status.nodeInfo.kubeletVersion'

echo -e "\n3. System Pods Status..."
kubectl get pods -n kube-system

echo -e "\n4. Cluster Health..."
kubectl get --raw='/readyz?verbose'

echo -e "\n5. Component Status..."
kubectl get componentstatuses

echo -e "\n6. API Server Health..."
curl -s -k https://localhost:6443/livez?verbose | head -10

echo -e "\n7. Test Deployment..."
kubectl create deployment test-nginx --image=nginx:alpine --replicas=2
sleep 10
kubectl get pods -l app=test-nginx
kubectl delete deployment test-nginx

echo -e "\n======================================"
echo "Verification Complete!"
echo "======================================"
EOF

chmod +x verify-upgrade.sh
./verify-upgrade.sh
```

### Check Component Versions

```bash
# Check Kubernetes version
kubectl version

# Check each component version
kubectl get nodes -o json | jq '.items[] | {name: .metadata.name, kubeletVersion: .status.nodeInfo.kubeletVersion}'

# Check control plane versions
kubectl get pods -n kube-system -o wide | grep -E "kube-apiserver|kube-controller|kube-scheduler"

# Check container runtime version
docker --version
containerd --version
```

---

## Rollback Procedure

### If Upgrade Fails

```bash
# 1. Check the issue
kubectl get nodes
kubectl get pods -A
journalctl -u kubelet -f

# 2. Revert kubeadm (if still in upgrade process)
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.29.x-*  # Previous version
apt-mark hold kubeadm

# 3. Revert kubelet and kubectl
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.29.x-* kubectl=1.29.x-*  # Previous version
apt-mark hold kubelet kubectl

# 4. Restart kubelet
systemctl daemon-reload
systemctl restart kubelet

# 5. Restore from backup (if necessary)
# Restore etcd snapshot
sudo ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot-YYYYMMDD.db \
  --data-dir /var/lib/etcd-backup

# Restore certificates
sudo tar -xzvf /backup/k8s-certs-YYYYMMDD.tar.gz -C /
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. **Kubeadm Upgrade Fails**

```bash
# Issue: "The connection to the server was refused"
# Solution: Check API server
kubectl cluster-info
systemctl status kubelet
kubectl get nodes

# Issue: "etcd cluster is not healthy"
# Solution: Check etcd
kubectl get pods -n kube-system | grep etcd
ETCDCTL_API=3 etcdctl endpoint health \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key
```

#### 2. **Node Not Ready After Upgrade**

```bash
# Check kubelet status
systemctl status kubelet
journalctl -u kubelet -f

# Check configuration
cat /var/lib/kubelet/config.yaml
cat /etc/default/kubelet

# Restart kubelet with debug
systemctl restart kubelet
systemctl status kubelet -l

# Check node conditions
kubectl describe node <node-name>
```

#### 3. **Pods Stuck in Terminating**

```bash
# Force delete pods
kubectl delete pod <pod-name> --force --grace-period=0

# Check why pods can't terminate
kubectl describe pod <pod-name>

# Check PDBs
kubectl get pdb -A
```

#### 4. **Certificate Issues**

```bash
# Check certificate expiration
kubeadm certs check-expiration

# Renew certificates if needed
kubeadm certs renew all

# Restart control plane components
kubectl delete pods -n kube-system -l component=kube-apiserver
```

#### 5. **API Server Not Responding**

```bash
# Check API server pod
kubectl get pods -n kube-system | grep apiserver
kubectl logs -n kube-system -l component=kube-apiserver

# Check API server service
kubectl get svc -n default kubernetes

# Test API server locally
curl -k https://localhost:6443/version
```

### Diagnostic Commands

```bash
# Comprehensive diagnostic script
cat << 'EOF' > diagnose-upgrade.sh
#!/bin/bash

echo "Kubernetes Upgrade Diagnostics"
echo "=============================="

echo -e "\n1. Node Status:"
kubectl get nodes

echo -e "\n2. Pod Status (all namespaces):"
kubectl get pods -A | grep -v Running

echo -e "\n3. Kubelet Status:"
for node in $(kubectl get nodes -o name); do
    echo "Node: $node"
    kubectl get --raw /api/v1/nodes/${node#node/}/proxy/healthz
    echo
done

echo -e "\n4. Control Plane Pods:"
kubectl get pods -n kube-system | grep -E "kube-apiserver|kube-controller|kube-scheduler"

echo -e "\n5. etcd Health:"
ETCDCTL_API=3 etcdctl endpoint health \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key

echo -e "\n6. Recent Events:"
kubectl get events -A --sort-by='.lastTimestamp' | tail -20

echo -e "\n7. Kubelet Logs (last 50 lines):"
journalctl -u kubelet -n 50 --no-pager
EOF

chmod +x diagnose-upgrade.sh
./diagnose-upgrade.sh
```

---

## Best Practices

### Pre-Upgrade Checklist

| Task | Status | Notes |
|------|--------|-------|
| Verify cluster health | ⬜ | `kubectl get nodes`, `kubectl get pods -A` |
| Backup etcd | ⬜ | Use etcdctl snapshot |
| Backup certificates | ⬜ | Backup /etc/kubernetes/pki |
| Backup manifests | ⬜ | Backup /etc/kubernetes/manifests |
| Check disk space | ⬜ | Ensure 20%+ free space |
| Review release notes | ⬜ | Check API changes |
| Verify compatibility | ⬜ | Check version matrix |
| Schedule maintenance window | ⬜ | Communicate with team |
| Review rollback procedure | ⬜ | Document rollback steps |
| Test in dev environment | ⬜ | Always test first |

### Upgrade Considerations

#### 1. **Version Compatibility**

```bash
# Check Kubernetes version compatibility
# Kubernetes supports upgrades from n-1 to n versions
# Example: v1.29.x → v1.30.x is supported

# Check API version changes
kubectl api-versions | sort
```

#### 2. **Maintenance Window**

```bash
# Plan for the following:
# - Control plane upgrade: 15-30 minutes per node
# - Worker node upgrade: 5-10 minutes per node
# - DNS propagation: 5-10 minutes
# - Application failover: 1-5 minutes
```

#### 3. **Rolling Upgrade Strategy**

```bash
# Sequential upgrade order:
# 1. Control plane nodes (one at a time)
# 2. Worker nodes (one at a time or in batches)

# Monitor during upgrade:
watch -n 5 'kubectl get nodes'
watch -n 5 'kubectl get pods -A'
```

#### 4. **Communication Plan**

```bash
# Notify stakeholders about:
# - Planned maintenance window
# - Expected downtime (if any)
# - Rollback procedure
# - Post-upgrade verification
```

### Post-Upgrade Checklist

| Task | Status | Command |
|------|--------|---------|
| Verify node versions | ⬜ | `kubectl get nodes -o json \| jq '.items[].status.nodeInfo.kubeletVersion'` |
| Verify all nodes Ready | ⬜ | `kubectl get nodes` |
| Verify system pods running | ⬜ | `kubectl get pods -A` |
| Verify application functionality | ⬜ | Test critical applications |
| Verify DNS resolution | ⬜ | `kubectl run test --image=busybox --rm -it -- nslookup kubernetes.default.svc.cluster.local` |
| Verify networking | ⬜ | Test pod-to-pod communication |
| Check monitoring | ⬜ | Verify monitoring stack working |
| Check logging | ⬜ | Verify log collection working |
| Document upgrade | ⬜ | Record any issues and resolution |

### Automated Upgrade Script

```bash
# Create automated upgrade script
cat << 'EOF' > upgrade-cluster.sh
#!/bin/bash

set -e

# Configuration
NEW_VERSION="1.30.7-1.1"
MASTER_NODES=("master1" "master2" "master3")
WORKER_NODES=("worker1" "worker2" "worker3")
KUBEADM_VERSION="v1.30.7"

echo "Starting Kubernetes Cluster Upgrade"

# Pre-upgrade backup
echo "Creating backups..."
./backup-cluster.sh

# Upgrade control plane nodes
for master in "${MASTER_NODES[@]}"; do
    echo "Upgrading master node: $master"
    
    # Upgrade kubeadm
    ssh $master "apt-mark unhold kubeadm && \
                 apt-get update && \
                 apt-get install -y kubeadm=${NEW_VERSION} && \
                 apt-mark hold kubeadm"
    
    # Apply upgrade
    if [ "$master" = "${MASTER_NODES[0]}" ]; then
        ssh $master "kubeadm upgrade apply ${KUBEADM_VERSION}"
    else
        ssh $master "kubeadm upgrade node"
    fi
    
    # Drain node
    kubectl drain $master --ignore-daemonsets
    
    # Upgrade kubelet and kubectl
    ssh $master "apt-mark unhold kubelet kubectl && \
                 apt-get update && \
                 apt-get install -y kubelet=${NEW_VERSION} kubectl=${NEW_VERSION} && \
                 apt-mark hold kubelet kubectl"
    
    # Restart kubelet
    ssh $master "systemctl daemon-reload && systemctl restart kubelet"
    
    # Uncordon node
    kubectl uncordon $master
done

# Upgrade worker nodes
for worker in "${WORKER_NODES[@]}"; do
    echo "Upgrading worker node: $worker"
    
    # Upgrade kubeadm
    ssh $worker "apt-mark unhold kubeadm && \
                 apt-get update && \
                 apt-get install -y kubeadm=${NEW_VERSION} && \
                 apt-mark hold kubeadm"
    
    # Run kubeadm upgrade
    ssh $worker "kubeadm upgrade node"
    
    # Drain node
    kubectl drain $worker --ignore-daemonsets
    
    # Upgrade kubelet
    ssh $worker "apt-mark unhold kubelet && \
                 apt-get update && \
                 apt-get install -y kubelet=${NEW_VERSION} && \
                 apt-mark hold kubelet"
    
    # Restart kubelet
    ssh $worker "systemctl daemon-reload && systemctl restart kubelet"
    
    # Uncordon node
    kubectl uncordon $worker
done

echo "Cluster upgrade completed successfully!"
echo "Verifying cluster status..."
./verify-upgrade.sh
EOF

chmod +x upgrade-cluster.sh
```

---

## Quick Reference

### Important Commands

| Action | Command |
|--------|---------|
| Check current version | `kubeadm version -o json` |
| Upgrade plan | `kubeadm upgrade plan` |
| Apply upgrade | `kubeadm upgrade apply v1.30.7` |
| Upgrade node | `kubeadm upgrade node` |
| Drain node | `kubectl drain node-name --ignore-daemonsets` |
| Uncordon node | `kubectl uncordon node-name` |
| Check node status | `kubectl get nodes` |
| Check cluster health | `kubectl get --raw='/readyz?verbose'` |
| Restart kubelet | `systemctl restart kubelet` |
| Check kubelet status | `systemctl status kubelet` |

---

*This guide provides comprehensive steps for upgrading Kubernetes clusters using kubeadm. Always test upgrades in a development environment first and maintain proper backups.*
