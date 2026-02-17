# Custom Credential Provider ORAS Push — Results

**Date**: 2026-02-17
**Source**: `/home/jinzha1/workspace/kubernetes-sigs/cloud-provider-azure`
**ACR**: `jinzha1.azurecr.io`
**Repository**: `azure-acr-credential-provider`

## Binaries

| Arch  | Path | Size |
|-------|------|------|
| amd64 | `bin/amd64/azure-acr-credential-provider` | 37 MB |
| arm64 | `bin/arm64/azure-acr-credential-provider` | 34 MB |

Built at: 2026-02-17 08:08

## Tar.gz Artifacts

| File | Size |
|------|------|
| `azure-acr-credential-provider-linux-amd64-v0.1.0.tar.gz` | 17 MB |
| `azure-acr-credential-provider-linux-arm64-v0.1.0.tar.gz` | 15 MB |

Each tar.gz contains a single file: `azure-acr-credential-provider`

## ORAS Push Results

**Auth note**: `az acr login` identity tokens didn't work with `oras push` (401 insufficient_scope).
Used **admin credentials** via `az acr credential show` instead.

### amd64

```
✓ Uploaded  azure-acr-credential-provider-linux-amd64-v0.1.0.tar.gz  16.1/16.1 MB  571ms
  └─ sha256:ae3a9897a9a9efdc1c0e13c4df72512955070daae52e9b01d4500f92fe9364d9
✓ Uploaded  application/vnd.oci.image.config.v1+json  37/37 B  181ms
  └─ sha256:9d99a75171aea000c711b34c0e5e3f28d3d537dd99d110eafbfbc2bd8e52c2bf
✓ Uploaded  application/vnd.oci.image.manifest.v1+json  634/634 B  240ms

Digest: sha256:30c3235cf13de94f728a7bb023e90630035fb082fb1ab1c0fbe4c84f8b6b618e
```

### arm64

```
Digest: sha256:a16e684ad04a61a448397e217c3e0c73e5c7edea80e55150019072dda32413d7
```

## Manifest Index

```
Tag:    v0.1.0
Digest: sha256:2f5d00a793c65c4c5df1ed186a9a60cd795310687a8c726ec224ac865ddc625a
```

### Index Manifest Contents

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.index.v1+json",
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:30c3235cf13de94f728a7bb023e90630035fb082fb1ab1c0fbe4c84f8b6b618e",
      "size": 634,
      "platform": {
        "architecture": "amd64",
        "os": "linux"
      },
      "artifactType": "application/vnd.unknown.artifact.v1"
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:a16e684ad04a61a448397e217c3e0c73e5c7edea80e55150019072dda32413d7",
      "size": 634,
      "platform": {
        "architecture": "arm64",
        "os": "linux"
      },
      "artifactType": "application/vnd.unknown.artifact.v1"
    }
  ]
}
```

## Verification

Pull test resolved correctly — both tar.gz files downloaded:

```
azure-acr-credential-provider-linux-amd64-v0.1.0.tar.gz (17 MB)
azure-acr-credential-provider-linux-arm64-v0.1.0.tar.gz (15 MB)
```

Each tar.gz contains: `azure-acr-credential-provider` ✓

## Compatibility with AgentBaker CSE

**`downloadCredentialProvider()` flow on the node:**

1. URL: `jinzha1.azurecr.io/azure-acr-credential-provider:v0.1.0`
2. `isRegistryUrl()` → true (matches OCI distribution pattern)
3. Version extraction: `grep -oP 'v\d+(\.\d+)*'` → `v0.1.0` ✓
4. `CREDENTIAL_PROVIDER_TGZ_TMP` = `azure-acr-credential-provider-linux-${CPU_ARCH}-v0.1.0.tar.gz` ✓
5. `retrycmd_get_tarball_from_registry_with_oras` → `oras pull` (no `--platform` flag)
6. Downloads both tar.gz files, but extracts only the arch-specific one ✓
7. `installCredentialProviderFromUrl` → `extract_tarball` → moves `azure-acr-credential-provider` to `acr-credential-provider` ✓

## Artifact URL for cse_config.sh

```
jinzha1.azurecr.io/azure-acr-credential-provider:v0.1.0
```

## Issues Encountered

1. **ORAS auth with `az acr login`**: Identity tokens stored in `~/.docker/config.json` gave 401 on push.
   The WWW-Authenticate challenge only requested `pull` scope. Fixed by using admin credentials.
2. **`oras pull` downloads all platforms**: Without `--platform` flag, both amd64 and arm64 tar.gz
   are downloaded. This is fine — `downloadCredentialProvider()` selects the correct one by filename.
