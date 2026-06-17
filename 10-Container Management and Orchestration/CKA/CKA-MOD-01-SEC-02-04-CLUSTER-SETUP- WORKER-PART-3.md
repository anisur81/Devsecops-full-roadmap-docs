# Comprehensive Guide: Kubernetes Worker Node Setup

## Table of Contents
- [Prerequisites](#prerequisites)
- [System Configuration](#system-configuration)
- [Container Runtime Installation](#container-runtime-installation)
- [Kubernetes Installation](#kubernetes-installation)
- [Worker Node Join Process](#worker-node-join-process)
- [Verification and Testing](#verification-and-testing)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

### System Requirements
- **OS**: Ubuntu 18.04/20.04/22.04 LTS
- **CPU**: 2+ cores
- **RAM**: 2+ GB
- **Storage**: 20+ GB
- **Network**: Reachable to master nodes

### Hosts File Configuration

```bash
# Edit /etc/hosts file
cat /etc/hosts
172.200.211.104 master1
172.200.211.105 worker1
172.200.211.106 worker2
```

### Set Hostname

```bash
# Set hostname for the worker node
hostnamectl set-hostname worker01

# Verify hostname change
hostnamectl
bash
```

---

## System Configuration

### Disable Swap

```bash
# Temporarily disable swap
swapoff -a

# Permanently disable swap (comment out swap line in /etc/fstab)
vi /etc/fstab
# Add '#' before the swap line:
# /swapfile none swap sw 0 0

# Verify swap is disabled
free -m
```

### Load Required Kernel Modules

```bash
# Create kernel modules configuration file
vi /etc/modules-load.d/k8s.conf
```

**Add the following content:**

```
# /etc/modules-load.d/k8s.conf
overlay
br_netfilter
```

```bash
# Load the modules
modprobe overlay
modprobe br_netfilter

# Verify modules are loaded
lsmod | grep overlay
lsmod | grep br_netfilter
```

### Configure System Parameters

```bash
# Create sysctl configuration
vi /etc/sysctl.d/10-k8s.conf
```

**Add the following sysctl parameters:**

```
# /etc/sysctl.d/10-k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

```bash
# Apply sysctl parameters
sysctl --system

# Verify settings
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward
```

---

## Container Runtime Installation

### Install Docker

```bash
# Install prerequisites
apt install apt-transport-https ca-certificates curl software-properties-common -y

# Add Docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package list
apt update -y

# Install Docker
apt install docker-ce -y

# Start and enable Docker
systemctl enable docker --now

# Verify Docker installation
docker --version
```

### Configure Docker for Kubernetes

```bash
# Create Docker daemon configuration
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

# Restart Docker
systemctl restart docker
systemctl status docker
```

---

## Kubernetes Installation

### Install Kubernetes Components

```bash
# Add Kubernetes repository GPG key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package list
apt update -y

# Install kubeadm and kubelet (no kubectl on workers)
apt install kubeadm kubelet -y

# Hold packages to prevent automatic updates
apt-mark hold kubeadm kubelet
```

### Configure Kubelet

```bash
# Edit kubelet configuration
vi /etc/default/kubelet
```

**Add the following:**

```
# /etc/default/kubelet
KUBELET_EXTRA_ARGS="--cgroup-driver=cgroupfs"
```

```bash
# Reload systemd and restart kubelet
systemctl daemon-reload
systemctl restart kubelet
systemctl status kubelet
```

### Clean Containerd Configuration

```bash
# Remove default containerd config to use Docker
rm -rf /etc/containerd/config.toml

# Restart containerd
systemctl restart containerd

# Verify containerd status
systemctl status containerd
```

---

## Worker Node Join Process

### Get Join Command from Master

**On Master Node:**

```bash
# Generate join token if needed
kubeadm token create --print-join-command

# List existing tokens
kubeadm token list

# Example output:
# kubeadm join 172.200.211.104:6443 --token v73r8y.kb435vyius9i0jp7 --discovery-token-ca-cert-hash sha256:dc2f22d55d09f23dc46c809cfc81801805cb31f95b3ac9ee300fb331f90420fb
```

### Join Worker to Cluster

**On Worker Node:**

```bash
# Join the cluster using the command from master
kubeadm join 172.200.211.104:6443 \
    --token v73r8y.kb435vyius9i0jp7 \
    --discovery-token-ca-cert-hash sha256:dc2f22d55d09f23dc46c809cfc81801805cb31f95b3ac9ee300fb331f90420fb

# For multi-master cluster with load balancer
kubeadm join 172.21.1.173:6443 \
    --token x1dmh5.w0nnknskd3d5s0vy \
    --discovery-token-ca-cert-hash sha256:2bbd80eb3549cccb6c2f944416cf092db291538e7c883f51dcc5ca530b011498
```

### Verify Join Status

```bash
# Check kubelet status after joining
systemctl status kubelet

# View kubelet logs
journalctl -u kubelet -f
```

---

## Verification and Testing

### Verify on Master Node

```bash
# List all nodes in the cluster
kubectl get nodes -o wide

# Expected output:
# NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP     EXTERNAL-IP
# master1    Ready    control-plane   10m   v1.30.0   172.200.211.104 <none>
# worker01   Ready    <none>          2m    v1.30.0   172.200.211.105 <none>

# Describe the worker node
kubectl describe node worker01

# Check pods on the worker node
kubectl get pods -A -o wide | grep worker01
```

### Test Workload Scheduling

```bash
# On master, create a deployment
kubectl create deployment nginx --image=nginx:alpine --replicas=3

# Check where pods are scheduled
kubectl get pods -o wide

# Scale deployment
kubectl scale deployment nginx --replicas=5

# Check node usage
kubectl get pods -o wide | grep worker01

# Clean up
kubectl delete deployment nginx
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. **Certificate Verification Failure**

```bash
# Issue: "error execution phase preflight: couldn't validate the identity..."
# Solution: Update the token
kubeadm token create --print-join-command
```

#### 2. **Kubelet Not Running**

```bash
# Check kubelet status
systemctl status kubelet

# View kubelet logs
journalctl -u kubelet -f

# Restart kubelet
systemctl restart kubelet

# Check if kubelet is enabled
systemctl is-enabled kubelet
```

#### 3. **Container Runtime Issues**

```bash
# Check Docker status
systemctl status docker

# Check containerd status
systemctl status containerd

# View Docker logs
journalctl -u docker -f

# Test Docker functionality
docker run hello-world
```

#### 4. **Node Not Ready**

```bash
# Check kubelet service
systemctl status kubelet

# Check for taints
kubectl describe node worker01 | grep Taints

# If tainted, remove taint (from master)
kubectl taint nodes worker01 <taint-name>-
```

#### 5. **Network Issues**

```bash
# Check network connectivity to master
ping -c 4 172.200.211.104

# Check DNS resolution
nslookup kubernetes.default.svc.cluster.local

# Check firewall rules
ufw status
```

### Reset Worker Node

```bash
# Reset the worker node
kubeadm reset -f

# Remove Kubernetes configuration
rm -rf /etc/kubernetes/
rm -rf /var/lib/kubelet/
rm -rf /var/lib/etcd/

# Restart services
systemctl restart docker
systemctl restart kubelet

# Rejoin the cluster
# Use kubeadm join command again
```

### Verification Script for Worker Node

```bash
# Create verification script
cat << 'EOF' > verify-worker.sh
#!/bin/bash

echo "======================================"
echo "Worker Node Verification Script"
echo "======================================"

echo -e "\n1. Checking Node Status..."
systemctl status kubelet --no-pager | grep Active

echo -e "\n2. Checking Docker Status..."
systemctl status docker --no-pager | grep Active

echo -e "\n3. Checking Disk Space..."
df -h /var/lib/docker

echo -e "\n4. Checking Network Connectivity..."
ping -c 2 172.200.211.104

echo -e "\n5. Checking Loaded Modules..."
lsmod | grep -E "overlay|br_netfilter"

echo -e "\n6. Checking Sysctl Parameters..."
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward

echo -e "\n======================================"
echo "Worker Node Verification Complete!"
echo "======================================"
EOF

# Make script executable and run
chmod +x verify-worker.sh
./verify-worker.sh
```

---

## Adding Multiple Worker Nodes

### Step 1: Prepare Additional Workers

```bash
# On worker2, worker3, etc.
# Repeat all setup steps (System Configuration, Container Runtime, Kubernetes Installation)

# Install Kubernetes components
apt install kubeadm kubelet -y

# Configure and start kubelet
systemctl enable kubelet
systemctl restart kubelet
```

### Step 2: Join All Workers

```bash
# On each worker node
kubeadm join 172.200.211.104:6443 \
    --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

### Step 3: Verify All Nodes

```bash
# From master node
kubectl get nodes

# Expected output:
# NAME       STATUS   ROLES           AGE   VERSION
# master1    Ready    control-plane   15m   v1.30.0
# worker01   Ready    <none>          5m    v1.30.0
# worker02   Ready    <none>          3m    v1.30.0
# worker03   Ready    <none>          1m    v1.30.0
```

---

## Best Practices

### Worker Node Security

1. **Firewall Configuration**:

```bash
# Allow required ports
ufw allow 6443/tcp
ufw allow 10250/tcp
ufw allow 30000-32767/tcp
ufw enable
```

2. **Regular Updates**:

```bash
# Update system regularly
apt update && apt upgrade -y
```

3. **Monitor Resource Usage**:

```bash
# Check disk space
df -h

# Check memory usage
free -m

# Check CPU usage
top -n1 | head -20
```

### Performance Optimization

1. **Docker Configuration**:

```bash
# Optimize Docker storage
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

systemctl restart docker
```

2. **Kubelet Optimization**:

```bash
# Configure kubelet resource limits
cat > /etc/systemd/system/kubelet.service.d/10-resource-limits.conf <<EOF
[Service]
CPUQuota=90%
MemoryLimit=90%
EOF

systemctl daemon-reload
systemctl restart kubelet
```

---

## Monitoring Worker Nodes

### Install Node Problem Detector

```bash
# On master node
kubectl apply -f https://raw.githubusercontent.com/kubernetes/node-problem-detector/master/node-problem-detector.yaml

# Verify installation
kubectl get pods -n kube-system | grep node-problem-detector
```

### Check Node Resources

```bash
# View node metrics
kubectl top node worker01

# View pod metrics on node
kubectl top pods --field-selector spec.nodeName=worker01
```

---

## Complete Worker Node Setup Script

```bash
# Create automated setup script
cat << 'EOF' > setup-worker.sh
#!/bin/bash

set -e

echo "Starting Kubernetes Worker Node Setup..."

# 1. System Configuration
echo "Configuring system..."
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# 2. Load Modules
echo "Loading kernel modules..."
cat > /etc/modules-load.d/k8s.conf <<MODULES
overlay
br_netfilter
MODULES

modprobe overlay
modprobe br_netfilter

# 3. Configure Sysctl
echo "Configuring sysctl parameters..."
cat > /etc/sysctl.d/10-k8s.conf <<SYSCTL
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
SYSCTL

sysctl --system

# 4. Install Docker
echo "Installing Docker..."
apt update -y
apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update -y
apt install docker-ce -y
systemctl enable docker --now

# 5. Configure Docker
echo "Configuring Docker..."
cat > /etc/docker/daemon.json <<DOCKER
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
DOCKER

systemctl restart docker

# 6. Install Kubernetes
echo "Installing Kubernetes..."
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
apt update -y
apt install kubeadm kubelet -y
apt-mark hold kubeadm kubelet

# 7. Configure Kubelet
echo "Configuring Kubelet..."
cat > /etc/default/kubelet <<KUBELET
KUBELET_EXTRA_ARGS="--cgroup-driver=cgroupfs"
KUBELET

systemctl daemon-reload
systemctl restart kubelet

echo "Worker Node setup complete!"
echo "Please run the join command from your master node."
EOF

# Make script executable
chmod +x setup-worker.sh

# Run the script
./setup-worker.sh
```

---

*This guide provides comprehensive steps for setting up Kubernetes worker nodes. Always verify connectivity with the master node before joining the cluster.*
