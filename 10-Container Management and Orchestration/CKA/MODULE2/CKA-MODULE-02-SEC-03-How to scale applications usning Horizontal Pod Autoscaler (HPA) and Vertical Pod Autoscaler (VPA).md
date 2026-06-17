# Scaling Applications with Horizontal Pod Autoscaler (HPA) and Vertical Pod Autoscaler (VPA)

## Prerequisites: Installing Metrics Server

The Metrics Server is required for both HPA and VPA to collect resource usage metrics from pods and nodes.

### Install Metrics Server

```bash
kubectl apply -f https://raw.githubusercontent.com/pythianarora/total-practice/master/sample-kubernetes-code/metrics-server.yaml
```

### Verify Installation

```bash
kubectl get pods -n kube-system | grep metrics-server
```

### Confirm Metrics Collection

```bash
kubectl top nodes
kubectl top pods --all-namespaces
```

---

## Horizontal Pod Autoscaler (HPA)

### How HPA Works

HPA operates through the following process:

1. **Monitoring Metrics**: Continuously monitors specified metrics (CPU usage, memory usage, custom metrics)
2. **Evaluating Thresholds**: Compares current metric values against predefined thresholds
3. **Scaling Pods**: 
   - Increases pod count when metrics exceed thresholds
   - Decreases pod count when metrics drop below thresholds

### Advantages of HPA

- **Elasticity**: Automatically adjusts pod count to meet current demand
- **Cost-Efficiency**: Optimizes resource usage by adding/removing pods as needed
- **Performance**: Maintains application performance during high traffic periods

### Step-by-Step HPA Deployment

#### Step 1: Create a Deployment

Create `hpa-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-deploy
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
        resources:
          requests:
            cpu: "25m"
          limits:
            cpu: "200m"
```

Apply the deployment:

```bash
kubectl apply -f hpa-deployment.yaml
```

#### Step 2: Create a Service

Create `nginx-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  labels:
    app: nginx
spec:
  ports:
  - port: 80
  selector:
    app: nginx
```

Apply the service:

```bash
kubectl apply -f nginx-service.yaml
```

#### Step 3: Create HPA Resource

Create `hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-deploy
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Apply the HPA:

```bash
kubectl apply -f hpa.yaml
```

#### Step 4: Test and Monitor

Check initial pod status:

```bash
kubectl top po
```

Generate load:

```bash
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://nginx-svc; done"
```

Monitor CPU utilization:

```bash
kubectl top po
```

Watch HPA status:

```bash
kubectl get hpa -w
```

---

## Vertical Pod Autoscaler (VPA)

### What is VPA?

Vertical Pod Autoscaler (VPA) adjusts CPU and memory resource requests and limits for pods. Instead of adding more pods, VPA modifies the resources allocated to existing pods.

### How VPA Works

1. **Monitoring Resource Usage**: Continuously monitors CPU and memory usage of pods
2. **Recommending Resources**: Based on usage patterns, VPA recommends resource adjustments
3. **Applying Changes**: Automatically applies recommended changes to pod specifications (typically requires pod restart)

### Advantages of VPA

- **Resource Optimization**: Ensures pods have appropriate resources, preventing over/under-provisioning
- **Simplified Resource Management**: Reduces manual adjustment of resource requests and limits
- **Improved Performance**: Dynamically adjusts resources based on actual usage

### VPA Implementation Example

#### Step 1: Create Namespace

```bash
kubectl create namespace mem-example
```

#### Step 2: Create VPA Application

Create `vpa-limit.yaml`:

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

> **Note**: The `args` section provides arguments to the container:
> - `--vm-bytes`, `150M`: Attempts to allocate 150 MiB of memory
> - `--vm-hang`, `1`: Hangs processes to simulate memory pressure

Apply the configuration:

```bash
kubectl apply -f vpa-limit.yaml --namespace=mem-example
```

Or directly from URL:

```bash
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit.yaml --namespace=mem-example
```

#### Step 3: Verify and Monitor

Check pod status:

```bash
kubectl get pod memory-demo --namespace=mem-example
```

View detailed pod information:

```bash
kubectl get pod memory-demo --output=yaml --namespace=mem-example
```

> **Expected Output**: The pod should use approximately 150 MiB of memory, which exceeds the 100 MiB request but stays within the 200 MiB limit.

---

## Best Practices Summary

### HPA Recommendations
- Set appropriate resource requests and limits
- Define realistic CPU/memory thresholds
- Consider using custom metrics for business-specific scaling
- Implement proper readiness/liveness probes

### VPA Recommendations
- Test thoroughly before production deployment
- Consider pod restart implications
- Monitor VPA recommendations before auto-applying
- Combine with HPA for comprehensive scaling

### Combined Approach
- Use HPA for application-level scaling based on demand
- Use VPA for optimizing individual pod resources
- Monitor and adjust configurations based on workload patterns
- Implement comprehensive monitoring and alerting
