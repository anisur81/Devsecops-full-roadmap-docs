
# Practical guide and hands-on lab exercise to disable defaults and minimize permissions using Service Account.

The Security Risks

Default Tokens: Every namespace has a default Service Account.
Auto-mounting: Pods automatically mount this token unless explicitly told not to.
Lateral Movement: Attackers exploiting a container can steal this token to move laterally.

### Disable Default Service Account Auto-mounting

 You should prevent the default service account from automatically mounting its API token into pods.
 This can be done at the Service Account level or the Pod level.
 
 #### Option A: Cluster/Namespace Level (Recommended)
 
 Modify the default service account in your namespace to turn off token automounting.
 
 ```
apiVersion: v1
kind: Service Account
metadata:
  name: default
  namespace: secure-apps
automountServiceAccountToken: false
```

#### Option B: Individual Pod Level If you cannot modify the default Service Account, 
disable it directly inside the Pod specification.yaml 
```
apiVersion: v1
kind: Pod
metadata:
  name: web-app
  namespace: secure-apps
spec:
  automountServiceAccountToken: false
  containers:
  - name: nginx
    image: nginx:alpine
	```

 ## Step 1: Lab Practice — Creating a Least-Privilege Service AccountInstead of using the default account, 

create a dedicated Service Account. 

Grant it only the exact permissions your application needs (Least Privilege).

Scenario: You have a Python script running in a pod that only needs to list pods in its own namespace to monitor health. 
It should not be able to create, delete, or modify anything.

1. Create the Dedicated Service Account
```
apiVersion: v1
kind: Service Account
metadata:
  name: pod-viewer-sa
  namespace: secure-apps
```
Ensure it doesn't accidentally inherit or pass tokens unnecessarily

automountServiceAccountToken: true 


## Step 2. Create a Minimal RBAC RoleDefine a Role that restricts actions to list and get on pods only.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: secure-apps
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "list"]
```

## Step 3. Bind the Role to your Service AccountLink the restrictive Role to your newly created Service Account using a RoleBinding.

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: secure-apps
subjects:
- kind: Service Account
  name: pod-viewer-sa
  namespace: secure-apps
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

```

## Step 4. Deploy the Pod using the Secure Service Account

Assign the scoped service account to the pod.
```
apiVersion: v1
kind: Pod
metadata:
  name: internal-tool
  namespace: secure-apps
spec:
  serviceAccountName: pod-viewer-sa # Explicitly using the minimal SA
  containers:
  - name: tool-container
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]

```
## Step 5. Verify Your Lab Setup Once deployed using auth can-i.
 
 ```
 Check if the pod can list pods (Should be YES):
 $ kubectl auth can-i list pods --as=system:serviceaccount:secure-apps:pod-viewer-sa -n secure-apps
 
 Check if the pod can delete pods (Should be NO):
 $ kubectl auth can-i delete pods --as=system:serviceaccount:secure-apps:pod-viewer-sa -n secure-apps
 
 Check if the pod can look at secrets (Should be NO):
 $ bash kubectl auth can-i get secrets --as=system:serviceaccount:secure-apps:pod-viewer-sa -n secure-apps

```
### Production Best Practices
 - **Audit regularly:** Use tools like **kubesec** or **krane** to identify over-privileged users, ServiceAccounts, Roles, and ClusterRoles.

- **Use Bound Service Account Tokens:** Use projected ServiceAccount tokens (Projected Volumes) so that tokens are short-lived, automatically rotated, and expire after a configurable duration.

- **Avoid ClusterRoles:** Use **Role** and **RoleBinding** whenever possible. Create **ClusterRole** and **ClusterRoleBinding** only when an application or user genuinely requires cluster-wide permissions.
 Audit regularly:  Use tools like kubesec or krane to find over-privileged accounts.
 Use Bound Service Account Tokens: Utilize Projected Volumes for tokens so they expire quickly.
 Avoid ClusterRoles: Never use ClusterRoleBinding unless the pod strictly requires cluster-wide access.


