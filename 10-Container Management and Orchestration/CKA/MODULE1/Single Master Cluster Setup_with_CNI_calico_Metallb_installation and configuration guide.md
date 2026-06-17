# Kubernetes Cluster Setup with Calico CNI and MetalLB Load Balancer

## Overview
This document explains how to build a Kubernetes single-master cluster using:
- kubeadm
- containerd runtime
- Calico CNI
- Single control-plane architecture

---

## Part 1: Initialize Kubernetes Cluster

```bash
sudo kubeadm join 192.168.172.131:6443 --token 2wkb9h.g8pufzls3ffunp1u --discovery-token-ca-cert-hash sha256:388f74a5a1c636dc52b5ecb14774151633f3f80f47883576d220c50c959d22be
```

---

## Part 2: Configure kubectl Access

### Create kubeconfig directory:
```bash
mkdir -p $HOME/.kube
```

### Copy admin config:
```bash
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

### Change ownership:
```bash
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## Part 3: Install Calico CNI

### Create directory and download manifests:
```bash
mkdir calico
cd calico
```

### Download Calico manifests:
```bash
wget https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/custom-resources.yaml
```

### Apply Calico manifest:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/calico.yaml
```

---

## Part 4: Configure Calico Network

### Custom Resources Configuration:
```yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr: 10.244.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
```

### Apply operator manifest:
```bash
kubectl create -f tigera-operator.yaml
```

### Apply Calico Custom Resources:
```bash
kubectl create -f custom-resources.yaml
```

---

## Part 5: Verify Cluster Status

### Check nodes:
```bash
kubectl get nodes
```

### Check pods:
```bash
kubectl get pods -A
```

---

## Part 6: MetalLB Load Balancer Setup and Testing Guide

### Overview
This document explains how to install, configure, and test MetalLB in a Kubernetes cluster. MetalLB provides LoadBalancer functionality for bare-metal Kubernetes clusters where cloud provider load balancers are not available.

---

## What is MetalLB?

MetalLB is a load-balancer implementation for:
- Bare-metal Kubernetes clusters
- On-premises Kubernetes environments
- VMware-based Kubernetes clusters
- VirtualBox or KVM Kubernetes labs
- Home lab Kubernetes setups

MetalLB assigns external IP addresses to Kubernetes services of type `LoadBalancer`.

---

## Prerequisites

### Verify Kubernetes Cluster
```bash
kubectl get nodes
```
All nodes should be in `Ready` status.

### Verify Pod Network
Ensure your CNI plugin is installed:
- Calico
- Flannel
- Cilium
- Weave

Check:
```bash
kubectl get pods -A
```

---

## Step 1: Install MetalLB

Install MetalLB using the official manifest:
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml
```

---

## Step 2: Verify MetalLB Installation

### Check MetalLB namespace:
```bash
kubectl get ns
```
Expected namespace: `metallb-system`

### Verify MetalLB Pods:
```bash
kubectl get pods -n metallb-system
```
Expected components:
- controller
- speaker

All pods should be `Running`.

---

## Step 3: Create MetalLB Configuration

### Create configuration file:
```bash
nano metallb-config.yaml
```

### Add the following configuration:
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.172.200-192.168.172.220
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-production
  namespace: metallb-system
spec:
  ipAddressPools:
  - production-pool
```

---

## Configuration Explanation

### IPAddressPool
Defines the external IP address range MetalLB can assign.

Example:
```yaml
addresses:
- 192.168.172.200-192.168.172.220
```
This creates a pool of usable LoadBalancer IPs.

### L2Advertisement
Enables Layer 2 mode advertisement. MetalLB uses ARP announcements to advertise external IPs on the local network.

### Important Network Requirement
The IP range must:
- belong to the same subnet/network
- not be used by DHCP
- not conflict with existing devices
- be reachable from your local network

Example:
If Kubernetes nodes are `192.168.172.x`, then MetalLB pool should also use `192.168.172.x`.

---

## Step 4: Apply MetalLB Configuration

Apply the configuration:
```bash
kubectl apply -f metallb-config.yaml
```

---

## Step 5: Verify MetalLB Resources

### Check IP address pools:
```bash
kubectl get ipaddresspools -n metallb-system
```

### Check Layer2 advertisements:
```bash
kubectl get l2advertisements -n metallb-system
```

---

## Step 6: Test MetalLB

### Deploy NGINX Application
```bash
kubectl create deployment nginx --image=nginx
```

### Verify deployment:
```bash
kubectl get deployments
```

### Expose Deployment as LoadBalancer
```bash
kubectl expose deployment nginx --port=80 --type=LoadBalancer
```

---

## Step 7: Verify External IP Assignment

### Check service:
```bash
kubectl get svc
```

Expected output:
```
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP
nginx        LoadBalancer   10.96.x.x       192.168.172.200
```

MetalLB automatically assigns an external IP from the configured pool.

---

## Step 8: Test Access

### From another machine in the same network:
```bash
curl http://192.168.172.200
```
Expected result: NGINX welcome page HTML output.

### Browser Test
Open browser: `http://192.168.172.200`

You should see: **Welcome to nginx!**

---

## Verify Service Details

### Describe service:
```bash
kubectl describe svc nginx
```
Check:
- external IP
- endpoints
- node port
- events

---

## Advanced Verification

### Check Speaker Logs
```bash
kubectl logs -n metallb-system -l component=speaker
```

### Check Controller Logs
```bash
kubectl logs -n metallb-system -l component=controller
```

---

## Common Troubleshooting

### Problem: External IP Pending
Example:
```
EXTERNAL-IP   <pending>
```

**Possible Causes:**
- MetalLB pods not running
- Incorrect IP pool
- Invalid subnet
- Missing L2Advertisement
- CNI networking issue

**Verify:**
```bash
kubectl get pods -n metallb-system
```

### Problem: IP Not Reachable

**Verify:**
- same network/subnet
- firewall rules
- no IP conflict
- node network connectivity

**Ping test:**
```bash
ping 192.168.172.200
```

### Problem: IP Conflict

Ensure DHCP server does not allocate the MetalLB IP range.

**Recommended:** Reserve a dedicated static range for MetalLB.

---

## Best Practices

### Recommended IP Range
Use a dedicated non-DHCP range.

Example:
```
192.168.172.200-192.168.172.220
```

### Production Recommendations
- Use monitoring for MetalLB
- Avoid overlapping DHCP ranges
- Use dedicated VLANs if possible
- Backup MetalLB configuration
- Use multiple worker nodes for HA

---

## Cleanup Test Resources

### Delete nginx service:
```bash
kubectl delete svc nginx
```

### Delete nginx deployment:
```bash
kubectl delete deployment nginx
```

---

## Remove MetalLB

### Delete MetalLB configuration:
```bash
kubectl delete -f metallb-config.yaml
```

### Remove MetalLB installation:
```bash
kubectl delete -f https://raw.githubusercontent.com/metallb/metallb/v0.15.3/config/manifests/metallb-native.yaml
```

---

## Summary

This guide covered:
- MetalLB installation
- IP pool configuration
- Layer2 advertisement setup
- LoadBalancer service creation
- External IP testing
- Troubleshooting
- Best practices

**MetalLB enables cloud-style LoadBalancer functionality for bare-metal Kubernetes environments.**
