#  Kubernetes Secrets Lab Guide

A hands-on lab series for practicing Kubernetes Secrets management, aligned with CKS
(Certified Kubernetes Security Specialist) exam objectives. 
Practice on   a kubeadm cluster (Lab 11 requires a self-managed control plane).

## Contents

1. [Create a Secret](#lab-1--create-a-secret-from-literals)
2. [View Secret Contents](#lab-2--view-secret-contents)
3. [Create Secret from File](#lab-3--create-secret-from-file)
4. [Create Secret from YAML](#lab-4--create-secret-from-yaml)
5. [Secrets as Environment Variables](#lab-5--use-secret-as-environment-variables)
6. [Mount Secret as Volume](#lab-6--mount-secret-as-volume)
7. [Update a Secret](#lab-7--update-a-secret)
8. [Imperative Secret Creation](#lab-8--secret-from-imperative-command)
9. [Secrets via envFrom](#lab-9--secret-via-envfrom)
10. [Restrict Access with RBAC](#lab-10--restrict-secret-access-using-rbac)
11. [Encrypt Secrets in etcd](#lab-11--encrypt-secrets-in-etcd-encryptionconfiguration)
12. [Secret Rotation](#lab-12--secret-rotation)
13. [Immutable Secrets](#lab-13--use-immutable-secrets)
14. [TLS & Docker-Registry Secret Types](#lab-14--tls-and-docker-registry-secret-types)
15. [External Secrets (CSI Driver preview)](#lab-15--external-secrets-with-secrets-store-csi-driver-conceptual)
16. [CKS Exam Command Cheat Sheet](#cks-exam-commands-to-memorize)
17. [CKS Best Practices](#cks-best-practices)

---

## Lab 1 – Create a Secret from Literals

**Objective:** Create a Secret from literal key/value pairs.

```bash
kubectl create ns secret-demo

kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=MyPassword123 \
  -n secret-demo
```

View and describe:

```bash
kubectl get secret -n secret-demo
kubectl describe secret db-secret -n secret-demo
```

`describe` lists key names only — values are never shown.

---

## Lab 2 – View Secret Contents

Secret data is stored **Base64-encoded**, not encrypted.

```bash
kubectl get secret db-secret -o yaml -n secret-demo
```

```yaml
data:
  password: TXlQYXNzd29yZDEyMw==
  username: YWRtaW4=
```

Decode:

```bash
echo TXlQYXNzd29yZDEyMw== | base64 -d   # MyPassword123
echo YWRtaW4= | base64 -d               # admin
```

**CKS Tip:** Base64 is encoding, not encryption. Anyone with `get` access to the Secret (or etcd) can trivially recover the plaintext.

---

## Lab 3 – Create Secret from File

```bash
echo -n admin > username.txt
echo -n MyPassword123 > password.txt

kubectl create secret generic file-secret \
  --from-file=username.txt \
  --from-file=password.txt \
  -n secret-demo

kubectl get secret file-secret -n secret-demo
```

---

## Lab 4 – Create Secret from YAML

Encode values first:

```bash
echo -n admin | base64
echo -n MyPassword123 | base64
```

`secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: yaml-secret
  namespace: secret-demo
type: Opaque
data:
  username: YWRtaW4=
  password: TXlQYXNzd29yZDEyMw==
```

```bash
kubectl apply -f secret.yaml
kubectl get secret -n secret-demo
```

> Tip: `stringData` accepts plain-text values and lets Kubernetes handle the Base64 encoding for you — useful to avoid manual encode/decode errors during the exam.

---

## Lab 5 – Use Secret as Environment Variables

`pod-env.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env
  namespace: secret-demo
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

```bash
kubectl apply -f pod-env.yaml
kubectl exec -it secret-env -n secret-demo -- env
```

Expected output includes `DB_USER=admin` and `DB_PASS=MyPassword123`.

**CKS Tip:** Env-var Secrets are visible via `/proc/<pid>/environ` inside the container and can leak into logs or child processes — prefer volume mounts for sensitive values where possible.

---

## Lab 6 – Mount Secret as Volume

`pod-volume.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume
  namespace: secret-demo
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secret
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: db-secret
```

```bash
kubectl apply -f pod-volume.yaml
kubectl exec -it secret-volume -n secret-demo -- ls /etc/secret
kubectl exec -it secret-volume -n secret-demo -- cat /etc/secret/password
```

---

## Lab 7 – Update a Secret

```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=NewPassword \
  -o yaml --dry-run=client \
  -n secret-demo | kubectl apply -f -

kubectl get secret db-secret -o yaml -n secret-demo
```

Volume-mounted Secrets update automatically (after a sync delay); env-var Secrets require a pod restart:

```bash
kubectl delete pod secret-env -n secret-demo
```

---

## Lab 8 – Secret from Imperative Command

```bash
kubectl create secret generic api-key --from-literal=token=123456789
kubectl get secret api-key
```

---

## Lab 9 – Secret via envFrom

`envfrom.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-demo
  namespace: secret-demo
spec:
  containers:
  - name: nginx
    image: nginx
    envFrom:
    - secretRef:
        name: db-secret
```

```bash
kubectl apply -f envfrom.yaml
kubectl exec -it envfrom-demo -n secret-demo -- env
```

---

## Lab 10 – Restrict Secret Access Using RBAC

```bash
kubectl create sa app-sa -n secret-demo
```

`role.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: secret-demo
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
```

`rolebinding.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: secret-demo
  name: read-secret
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: secret-demo
roleRef:
  kind: Role
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f role.yaml
kubectl apply -f rolebinding.yaml

kubectl auth can-i get secret \
  --as=system:serviceaccount:secret-demo:app-sa \
  -n secret-demo
# yes
```

Also verify the *negative* case, which the exam often tests:

```bash
kubectl auth can-i list secret \
  --as=system:serviceaccount:secret-demo:app-sa \
  -n secret-demo
# no (Role only granted "get")
```

---

## Lab 11 – Encrypt Secrets in etcd (EncryptionConfiguration)

> **Prerequisite:** A self-managed control plane (kubeadm). Not applicable to most managed services (EKS, GKE, AKS) unless they expose their own mechanism.

`/etc/kubernetes/encryption-config.yaml`:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: c2VjcmV0a2V5MTIzNDU2Nzg5MDEyMzQ1Njc4OTA=
  - identity: {}
```

Update the kube-apiserver static pod manifest:

```bash
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Add:

```text
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

Also mount the config file into the pod (add a `hostPath` volume + `volumeMount` for `/etc/kubernetes/encryption-config.yaml`), then let the static pod reload.

Re-encrypt existing Secrets so they're rewritten under the new provider:

```bash
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
```

Verify encryption directly against etcd (requires etcdctl and certs):

```bash
ETCDCTL_API=3 etcdctl get /registry/secrets/secret-demo/db-secret \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

The output should show the `k8s:enc:aescbc:v1:key1` prefix instead of readable data.

---

## Lab 12 – Secret Rotation

```bash
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=SuperSecure \
  -o yaml --dry-run=client | kubectl apply -f -

kubectl rollout restart deployment app
kubectl rollout status deployment app
```

---

## Lab 13 – Use Immutable Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: immutable-secret
  namespace: secret-demo
immutable: true
type: Opaque
stringData:
  username: admin
  password: MyPassword123
```

```bash
kubectl apply -f immutable-secret.yaml
```

Attempting to edit `data`/`stringData` afterward is rejected by the API server — this also protects against accidental writes and reduces load on kube-apiserver watch caches.

---

## Lab 14 – TLS and Docker-Registry Secret Types

Two built-in Secret types the exam expects you to know:

```bash
# TLS certificate/key pair
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  -n secret-demo

# Private image registry credentials
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=me@example.com \
  -n secret-demo
```

Reference the registry Secret in a Pod:

```yaml
spec:
  imagePullSecrets:
  - name: regcred
```

---

## Lab 15 – External Secrets with Secrets Store CSI Driver (Conceptual)

For production and CKS awareness (not usually a full hands-on build during the exam itself):

* **HashiCorp Vault** — Secrets stay in Vault; pods pull them at runtime via the Vault Agent Injector or the CSI provider, so nothing sensitive is stored as a native Kubernetes Secret object.
* **Secrets Store CSI Driver** — mounts secrets from external providers (Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager) as a volume, using a `SecretProviderClass` resource to define the mapping.
* **Sealed Secrets (Bitnami)** — encrypts Secrets client-side into a `SealedSecret` CRD safe to store in Git; only the in-cluster controller (holding the private key) can decrypt it back into a normal Secret.

Know the trade-offs: native Secrets are simple but only Base64-encoded at rest (unless etcd encryption is enabled); external managers add encryption, rotation, and audit at the cost of extra components.

---

## CKS Exam Commands to Memorize

```bash
kubectl create secret generic mysecret --from-literal=user=admin
kubectl get secret
kubectl describe secret mysecret
kubectl get secret mysecret -o yaml
kubectl exec POD -- env
kubectl exec POD -- cat /etc/secret/password
echo BASE64 | base64 -d
kubectl auth can-i get secrets
kubectl auth can-i get secrets --as=system:serviceaccount:NS:SA -n NS
kubectl rollout restart deployment APP
kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
kubectl create secret docker-registry regcred \
  --docker-server=SERVER --docker-username=USER \
  --docker-password=PASS --docker-email=EMAIL
```

---

## CKS Best Practices

* Never hardcode passwords or tokens directly in Pod manifests.
* Use Secrets (not ConfigMaps) for anything sensitive.
* Enable etcd encryption (`EncryptionConfiguration`) for Secrets on self-managed clusters.
* Grant Secret access only through least-privilege RBAC — scope Roles to specific verbs (`get` vs `list`/`watch`) and specific Secret names where feasible.
* Prefer mounting Secrets as volumes over environment variables; volume-mounted Secrets can update in place, and env vars are more exposure-prone.
* Rotate Secrets regularly and restart/rollout workloads afterward so the new values take effect.
* Use immutable Secrets for values that should never change, to reduce accidental modification and API server load.
* Consider external secret managers (Vault, Secrets Store CSI Driver, Sealed Secrets) for production, especially where Git-stored manifests or audit/rotation requirements are involved.
* Restrict `etcd` access itself — anyone with direct etcd access bypasses Kubernetes RBAC entirely.

---

*This guide covers the core Secret-management skills tested in CKS-style scenarios: creation, consumption, RBAC restriction, etcd encryption, rotation, immutability, and awareness of external secret managers.*
