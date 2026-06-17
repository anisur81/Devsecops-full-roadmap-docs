# Kubernetes Service Types: ClusterIP, NodePort, LoadBalancer, and Endpoints

## Overview

In Kubernetes, a **Service** is a method for exposing a network application that is running as one or more Pods in your cluster. Services provide stable network endpoints to access your applications, regardless of pod lifecycles.

---

## Kubernetes Service Types

All Kubernetes Services ultimately forward network traffic to a set of Pods they represent. However, several different types of Service exist with their own characteristics and use cases:

### 1. ClusterIP Services

**ClusterIP Services** assign an IP address that can be used to reach the Service from within your cluster. This type doesn't expose the Service externally.

**Key Characteristics:**
- Internal cluster access only
- Assigns a virtual IP address
- Default service type if not specified
- Best for internal communication between microservices

---

### 2. NodePort Services

**NodePort Services** are exposed externally through a specified static port binding on each of your Nodes. You can access the Service by connecting to the port on any of your cluster's Nodes.

**⚠️ Important Limitations:**

| Limitation | Description |
|------------|-------------|
| **Security Risk** | Anyone who can connect to the port on your Nodes can access the Service |
| **Port Conflict** | Each port number can only be used by one NodePort Service at a time |
| **Node Overhead** | Every Node listens to the port by default, even if not running a Pod for the Service |
| **No Load-Balancing** | Clients are served by the Node they connect to - no automatic distribution |

**💡 Use Case:** NodePort Services are generally used to facilitate your own load-balancing solution that reroutes traffic from outside the cluster.

---

### 3. LoadBalancer Services

**LoadBalancer Services** are exposed outside your cluster using an external load balancer resource. This requires integration with a cloud provider (AWS, GCP, Azure, etc.).

**Key Characteristics:**
- Automatically provisions a load balancer in your cloud account
- Provides an external IP address
- Handles traffic distribution across nodes
- Most expensive option (cloud provider costs)

---

### 4. ExternalName Services

**ExternalName Services** allow you to conveniently access external resources from within your Kubernetes cluster. Unlike other Service types, they don't proxy traffic to your Pods.

**How It Works:**
1. Set `spec.externalName` to the external address (e.g., `example.com`)
2. Kubernetes adds a CNAME DNS record to your cluster
3. Internal addresses resolve to the external address
4. Easy to change external addresses without reconfiguring workloads

---

## Hands-On Deployment Guide

### Step 1: Deploy the Sample Application

Create the deployment manifest (`app.yaml`):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
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
        image: nginx
        ports:
          - containerPort: 80
```

Deploy the application:

```bash
kubectl apply -f app.yaml
```

---

### Step 2: Create a ClusterIP Service

Create `clusterip-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: clusterip-service
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 8080
      targetPort: 80
```

**📝 Manifest Notes:**
- `spec.type` set to `ClusterIP`
- `spec.selector` selects NGINX Pods using `app: nginx` label
- Traffic to port `8080` on Service IP routes to port `80` on Pods

Apply and test:

```bash
kubectl apply -f clusterip-service.yaml
kubectl get services

# Test from inside a Pod
kubectl exec deployment/nginx -- curl 10.109.128.34:8080

# Using DNS name
# nginx-clusterip.default.svc.cluster.local:8080
```

---

### Step 3: Create a NodePort Service

Create `nodeport-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      nodePort: 32000
```

Apply and test:

```bash
kubectl apply -f nodeport-service.yaml

# Find Node IP address
kubectl get nodes -o wide

# Access the service
curl 192.168.49.2:32000
```

---

### Step 4: Create a LoadBalancer Service

Create `loadbalancer-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 8080
      targetPort: 80
```

Apply and test:

```bash
kubectl apply -f lb.yaml

# View assigned external IP
kubectl get services -o wide

# Access the service
curl 10.96.153.245:8080
```

---

## Service Type Comparison Table

| Feature | ClusterIP | NodePort | LoadBalancer | ExternalName |
|---------|-----------|----------|--------------|--------------|
| **Access Scope** | Internal only | External | External | Internal to External |
| **IP Assignment** | Virtual IP | Node IP + Port | External IP | DNS CNAME |
| **Cloud Provider Required** | No | No | Yes | No |
| **Cost** | Free | Free | $$ | Free |
| **Load Balancing** | Internal round-robin | None (node-specific) | External LB | None |
| **Port Range** | Any | 30000-32767 | Any | N/A |
| **Use Case** | Internal microservices | Testing/bare-metal | Production cloud | External service access |

---

## Key Concepts: Endpoints

**Endpoints** in Kubernetes represent the actual IP addresses of Pods that a Service routes traffic to.

- Created automatically when a Service has a selector
- Store the list of Pod IPs that match the selector
- Updated automatically as Pods scale or restart
- Can be manually defined for external services

**View Endpoints:**
```bash
kubectl get endpoints
kubectl describe endpoints <service-name>
```

---

## Best Practices Summary

1. **✅ Use ClusterIP** for internal microservice communication
2. **✅ Use LoadBalancer** for production cloud deployments
3. **❌ Avoid NodePort** in production unless necessary
4. **✅ Use Ingress Controllers** as a more flexible alternative to NodePort
5. **✅ Use ExternalName** for integrating with external APIs
6. **✅ Always set resource limits** on your Services
7. **✅ Use network policies** to restrict access to ClusterIP services

---

## Additional Resources

- [Kubernetes Services Documentation](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
