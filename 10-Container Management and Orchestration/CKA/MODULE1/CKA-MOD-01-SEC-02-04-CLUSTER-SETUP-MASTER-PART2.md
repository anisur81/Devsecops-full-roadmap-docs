# Comprehensive Guide: Kubernetes Cluster Setup with Calico Networking

## Table of Contents
- [Prerequisites and Network Configuration](#prerequisites-and-network-configuration)
- [Container Runtime Setup (Docker)](#container-runtime-setup-docker)
- [Kubernetes Installation](#kubernetes-installation)
- [System Configuration](#system-configuration)
- [Kubernetes Master Initialization](#kubernetes-master-initialization)
- [Calico Network Plugin Installation](#calico-network-plugin-installation)
- [Multi-Master Cluster Setup](#multi-master-cluster-setup)
- [Post-Installation Verification](#post-installation-verification)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites and Network Configuration

### Hosts File Configuration

```bash
# Edit hosts file to add all nodes
cat /etc/hosts
172.200.211.104 master1
172.200.211.105 worker1
172.200.211.106 worker2
```

### Set Hostname

```bash
# Set hostname for the node
hostnamectl set-hostname master1

# Verify hostname
hostnamectl
```

### Network Configuration (Netplan)

```bash
# Navigate to netplan directory
cd /etc/netplan/

# Edit network configuration
vi 50-cloud-init.yaml
```

**Network Configuration File:**

```yaml
# /etc/netplan/50-cloud-init.yaml
network:
    ethernets:
        ens192:
            dhcp4: false
            addresses: [172.200.211.104/24]
            gateway4: 172.200.211.1
            nameservers:
                addresses: [8.8.8.8, 12.168.100.10]
    version: 2
```

```bash
# Apply network configuration
netplan apply

# Verify network settings
ip addr show
ping -c 4 google.com
```

---

## Container Runtime Setup (Docker)

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

### Configure Docker (Optional)

```bash
# Configure Docker daemon for Kubernetes
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

# Install kubeadm, kubelet, kubectl
apt install kubeadm kubelet kubectl -y

# Hold packages to prevent automatic updates
apt-mark hold kubeadm kubelet kubectl
```

### Verify Installation

```bash
# Check versions
kubeadm version
kubelet --version
kubectl version --client
```

---

## System Configuration

### Disable Swap

```bash
# Temporarily disable swap
swapoff -a

# Permanently disable swap (comment out in fstab)
sed -i '/ swap / s/^/#/' /etc/fstab

# Verify swap is disabled
free -m
```

### Load Required Kernel Modules

```bash
# Create modules configuration file
vi /etc/modules-load.d/k8s.conf
```

**Add the following modules:**

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

---

## Kubernetes Master Initialization

### Pre-flight Checks

```bash
# Pull required container images
kubeadm config images pull

# Verify images are pulled
docker images | grep kube
```

### Initialize Kubernetes Cluster

```bash
# Initialize the cluster (Single Master)
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Initialize with specific advertise address
sudo kubeadm init --apiserver-advertise-address=192.168.172.131 --pod-network-cidr=10.244.0.0/16

# Initialize for multi-master setup
sudo kubeadm init \
    --control-plane-endpoint="172.21.1.173:6443" \
    --upload-certs \
    --apiserver-advertise-address=172.21.1.167 \
    --pod-network-cidr=192.168.0.0/16
```

### Configure kubectl

```bash
# Method 1: Configure for current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Method 2: Set environment variable
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> ~/.bashrc
source ~/.bashrc

# Method 3: Add kubectl auto-completion
echo "source <(kubectl completion bash)" >> ~/.bashrc
source ~/.bashrc

# Install bash completion if needed
apt-get install bash-completion -y
```

---

## Calico Network Plugin Installation

### Download Calico Manifests

```bash
# Create calico directory
mkdir calico && cd calico

# Download Tigera Operator manifest
wget https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/tigera-operator.yaml

# Download custom resources manifest
wget https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/custom-resources.yaml

# Alternative: Apply Calico directly
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.32.0/manifests/calico.yaml
```

### Configure Custom Resources

```bash
# Edit custom resources to match your pod network CIDR
cat custom-resources.yaml
```

**Sample custom-resources.yaml:**

```yaml
# custom-resources.yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - name: default-ipv4-ippool
      blockSize: 26
      cidr: 10.244.0.0/16  # Match with kubeadm init --pod-network-cidr
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
```

### Install Calico

```bash
# Apply Tigera Operator
kubectl create -f tigera-operator.yaml

# Wait for operator to be ready
kubectl get pods -n tigera-operator

# Apply custom resources
kubectl create -f custom-resources.yaml

# Verify Calico pods are running
kubectl get pods -A

# Check all pods in the system
kubectl get pods -n calico-system
kubectl get pods -n calico-apiserver
```

---

## Multi-Master Cluster Setup

### Join Additional Master Nodes

**On Additional Master Nodes (master2, master3):**

```bash
# Use the join command from the first master
kubeadm join 172.21.1.173:6443 \
    --token x1dmh5.w0nnknskd3d5s0vy \
    --discovery-token-ca-cert-hash sha256:2bbd80eb3549cccb6c2f944416cf092db291538e7c883f51dcc5ca530b011498 \
    --control-plane \
    --certificate-key 491f2ef94a31a12dcef302e9ebb38e4b2dd906e25cfce5148b017cb8213ec9cb
```

### Generate New Join Commands

```bash
# Generate new token with join command
kubeadm token create --print-join-command

# Generate new certificate key for additional masters
kubeadm init phase upload-certs --upload-certs
```

---

## Post-Installation Verification

### Check Cluster Status

```bash
# Check all nodes
kubectl get nodes -o wide

# Check all pods in all namespaces
kubectl get pods -A

# Check cluster health
kubectl get --raw='/readyz?verbose'

# Check API server health
curl -k https://localhost:6443/livez?verbose

# Get cluster information
kubectl cluster-info

# Get component statuses
kubectl get componentstatuses
```

### Verify Calico Network

```bash
# Check Calico system pods
kubectl get pods -n calico-system

# Check Calico API server
kubectl get pods -n calico-apiserver

# Check Tigera operator
kubectl get pods -n tigera-operator

# Check network policies
kubectl get networkpolicies -A
```

### Test Cluster Functionality

```bash
# Deploy a test pod
kubectl run nginx-test --image=nginx:alpine --restart=Never

# Check pod status
kubectl get pods

# Expose the pod as a service
kubectl expose pod nginx-test --port=80 --type=NodePort

# Get service details
kubectl get svc

# Clean up test resources
kubectl delete pod nginx-test
kubectl delete svc nginx-test
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. **Kubelet not starting**

```bash
# Check kubelet status
systemctl status kubelet

# View kubelet logs
journalctl -u kubelet -f

# Check kubelet configuration
cat /var/lib/kubelet/config.yaml

# Restart kubelet with debug
systemctl restart kubelet
```

#### 2. **Container runtime issues**

```bash
# Check containerd status
systemctl status containerd

# View containerd logs
journalctl -u containerd -f

# Clean containerd cache
rm -rf /etc/containerd/config.toml
systemctl restart containerd
```

#### 3. **Pod network issues**

```bash
# Check Calico pods
kubectl get pods -n calico-system

# Check Calico logs
kubectl logs -n calico-system -l k8s-app=calico-node

# Describe a pod for detailed info
kubectl describe pod <pod-name> -n calico-system
```

#### 4. **Token expiration**

```bash
# List existing tokens
kubeadm token list

# Generate new token
kubeadm token create --print-join-command

# Generate new certificate key
kubeadm init phase upload-certs --upload-certs
```

#### 5. **Node taints**

```bash
# Check node taints
kubectl describe nodes master1 | grep Taints

# Remove taint if needed
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Recovery Commands

```bash
# Reset Kubernetes cluster
sudo kubeadm reset -f

# Remove Kubernetes configuration
rm -rf $HOME/.kube

# Clean up etcd
rm -rf /var/lib/etcd

# Restart services
systemctl restart docker
systemctl restart kubelet
```

### Monitoring Commands

```bash
# Watch node status
watch kubectl get nodes

# Watch pod status
watch kubectl get pods -A

# Check resource usage
kubectl top nodes
kubectl top pods -A

# Check events
kubectl get events -A --sort-by='.lastTimestamp'
```

---

## Cluster Access for Remote Users

### Copy Admin Configuration

```bash
# On master node
cat /etc/kubernetes/admin.conf

# Copy the content and save on local machine as ~/.kube/config
# Modify the server IP to point to your master node or load balancer
```

### Create Service Account for API Access

```bash
# Create service account
kubectl create serviceaccount cluster-admin-sa -n kube-system

# Create cluster role binding
kubectl create clusterrolebinding cluster-admin-sa-binding \
    --clusterrole=cluster-admin \
    --serviceaccount=kube-system:cluster-admin-sa

# Get token
kubectl create token cluster-admin-sa -n kube-system
```

---

## Complete Verification Script

```bash
# Create verification script
cat << 'EOF' > verify-cluster.sh
#!/bin/bash

echo "======================================"
echo "Kubernetes Cluster Verification Script"
echo "======================================"

echo -e "\n1. Checking Node Status..."
kubectl get nodes -o wide

echo -e "\n2. Checking System Pods..."
kubectl get pods -n kube-system

echo -e "\n3. Checking Calico Network..."
kubectl get pods -n calico-system

echo -e "\n4. Checking Cluster Health..."
kubectl get --raw='/readyz?verbose' | head -10

echo -e "\n5. Testing Deployment..."
kubectl create deployment test-nginx --image=nginx:alpine --replicas=2
sleep 10
kubectl get pods -l app=test-nginx
kubectl delete deployment test-nginx

echo -e "\n6. Checking Component Status..."
kubectl get componentstatus

echo -e "\n======================================"
echo "Verification Complete!"
echo "======================================"
EOF

# Make script executable
chmod +x verify-cluster.sh

# Run verification
./verify-cluster.sh
```

---

## Best Practices

### Security Recommendations

1. **Use RBAC**: Implement Role-Based Access Control
2. **Enable TLS**: Ensure TLS is enabled for all components
3. **Regular Updates**: Keep Kubernetes and Docker updated
4. **Network Policies**: Implement network policies for pod isolation
5. **Audit Logging**: Enable audit logging for security monitoring

### Performance Optimization

1. **Resource Limits**: Set resource limits for pods
2. **Horizontal Pod Autoscaling**: Implement HPA for workloads
3. **Node Affinity**: Use node affinity for workload distribution
4. **Pod Disruption Budgets**: Implement PDB for critical workloads

### Backup and Disaster Recovery

```bash
# Backup etcd
sudo ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
    --cacert /etc/kubernetes/pki/etcd/ca.crt \
    --cert /etc/kubernetes/pki/etcd/server.crt \
    --key /etc/kubernetes/pki/etcd/server.key

# Backup certificates
sudo tar -czf /backup/k8s-certificates-$(date +%Y%m%d).tar.gz /etc/kubernetes/pki/
```

---

*This guide provides a comprehensive setup for a Kubernetes cluster with Calico networking. Always follow security best practices and test in a development environment first.*
