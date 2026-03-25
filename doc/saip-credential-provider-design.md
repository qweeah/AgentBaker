# Supporting Beta Credential Provider Versions in AgentBaker

## Problem

SAIP (Service Account Image Pull) requires a specific **beta** version of `azure-acr-credential-provider` (e.g. `1.34.2-beta.1-ubuntu24.04u1`) instead of the latest stable version (e.g. `1.34.3-ubuntu24.04u1`).

The existing `getLatestPkgVersionFromK8sVersion` function always picks the **highest** version matching `major.minor`. Since `1.34.3 > 1.34.2`, the stable version always wins. We need a way to select the beta version when SAIP is enabled.

Both options require adding a beta version entry to `components.json` under the existing `azure-acr-credential-provider-pmc` component.

---

## Option 1: Add `useBeta` flag

### How it works

1. `cse_config.sh` passes `useBeta=true` when SAIP is enabled
2. `installCredentialProviderFromPkg` forwards the flag to `getLatestPkgVersionFromK8sVersion`
3. The function loops through versions matching `major.minor`; when `useBeta=true`, it skips non-beta versions and picks the first version containing `"beta"`
4. components.json uses normal `"k8sVersion": "1.34"` — no special tag

### components.json change

A new entry must be added to each OS/release `versionsV2` array under the existing `azure-acr-credential-provider-pmc` component. The entry uses the same `k8sVersion` as the stable version (e.g. `"1.34"`), with `"<DO_NOT_UPDATE>"` to prevent Renovate from auto-bumping it, and the beta version string as `latestVersion`.

The `renovateTag` should be set to `"<DO_NOT_UPDATE>"` initially. Once the beta package is published to PMC, it can be changed to the appropriate renovateTag (e.g. `repository=production` or `repository=test`) to enable Renovate auto-updates. See [pmc-renovate-components.md](pmc-renovate-components.md) for details on how `renovateTag` maps to PMC repositories.

This entry must be added for **every supported OS/release**: Ubuntu 24.04 (`r2404`), Ubuntu 22.04 (`r2204`), Ubuntu 20.04 (`r2004`), and AzureLinux 3.0 (`v3.0`). The example below shows Ubuntu 24.04 only:

```json
{
  "k8sVersion": "1.34",
  "renovateTag": "<DO_NOT_UPDATE>",
  "latestVersion": "1.34.2-beta.1-ubuntu24.04u1"
}
```

This entry sits alongside the existing stable entry with the same `k8sVersion: "1.34"`. The `versionsV2` array allows duplicate `k8sVersion` values (it's a list, not a map).

### Key code changes

#### cse_config.sh — pass `true` flag for SAIP

```diff
         elif [ "$(type -t installCredentialProviderFromPkg)" = function ]; then
-            logs_to_events "AKS.CSE.ensureKubelet.installCredentialProviderFromPkg" "installCredentialProviderFromPkg ${KUBERNETES_VERSION}"
+            if [ "${SERVICE_ACCOUNT_IMAGE_PULL_ENABLED}" = "true" ]; then
+                # For SAIP-enabled clusters, install the beta credential provider binary
+                logs_to_events "AKS.CSE.ensureKubelet.installCredentialProviderFromPkg.saip" "installCredentialProviderFromPkg ${KUBERNETES_VERSION} true"
+            else
+                logs_to_events "AKS.CSE.ensureKubelet.installCredentialProviderFromPkg" "installCredentialProviderFromPkg ${KUBERNETES_VERSION}"
+            fi
```

#### cse_install_ubuntu.sh / cse_install_mariner.sh — forward useBeta

```diff
 installCredentialProviderFromPkg() {
     k8sVersion="${1:-}"
+    local useBeta="${2:-}"
     os=${UBUNTU_OS_NAME}
     ...
-    getLatestPkgVersionFromK8sVersion "$k8sVersion" "azure-acr-credential-provider-pmc" "$os" "$os_version" "${OS_VARIANT}"
+    getLatestPkgVersionFromK8sVersion "$k8sVersion" "azure-acr-credential-provider-pmc" "${useBeta}" "$os" "$os_version" "${OS_VARIANT}"
```

#### cse_helpers.sh — select beta when flag is set

```diff
 getLatestPkgVersionFromK8sVersion() {
     local k8sVersion="$1"
     local componentName="$2"
-
-    k8sMajorMinorVersion="$(echo "$k8sVersion" | cut -d- -f1 | cut -d. -f1,2)"
-
-    package=$(jq ".Packages" "$COMPONENTS_FILEPATH" | jq ".[] | select(.name == \"${componentName}\")")
-    updatePackageVersions "${package}" "${@:3}"
+    local useBeta="${3:-}"
+
+    k8sMajorMinorVersion="$(echo "$k8sVersion" | cut -d- -f1 | cut -d. -f1,2)"
+
+    package=$(jq ".Packages" "$COMPONENTS_FILEPATH" | jq ".[] | select(.name == \"${componentName}\")")
+    updatePackageVersions "${package}" "${@:4}"
     ...
     for version in "${sortedPackageVersions[@]}"; do
         majorMinorVersion="$(echo "$version" | cut -d- -f1 | cut -d. -f1,2)"
         if [ "$majorMinorVersion" = "$k8sMajorMinorVersion" ]; then
-            PACKAGE_VERSION=$version
-            break
+            if [ "${useBeta}" = "true" ]; then
+                if [[ "$version" == *beta* ]]; then
+                    PACKAGE_VERSION=$version
+                    break
+                fi
+            else
+                PACKAGE_VERSION=$version
+                break
+            fi
         fi
     done
```

### Version selection trace (k8s 1.34, Ubuntu 24.04)

Sorted versions (descending): `1.34.3`, `1.34.2-beta.1`, `1.34.1`, `1.33.6`, `1.32.11`

| Scenario | Loop behavior | Result |
|---|---|---|
| **Non-SAIP** (`useBeta` empty) | `1.34.3` matches `1.34` → picks immediately | `1.34.3-ubuntu24.04u1` |
| **SAIP** (`useBeta = "true"`) | `1.34.3` matches `1.34` but no `beta` → skip; `1.34.2-beta.1` matches `1.34` AND contains `beta` → picks | `1.34.2-beta.1-ubuntu24.04u1` |

### Pros & Cons

| Pros | Cons |
|---|---|
| components.json looks completely natural — no fake k8s versions | Requires modifying `getLatestPkgVersionFromK8sVersion` signature (adding `$3`) |
| No impact on Renovate — `"<DO_NOT_UPDATE>"` skips it | Must update `installCredentialProviderFromPkg` in both mariner and ubuntu to forward `useBeta` |
| The `"beta"` in version string is the natural discriminator | `${@:4}` shift for OS args — must verify all callers pass args correctly |
| Doesn't affect other components (flag defaults to empty) | Couples "beta" as a keyword convention |

---

## Option 2: Use `k8sVersion: "1.34-beta"` in components.json

### How it works

1. `cse_config.sh` passes `installCredentialProviderFromPkg 1.34-beta` (synthetic k8s version)
2. `getLatestPkgVersionFromK8sVersion` is modified to filter by `k8sVersion` JSON field when the input doesn't match version strings by `major.minor`
3. components.json uses `"k8sVersion": "1.34-beta"` for the beta entry

### components.json change

```json
{
  "k8sVersion": "1.34-beta",
  "renovateTag": "<DO_NOT_UPDATE>",
  "latestVersion": "1.34.2-beta.1-ubuntu24.04u1"
}
```

### Key code change

#### cse_helpers.sh — filter by k8sVersion JSON field

```diff
 getLatestPkgVersionFromK8sVersion() {
     ...
+    # When input contains a non-numeric suffix (e.g. "1.34-beta"),
+    # filter versionsV2 by the k8sVersion JSON field directly
+    if [[ "$k8sVersion" == *-* ]] && ! [[ "$k8sVersion" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
+        local packageJSON
+        packageJSON=$(getPackageJSON "${package}" "${@:3}")
+        PACKAGE_VERSION=$(jq -r ".versionsV2[] | select(.k8sVersion == \"${k8sVersion}\") | .latestVersion // empty" <<< "${packageJSON}" | head -1)
+        echo "$PACKAGE_VERSION"
+        return 0
+    fi
     ...
```

### Pros & Cons

| Pros | Cons |
|---|---|
| Clear intent — `"1.34-beta"` is self-documenting | **Synthetic k8sVersion** — not a real k8s version, hard to justify to components.json owners |
| Explicit JSON field matching, not version string pattern | Still requires modifying `getLatestPkgVersionFromK8sVersion` |
| Could support multiple variants (e.g. `1.34-saip`, `1.34-fips`) | Changes semantics of `k8sVersion` from "real k8s major.minor" to "arbitrary tag" |
| | VHD download logs show `"1.34-beta"` which looks odd |
