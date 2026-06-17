# Kubernetes RBAC Management Guide

## Table of Contents
1. [RBAC Overview](#rbac-overview)
2. [RBAC Components](#rbac-components)
3. [User Accounts vs Service Accounts](#user-accounts-vs-service-accounts)
4. [Service Account RBAC Example](#service-account-rbac-example)
5. [User RBAC Example](#user-rbac-example)
6. [ClusterRole and ClusterRoleBinding Examples](#clusterrole-and-clusterrolebinding-examples)
7. [Basic Authentication User Example](#basic-authentication-user-example)

---

## RBAC Overview

**RBAC (Role-Based Access Control)** in Kubernetes is a sophisticated framework designed to regulate access to the Kubernetes API, offering fine-grained control over who can access specific resources and what actions they can perform.

### How RBAC Works

RBAC relies on a series of Kubernetes objects:
- **Roles & ClusterRoles**: Define permissions
- **RoleBindings & ClusterRoleBindings**: Associate permissions with users/groups/service accounts

---

## RBAC Components

| Component | Description | Scope |
|-----------|-------------|-------|
| **Role** | Defines a set of permissions | Namespace-specific |
| **ClusterRole** | Defines a set of permissions | Cluster-wide |
| **RoleBinding** | Associates roles with users/groups/SA | Namespace-specific |
| **ClusterRoleBinding** | Associates cluster roles with users/groups/SA | Cluster-wide |

---

## User Accounts vs Service Accounts

### Key Differences

| Feature | User Account | Service Account |
|---------|--------------|-----------------|
| **Represents** | Human users | Application processes |
| **Authentication** | External service | Kubernetes API |
| **Scope** | Global (cluster-wide) | Namespaced |
| **Managed via API** | No | Yes |
| **Backed by Kubernetes Object** | No | Yes (ServiceAccount) |

---

## Service Account RBAC Example

### Step 1: Check RBAC is Enabled

```bash
# Verify RBAC is enabled in your cluster
kubectl api-versions | grep rbac
```

### Step 2: Create Service Account

```bash
# Create a Service Account
kubectl create serviceaccount demo-user

# Generate authorization token
TOKEN=$(kubectl create token demo-user)
```

### Step 3: Switch to Service Account Context

```bash
# Check current context
kubectl config current-context

# Add new context for demo-user
kubectl config set-credentials demo-user --token=$TOKEN
kubectl config set-context demo-user-context \
  --cluster=kubernetes \
  --namespace=default \
  --user=demo-user

# Switch to demo-user context
kubectl config use-context demo-user-context

# Test access (should be forbidden initially)
kubectl get pods
```

### Step 4: Create a Role

```yaml
# role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: demo-role
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "create", "update"]
```

```bash
# Apply the Role
kubectl apply -f role.yaml
```

### Step 5: Create a RoleBinding

```yaml
# rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: demo-role-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: demo-role
subjects:
  - namespace: default
    kind: ServiceAccount
    name: demo-user
```

```bash
# Apply the RoleBinding
kubectl apply -f rolebinding.yaml
```

### Step 6: Verify Permissions

```bash
# Switch to demo-user context
kubectl config use-context demo-user-context

# Test Pod operations
kubectl get pods                    # ✅ Success
kubectl run nginx --image=nginx:latest  # ✅ Success
kubectl delete pod nginx            # ❌ Forbidden (delete verb not included)
```

---

## User RBAC Example

### Step 1: Generate User Keys

```bash
# Create directory for user keys
mkdir userkey && cd userkey

# Generate private key
openssl genrsa -out demo-user.key 2048

# Create Certificate Signing Request
openssl req -new -key demo-user.key -out demo-user.csr \
  -subj "/CN=demo-user"
```

### Step 2: Sign Certificate with Kubernetes CA

```bash
# Verify CA certificate and key location
# Typically in /etc/kubernetes/pki/
file /etc/kubernetes/pki/ca.crt
file /etc/kubernetes/pki/ca.key

# Move CSR to PKI directory
cp demo-user.csr /etc/kubernetes/pki/
cp demo-user.key /etc/kubernetes/pki/

# Sign the CSR
cd /etc/kubernetes/pki/
openssl x509 -req -CA ca.crt -CAkey ca.key \
  -CAcreateserial -days 2000 \
  -in demo-user.csr -out demo-user.crt
```

### Step 3: Configure kubectl for User

```bash
# Set user credentials
kubectl config set-credentials demo-user \
  --client-certificate=/etc/kubernetes/pki/demo-user.crt \
  --client-key=/etc/kubernetes/pki/demo-user.key \
  --embed-certs=true

# Get cluster name
kubectl config get-contexts

# Set user context
kubectl config set-context demo-user-kubernetes \
  --cluster=kubernetes \
  --namespace=default \
  --user=demo-user
```

### Step 4: Test Access

```bash
# Switch to user context
kubectl config use-context demo-user-kubernetes

# Test access (will be forbidden without RBAC)
kubectl get pods

# Alternative: Use --as flag without switching context
kubectl get pods --as demo-user
```

---

## ClusterRole and ClusterRoleBinding Examples

### Example 1: Node Reader

```yaml
# clusterrole-node-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "watch", "list"]
```

```yaml
# clusterrolebinding-node-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
  - kind: User
    name: demo-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Apply the configuration
kubectl apply -f clusterrole-node-reader.yaml
kubectl apply -f clusterrolebinding-node-reader.yaml

# Test access
kubectl get nodes --as demo-user
```

### Example 2: Pods Viewer Across All Namespaces

```yaml
# clusterrole-pod-viewer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pods-viewer
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
```

```yaml
# clusterrolebinding-pod-viewer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pods-viewer-binding
subjects:
  - kind: User
    name: demo-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pods-viewer
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Apply the configuration
kubectl apply -f clusterrole-pod-viewer.yaml
kubectl apply -f clusterrolebinding-pod-viewer.yaml

# Test access across all namespaces
kubectl get pods -A --as demo-user
```

---

## Basic Authentication User Example

### Step 1: Configure Basic Auth User

```bash
# Set credentials with basic authentication
kubectl config set-credentials basic-auth-user \
  --username=basic-auth-user \
  --password=admin123

# Create context
kubectl config set-context basic-auth-user-kubernetes \
  --cluster=kubernetes \
  --user=basic-auth-user
```

### Step 2: Create RBAC for Basic Auth User

```yaml
# clusterrole-basic-auth.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pods-viewer-basic-auth
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]
```

```yaml
# clusterrolebinding-basic-auth.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pods-viewer-basic-auth-binding
subjects:
  - kind: User
    name: basic-auth-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pods-viewer-basic-auth
  apiGroup: rbac.authorization.k8s.io
```

```bash
# Apply RBAC configurations
kubectl apply -f clusterrole-basic-auth.yaml
kubectl apply -f clusterrolebinding-basic-auth.yaml

# Test access
kubectl get pods -A --as basic-auth-user
```

---

## Common RBAC Verbs

| Verb | Description |
|------|-------------|
| `get` | Retrieve a specific resource |
| `list` | List multiple resources |
| `watch` | Watch for changes |
| `create` | Create a new resource |
| `update` | Update an existing resource |
| `patch` | Partially update a resource |
| `delete` | Delete a resource |
| `deletecollection` | Delete multiple resources |

---

## Troubleshooting RBAC

### Verify Current Context
```bash
kubectl config current-context
kubectl config view
```

### Test Permissions
```bash
# Check if you can perform an action
kubectl auth can-i list pods

# Check for a specific user
kubectl auth can-i list pods --as=demo-user

# Check all permissions
kubectl auth can-i --list
```

### Common Errors
- **"Forbidden"**: User lacks required permissions
- **"Unauthorized"**: Authentication failed
- **"No RBAC rules found"**: No roles assigned

---

## Best Practices

1. **Principle of Least Privilege**: Grant only necessary permissions
2. **Use Namespaces**: Isolate resources and permissions
3. **Avoid ClusterRoles**: Use Roles when possible
4. **Regular Audits**: Review RBAC configurations regularly
5. **Use Groups**: Group users with similar roles
6. **Service Accounts for Pods**: Never use default service account in production

---

## Quick Reference

```bash
# List all RBAC objects
kubectl get roles,clusterroles,rolebindings,clusterrolebindings -A

# Describe a role
kubectl describe role demo-role

# Check who can do what
kubectl auth can-i --list --as=demo-user

# Switch context
kubectl config use-context demo-user-kubernetes

# View current configuration
kubectl config view --raw
```

---

*This guide provides comprehensive examples for implementing RBAC in Kubernetes. Always test in a non-production environment first and follow security best practices.*
