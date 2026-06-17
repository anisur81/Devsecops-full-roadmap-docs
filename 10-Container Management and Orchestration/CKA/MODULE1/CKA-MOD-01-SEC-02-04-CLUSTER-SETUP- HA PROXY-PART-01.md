# Kubernetes High Availability (HA) Cluster Setup with HAProxy Load Balancer

## Table of Contents
- [Prerequisites](#prerequisites)
- [Network Requirements](#network-requirements)
- [HAProxy Load Balancer Configuration](#haproxy-load-balancer-configuration)
- [Kubernetes Node Preparation](#kubernetes-node-preparation)
- [Kubernetes Master Node Initialization](#kubernetes-master-node-initialization)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

### System Requirements
- **Operating System**: Ubuntu (18.04/20.04/22.04)
- **Minimum Resources per Node**:
  - 2+ CPUs
  - 2+ GB RAM
  - 20+ GB Storage
- **Network**: All nodes must be able to communicate with each other

### Node Architecture
```
+------------------+
|   HAProxy LB     |
|  192.168.172.146 |
+--------+---------+
         |
         | 6443
         |
    +----+----+----+
    |    |    |    |
  +----+ +----+ +----+
  |M1  | |M2  | |M3  |
  |.131| |.132| |.133|
  +----+ +----+ +----+
```

---

## Network Requirements

### Verify Node Connectivity

```bash
# Test connectivity from each node to all other nodes
ping -c 4 192.168.172.131
ping -c 4 192.168.172.132
ping -c 4 192.168.172.133

# Test connectivity from HAProxy to master nodes
ping -c 4 192.168.172.131
ping -c 4 192.168.172.132
ping -c 4 192.168.172.133
```

### Required Ports

| Protocol | Port | Source | Destination | Purpose |
|----------|------|--------|-------------|---------|
| TCP | 6443 | HAProxy | Master Nodes | Kubernetes API Server |
| TCP | 2379-2380 | All | Master Nodes | etcd Server |
| TCP | 10250 | All | All | Kubelet API |
| TCP | 10251 | All | Master Nodes | kube-scheduler |
| TCP | 10252 | All | Master Nodes | kube-controller-manager |
| TCP | 10255 | All | All | Read-only Kubelet API |
| TCP | 30000-32767 | All | All | NodePort Services |

---

## HAProxy Load Balancer Configuration

### Step 1: Install HAProxy

```bash
# Update package list and install HAProxy
sudo apt update && sudo apt install -y haproxy

# Verify installation
haproxy -v
```

### Step 2: Configure HAProxy

Edit the HAProxy configuration file:

```bash
sudo vi /etc/haproxy/haproxy.cfg
```

Append the following configuration (replace IP addresses with your actual IPs):

```haproxy
# /etc/haproxy/haproxy.cfg

frontend kubernetes
    bind 192.168.172.146:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes

backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server master01 192.168.172.131:6443 check fall 3 rise 2
    server master02 192.168.172.132:6443 check fall 3 rise 2
    server master03 192.168.172.133:6443 check fall 3 rise 2
```

**Configuration Explanation:**

| Parameter | Description |
|-----------|-------------|
| `frontend kubernetes` | Defines the entry point for traffic |
| `bind 192.168.172.146:6443` | HAProxy listens on this IP:Port |
| `mode tcp` | TCP load balancing mode |
| `balance roundrobin` | Distributes traffic evenly |
| `option tcp-check` | Performs health checks via TCP |
| `server masterXX` | Defines backend master nodes |
| `check fall 3 rise 2` | Health check parameters |

### Step 3: Start and Enable HAProxy

```bash
# Start HAProxy service
sudo systemctl restart haproxy

# Enable HAProxy to start on boot
sudo systemctl enable haproxy

# Check HAProxy status
sudo systemctl status haproxy

# Verify HAProxy is listening on the port
sudo netstat -tulpn | grep 6443
```

### Step 4: Test HAProxy

```bash
# Test connectivity through HAProxy
curl -k https://192.168.172.146:6443

# Expected output shows Kubernetes API version info
```

---

## Kubernetes Node Preparation

### Step 1: Disable Swap

```bash
# Temporarily disable swap
sudo swapoff -a

# Permanently disable swap (comment out swap entry in /etc/fstab)
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Verify swap is disabled
free -m
```

### Step 2: Configure System Settings

Create Kubernetes sysctl configuration:

```bash
sudo vi /etc/sysctl.d/kubernetes.conf
```

Add the following content:

```conf
# /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

Apply the settings:

```bash
# Apply sysctl settings
sudo sysctl --system

# Verify settings
sysctl net.bridge.bridge-nf-call-iptables
```

### Step 3: Install Container Runtime (Docker)

```bash
# Install prerequisites
sudo apt-get update
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# Add Docker repository
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

# Install Docker
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Configure Docker to use systemd as cgroup driver
cat <<EOF | sudo tee /etc/docker/daemon.json
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
sudo systemctl restart docker
sudo systemctl enable docker
```

### Step 4: Install Kubernetes Components

```bash
# Add Kubernetes repository
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Update package list
sudo apt-get update

# Install Kubernetes components
sudo apt-get install -y kubelet kubeadm kubectl

# Hold the versions to prevent automatic updates
sudo apt-mark hold kubelet kubeadm kubectl

# Enable kubelet service
sudo systemctl enable kubelet
sudo systemctl start kubelet
```

---

## Kubernetes Master Node Initialization

### Prerequisites Check

```bash
# Verify all prerequisites
kubeadm version
kubectl version --client
kubelet --version
docker --version
```

### Initialize First Master Node

**On Master01 (192.168.172.131):**

```bash
# Initialize the cluster with HAProxy endpoint
sudo kubeadm init \
    --control-plane-endpoint="192.168.172.146:6443" \
    --upload-certs \
    --apiserver-advertise-address=192.168.172.131 \
    --pod-network-cidr=192.168.0.0/16 \
    --ignore-preflight-errors all
```

**Parameter Explanation:**

| Parameter | Description |
|-----------|-------------|
| `--control-plane-endpoint` | HAProxy load balancer address |
| `--upload-certs` | Upload certificates for automatic distribution |
| `--apiserver-advertise-address` | IP address of the specific master node |
| `--pod-network-cidr` | CIDR range for pod network |
| `--ignore-preflight-errors` | Ignore specific preflight errors |

### Configure kubectl for First Master

```bash
# Configure kubectl for the admin user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify the cluster is working
kubectl get nodes
kubectl cluster-info
```

### Join Additional Master Nodes

**On Master02 (192.168.172.132):**

```bash
# Use the join command from the first master's output
sudo kubeadm join 192.168.172.146:6443 \
    --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane \
    --certificate-key <certificate-key> \
    --apiserver-advertise-address=192.168.172.132
```

**On Master03 (192.168.172.133):**

```bash
# Join with similar command for third master
sudo kubeadm join 192.168.172.146:6443 \
    --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane \
    --certificate-key <certificate-key> \
    --apiserver-advertise-address=192.168.172.133
```

### Verify Cluster Status

```bash
# Check all nodes in the cluster
kubectl get nodes

# Expected output should show all master nodes Ready
# NAME      STATUS   ROLES    AGE   VERSION
# master01  Ready    master   10m   v1.28.0
# master02  Ready    master   5m    v1.28.0
# master03  Ready    master   2m    v1.28.0

# Check control plane pods
kubectl get pods -n kube-system

# Verify etcd cluster health
kubectl get pods -n kube-system | grep etcd
```

---

## Install Network Plugin (Calico)

```bash
# Install Calico network plugin
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml

# Download and customize Calico manifest
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml -O

# Apply the Calico manifest
kubectl apply -f custom-resources.yaml

# Verify Calico pods are running
kubectl get pods -n calico-system
```

---

## Worker Node Setup (Optional)

### Join Worker Nodes

```bash
# On each worker node, use the join command from kubeadm init
sudo kubeadm join 192.168.172.146:6443 \
    --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

### Verify All Nodes

```bash
# View all nodes in the cluster
kubectl get nodes -o wide

# View all pods across all namespaces
kubectl get pods -A

# Check cluster health
kubectl cluster-info
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. **Swap is enabled**
```bash
# Solution
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

#### 2. **Port conflicts**
```bash
# Check port usage
sudo netstat -tulpn | grep 6443
sudo netstat -tulpn | grep 10250
```

#### 3. **Certificate issues**
```bash
# Regenerate certificates
sudo kubeadm init phase upload-certs --upload-certs

# Check certificates
sudo kubeadm certs check-expiration
```

#### 4. **HAProxy not responding**
```bash
# Check HAProxy status
sudo systemctl status haproxy

# Check HAProxy logs
sudo tail -f /var/log/haproxy.log

# Test HAProxy configuration
haproxy -c -f /etc/haproxy/haproxy.cfg
```

#### 5. **Node not Ready**
```bash
# Check kubelet status
sudo systemctl status kubelet

# Check kubelet logs
sudo journalctl -u kubelet -f

# Reset kubeadm if necessary
sudo kubeadm reset -f
```

### Verification Commands

```bash
# Check all components
kubectl get componentstatus

# Check API server connectivity through HAProxy
curl -k https://192.168.172.146:6443/version

# Check etcd health
kubectl get endpoints -n kube-system

# Test pod creation
kubectl run test-pod --image=nginx:latest --restart=Never
kubectl get pods

# Clean up test pod
kubectl delete pod test-pod
```

---

## Complete Cluster Verification Checklist

```bash
# Create verification script
cat << 'EOF' > verify-cluster.sh
#!/bin/bash

echo "==> Checking HAProxy Status..."
systemctl status haproxy --no-pager

echo -e "\n==> Checking Nodes..."
kubectl get nodes

echo -e "\n==> Checking Control Plane Components..."
kubectl get pods -n kube-system

echo -e "\n==> Checking API Server via HAProxy..."
curl -k https://192.168.172.146:6443/version 2>/dev/null | grep -o '"major":"[^"]*"' | head -1

echo -e "\n==> Checking etcd Cluster Health..."
kubectl get endpoints -n kube-system

echo -e "\n==> Testing Pod Creation..."
kubectl run test-pod --image=nginx:latest --restart=Never
sleep 10
kubectl get pod test-pod
kubectl delete pod test-pod --ignore-not-found

echo -e "\n==> All Checks Complete!"
EOF

# Make script executable and run
chmod +x verify-cluster.sh
./verify-cluster.sh
```

---

## Backup and Recovery

### Backup Certificates

```bash
# Backup PKI directory
sudo tar -czvf k8s-pki-backup-$(date +%Y%m%d).tar.gz /etc/kubernetes/pki/

# Backup etcd
sudo ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
    --cacert /etc/kubernetes/pki/etcd/ca.crt \
    --cert /etc/kubernetes/pki/etcd/server.crt \
    --key /etc/kubernetes/pki/etcd/server.key
```

### Recovery Steps

```bash
# Restore certificates
sudo tar -xzvf k8s-pki-backup-YYYYMMDD.tar.gz -C /

# Restore etcd snapshot
sudo ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
    --data-dir /var/lib/etcd-backup
```

 
---

*This guide provides a complete setup for a Kubernetes High Availability cluster with HAProxy load balancer. Always test in a development environment before production deployment.*
