# CIS Benchmark & Security Assessment Tools

## Overview

### What is CIS?
The Center for Internet Security (CIS) is a non-profit organization focused on enhancing cybersecurity readiness and response. They develop security best practices and benchmarks for various software and systems, including Kubernetes.

### CIS Offerings
- **Development of Security Best Practices**
- **Security Benchmarks**
- **Tools and Resources**
  - **Free Tools:** CIS-CAT Lite
  - **Paid Tools:** CIS-CAT Pro

---

## CIS-CAT Tool

### CIS-CAT Lite
- Developed by CIS Organization
- Designed to help organizations assess and audit security configurations
- Built using **Java** (requires JRE to run)
- Free version available

### CIS-CAT Pro
- Includes all features of CIS-CAT Lite
- Additional advanced features

---

## CIS Benchmarks for Kubernetes

### Key Areas Covered
#### Control Plane Node Configuration
- API Server Configuration
  - Disable Anonymous Authentication
  - Ensure RBAC is enabled
  - Include Node Authorization Mode
- Controller Manager Configuration
- Scheduler Configuration
- Etcd Configuration

#### Worker Node Configuration
- Kubelet Configuration
- Configuration Files

---

## Installing & Running CIS-CAT Lite

### Download Options
1. **Direct Download:**
   ```
   http://temp.sh/BqGcr/CIS-CAT_Lite_Assessor_v4.55.0.zip
   ```

2. **Official Download:**
   - Visit: https://learn.cisecurity.org/cis-cat-lite
   - Fill out the form to download CIS-CAT Lite demo

### Installation Steps

```bash
# Install Java Dependencies
apt install openjdk-8* -y

# Navigate to Downloads
cd /home/koenig/Downloads

# Extract the ZIP file
unzip CIS-CAT-Lite-v4.53.0.zip

# Enter the Assessor directory
cd Assessor

# View help menu
./Assessor-CLI.sh --help
```

### Running a Benchmark Test

```bash
# Basic command structure
./Assessor-CLI.sh -i -rd /home/koenig -nts -rp sample

# Full example (most used)
sudo ./CIS-CAT.sh -a \
  -b benchmarks/CIS_Ubuntu_22.04_L1.xml \
  -o html,json \
  -rd /opt/cis-reports/
```

**Note:** When prompted, select:
- Option 7 for Ubuntu OS
- Option 4 for Server

### Viewing Report
Open the generated report in browser:
```
/home/koenig/sample.html
```

---

## Kube-Bench: Kubernetes CIS Benchmarking Tool

### What is Kube-bench?
- Open-source tool developed by **Aqua Security**
- Written in **GoLang**
- Assesses Kubernetes cluster security against CIS Kubernetes Benchmark
- Automates cluster hardening checks

### Capabilities
| Feature | Description |
|---------|-------------|
| **Cluster Hardening** | Automates configuration checks per CIS guidelines |
| **Policy Enforcement** | Checks RBAC, service accounts, pod security standards, secret management |
| **Network Segmentation** | Verifies CNI and network policy support |

---

## Installing Kube-Bench

### Method 1: From Command Line (Control Plane Access)

```bash
# Create directory
sudo mkdir -p /opt/kube-bench

# Download the binary (check for latest version)
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.13.0/kube-bench_0.13.0_linux_amd64.tar.gz -o /opt/kube-bench.tar.gz

# Extract
tar -xvf kube-bench.tar.gz -C /opt/kube-bench

# Move executable to PATH
sudo mv /opt/kube-bench/kube-bench /usr/local/bin/
```

### Directory Structure
```
/opt/kube-bench/
├── cfg/
│   ├── ack-1.0/
│   ├── aks-1.0/
│   ├── cis-1.6-k3s/
│   ├── eks-stig-kubernetes-v1r6/
│   ├── gke-1.2.0/
│   ├── rh-1.0/
│   └── config.yaml
└── kube-bench
```

### Running Kube-Bench

```bash
# Run with config directory specified
sudo kube-bench --config-dir /opt/kube-bench/cfg --config /opt/kube-bench/cfg/config.yaml

# Save output to file
sudo kube-bench --config-dir /opt/kube-bench/cfg --config /opt/kube-bench/cfg/config.yaml > kube-bench.report
```

### Sample Output
```bash
[INFO] 1 Control Plane Security Configuration
[INFO] 1.1 Control Plane Node Configuration Files
[PASS] 1.1.1 Ensure API server pod spec file permissions are 644 or more restrictive
[PASS] 1.1.2 Ensure API server pod spec file ownership is root:root

== Remediations master ==
1.1.9 Run chmod 600 <path/to/cni/files>
1.1.12 On etcd server, get data directory from --data-dir

== Summary master ==
41 checks PASS
9 checks FAIL
11 checks WARN
0 checks INFO
```

---

## Method 2: Package Installation

### Debian/Ubuntu
```bash
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.13.0/kube-bench_0.13.0_linux_amd64.deb
sudo dpkg -i kube-bench.deb

# Run after installation
sudo kube-bench
```

**Note:** Configuration files located at `/etc/kube-bench/cfg/`

---

## Method 3: Running as Kubernetes Pod

### Deploy as Job
```bash
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Or download and modify
curl https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml > job.yaml
kubectl apply -f job.yaml
```

### View Results
```bash
# List pods
kubectl get pods

# View logs
kubectl logs kube-bench-4j2bs

# Export to file
kubectl logs kube-bench-4j2bs > kube-bench.report
```

---

## Important Notes

### Support Coverage
- **Kube-bench** implements CIS Kubernetes Benchmark as closely as possible
- Not a one-to-one mapping between Kubernetes releases and CIS benchmark releases
- Default behavior: determines test set based on Kubernetes version
- [Report issues](https://github.com/aquasecurity/kube-bench/issues) if tests not correctly implemented
- Use [CIS Community](https://www.cisecurity.org/) to report benchmark issues

### Documentation
- See: [Running kube-bench documentation](https://github.com/aquasecurity/kube-bench#running-kube-bench)

---

## Quick Reference

| Tool | Type | Language | Purpose |
|------|------|----------|---------|
| **CIS-CAT Lite** | Free | Java | General security configuration auditing |
| **CIS-CAT Pro** | Paid | Java | Advanced security auditing |
| **Kube-bench** | Open-source | GoLang | Kubernetes-specific CIS benchmarking |

### CIS Benchmark Coverage for Kubernetes
- ✅ Control Plane Node Configuration
- ✅ API Server Configuration
- ✅ Controller Manager Configuration
- ✅ Scheduler Configuration
- ✅ Etcd Configuration
- ✅ Worker Node Configuration
- ✅ Kubelet Configuration
- ✅ RBAC and Policies
