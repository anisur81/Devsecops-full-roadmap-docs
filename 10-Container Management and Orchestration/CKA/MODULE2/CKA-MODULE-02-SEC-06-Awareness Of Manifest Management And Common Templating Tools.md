# Awareness of Manifest Management and Common Templating Tools

## Declarative Management of Kubernetes Objects Using Kustomize

Kustomize is a standalone tool to customize Kubernetes objects through a kustomization file.

### ConfigMap Generation

To use a generated ConfigMap in a Deployment, reference it by the name of the configMapGenerator. Kustomize will automatically replace this name with the generated name.

**Example: Creating a ConfigMap from a file**

```bash
# Create a application.properties file
cat <<EOF >application.properties
FOO=Bar
EOF

cat <<EOF >deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: example-configmap-1
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
configMapGenerator:
- name: example-configmap-1
  files:
  - application.properties
EOF
```

**Generate the ConfigMap and Deployment:**

```bash
kubectl kustomize ./
```

**Generated Output:**

```yaml
apiVersion: v1
data:
  application.properties: |
    FOO=Bar    
kind: ConfigMap
metadata:
  name: example-configmap-1-g4hk9g2ff8
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: my-app
  name: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - image: my-app
        name: app
        volumeMounts:
        - mountPath: /config
          name: config
      volumes:
      - configMap:
          name: example-configmap-1-g4hk9g2ff8
        name: config
```

### Secret Generation

You can generate Secrets from files or literal key-value pairs.

**Generating a Secret from a File:**

```bash
# Create a password.txt file
cat <<EOF >./password.txt
username=admin
password=secret
EOF

cat <<EOF >./kustomization.yaml
secretGenerator:
- name: example-secret-1
  files:
  - password.txt
EOF
```

**Generated Secret:**

```yaml
apiVersion: v1
data:
  password.txt: dXNlcm5hbWU9YWRtaW4KcGFzc3dvcmQ9c2VjcmV0Cg==
kind: Secret
metadata:
  name: example-secret-1-t2kt65hgtb
type: Opaque
```

**Generating a Secret from Literal Values:**

```bash
cat <<EOF >./kustomization.yaml
secretGenerator:
- name: example-secret-2
  literals:
  - username=admin
  - password=secret
EOF
```

**Generated Secret:**

```yaml
apiVersion: v1
data:
  password: c2VjcmV0
  username: YWRtaW4=
kind: Secret
metadata:
  name: example-secret-2-t52t6g96d8
type: Opaque
```

**Using Secrets in Deployments:**

```bash
# Create a password.txt file
cat <<EOF >./password.txt
username=admin
password=secret
EOF

cat <<EOF >deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: my-app
        volumeMounts:
        - name: password
          mountPath: /secrets
      volumes:
      - name: password
        secret:
          secretName: example-secret-1
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
secretGenerator:
- name: example-secret-1
  files:
  - password.txt
EOF
```

### Generator Options

The generated ConfigMaps and Secrets have a content hash suffix appended. This ensures that a new ConfigMap or Secret is generated when the contents are changed. To disable the behavior of appending a suffix, one can use generatorOptions.

```bash
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: example-configmap-3
  literals:
  - FOO=Bar
generatorOptions:
  disableNameSuffixHash: true
  labels:
    type: generated
  annotations:
    note: generated
EOF
```

**Generated Output:**

```yaml
apiVersion: v1
data:
  FOO: Bar
kind: ConfigMap
metadata:
  annotations:
    note: generated
  labels:
    type: generated
  name: example-configmap-3
```

### Setting Cross-Cutting Fields

It is quite common to set cross-cutting fields for all Kubernetes resources in a project:

- Setting the same namespace for all Resources
- Adding the same name prefix or suffix
- Adding the same set of labels
- Adding the same set of annotations

**Example:**

```bash
# Create a deployment.yaml
cat <<EOF >./deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
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
EOF

cat <<EOF >./kustomization.yaml
namespace: my-namespace
namePrefix: dev-
nameSuffix: "-001"
commonLabels:
  app: bingo
commonAnnotations:
  oncallPager: 800-555-1212
resources:
- deployment.yaml
EOF
```

**Generated Output:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    oncallPager: 800-555-1212
  labels:
    app: bingo
  name: dev-nginx-deployment-001
  namespace: my-namespace
spec:
  selector:
    matchLabels:
      app: bingo
  template:
    metadata:
      annotations:
        oncallPager: 800-555-1212
      labels:
        app: bingo
    spec:
      containers:
      - image: nginx
        name: nginx
```

### Composing Resources

Here is an example of an NGINX application comprised of a Deployment and a Service:

```bash
# Create a deployment.yaml file
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Create a service.yaml file
cat <<EOF > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF

# Create a kustomization.yaml composing them
cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
- service.yaml
EOF
```

### Customizing with Patches

Create one patch for increasing the deployment replica number and another patch for setting the memory limit:

```bash
# Create a deployment.yaml file
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Create a patch increase_replicas.yaml
cat <<EOF > increase_replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
EOF

# Create another patch set_memory.yaml
cat <<EOF > set_memory.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  template:
    spec:
      containers:
      - name: my-nginx
        resources:
          limits:
            memory: 512Mi
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
patchesStrategicMerge:
- increase_replicas.yaml
- set_memory.yaml
EOF
```

**Generated Output:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: nginx
        name: my-nginx
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: 512Mi
```

### Customizing Container Images

In addition to patches, Kustomize also offers customizing container images without creating patches:

```bash
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
images:
- name: nginx
  newName: my.image.registry/nginx
  newTag: 1.4.0
EOF
```

**Generated Output:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: my.image.registry/nginx:1.4.0
        name: my-nginx
        ports:
        - containerPort: 80
```

### Using Vars to Inject Configuration

Sometimes, the application running in a Pod may need to use configuration values from other objects. Kustomize can inject the Service name into containers through vars:

```bash
# Create a deployment.yaml file (quoting the here doc delimiter)
cat <<'EOF' > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        command: ["start", "--host", "$(MY_SERVICE_NAME)"]
EOF

# Create a service.yaml file
cat <<EOF > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF

cat <<EOF >./kustomization.yaml
namePrefix: dev-
nameSuffix: "-001"

resources:
- deployment.yaml
- service.yaml

vars:
- name: MY_SERVICE_NAME
  objref:
    kind: Service
    name: my-nginx
    apiVersion: v1
EOF
```

**Generated Output:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-my-nginx-001
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - command:
        - start
        - --host
        - dev-my-nginx-001
        image: nginx
        name: my-nginx
```

### Bases and Overlays

Kustomize has the concepts of bases and overlays. A base is a directory with a kustomization.yaml, which contains a set of resources and associated customization. An overlay is a directory with a kustomization.yaml that refers to other kustomization directories as its bases.

**Example Base:**

```bash
# Create a directory to hold the base
mkdir base

# Create a base/deployment.yaml
cat <<EOF > base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
EOF

# Create a base/service.yaml file
cat <<EOF > base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF

# Create a base/kustomization.yaml
cat <<EOF > base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
EOF
```

**Overlays Using the Base:**

```bash
mkdir dev
cat <<EOF > dev/kustomization.yaml
resources:
- ../base
namePrefix: dev-
EOF

mkdir prod
cat <<EOF > prod/kustomization.yaml
resources:
- ../base
namePrefix: prod-
EOF
```

---

## Templating YAML with Code

### Using Templates with Search and Replace

A better strategy is to have a placeholder and replace it with the real value before the YAML is submitted to the cluster.

**Example with Sed:**

```yaml
# vi yaml-templating-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: test-container
    image: registry.k8s.io/busybox
    env:
    - name: ENV
      value: %ENV_NAME%
```

**Replacing the value:**

```bash
sed s/%ENV_NAME%/$production/g pod_template.yaml > pod_production.yaml
```

### Using YQ for YAML Manipulation

**Install YQ in Linux:**

```bash
add-apt-repository ppa:rmescandon/yq
apt-get install yq
```

**Example YAML File:**

```yaml
# vi yaml-templating-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: test-container
    image: registry.k8s.io/busybox
    env:
    - name: DB_URL
      value: postgres://db_url:5432
```

**Reading a Value with YQ:**

```bash
yq r yaml-templating-pod.yaml "spec.containers[0].env[0].value"
```

**Writing/Updating a Value with YQ:**

```bash
yq w yaml-templating-pod.yaml "spec.containers[0].env[0].value" "postgres://prod:5432"
```

**Generated Output:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: test-container
    image: registry.k8s.io/busybox
    env:
    - name: DB_URL
      value: postgres://prod:5432
```

### Merging YAML Files

You can execute the following command and merge the two YAMLs:

```bash
yq m -a append pod.yaml envoy-pod.yaml
```

---

## Helm Package Manager

### Installing Helm

**From Package Managers:**

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
apt-get update
apt-get install helm
```

**Via Script (Latest Version):**

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Adding Stable Helm Repository

```bash
helm repo add stable https://charts.helm.sh/stable
```

**Search for Charts:**

```bash
helm search repo jenkins
```

### Installing and Validating a Helm Chart

**Example: Installing Nginx Ingress Controller:**

```bash
# Step 1: Add the nginx-ingress helm repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Step 2: Update the chart repo
helm repo update

# Step 3: Install a stable Nginx chart
helm install ingress-controller ingress-nginx/ingress-nginx

# Step 4: Check the status of the helm deployment
helm ls

# Or use kubectl to check the ingress deployment
kubectl get deployments

# Step 5: Remove the deployment after validation
helm uninstall ingress-controller
```

### Creating a Simple Helm Chart

**Step 1: Initialize a New Chart:**

```bash
helm create myfirstchart
```

This creates a directory named `myfirstchart` with the basic structure of a Helm chart.

**Explore the Directory Structure:**

```
myfirstchart/
  ├── Chart.yaml          # Contains metadata about the chart
  ├── values.yaml         # Defines default values
  └── templates/          # Contains Kubernetes manifests
```

### Customizing the Helm Chart

**Step 1: Edit values.yaml**

Open the `values.yaml` file and customize the values to match your application's requirements. For example, you can set the image repository, tag, and service port.

**Step 2: Modify Kubernetes Manifests**

Edit the Kubernetes manifests in the `templates/` directory to configure your application. You can define services, deployments, config maps, and more.

### Validation of Helm Manifests

```bash
helm template . -f values.yaml
```

### Deploying the Helm Chart

**Step 1: Package the Chart:**

```bash
helm package .
```

This creates a `.tgz` file containing your chart.

**Step 2: Install the Chart:**

```bash
helm install myfirstrelease mychart-0.1.0.tgz
```

Replace `myfirstrelease` with a suitable release name. Helm will deploy your application using the configurations you defined.

---

## Summary Comparison

| Tool | Purpose | Key Features |
|------|---------|--------------|
| **Kustomize** | Kubernetes-native configuration management | No templating, uses patches and overlays, built into kubectl |
| **YQ** | YAML processor | Query, update, and merge YAML files |
| **Helm** | Kubernetes package manager | Templating with Go templates, chart versioning, release management |

---

## Best Practices

1. **Use Kustomize** for managing Kubernetes manifests when you need to maintain multiple environments with minimal changes
2. **Use Helm** when you need to package and distribute applications, or when you need complex templating
3. **Use YQ** for quick YAML manipulations in scripts or CI/CD pipelines
4. **Combine tools** when appropriate (e.g., using Helm to package charts and Kustomize for environment-specific overlays)
