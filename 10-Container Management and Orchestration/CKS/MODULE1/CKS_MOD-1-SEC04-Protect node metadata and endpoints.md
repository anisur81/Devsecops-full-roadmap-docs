# Protecting Node Metadata and Endpoints

## Overview

Protecting node metadata and API endpoints helps safeguard your cluster from unauthorized access. In our example, the following endpoint must be protected:

```
http://169.254.169.254/latest/meta-data/
```

**Action:** Implement IAM policies and network security groups to restrict access to this metadata.

---

## Creating Networking Policies To Allow/Deny Access to the Endpoint

Ensure that your node metadata API is exposed only to the pods that have the `security: metadata-access` label.

### Step 1: Deny Egress Access to All Pods

First, we deny egress access to all pods to that IP address on port 80:

**metadata-deny.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-metadata-access
  namespace: default
spec:
  podSelector: {}  # Selects ALL pods
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32
    ports:
    - protocol: TCP
      port: 80
```

### Step 2: Allow Access to Specific Pods

Now, we allow only the pods on default namespace that match the label `security: metadata-access` to access that address on port 80:

**metadata-allow.yaml**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-metadata-access
  namespace: default
spec:
  podSelector:
    matchLabels:
      security: metadata-access
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 169.254.169.254/32
    ports:
    - protocol: TCP
      port: 80
```

---

## Set Kubelet Parameters Via A Configuration File

A subset of the kubelet's configuration parameters may be set via an on-disk config file, as a substitute for command-line flags. Providing parameters via a config file is the recommended approach because it simplifies node deployment and configuration management.

### Create the Config File

The subset of the kubelet's configuration that can be configured via a file is defined by the `KubeletConfiguration` struct. The configuration file must be a JSON or YAML representation of the parameters in this struct. Make sure the kubelet has read permissions on the file.

Here is an example of what this file might look like:

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: "192.168.0.8"
port: 20250
serializeImagePulls: false
evictionHard:
    memory.available:  "100Mi"
    nodefs.available:  "10%"
    nodefs.inodesFree: "5%"
    imagefs.available: "15%"
    imagefs.inodesFree: "5%"
```

### Configuration Settings Explained

| Parameter | Value | Description |
|-----------|-------|-------------|
| `address` | `192.168.0.8` | The kubelet will serve on IP address 192.168.0.8 |
| `port` | `20250` | The kubelet will serve on port 20250 |
| `serializeImagePulls` | `false` | Image pulls will be done in parallel |
| `evictionHard` | See below | The kubelet will evict Pods under specific conditions |

### Eviction Conditions

The kubelet will evict Pods under one of the following conditions:

- When the node's available memory drops below `100MiB`
- When the node's main filesystem's available space is less than `10%`
- When the image filesystem's available space is less than `15%`
- When more than `95%` of the node's main filesystem's inodes are in use

---

## Minimize Use of, and Access to, GUI Elements

### Kubernetes Dashboard

The Kubernetes Dashboard is a web-based user interface that provides a graphical representation of your Kubernetes cluster, allowing users to:

- View and manage various resources of Kubernetes
- Monitor cluster performance
- Access all from a graphical user interface (GUI)

⚠️ **Important Security Considerations:**

- While the Kubernetes Dashboard can be helpful, it has both advantages and disadvantages
- Be careful about the privileges or permissions you provide to the Kubernetes Dashboard
- **It is recommended to NOT expose the Kubernetes Dashboard on the internet**

### Installing the Dashboard

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

### View Dashboard Resources

```bash
kubectl get ns
kubectl get all -n kubernetes-dashboard
kubectl get sa -n kubernetes-dashboard
kubectl get secrets -n kubernetes-dashboard
```

---

## Creating a Service Account

We are creating a Service Account with the name `admin-user` in namespace `kubernetes-dashboard`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

---

## Creating a ClusterRoleBinding

In most cases after provisioning the cluster using kops, kubeadm or any other popular tool, the ClusterRole `cluster-admin` already exists in the cluster. We can use it and create only a ClusterRoleBinding for our ServiceAccount. If it does not exist, then you need to create this role first and grant required privileges manually.

---

## Getting a Bearer Token for ServiceAccount

### Short-lived Token

Now we need to find the token we can use to log in. Execute the following command:

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

It should print something like:

```
eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXY1N253Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIwMzAzMjQzYy00MDQwLTRhNTgtOGE0Ny04NDllZTliYTc5YzEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.Z2JrQlitASVwWbc-s6deLRFVk5DWD3P_vjUFXsqVSY10pbjFLG4njoZwh8p3tLxnX_VBsr7_6bwxhWSYChp9hwxznemD5x5HLtjb16kI9Z7yFWLtohzkTwuFbqmQaMoget_nYcQBUC5fDmBHRfFvNKePh_vSSb2h_aYXa8GV5AcfPQpY7r461itme1EXHQJqv-SN-zUnguDguCTjD80pFZ_CmnSE1z9QdMHPB8hoB4V68gtswR1VLa6mSYdgPwCHauuOobojALSaMc3RH7MmFUumAgguhqAkX3Omqd3rJbYOMRuMjhANqd08piDC3aIabINX6gP5-Tuuw2svnV6NYQ
```

---

## Getting a Long-lived Bearer Token for ServiceAccount

We can also create a token with the secret which binds the service account and the token will be saved in the Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: "admin-user"   
type: kubernetes.io/service-account-token
```

After the Secret is created, execute the following command to get the token saved in the Secret:

```bash
kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath="{.data.token}" | base64 -d
```

---

## Accessing Dashboard

1. Copy the token from either step above
2. Navigate to the dashboard URL
3. Paste it into the **Enter token** field on the login screen
4. Click the **Sign in** button

That's it! You are now logged in as an admin.

---

## Security Best Practices Summary

| Area | Recommendation |
|------|---------------|
| **Node Metadata** | Restrict access using Network Policies and IAM policies |
| **Kubelet Configuration** | Use config files instead of command-line flags for easier management |
| **Dashboard Access** | Do not expose the Kubernetes Dashboard to the internet |
| **Service Accounts** | Use least privilege principle when creating roles and bindings |
| **Network Policies** | Default-deny approach, then explicitly allow required access |
