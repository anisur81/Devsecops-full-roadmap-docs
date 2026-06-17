# Kubernetes Taints 🔒 & Tolerations 🔑

## Managing Pod Scheduling in a Kubernetes Cluster

In this example, the nginx pod has a toleration that matches the taint with `nginx-node=true` and `NoSchedule` effect, allowing it to be scheduled on nodes with that taint.

### Adding a Taint to a Node

Let's first add a taint to a node using the below command:

```bash
kubectl taint nodes worker03 nginx-node=true:NoSchedule
```

With the above command, the node is tainted with `nginx-node=true:NoSchedule`. Now let's create an nginx pod without any tolerations and see if it schedules on any node.

#### Create Pod Without Tolerations

```yaml
# vi nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
```

When you try to create this pod in your Kubernetes cluster, it won't be placed on the target node and will remain in a pending state since we have added the taint to the worker node.

```bash
kubectl apply -f nginx-pod.yaml
```

### Adding Tolerations to the Pod

Now modify the nginx pod manifest by adding tolerations as shown below:

```yaml
# vi nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "nginx-node"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

Apply the above manifest file and see if it schedules on the target node:

```bash
kubectl apply -f nginx-pod.yaml
# Output: pod/nginx created
```

Now the pod is scheduled on the target worker node.

---

## Tainting Nodes with NoSchedule and NoExecute Effects

Now, let's taint `node1` with the `NoSchedule` effect:

```bash
kubectl taint nodes node1.compute.infracloud.io thisnode=HatesPods:NoSchedule
```

You should be able to see that `node1` is now tainted:

```
node "node1.compute.infracloud.io" tainted
```

Let's run the deployment to see where pods are deployed.

### Create Deployment Without Tolerations

```yaml
# vi deployment.yaml
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

```bash
kubectl create -f deployment.yaml
```

Check the output using:

```bash
kubectl get pods -o wide
```

### Tainting Node3 with NoExecute Effect

Now let's taint `node3` with the `NoExecute` effect, which will evict both the pods from `node3` and schedule them on `node2`:

```bash
kubectl taint nodes node3 thisnode=AlsoHatesPods:NoExecute
```

In a few seconds, you'll see that the pods are terminated on `node3` and spawned on `node2`:

```bash
kubectl get pods -o wide
```

### Create Deployment With Tolerations

```yaml
# vi deployment-toleration.yaml
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 10
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
      tolerations:
      - effect: NoSchedule
        operator: Exists
```

```bash
kubectl create -f taints/deployment-toleration.yaml
```

Check the output by running:

```bash
kubectl get pods -o wide
```

You should see that some pods are scheduled on `node1` and some on `node2`. However, no pod is scheduled on `node3`. This is because in the new deployment spec, we are tolerating the `NoSchedule` effect. `node3` is tainted with `NoExecute` which we have not tolerated, so no pods will be scheduled there.

### Removing Taints

To finish off, let's remove the taints from the nodes:

```bash
kubectl taint nodes node3.compute.infracloud.io thisnode:NoExecute-
kubectl taint nodes node1.compute.infracloud.io thisnode:NoSchedule-
```

---

## Node Selector

Node selector is a feature in Kubernetes that allows users to specify which nodes a pod should be scheduled on based on node labels.

### Adding Labels to a Node

```bash
kubectl label nodes worker02 nodelbl=wrk2
```

Node selector is similar to node affinity in that both allow users to specify which nodes a pod should be scheduled on based on node labels. However, node selector is a simpler and more primitive mechanism compared to node affinity, which provides more fine-grained control over pod scheduling.

### Create a Pod That Gets Scheduled to a Specific Node

You can also schedule a pod to one specific node via setting `nodeName`.

```yaml
# vi pod-nginx-specific-node.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: worker03  # schedule pod to specific node
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

---

## Pod Affinity & AntiAffinity

Pod affinity can be used for advanced scheduling in Kubernetes.

### Overview

Node affinity allows you to schedule a pod on a set of nodes based on labels present on the nodes. However, in certain scenarios, we might want to schedule certain pods together or we might want to make sure that certain pods are never scheduled together. This can be achieved by `PodAffinity` and/or `PodAntiAffinity` respectively.

### Variants

Similar to node affinity, there are a couple of variants in pod affinity:
- `requiredDuringSchedulingIgnoredDuringExecution`
- `preferredDuringSchedulingIgnoredDuringExecution`

### Use Cases

- While scheduling workload, when we need to schedule a certain set of pods together, `PodAffinity` makes sense (e.g., a web server and a cache).
- While scheduling workload, when we need to make sure that a certain set of pods are not scheduled together, `PodAntiAffinity` makes sense.

---

### Pod Affinity Example

Let's begin with listing nodes:

```bash
kubectl get nodes
```

```yaml
# vi deployment-pod-Affinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3
  template: 
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

Here we are specifying that all nginx pods should be scheduled together.

```bash
kubectl apply -f deployment-pod-Affinity.yaml
```

Check the pods using:

```bash
kubectl get pods -o wide
```

To clean up:

```bash
kubectl delete -f deployment-pod-Affinity.yaml
```

---

### Pod AntiAffinity Example

```yaml
# vi deployment-pod-AntiAffinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template: 
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

Here we are specifying that no two nginx pods should be scheduled together.

```bash
kubectl apply -f deployment-pod-AntiAffinity.yaml
```

Check the pods using:

```bash
kubectl get pods -o wide
```

**Note:** In the above example, if the number of replicas is more than the number of nodes, some pods will remain in a pending state.

To clean up:

```bash
kubectl delete -f deployment-AntiAffinity.yaml
```

---

### Pod Affinity and AntiAffinity Combined Example

```yaml
# vi pod-with-pod-affinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: topology.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: registry.k8s.io/pause:2.0
```

```bash
kubectl apply -f pod-with-pod-affinity.yaml
```

Possible error output:
```
Warning  FailedScheduling  18s   default-scheduler  0/4 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 1 node(s) were unschedulable, 2 node(s) didn't match pod affinity rules. preemption: 0/4 nodes are available: 4 Preemption is not helpful for scheduling.
```

This example defines one Pod affinity rule and one Pod anti-affinity rule. The Pod affinity rule uses the "hard" `requiredDuringSchedulingIgnoredDuringExecution`, while the anti-affinity rule uses the "soft" `preferredDuringSchedulingIgnoredDuringExecution`.

---

## Node Affinity

Node affinity is a way to set rules based on which the scheduler can select the nodes for scheduling workload. Node affinity can be thought of as the opposite of taints. Taints repel a certain set of nodes whereas node affinity attracts a certain set of nodes.

### Use Cases

While scheduling workload, when we need to schedule a certain set of pods on a certain set of nodes but do not want those nodes to reject everything else, using node affinity makes sense.

### Node Affinity Example

Let's begin with listing nodes:

```bash
kubectl get nodes
```

NodeAffinity works on label matching. Let's label `node1`:

```bash
kubectl label nodes worker02 nodelbl=wrk2
```

```yaml
# vi node-affinity.yaml
apiVersion: apps/v1  
kind: Deployment
metadata:
  name: redis-master
spec:
  selector:
    matchLabels:
      app: redis
  replicas: 1
  template:
    metadata:
      labels:
        app: redis        
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: "nodelbl"
                operator: In
                values: ["wrk2"]
      containers:
      - name: master
        image: redis
```

The pod should be created on node `worker02`.

---

## nodeName

`nodeName` is a more direct form of node selection than affinity or `nodeSelector`. `nodeName` is a field in the Pod spec.

### Example of a Pod Spec Using nodeName

```yaml
# vi pod-with-node-name.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: worker02
```

```bash
kubectl apply -f pod-with-node-name.yaml
```

The above Pod will only run on the node `worker02`.

---

## Implement Node Anti-Affinity

To distribute web server pods across different nodes, modify the deployment to include a node anti-affinity rule:

```yaml
# vi node-antiaffinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webserver
  template:
    metadata:
      labels:
        app: webserver
    spec:
      containers:
      - name: nginx
        image: nginx
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - webserver
            topologyKey: "kubernetes.io/hostname"
```

In this configuration, the `podAntiAffinity` rule prevents the scheduler from placing two pods with the label `app: webserver` on the same node, ensuring that your web servers are spread across different nodes.

```bash
kubectl apply -f node-antiaffinity.yaml
```

### Verify Pod Distribution

After deploying, verify that the pods are distributed across different nodes:

```bash
kubectl get pods -o wide -l app=webserver
```

### Clean Up

```bash
kubectl delete -f node-antiaffinity.yaml
```
