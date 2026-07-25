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

Install Falco

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts

helm install falco falcosecurity/falco
```

---

Attacker executes shell

```
kubectl exec
```

Falco alert

```
Terminal shell in container
```

---

Attacker reads password

```
cat /etc/shadow
```

Falco

```
Sensitive file opened
```

---

Attacker modifies binaries

```
touch /usr/bin/backdoor
```

Falco

```
Write below binary directory
```

---

## Reading real Falco output (the exam shows raw alert lines, not a summary label)

A real default-ruleset alert looks like this:

```
23:14:02.442731000: Warning Shell spawned in a container with an attached
terminal (user=root user_loginuid=-1 k8s.ns=security-lab k8s.pod=attacker
container=a1b2c3d4e5f6 shell=sh parent=runc cmdline=sh terminal=34816)
```

Key fields to extract for your incident report:

| Field | Meaning |
|---|---|
| `k8s.ns` / `k8s.pod` | which workload triggered it |
| `user` | UID inside the container |
| `parent` | process that spawned the shell (helps spot container breakout) |
| `cmdline` | the actual command run |

Follow Falco logs live during investigation:

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco -f
```

## Minimal custom rule (exam may ask you to write one, not just read alerts)

```yaml
- rule: Unexpected Privileged Pod Created
  desc: Detect creation of a privileged container in security-lab
  condition: >
    k8s_audit and ka.verb=create and ka.target.resource=pods
    and ka.req.pod.containers.privileged intersects (true)
    and ka.target.namespace=security-lab
  output: >
    Privileged pod created (user=%ka.user.name pod=%ka.target.name
    ns=%ka.target.namespace)
  priority: WARNING
  source: k8s_audit
```

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

# Lab 7 – Identify Malicious Actor

Audit log

```
User:

developer
```

IP

```
10.1.5.100
```

User Agent

```
kubectl/v1.31
```

Determine

```
Actor:

developer account

Source IP:

10.1.5.100

Tool:

kubectl
```

---

# Lab 8 – MITRE Kubernetes Mapping

| Activity             | MITRE Technique                |
| --------------------- | ------------------------------- |
| Create Pod            | Deploy Container                |
| Exec Into Pod         | Exec Into Container             |
| Read Secrets          | Steal Application Access Token |
| ServiceAccount Token  | Steal Credentials               |
| Privileged Pod        | Privilege Escalation            |
| Mount HostPath        | Escape to Host                  |
| Download Malware      | Ingress Tool Transfer           |
| Delete Audit Logs     | Indicator Removal on Host       |

Reference: the current MITRE ATT&CK for Containers matrix (`https://attack.mitre.org/matrices/enterprise/containers/`) and the Microsoft **Threat Matrix for Kubernetes** cover the same tactic categories the CKS exam draws from: Initial Access, Execution, Persistence, Privilege Escalation, Defense Evasion, Credential Access, Discovery, Lateral Movement, Collection, Impact.

---

# Lab 9 – Timeline Investigation

Example

```
10:01 Login
  |
10:02 Create Privileged Pod
  |
10:03 Exec into nginx
  |
10:04 Read Secret
  |
10:05 Install Malware
  |
10:07 Connect External Server
```

Questions

Who?

```
developer
```

When?

```
10:01
```

How?

```
kubectl
```

Goal?

```
Steal Secret
```

---

# Lab 10 – CKS Exam Scenario

Audit log

```
User:

john
```

Actions

```
Create pod

Exec pod

Read secret

Delete logs
```

Tasks

1. Find attacker username.

```bash
grep '"user"' audit.log
```

2. Find secret accessed.

```bash
grep '"secrets"' audit.log
```

3. Find executed commands.

```bash
grep 'pods/exec' audit.log
```

4. Find created pods.

```bash
grep '"create"' audit.log
```

5. Build timeline.

---

# Expected Findings

```
Attacker

john

Source

kubectl

Phase 1

Initial Access

Phase 2

Execution

Phase 3

Privilege Escalation

Phase 4

Credential Access

Phase 5

Discovery

Phase 6

Impact
```

---

# Lab 11 – Containment and Remediation (the CKS exam grades this, not just detection)

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
* Preserve evidence (copy logs out) before deleting any compromised resource.

---

This lab closely mirrors real CKS scenarios by requiring you to **investigate logs, identify the attacker, determine the attack phase, correlate evidence from Kubernetes audit logs and Falco runtime alerts, and then contain and remediate the incident**.
