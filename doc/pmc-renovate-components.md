# PMC, Renovate, and components.json

## What is PMC?

PMC stands for **P**ackages.**M**icrosoft.**C**om (`packages.microsoft.com`) — a colloquial abbreviation of the domain name. It is Microsoft's Linux package hosting service. It hosts deb packages for Ubuntu and RPM packages for Azure Linux / Mariner. AKS components like kubelet, kubectl, containerd, runc, and azure-acr-credential-provider are published here.

## PMC Repositories (Distributions)

Within a single OS release on PMC, packages are published to different **distributions** (dists). These are separate package indexes that can contain different versions of the same package.

For Ubuntu 24.04, the URL structure is:

```
https://packages.microsoft.com/ubuntu/24.04/prod/dists/{dist}/main/binary-{arch}/Packages
```

| Distribution | URL dist value | Purpose |
|---|---|---|
| **Production** | `noble` (24.04), `jammy` (22.04), `focal` (20.04) | Stable, GA packages |
| **Testing** | `testing` | Pre-release, beta, or packages being validated before promotion to production |

For example:
- **Production**: `https://packages.microsoft.com/ubuntu/24.04/prod/dists/noble/main/binary-amd64/Packages`
- **Testing**: `https://packages.microsoft.com/ubuntu/24.04/prod/dists/testing/main/binary-amd64/Packages`

### How to check what's available

```bash
# List all azure-acr-credential-provider versions in production
curl -s "https://packages.microsoft.com/ubuntu/24.04/prod/dists/noble/main/binary-amd64/Packages" | grep -A1 "Package: azure-acr-credential-provider"

# List all azure-acr-credential-provider versions in testing
curl -s "https://packages.microsoft.com/ubuntu/24.04/prod/dists/testing/main/binary-amd64/Packages" | grep -A1 "Package: azure-acr-credential-provider"

# Search for beta versions in testing
curl -s "https://packages.microsoft.com/ubuntu/24.04/prod/dists/testing/main/binary-amd64/Packages" | grep -B1 "beta"
```

For Azure Linux RPMs, the repos are:
- **Production**: `https://packages.microsoft.com/azurelinux/3.0/prod/cloud-native/x86_64/repodata`
- **Base**: `https://packages.microsoft.com/azurelinux/3.0/prod/base/x86_64/repodata`

## The `renovateTag` Field in components.json

The `renovateTag` field tells [Renovate](https://docs.renovatebot.com/) (the auto-update bot) **where to look** for newer versions. It is metadata only — it does NOT affect VHD build or node provisioning. Those use the `latestVersion` string directly.

### Who writes `renovateTag`?

**You do, manually.** When you add a new component or version entry to `components.json`, you write the `renovateTag` by hand following the format conventions below. Renovate reads this tag to know which PMC repository/registry to poll for updates.

- The `renovateTag` is **not generated** by any tool — it's a human-authored string.
- The Renovate configuration in `.github/renovate.json` defines **regex custom managers** that parse `renovateTag` values from `components.json` and map them to datasources (PMC URLs, container registries, etc.).
- When Renovate finds a newer version in the datasource, it automatically opens a PR to update `latestVersion` (and rotate `previousLatestVersion`).
- If you set `"renovateTag": "<DO_NOT_UPDATE>"`, Renovate ignores the entry and you must update `latestVersion` manually.

For the full Renovate configuration details, see [.github/README-RENOVATE.md](../.github/README-RENOVATE.md).

### Format

```json
"renovateTag": "name={package_name}, repository={repo}, os={os}, release={release}"
```

| Field | Description | Example |
|---|---|---|
| `name` | Package name in PMC | `azure-acr-credential-provider`, `moby-containerd` |
| `repository` | PMC distribution to check | `production` or `test` |
| `os` | Operating system | `ubuntu`, `azurelinux`, `mariner` |
| `release` | OS version | `24.04`, `22.04`, `20.04`, `3.0` |

### Mapping to Renovate Custom Datasources

The `repository` value maps to Renovate custom datasources defined in `.github/renovate.json`:

```
repository=production, os=ubuntu, release=24.04
  → datasource: custom.deb2404
  → URL: https://packages.microsoft.com/ubuntu/24.04/prod/dists/noble/main/binary-amd64/Packages

repository=test, os=ubuntu, release=24.04
  → datasource: custom.deb2404-test
  → URL: https://packages.microsoft.com/ubuntu/24.04/prod/dists/testing/main/binary-amd64/Packages
```

### Special Values

| Value | Meaning |
|---|---|
| `"<DO_NOT_UPDATE>"` | Renovate will skip this entry — version must be updated manually |
| `"RPM_registry=..."` | Used for Azure Linux RPM packages, points to the RPM repodata URL |
| `"OCI_registry=..."` | Used for OCI artifacts hosted on MCR (e.g. kubernetes-binaries) |
| `"github-releases=..."` | Used for packages distributed via GitHub releases |

### Example: containerd on Ubuntu 24.04 uses `test` repo

```json
{
  "renovateTag": "name=moby-containerd, repository=test, os=ubuntu, release=24.04",
  "latestVersion": "2.1.6-ubuntu24.04u1"
}
```

This means containerd v2 for Ubuntu 24.04 is published to the **testing** distribution on PMC, not production. Renovate checks the testing endpoint for newer versions.

### Example: credential provider uses `production` repo

```json
{
  "renovateTag": "name=azure-acr-credential-provider, repository=production, os=ubuntu, release=24.04",
  "latestVersion": "1.34.3-ubuntu24.04u1"
}
```

Renovate checks the **production** (noble) endpoint for newer versions.

### Example: manually managed version (no auto-update)

```json
{
  "renovateTag": "<DO_NOT_UPDATE>",
  "latestVersion": "1.34.2-beta.1-ubuntu24.04u1"
}
```

Renovate ignores this entry entirely. The version must be updated by hand.

## Summary

- `renovateTag` = **where Renovate looks** for new versions (auto-update metadata)
- `repository=production` = stable PMC dist (`noble`, `jammy`, `focal`)
- `repository=test` = pre-release PMC dist (`testing`)
- `latestVersion` = **what gets installed** on VHDs and nodes (the actual version used)
- `<DO_NOT_UPDATE>` = Renovate won't touch it, manual updates only
