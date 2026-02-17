# Custom Credential Provider — Build & Push Setup

## Prerequisites

- [ORAS CLI](https://oras.land/docs/installation) v1.1+ installed locally (v1.3.0 on devbox)
- An Azure Container Registry you have push access to
- Go toolchain for cross-compilation (already done)

## 1. Build binaries (already done)

```bash
GOOS=linux GOARCH=amd64 go build -o bin/amd64/azure-acr-credential-provider ./cmd/acr-credential-provider/
GOOS=linux GOARCH=arm64 go build -o bin/arm64/azure-acr-credential-provider ./cmd/acr-credential-provider/
```

## 2. Package as tar.gz

The tar.gz **filename must match** what `downloadCredentialProvider()` constructs:
`azure-acr-credential-provider-linux-${CPU_ARCH}-${cred_version}.tar.gz`
where `cred_version` is regex-extracted from the tag (`grep -oP 'v\d+(\.\d+)*'`).

Each tar.gz must contain a single binary named `azure-acr-credential-provider` at its root
(`installCredentialProviderFromUrl` extracts and renames it to `acr-credential-provider`).

```bash
VERSION=v0.1.0   # must be pure semver (no suffix) — this is what the regex extracts

tar czf azure-acr-credential-provider-linux-amd64-${VERSION}.tar.gz \
  -C bin/amd64 azure-acr-credential-provider

tar czf azure-acr-credential-provider-linux-arm64-${VERSION}.tar.gz \
  -C bin/arm64 azure-acr-credential-provider
```

## 3. Login to ACR

```bash
ACR=jinzha1
az acr login -n $ACR
```

## 4. Push per-platform artifacts with ORAS

Push each architecture's tar.gz **without a tag**, capturing the digest.
`--artifact-platform` embeds platform metadata so `oras pull` can auto-resolve.

```bash
REPO=${ACR}.azurecr.io/azure-acr-credential-provider

# Push amd64 tar.gz (capture digest)
DIGEST_AMD64=$(oras push --artifact-platform linux/amd64 \
  --artifact-type application/vnd.unknown.artifact.v1 \
  --format go-template='{{.digest}}' \
  ${REPO} \
  azure-acr-credential-provider-linux-amd64-${VERSION}.tar.gz)
echo "amd64 digest: ${DIGEST_AMD64}"

# Push arm64 tar.gz (capture digest)
DIGEST_ARM64=$(oras push --artifact-platform linux/arm64 \
  --artifact-type application/vnd.unknown.artifact.v1 \
  --format go-template='{{.digest}}' \
  ${REPO} \
  azure-acr-credential-provider-linux-arm64-${VERSION}.tar.gz)
echo "arm64 digest: ${DIGEST_ARM64}"
```

## 5. Create multi-platform manifest index

Create a manifest index from the two digests, tagged for consumption.

```bash
TAG=${VERSION}   # tag must contain the version so downloadCredentialProvider can regex-extract it

oras manifest index create ${REPO}:${TAG} \
  ${DIGEST_AMD64} ${DIGEST_ARM64}
```

After this, `oras pull ${REPO}:${TAG}` on an amd64 node auto-resolves to
`azure-acr-credential-provider-linux-amd64-v0.1.0.tar.gz`, matching what
`downloadCredentialProvider()` expects for `CREDENTIAL_PROVIDER_TGZ_TMP`.

## 6. Verify

```bash
# Check the index manifest
oras manifest fetch ${REPO}:${TAG} | jq .

# Verify pull resolves correctly (should download the arch-specific tar.gz)
mkdir -p /tmp/verify-cred && cd /tmp/verify-cred
oras pull ${REPO}:${TAG}
ls -l   # should show azure-acr-credential-provider-linux-amd64-v0.1.0.tar.gz
tar tzf azure-acr-credential-provider-linux-*.tar.gz
# should list: azure-acr-credential-provider
```

## 7. Update AgentBaker cse_config.sh

Since the multi-platform manifest auto-resolves architecture, use a
**single tag without `${CPU_ARCH}`**:

In `parts/linux/cloud-init/artifacts/cse_config.sh`, set:

```bash
CREDENTIAL_PROVIDER_DOWNLOAD_URL="jinzha1.azurecr.io/azure-acr-credential-provider:v0.1.0"
```

Then regenerate and amend:

```bash
make generate
git add -A
git commit --amend --no-gpg-sign
```

## 8. Tag and test via standalone RP

```bash
# Push to your AgentBaker fork
git push origin e2e --force

# Tag — format: v0.YYYYMMDD.<alias><index>
git tag v0.20260217.jinzha10
git push origin --tags
```

> **Tag size issue**: If `git push --tags` fails with 404 from goproxy, the source
> tree may exceed 500 MB. Delete `vhdbuilder/release-notes/` to reduce size:
> ```bash
> rm -rf vhdbuilder/release-notes/
> git add . && git commit --amend --no-edit --no-gpg-sign
> git tag -d v0.20260217.jinzha10 && git tag v0.20260217.jinzha10
> git push origin --tags --force
> ```

### Vendor into RP

```bash
cd ~/go/src/go.goms.io/aks/rp/
git checkout test-standalone
git checkout -b jinzha1/saip-cred-provider

# Vendor AgentBaker (~15-20 min)
make update_agent_baker
# Prompt 1 — GitHub username: qweeah
# Prompt 2 — Tag: v0.20260217.jinzha10

git add . && git commit -m "vendor custom AgentBaker for SAIP credential provider testing"
git push origin jinzha1/saip-cred-provider
```

### Deploy standalone

1. Go to [AKS Dev Deploy Pipeline](https://dev.azure.com/msazure/CloudNativeCompute/_build?definitionId=68881)
2. Start a new build off branch `jinzha1/saip-cred-provider`
3. Wait for completion (~1 hour)
4. Download `azureconfig.yaml` from build artifacts

### Verify on node

```bash
# Create cluster
./aksdev cluster create saip-test --wait

# SSH to node and verify:
kubectl node-shell <node-name>
ls -la /var/lib/kubelet/credential-provider/acr-credential-provider
/var/lib/kubelet/credential-provider/acr-credential-provider --version
cat /var/lib/kubelet/credential-provider-config.yaml
cat /var/log/azure/cluster-provision.log | grep -i "credential\|error\|fail"
```

## Quick reference — all-in-one script

```bash
#!/bin/bash
set -euo pipefail

ACR=jinzha1
VERSION=v0.1.0
REPO=${ACR}.azurecr.io/azure-acr-credential-provider

# Package as tar.gz (filenames must match downloadCredentialProvider convention)
tar czf azure-acr-credential-provider-linux-amd64-${VERSION}.tar.gz \
  -C bin/amd64 azure-acr-credential-provider
tar czf azure-acr-credential-provider-linux-arm64-${VERSION}.tar.gz \
  -C bin/arm64 azure-acr-credential-provider

# Login
az acr login -n $ACR

# Push per-arch (tagless — capture digests)
DIGEST_AMD64=$(oras push --artifact-platform linux/amd64 \
  --artifact-type application/vnd.unknown.artifact.v1 \
  --format go-template='{{.digest}}' \
  ${REPO} \
  azure-acr-credential-provider-linux-amd64-${VERSION}.tar.gz)

DIGEST_ARM64=$(oras push --artifact-platform linux/arm64 \
  --artifact-type application/vnd.unknown.artifact.v1 \
  --format go-template='{{.digest}}' \
  ${REPO} \
  azure-acr-credential-provider-linux-arm64-${VERSION}.tar.gz)

# Create multi-platform index
oras manifest index create ${REPO}:${VERSION} \
  ${DIGEST_AMD64} ${DIGEST_ARM64}

# Verify
oras manifest fetch ${REPO}:${VERSION} | jq .
echo "Done! Use: ${REPO}:${VERSION}"
```
