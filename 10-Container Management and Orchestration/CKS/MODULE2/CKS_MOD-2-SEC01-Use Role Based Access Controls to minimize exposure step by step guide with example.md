# Use Role Based Access Controls to minimize exposure step by step guide with example

## Minimize Exposure using Service Account

### 1. Check whether RBAC is enabled
```
$  kubectl api-versions | grep rbac

rbac.authorization.k8s.io/v1
```

Use Role Based Access Controls to minimize exposure step by step guide with example

### 2. Create a Service Account

You’ll bind the Role you create to this Service Account:
```
$ kubectl create serviceaccount demo-user

serviceaccount/demo-user created
```

### 3. Next, run the following command to create an authorization token for your Service Account:
```
$ TOKEN=$(kubectl create token demo-user)
```
The token’s value will now be saved to the $TOKEN environment variable in your terminal.

Configure kubectl with your Service Account

### 4. Now add a new kubectl context that lets you authenticate as your Service Account. 

First, add your Service Account as a credential in your Kubeconfig file:
```
$  kubectl config set-credentials demo-user --token=$TOKEN
User "demo-user" set.
```

The token’s value will now be saved to the $TOKEN environment variable in your terminal.

Next, add your new context—we’re calling it demo-user-context. 
```
$ kubectl config set-context demo-user-context --cluster=kubernetes --user=demo-user

Context "demo-user-context" created.
```
Before switching to your new context, first check the name of your current context so you can easily 
switch back to your administrative account in the next section:
```
$ kubectl config current-context
kubernetes-admin@kubernetes
```
### 5. Now, switch over to your new context that authenticates as your service account:
```
$ kubectl config use-context demo-user-context

Switched to context "demo-user-context".
```
Try to list the Pods in the namespace:
```
$ kubectl get pods

Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:default:demo-user" cannot list resource "pods" in API group "" in the namespace "default"
```

Before continuing, switch back to your original Kubectl context to restore your administrator privileges. 
This will allow you to create your Role and RoleBinding objects in the next section.
```
$ kubectl config use-context kubernetes-admin@kubernetes
Switched to context "kubernetes-admin@kubernetes".
```
### 6. Create a Role
``` 
$ vi role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: demo-role
  namespace: default
rules:
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
      - list
      - create
      - update
```
 Create the role by applying the file
```
$ kubectl apply -f role.yaml
role.rbac.authorization.k8s.io/demo-role created
```
### 7. Create a RoleBinding

The Role has been created but it’s not yet assigned to your Service Account. 
A RoleBinding is required to make this connection.

 ```
$ vi rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: demo-role-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: demo-role
subjects:
  - namespace: default
    kind: ServiceAccount
    name: demo-user
	
 
$ kubectl apply -f rolebinding.yaml
rolebinding.rbac.authorization.k8s.io/demo-role-binding created
```

### 8. Verify your service account has been granted the Role’s Permissions

Switch back to the Kubectl context that authenticates as the Service Account user:
```
$ kubectl config use-context demo-user-context
Switched to context "demo-user-context".
```
Verify that the get pods command now runs successfully:
```
$ kubectl get pods
NAME       READY   STATUS    RESTARTS       AGE
netshoot   1/1     Running   23 (11m ago)   23h
 ```
You can also try creating a Pod as your Service Account:
```
$ kubectl run nginx --image=nginx:latest
pod/nginx created

```
The Pod is successfully created:
Now the check the newly created ones.

```
$ kubectl get pods

NAME    READY   STATUS    RESTARTS   AGE
netshoot   1/1     Running   23 (11m ago)   23h
nginx   1/1     Running   0          15s
```

 
However, it’s not possible for the Service Account user to delete Pods because the Role you’ve assigned doesn’t include the required delete action verb:
 
```
$ kubectl delete pod nginx
Error from server (Forbidden): pods "nginx" is forbidden: User "system:serviceaccount:default:demo-user" cannot delete resource "pods" in API group "" in the namespace "default"
```

## 9 Clean the demo-user from the k8s cluster
```
oracle@dockertest01:~/RBAC$ kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          demo-user-context             kubernetes   demo-user
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin

oracle@dockertest01:~/RBAC$ kubectl config unset users.demo-user
Property "users.demo-user" unset.

oracle@dockertest01:~/RBAC$ kubectl config delete-context demo-user-context
deleted context demo-user-context from /home/oracle/.kube/config

oracle@dockertest01:~/RBAC$ kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          demo-user-context             kubernetes   demo-user
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
```

## Minimize Exposure using Creating a new user in Kubernetes Cluster

### Step-1: Create the required Directory and Private key
```
$ mkdir userkey
$ cd userkey
```
Generate the user's private key
```
$ sudo openssl genrsa -out demo-user.key 2048
```
Lets now create a Certification Signing Request (CSR) for each of the users. When you generate the csr make sure you also provide

CN: This will be set as username
O: Org name. This is actually used as a group by kubernetes while authenticating/authorizing users. 
You could add as many as you need

### Step-2: Create a Certificate Signing Request (CSR) for demo-user and sign the csr for generating the certificate
```
$ sudo openssl req -new -key demo-user.key -out demo-user.csr -subj "/CN=demo-user"
```
In order to be deemed authentic, these CSRs need to be signed by the Certification Authority (CA) which in this case is Kubernetes Master. 

You need access to the folllwing files on kubernetes master.

Certificate : ca.crt (kubeadm)  
Pricate Key : ca.key (kubeadm)  

You would typically find it the following paths
```
$ ls -lrt /etc/kubernetes/pki
-rw------- 1 root root 1675 Jul 12 10:20 ca.key
-rw-r--r-- 1 root root 1107 Jul 12 10:20 ca.crt
-rw------- 1 root root 1675 Jul 12 10:20 apiserver.key
-rw-r--r-- 1 root root 1289 Jul 12 10:20 apiserver.crt
-rw------- 1 root root 1675 Jul 12 10:20 apiserver-kubelet-client.key
-rw-r--r-- 1 root root 1131 Jul 12 10:20 apiserver-kubelet-client.crt
-rw------- 1 root root 1675 Jul 12 10:20 front-proxy-ca.key
-rw-r--r-- 1 root root 1123 Jul 12 10:20 front-proxy-ca.crt
-rw------- 1 root root 1675 Jul 12 10:20 front-proxy-client.key
-rw-r--r-- 1 root root 1119 Jul 12 10:20 front-proxy-client.crt
drwxr-xr-x 2 root root 4096 Jul 12 10:20 etcd
-rw------- 1 root root 1675 Jul 12 10:20 apiserver-etcd-client.key
-rw-r--r-- 1 root root 1123 Jul 12 10:20 apiserver-etcd-client.crt
-rw------- 1 root root 1675 Jul 12 10:20 sa.key
-rw------- 1 root root  451 Jul 12 10:20 sa.pub

```

To verify which one is your cert and which one is key, use the following command,
```
$ file /etc/kubernetes/pki/ca.crt
ca.pem: PEM certificate

$ sudo file /etc/kubernetes/pki/ca.key
ca-key.pem: PEM RSA private key
```

Move the demo-user.csr and demo-user.key file into the /etc/kubernetes/pki location

```
$ mv demo-user.csr demo-user.key /etc/kubernetes/pki 
```

So all the files are in the same directory, sign the CSR as,
```
$ sudo openssl x509 -req -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key -CAcreateserial -days 2000 -in demo-user.csr -out demo-user.crt
$ Certificate request self-signature ok
subject=CN=demo-user
$ ls
demo-user.crt  demo-user.csr  demo-user.key 
```

Setting up User configs with kubectl
In order to configure the users that you created above, following steps need to be performed with kubectl
Add credentials in the configurations
Set context to login as a user to a cluster
Switch context in order to assume the user's identity while working with the cluster to add credentials,

```
$ kubectl config set-credentials demo-user --client-certificate=/etc/kubernetes/pki/demo-user.crt --client-key=/etc/kubernetes/pki/demo-user.key --embed-certs=true
```
And proceed to set/create contexts (user@cluster).
If you are not sure whats the cluster name, use the following command to find,
``` 
$ kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
```

where, kubernetes is the cluster name.
To set context for kubernetes cluster for the demo-user 
```
$ kubectl config set-context demo-user-kubernetes --cluster=kubernetes --namespace=default --user=demo-user
```
Now, switch over to demo-user-kubernetes  context that authenticates as demo-user
```
$ kubectl config use-context demo-user-kubernetes
$ kubectl get pods 

$ kubectl get pods --as demo-user (without switching the context)
```

ClusterRoleBinding example
To grant permissions across a whole cluster, you can use a ClusterRoleBinding. 
The following ClusterRoleBinding allows any user in the user "demo-user" to read nodes

Create the cluster role 
```
$ vi clusterrole.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
--"namespace" omitted since ClusterRoles are not namespaced
  name: pod-lister
rules:
- apiGroups: [""]
  #
  # objects is "nodes"
  resources: ["nodes"]
  verbs: ["get", "watch", "list"]
```
Create the clusterrole binding

```
$ vi clusterrolebinding.yaml
---This cluster role binding allows anyone in the "demo-user" user to read nodes

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pod-lister-binding
subjects:
- kind: User
  name: demo-user # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pod-lister
  apiGroup: rbac.authorization.k8s.io
```

Apply the ClusterRole using:
```
$ kubectl apply -f  clusterrole.yaml
```
Apply the ClusterRoleBinding using:
```
$ kubectl apply -f  clusterrolebinding.yaml

oracle@dockertest01:~/RBAC$ kubectl get nodes --as demo-user
NAME           STATUS   ROLES           AGE     VERSION
dockertest01   Ready    control-plane   3d21h   v1.35.6
dockertest02   Ready    <none>          3d21h   v1.35.6
dockertest03   Ready    <none>          3d21h   v1.35.6
```

Another example for clusterrole and clusterrolebinding
```
$ vi clusterrolepodread.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: podsview
rules:
- apiGroups: [""]
  #
  #
  # objects is "pods"
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
  
  ```
  Create the cluster role binding for pod read
  ```
$ vi clusterrolebindingpodread.yaml

apiVersion: rbac.authorization.k8s.io/v1
# This cluster role binding allows anyone in the "demo-user" user to read secrets in any namespace.
kind: ClusterRoleBinding
metadata:
  name: podsview-binding
subjects:
- kind: User
  name: demo-user # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: podsview
  apiGroup: rbac.authorization.k8s.io
  ```

  Apply the ClusterRole using:
```
 $ kubectl apply -f  clusterrolepodread.yaml
 ```
 Apply the ClusterRolebinding using:

```
 $ kubectl apply -f  clusterrolebindingpodread.yaml
 
 Check the permitted operation
 
$ kubectl get pods -A --as demo-user

oracle@dockertest01:~/RBAC$ kubectl get pods  --as demo-user
NAME       READY   STATUS    RESTARTS       AGE
netshoot   1/1     Running   26 (15m ago)   26h
nginx      1/1     Running   0              3h3m
```
