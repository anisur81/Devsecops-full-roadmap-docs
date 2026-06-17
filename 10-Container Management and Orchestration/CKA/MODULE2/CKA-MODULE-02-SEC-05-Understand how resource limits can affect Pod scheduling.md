# Understanding Resource Limits and Pod Scheduling in Kubernetes

## Overview

Resource limits and requests play a crucial role in how Kubernetes schedules pods and manages cluster resources. This guide demonstrates practical examples of CPU and memory resource management.

## Prerequisites

### 1. Install Metrics Server

Metrics Server is required for resource monitoring and autoscaling capabilities:

```bash
kubectl apply -f https://raw.githubusercontent.com/pythianarora/total-practice/master/sample-kubernetes-code/metrics-server.yaml
```

### 2. Verify Installation

Check that Metrics Server is running:

```bash
kubectl get pods -n kube-system | grep metrics-server
```

### 3. Confirm Metrics Collection

Verify that metrics are being collected:

```bash
# Check node metrics
kubectl top nodes

# Check pod metrics across all namespaces
kubectl top pods --all-namespaces
```

## CPU Resource Management

### Creating a Namespace for CPU Examples

```bash
kubectl create namespace cpu-example
```

### Basic CPU Request and Limit Example

Create a pod with CPU requests and limits (`cpu-request-limit.yaml`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo
  namespace: cpu-example
spec:
  containers:
  - name: cpu-demo-ctr
    image: vish/stress
    resources:
      limits:
        cpu: "1"
      requests:
        cpu: "0.5"
    args:
    - -cpus
    - "2"
```

**Key Points:**
- The `args` section tells the container to attempt to use 2 CPUs
- The container has a CPU request of 500 milliCPU (0.5 CPU)
- The container has a CPU limit of 1 CPU

**Create and verify the pod:**

```bash
# Create the pod
kubectl apply -f https://k8s.io/examples/pods/resource/cpu-request-limit.yaml --namespace=cpu-example

# Check pod status
kubectl get pod cpu-demo --namespace=cpu-example

# View detailed pod information
kubectl get pod cpu-demo --output=yaml --namespace=cpu-example

# View resource usage
kubectl top pod cpu-demo --namespace=cpu-example
```

### CPU Request Too Large (Scheduling Failure)

When a CPU request exceeds available node resources, the pod remains in Pending state:

`cpu-request-limit-2.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo-2
  namespace: cpu-example
spec:
  containers:
  - name: cpu-demo-ctr-2
    image: vish/stress
    resources:
      limits:
        cpu: "100"
      requests:
        cpu: "100"
    args:
    - -cpus
    - "2"
```

**Create and observe:**

```bash
# Create the pod
kubectl apply -f https://k8s.io/examples/pods/resource/cpu-request-limit-2.yaml --namespace=cpu-example

# Check pod status (will show Pending)
kubectl get pod cpu-demo-2 --namespace=cpu-example

# View detailed events to see scheduling failure
kubectl describe pod cpu-demo-2 --namespace=cpu-example
```

Expected output shows:
```
Events:
  Reason                        Message
  ------                        -------
  FailedScheduling      No nodes are available that match all of the following predicates:: Insufficient cpu (3)
```

### Important Note About CPU Limits

If you don't specify a CPU limit:
- The container has no upper bound on CPU usage
- It could consume all available CPU resources on the node
- Or it inherits a default limit from the namespace via LimitRange

## Memory Resource Management

### Creating a Namespace for Memory Examples

```bash
kubectl create namespace mem-example
```

### Basic Memory Request and Limit Example

`memory-request-limit.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "100Mi"
      limits:
        memory: "200Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

**Key Points:**
- The container requests 100 MiB of memory
- The container is limited to 200 MiB
- The `--vm-bytes 150M` argument tells it to allocate 150 MiB

**Create and verify:**

```bash
# Create the pod
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit.yaml --namespace=mem-example

# Check pod status
kubectl get pod memory-demo --namespace=mem-example

# View detailed pod information
kubectl get pod memory-demo --output=yaml --namespace=mem-example

# View memory usage
kubectl top pod memory-demo --namespace=mem-example
```

### Exceeding Memory Limits (OOM Killer)

When a container exceeds its memory limit, it gets killed by the OOM killer:

`memory-request-limit-2.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo-2
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-2-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```

**Create and observe:**

```bash
# Create the pod
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit-2.yaml --namespace=mem-example

# Check pod status (will show OOMKilled)
kubectl get pod memory-demo-2 --namespace=mem-example
kubectl get pod memory-demo-2 --output=yaml --namespace=mem-example
```

Expected output:
```
NAME            READY     STATUS      RESTARTS   AGE
memory-demo-2   0/1       OOMKilled   1          24s
```

### Memory Request Too Large (Scheduling Failure)

`memory-request-limit-3.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo-3
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-3-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "1000Gi"
      limits:
        memory: "1000Gi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

**Create and observe:**

```bash
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit-3.yaml --namespace=mem-example
kubectl get pod memory-demo-3 --namespace=mem-example
kubectl describe pod memory-demo-3 --namespace=mem-example
```

## Default Resource Configuration with LimitRange

### Creating a Namespace with Default Limits

```bash
kubectl create namespace default-mem-example
```

### Creating a LimitRange

`memory-default.yaml`:
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
    defaultRequest:
      memory: 256Mi
    type: Container
```

**Apply the LimitRange:**

```bash
kubectl apply -f https://k8s.io/examples/admin/resource/memory-defaults.yaml --namespace=default-mem-example
```

### Scenario 1: Pod with No Memory Request/Limit

`memory-default-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-mem-demo
spec:
  containers:
  - name: default-mem-demo-ctr
    image: nginx
```

**Create and observe:**

```bash
kubectl apply -f https://k8s.io/examples/admin/resource/memory-defaults-pod.yaml --namespace=default-mem-example
kubectl get pod default-mem-demo --output=yaml --namespace=default-mem-example
```

The pod receives the default values:
```yaml
resources:
  limits:
    memory: 512Mi
  requests:
    memory: 256Mi
```

### Scenario 2: Pod with Limit Only

`default-memory-pod-2.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-mem-demo-2
spec:
  containers:
  - name: default-mem-demo-2-ctr
    image: nginx
    resources:
      limits:
        memory: "1Gi"
```

**Create and observe:**

```bash
kubectl apply -f https://k8s.io/examples/admin/resource/memory-defaults-pod-2.yaml --namespace=default-mem-example
kubectl get pod default-mem-demo-2 --output=yaml --namespace=default-mem-example
```

Result: Request equals limit (1Gi), not the default request value.

### Scenario 3: Pod with Request Only

`memory-default-pod-3.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-mem-demo-3
spec:
  containers:
  - name: default-mem-demo-3-ctr
    image: nginx
    resources:
      requests:
        memory: "128Mi"
```

**Create and observe:**

```bash
kubectl apply -f https://k8s.io/examples/admin/resource/memory-defaults-pod-3.yaml --namespace=default-mem-example
kubectl get pod default-mem-demo-3 --output=yaml --namespace=default-mem-example
```

Result: Request = 128Mi, Limit = 512Mi (default limit)

## Key Takeaways

### Resource Requests
- Used for **scheduling decisions**
- Guarantees minimum resources for the container
- If request > available resources, pod stays pending

### Resource Limits
- Used for **enforcing resource usage**
- Prevents containers from consuming excessive resources
- Exceeding memory limits triggers OOM killer

### Default Values with LimitRange
- Namespace-wide defaults for requests and limits
- If only limit is specified → request equals limit
- If only request is specified → limit uses namespace default
- If neither specified → both use namespace defaults

### Scheduling Impact
- Pods with high requests may remain pending
- Resource requests must be schedulable on available nodes
- Understanding resource requirements is crucial for cluster efficiency

## Cleanup

```bash
# Delete namespaces and all resources within
kubectl delete namespace cpu-example
kubectl delete namespace mem-example
kubectl delete namespace default-mem-example
```
