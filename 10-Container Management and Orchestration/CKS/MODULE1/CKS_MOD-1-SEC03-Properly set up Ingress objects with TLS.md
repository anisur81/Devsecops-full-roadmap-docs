# Step-by-Step Guide to Setting Up TLS-Secured Ingress

## Prerequisites

- A running Kubernetes cluster.
- Administrative access to the cluster.
- A domain name for your application.
- A TLS certificate (can be self-signed or obtained from a Certificate Authority).

---
Lab Architecture

```
Client
   |
 HTTPS (443)
   |
Ingress Controller (NGINX)
   |
Service (ClusterIP)
   |
Nginx Pod
```
---
## Step 1: Obtain a TLS Certificate

###  
```
1. Generate the  Server private key(tls.key)  and Server certificate (tls.crt) signed by Root CA 
Note: This two file generation process has been documented into the separate file
2. Now Create the k8s tls secret using the above two files
```
---

## Step 2: Create a Kubernetes Secret
Store the TLS certificate and key in a Kubernetes secret:

```bash
kubectl create secret tls nginx-tls --cert=tls.crt --key=tls.key
```

- This command creates a secret named `tls-secret` in your default namespace.
- You can specify a different namespace if needed.

---

## Step 3: Install the NGINX Ingress Controller

Apply the official manifest.

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

Check deployments

```
kubectl get deploy -n ingress-nginx
```

Check services

```
kubectl get svc -n ingress-nginx
```

Verify the IngressClass
```
kubectl get ingressclass
```

Verify controller
```
kubectl describe deployment ingress-nginx-controller -n ingress-nginx
```
Check logs
```
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```
Check the controller service
```
kubectl get svc -n ingress-nginx
```
---

## Step 4: Deploy a sample nginx Application

deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: tls-demo
spec:
  replicas: 2
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
Deploy
```
kubectl apply -f deployment.yaml
```
Verify
```
kubectl get pods -n tls-demo
```
## Step 5 Create ClusterIP Service

service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: tls-demo
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```
Apply
```
kubectl apply -f service.yaml
```
Verify
```
kubectl get svc -n tls-demo
```
---
## Step 6: Create an Ingress Resource
Define an Ingress resource that uses the TLS secret:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: tls-demo
spec:
  ingressClassName: nginx

  tls:
  - hosts:
    - devopslab.com
    secretName: nginx-tls

  rules:
  - host: devopslab.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service
            port:
              number: 80

```
---

Apply
```
kubectl apply -f ingress.yaml
```
Verify Ingress
```
kubectl get ingress -n tls-demo
```
Example output
```
NAME             HOSTS
nginx-ingress    devopslab.com
```
Describe
```
kubectl describe ingress nginx-ingress -n tls-demo
 ```
---

Apply the Ingress resource to your cluster:
```
kubectl apply -f ingress.yaml
```

---

## Step 7: Verify the Setup
 
---

Find ingress IP

```
kubectl get ingress -n tls-demo
or
kubectl get svc -n ingress-nginx

Update /etc/hosts on your local machine:

<INGRESS_IP> devopslab.com
```
Test HTTPS
```
curl -k https://devopslab.com
```
Expected
```
Welcome to nginx!
```
Verify TLS Secret

```
kubectl describe secret nginx-tls -n tls-demo
```
Check Ingress TLS configuration
```
kubectl describe ingress nginx-ingress -n tls-demo
```
---

## Step 8 Troubleshooting
```
Check ingress controller

kubectl get pods -n ingress-nginx

View controller logs

kubectl logs -n ingress-nginx deploy/ingress-nginx-controller

Verify endpoints

kubectl get endpoints -n tls-demo

Check services

kubectl get svc -n tls-demo

Check ingress

kubectl describe ingress nginx-ingress -n tls-demo
```

# Understanding TLS Passthrough and SSL Offloading

## What is TLS Passthrough?

TLS Passthrough means:

- Ingress does **NOT** terminate TLS.
- TLS traffic is passed directly to the backend Pod.
- The application (Pod) terminates TLS instead of Ingress.

### What Is SSL Passthrough?
The action of transmitting data to a server via a load balancer without decrypting the same is called **SSL passthrough**.

- Generally, SSL termination (decryption) occurs at the load balancer, and then the data in plain format is transmitted to the web server.
- In SSL passthrough, data stays encrypted when passing through the load balancer. The encrypted data is decrypted directly by the web server.

---

## Comparison: TLS Termination vs. TLS Passthrough

| Feature                        | TLS Termination             | TLS Passthrough                                              |
| ------------------------------ | --------------------------- | ------------------------------------------------------------ |
| Who decrypts traffic?          | Ingress Controller          | Application Pod                                              |
| Backend protocol               | HTTP                        | HTTPS                                                        |
| Use case                       | Most apps (centralized TLS) | Apps that must hold the cert (banking, PCI, strict security) |
| Can use Ingress rules by path? | YES                         | NO (TCP routing only)                                        |
| Can use cert-manager?          | YES                         | NO                                                           |

---

## When Should You Use TLS Passthrough?

Use passthrough when:

- Backend applications must own their certificates.
- Mutual TLS (mTLS) between client and service is required.
- Applications require end-to-end encryption.
- Databases with TLS (Postgres, MongoDB, MySQL).
- Banking/PCI/FIPS workloads.

---

## Limitations of Nginx TLS Passthrough

- No path-based routing.
- No HTTP rules.
- No cert-manager (Ingress never sees the cert).
- No sticky sessions.
- Only SNI hostname-based routing.

---

## What is SSL Offloading?

SSL offloading (or SSL termination) handles HTTPS traffic by offloading the encryption/decryption workload to the load balancer.

- Load balancers decrypt traffic between client and server and re-encrypt responses.
- This offloads the CPU-intensive encryption/decryption from web servers, allowing them to serve content more efficiently.

### Drawbacks of SSL Offloading
- Plain data (unencrypted traffic) crosses the load balancer to backend servers, potentially enabling man-in-the-middle attacks.
- Attackers can penetrate networks and steal data more easily.
- This risk increases when encryption/decryption keys are shared with the load balancer.
- Not recommended for highly sensitive traffic or critical network security.

---

# Enable SSL Passthrough in NGINX Ingress Controller

SSL passthrough is **disabled by default** in NGINX Ingress.

### If Using Helm to Deploy Ingress-Nginx

```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --set controller.extraArgs.enable-ssl-passthrough=""
```

### Restart Ingress Controller

```bash
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
```

### Backend Pod Must Have SSL Certificate

- Example: Backend app listens on port `8443` (HTTPS).

### Create Kubernetes Service (HTTPS Target)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: secure-app
  namespace: prod
spec:
  selector:
    app: secure-app
  ports:
    - name: https
      port: 443
      targetPort: 8443
```

### Create Ingress with SSL Passthrough

**Important:** No TLS secret is used because TLS terminates at the backend.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: secure-app-ingress
  namespace: prod
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
    - host: secure.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: secure-app
                port:
                  number: 443
```

> **Note:** You **do NOT** define a `tls:` block here because Ingress does not terminate TLS.

---

## How SSL Passthrough Works (Flow)

```
Client ---> Ingress Controller ---> Encrypted traffic forwarded ---> Backend Pod
        (TLS NOT terminated here)                                 (TLS terminated here)
```

---

## Validate SSL Passthrough

### Test DNS Routing

```bash
curl -vk https://secure.example.com --resolve secure.example.com:443:<INGRESS_IP>
```

### Check Logs

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller | grep passthrough
```

---

## Why Annotations Are Needed

- `nginx.ingress.kubernetes.io/ssl-passthrough: "true"`  
  Tells NGINX to operate at L4 TCP mode.

- `nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`  
  Ensures the backend service uses TLS.

---

## Limitations of SSL Passthrough

| Feature                    | Supported? |
| -------------------------- | ---------- |
| Path routing               | ❌ No      |
| TLS termination at Ingress | ❌ No      |
| WAF rules                  | ❌ No      |
| SNI-based routing          | ✅ Yes     |
| End-to-end encryption      | ✅ Yes     |
