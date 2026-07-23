#   Minimize Base Image Footprint (Hands-on Practice)

You will learn how to:

* Compare large vs small images
* Build secure Docker images
* Use Alpine images
* Use Distroless images
* Use Multi-stage builds
* Remove unnecessary packages
* Scan images
* Verify image size and attack surface

 
 # Lab: Minimizing Docker Base Image Footprint

## Objective
Learn practical techniques to reduce Docker image size — smaller images pull faster, deploy faster, and have a smaller attack surface. You will build the *same* simple Node.js app four different ways, measure the image size at each step, and see the size drop from **~1.1 GB down to ~50 MB**.

## Prerequisites
- Docker installed (`docker --version`)
- Basic familiarity with Dockerfiles
- ~15–20 minutes

---

## Lab Setup

Create a working directory and a trivial app to containerize.

```bash
mkdir docker-slim-lab && cd docker-slim-lab
```

**`app.js`**
```javascript
const http = require('http');

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('Hello from a slim container!\n');
});

server.listen(3000, () => console.log('Server running on port 3000'));
```

**`package.json`**
```json
{
  "name": "slim-lab-app",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  }
}
```

Keep these two files in the directory for every step below — only the `Dockerfile` changes.

---

## Step 1: The Naive Approach (Baseline)

**`Dockerfile.v1`**
```dockerfile
FROM node
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "app.js"]
```

Build and measure:
```bash
docker build -t slim-lab:v1 -f Dockerfile.v1 .
docker images slim-lab:v1
```

**Expected result:** ~1.0–1.1 GB. `node` (no tag) pulls the full Debian-based image with build tools, docs, and package managers you don't need at runtime.

---

## Step 2: Switch to a Slim Base Image

**`Dockerfile.v2`**
```dockerfile
FROM node:20-slim
WORKDIR /app
COPY . .
RUN npm install --omit=dev
CMD ["node", "app.js"]
```

```bash
docker build -t slim-lab:v2 -f Dockerfile.v2 .
docker images slim-lab:v2
```

**Expected result:** ~150–180 MB. `slim` variants strip out documentation, man pages, and many CLI tools not needed to *run* the app. `--omit=dev` also skips devDependencies.

---

## Step 3: Go Further with Alpine

**`Dockerfile.v3`**
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json ./
RUN npm install --omit=dev
COPY app.js ./
CMD ["node", "app.js"]
```

```bash
docker build -t slim-lab:v3 -f Dockerfile.v3 .
docker images slim-lab:v3
```

**Expected result:** ~130–140 MB. Alpine uses `musl libc` and BusyBox instead of glibc/coreutils, shrinking the base OS layer to ~5 MB before Node is even added.

> **Note:** Copying `package.json` before the rest of the source (Step 3) is also a caching optimization — Docker only re-runs `npm install` when dependencies actually change, not on every code edit.

---

## Step 4: Multi-Stage Build (Biggest Win)

Separate the **build environment** (compilers, dev dependencies, npm cache) from the **runtime environment** (just the app + Node runtime).

**`Dockerfile.v4`**
```dockerfile
# ---- Stage 1: Build ----
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json ./
RUN npm install --omit=dev
COPY app.js ./

# ---- Stage 2: Runtime ----
FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app /app
# Run as non-root for security
USER node
CMD ["node", "app.js"]
```

```bash
docker build -t slim-lab:v4 -f Dockerfile.v4 .
docker images slim-lab:v4
```

**Expected result:** ~130 MB (for this trivial app, similar to v3 since there's no real "build" step — but for apps with compilers, TypeScript, native modules, or large build toolchains, multi-stage builds are where you see the *biggest* drop, e.g., 900 MB → 100 MB).

---

## Step 5 (Bonus): Distroless / Minimal Runtime

For maximum reduction, drop the OS package manager and shell entirely using Google's **distroless** images — only the language runtime and your app remain.

**`Dockerfile.v5`**
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json ./
RUN npm install --omit=dev
COPY app.js ./

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=builder /app /app
CMD ["app.js"]
```

```bash
docker build -t slim-lab:v5 -f Dockerfile.v5 .
docker images slim-lab:v5
```

**Expected result:** ~50–70 MB, with no shell, no package manager, no `apt`/`apk` — meaningfully reducing the attack surface (a compromised container can't `curl` a payload or spawn a shell).

---

## Step 6: Scan Images

Install Trivy

```bash
sudo apt install wget apt-transport-https gnupg -y

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/trivy.gpg >/dev/null

echo \
"deb [signed-by=/usr/share/keyrings/trivy.gpg] \
https://aquasecurity.github.io/trivy-repo/deb generic main" \
| sudo tee /etc/apt/sources.list.d/trivy.list

sudo apt update

sudo apt install trivy
```

Scan Ubuntu image

```
Scan Ubuntu Image
trivy image flask:bad

Example output

Total: 50+

CRITICAL: 8

HIGH: 42

MEDIUM: 60

LOW: 30 
```

---
## Step 7: Compare All Results

```bash
docker images slim-lab
```

| Tag | Base Image        | Approx. Size | Technique                         |
|-----|--------------------|--------------|-------------------------------------|
| v1  | `node`             | ~1.1 GB      | None (naive baseline)              |
| v2  | `node:20-slim`     | ~170 MB      | Slim base + prod-only deps         |
| v3  | `node:20-alpine`   | ~135 MB      | Alpine base + layer caching        |
| v4  | `node:20-alpine`   | ~130 MB      | Multi-stage build                  |
| v5  | `distroless`       | ~55 MB       | Multi-stage + distroless runtime   |

Run this to visualize the layers of any image and spot where the bloat is:
```bash
docker history slim-lab:v1
docker history slim-lab:v5
```

---

## Additional Techniques to Practice

1. **`.dockerignore`** — prevents `node_modules`, `.git`, and local build artifacts from bloating the build context. Try creating one:
   ```
   node_modules
   .git
   *.log
   Dockerfile*
   ```

2. **Combine `RUN` layers** to avoid leaving cache/apt-list residue in intermediate layers:
   ```dockerfile
   RUN apt-get update && apt-get install -y curl \
       && rm -rf /var/lib/apt/lists/*
   ```

3. **Pin exact versions** (`node:20.11.1-alpine3.19` instead of `node:latest`) for reproducibility and to avoid surprise size changes on rebuild.

4. **Scan for further slimming opportunities** with a tool like `dive`:
   ```bash
   dive slim-lab:v4
   ```

---

## Cleanup

```bash
docker rmi slim-lab:v1 slim-lab:v2 slim-lab:v3 slim-lab:v4 slim-lab:v5
```

## Key Takeaways
- **Base image choice** is the single biggest lever (full → slim → alpine → distroless).
- **Multi-stage builds** let you use heavy build tools without shipping them.
- **Layer ordering** affects build cache efficiency, not just final size.
- **Smaller images** = faster pulls/deploys, smaller attack surface, lower storage/bandwidth cost.
---

 
