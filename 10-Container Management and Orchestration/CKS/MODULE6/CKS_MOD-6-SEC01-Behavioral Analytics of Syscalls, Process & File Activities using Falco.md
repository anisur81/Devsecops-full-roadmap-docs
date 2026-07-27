# Behavioral Analytics of Syscalls, Process & File Activities using Falco

This lab covers the CKS objective:

> **Perform behavioral analytics of syscall process and file activities at the host and container level to detect malicious activities.**

The primary tool for CKS is **Falco**, an open-source CNCF runtime security engine that monitors Linux syscalls using eBPF (or a kernel module) to detect suspicious behavior.

---

# Lab Objectives

After completing this lab, you will be able to:

* Install Falco
* Understand Falco architecture
* Detect malicious process execution
* Detect shell access inside containers
* Detect sensitive file modifications
* Detect privilege escalation
* Detect unexpected network connections
* Write custom Falco rules
* View Falco alerts

---

# Lab Environment

Minimum requirements:

* Ubuntu 22.04
* Kubernetes v1.30+
* kubectl
* Helm
* Kind or Minikube

```
Ubuntu
   |
Kubernetes Cluster
   |
Falco DaemonSet
   |
Every Node
   |
Kernel Syscalls
```

---

# Step 0: Install Helm

Install Helm (Recommended)

Run:

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

Verify:

helm version


# Step 1: Install Falco

Add the Helm repository:

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

Install:

```bash
helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set tty=true
```
If  cluster has OPA Gatekeeper enforcing a policy (must-have-owner) that requires every pod to have an owner label — and the Falco chart's pod template doesn't set one by default, so Gatekeeper is rejecting pod creation outright

```
helm upgrade falco falcosecurity/falco \
  --namespace falco \
  --set tty=true \
  --set podLabels."owner"=jasper
  ```


> **Note:** `--set tty=true` is useful in lab/demo environments so `kubectl logs -f` streams alerts to stdout immediately without buffering.

Verify:

```bash
kubectl get pods -n falco
```

Expected (one pod per node):

```
NAME           READY   STATUS
falco-xxxxx    1/1     Running
falco-yyyyy    1/1     Running
```

---

# Step 2: Verify the DaemonSet

```bash
kubectl get daemonset -n falco
```

Output:

```
NAME    DESIRED   CURRENT   READY
falco   1         1         1
```

Each node should have one Falco pod (`DESIRED` should match your node count).

---

# Step 3: View Falco Logs

```bash
kubectl logs -n falco daemonset/falco -f
```

Initially you should see something like:

```
Falco version: x.x.x
Loading rules from file /etc/falco/falco_rules.yaml
Starting internal webserver
```

---

# Lab 1: Detect Shell Inside Container

Create `nginx.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
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
      - image: nginx
        name: nginx
```

Deploy:

```bash
kubectl apply -f nginx.yaml
```

Find the pod name (deployments create pods with a generated suffix, e.g. `nginx-7f8fd9c4d-abcde`):

```bash
kubectl get pods -l app=nginx
```

Set it as a variable for convenience:

```bash
POD=$(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
```

Open a shell:

```bash
kubectl exec -it $POD -- /bin/sh
```

Falco should immediately detect this (default rule: **Terminal shell in container**):

```
Warning A shell was spawned in a container with an attached terminal
(user=root user_loginuid=-1 container_id=xxxx container_name=nginx
shell=sh parent=runc cmdline=sh terminal=34816)
```

This is one of the most common CKS demonstrations.

---

# Lab 2: Detect Sensitive File Access

Enter the container:

```bash
kubectl exec -it $POD -- sh
```

Read passwd:

```bash
cat /etc/passwd
```

Modify the file:

```bash
echo test >> /etc/passwd
```

Falco should generate an alert similar to (default rule: **Write below etc**):

```
Warning File below /etc opened for writing
(user=root command=sh file=/etc/passwd container_name=nginx)
```

---

# Lab 3: Detect Package Installation

Inside the container:

Debian/Ubuntu-based images:

```bash
apt update
apt install vim
```

Alpine-based images:

```bash
apk add curl
```

Falco default rule **Launch Package Management Process** watches for:

```
apt, apt-get, dpkg
yum, dnf, rpm
apk
```

Reason: containers should generally be treated as immutable — package installs at runtime are a common indicator of compromise or drift.

---

# Lab 4: Detect Privilege Escalation

Create `privileged.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged
spec:
  containers:
  - name: test
    image: ubuntu
    command:
      - sleep
      - "3600"
    securityContext:
      privileged: true
```

Deploy:

```bash
kubectl apply -f privileged.yaml
```

Falco alert (default rule: **Launch Privileged Container**):

```
Warning Privileged container started
(user=root container_name=test image=ubuntu)
```

> **Note:** if your cluster enforces a Pod Security Admission policy (e.g. `restricted` or `baseline`), this pod may be rejected before it ever runs. For the lab to work, deploy it into a namespace with no PSA restriction (or `privileged` PSA level), since a privileged pod that's blocked at admission never reaches the kubelet/Falco.

---

# Lab 5: Detect Shell Spawned by Nginx

Enter nginx:

```bash
kubectl exec -it $POD -- sh
```

Run:

```bash
bash
```

or:

```bash
sh
```

Falco alert (same underlying rule as Lab 1, **Terminal shell in container**, but now with `nginx` visible as the parent process in the cmdline/lineage):

```
Warning A shell was spawned in a container with an attached terminal
(user=root parent=nginx shell=sh container_name=nginx)
```

---

# Lab 6: Detect Unexpected Network Tool

Inside the pod:

```bash
wget google.com
```

or:

```bash
curl https://google.com
```

Falco default rule **Contact K8S API Server From Container** / **Unexpected outbound connection** (rule name depends on your ruleset version) flags binaries like:

```
curl
wget
nc
ncat
```

Useful as a coarse malware/exfiltration indicator, though it will also fire on legitimate health checks — tune allow-lists accordingly.

---

# Lab 7: Detect Crypto-Miner-Like Behavior

Inside the container:

```bash
yes > /dev/null &
```

or:

```bash
openssl speed
```

Falco's default rules don't include a dedicated "high CPU" behavioral detector — CPU-based anomaly detection is not something eBPF/syscall tracing does out of the box. What *can* trigger default rules here is process/behavior-based, not load-based, e.g. known miner binary names, or connections to known mining pool domains/ports if the falls under a network-based rule. Treat this lab as illustrative rather than a guaranteed alert; for real crypto-mining detection you typically need a curated rule referencing known miner process names or pool domains.

---

# Lab 8: Detect File Deletion

Inside the container:

```bash
rm -rf /etc/*
```

Falco default rule **Remove Bin Directory** / **Delete or rename shell history** covers specific sensitive paths; a broad `rm -rf /etc/*` will likely also trip the **Write below etc** family of rules as files are removed/modified under `/etc`. Exact rule name depends on your Falco ruleset version — check `kubectl logs` output for the actual rule that fires.

---

# Lab 9: Detect Host File Access

Create `hostmount.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostmount
spec:
  containers:
  - name: host
    image: ubuntu
    command:
    - sleep
    - "3600"
    volumeMounts:
    - name: host
      mountPath: /host
  volumes:
  - name: host
    hostPath:
      path: /
```

Enter:

```bash
kubectl exec -it hostmount -- bash
```

Read:

```bash
cat /host/etc/shadow
```

Falco default rule **Read sensitive file untrusted** (or **Read sensitive file trusted after startup**):

```
Warning Sensitive file opened for reading by non-trusted program
(user=root file=/host/etc/shadow container_name=host)
```

> **Note:** this pod does not need `privileged: true` to mount the host root via `hostPath` — that's exactly why unrestricted `hostPath` mounts are a CKS exam topic in their own right (see the Pod Security Standards / admission control objectives).

---

# Lab 10: Detect New/Unexpected Binary Execution

```bash
kubectl exec -it $POD -- sh
```

Run:

```bash
nc -l -p 4444
```

or:

```bash
python3
```

Falco flags these via the same **Terminal shell in container** / suspicious-binary rules already covered in Labs 1, 5, and 6, since `nc` and interpreters like `python` are common post-exploitation tools inside a normally single-purpose container like nginx.

---

# Behavioral activities using Custom Falco Rules

1. Create custom-rules.yaml (reference file, not deployed directly)

```
cat > custom-rules.yaml << 'EOF'
- rule: Detect Curl Execution
  desc: Detect curl inside containers
  condition: spawned_process and container and proc.name = "curl"
  output: >
    Curl executed in container
    (user=%user.name command=%proc.cmdline container=%container.name)
  priority: WARNING
  tags: [container]
EOF
```

2. Build values.yaml with the rule embedded + the required owner label

```   
 cat > values.yaml << 'EOF'
customRules:
  custom-rules.yaml: |-
    - rule: Detect Curl Execution
      desc: Detect curl inside containers
      condition: spawned_process and container and proc.name = "curl"
      output: >
        Curl executed in container
        (user=%user.name command=%proc.cmdline container=%container.name)
      priority: WARNING
      tags: [container]

podLabels:
  owner: jasper
EOF
```


3. Deploy/upgrade via Helm

```
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm upgrade --install falco falcosecurity/falco \
  -n falco --create-namespace \
  -f values.yaml
```

4. Confirm the rollout completes clean


```

kubectl rollout status daemonset/falco -n falco
kubectl get pods -n falco -o wide

```
Expect 3 out of 3 new pods updated and all 3 pods Running with no Gatekeeper FailedCreate events.

5. Verify the rules loaded

```
kubectl logs -n falco daemonset/falco -c falco | grep -A5 "Loading rules"

Expect:
Loading rules from:
   /etc/falco/falco_rules.yaml | schema validation: ok
   /etc/falco/rules.d/custom-rules.yaml | schema validation: ok
 
```
6. Trigger and confirm the alert

```
POD=$(kubectl get pods -n falco -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD -n falco -- curl google.com

kubectl logs -n falco daemonset/falco -c falco -f | grep -i curl
Expect:
Warning Curl executed in container (user=root command=curl google.com container=falco) ...
```   
---

# Viewing Events

Plain logs:

```bash
kubectl logs -n falco daemonset/falco -f
```

As JSON (if Falco is configured with `json_output: true`):

```bash
kubectl logs -n falco daemonset/falco | jq
```

---

# CKS Exam Scenario 1: Investigate a Compromised Pod

```bash
kubectl get pods
kubectl logs -n falco daemonset/falco --since=10m
kubectl exec -it compromised -- sh
```

Look for alert categories such as:

```
Shell spawned in container
Sensitive file read/write
Unexpected process execution
```

---

# CKS Exam Scenario 2: Detect Privileged Containers

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace} {.metadata.name} {.spec.containers[*].securityContext.privileged}{"\n"}{end}'
```

Cross-reference with Falco:

```
Privileged container started
```

---

# CKS Exam Scenario 3: Detect Host Filesystem Access

```bash
kubectl apply -f hostmount.yaml
kubectl exec -it hostmount -- cat /host/etc/passwd
```

Falco:

```
Sensitive file opened for reading by non-trusted program
```

---

# CKS Exam Tips

* Know how to install Falco using Helm and where its config/rules live.
* Verify the Falco DaemonSet has one ready pod per node.
* Tail Falco logs with `kubectl logs -n falco daemonset/falco -f`.
* Recognize common default detections:
  * Shell spawned inside a container
  * Privileged container started
  * Sensitive file read/write (`/etc/passwd`, `/etc/shadow`)
  * Package manager execution (`apt`, `yum`, `apk`)
  * Unexpected binaries (`curl`, `wget`, `nc`)
  * Host filesystem access via `hostPath`
* Be able to add a custom rule and get Falco to pick it up (via Helm `customRules` values or a rules ConfigMap), then confirm with a rollout restart.
* Understand that Falco provides **runtime** security by analyzing Linux syscalls via eBPF or a kernel module — complementary to static/pre-deployment scanners such as kube-score, Checkov, or Trivy.
* Exact default rule *names* vary slightly between Falco ruleset versions — during the exam, don't memorize exact alert text; focus on the behavior categories and how to read/filter `kubectl logs` for them.

## Official References

* Falco project: <https://falco.org/>
* Falco default rules reference: <https://falco.org/docs/reference/rules/default-rules/>
* Sysdig blog (CVE-2019-11246 detection with Falco): <https://sysdig.com/blog/how-to-detect-kubernetes-vulnerability-cve-2019-11246-using-falco/>
* Kubernetes security monitoring at scale with Falco: <https://medium.com/@SkyscannerEng/kubernetes-security-monitoring-at-scale-with-sysdig-falco-a60cfdb0f67a>
---

 
