# Comprehensive Guide: Backup and Restore ETCD in Kubernetes Cluster

## Table of Contents
- [Understanding ETCD in Kubernetes](#understanding-etcd-in-kubernetes)
- [Imperative Method (Command Line)](#imperative-method-command-line)
- [Declarative Method (Automated)](#declarative-method-automated)
- [Scenarios & Use Cases](#scenarios--use-cases)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

---

## Understanding ETCD in Kubernetes

### What is ETCD?

ETCD is a distributed key-value store that stores all cluster data, including:
- Cluster state
- Configuration data
- Secrets
- Pod and service information

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│  │  Master 1   │  │  Master 2   │  │  Master 3   │       │
│  │   ETCD      │  │   ETCD      │  │   ETCD      │       │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘       │
│         │                │                │               │
│         └────────────────┼────────────────┘               │
│                          │                                │
│                     ┌────▼────┐                           │
│                     │  ETCD   │                           │
│                     │ Cluster │                           │
│                     └─────────┘                           │
└─────────────────────────────────────────────────────────────┘
```

---

## Imperative Method (Command Line)

### Step 1: Identify the ETCD Pod

```bash
# List all pods in kube-system namespace
kubectl get pods -n kube-system | grep etcd

# Output example:
# etcd-master01   1/1     Running   0          10d
# etcd-master02   1/1     Running   0          10d
# etcd-master03   1/1     Running   0          10d
```

### Step 2: Check ETCD Certificate Locations

```bash
# View the etcd manifest to find certificate paths
cat /etc/kubernetes/manifests/etcd.yaml | grep -E "file|data-dir"

# Output:
#     - --cert-file=/etc/kubernetes/pki/etcd/server.crt
#     - --key-file=/etc/kubernetes/pki/etcd/server.key
#     - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
#     - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
#     - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
#     - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
#     - --data-dir=/var/lib/etcd
```

### Step 3: Take ETCD Snapshot Backup

```bash
# Create backup directory if it doesn't exist
mkdir -p /root/backup

# Take snapshot using etcdctl
ETCDCTL_API=3 etcdctl \
  --endpoints=https://192.168.233.133:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /root/backup/etcdbkp.db

# Output:
# Snapshot saved at /root/backup/etcdbkp.db
```

#### Backup Command Breakdown

| Parameter | Description |
|-----------|-------------|
| `ETCDCTL_API=3` | Use ETCD API version 3 |
| `--endpoints` | ETCD endpoint URL and port |
| `--cacert` | CA certificate for authentication |
| `--cert` | Server certificate for authentication |
| `--key` | Server private key for authentication |
| `snapshot save` | Command to save snapshot |
| `/root/backup/etcdbkp.db` | Location to save the backup file |

### Step 4: Verify Backup Integrity

```bash
# Check snapshot status
ETCDCTL_API=3 etcdctl snapshot status --write-out=table /root/backup/etcdbkp.db

# Output example:
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | fb8fa466 | 1067874  |      1371  |    6.5 MB  |
# +----------+----------+------------+------------+
```

### Step 5: Restore ETCD from Backup

#### 5.1 Test by Deleting a Pod

```bash
# Delete a test pod to verify restore
kubectl delete pods nginx

# Output:
# pod "nginx" deleted

# Verify pod is gone
kubectl get pods
# No resources found in default namespace.
```

#### 5.2 Restore the Snapshot

```bash
# Restore the snapshot to a new data directory
ETCDCTL_API=3 etcdctl \
  --data-dir="/var/lib/etcd-backup" \
  --endpoints=https://192.168.233.133:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot restore /root/backup/etcdbkp.db

# Output:
# 2024-11-26 09:34:20.817355 I | mvcc: restore compact to 1066941
# 2024-11-26 09:34:20.828225 I | etcdserver/membership: added member 8e9e05c52164694d [http://localhost:2380] to cluster cdf818194e3a8c32
```

#### 5.3 Update ETCD Manifest

```bash
# Edit the etcd manifest
vi /etc/kubernetes/manifests/etcd.yaml
```

**Before Modification:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: etcd-master01
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://192.168.233.133:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd         # Original data directory
    - --experimental-initial-corrupt-check=true
    ...
    volumeMounts:
    - mountPath: /var/lib/etcd         # Original mount path
      name: etcd-data
    - mountPath: /etc/kubernetes/pki/etcd
      name: etcd-certs
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd              # Original host path
      type: DirectoryOrCreate
    name: etcd-data
```

**After Modification:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: etcd-master01
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://192.168.233.133:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcdbkp      # New data directory
    - --experimental-initial-corrupt-check=true
    ...
    volumeMounts:
    - mountPath: /var/lib/etcdbkp      # New mount path
      name: etcd-data
    - mountPath: /etc/kubernetes/pki/etcd
      name: etcd-certs
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcdbkp           # New host path
      type: DirectoryOrCreate
    name: etcd-data
```

**Important Note:** Update all three data-dir references:
1. `--data-dir` parameter in command
2. `mountPath` in volumeMounts
3. `path` in hostPath

#### 5.4 Verify Restoration

```bash
# Check pods to confirm nginx is restored
kubectl get pods

# Output:
# NAME    READY   STATUS    RESTARTS   AGE
# nginx   1/1     Running   0          2d1h

# Check ETCD container
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps

# Check ETCD member list
ETCDCTL_API=3 etcdctl \
  --endpoints=https://192.168.233.133:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

# Output:
# 4686573015862b34, started, master03, https://172.21.1.169:2380, https://172.21.1.169:2379
# b4fa34408c9d9fad, started, master01, https://172.21.1.167:2380, https://172.21.1.167:2379
# feb36f8f8de8fa53, started, master02, https://172.21.1.168:2380, https://172.21.1.168:2379
```

---

## Declarative Method (Automated)

### Architecture Overview

```
┌────────────────────────────────────────────────────────────────┐
│                    Automated Backup System                     │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────┐        ┌──────────────────┐            │
│  │   CronJob        │        │   PVC            │            │
│  │   (Scheduler)    │───────▶│   (Storage)      │            │
│  └──────────────────┘        └──────────────────┘            │
│         │                           │                         │
│         ▼                           │                         │
│  ┌──────────────────┐               │                         │
│  │   Backup Job     │───────────────┤                         │
│  │   (Execution)    │               │                         │
│  └──────────────────┘               │                         │
│                                     │                         │
│  ┌──────────────────┐               │                         │
│  │   Restore Job    │───────────────┘                         │
│  │   (Recovery)     │                                         │
│  └──────────────────┘                                         │
└────────────────────────────────────────────────────────────────┘
```

### Step 1: Create Storage for Backups

```yaml
# etcd-backup-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: etcd-backup-pvc
  namespace: kube-system
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

```bash
# Apply the PVC
kubectl apply -f etcd-backup-pvc.yaml
```

### Step 2: Create Secret for ETCD Certificates

```bash
# Create secret from ETCD certificates
kubectl create secret generic etcd-certs \
  -n kube-system \
  --from-file=ca.crt=/etc/kubernetes/pki/etcd/ca.crt \
  --from-file=server.crt=/etc/kubernetes/pki/etcd/server.crt \
  --from-file=server.key=/etc/kubernetes/pki/etcd/server.key
```

### Step 3: Create CronJob for Automated Backups

```yaml
# etcd-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 2 * * *"  # Run daily at 2 AM
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          containers:
          - name: etcd-backup
            image: bitnami/etcd:3.5.9
            command:
            - /bin/sh
            - -c
            - |
              export ETCDCTL_API=3
              TIMESTAMP=$(date +%Y%m%d_%H%M%S)
              etcdctl --endpoints=https://192.168.233.133:2379 \
                --cacert=/certs/ca.crt \
                --cert=/certs/server.crt \
                --key=/certs/server.key \
                snapshot save /backups/etcd-backup-${TIMESTAMP}.db
              # Keep only last 7 days of backups
              find /backups -name "etcd-backup-*.db" -mtime +7 -delete
            volumeMounts:
            - name: etcd-certs
              mountPath: /certs
              readOnly: true
            - name: etcd-backup
              mountPath: /backups
          restartPolicy: OnFailure
          volumes:
          - name: etcd-certs
            secret:
              secretName: etcd-certs
          - name: etcd-backup
            persistentVolumeClaim:
              claimName: etcd-backup-pvc
```

```bash
# Apply the CronJob
kubectl apply -f etcd-backup-cronjob.yaml
```

### Step 4: Verify CronJob

```bash
# Check CronJob status
kubectl get cronjob -n kube-system

# Output:
# NAME           SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# etcd-backup    0 2 * * *   False     0        2h ago          5d

# Check recent jobs
kubectl get jobs -n kube-system

# Check backup pods
kubectl get pods -n kube-system | grep etcd-backup

# Verify backup files exist
kubectl exec -it -n kube-system <backup-pod-name> -- ls -la /backups/
```

### Step 5: Create Restore Job

```yaml
# etcd-restore-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: etcd-restore
  namespace: kube-system
spec:
  template:
    spec:
      hostNetwork: true
      containers:
      - name: etcd-restore
        image: bitnami/etcd:3.5.9
        command:
        - /bin/sh
        - -c
        - |
          export ETCDCTL_API=3
          # Find the latest backup file
          BACKUP_FILE=$(ls -t /backups/etcd-backup-*.db | head -1)
          echo "Restoring from: $BACKUP_FILE"
          
          # Restore the snapshot
          etcdctl \
            --data-dir="/var/lib/etcd-restore" \
            --endpoints=https://192.168.233.133:2379 \
            --cacert=/certs/ca.crt \
            --cert=/certs/server.crt \
            --key=/certs/server.key \
            snapshot restore $BACKUP_FILE
          
          # Copy restored data to the correct location
          cp -r /var/lib/etcd-restore/* /var/lib/etcd-new/
          echo "Restore completed successfully"
        volumeMounts:
        - name: etcd-certs
          mountPath: /certs
          readOnly: true
        - name: etcd-backup
          mountPath: /backups
        - name: etcd-data
          mountPath: /var/lib/etcd-new
      restartPolicy: OnFailure
      volumes:
      - name: etcd-certs
        secret:
          secretName: etcd-certs
      - name: etcd-backup
        persistentVolumeClaim:
          claimName: etcd-backup-pvc
      - name: etcd-data
        persistentVolumeClaim:
          claimName: etcd-data-pvc
```

```bash
# Apply the restore job
kubectl apply -f etcd-restore-job.yaml
```

### Step 6: Monitor Restore Job

```bash
# Check job status
kubectl get jobs -n kube-system

# Check pod logs
kubectl logs -n kube-system <restore-pod-name>

# Check pod status
kubectl get pods -n kube-system
```

### Step 7: Advanced CronJob with Retention Policy

```yaml
# etcd-backup-advanced.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup-advanced
  namespace: kube-system
  labels:
    app: etcd-backup
    backup-type: etcd
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 3600
      template:
        spec:
          serviceAccountName: etcd-backup-sa
          hostNetwork: true
          containers:
          - name: etcd-backup
            image: bitnami/etcd:3.5.9
            command:
            - /bin/sh
            - -c
            - |
              set -e
              export ETCDCTL_API=3
              
              # Get endpoints from environment or config
              ENDPOINTS="https://192.168.233.133:2379"
              TIMESTAMP=$(date +%Y%m%d_%H%M%S)
              BACKUP_PATH="/backups/etcd-backup-${TIMESTAMP}.db"
              
              echo "Starting ETCD backup at $(date)"
              echo "Backup path: ${BACKUP_PATH}"
              
              # Take snapshot
              etcdctl --endpoints=${ENDPOINTS} \
                --cacert=/certs/ca.crt \
                --cert=/certs/server.crt \
                --key=/certs/server.key \
                snapshot save ${BACKUP_PATH}
              
              # Verify snapshot
              etcdctl snapshot status ${BACKUP_PATH} --write-out=table
              
              # Upload to remote storage (optional)
              # aws s3 cp ${BACKUP_PATH} s3://my-bucket/etcd-backups/
              
              # Clean up old backups - keep 7 days
              echo "Cleaning up old backups..."
              find /backups -name "etcd-backup-*.db" -mtime +7 -delete
              
              echo "Backup completed successfully at $(date)"
            env:
            - name: ETCDCTL_API
              value: "3"
            resources:
              requests:
                memory: "256Mi"
                cpu: "200m"
              limits:
                memory: "1Gi"
                cpu: "500m"
            volumeMounts:
            - name: etcd-certs
              mountPath: /certs
              readOnly: true
            - name: etcd-backup
              mountPath: /backups
          restartPolicy: OnFailure
          volumes:
          - name: etcd-certs
            secret:
              secretName: etcd-certs
          - name: etcd-backup
            persistentVolumeClaim:
              claimName: etcd-backup-pvc
```

---

## Scenarios & Use Cases

### 1. Prevent Data Loss

```bash
# Create a pre-upgrade backup
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-pre-upgrade.db \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify the backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-pre-upgrade.db
```

### 2. Disaster Recovery Script

```bash
#!/bin/bash
# disaster-recovery.sh

BACKUP_FILE="/backup/etcdbkp.db"
ETCD_DATA_DIR="/var/lib/etcd-restore"

# Check if backup exists
if [ ! -f "$BACKUP_FILE" ]; then
    echo "Backup file not found: $BACKUP_FILE"
    exit 1
fi

# Stop Kubernetes API server
kubectl delete pods -n kube-system -l component=kube-apiserver

# Restore etcd
ETCDCTL_API=3 etcdctl snapshot restore $BACKUP_FILE \
  --data-dir=$ETCD_DATA_DIR

# Update etcd manifest
sed -i 's|/var/lib/etcd|/var/lib/etcd-restore|g' /etc/kubernetes/manifests/etcd.yaml

# Wait for etcd to restart
sleep 30

# Restart API server
kubectl apply -f /etc/kubernetes/manifests/kube-apiserver.yaml

echo "Disaster recovery completed"
```

### 3. Cluster Migration

```bash
# On source cluster
ETCDCTL_API=3 etcdctl snapshot save /backup/migration.db \
  --endpoints=https://source-cluster:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Copy backup to destination cluster
scp /backup/migration.db user@destination-cluster:/backup/

# On destination cluster
ETCDCTL_API=3 etcdctl snapshot restore /backup/migration.db \
  --data-dir=/var/lib/etcd-migration
```

### 4. Rollback to Stable State

```bash
# Create backup before changes
ETCDCTL_API=3 etcdctl snapshot save /backup/pre-change.db \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# If changes cause issues, restore backup
ETCDCTL_API=3 etcdctl snapshot restore /backup/pre-change.db \
  --data-dir=/var/lib/etcd-rollback
```

### 5. Testing & Development Environment

```bash
# Create test environment backup
ETCDCTL_API=3 etcdctl snapshot save /backup/test-env.db \
  --endpoints=https://test-cluster:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Restore to development cluster
ETCDCTL_API=3 etcdctl snapshot restore /backup/test-env.db \
  --data-dir=/var/lib/etcd-dev
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. **Connection Refused**

```bash
# Issue: failed to connect to etcd endpoint
# Solution: Check if etcd is running
kubectl get pods -n kube-system | grep etcd

# Check etcd logs
kubectl logs -n kube-system etcd-master01

# Verify endpoints
kubectl get endpoints -n kube-system
```

#### 2. **Certificate Authentication Failed**

```bash
# Issue: authentication failed
# Solution: Verify certificate paths
ls -la /etc/kubernetes/pki/etcd/

# Check certificate validity
openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -text -noout

# Verify CA trust
openssl verify -CAfile /etc/kubernetes/pki/etcd/ca.crt /etc/kubernetes/pki/etcd/server.crt
```

#### 3. **Snapshot Restoration Fails**

```bash
# Issue: snapshot restore failed
# Solution: Check snapshot integrity
ETCDCTL_API=3 etcdctl snapshot status /backup/etcdbkp.db

# Try with different data directory
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcdbkp.db \
  --data-dir=/var/lib/etcd-new

# Check disk space
df -h /var/lib/
```

#### 4. **ETCD Member List Issues**

```bash
# Issue: member list not showing all members
# Solution: Check cluster health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://192.168.233.133:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Check member list
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://192.168.233.133:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Diagnostic Commands

```bash
# Create diagnostic script
cat << 'EOF' > etcd-diag.sh
#!/bin/bash

echo "ETCD Diagnostics"
echo "================="

echo -e "\n1. ETCD Pod Status:"
kubectl get pods -n kube-system | grep etcd

echo -e "\n2. ETCD Services:"
kubectl get svc -n kube-system | grep etcd

echo -e "\n3. ETCD Endpoints:"
kubectl get endpoints -n kube-system

echo -e "\n4. ETCD Logs (last 20 lines):"
kubectl logs -n kube-system etcd-master01 --tail=20

echo -e "\n5. ETCD Data Directory:"
ls -la /var/lib/etcd/ | head -10
du -sh /var/lib/etcd/

echo -e "\n6. Available Backups:"
ls -la /root/backup/*.db 2>/dev/null || echo "No backups found"
EOF

chmod +x etcd-diag.sh
./etcd-diag.sh
```

---

## Best Practices

### Backup Strategy Matrix

| Scenario | Frequency | Retention | Method |
|----------|-----------|-----------|---------|
| Production | Daily | 30 days | CronJob + Off-site |
| Pre-upgrade | On-demand | 7 days | Manual |
| Pre-change | On-demand | 7 days | Manual |
| Disaster Recovery | Continuous | 90 days | Off-site |

### Security Best Practices

1. **Encrypt Backups**
```bash
# Encrypt backup with GPG
gpg --symmetric --cipher-algo AES256 /backup/etcdbkp.db

# Decrypt backup
gpg --decrypt /backup/etcdbkp.db.gpg > /backup/etcdbkp.db
```

2. **Secure Backup Storage**
```yaml
# Store in secure location with proper permissions
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: etcd-backup-pvc
  annotations:
    security.kubernetes.io/backup: "encrypted"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: encrypted-storage
```

3. **Access Control**
```yaml
# Restrict who can perform backups
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: kube-system
  name: etcd-backup-manager
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "create", "delete", "list"]
```

### Monitoring & Alerting

```yaml
# Prometheus alert rule for backup failures
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: etcd-backup-alerts
  namespace: monitoring
spec:
  groups:
  - name: etcd-backup
    rules:
    - alert: ETCDBackupFailed
      expr: kube_job_status_failed{job="etcd-backup"} > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "ETCD backup job failed"
        description: "The etcd backup job has failed. Immediate action required."
```

### Automated Backup Verification

```yaml
# Add verification step to CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup-with-verify
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: etcd-backup
            image: bitnami/etcd:3.5.9
            command:
            - /bin/sh
            - -c
            - |
              set -e
              TIMESTAMP=$(date +%Y%m%d_%H%M%S)
              BACKUP_FILE="/backups/etcd-backup-${TIMESTAMP}.db"
              
              # Create backup
              etcdctl snapshot save ${BACKUP_FILE} \
                --cacert=/certs/ca.crt \
                --cert=/certs/server.crt \
                --key=/certs/server.key
              
              # Verify backup
              etcdctl snapshot status ${BACKUP_FILE} --write-out=table
              
              # Test restore to verify integrity
              TEST_DIR="/tmp/etcd-test-restore"
              etcdctl snapshot restore ${BACKUP_FILE} \
                --data-dir=${TEST_DIR}
              
              # Clean up test restore
              rm -rf ${TEST_DIR}
              
              echo "Backup verified and restored successfully"
```

### Monitoring Script

```bash
# Create monitoring script
cat << 'EOF' > monitor-etcd-backup.sh
#!/bin/bash

# Check latest backup
BACKUP_PATH="/root/backup"
LATEST_BACKUP=$(ls -t ${BACKUP_PATH}/etcdbkp*.db 2>/dev/null | head -1)

if [ -z "$LATEST_BACKUP" ]; then
    echo "ERROR: No ETCD backup found"
    exit 1
fi

# Check backup age
BACKUP_AGE=$(find $LATEST_BACKUP -mtime +1)
if [ ! -z "$BACKUP_AGE" ]; then
    echo "WARNING: Backup is older than 24 hours"
fi

# Verify backup integrity
ETCDCTL_API=3 etcdctl snapshot status $LATEST_BACKUP --write-out=table

# Check backup size
BACKUP_SIZE=$(du -m $LATEST_BACKUP | cut -f1)
if [ $BACKUP_SIZE -lt 1 ]; then
    echo "ERROR: Backup file size is too small: ${BACKUP_SIZE}MB"
    exit 1
fi

echo "Backup status: OK"
EOF

chmod +x monitor-etcd-backup.sh
```

---

## Quick Reference

### Important Commands

| Operation | Command |
|-----------|---------|
| **Backup** | `ETCDCTL_API=3 etcdctl snapshot save /backup/etcdbkp.db --endpoints=https://192.168.233.133:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key` |
| **Verify Backup** | `ETCDCTL_API=3 etcdctl snapshot status /backup/etcdbkp.db --write-out=table` |
| **Restore** | `ETCDCTL_API=3 etcdctl snapshot restore /backup/etcdbkp.db --data-dir=/var/lib/etcd-restore` |
| **Check ETCD Pod** | `kubectl get pods -n kube-system \| grep etcd` |
| **View ETCD Logs** | `kubectl logs -n kube-system etcd-master01 --tail=50` |
| **Check Member List** | `ETCDCTL_API=3 etcdctl member list --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key` |
| **Check Endpoint Health** | `ETCDCTL_API=3 etcdctl endpoint health --endpoints=https://192.168.233.133:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key` |
| **Apply CronJob** | `kubectl apply -f etcd-backup-cronjob.yaml` |
| **Check CronJob** | `kubectl get cronjob -n kube-system` |

---

*This guide provides comprehensive steps for backing up and restoring ETCD in Kubernetes clusters using both imperative and declarative methods. Regular backups are essential for cluster reliability and disaster recovery.*
