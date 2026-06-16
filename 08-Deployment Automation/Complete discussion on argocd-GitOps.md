# ArgoCD Installation and Configuration Guide


## What is Argo CD?

Argo CD is a Kubernetes-native GitOps continuous delivery tool. It automatically deploys applications to Kubernetes by monitoring a Git
repository and ensuring the cluster state matches the desired state stored in Git.

## Architecture
## Components
1. API Server
Provides UI, CLI, and API access.
2. Repository Server
Clones Git repositories and generates manifests.
3. Application Controller
Monitors application state and performs synchronization.
4. Redis
Caching and performance optimization.

## 1. Install ArgoCD

### Create Namespace
```bash
kubectl create namespace argocd
```

### Install ArgoCD (Server-Side)
```bash
kubectl apply --server-side \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Force Conflict Override (if conflicts exist)
```bash
kubectl apply --server-side --force-conflicts \
  -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
> **Note**: This tells Kubernetes "Argo CD should take full control even if conflicts exist"

### Verify CRD Installation
```bash
kubectl get crd | grep argoproj
```

## 2. Configure External Access

### Change Service Type to NodePort
```bash
kubectl edit svc argocd-server -n argocd
```
Change `type: ClusterIP` to `type: NodePort`

## 3. Get Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

## 4. Create GitHub Repository

Create a new repository: `https://github.com/anisur81/argocdnginx`

## 5. Application Manifests

### Deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argonginx-deployment
  labels:
    app: argonginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: argonginx
  template:
    metadata:
      labels:
        app: argonginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

### Service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: argonginx-service
spec:
  selector:
    app: argonginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

## 6. ArgoCD Application Definition

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
  namespace: argolab
spec:
  project: default
  source:
    repoURL: https://github.com/anisur81/argocdnginx.git
    path: .
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: argolab
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 7. Apply ArgoCD Application

Save the Application definition to a file (e.g., `application.yaml`) and apply:

```bash
kubectl apply -f application.yaml
```

## Access ArgoCD UI

1. Get the NodePort:
```bash
kubectl get svc argocd-server -n argocd
```

2. Access the UI: `https://<NODE_IP>:<NODE_PORT>`

3. Login with username: `admin` and the password retrieved in step 3

## Verification Commands

```bash
# Check Application status
kubectl get application -n argolab

# Check deployment status
kubectl get deployment -n argolab

# Check service
kubectl get svc -n argolab

# Check pods
kubectl get pods -n argolab
```
```

This Markdown document contains all the commands and configurations you need to:
- Install ArgoCD
- Configure external access
- Set up your GitHub repository
- Deploy the NGINX application
- Access the ArgoCD UI
