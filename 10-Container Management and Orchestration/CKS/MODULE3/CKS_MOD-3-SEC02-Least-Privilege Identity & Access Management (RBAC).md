#  Least-Privilege Identity & Access Management (RBAC)

---
## 0. Prerequisites

Any cluster with RBAC enabled works (`kind`, `minikube`, or the real exam clusters).
Verify RBAC is on:

```
kubectl api-versions | grep rbac.authorization.k8s.io
```

Create a scratch namespace so you don't pollute `default`:

```
kubectl create namespace cks-lab
kubectl config set-context --current --namespace=cks-lab
```

---

## 1. Core Concept: Deny by Default

Kubernetes RBAC is **deny-by-default** — a subject (user, group, or ServiceAccount) has
zero permissions until a `Role`/`ClusterRole` is bound to it. Your job in the exam is
almost always: **"give this ServiceAccount/user exactly the verbs on exactly these
resources, nothing more."**

Four objects to know cold:

| Object | Scope | Purpose |
|---|---|---|
| `Role` | Namespaced | Defines permissions within one namespace |
| `ClusterRole` | Cluster-wide | Defines permissions cluster-wide, or reusable across namespaces |
| `RoleBinding` | Namespaced | Binds a Role (or ClusterRole) to a subject, scoped to one namespace |
| `ClusterRoleBinding` | Cluster-wide | Binds a ClusterRole to a subject, cluster-wide |

**Exam trap:** a `ClusterRole` bound via a `RoleBinding` is still restricted to that
namespace. This is the standard trick for reusing one ClusterRole (e.g. `view`) across
many namespaces without duplicating YAML.

---

## 2. Lab: Build a Least-Privilege ServiceAccount for a Pod

**Scenario:** an app running in a pod needs to *list and get* Pods and ConfigMaps in its
own namespace only — nothing else.

### Step 1 — Create the ServiceAccount

```
kubectl create serviceaccount pod-reader-sa -n cks-lab
```

### Step 2 — Create the minimal Role

```
# role-pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: cks-lab
rules:
- apiGroups: [""]
  resources: ["pods", "configmaps"]
  verbs: ["get", "list", "watch"]
```

```
kubectl apply -f role-pod-reader.yaml
```

### Step 3 — Bind it

```yaml
# rolebinding-pod-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: cks-lab
subjects:
- kind: ServiceAccount
  name: pod-reader-sa
  namespace: cks-lab
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```
kubectl apply -f rolebinding-pod-reader.yaml
```

### Step 4 — Verify with `kubectl auth can-i` (memorize this — it's your exam verification tool)

```
# Should be "yes"
kubectl auth can-i list pods --as=system:serviceaccount:cks-lab:pod-reader-sa -n cks-lab

# Should be "no" — proves least privilege
kubectl auth can-i delete pods --as=system:serviceaccount:cks-lab:pod-reader-sa -n cks-lab
kubectl auth can-i list secrets --as=system:serviceaccount:cks-lab:pod-reader-sa -n cks-lab
kubectl auth can-i list pods --as=system:serviceaccount:cks-lab:pod-reader-sa -n default
```

If any of the "should be no" checks return `yes`, your Role is too broad — fix it before
moving on. This can-i loop is exactly what exam graders check.

---

## 3. Lab: Attach the ServiceAccount and Disable Auto-Mount Elsewhere

A common CKS finding: pods that don't need API access still get a token auto-mounted.
Least privilege means turning that off by default and opting in only where needed.

### Step 1 — Disable auto-mount at the ServiceAccount level

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader-sa
  namespace: cks-lab
automountServiceAccountToken: false
```

```
kubectl apply -f -  <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader-sa
  namespace: cks-lab
automountServiceAccountToken: false
EOF
```

### Step 2 — Opt in only on the pod that actually needs it

```
# pod-with-sa.yaml
apiVersion: v1
kind: Pod
metadata:
  name: reader-pod
  namespace: cks-lab
spec:
  serviceAccountName: pod-reader-sa
  automountServiceAccountToken: true   # explicit opt-in overrides the SA default
  containers:
  - name: main
    image: nginx
```

```
kubectl apply -f pod-with-sa.yaml
```

**Exam tip:** `automountServiceAccountToken` can be set on the ServiceAccount *and* on
the Pod spec. The Pod-level setting wins. Default (no field set) is `true` — so every
pod gets a token mounted unless you actively disable it. This is a frequent
"harden the cluster" finding.

---

## 4. Lab: Audit and Tighten an Over-Privileged Binding

**Scenario (very CKS-like):** you're given a namespace where a ServiceAccount already
has a ClusterRoleBinding to `cluster-admin`, and told to fix it.

### Step 1 — Find over-privileged bindings

```
# List all ClusterRoleBindings that reference cluster-admin
kubectl get clusterrolebindings -o json | \
  jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name'

# Inspect one
kubectl get clusterrolebinding <name> -o yaml
```

### Step 2 — Identify the actual subject and what it truly needs

```
kubectl describe clusterrolebinding <name>
```

Say it binds `system:serviceaccount:cks-lab:app-sa` and the app only calls
`GET /api/v1/namespaces/cks-lab/pods`.

### Step 3 — Replace with a scoped Role + RoleBinding, delete the broad binding

```
kubectl delete clusterrolebinding <name>
```

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-sa-minimal
  namespace: cks-lab
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-sa-minimal-binding
  namespace: cks-lab
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: cks-lab
roleRef:
  kind: Role
  name: app-sa-minimal
  apiGroup: rbac.authorization.k8s.io
```

### Step 4 — Re-verify

```
kubectl auth can-i --list --as=system:serviceaccount:cks-lab:app-sa -n cks-lab
```

`--list` is invaluable during the exam — it dumps every verb/resource the subject can
touch so you can eyeball whether it's actually minimal.

---

## 5. Lab: Reusable ClusterRole, Namespace-Scoped Binding

**Scenario:** you need three teams' ServiceAccounts to have read-only access to Pods,
each restricted to their own namespace, without duplicating the Role definition.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-viewer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

Bind it per-namespace with `RoleBinding`s (not `ClusterRoleBinding`):

```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-pod-viewer
  namespace: team-a
subjects:
- kind: ServiceAccount
  name: team-a-sa
  namespace: team-a
roleRef:
  kind: ClusterRole
  name: pod-viewer
  apiGroup: rbac.authorization.k8s.io
```

Repeat for `team-b`, `team-c` with their own RoleBindings, same ClusterRole. One
definition, N scoped bindings — this pattern shows up often in exam tasks phrased as
"do not create duplicate Roles."

---

## 6. Quick-Fire Verification Commands (keep these on your exam scratchpad)

```
# Can a specific SA do X in namespace Y?
kubectl auth can-i <verb> <resource> --as=system:serviceaccount:<ns>:<sa> -n <ns>

# Full permission dump for a subject
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa> -n <ns>

# Same, but for a real user with groups
kubectl auth can-i <verb> <resource> --as=<user> --as-group=<group>

# Find every binding that touches a given ClusterRole/Role
kubectl get rolebindings,clusterrolebindings -A -o json | \
  jq -r '.items[] | select(.roleRef.name=="<role-name>") | "\(.metadata.namespace // "cluster")/\(.metadata.name)"'

# List what a Role/ClusterRole actually grants
kubectl describe role <name> -n <ns>
kubectl describe clusterrole <name>
```

---

## 7. Self-Test Exercises (do these timed, ~5-7 min each — realistic exam pace)

1. Create a namespace `finance`. Create ServiceAccount `billing-sa` that can only
   `create` and `get` `Secrets` in that namespace — nothing else, no list/watch/delete.
   Verify with `can-i`.

2. You're given an existing `ClusterRoleBinding` called `legacy-full-access` binding
   ServiceAccount `legacy-sa` in namespace `ops` to `cluster-admin`. The app only needs
   to read Deployments in `ops`. Fix it.

3. Disable token auto-mounting cluster-wide for the `default` ServiceAccount in every
   namespace (hint: this can't be done in one command — think about why, and what
   `automountServiceAccountToken: false` at the SA level actually changes for pods that
   don't specify a ServiceAccount).

4. Create a ClusterRole `configmap-reader` scoped to `get`/`list` on ConfigMaps, and
   bind it via RoleBinding to a new ServiceAccount in two different namespaces,
   without creating a second ClusterRole.

<details>
<summary>Solutions (expand only after attempting)</summary>

**1.**
```
kubectl create ns finance
kubectl create sa billing-sa -n finance
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-writer
  namespace: finance
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["create", "get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: billing-sa-binding
  namespace: finance
subjects:
- kind: ServiceAccount
  name: billing-sa
  namespace: finance
roleRef:
  kind: Role
  name: secret-writer
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl auth can-i list secrets --as=system:serviceaccount:finance:billing-sa -n finance   # no
kubectl auth can-i create secrets --as=system:serviceaccount:finance:billing-sa -n finance # yes
```

**2.**
```
kubectl delete clusterrolebinding legacy-full-access
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-reader
  namespace: ops
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: legacy-sa-binding
  namespace: ops
subjects:
- kind: ServiceAccount
  name: legacy-sa
  namespace: ops
roleRef:
  kind: Role
  name: deployment-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

**3.** There's no single kubectl command for "every namespace" — you must patch each
namespace's `default` ServiceAccount individually (or script a loop):
```
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  kubectl patch sa default -n "$ns" -p '{"automountServiceAccountToken": false}'
done
```
Setting it at the SA level changes the *default* for any pod that doesn't explicitly
set `automountServiceAccountToken` itself — a pod can still override it back to `true`.

**4.**
```
kubectl create ns team-a; kubectl create ns team-b
kubectl create sa cm-sa -n team-a
kubectl create sa cm-sa -n team-b
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: configmap-reader
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list"]
EOF
for ns in team-a team-b; do
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cm-reader-binding
  namespace: $ns
subjects:
- kind: ServiceAccount
  name: cm-sa
  namespace: $ns
roleRef:
  kind: ClusterRole
  name: configmap-reader
  apiGroup: rbac.authorization.k8s.io
EOF
done
```

</details>

---
 
