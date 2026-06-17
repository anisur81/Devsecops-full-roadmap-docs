# Kubernetes Ingress Controllers and Ingress Resources for Enterprise Applications

## Learning Objective
Understand how to use Kubernetes Ingress Controllers and Ingress Resources to manage external access to enterprise applications in your cluster, ensuring high availability, security, and scalability.

---

## 📖 Scenario
You need to expose multiple enterprise applications running in your Kubernetes cluster to the outside world using a single entry point, with advanced features like load balancing, SSL termination, and name-based virtual hosting to ensure seamless and secure access.

---

## 📘 Explanation
Ingress Controllers and Ingress Resources in Kubernetes manage external access to services in the cluster. An Ingress Controller handles the traffic routing, while Ingress Resources define the routing rules.

### Key Concepts

#### Ingress Controller
- A daemon that watches the Kubernetes API server for updates to Ingress Resources and configures the ingress load balancer accordingly
- Popular options include NGINX, Traefik, and HAProxy

#### Ingress Resource
- An API object that defines rules for routing external HTTP/S traffic to services within the cluster
- Supports host and path-based routing, SSL termination, and load balancing

---

## 🚀 Setting up an NGINX Ingress Controller

### Method 1: Direct Installation
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.3.0/deploy/static/provider/cloud/deploy.yaml --namespace ingress-nginx --create-namespace
```

### Method 2: Helm Installation (Alternative)
```bash
helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --create-namespace
```

### Verify Installation

**Check Ingress Controller Pod Status:**
```bash
kubectl get pods -A | grep ingress
```

**Detailed Pod Status:**
```bash
kubectl get pods --namespace ingress-nginx
```

**Check Public IP Assignment:**
```bash
kubectl get service ingress-nginx-controller --namespace=ingress-nginx
```

> **Note:** The service type should be `LoadBalancer`. Browsing to this IP address will show the NGINX 404 page until routing rules are configured.

---

## 📦 Deploying Sample Applications

### Application 1: `aks-helloworld-one.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aks-helloworld-one  
spec:
  replicas: 1
  selector:
    matchLabels:
      app: aks-helloworld-one
  template:
    metadata:
      labels:
        app: aks-helloworld-one
    spec:
      containers:
      - name: aks-helloworld-one
        image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
        ports:
        - containerPort: 80
        env:
        - name: TITLE
          value: "Welcome to Azure Kubernetes Service (AKS)"
---
apiVersion: v1
kind: Service
metadata:
  name: aks-helloworld-one  
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: aks-helloworld-one
```

### Application 2: `aks-helloworld-two.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aks-helloworld-two  
spec:
  replicas: 1
  selector:
    matchLabels:
      app: aks-helloworld-two
  template:
    metadata:
      labels:
        app: aks-helloworld-two
    spec:
      containers:
      - name: aks-helloworld-two
        image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
        ports:
        - containerPort: 80
        env:
        - name: TITLE
          value: "AKS Ingress Demo"
---
apiVersion: v1
kind: Service
metadata:
  name: aks-helloworld-two  
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: aks-helloworld-two
```

### Deploy Applications
```bash
kubectl apply -f aks-helloworld-one.yaml --namespace ingress-nginx
kubectl apply -f aks-helloworld-two.yaml --namespace ingress-nginx
```

### Verify Deployment
```bash
kubectl get pods --namespace ingress-nginx
```

---

## 🌐 Accessing the Application

### Create NodePort Service
```bash
kubectl expose pod myapp --port=80 --name myapp-service --type=NodePort
```

### View Service Details
```bash
kubectl get svc | grep myapp-service
```

> **Note:** This exposes the application on a NodePort (e.g., 32017). You can access it locally via `http://localhost:32017`.

### Limitations of NodePort Access
- Not suitable for production environments
- Users access endpoints using cluster IP addresses
- Lacks security features and proper DNS management

---

## 🔗 Adding a Custom Domain

### Configure Local Hosts File
```bash
# Add domain mapping
echo '[ClusterIP] my-app.com' | sudo tee -a /etc/hosts

# Get ClusterIP if needed
kubectl describe svc/myapp-service | grep IP: | awk '{print $2;}'
```

> **Replace `[ClusterIP]` with your actual Cluster IP from the service**

### Update Ingress Configuration
Update your `ingress.yaml` with a host (`my-app.com`) and appropriate annotations.

---

## 🛣️ Setting Up Path-Based Routing

### Ingress Configuration: `ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /hello-world-one(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: aks-helloworld-one
            port:
              number: 80
      - path: /hello-world-two(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: aks-helloworld-two
            port:
              number: 80
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: aks-helloworld-one
            port:
              number: 80
```

### Routing Rules Explained
| Path | Service | Description |
|------|---------|-------------|
| `/hello-world-one(/|$)(.*)` | `aks-helloworld-one` | Routes traffic for the first application |
| `/hello-world-two(/|$)(.*)` | `aks-helloworld-two` | Routes traffic for the second application |
| `/(.*)` | `aks-helloworld-one` | Default route for unspecified paths |

### Create Ingress Resource
```bash
kubectl apply -f ingress.yaml --namespace ingress-nginx
```

---

## 🎯 Best Practices for Enterprise Applications

### Security
- **SSL/TLS Termination**: Configure HTTPS with proper certificates
- **Authentication**: Implement OAuth2 or JWT authentication
- **Rate Limiting**: Protect against DDoS attacks

### High Availability
- **Multiple Replicas**: Deploy ingress controller with multiple replicas
- **Health Checks**: Configure liveness and readiness probes
- **Autoscaling**: Implement HPA for ingress controllers

### Scalability
- **Load Balancing**: Distribute traffic across multiple pods
- **Caching**: Implement caching strategies for static content
- **Monitoring**: Set up comprehensive monitoring and alerting

### Advanced Features
- **Canary Deployments**: Gradual rollout of new versions
- **A/B Testing**: Route traffic to different versions
- **Circuit Breaking**: Prevent cascading failures

---

## 📊 Monitoring and Troubleshooting

### Useful Commands
```bash
# Check ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Describe ingress resource
kubectl describe ingress hello-world-ingress -n ingress-nginx

# Check endpoints
kubectl get endpoints -n ingress-nginx

# Verify service connectivity
kubectl run -it --rm test-pod --image=curlimages/curl -- sh
# Inside the pod: curl http://aks-helloworld-one:80
```

### Common Issues and Solutions
| Issue | Solution |
|-------|----------|
| 404 Not Found | Check path rules and service endpoints |
| SSL Errors | Verify certificate configuration |
| Timeout Errors | Adjust timeout settings in annotations |
| Connection Refused | Verify service selectors and ports |

---

## 🔒 Production Readiness Checklist

- [ ] Configure SSL/TLS certificates (use cert-manager for automation)
- [ ] Implement proper logging and monitoring
- [ ] Set up resource limits and requests
- [ ] Configure health checks for ingress controller
- [ ] Implement network policies
- [ ] Set up backup and disaster recovery
- [ ] Document routing rules and architecture
- [ ] Create CI/CD pipeline for configuration updates

---

## 📚 Additional Resources

- [Kubernetes Ingress Documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [NGINX Ingress Controller Documentation](https://kubernetes.github.io/ingress-nginx/)
- [Traefik Ingress Controller](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)
- [Cert-Manager for SSL Automation](https://cert-manager.io/)

---

*This guide provides a comprehensive overview of implementing Kubernetes Ingress for enterprise applications, ensuring secure, scalable, and highly available external access to your services.*
