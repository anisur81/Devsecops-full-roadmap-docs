# Using ConfigMaps and Secrets to Configure Applications

## Overview

ConfigMaps and Secrets are Kubernetes objects used to decouple configuration artifacts from container images, making applications more portable and manageable.

---

## ConfigMaps

ConfigMaps store non-sensitive configuration data in key-value pairs.

### Creating ConfigMaps

#### Method 1: Using `kubectl create configmap`

**From Literal Values:**
```bash
kubectl create configmap app-config --from-literal=key1=value1 --from-literal=key2=value2
```

**From Files:**
```bash
# Create from file (key = filename)
kubectl create configmap game-config --from-file=/path/to/file

# Create with custom key name
kubectl create configmap game-config --from-file=my-special-key=/path/to/file.properties
```

**Example with Custom Key:**
```bash
# Create directory and download files
mkdir -p /root/configmap
cd /root/configmap
wget https://kubernetes.io/examples/configmap/game.properties -O game.properties
wget https://kubernetes.io/examples/configmap/ui.properties -O ui.properties

# Create configmap with custom key
kubectl create configmap game-config-3 \
  --from-file=game-special-key=./game.properties
```

**Verify the ConfigMap:**
```bash
kubectl get configmaps game-config-3 -o yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: game-config-3
  namespace: default
data:
  game-special-key: |
    enemies=aliens
    lives=3
    enemies.cheat=true
    secret.code.passphrase=UUDDLRLRBABAS
```

---

#### Method 2: Using Kustomization.yaml

**Create kustomization.yaml:**
```yaml
# kustomization.yaml
configMapGenerator:
- name: game-config-4
  options:
    labels:
      game-config: config-4
  files:
  - ./game.properties
```

**Apply the configuration:**
```bash
kubectl apply -k .
```

**Verify:**
```bash
kubectl get configmap
kubectl describe configmaps/game-config-4-xxxxx
```

> **Note:** Generated ConfigMap names have a suffix appended (hash of contents) ensuring new ConfigMaps are created when content changes.

**Custom Key with Kustomization:**
```yaml
# kustomization.yaml
configMapGenerator:
- name: game-config-5
  options:
    labels:
      game-config: config-5
  files:
  - game-special-key=./game.properties
```

---

### Using ConfigMaps in Pods

#### 1. Environment Variables from Single ConfigMap

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: configmap-env-pod
spec:
  containers:
  - name: test-container
    image: nginx
    env:
    - name: DATABASE_URL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database.url
```

#### 2. Environment Variables from Multiple ConfigMaps

```yaml
# configmaps.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
data:
  special.how: very
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
data:
  log_level: INFO
```

```yaml
# pod-multiple-config-env-variable.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
  - name: test-container
    image: registry.k8s.io/busybox
    command: ["/bin/sh", "-c", "env"]
    env:
    - name: SPECIAL_LEVEL_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: special.how
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: env-config
          key: log_level
  restartPolicy: Never
```

**Create and verify:**
```bash
kubectl create -f configmaps.yaml
kubectl create -f pod-multiple-config-env-variable.yaml
kubectl logs dapi-test-pod
# Output: SPECIAL_LEVEL_KEY=very and LOG_LEVEL=INFO
```

#### 3. All ConfigMap Data as Environment Variables (envFrom)

```yaml
# configmap-multikeys.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
data:
  SPECIAL_LEVEL: very
  SPECIAL_TYPE: charm
```

```yaml
# pod-configmap-envfrom.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
  - name: test-container
    image: registry.k8s.io/busybox
    command: ["/bin/sh", "-c", "env"]
    envFrom:
    - configMapRef:
        name: special-config
  restartPolicy: Never
```

#### 4. ConfigMap Values in Pod Commands

```yaml
# pod-configmap-env-var-valuefrom.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
  - name: test-container
    image: registry.k8s.io/busybox
    command: ["/bin/echo", "$(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)"]
    env:
    - name: SPECIAL_LEVEL_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: SPECIAL_LEVEL
    - name: SPECIAL_TYPE_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: SPECIAL_TYPE
  restartPolicy: Never
```

**Output:**
```bash
kubectl logs dapi-test-pod
# very charm
```

---

#### 5. ConfigMap as Volume

**Mount entire ConfigMap:**
```yaml
# pod-config-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
  - name: test-container
    image: registry.k8s.io/busybox
    command: ["/bin/sh", "-c", "ls /etc/config/"]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: special-config
  restartPolicy: Never
```

**Mount specific keys with custom paths:**
```yaml
# pod-config-volume-specific-key.yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
  - name: test-container
    image: registry.k8s.io/busybox
    command: ["/bin/sh", "-c", "cat /etc/config/keys"]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: special-config
      items:
      - key: SPECIAL_LEVEL
        path: keys
  restartPolicy: Never
```

---

## Secrets

Secrets store sensitive information like passwords, tokens, or keys.

### Creating Secrets

#### Method 1: Using Literal Values
```bash
kubectl create secret generic db-user-pass \
  --from-literal=username=admin \
  --from-literal=password='pw123'
```

#### Method 2: Using Files
```bash
# Create files without newline
echo -n 'admin' > ./username.txt
echo -n 'S!B\*d$zDsb=' > ./password.txt

# Create secret from files
kubectl create secret generic db-user-pass \
  --from-file=./username.txt \
  --from-file=./password.txt

# Or with custom keys
kubectl create secret generic db-user-pass \
  --from-file=username=./username.txt \
  --from-file=password=./password.txt
```

### Verifying Secrets

```bash
# List secrets
kubectl get secrets

# View encoded data
kubectl get secret db-user-pass -o jsonpath='{.data}'

# Decode specific value
kubectl get secret db-user-pass -o jsonpath='{.data.password}' | base64 --decode
```

### Editing Secrets
```bash
kubectl edit secrets generic
```

---

### Secret Types

#### Basic Authentication Secret
```yaml
# basic-auth-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: t0p-Secret
```

**Create:**
```bash
kubectl apply -f basic-auth-secret.yaml
```

#### TLS Secret
```yaml
# tls-auth-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret-tls
type: kubernetes.io/tls
data:
  tls.crt: |
    LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0t...
  tls.key: |
    RXhhbXBsZSBkYXRhIGZvciB0aGUgVExTIGNydCBmaWVsZA==
```

**Create using kubectl:**
```bash
kubectl create secret tls my-tls-secret \
  --cert=path/to/cert/file \
  --key=path/to/key/file
```

---

### Using Secrets in Pods

#### Mount Secret as Volume
```yaml
# secret-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-test-pod
spec:
  containers:
  - name: test-container
    image: nginx
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secret-volume
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: test-secret
```

**Verify:**
```bash
kubectl apply -f secret-pod.yaml
kubectl get pod secret-test-pod
kubectl exec -i -t secret-test-pod -- /bin/bash

# Inside container:
ls /etc/secret-volume
cat /etc/secret-volume/username
cat /etc/secret-volume/password
```

---

## Best Practices

1. **Use ConfigMaps for non-sensitive configuration data**
2. **Use Secrets for sensitive information**
3. **Always encode sensitive data in base64**
4. **Consider using external secret management tools** (HashiCorp Vault, AWS Secrets Manager)
5. **Enable encryption at rest for Secrets**
6. **Use RBAC to restrict Secret access**
7. **Avoid storing Secrets in version control**
8. **Use environment variables for simple configs, volumes for complex configs**

---

## Comparison: ConfigMap vs Secret

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| **Purpose** | Non-sensitive configuration | Sensitive information |
| **Encoding** | Plain text | Base64 encoded |
| **Size Limit** | 1MiB per object | 1MiB per object |
| **Security** | Less secure | More secure |
| **Use Cases** | Environment variables, app config | Passwords, tokens, keys, certificates |
