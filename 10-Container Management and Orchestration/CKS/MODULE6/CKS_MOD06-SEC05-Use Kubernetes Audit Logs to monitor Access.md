# Use Kubernetes audit logs to monitor access

---

## 1. Concept Overview

Kubernetes audit logging records a chronological sequence of API requests made to the
`kube-apiserver`. Every request — who made it, what it did, on which resource, when, and
whether it was allowed — can be captured. This is critical for security monitoring,
forensic investigation, and compliance.

Key pieces you must know for the exam:

| Component | Purpose |
|---|---|
| **Audit Policy** | Defines *what* gets logged and at *what level* (`None`, `Metadata`, `Request`, `RequestResponse`) |
| **Audit Backend** | Defines *where* logs go: `log` (file) or `webhook` |
| **kube-apiserver flags** | Wire the policy file and backend into the Kube API server |
| **Audit stages** | `RequestReceived`, `ResponseStarted`, `ResponseComplete`, `Panic` |

Audit levels (least to most verbose):
- `None` — don't log
- `Metadata` — log request metadata (user, timestamp, resource, verb) but not body
- `Request` — log metadata + request body
- `RequestResponse` — log metadata + request body + response body

---

## 2. Lab Environment Assumptions

This lab assumes a `kubeadm`-based cluster where you have SSH/root access to the control-plane node  

Verify the API server is static-pod managed:

```bash
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

---

## 3. Step-by-Step Lab

### Step 1 — Create directories for policy and logs

```bash
sudo mkdir -p /etc/kubernetes/audit-policy
sudo mkdir -p /var/log/kubernetes/audit
```

### Step 2 — Write an Audit Policy file

Create `/etc/kubernetes/audit-policy/policy.yaml`.  

``` 
apiVersion: audit.k8s.io/v1
kind: Policy

# Skip the duplicate "RequestReceived" stage event for every request;
# we only care about the outcome (ResponseComplete etc.)
omitStages:
  - "RequestReceived"

rules:
  # Don't log read-only requests to non-resource URLs (health checks etc.)
  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
      - "/healthz*"
      - "/version"
      - "/metrics"
      - "/api*"
    verbs: ["get", "watch", "list"]

  # Don't log requests from noisy system components
  - level: None
    users:
      - "system:kube-proxy"
      - "system:kube-scheduler"
      - "system:kube-controller-manager"
    verbs: ["get", "watch", "list"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]

  # Don't log kubelet (node) traffic — very high volume, low audit value
  - level: None
    userGroups:
      - "system:nodes"
    verbs: ["get", "watch", "list"]

  # Don't log churny housekeeping objects (leader election / heartbeats)
  - level: None
    resources:
      - group: ""
        resources: ["endpoints"]
      - group: "coordination.k8s.io"
        resources: ["leases"]
    namespaces: ["kube-system"]
    verbs: ["get", "watch", "list", "update"]

# Don't log requests to a configmap called "controller-leader"
  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  # Log Secrets and ConfigMaps access at Metadata level only
  # (never capture body content — avoids leaking sensitive data into logs)
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]

 # Log "pods/log", "pods/status" at Metadata level
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

  # Log pod exec / attach / portforward at RequestResponse (high risk actions)
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/attach", "pods/portforward"]

  # Log everything else in the "default" namespace at Request level
  - level: Request
    namespaces: ["default"]

  # Catch-all: log metadata for everything else
  - level: Metadata

 # A catch-all rule to log all other requests at the Metadata level.
  - level: Metadata
    # Long-running requests like watches that fall under this rule will not
    # generate an audit event in RequestReceived.
    omitStages:
      - "RequestReceived"

 
```

**Exam tip:** Policy rules are evaluated **top to bottom**, first match wins. Put the most specific `None`/exclusion rules first, and a catch-all rule last.

### Step 3 — Edit the kube-apiserver static pod manifest

Edit `/etc/kubernetes/manifests/kube-apiserver.yaml`.

Add these flags under `spec.containers[0].command`:

```yaml
    - --audit-policy-file=/etc/kubernetes/audit-policy/policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-log-maxage=7
    - --audit-log-maxbackup=3
    - --audit-log-maxsize=100
```

Add matching `volumeMounts`:

```yaml
    volumeMounts:
      - name: audit-policy
        mountPath: /etc/kubernetes/audit-policy
        readOnly: true
      - name: audit-log
        mountPath: /var/log/kubernetes/audit
        readOnly: false
```

Add matching `volumes`:

```yaml
  volumes:
    - name: audit-policy
      hostPath:
        path: /etc/kubernetes/audit-policy
        type: DirectoryOrCreate
    - name: audit-log
      hostPath:
        path: /var/log/kubernetes/audit
        type: DirectoryOrCreate
```

Save the file. Because it's a static pod, kubelet detects the change automatically and restarts `kube-apiserver` — no `kubectl apply` needed.

### Step 4 — Verify the API server restarted successfully

```bash
watch crictl ps       # wait for a fresh kube-apiserver container
kubectl get pods -n kube-system | grep apiserver
```

If it doesn't come back up, check `crictl logs <container-id>` — a common exam mistake is a typo in the flag or a policy YAML syntax error.

### Step 5 — Generate some traffic to audit

```bash
kubectl create namespace demo-audit
kubectl create configmap demo-cm --from-literal=foo=bar -n demo-audit
kubectl get pods -A
kubectl create secret generic demo-secret --from-literal=pass=1234 -n demo-audit
```

### Step 6 — Inspect the audit log

```bash
sudo tail -f /var/log/kubernetes/audit/audit.log
```

Pretty-print with `jq` (each line is one JSON event):

```bash
sudo cat /var/log/kubernetes/audit/audit.log | jq -c '{user: .user.username, verb, resource: .objectRef.resource, ns: .objectRef.namespace, level, stage}'
```

Search for a specific action, e.g. who accessed secrets:

```bash
sudo cat /var/log/kubernetes/audit/audit.log \
  | jq 'select(.objectRef.resource=="secrets")'
```

Find failed/denied requests (look for non-2xx response codes):

```bash
sudo cat /var/log/kubernetes/audit/audit.log \
  | jq 'select(.responseStatus.code >= 400) | {user: .user.username, verb, uri: .requestURI, code: .responseStatus.code}'
```

---

## 4. Practice Exercises

Try these without looking at the solution first — this mirrors real exam phrasing.

**Exercise 1:**
> A cluster is running with no audit logging. Create a policy that logs `RequestResponse`
> for all operations on `deployments` in the `apps` group, and `Metadata` for everything
> else. Configure the API server to write logs to `/var/log/kube-audit/audit.log`, keep a
> maximum of 5 backups, and cap each file at 50MB.

<details>
<summary>Solution</summary>

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: RequestResponse
    resources:
      - group: "apps"
        resources: ["deployments"]
  - level: Metadata
```

apiserver flags:
```
--audit-policy-file=/etc/kubernetes/audit-policy/policy.yaml
--audit-log-path=/var/log/kube-audit/audit.log
--audit-log-maxbackup=5
--audit-log-maxsize=50
```
Remember to add the hostPath volume/mount for `/var/log/kube-audit`.
</details>

**Exercise 2:**
> Using the audit log, find the username of whoever most recently executed a command
> inside a pod (`pods/exec`) in the `kube-system` namespace.

<details>
<summary>Solution</summary>

```bash
sudo cat /var/log/kubernetes/audit/audit.log \
  | jq 'select(.objectRef.resource=="pods" and .objectRef.subresource=="exec" and .objectRef.namespace=="kube-system")' \
  | jq -s 'sort_by(.requestReceivedTimestamp) | last | .user.username'
```
</details>

**Exercise 3:**
> Modify the policy so that `system:serviceaccount:kube-system:*` service accounts are
> never logged, while all other users are logged at `Request` level.

<details>
<summary>Solution</summary>

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  - level: None
    userGroups:
      - "system:serviceaccounts:kube-system"
  - level: Request
```
Note: exclusion rules must come **before** the catch-all rule, since first match wins.
</details>

---

## 5. Common Exam Pitfalls

- Forgetting to mount both the **policy file directory** and the **log output directory**   as volumes in the static pod — the apiserver will crash-loop with a "no such file"  error.
- Wrong `apiVersion` in the policy (`audit.k8s.io/v1` is current; older exam guides show   `v1beta1`, which will fail on modern clusters).
- Forgetting that after editing a static pod manifest you do **not** run `kubectl apply` —
  the kubelet watches the manifests directory directly.
- Putting the catch-all `level: Metadata` rule **first**, which silently makes all your more specific rules below it unreachable (first match wins, so order matters).
- Not verifying the apiserver actually came back up before assuming the task is done —
  always `crictl ps` / `kubectl get pods -n kube-system` after editing.
- Using `--audit-log-path=-` sends logs to stdout instead of a file — useful to know if the task asks for stdout logging specifically.

---

## 6. Quick Reference — Flags Cheat Sheet

```
--audit-policy-file=<path>        # required: policy file
--audit-log-path=<path>           # log to file (use "-" for stdout)
--audit-log-maxage=<days>
--audit-log-maxbackup=<count>
--audit-log-maxsize=<megabytes>
--audit-webhook-config-file=<path>  # alternative: send to webhook backend
--audit-webhook-batch-max-wait=<duration>
```

---

## 7. Cleanup

```bash
kubectl delete namespace demo-audit
# Revert kube-apiserver.yaml if you want to remove audit logging afterward
```
