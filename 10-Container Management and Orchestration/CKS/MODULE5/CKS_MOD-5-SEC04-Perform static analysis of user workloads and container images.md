## Perform static analysis of user workloads and container images (e.g. Kubesec, KubeLinter)**

Your original notes were built around an older CKS syllabus that named **kube-score** and **Checkov** as the reference tools. The current curator page for this domain names **Kubesec** and **KubeLinter** explicitly, and the objective now also calls out **container images**, not just workload manifests. Kubesec/kube-score/KubeLinter/Checkov all only look at YAML — none of them inspect the actual image layers — so a container-image scanner (e.g. **Trivy**) needs to be added to cover that half of the objective.

This updated version:

- Adds **KubeLinter**, the tool the current objective actually names, with install + scan steps
- Adds a **container image scanning** step (Trivy) since the objective text now includes "container images"
- Keeps kube-score and Checkov as **supplementary/bonus tools** (still valid, still show up in real pipelines, but not the tools the objective names)
- Fixes the **kube-hunter placement** — it does not belong to this objective at all (see note below)
- Swaps hardcoded release versions for "always grab latest" install commands, since pinned URLs go stale
- Corrects a factual error in the original: Checkov is not limited to "Kubernetes, Dockerfile, Terraform, Helm, CloudFormation" — it also covers Serverless, ARM, OpenAPI, and more; the comparison table below reflects that generically

---

# Lab Architecture

```
Ubuntu VM
│
├── Docker
├── kubectl
├── kind
├── kubesec        (exam tool)
├── kube-linter     (exam tool)
├── trivy          (container image scanning — exam tool, image half of objective)
├── kube-score     (bonus / real-world pipelines)
├── checkov        (bonus / real-world pipelines)
└── Vulnerable Application
      │
      ├── Dockerfile
      ├── deployment.yaml
      ├── service.yaml
      └── namespace.yaml
```

---

# Step 1 — Create Project

```bash
mkdir cks-static-analysis
cd cks-static-analysis
mkdir manifests
```

---

# Step 2 — Create an Insecure Dockerfile

```bash
nano Dockerfile
```

```dockerfile
FROM ubuntu:latest

RUN apt update && apt install -y nginx curl

COPY . /app

WORKDIR /app

RUN chmod -R 777 /app

EXPOSE 80

CMD ["nginx","-g","daemon off;"]
```

Problems: `latest` tag, runs as root (no `USER`), `chmod 777`, large base image, no `HEALTHCHECK`.

---

# Step 3 — Create Vulnerable Deployment

```bash
nano manifests/deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: insecure-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: insecure
  template:
    metadata:
      labels:
        app: insecure
    spec:
      hostNetwork: true
      containers:
      - name: app
        image: nginx:latest
        securityContext:
          privileged: true
          allowPrivilegeEscalation: true
          runAsUser: 0
        ports:
        - containerPort: 80
```

Problems: `latest` image, `privileged`, root, privilege escalation allowed, `hostNetwork`.

---

# Step 4 — Service

```bash
nano manifests/service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: insecure-service
spec:
  selector:
    app: insecure
  ports:
  - port: 80
```

---

# Step 5 — Install Kubesec

Use the GitHub API to always grab the current release instead of a hardcoded version tag:

```bash
KS_URL=$(curl -s https://api.github.com/repos/controlplaneio/kubesec/releases/latest \
  | grep browser_download_url | grep linux_amd64.tar.gz | cut -d '"' -f4)

curl -sL "$KS_URL" -o kubesec.tar.gz
tar -xvf kubesec.tar.gz
sudo mv kubesec /usr/local/bin
kubesec version
```

# Step 6 — Scan Deployment with Kubesec

```bash
kubesec scan manifests/deployment.yaml
```

Example (illustrative) output:

```
Score: -20

Critical Issues
✗ privileged container
✗ run as root
✗ hostNetwork enabled
✗ privilege escalation
✗ latest image
```

A negative score means the workload is very insecure.

---

# Step 7 — Fix Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure
  template:
    metadata:
      labels:
        app: secure
    spec:
      securityContext:
        runAsNonRoot: true
      containers:
      - name: app
        image: nginx:1.27.5
        securityContext:
          privileged: false
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        ports:
        - containerPort: 80
```

Re-run `kubesec scan manifests/deployment.yaml` — score should move well into positive territory.

---

# Step 8 — Install KubeLinter *(exam-named tool)*

```bash
LOCATION=$(curl -s https://api.github.com/repos/stackrox/kube-linter/releases/latest \
  | grep browser_download_url | grep kube-linter-linux.tar.gz | cut -d '"' -f4)

curl -sL -o kube-linter-linux.tar.gz "$LOCATION"
mkdir -p kube-linter
tar -xf kube-linter-linux.tar.gz -C kube-linter/
sudo mv kube-linter/kube-linter /usr/local/bin/kube-linter
kube-linter version
```

(A one-line install script is also published at `https://raw.githubusercontent.com/stackrox/kube-linter/main/scripts/install.sh` — pipe it to `sh` if you prefer.)

# Step 9 — Scan with KubeLinter

```bash
kube-linter lint manifests/
```

Typical findings against the insecure deployment:

```
manifests/deployment.yaml: (object: <no namespace>/insecure-app apps/v1, Kind=Deployment)
  privileged container "app" is not allowed (check: privileged-container)
  container "app" is running as root (check: run-as-non-root)
  container "app" has hostNetwork enabled (check: host-network)
  container "app" has allowPrivilegeEscalation set to true (check: privilege-escalation-container)
  image "nginx:latest" uses the 'latest' tag, which is dangerous (check: latest-tag)
  container "app" does not have a read-only root file system (check: no-read-only-root-fs)
  container "app" has no resource requests/limits set (check: unset-cpu-requirements, unset-memory-requirements)
```

Re-run `kube-linter lint manifests/` against the fixed `secure-app` deployment — most/all checks should pass. To fail a pipeline on parse errors, add `--fail-on-invalid-resource`. `kube-linter` also lints Helm charts directly (`kube-linter lint ./my-chart --format helm` style usage — check `kube-linter lint --help` for the current flags).

---

# Step 10 — Container Image Scanning (Trivy) — the "container images" half of the objective

Kubesec and KubeLinter only ever look at YAML. To actually cover "container images" in the objective, scan the built image for known CVEs:

```bash
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

docker build -t insecure-app:latest .
trivy image insecure-app:latest
```

Expect findings such as outdated base-image packages, known CVEs in installed `curl`/`nginx` versions, and a warning about scanning a `latest`-tagged image. Rebuild from `Step 12`'s hardened Dockerfile and re-scan — the CVE count should drop.

---

# Step 11 (Bonus) — kube-score

Not named in the current exam objective, but still common in real pipelines and worth knowing conceptually.

```bash
KSCORE_URL=$(curl -s https://api.github.com/repos/zegl/kube-score/releases/latest \
  | grep browser_download_url | grep kube-score_linux_amd64 | cut -d '"' -f4)
curl -sL "$KSCORE_URL" -o kube-score
chmod +x kube-score
sudo mv kube-score /usr/local/bin/kube-score
kube-score score manifests/deployment.yaml
```

Flags things like missing probes, missing resource requests/limits, and `latest` tags.

---

# Step 12 (Bonus) — Checkov

```bash
pip install checkov
checkov --version
checkov -d manifests
checkov -f Dockerfile
```

Checkov's real value is that it's policy-as-code across many IaC formats (Kubernetes, Dockerfile, Terraform, CloudFormation, Helm, Serverless, ARM, and more) — not just Kubernetes.

---

# Step 13 — Secure Dockerfile

```dockerfile
FROM nginx:1.27.5-alpine

RUN addgroup app && adduser -S app -G app

COPY . /usr/share/nginx/html

USER app

HEALTHCHECK CMD wget --spider http://localhost || exit 1

EXPOSE 80

CMD ["nginx","-g","daemon off;"]
```

Re-run whichever scanners you installed against this file — failures should drop across the board.

---

# Note on kube-hunter — it is NOT part of this objective

The original doc placed kube-hunter here, but kube-hunter attacks a **live, running cluster** (looking for exposed dashboards, anonymous API access, exposed kubelet/etcd ports, etc.) — it does zero static analysis of YAML or images. In the current CKS curriculum this belongs under a separate runtime/cluster-hardening objective, not "static analysis of user workloads and container images." Keep it in your notes, just don't file it under this domain.

```bash
docker run --network host aquasec/kube-hunter
```

---

# Comparison

| Tool | Named in current CKS objective? | Scans | Best for |
|---|---|---|---|
| Kubesec | Yes | K8s manifests | Security-context / privilege scoring |
| KubeLinter | Yes | K8s manifests, Helm charts | Best-practice + security linting, CI-friendly |
| Trivy (or similar) | Implied by "container images" | Container image layers | CVEs in OS packages / app dependencies |
| kube-score | No (legacy/bonus) | K8s manifests | Reliability + best practices |
| Checkov | No (legacy/bonus) | K8s, Dockerfile, Terraform, Helm, CloudFormation, etc. | Multi-platform policy-as-code |
| kube-hunter | No — different objective | Live running cluster | Runtime exposure / penetration testing |

---

# CKS Exam Tips

- **Kubesec** — detects privileged containers, root users, `hostNetwork`, `allowPrivilegeEscalation`, missing security settings; outputs a numeric score.
- **KubeLinter** — the second exam-named tool; lints YAML and Helm charts against best-practice checks, is CI/CD-friendly, and can fail builds on invalid resources (`--fail-on-invalid-resource`).
- **Container image scanning** (Trivy or equivalent) — covers the "container images" part of the objective; Kubesec/KubeLinter/kube-score/Checkov do not scan image layers themselves.
- **kube-score / Checkov** — still useful, still appear in real-world pipelines and possibly in scenario questions, but are not the tools this specific objective names — don't confuse them with the primary two.
- **kube-hunter** — remember it's a *live cluster* runtime tool, not static analysis; don't file it under this domain on the exam.
