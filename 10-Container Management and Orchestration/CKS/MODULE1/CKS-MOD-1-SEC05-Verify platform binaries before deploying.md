## How to Verify Kubernetes Platform Binaries

Verifying platform binaries before deployment is a critical security step that ensures the downloaded files are authentic and haven't been tampered with during transit . This is especially important when downloading binaries from the internet, as an attacker could intercept requests and replace genuine files with malicious ones .

### Step-by-Step Verification Process

#### 1. Download the Binary

First, download the Kubernetes binary you want to verify. For example, to download the `kubernetes.tar.gz` file for version 1.26.0:

```bash
curl https://dl.k8s.io/v1.26.0/kubernetes.tar.gz -L -o kubernetes.tar.gz
```

#### 2. Generate the Checksum

After downloading, generate the SHA-512 checksum of the file. On Linux, use:

```bash
sha512sum kubernetes.tar.gz
```

On macOS, you can use:

```bash
shasum -a 512 kubernetes.tar.gz
```

#### 3. Compare with the Official Hash

Compare the generated hash with the official one from the Kubernetes release notes. The SHA-512 checksums for each release are listed in the CHANGELOG file on GitHub.

For **Kubernetes v1.26.0**, the official SHA-512 hash for `kubernetes.tar.gz` is :

```
3062a427a45548bd9c5a8358c740f0a5cfea7b546dca724c71d28768bb36c628280c91263a362afd01c89ef3944f5a768ed44e75d421fe9dc1ec2e8ba26214f3
```

**Important Note:** The hash varies by version. For example, **v1.26.1** has a different hash (`65d8e6456b48737e496bd7e57ed630825654a1527de52890481be5f6df89a88675b809e6cb0ffa75b71c7d723ee64145a92b859afff296c7ee6f077f66c05c48`) , and **v1.26.3** has yet another (`d1c6c8c9844ec63c043138bed0f01a0129d89c9c7cc130b794db0e1abea0062f4b2fe78ec7904cb32a1d628b6f5fb6f72c8374197e415f21ae3afd41fc82e0be`) .

### Finding the Correct Hash

To find the correct hash for your version:

1. Navigate to the Kubernetes CHANGELOG on GitHub:
   - CHANGELOG-1.26.md: `https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.26.md`
   - CHANGELOG-1.27.md: `https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.27.md`

2. Locate the section for your specific patch version (e.g., `# v1.26.3`)

3. Find the SHA-512 hash under the "Source Code" section for `kubernetes.tar.gz`

### Verify the Hash

**If the hash matches:**
```
kubectl: OK
```
The file is verified and safe to use .

**If the hash does NOT match:**
```
kubernetes.tar.gz: FAILED
sha512sum: WARNING: 1 computed checksum did NOT match
```
The file may have been corrupted during download or tampered with, and should not be used .

### Additional Verification Methods

#### Using Checksum Files

For specific binaries like `kubectl`, you can download a separate `.sha256` checksum file and validate automatically :

```bash
# Download the binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Download the checksum file
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

# Verify
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
```

#### Using Cosign (Keyless Signing)

Kubernetes 1.26 and later supports verification using **cosign's keyless signing** for all binary artifacts :

```bash
# Download binary, signature, and certificate
URL=https://dl.k8s.io/release/v1.32.0/bin/linux/amd64
BINARY=kubectl

for FILE in "$BINARY" "$BINARY.sig" "$BINARY.cert"; do
    curl -sSfL --retry 3 --retry-delay 3 "$URL/$FILE" -o "$FILE"
done

# Verify the binary
cosign verify-blob "$BINARY" \
  --signature "$BINARY".sig \
  --certificate "$BINARY".cert \
  --certificate-identity krel-staging@k8s-releng-prod.iam.gserviceaccount.com \
  --certificate-oidc-issuer https://accounts.google.com
```

### Important Considerations

- **Use the same version** for the binary and checksum file when verifying 
- **Kubernetes 1.26** is now end-of-life (EOL) as of December 2023, with security fixes ending February 2024 . Consider using a supported version for production deployments.
- **Always verify** before deploying to ensure the integrity and authenticity of your platform binaries
