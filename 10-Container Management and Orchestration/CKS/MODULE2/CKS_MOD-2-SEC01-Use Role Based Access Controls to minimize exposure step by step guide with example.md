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


