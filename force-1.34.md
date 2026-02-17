# Force Custom Credential Provider for k8s 1.34

## Background

For k8s >= 1.34, AgentBaker's `cse_config.sh` takes the **PMC path** in
`configureKubeletAndKubectl()` — it installs the credential provider via
`apt-get`/`tdnf` using versions from `components.json`, **not** from
`CREDENTIAL_PROVIDER_DOWNLOAD_URL`.

Your override block in `ensureKubelet()` runs *after*
`configureKubeletAndKubectl()` and calls `installCredentialProviderFromUrl`
directly, which overwrites whatever PMC installed. So no RP code change is
needed — the override handles everything.

## What to change

In `parts/linux/cloud-init/artifacts/cse_config.sh`, replace the
`REPLACE_ME` placeholders in the override block (~line 766):

```bash
# Before (placeholders):
CREDENTIAL_PROVIDER_DOWNLOAD_URL="REPLACE_ME.azurecr.io/azure-acr-credential-provider:REPLACE_ME_TAG-linux-${CPU_ARCH}"

# After (your ACR):
CREDENTIAL_PROVIDER_DOWNLOAD_URL="jinzha1.azurecr.io/azure-acr-credential-provider:v0.1.0"
```

Note: **remove** the `-linux-${CPU_ARCH}` suffix. The ORAS multi-platform
manifest auto-resolves architecture. `downloadCredentialProvider()` already
detects registry URLs (contains `azurecr.io` or `/`) and uses
`downloadFromOrasUrl` which handles platform resolution.

## Verify the flow

1. `ensureKubelet()` → checks `KUBELET_FLAGS` for `image-credential-provider-config`
2. Enters the override block → sets `CREDENTIAL_PROVIDER_DOWNLOAD_URL`
3. Calls `installCredentialProviderFromUrl` → calls `downloadCredentialProvider()`
4. `downloadCredentialProvider()` sees a registry URL → uses `downloadFromOrasUrl`
5. ORAS pulls the arch-specific tar.gz → extracts `azure-acr-credential-provider`
6. Renames to `acr-credential-provider` in `/var/lib/kubelet/credential-provider/`

## Regenerate and commit

```bash
cd ~/workspace/azure/AgentBaker
make generate
git add -A
git commit --amend --no-gpg-sign
```

## No RP change needed

The RP's `GetCredentialProviderURL()` will return the default 1.33.3 path for
k8s 1.34 (via the `default` switch case). The RP sets
`CREDENTIAL_PROVIDER_DOWNLOAD_URL` to a blob storage URL. But this is
irrelevant — your `cse_config.sh` override **replaces** it before calling
`installCredentialProviderFromUrl`.
