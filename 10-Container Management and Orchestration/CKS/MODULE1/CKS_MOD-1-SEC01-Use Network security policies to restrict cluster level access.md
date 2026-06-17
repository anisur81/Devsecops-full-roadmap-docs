
# Kubernetes Network Policy - Complete Guide

## Table of Contents
1. [Network Policy Fundamentals](#network-policy-fundamentals)
2. [Core Concepts](#core-concepts)
3. [Practice Scenarios](#practice-scenarios)
   - [Deny All Traffic](#1-deny-all-traffic-to-an-application)
   - [Block Inter-Namespace Traffic](#2-block-inter-namespace-traffic)
   - [Restrict Pod-to-Pod Communication](#3-restrict-pod-to-pod-communication)
   - [Specific Ingress/Egress](#4-allow-only-specific-ingressegress)
   - [Block Critical Resources](#5-prevent-access-to-k8s-api-server-etcd-node-metadata)
   - [Whitelist IPs](#6-allow-only-whitelisted-pods-or-ips)
4. [CKS Practice Tasks](#cks-practice-tasks)
   - [Task 1: Zero-Trust Multi-Tier App](#task-1-zero-trust-multi-tier-app)
   - [Task 2: Allow Metrics Scraping](#task-2-allow-metrics-scraping-from-monitoring-only)
   - [Task 3: Block Cross-Namespace Traffic](#task-3-block-all-cross-namespace-traffic-except-one)
   - [Task 4: External APIs Only](#task-4-allow-egress-only-to-specific-external-apis)
   - [Task 5: Hard Restriction](#task-5-hard-restriction-using-namespaceselector--podselector)
   - [Task 6: Asymmetric Traffic](#task-6-restrict-pod-to-pod-traffic-inside-namespace)
   - [Task 7: Internal Cluster Only](#task-7-allow-internal-cluster-but-block-cloud-metadata)
   - [Task 8: Dual Conditions](#task-8-ingress-allowed-only-if-both-conditions-match)
   - [Task 9: StatefulSet Isolation](#task-9-allow-statefulset-pods-only-to-their-own-pod)
   - [Task 10: Multi-Policy Challenge](#task-10-multi-policy-combination-challenge)

---

## Network Policy Fundamentals

### The Network Policy Spec

A network policy specification consists of four elements:

| Element | Description | Required |
|---------|-------------|----------|
| **podSelector** | The pods that will be subject to this policy (the policy target) | **Mandatory** |
| **policyTypes** | Specifies which types of policies are included (ingress and/or egress) | Optional* |
| **ingress** | Allowed inbound traffic to the target pods | Optional |
| **egress** | Allowed outbound traffic from the target pods | Optional |

> **Note**: `policyTypes` is optional but **recommended** to always specify explicitly.

### How policyTypes is Inferred

If you omit `policyTypes`, it will be inferred as:

1. **Ingress** - Always assumed. If not explicitly specified, considered as "no traffic allowed"
2. **Egress** - Induced from existence or non-existence of the egress element

> **Best Practice**: Always specify `policyTypes` explicitly to avoid mistakes.

---

## Core Concepts

### Namespaces and Network Policies

- Namespaces are Kubernetes' multi-tenancy mechanism
- Communications between namespaces are **allowed by default**
- Use network security policies to restrict cluster-level access

### Default Behavior

By default, pods in a Kubernetes namespace can:
- **Accept incoming traffic** from any source
- **Send outgoing traffic** to any destination

### Key Points

- A NetworkPolicy only applies to pods selected by `podSelector`
- A pod becomes isolated **only if a policy selects it**

### Types of Isolation

| Isolation Type | Trigger |
|----------------|---------|
| **Ingress-isolated** | NetworkPolicy with ingress rules selects the pod |
| **Egress-isolated** | NetworkPolicy with egress rules selects the pod |

A pod may be:
- Ingress-isolated only
- Egress-isolated only
- Both
- Neither

### Ingress vs Egress Comparison

| Feature | Ingress | Egress |
|---------|---------|--------|
| **Direction** | INTO the Pod | OUT of the Pod |
| **Keyword** | `from:` | `to:` |
| **Controls** | Who can talk *to* the Pod | Where the Pod can talk *to* |
| **Default behavior** | Allow all | Allow all |
| **DNS impact** | None | Must allow UDP 53 if egress denied |
| **Common use** | Allow frontend → backend | Restrict internet or outbound calls |
| **Applies to** | Incoming connections | Outgoing connections |

---

## Practice Scenarios

### 1. Deny All Traffic to an Application

**File: `deny-all-demo.yaml`**

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: demo-app
  labels:
    name: demo-app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
  namespace: demo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: demo
        image: nginx:1.23
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: demo-service
  namespace: demo-app
spec:
  selector:
    app: demo
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-traffic
  namespace: demo-app
spec:
  # Empty selector = applies to ALL pods in namespace
  podSelector: {}
  policyTypes:
  - Ingress  # Block all ingress unless allowed
  - Egress   # Block all egress unless allowed
  ingress: []  # NO incoming traffic is allowed
  egress: []   # NO outbound connections are allowed
```

**Testing Commands:**

```bash
# Deploy the policy
kubectl apply -f deny-all-demo.yaml

# Exec into a Pod
kubectl -n demo-app exec -it deploy/demo -- sh

# Try DNS → should fail
nslookup google.com

# Try outgoing HTTP → fail
curl https://www.google.com

# Try incoming traffic from another namespace
kubectl run test --image=busybox --rm -it -- wget -qO- demo-service.demo-app.svc.cluster.local
# Should fail / timeout
```

---

### 2. Block Inter-Namespace Traffic

**Option 1: Deny all inter-namespace ingress**

**File: `Block_All_Inter_Namespace.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-other-namespaces
  namespace: my-namespace
spec:
  podSelector: {}
  ingress:
    - from:
        - podSelector: {}  # Allow only pods from the same namespace
  policyTypes:
    - Ingress
```

**Option 2: Allow only same-namespace traffic**

**File: `Allow_only_same-namespace_traffic.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-cross-namespace
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: backend
```

```bash
# Label the namespace
kubectl label namespace backend name=backend

# Test communication
kubectl exec -it <frontend-pod> -- curl <backend-service>:8080
```

**Option 3: Block egress to other namespaces too**

**File: `Block_Egress_to_Other_Namespaces.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-other-ns
  namespace: my-namespace
spec:
  podSelector: {}
  egress:
    - to:
        - podSelector: {}  # Allow egress only within same namespace
  policyTypes:
    - Egress
```

---

### 3. Restrict Pod-to-Pod Communication

**Example: Allow frontend → backend traffic only**

```bash
# Label your pods
kubectl label pod frontend-pod app=frontend
kubectl label pod backend-pod app=backend
```

**File: `Allow_frontend_backend_traffic.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: my-namespace
spec:
  podSelector:
    matchLabels:
      app: backend  # backend is the target
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend  # only frontend allowed
    ports:
    - protocol: TCP
      port: 8080
```

---

### 4. Allow Only Specific Ingress/Egress

**File: `Allow_Specific_Ingress_Specific_Egress.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-ingress-egress
  namespace: secure-ns
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: trusted
        - podSelector:
            matchLabels:
              role: gateway
      ports:
        - protocol: TCP
          port: 9000
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: database
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
```

**What this does:**

- **Ingress**: Only traffic from namespace labeled `name=trusted` AND pods labeled `role=gateway` can reach the API pod on port 9000
- **Egress**: API pod can only reach namespace labeled `name=database` AND pods labeled `app=postgres` on port 5432

---

### 5. Prevent Access to K8s API Server, ETCD, Node Metadata

**File: `deny_access_to_API_SERVER_ETCD_NODE-Metadata.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-api-etcd-metadata
  namespace: default
spec:
  podSelector: {}  # applies to all pods in this namespace
  policyTypes:
    - Egress
  egress:
    # 1) Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

    # 2) Block Cloud Metadata (IMDS)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32

    # 3) Block Kubernetes API Server (Port 443)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - <API_SERVER_IP>/32
      ports:
        - protocol: TCP
          port: 443

    # 4) Block ETCD (Port 2379–2380)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - <CONTROL_PLANE_SUBNET>/24
      ports:
        - protocol: TCP
          port: 2379
        - protocol: TCP
          port: 2380
```

```bash
# Find your API server IP
kubectl get endpoints kubernetes -n default -o wide
```

---

### 6. Allow Only Whitelisted Pods or IPs

**File: `allow_whitelisted_IP.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-whitelisted-ips
  namespace: your-namespace
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - ipBlock:
            cidr: 10.0.1.15/32          # Specific IP
        - ipBlock:
            cidr: 192.168.10.0/24       # Subnet
```

---

## CKS Practice Tasks

### Task 1: Zero-Trust Multi-Tier App

**Namespace**: `shop`  
**Labels**:
- Frontend → `tier=frontend`
- Backend → `tier=backend`
- Database → `tier=db`

**Requirements:**
1. Frontend → Backend allowed (TCP 8080)
2. Backend → Database allowed (TCP 5432)
3. Nothing else is allowed
4. No Pod may access the internet

#### Solution

**1. Default deny all in shop:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: shop-default-deny
  namespace: shop
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**2. Allow frontend → backend (Ingress on backend):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: shop
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 8080
```

**3. Allow backend → db (Egress from backend to Pod tier=db):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-to-db
  namespace: shop
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: db
    ports:
    - protocol: TCP
      port: 5432
```

---

### Task 2: Allow Metrics Scraping From Monitoring Only

**Namespace**: `prod`
- Prometheus runs in namespace `monitoring`
- App Pods: `app=service` expose metrics on TCP 9100

**Requirements:**
- Only Pods in namespace labeled `team=monitoring`
- Access only port 9100
- Deny all other ingress
- Deny all egress except DNS

#### Solution

**1. Default deny ingress+egress for prod service pods:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prod-default-deny
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: service
  policyTypes:
  - Ingress
  - Egress
```

**2. Allow scraping from monitoring namespace on port 9100:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-scrape
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: service
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          team: monitoring
    ports:
    - protocol: TCP
      port: 9100
```

**3. Allow DNS egress only:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: service
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

> **Note**: `namespaceSelector: { team: monitoring }` requires the monitoring namespace to be labeled `team=monitoring`.

---

### Task 3: Block All Cross-Namespace Traffic Except One

**Requirements:**
- All cross-namespace Pod communication must be blocked
- **Exception**: Namespace `gateway` may access namespace `internal` on TCP 9443

#### Solution

**1. Default deny in every namespace:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: internal
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

> Repeat the same `default-deny-all` in every namespace.

**2. Allow gateway → internal on 9443:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-gateway-to-internal-9443
  namespace: internal
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: gateway
    ports:
    - protocol: TCP
      port: 9443
```

```bash
# Label the gateway namespace
kubectl label namespace gateway name=gateway
```

> **Note**: Apply the default-deny into every namespace to enforce cluster-wide default deny.

---

### Task 4: Allow Egress Only to Specific External APIs

**Pod**: `app=payments`  
**Namespace**: `finance`

**Requirements:**
- Call only three IPs on port 443:
  - `34.102.120.11/32`
  - `142.250.190.78/32`
  - `13.107.246.45/32`
- DNS must work
- Everything else blocked

#### Solution

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payments-egress-restrict
  namespace: finance
spec:
  podSelector:
    matchLabels:
      app: payments
  policyTypes:
  - Egress
  - Ingress
  egress:
  # IP 1
  - to:
    - ipBlock:
        cidr: 34.102.120.11/32
    ports:
    - protocol: TCP
      port: 443
  # IP 2
  - to:
    - ipBlock:
        cidr: 142.250.190.78/32
    ports:
    - protocol: TCP
      port: 443
  # IP 3
  - to:
    - ipBlock:
        cidr: 13.107.246.45/32
    ports:
    - protocol: TCP
      port: 443
  # Allow DNS to cluster DNS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

> **Note**: No general `0.0.0.0/0` egress allowed — internet blocked except those IPs.

---

### Task 5: Hard Restriction Using namespaceSelector + podSelector

**Namespace**: `core`  
**Vault Pod**: `app=vault`

**Requirements:**
- Accept traffic only from Pods with `access=vault`
- Those Pods must be in namespaces labeled `team=security`
- Deny all other ingress and all egress traffic

#### Solution

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: vault-restrict
  namespace: core
spec:
  podSelector:
    matchLabels:
      app: vault
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          team: security
      podSelector:
        matchLabels:
          access: vault
```

> **Note**: The single rule uses both `namespaceSelector` and `podSelector` — the peer must satisfy **both**. Because `policyTypes` includes Egress but there are no egress entries, all egress is denied.

---

### Task 6: Restrict Pod-to-Pod Traffic Inside Namespace

**Namespace**: `data`

**Rules:**
- Pods with label `role=api` may talk to `role=cache`
- Pods with label `role=cache` may NOT talk to `api`
- All other Pods in namespace must be completely isolated

#### Solution

**1. Default deny in data:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: data-default-deny
  namespace: data
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**2. Allow api → cache (egress from api to cache):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-egress-to-cache
  namespace: data
spec:
  podSelector:
    matchLabels:
      role: api
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: cache
```

**3. Allow cache to accept connections from api:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cache-ingress-from-api
  namespace: data
spec:
  podSelector:
    matchLabels:
      role: cache
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api
```

**Why asymmetric works:**
- `api → cache` requires api egress allowed AND cache ingress allowed
- To prevent `cache → api`, do NOT create egress rules on cache allowing api
- Do NOT create an ingress rule on api allowing cache
- The `data-default-deny` isolates them by default

---

### Task 7: Allow Internal Cluster But Block Cloud Metadata

**Namespace**: `default`

**Requirements:**
- Allow all traffic to private cluster CIDRs (`10.0.0.0/8`, `172.16.0.0/12`)
- Block access to metadata service: `169.254.169.254/32`
- Allow DNS
- Deny internet entirely

#### Solution

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: internal-only-block-metadata
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  - Ingress
  egress:
  # Allow 10.0.0.0/8
  - to:
    - ipBlock:
        cidr: 10.0.0.0/8
  # Allow 172.16.0.0/12
  - to:
    - ipBlock:
        cidr: 172.16.0.0/12
  # Allow DNS to cluster pods
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

> **Important Caveat**: NetworkPolicy is allow-list only (cannot explicitly blacklist). To block metadata address, ensure you do NOT include any egress rule that covers metadata, and do NOT allow a broader `0.0.0.0/0` that includes it.

---

### Task 8: Ingress Allowed Only If BOTH Conditions Match

**Namespace**: `secure`

**Requirements:**
- Allow ingress to Pods with `app=api` only if BOTH:
  - Namespace has label `client=true`
  - Pod making request has label `tier=web`
- Deny all other ingress
- Deny all egress except DNS

#### Solution

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-restrict-both-conditions
  namespace: secure
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          client: "true"
      podSelector:
        matchLabels:
          tier: web
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

> **Note**: The `from` block includes both `namespaceSelector` and `podSelector`; the peer must match **both** selectors.

---

### Task 9: Allow StatefulSet Pods Only to Their Own Pod

**Namespace**: `state`

**StatefulSet Pods:**
- `pod-0` should connect only to `pod-0`
- `pod-1` only to `pod-1`
- `pod-2` only to `pod-2`

#### Solution

**Approach**: Ensure each stateful pod gets a unique label (e.g., `stateful-index: "0"`, `"1"`, `"2"`).

```bash
# Label pods manually
kubectl label pod statefulset-0 stateful-index=0
kubectl label pod statefulset-1 stateful-index=1
kubectl label pod statefulset-2 stateful-index=2
```

**Policy for pod-0:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: stateful-0-self-only
  namespace: state
spec:
  podSelector:
    matchLabels:
      stateful-index: "0"
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          stateful-index: "0"
  egress:
  - to:
    - podSelector:
        matchLabels:
          stateful-index: "0"
```

> Repeat similar policy for index "1" and "2".

> **Note**: NetworkPolicy cannot match a Pod's hostname or ordinal directly. Per-pod isolation requires unique labels.

---

### Task 10: Multi-Policy Combination Challenge

**Namespace**: `prod`

**Requirements:**
- Default deny everything
- Allow ingress only from Pods labeled `allow=true`
- Allow egress to database Pod labeled `db=true` on 3306
- Allow egress to KMS IP `10.16.45.20/32` on port 443
- Allow DNS
- Block internet fully

#### Solution

**1. Default deny for all Pods in prod:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prod-default-deny
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**2. Allow ingress from pods allow=true:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prod-allow-ingress-from-allow-true
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          allow: "true"
```

**3. Allow egress to DB Pod on 3306:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prod-egress-to-db
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          db: "true"
    ports:
    - protocol: TCP
      port: 3306
```

**4. Allow egress to KMS IP on 443:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prod-egress-to-kms
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.16.45.20/32
    ports:
    - protocol: TCP
      port: 443
```

**5. Allow DNS egress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prod-allow-dns
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

> **Note**: Kubernetes evaluates multiple NetworkPolicies additively. The union of egress rules yields exactly the allowed destinations. Internet is blocked because no `0.0.0.0/0` egress is present.

---

## Quick Reference: Network Policy Patterns

| Pattern | podSelector | policyTypes | ingress | egress |
|---------|-------------|-------------|---------|--------|
| **Deny all** | `{}` | `[Ingress, Egress]` | `[]` | `[]` |
| **Allow all within namespace** | `{}` | `[Ingress]` | `from: [podSelector: {}]` | - |
| **Allow specific pod** | `{matchLabels: {app: target}}` | `[Ingress]` | `from: [{podSelector: {matchLabels: {app: source}}}]` | - |
| **Allow namespace** | `{matchLabels: {app: target}}` | `[Ingress]` | `from: [{namespaceSelector: {matchLabels: {team: ops}}}]` | - |
| **Allow both** | `{matchLabels: {app: target}}` | `[Ingress]` | `from: [{namespaceSelector: {...}, podSelector: {...}}]` | - |
| **Egress only** | `{matchLabels: {app: source}}` | `[Egress]` | - | `to: [{podSelector: {...}}]` |
| **IP whitelist** | `{matchLabels: {app: target}}` | `[Ingress]` | `from: [{ipBlock: {cidr: 10.0.0.0/24}}]` | - |

---

## Key Takeaways for CKS

1. **Default deny** - Start with deny-all policies and explicitly allow what's needed
2. **Always specify `policyTypes`** - Avoid ambiguity
3. **Namespace isolation** - Cross-namespace traffic is allowed by default; block with network policies
4. **DNS is critical** - When blocking egress, always allow DNS (UDP 53)
5. **Combine selectors** - Use `namespaceSelector` + `podSelector` for fine-grained control
6. **Allow-list only** - NetworkPolicy only supports allow rules; cannot explicitly deny
7. **Multiple policies are additive** - The union of all applicable policies determines allowed traffic
8. **Apply to all namespaces** - To enforce cluster-wide policies, apply to each namespace
