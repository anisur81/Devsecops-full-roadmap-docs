# Understand Your Supply Chain (SBOM, CI/CD, Artifact Repositories)


You should understand: 

* Minimize base image footprint
* Generate and use SBOM (Software Bill of Materials)
* Static analysis of workload manifests
* Secure the CI/CD pipeline
* Secure artifact repositories (Docker Registry, Harbor, GHCR, ECR, etc.)
* Scan images for known vulnerabilities (Trivy, Grype)
* Sign container images
* Verify images before deployment (Admission Control — this is the part actually tested)

> **Note:** the original version of this doc stopped at "an admission controller verifies signature/registry/digest" without ever showing *how*. On the real exam you are usually asked to configure that enforcement yourself (Kyverno, OPA/Gatekeeper, or the built-in `ImagePolicyWebhook`), so that's the biggest addition here.

---

# Lab Architecture

```
Developer
     │
     ▼
 GitHub Repository
     │
     ▼
GitHub Actions CI/CD
     │
 ┌────────────────────┐
 │ Minimize Base Image │
 │ Generate SBOM       │
 │ Scan Image          │
 │ Sign Image          │
 └────────────────────┘
     │
     ▼
Artifact Registry (GHCR/Harbor)
     │
     ▼
Admission Controller (Kyverno/Gatekeeper/ImagePolicyWebhook)
     │
     ▼
Kubernetes Cluster
```

---

# Lab Prerequisites

Ubuntu VM

Install

```bash
kubectl
docker
kind
git
```

Create Kind Cluster

```bash
kind create cluster --name cks
```

Verify

```bash
kubectl get nodes
```

Output

```
NAME                STATUS
cks-control-plane   Ready
```

---

# Lab 1 — Build a Container Image

Create directory

```bash
mkdir supplychain
cd supplychain
```

Create app

```bash
cat > app.py <<EOF
print("Hello CKS")
EOF
```

Create Dockerfile

```Dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY app.py .

CMD ["python","app.py"]
```

Build image

```bash
docker build -t demo:v1 .
```

Verify

```bash
docker images
```

---

# Lab 1b — Minimize Base Image Footprint (NEW)

A smaller image = smaller attack surface = fewer CVEs to scan and sign. This is an explicit CKS curriculum line item ("minimize base image footprint").

Key techniques:

* Prefer minimal/distroless images over full OS images
* Use multi-stage builds so build tools never ship in the final image
* Run as non-root
* Avoid `:latest` — pin versions
* Remove package manager caches and unused packages
* No secrets, SSH keys, or `.git` folders baked into the image

Example hardened Dockerfile:

```Dockerfile
# Stage 1: build
FROM python:3.12-slim AS build
WORKDIR /app
COPY app.py .

# Stage 2: minimal runtime
FROM gcr.io/distroless/python3-debian12
WORKDIR /app
COPY --from=build /app/app.py .
USER nonroot:nonroot
CMD ["app.py"]
```

Compare sizes:

```bash
docker images demo:v1 --format "{{.Size}}"
```

`dive` can also be used to inspect image layers and find bloat:

```bash
dive demo:v1
```

---

# Lab 2 — Generate SBOM

## Install Syft

```bash
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh
```

Verify

```bash
syft version
```

Generate SBOM

```bash
syft demo:v1
```

Example Output

```
python
glibc
openssl
pip
setuptools
```

Save SBOM

```bash
syft demo:v1 -o json > sbom.json
```

View

```bash
cat sbom.json
```

---

## Why SBOM?

SBOM tells you

* every package
* every library
* every dependency

inside an image.

---

# Lab 3 — Scan Image/SBOM for Vulnerabilities

## Grype

Install Grype

```bash
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh
```

Scan

```bash
grype demo:v1
```

Scan an existing SBOM directly (faster, no re-pull needed):

```bash
grype sbom:sbom.json
```

Output

```
Critical 1
High 3
Medium 10
```

Export report

```bash
grype demo:v1 -o json > report.json
```

## Trivy (NEW — this is the scanner most CKS exam tasks actually reference)

Install

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```

Scan an image:

```bash
trivy image demo:v1
```

Scan for only HIGH/CRITICAL and fail the pipeline if found (common CI gate):

```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 demo:v1
```

Scan a Kubernetes manifest or Dockerfile for misconfigurations:

```bash
trivy config .
```

Scan a running cluster's workloads (kbom/cluster mode):

```bash
trivy k8s --report summary cluster
```

---

# Lab 3b — Static Analysis of Manifests (NEW)

CKS expects you to catch insecure manifests *before* they reach the cluster, not just scan images.

## kubesec

```bash
docker run -i kubesec/kubesec:v2 scan /dev/stdin < deploy.yaml
```

Flags issues like missing `runAsNonRoot`, missing resource limits, allowed privilege escalation, etc.

## conftest (OPA policies against YAML)

```bash
conftest test deploy.yaml -p policy/
```

Example policy (`policy/deny_privileged.rego`):

```rego
package main

deny[msg] {
  input.spec.template.spec.containers[_].securityContext.privileged == true
  msg = "Privileged containers are not allowed"
}
```

---

# Lab 4 — Push Image to Artifact Repository

Login

Example GHCR

```bash
docker login ghcr.io
```

Tag

```bash
docker tag demo:v1 ghcr.io/myuser/demo:v1
```

Push

```bash
docker push ghcr.io/myuser/demo:v1
```

Verify image exists in registry.

---

# Lab 5 — Sign Container Image

## Key-based signing

Install Cosign

```bash
brew install cosign
```

or

```bash
sudo apt install cosign
```

Generate key

```bash
cosign generate-key-pair
```

Sign image

```bash
cosign sign \
--key cosign.key \
ghcr.io/myuser/demo:v1
```

Verify signature

```bash
cosign verify \
--key cosign.pub \
ghcr.io/myuser/demo:v1
```

Expected

```
Verified OK
```

## Keyless signing (NEW — Sigstore/Rekor, increasingly common in real pipelines)

```bash
COSIGN_EXPERIMENTAL=1 cosign sign ghcr.io/myuser/demo:v1
```

This uses an OIDC identity (e.g. GitHub Actions token) instead of a static private key, and records the signature in the public Rekor transparency log — no key management needed.

Verify:

```bash
COSIGN_EXPERIMENTAL=1 cosign verify ghcr.io/myuser/demo:v1
```

---

# Lab 6 — Enforce Verification Before Deployment (Admission Control) (EXPANDED)

The original doc said "an admission controller verifies signature/registry/digest" — here's how that's actually configured. Pick whichever tool is available in your exam environment.

## Option A: Kyverno — verify image signatures

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image-signature
spec:
  validationFailureAction: Enforce
  rules:
  - name: verify-signature
    match:
      resources:
        kinds:
        - Pod
    verifyImages:
    - imageReferences:
      - "ghcr.io/myuser/*"
      key: |-
        -----BEGIN PUBLIC KEY-----
        <cosign.pub contents>
        -----END PUBLIC KEY-----
```

Any pod using an unsigned or wrongly-signed image from `ghcr.io/myuser/*` is rejected at admission time.

## Option B: Kyverno — restrict allowed registries

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-registries
spec:
  validationFailureAction: Enforce
  rules:
  - name: allowed-registries
    match:
      resources:
        kinds:
        - Pod
    validate:
      message: "Only images from ghcr.io/myuser are allowed"
      pattern:
        spec:
          containers:
          - image: "ghcr.io/myuser/*"
```

## Option C: OPA Gatekeeper — disallow `:latest` tag / require digest

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDisallowedTags
metadata:
  name: no-latest-tag
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    tags: ["latest"]
```

## Option D: Built-in `ImagePolicyWebhook` (native admission controller, exam-relevant)

Enabled via the API server flag:

```bash
--enable-admission-plugins=ImagePolicyWebhook
```

With an `AdmissionConfiguration` pointing at a webhook backend that decides allow/deny per image — this is the "native Kubernetes" mechanism referenced in the CKS curriculum for image verification, as opposed to Kyverno/Gatekeeper which are third-party add-ons.

---

Deploy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: demo
        image: ghcr.io/myuser/demo:v1
```

Apply

```bash
kubectl apply -f deploy.yaml
```

If a policy above is active and the image fails verification, you should see something like:

```
Error from server: admission webhook "validate.kyverno.svc-fail" denied the request
```

---

# Lab 7 — Image Digest

Instead of

```yaml
image: demo:v1
```

Use

```yaml
image: ghcr.io/myuser/demo@sha256:abcd1234...
```

Benefits

* immutable
* cannot be overwritten
* reproducible

---

# Lab 8 — Secure CI/CD Pipeline

Example GitHub Actions

```yaml
jobs:

 build:

 scan:

 sbom:

 sign:

 push:

 deploy:
```

Recommended order

```
Checkout

↓

Lint

↓

SAST

↓

Unit Test

↓

Build Minimal Image

↓

Generate SBOM

↓

Scan Image (Trivy/Grype)

↓

Sign Image (Cosign)

↓

Push Registry

↓

Admission Control Verification

↓

Deploy Kubernetes
```

This is the order expected in real-world pipelines and aligns well with CKS concepts.

CI/CD hardening notes:

* Never store the cosign private key or registry credentials in plaintext in the repo — use CI secrets/vault
* Restrict who can approve merges that trigger `deploy`
* Pin GitHub Actions to commit SHAs, not `@main` or `@latest`, to prevent supply-chain tampering of the pipeline itself

---

# Lab 9 — Verify Running Image

List images

```bash
kubectl get pods
```

Describe pod

```bash
kubectl describe pod demo
```

Verify image

```
Image:
ghcr.io/myuser/demo@sha256:...
```

---

# Lab 10 — Continuous Scanning

Whenever a new CVE appears

Run

```bash
grype ghcr.io/myuser/demo:v1
```

or

```bash
trivy image ghcr.io/myuser/demo:v1
```

You may discover

```
Critical: openssl
```

Rebuild

```
FROM python:3.12.5-slim
```

Re-push image.

---

# CKS Exam Tips (EXPANDED)

Know how to:

* Minimize a base image (multi-stage build, distroless, non-root)
* Generate an SBOM (`syft`)
* Scan images (`grype`, `trivy`) — know both, Trivy is very commonly referenced
* Statically analyze manifests (`kubesec`, `conftest`/OPA rego policies)
* Push to a trusted registry
* Use image digests instead of tags
* Sign images (`cosign`), both key-based and keyless
* **Configure** an admission controller to enforce signature/registry/digest policy (Kyverno `verifyImages`, OPA Gatekeeper constraints, or `ImagePolicyWebhook`) — don't just describe it, be ready to write the YAML
* Explain the role of artifact repositories and CI/CD in the supply chain

---

# End-to-End Workflow

```text
Developer
      │
      ▼
Git Repository
      │
      ▼
CI/CD Pipeline
      │
      ├── Minimize Base Image
      ├── Build Image
      ├── Generate SBOM (Syft)
      ├── Scan (Grype/Trivy)
      ├── Static Manifest Analysis (kubesec/conftest)
      ├── Sign (Cosign)
      ▼
Artifact Registry (GHCR/Harbor)
      │
      ▼
Admission Controller (Kyverno / Gatekeeper / ImagePolicyWebhook)
      │
      ▼
Kubernetes Cluster
```

---

## CKS Exam Focus

A common CKS exam scenario is to:

1. Build a minimal container image (multi-stage/distroless, non-root).
2. Generate an SBOM.
3. Scan the image for vulnerabilities (Trivy or Grype).
4. Statically analyze the Kubernetes manifest before it's applied.
5. Sign the image.
6. Store it in a trusted artifact registry.
7. **Configure the Kubernetes cluster (via Kyverno, OPA Gatekeeper, or ImagePolicyWebhook) to admit only trusted, verified, correctly-registried images — and prove it by showing a bad image get rejected.**

Practicing this complete workflow, including step 7 with real YAML, will prepare you for both the exam and production-grade Kubernetes supply chain security.
