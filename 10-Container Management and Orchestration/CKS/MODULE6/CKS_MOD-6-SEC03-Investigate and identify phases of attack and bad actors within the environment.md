# Investigate and identify phases of attack and bad actors within the environment

## Objective

Simulate an attacker who:

* Gains access
* Creates a privileged pod
* Reads Secrets
* Executes commands inside a container
* Tries to escape the container

You will investigate the attack, then **contain and remediate** it — the part most CKS practice labs skip but the real exam expects.

---

# Lab Architecture

```
+------------------------+
| Kubernetes Cluster     |
|                        |
|  nginx Pod             |
|  secret                |
|                        |
| Audit Logs             |
| Falco                  |
+-----------+------------+
            |
            |
     Security Analyst
            |
     kubectl logs
     audit logs
     Falco alerts
```

---

# Lab 0 – Enable Audit Logging (prerequisite, often assumed but not shown)

Nothing in Lab 4 works unless the API server is actually configured to write audit logs. On the exam you may be asked to set this up from scratch.

Create an audit policy file on the control-plane node:

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods", "pods/exec"]
  - level: Metadata
    omitStages:
      - "RequestReceived"
```

Add the following flags to the kube-apiserver static pod manifest
(`/etc/kubernetes/manifests/kube-apiserver.yaml`):

```yaml
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --audit-log-path=/var/log/kubernetes/audit/audit.log
- --audit-log-maxage=7
- --audit-log-maxbackup=3
- --audit-log-maxsize=100
```

Mount `/etc/kubernetes/audit-policy.yaml` and `/var/log/kubernetes/audit` as `hostPath` volumes in the same manifest. The kubelet will restart the static pod automatically — watch for it:

```bash
watch crictl ps | grep kube-apiserver
```

---

# Lab 1 – Environment Setup

Create namespace

```bash
kubectl create ns security-lab
```

---

Create secret

```bash
kubectl create secret generic db-secret \
--from-literal=password=SuperSecret \
-n security-lab
```

Verify

```bash
kubectl get secret -n security-lab
```

---

Deploy nginx

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: security-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```

Apply

```bash
kubectl apply -f nginx.yaml
```

---

Verify

```bash
kubectl get pods -n security-lab
```

Expected

```
nginx-xxxxx Running
```

---

# Lab 2 – Simulate Attacker

Imagine attacker compromised developer credentials.

Create privileged pod

```bash
kubectl run attacker \
--image=busybox \
--privileged \
--restart=Never \
-it -- sh
```

Now attacker has shell.

---

List secrets

```bash
kubectl get secrets -n security-lab
```

Expected

```
db-secret
```

---

Read secret

```bash
kubectl get secret db-secret \
-o yaml \
-n security-lab
```

Decode

```bash
echo "U3VwZXJTZWNyZXQ=" | base64 -d
```

Output

```
SuperSecret
```

---

# Lab 3 – Execute Into Existing Pod

Attacker runs

```bash
kubectl exec -it deployment/nginx \
-n security-lab -- sh
```

Inside pod

```
cat /etc/passwd

ls /

hostname

env
```

---

# Lab 3b – Attempted Container Escape (referenced by the objective, not shown before)

Because the pod was created with `--privileged`, the attacker can mount the host filesystem from inside the container:

```bash
mkdir /mnt/hostfs
mount /dev/sda1 /mnt/hostfs
chroot /mnt/hostfs
```

This is the step that turns "privilege escalation inside the cluster" into "host compromise." On the exam, recognizing that `--privileged` + hostPath/host device access is what enables escape (not the `kubectl exec` itself) is the key concept — memorize the causal chain, not just the command.

Falco flags this distinctly from a normal exec — see Lab 5.

---

# Lab 4 – Investigate Using Audit Logs

Example audit log

```json
{
"user":"developer",
"verb":"create",
"objectRef":{
"resource":"pods",
"name":"attacker"
}
}
```

Find pod creation

```bash
grep '"create"' audit.log
```

Output

```
developer created attacker pod
```

---

Search exec operations

```bash
grep '"pods/exec"' audit.log
```

Example

```
developer executed command inside nginx
```

---

Search secret access

```bash
grep '"secrets"' audit.log
```

Output

```
developer requested db-secret
```

---

## Better audit-log queries with `jq` (the exam gives you real JSON lines, not pre-filtered text)

```bash
# All requests by a specific user
jq -r 'select(.user.username=="developer")' audit.log

# All pod creations, with timestamp and source IP
jq -r 'select(.verb=="create" and .objectRef.resource=="pods") |
  "\(.requestReceivedTimestamp) \(.user.username) \(.sourceIPs[0]) \(.objectRef.name)"' audit.log

# All exec calls
jq -r 'select(.objectRef.subresource=="exec") |
  "\(.requestReceivedTimestamp) \(.user.username) -> \(.objectRef.namespace)/\(.objectRef.name)"' audit.log

# All secret reads
jq -r 'select(.objectRef.resource=="secrets" and .verb=="get") |
  "\(.requestReceivedTimestamp) \(.user.username) read \(.objectRef.name)"' audit.log
```

`jq` is available in the exam environment and is far faster than `grep` for correlating fields across a timeline.

---

Questions

Who?

```
developer
```

What?

```
Secret Access
```

Attack Phase?

```
Credential Access
```

---

# Lab 4b – Check Whether the Actor's Permissions Were Legitimate (RBAC review)

Once you know *who*, confirm *what they were authorized to do*. This is a standard CKS task type on its own.

```bash
kubectl auth can-i get secrets \
  --as=developer -n security-lab

kubectl auth can-i create pods \
  --as=developer -n security-lab

# Find the RoleBinding/ClusterRoleBinding granting access
kubectl get rolebindings,clusterrolebindings -A \
  -o json | jq -r '.items[] | select(.subjects[]?.name=="developer") | .metadata.name'

kubectl describe role <role-name> -n security-lab
```

If `developer` was never supposed to have `create` on `pods` or `get` on `secrets`, that's your evidence of privilege misconfiguration compounding the compromise — call this out explicitly in your findings.

---

# Lab 5 – Investigate Using Falco

## For a detailed, step-by-step guide, please follow the complete documentation here:
[Behavioral Analytics of Syscalls, Process & File Activities using Falco](https://github.com/anisur81/Devsecops-full-roadmap-docs/blob/main/10-Container%20Management%20and%20Orchestration/CKS/MODULE6/CKS_MOD-6-SEC01-Perform%20behavioral%20analytics%20to%20detect%20malicious%20activities.md)
 
---

# Lab 6 – Identify Attack Phase

Example activities

| Activity                      | Attack Phase         |
| ------------------------------ | --------------------- |
| Login using stolen kubeconfig | Initial Access       |
| kubectl exec                  | Execution             |
| Create privileged Pod         | Privilege Escalation |
| Read Secret                   | Credential Access    |
| Read ServiceAccount Token     | Credential Access    |
| Connect to API Server         | Discovery             |
| List Pods                     | Discovery             |
| Chroot into mounted hostfs    | Privilege Escalation / Escape to Host |
| Download malware              | Command and Control  |
| Crypto Miner                  | Impact                |
| Delete audit logs             | Defense Evasion       |

---
 

# Lab 7 – Containment and Remediation (the CKS exam grades this, not just detection)

Investigation without containment is only half the task on the real exam. Once the actor and pod are identified:

Isolate the compromised pod with a deny-all NetworkPolicy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: quarantine-attacker
  namespace: security-lab
spec:
  podSelector:
    matchLabels:
      run: attacker
  policyTypes:
    - Ingress
    - Egress
```

Kill the pod and rotate the exposed secret:

```bash
kubectl delete pod attacker -n security-lab --force --grace-period=0

kubectl delete secret db-secret -n security-lab
kubectl create secret generic db-secret \
  --from-literal=password=NewRotatedSecret \
  -n security-lab
```

Revoke or scope down the compromised identity:

```bash
kubectl delete rolebinding <binding-name-for-developer> -n security-lab
# or replace it with a Role that removes 'create pods' and 'get secrets'
```

Preserve evidence before cleanup — copy relevant audit log lines and Falco alerts to a separate file/location prior to deleting any resources, since deleting the pod does not delete the audit trail but does delete `kubectl logs` history.

---

# CKS Exam Tips

* Inspect Kubernetes **audit logs** for `create`, `get`, `list`, `watch`, and `pods/exec` events; use `jq` rather than `grep` when fields need to be correlated.
* Confirm the audit policy and `kube-apiserver` audit flags are actually in place before assuming logs exist — the exam sometimes requires you to configure this first.
* Use **Falco** alerts to identify runtime anomalies such as shell execution, sensitive file access, unexpected network connections, or privileged container creation — know how to read a raw alert line, not just recognize the alert name.
* Cross-check the actor's **RBAC permissions** (`kubectl auth can-i`, RoleBindings) against what they actually did — misconfigured RBAC is often part of the finding.
* Build a **timeline** by correlating timestamps from audit logs, Falco alerts, and container logs.
* Map attacker actions to the **MITRE ATT&CK for Containers** matrix (Initial Access → Execution → Persistence → Privilege Escalation → Defense Evasion → Credential Access → Discovery → Lateral Movement → Impact).
* Identify the **actor** by correlating the Kubernetes user, source IP, user agent, and affected resources.
* Don't stop at identification — the exam expects **containment**: NetworkPolicy isolation, secret rotation, pod deletion, and RBAC revocation.
---
 
