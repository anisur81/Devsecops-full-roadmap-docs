# Step-by-Step Guide to Setting Up TLS-Secured Ingress

## Prerequisites

- A running Kubernetes cluster.
- Administrative access to the cluster.
- A domain name for your application.
- A TLS certificate (can be self-signed or obtained from a Certificate Authority).

---

## Step 1: Obtain a TLS Certificate

### Install Cert-Manager (Automated)
Cert-Manager is a Kubernetes tool that automates the management and issuance of TLS certificates.

```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.5.3/cert-manager.yaml
```

### Option B: Creating a Self-Signed Certificate
For testing purposes, you can create a self-signed certificate using OpenSSL:

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=yourdomain.com/O=yourdomain.com"
```

---

## Step 2: Create a Kubernetes Secret
Store the TLS certificate and key in a Kubernetes secret:

```bash
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```

- This command creates a secret named `tls-secret` in your default namespace.
- You can specify a different namespace if needed.

---

## Step 3: Create an Ingress Resource
Define an Ingress resource that uses the TLS secret:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - yourdomain.com
    secretName: tls-secret
  rules:
  - host: yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

> **Note:** Replace `yourdomain.com` with your actual domain name and `my-service` with the name of the Kubernetes service you want to expose.

---

## Step 4: Apply the Ingress Resource
Apply the Ingress resource to your cluster:

```bash
kubectl apply -f ingress.yaml
```

---

## Step 5: Verify the Setup
Ensure that your Ingress controller is correctly configured and supports TLS.

- You can verify the setup by accessing your application via the domain name in a web browser.
- The connection should be secured with HTTPS.

---

## Step 6: Monitoring and Maintaining TLS
- Regularly monitor your certificates’ expiration dates and renew them as needed.
- Cert-Manager can automate this process for certificates issued by Let’s Encrypt.

---

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
