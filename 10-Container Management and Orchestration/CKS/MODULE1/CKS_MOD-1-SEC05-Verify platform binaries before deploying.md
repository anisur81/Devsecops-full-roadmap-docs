# How to Verify Kubernetes Platform Binaries

Verifying platform binaries before deployment is a critical security step that ensures the downloaded files are authentic and haven't been tampered with during transit. This is especially important when downloading binaries from the internet, as an attacker could intercept requests and replace genuine files with malicious ones .

## Step-by-Step Binaries Verification Process

 
 
Lab Topology:

```
| Component  | Version                            |
| ---------- | ---------------------------------- |
| Ubuntu     | 24.04/26.04                        |
| Kubernetes | v1.36.x (or latest available)      |
| Internet   | Required                           |
| Tool       | curl, sha256sum, sha512sum, cosign |

```


### Lab 1 – Verify Using SHA256
#### Step 1 Download kubectl

```
mkdir ~/verify-lab
cd ~/verify-lab

curl -LO https://dl.k8s.io/release/v1.36.0/bin/linux/amd64/kubectl
```

#### Step 2 Download SHA256 checksum
```
curl -LO https://dl.k8s.io/release/v1.36.0/bin/linux/amd64/kubectl.sha256
```
#### Step 3 Verify checksum

```
echo "$(cat kubectl.sha256) kubectl" | sha256sum --check
Expected output
kubectl: OK

```

If you see FAILED the binary must not be used.


#### Lab 2 – Verify Using SHA512

Download Kubernetes Server package.

```
wget https://dl.k8s.io/v1.36.0/kubernetes-server-linux-amd64.tar.gz
```

Download SHA512 value from the Kubernetes release page.

Verify

sha512sum kubernetes-server-linux-amd64.tar.gz

Compare the output with the published SHA512.

If identical Safe to install

Otherwise DO NOT INSTALL

SHA verification detects accidental corruption and unauthorized modification of downloaded files.


 #### Lab 3 – Verify Kubernetes Binary Signature Using Cosign

Modern Kubernetes releases are signed with Cosign.

Install Cosign

```
sudo apt update
sudo snap install cosign
```
Download Binary
```
URL=https://dl.k8s.io/release/v1.36.0/bin/linux/amd64

BINARY=kubectl

curl -LO $URL/$BINARY
curl -LO $URL/$BINARY.sig
curl -LO $URL/$BINARY.cert
```

Verify Signature
```
cosign verify-blob kubectl \
  --signature kubectl.sig \
  --certificate kubectl.cert \
  --certificate-identity krel-staging@k8s-releng-prod.iam.gserviceaccount.com \
  --certificate-oidc-issuer https://accounts.google.com

Expected output

Verified OK
```

This confirms that: the binary is authentic, it was signed by the Kubernetes release process, it has not been altered since signing.

#### Lab 4 – Verify Kubernetes Container Images

Verify a control plane image.
```
cosign verify registry.k8s.io/kube-apiserver-amd64:v1.36.0 \
  --certificate-identity krel-trust@k8s-releng-prod.iam.gserviceaccount.com \
  --certificate-oidc-issuer https://accounts.google.com
```

Successful verification confirms the image is an authentic Kubernetes release image.

#### Lab 5 – Simulate a Tampered Binary

Download the binary.
```
curl -LO https://dl.k8s.io/release/v1.36.0/bin/linux/amd64/kubectl

Modify it.

echo "Hacked" >> kubectl

Run verification again.

echo "$(cat kubectl.sha256) kubectl" | sha256sum --check

Output

kubectl: FAILED
```

The checksum no longer matches because the file contents changed.

#### Lab 6 – Verify Before Cluster Installation

Example workflow before installing Kubernetes.

```
Download binary
↓
Download checksum
↓
Verify checksum
↓
Verify Cosign signature
↓
Install binary
↓
Create Kubernetes cluster
```

This prevents installing compromised software.

Real-World DevSecOps CI/CD Example

```
GitHub Release
        ↓
Download kubectl
        ↓
Verify SHA256
        ↓
Verify Cosign Signature
        ↓
Store in Artifact Repository
        ↓
Deploy to Kubernetes Cluster
```
Many organizations automate these verification steps in CI/CD pipelines before promoting artifacts to production.


#### CKS Exam Tips
1. Verify downloaded Kubernetes binaries before installation.
2. Know how to use sha256sum and sha512sum.
3. Understand that Kubernetes release artifacts are signed with Cosign.
4. Treat any checksum mismatch or failed signature verification as a reason not to deploy.
5. Prefer deploying container images by immutable digest (sha256:...) rather than mutable tags in production.
