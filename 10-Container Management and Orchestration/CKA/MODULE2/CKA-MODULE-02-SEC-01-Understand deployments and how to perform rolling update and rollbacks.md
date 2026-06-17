# Step-by-Step Guide to Kubernetes Deployment: Update, Rollback, Scale & Delete

## Table of Contents
- [Introduction to Deployments](#introduction-to-deployments)
- [Use Cases](#use-cases)
- [Types of Kubernetes Deployment Strategies](#types-of-kubernetes-deployment-strategies)
- [Creating a Kubernetes Deployment](#creating-a-kubernetes-deployment)
- [Updating a Deployment](#updating-a-deployment)
- [Rolling Back a Deployment](#rolling-back-a-deployment)
- [Scaling a Deployment](#scaling-a-deployment)
- [Pausing and Resuming a Rollout](#pausing-and-resuming-a-rollout)
- [Deleting a Deployment](#deleting-a-deployment)

---

## Introduction to Deployments

A **Deployment** manages a set of Pods to run an application workload, typically one that doesn't maintain state. It provides declarative updates for Pods and ReplicaSets.

### Key Features
- **Rolling Updates**: Update Pods with zero downtime
- **Rollback**: Revert to previous versions
- **Scaling**: Scale up/down the number of replicas
- **Pause/Resume**: Control the rollout process
- **History**: Track revision history of deployments

---

## Use Cases

Deployments handle several operational scenarios:

- **Create**: Deploy a ReplicaSet that creates Pods in the background
- **Update**: Declare new state by updating PodTemplateSpec
- **Rollback**: Revert to stable versions when current state is unstable
- **Scale**: Increase/decrease replicas to handle load
- **Pause/Resume**: Apply multiple fixes before starting rollout
- **Status Monitoring**: Track rollout status and identify stuck rollouts
- **Cleanup**: Remove older ReplicaSets that are no longer needed

---

## Types of Kubernetes Deployment Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **Recreate** | Terminates old version, then releases new one | Development/testing, no downtime tolerance |
| **Ramped (Rolling Update)** | Releases new version incrementally | Production, zero-downtime requirement |
| **Blue/Green** | New version alongside old, then traffic switch | Minimal risk, instant rollback capability |
| **Canary** | Releases to subset of users, then full rollout | Gradual testing with real traffic |

---

## Creating a Kubernetes Deployment

### 1. Write the Deployment Manifest

```yaml
# vi nginx-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  annotations: 
    kubernetes.io/change-cause: "image updated to 1.16.1"
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

### Understanding the Manifest

- **`.spec.selector`**: Defines how the ReplicaSet finds which Pods to manage
- **`kubernetes.io/change-cause`**: Annotation that records why the change was made
- **`replicas`**: Number of desired Pod instances
- **`template`**: Pod specification for the deployment

### 2. Create the Deployment

```bash
# Using URL
kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml

# Or using local file
kubectl apply -f nginx-deployment.yml
```

### 3. Check Rollout Status

```bash
kubectl rollout status deployment/nginx-deployment
```

**Expected Output:**
```
Waiting for deployment "nginx-deployment" rollout to finish: 2 of 3 updated replicas are available...
deployment "nginx-deployment" successfully rolled out
```

### 4. Verify Resources

```bash
# Check deployment
kubectl get deployment

# Check ReplicaSets
kubectl get rs

# Check Pods with labels
kubectl get pod --show-labels
```

### 5. View Rollout History

```bash
kubectl rollout history deployment/nginx-deployment
```

**Expected Output:**
```
deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
1         image updated to 1.14.2
```

---

## Updating a Deployment

### Option 1: Using `kubectl set image`

```bash
# Update image to version 1.16.1
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1

# Annotate the change cause
kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="image updated to 1.16.1"
```

### Option 2: Using `kubectl replace`

```yaml
# Update the image version in the YAML file
# Change image: nginx:1.14.2 to image: nginx:1.16.1
```

```bash
kubectl replace -f nginx-deployment.yaml
```

**Output:**
```
deployment.apps/nginx-deployment replaced
```

### Option 3: Using `kubectl edit`

```bash
# Edit the deployment interactively
kubectl edit deployment/nginx-deployment
```

### Check Rollout Status

```bash
kubectl rollout status deployment nginx-deployment
```

**Expected Output:**
```
deployment "nginx-deployment" successfully rolled out
# or
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
```

---

## Rolling Back a Deployment

### Scenario: Failed Update

Suppose you make a typo in the image name:

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.161
```

**Output:**
```
deployment.apps/nginx-deployment image updated
```

### 1. Check the Stuck Rollout

```bash
kubectl rollout status deployment/nginx-deployment
```

**Output:**
```
Waiting for rollout to finish: 1 out of 3 new replicas have been updated...
```

Press **Ctrl+C** to stop the status watch.

### 2. Verify ReplicaSets

```bash
kubectl get rs
```

### 3. Check Pod Status

```bash
kubectl get pods
```

**Output Example:**
```
NAME                                READY   STATUS             RESTARTS   AGE
nginx-deployment-1564180365-70iae   1/1     Running            0          25s
nginx-deployment-1564180365-jbqqo   1/1     Running            0          25s
nginx-deployment-1564180365-hysrc   1/1     Running            0          25s
nginx-deployment-3066724191-08mng   0/1     ImagePullBackOff   0          6s
```

### 4. Check Rollout History

```bash
kubectl rollout history deployment/nginx-deployment
```

**Output:**
```
deployments "nginx-deployment"
REVISION    CHANGE-CAUSE
1           kubectl apply --filename=nginx-deployment.yaml
2           kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
3           kubectl set image deployment/nginx-deployment nginx=nginx:1.161
```

### 5. View Specific Revision Details

```bash
kubectl rollout history deployment/nginx-deployment --revision=2
```

### 6. Rollback to Previous Version

```bash
# Rollback to the previous revision
kubectl rollout undo deployment/nginx-deployment

# Or rollback to a specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

**Output:**
```
deployment.apps/nginx-deployment rolled back
```

---

## Scaling a Deployment

### Manual Scaling

```bash
# Scale to 10 replicas
kubectl scale deployment/nginx-deployment --replicas=10
```

**Output:**
```
deployment.apps/nginx-deployment scaled
```

### Automatic Scaling (Horizontal Pod Autoscaler)

```bash
# Set up autoscaling based on CPU utilization
kubectl autoscale deployment/nginx-deployment --min=10 --max=15 --cpu-percent=80
```

**Output:**
```
deployment.apps/nginx-deployment scaled
```

### Proportional Scaling

When scaling a Deployment during a rollout, the controller balances additional replicas between old and new ReplicaSets.

**Example Configuration:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

#### Scaling During Rollout Example

**Initial State:**
```bash
kubectl get deploy
```
```
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     10        10        10           10          50s
```

**Update Image:**
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.161
```

**Check ReplicaSets:**
```bash
kubectl get rs
```
```
NAME                          DESIRED   CURRENT   READY     AGE
nginx-deployment-1989198191   5         5         0         9s
nginx-deployment-618515232    8         8         8         1m
```

**Verify Deployment:**
```bash
kubectl get deploy
```
```
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment     15        18        7            8           7m
```

---

## Pausing and Resuming a Rollout

### Pause a Rollout

```bash
kubectl rollout pause deployment/nginx-deployment
```

**Output:**
```
deployment.apps/nginx-deployment paused
```

### Make Updates While Paused

```bash
# Update image
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1

# Update resources
kubectl set resources deployment/nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
```

### Check History During Pause

```bash
kubectl rollout history deployment/nginx-deployment
```
```
deployments "nginx"
REVISION  CHANGE-CAUSE
1   <none>
```

### Verify ReplicaSets

```bash
kubectl get rs
```

### Resume the Rollout

```bash
kubectl rollout resume deployment/nginx-deployment
```

**Output:**
```
deployment.apps/nginx-deployment resumed
```

### Monitor After Resume

```bash
# Watch ReplicaSets
kubectl get rs --watch

# Check final state
kubectl get rs
```
```
NAME               DESIRED   CURRENT   READY     AGE
nginx-2142116321   0         0         0         2m
nginx-3926361531   3         3         3         28s
```

> **Note**: You cannot rollback a paused Deployment until you resume it.

---

## Deleting a Deployment

### Delete the Deployment

```bash
# Delete by name
kubectl delete deployment nginx-deployment

# Delete using manifest file
kubectl delete -f nginx-deployment.yml
```

### Clean Up Resources

```bash
# Delete specific resources
kubectl delete pod,service,deployment --all

# Delete all resources in namespace
kubectl delete all --all
```

---

## Summary of Commands

| Operation | Command |
|-----------|---------|
| **Create** | `kubectl apply -f deployment.yaml` |
| **View** | `kubectl get deployments` |
| **Update** | `kubectl set image deployment/name container=image:tag` |
| **Status** | `kubectl rollout status deployment/name` |
| **History** | `kubectl rollout history deployment/name` |
| **Rollback** | `kubectl rollout undo deployment/name` |
| **Scale** | `kubectl scale deployment/name --replicas=n` |
| **Pause** | `kubectl rollout pause deployment/name` |
| **Resume** | `kubectl rollout resume deployment/name` |
| **Delete** | `kubectl delete deployment/name` |

---

## Best Practices

1. **Always use `--record` flag** when applying changes (or annotate manually)
2. **Monitor rollouts** to catch issues early
3. **Use meaningful change-cause annotations** for better history tracking
4. **Test updates in dev/staging** before production
5. **Set appropriate `maxSurge` and `maxUnavailable`** values for your workload
6. **Use probes** (`livenessProbe`, `readinessProbe`) for health checks
7. **Implement proper resource limits** to avoid overloading nodes
8. **Consider using GitOps** for declarative management of manifests

---

## Key Concepts Glossary

| Term | Description |
|------|-------------|
| **ReplicaSet** | Ensures a specified number of Pod replicas are running |
| **Rolling Update** | Gradually replaces old Pods with new ones |
| **Revision** | Version history of Deployment changes |
| **maxSurge** | Maximum extra Pods that can be created during update |
| **maxUnavailable** | Maximum unavailable Pods during update |
| **PodTemplateSpec** | Template for creating new Pods |
| **change-cause** | Annotation describing why a change was made |
