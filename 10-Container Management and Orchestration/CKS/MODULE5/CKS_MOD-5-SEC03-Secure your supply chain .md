# Secure your supply chain (permitted registries, sign and validate artifacts, etc.)

## Objective

This lab covers one of the CKS exam objectives:

* ✅ Whitelist allowed image registries
* ✅ Reject images from unauthorized registries
* ✅ Sign container images
* ✅ Verify signed images before deployment
* ✅ Scan images for known vulnerabilities (added — this is part of the same official CKS "Supply Chain Security" domain)

This lab is divided into **six** parts.

```
Part 1
-------
Allow images only from trusted registries
(OPA Gatekeeper)

Part 2
-------
Reject images from Docker Hub

Part 3
-------
Sign Docker images
(Docker Content Trust) — legacy, low exam weight now

Part 4
-------
Validate image signatures
(Concept + Production approach)

Part 5  [NEW]
-------
Scan images for vulnerabilities
(Trivy)

Part 6  [NEW]
-------
Enforce signature verification at admission time
(Kyverno + Cosign — key-based and keyless)
```

> **Why the additions matter:** The official CKS curriculum groups "Minimize base image footprint / scan images for vulnerabilities" and "whitelist registries / sign & validate artifacts" under the same **Supply Chain Security** domain (~20% of the exam). A lab that only covers registries + DCT signing is missing roughly half of that domain's likely question surface — image scanning and modern policy-based signature verification (Kyverno) are just as commonly tested as Gatekeeper registry rules.

---

# Lab Environment

Ubuntu VM

```
Docker
kubectl
Helm
Kind or Kubernetes Cluster
```

Verify

```bash
kubectl get nodes
docker version
helm version
```

---

# Part 1

# Restrict Allowed Registries using OPA Gatekeeper

## Architecture

```
User
 │
 │ kubectl apply
 ▼
API Server
 │
 ▼
OPA Gatekeeper
 │
 ├── Image from Docker Hub ❌ Reject
 └── Image from GHCR ✅ Allow
```

---

## Step 1 Install Gatekeeper

```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.17/deploy/gatekeeper.yaml
```

Wait

```bash
kubectl get pods -n gatekeeper-system
```

Expected

```
gatekeeper-controller-manager
Running
```

---

# Step 2

Create ConstraintTemplate

template.yaml

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sallowedrepos

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not startswith(container.image, "ghcr.io/")
        msg := sprintf("Image '%v' is not from allowed registry", [container.image])
      }
```

Apply

```bash
kubectl apply -f template.yaml
```

---

# Step 3

Create Constraint

constraint.yaml

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: only-ghcr
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

Apply

```bash
kubectl apply -f constraint.yaml
```

---

# Step 4

Deploy Allowed Image

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-good
spec:
  containers:
  - name: nginx
    image: ghcr.io/my-org/nginx:latest
```

Apply

```bash
kubectl apply -f pod-good.yaml
```

Expected

```
Created
```

---

# Step 5

Deploy Unauthorized Image

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-bad
spec:
  containers:
  - name: nginx
    image: nginx:latest
```

Apply

```bash
kubectl apply -f pod-bad.yaml
```

Expected

```
Error

admission webhook denied the request

Image nginx:latest is not from allowed registry
```

This is a very common CKS scenario.

---

# Part 2

# Allow Multiple Registries

Modify Rego

```rego
package k8sallowedrepos

violation[{"msg": msg}] {

 container := input.review.object.spec.containers[_]

 not startswith(container.image,"ghcr.io/")
 not startswith(container.image,"registry.k8s.io/")
 not startswith(container.image,"quay.io/")

 msg := sprintf("Registry not allowed: %v",[container.image])

}
```

Allowed

```
ghcr.io

registry.k8s.io

quay.io
```

Rejected

```
docker.io

gcr.io

private registry
```

---

# Part 3

# Docker Content Trust

Docker Content Trust signs Docker images.

> **Exam note:** DCT is still mentioned in some older CKS material, but Docker's own trust server (Notary v1) is largely deprecated and DCT rarely appears in real production clusters today. Treat this part as *concept familiarity only* — expect Part 6 (Cosign/Kyverno) to be the more heavily tested modern equivalent.

Enable

```bash
export DOCKER_CONTENT_TRUST=1
```

Check

```bash
echo $DOCKER_CONTENT_TRUST
```

Expected

```
1
```

---

## Create Repository Keys

Generate signing keys

```bash
docker trust key generate demo
```

Example

```
Generating key...

demo.pub
demo.key
```

---

## Sign Image

Pull image

```bash
docker pull nginx:latest
```

Tag

```bash
docker tag nginx:latest myrepo/nginx:v1
```

Sign

```bash
docker trust sign myrepo/nginx:v1
```

Expected

```
Signing and pushing trust metadata

Successfully signed
```

---

## Inspect Signature

```bash
docker trust inspect --pretty myrepo/nginx:v1
```

Expected

```
Signatures

Signed by

demo
```

---

# Verify Signature

```bash
docker trust inspect myrepo/nginx:v1
```

Output

```
Signed

Yes
```

---

# Unsiged Image

Try

```bash
docker pull ubuntu:latest
```

If DCT is enabled and the image is unsigned:

```
Error

No valid trust data
```

---

# Disable Content Trust

```bash
export DOCKER_CONTENT_TRUST=0
```

---

# Part 4

# ImagePolicyWebhook (Concept)

Flow

```
kubectl apply

      │

API Server

      │

ImagePolicyWebhook

      │

Webhook Server

      │

Verify

Registry

Image Signature

Digest

Organization

      │

Allow / Deny
```

Admission Controller decides whether the image is trusted.

---

# Example ImagePolicyWebhook Configuration

AdmissionConfiguration

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/imagepolicy/kubeconfig
      allowTTL: 30
      denyTTL: 30
      retryBackoff: 500
```

Enable API Server

```
--enable-admission-plugins=ImagePolicyWebhook
```

---

# Example Policy

Allow

```
ghcr.io/company/*
```

Reject

```
docker.io/*
```

Reject

```
Unsigned image
```

Reject

```
Latest tag
```

Allow

```
Image Digest
```

---

# Production Image Signing (Recommended)

Docker Content Trust is useful for learning and legacy workflows, but modern Kubernetes environments typically use:

* **Cosign** to sign OCI container images.
* **Sigstore** for keyless or key-based signing and verification.
* **Kyverno** or **OPA Gatekeeper** to enforce signature verification during admission.
* **Image digests** (`image@sha256:...`) instead of mutable tags like `latest`.

Example Cosign workflow:

```bash
# Generate a key pair
cosign generate-key-pair

# Sign an image
cosign sign ghcr.io/my-org/myapp:v1

# Verify the signature
cosign verify ghcr.io/my-org/myapp:v1
```

A Kyverno policy can then reject any image that is not signed by your trusted key. See Part 6 for the full working example.

---

# Part 5 (NEW)

# Scan Images for Known Vulnerabilities (Trivy)

This is the "minimize base image footprint / scan images" half of the CKS Supply Chain Security domain, and it is frequently tested alongside registry whitelisting.

## Install Trivy

```bash
sudo apt-get install -y wget apt-transport-https gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install -y trivy
```

## Scan an Image

```bash
trivy image nginx:latest
```

Expected (abridged)

```
Total: 120 (UNKNOWN: 0, LOW: 40, MEDIUM: 50, HIGH: 25, CRITICAL: 5)
```

## Fail a Pipeline on High/Critical Findings

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 nginx:latest
```

* Exit code `1` → vulnerabilities found at that severity → CI pipeline fails the build.
* Exit code `0` → clean, or only below-threshold findings.

## Scan a Local Tarball or Dockerfile

```bash
# Scan an image saved as a tar archive
trivy image --input nginx.tar

# Scan a Dockerfile for misconfigurations
trivy config ./Dockerfile
```

## Generate an SBOM (Software Bill of Materials)

SBOMs are an increasingly common CKS-adjacent topic — they document exactly what's inside an image (packages, versions, licenses).

```bash
trivy image --format cyclonedx --output sbom.json nginx:latest
```

---

# Part 6 (NEW)

# Enforce Signature Verification at Admission Time (Kyverno + Cosign)

`ImagePolicyWebhook` (Part 4) is the *conceptual* mechanism the CKS curriculum names, but in practice — and increasingly in exam scenarios — **Kyverno's `verifyImages` rule** is the concrete, testable way to block unsigned images at admission time. It is simpler to install and inspect than a custom ImagePolicyWebhook backend.

## Install Kyverno

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

Verify

```bash
kubectl get pods -n kyverno
```

## Sign an Image with a Key Pair

```bash
cosign generate-key-pair
cosign sign --key cosign.key ghcr.io/my-org/myapp:v1
```

## Key-Based Verification Policy

verify-image-key.yaml

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  webhookConfiguration:
    failurePolicy: Fail
    timeoutSeconds: 30
  background: false
  rules:
  - name: verify-signature
    match:
      any:
      - resources:
          kinds:
          - Pod
    verifyImages:
    - imageReferences:
      - "ghcr.io/my-org/*"
      mutateDigest: true
      required: true
      attestors:
      - count: 1
        entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              <contents of cosign.pub>
              -----END PUBLIC KEY-----
```

Apply

```bash
kubectl apply -f verify-image-key.yaml
```

Expected behavior:

```
Signed image, correct key   → Pod created (image mutated to @sha256 digest)
Signed image, wrong key     → admission denied
Unsigned image              → admission denied
```

## Keyless Verification (Sigstore Fulcio + Rekor)

No long-lived private key to manage — identity comes from an OIDC issuer (e.g., a GitHub Actions workflow) and is recorded in the public Rekor transparency log.

Sign in CI:

```bash
COSIGN_EXPERIMENTAL=1 cosign sign ghcr.io/my-org/myapp:v1
```

Policy:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-keyless
spec:
  webhookConfiguration:
    timeoutSeconds: 30
  background: false
  rules:
  - name: keyless-verification
    match:
      any:
      - resources:
          kinds:
          - Pod
    verifyImages:
    - imageReferences:
      - "ghcr.io/my-org/*"
      attestors:
      - count: 1
        entries:
        - keyless:
            subject: "https://github.com/my-org/*"
            issuer: "https://token.actions.githubusercontent.com"
            rekor:
              url: https://rekor.sigstore.dev
```

## Restricting Registries with Kyverno (Alternative to Gatekeeper)

Kyverno can also replace Part 1/2's Gatekeeper rules with a native `validate` rule — useful to know since the exam may ask for "a policy engine" without naming one specifically.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-image-registries
spec:
  validationFailureAction: Enforce
  background: false
  rules:
  - name: allowed-registries
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Images must come from an allowed registry"
      pattern:
        spec:
          containers:
          - image: "ghcr.io/* | registry.k8s.io/* | quay.io/*"
```

---

# CKS Exam Practice Tasks

### Task 1

Allow images only from:

```
registry.k8s.io
```

Reject everything else.

---

### Task 2

Allow:

```
ghcr.io/company/*
quay.io/company/*
```

Reject Docker Hub.

---

### Task 3

Enable Docker Content Trust.

Sign an image.

Verify the signature.

---

### Task 4

Write a Gatekeeper policy that blocks:

```
:latest
```

but allows:

```
:v1.2.3
```

---

### Task 5

Update the Gatekeeper policy to permit only these registries:

```
registry.k8s.io
ghcr.io
quay.io
```

and ensure all images are referenced by immutable digests (`@sha256:...`) instead of tags.

---

### Task 6 (NEW)

Scan `myapp:v1` with Trivy and fail the build if any `CRITICAL` vulnerabilities are found.

---

### Task 7 (NEW)

Write a Kyverno `ClusterPolicy` that:

* Rejects any Pod in the `production` namespace whose image is not signed by `cosign.pub`.
* Mutates verified images to their digest form automatically.

---

### Task 8 (NEW)

Configure a Kyverno policy to verify images signed **keylessly** via a specific GitHub Actions repository, using Fulcio/Rekor.

---

# CKS Exam Tips

| Requirement                     | Recommended Solution                                                             |
| -------------------------------- | --------------------------------------------------------------------------------|
| Restrict image registries        | OPA Gatekeeper or Kyverno `validate` policy                                     |
| Prevent untrusted images          | Admission Controller (`ImagePolicyWebhook`) — concept; Kyverno `verifyImages` — practical |
| Ensure image integrity            | Sign images with Cosign (preferred) or Docker Content Trust (legacy)            |
| Verify image authenticity         | Cosign verification or Kyverno/OPA admission policy                            |
| Avoid mutable images              | Use image digests (`@sha256`) instead of `:latest`                             |
| Block `latest` tag                | Admission policy (OPA/Kyverno)                                                  |
| Whitelist registries              | `ghcr.io`, `registry.k8s.io`, `quay.io`, or your organization's private registry|
| Scan images for CVEs (NEW)        | `trivy image <image>` — gate CI on `--exit-code 1 --severity HIGH,CRITICAL`     |
| Keyless signing / identity-based trust (NEW) | Cosign + Sigstore Fulcio/Rekor, enforced via Kyverno `keyless` attestor |
| Document image contents (NEW)     | Generate an SBOM with `trivy image --format cyclonedx`                         |
 
