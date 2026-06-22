<!--
To submit this proposal:
1. Copy to kroxylicious/design repo as proposals/000-rendered-install-manifests.md
2. Open a PR
3. Rename file using PR number: proposals/<PR#>-rendered-install-manifests.md
4. Update heading to include PR number
5. Push changes
-->

# 115 - All-in-one Install YAML Manifests for GitOps

Publish all-in-one Kubernetes installation manifests as GitHub release assets to enable GitOps workflows and direct installation without downloading archives.

## Current situation

Kroxylicious publishes Kubernetes operator and admission webhook as `.tar.gz` and `.zip` archives containing installation manifests (multiple files in a directory), example configurations, and documentation.

## Motivation

**Problem:** The archive format is incompatible with GitOps workflows. Users must download, extract, locate manifests, manually commit to their GitOps repository, and repeat for every release. GitOps tools (Flux CD, Argo CD) expect to pull manifests directly from URLs or Git repositories, not from archives.

**User impact:** Teams using GitOps must maintain manual workarounds, creating friction and delaying adoption.

**Ecosystem precedent:** Projects like Strimzi and cert-manager solve this by publishing single-file install manifests as release assets:
- Strimzi: `strimzi-cluster-operator-{version}.yaml`, `strimzi-crds-{version}.yaml`
- cert-manager: `cert-manager.yaml`, `cert-manager.crds.yaml`

This pattern enables one-command installation:
```bash
kubectl apply -f https://github.com/project/releases/download/v1.0.0/install.yaml
```

**GitOps integration:** GitOps tools can reference these URLs directly in their configuration, enabling automated deployment and upgrades.

## Proposal

Publish the following artifacts as GitHub release assets for each release:

### Operator Artifacts

All artifacts follow Maven classifier naming conventions (`artifactId-version-classifier.extension`):

| Artifact | Maven Coordinates | Use Case |
|----------|-------------------|----------|
| `kroxylicious-operator-dist-{version}-install.yaml` | `io.kroxylicious:kroxylicious-operator-dist:install:yaml` | Complete installation (CRDs + operator), single-file GitOps reference |
| `kroxylicious-operator-dist-{version}-crds.yaml` | `io.kroxylicious:kroxylicious-operator-dist:crds:yaml` | CRDs only for separate lifecycle management |
| `kroxylicious-operator-dist-{version}-examples.tar.gz` | `io.kroxylicious:kroxylicious-operator-dist:examples:tar.gz` | Example configurations |
| `kroxylicious-operator-dist-{version}-examples.zip` | `io.kroxylicious:kroxylicious-operator-dist:examples:zip` | Example configurations (ZIP) |

### Admission Webhook Artifacts

All artifacts follow Maven classifier naming conventions:

| Artifact | Maven Coordinates | Use Case |
|----------|-------------------|----------|
| `kroxylicious-admission-dist-{version}-install.yaml` | `io.kroxylicious:kroxylicious-admission-dist:install:yaml` | Complete installation (CRDs + webhook) |
| `kroxylicious-admission-dist-{version}-crds.yaml` | `io.kroxylicious:kroxylicious-admission-dist:crds:yaml` | CRDs only |
| `kroxylicious-admission-dist-{version}-examples.tar.gz` | `io.kroxylicious:kroxylicious-admission-dist:examples:tar.gz` | Example configurations |
| `kroxylicious-admission-dist-{version}-examples.zip` | `io.kroxylicious:kroxylicious-admission-dist:examples:zip` | Example configurations (ZIP) |

### Manifest Structure

**Full install manifests** contain (in order):
1. Namespace
2. CustomResourceDefinitions
3. ServiceAccount
4. ClusterRole(s)
5. ClusterRoleBinding(s)
6. Deployment
7. Additional resources (Service, PodDisruptionBudget, MutatingWebhookConfiguration for admission)

Resources separated by `---` (YAML document separator).

**CRDs-only manifests** contain:
- Only CustomResourceDefinition resources
- Same variable substitution as full manifests
- Separated by `---`

**Header comments** in each manifest include:
- Copyright notice
- Installation instructions with URL example
- Version information

### Variable Substitution

All Maven template variables are substituted during build:
- `$[io.kroxylicious.operator.image.name]` → `quay.io/kroxylicious/operator:0.22.0`
- `$[io.kroxylicious.webhook.image.name]` → `quay.io/kroxylicious/webhook:0.22.0`
- `$[project.version]` → actual version number

No template markers remain in published manifests.

### Installation Workflows

**Single-step installation:**
```bash
kubectl apply -f https://github.com/kroxylicious/kroxylicious/releases/download/v0.22.0/kroxylicious-operator-dist-0.22.0-install.yaml
```

**Two-stage installation (CRDs first):**
```bash
# Cluster-admin installs CRDs
kubectl apply -f https://github.com/kroxylicious/kroxylicious/releases/download/v0.22.0/kroxylicious-operator-dist-0.22.0-crds.yaml

# Namespace-admin installs operator
kubectl apply -f https://github.com/kroxylicious/kroxylicious/releases/download/v0.22.0/kroxylicious-operator-dist-0.22.0-install.yaml
```

**GitOps (Kustomize example):**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - https://github.com/kroxylicious/kroxylicious/releases/download/v0.22.0/kroxylicious-operator-dist-0.22.0-install.yaml
```

This Kustomization can be used with any GitOps tool that supports Kustomize (Flux CD, Argo CD, etc.).

### Signing

All artifacts (YAML manifests and archives) are automatically GPG-signed by `maven-gpg-plugin` during the Maven build as they are attached Maven artifacts. Corresponding `.asc` signature files are published alongside each artifact.

### Examples Separation

Examples move to dedicated archives separate from install manifests:
- Reduces size of install manifests
- Matches Strimzi pattern (examples in archive, install as single file)
- Allows examples to include multi-file scenarios

## Affected/not affected projects

| Project | Affected? | Impact |
|---------|-----------|--------|
| kroxylicious-operator | ✅ Yes | New artifacts published |
| kroxylicious-admission | ✅ Yes | New artifacts published |
| kroxylicious-docs | ✅ Yes | Installation guide updates |
| kroxylicious-app | ❌ No | No change |
| kroxylicious-filters | ❌ No | No change |
| kroxylicious-kms | ❌ No | No change |

## Compatibility

### Backwards Compatibility

**Preserved:** Existing `.tar.gz` and `.zip` archives continue to be published for 2 releases (v0.22.0, v0.23.0).

**Deprecated:** These combined archives marked as deprecated in release notes.

**Removed:** Archives removed in v0.24.0 (gives users 2 releases to migrate).

**Migration path:** Documentation provides clear instructions for switching from archive-based to manifest-based installation.

### Future Compatibility

**Version pinning:** Users referencing specific version URLs (`/v0.22.0/install.yaml`) are insulated from future changes.

**Manifest stability:** As long as the Kubernetes API versions remain compatible (v1, apps/v1, rbac.authorization.k8s.io/v1), these manifests remain usable.

**CRD versioning:** Follows Kubernetes API versioning (v1alpha1, v1beta1, v1). Changes to CRD API versions require new manifests per version.

### Deprecation Plan

**Why deprecate archives:** Combined archives that include both install manifests and examples serve overlapping purposes with the new artifacts. Continuing to ship both creates confusion and maintenance burden.

**Timeline:**

Combined archives will be deprecated immediately in v0.22.0 and removed at the **v1.0.0 milestone** (whichever comes first: v0.24.0 or v1.0.0).

| Release | Archive Status | Manifest Status | Notes |
|---------|---------------|-----------------|-------|
| v0.22.0 | ⚠️ **Deprecated** - Published with deprecation notice | ✅ **New** - Full support | Introduction release. Release notes highlight new manifests as recommended approach. |
| v0.23.0 | ⚠️ **Deprecated** - Published | ✅ **Supported** | Deprecation notice continues in release notes. |
| v0.24.0 or v1.0.0 | ❌ **Removed** - No longer published | ✅ **Supported** | Archives removed at v1.0.0 milestone or v0.24.0, whichever comes first. |

**What gets deprecated:**
- `kroxylicious-operator-{version}.tar.gz` (combined install + examples)
- `kroxylicious-operator-{version}.zip` (combined install + examples)
- `kroxylicious-admission-{version}.tar.gz` (combined install + examples)
- `kroxylicious-admission-{version}.zip` (combined install + examples)

**What replaces them:**
- `kroxylicious-operator-dist-{version}-install.yaml` (install only, fully rendered)
- `kroxylicious-operator-dist-{version}-crds.yaml` (CRDs only, fully rendered)
- `kroxylicious-operator-dist-{version}-examples.tar.gz` (examples only)
- `kroxylicious-operator-dist-{version}-examples.zip` (examples only)
- Same for admission webhook

**Migration guidance (documented in release notes and installation guide):**

*For users currently using archives:*

Before (v0.21.0 and earlier):
```bash
# Download archive
curl -L https://github.com/kroxylicious/kroxylicious/releases/download/v0.21.0/kroxylicious-operator-0.21.0.tar.gz | tar xz

# Navigate to install directory
cd kroxylicious-operator-0.21.0/install

# Apply manifests
kubectl apply -f .
```

After (v0.22.0 and later):
```bash
# Direct installation (no download/extract needed)
kubectl apply -f https://github.com/kroxylicious/kroxylicious/releases/download/v0.22.0/kroxylicious-operator-dist-0.22.0-install.yaml

# Download examples separately (only if needed)
curl -L https://github.com/kroxylicious/kroxylicious/releases/download/v0.22.0/kroxylicious-operator-dist-0.22.0-examples.tar.gz | tar xz
```

*For GitOps users:*

Before: Manual workaround involving downloading archives and committing rendered manifests to Git.

After: Direct reference to release URL:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - https://github.com/kroxylicious/kroxylicious/releases/download/v0.22.0/kroxylicious-operator-dist-0.22.0-install.yaml
```

**Rationale for 3-release timeline:**
- v0.22.0: Users discover new approach, begin migration
- v0.23.0: Final warning allows remaining users to migrate
- v0.24.0: Clean removal with clear documented migration path
- Matches Kubernetes deprecation guidelines (minimum 2 releases notice)

## Rejected alternatives

### Alternative 1: Helm Charts

**Rejected because:**
- Adds Helm as a new dependency for users
- Requires maintaining Helm chart structure
- Less direct for users who just want `kubectl apply`
- Can be added later without conflicting with this approach

### Alternative 2: Kustomize Bases

**Rejected because:**
- Requires users to create Kustomization files
- More complex than single-file reference
- Less discoverable than release assets
- Kustomize users can still reference the YAML manifests

### Alternative 3: OCI Artifacts Only

**Rejected because:**
- OCI artifact support varies by GitOps tool and version
- Less discoverable than GitHub release assets
- Can be added later as complementary distribution
- GitHub release URLs work universally

### Alternative 4: Keep Combined Archives Only

**Rejected because:**
- Doesn't solve the GitOps problem
- Forces manual extraction workflow
- Out of step with ecosystem practices (Strimzi, cert-manager)
