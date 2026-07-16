# Issues and Solutions: EaaS Certsuite Pipeline

## Context

| | Detail |
|---|---|
| **Operator monorepo** | [redhat-cne/downstream-ptp-operator-monorepo](https://github.com/redhat-cne/downstream-ptp-operator-monorepo) (branch: `release-4.22`) |
| **Pipeline repo** | [redhat-best-practices-for-k8s/konflux-certsuite](https://github.com/redhat-best-practices-for-k8s/konflux-certsuite) |
| **Working fork** | [yaronh12/konflux-certsuite](https://github.com/yaronh12/konflux-certsuite) (branch: `fix/eaas-deploy-fixes`) |
| **Reference fork** | [aabughosh/konflux-certsuite](https://github.com/aabughosh/konflux-certsuite) (branch: `fix/wire-unreleased-bundle-result`) |

The goal was to get the `certsuite-operator-test-eaas` pipeline running end-to-end on Konflux EaaS (Hypershift) clusters, testing the PTP operator before release.

This document describes all the issues encountered and how each was resolved.

### Comparison with Amal's Approach

Both forks solve the same problem — deploying an unreleased operator on a Hypershift cluster that can't pull from `registry.redhat.io` — but take opposite strategies. Amal's approach bypasses OLM entirely: it uses `oc image extract` to pull manifests from the bundle image on `quay.io`, applies CRDs and CSV directly, pre-creates ServiceAccounts with `cluster-admin`, and creates a Subscription only as metadata for certsuite discovery (OLM never actually installs anything). This is simpler but fails certsuite's `operator-install-source` test because the Subscription was never used for a real install. Our approach keeps OLM in the loop: we patch the FBC catalog to point to `quay.io`, serve it via an in-cluster `opm serve` pod, and let OLM handle the full Subscription → InstallPlan → CSV lifecycle. After OLM creates the CSV, we patch it to fix remaining image references and Hypershift constraints. This is more complex but produces a genuine OLM install that passes all certsuite operator tests. Most other fixes (CSV image replacement, minKubeVersion removal, nodeSelector removal, certsuite labels, kubeconfig handling) are shared between both approaches.

---

## 1. Pipeline Result Wiring

### Issue: `unreleasedBundle` task result not wired
The `get-unreleased-bundle` task declared the `unreleasedBundle` result but never mapped it from the step-level result. This caused it to always be empty, blocking the entire pipeline.

**Fix:** Added `value: "$(steps.get-bundle.results.unreleasedBundle)"` to wire the step result to the task result.

### Issue: Workspace binding not provided
Konflux IntegrationTestScenarios don't provide pipeline-level workspace bindings when generating PipelineRuns. The pipeline declared a required `shared` workspace used by `collect-results`.

**Fix:** Removed the `workspaces` declaration and `collect-results` task from the EaaS pipeline variant.

### Issue: Redundant `test-event-type` when guards
The EaaS variant is only triggered by Konflux IntegrationTestScenarios (always push events). The `test-event-type` checks were unnecessary.

**Fix:** Removed all `test-event-type` when guards from the EaaS pipeline.

---

## 2. Unreleased Bundle Image Resolution

### Issue: Bundle image at `registry.redhat.io` returns 401
The FBC catalog stores a `registry.redhat.io` reference for the bundle image. The image is unreleased (built by Konflux) and doesn't exist there — it only exists on `quay.io/redhat-user-workloads`.

**Fix:** Derive the `quay.io` path from the FBC fragment's naming convention. The FBC component is `ptp-operator-fbc-4-22`, so the bundle is at `ptp-operator-bundle-mono-4-22` in the same tenant. Verify with `skopeo inspect` before using. Patch the FBC content with `sed` to replace the reference.

> **⚠️ Important: Images are NOT on `registry.redhat.io`**
>
> The `experimental-ptp-tenant` monorepo images exist ONLY on `quay.io/redhat-user-workloads`. They are pre-release Konflux builds with digests that have never been published to `registry.redhat.io`. Even with a valid pull secret, the cluster cannot pull these images from the Red Hat registry. The image path naming is also different (e.g., `ptp-rhel9-operator` vs `ose-ptp-rhel9-operator`). This makes the CSV image patching workaround **essential and non-removable** for unreleased operators.

---

## 3. Operator Deployment on Hypershift EaaS

### Issue: OLM can't pull bundle (IDMS blocked on Hypershift)
OLM's Subscription/InstallPlan mechanism tries to pull the bundle from `registry.redhat.io` to unpack it. On Hypershift, IDMS (ImageDigestMirrorSet) creation is blocked by a `ValidatingAdmissionPolicy` — only the management cluster admin can configure image mirrors.

**Fix:** Render the FBC content with `opm render`, patch bundle references with `sed` to point to `quay.io`, serve the patched catalog via an in-cluster `opm serve` Deployment + Service + ConfigMap. OLM's CatalogSource points to this local gRPC server instead of pulling from a registry.

### Issue: ConfigMap volume mount `..data` symlink crash
When mounting a ConfigMap as a volume, Kubernetes creates a `..data` symlink directory structure. `opm serve` tried to read `/catalog/..data` as a file and crashed: `failed to rebuild cache: read /catalog/..data: is a directory`.

**Fix:** Use `subPath: catalog.json` in the volumeMount to mount the file directly at `/catalog/catalog.json`, bypassing Kubernetes's symlink structure.

### Issue: CSV images reference `registry.redhat.io`
The CSV's deployment spec references operator images (controller, daemon, sidecars) at `registry.redhat.io` with digests. The Hypershift cluster cannot pull these unreleased images.

**Fix:** After OLM creates the CSV, scan for all `registry.redhat.io` references and replace each with the corresponding `quay.io/redhat-user-workloads` tenant image:

| Original (`registry.redhat.io/openshift4/...`) | Replacement (`quay.io/redhat-user-workloads/experimental-ptp-tenant/...`) |
|---|---|
| `ptp-rhel9-operator` | `ptp-operator-mono-4-22` |
| `ptp-rhel9` (linuxptp-daemon) | `linuxptp-daemon-mono-4-22` |
| `cloud-event-proxy-rhel9` | `cloud-event-proxy-mono-4-22` |
| `ptp-operator-must-gather` | `ptp-must-gather-mono-4-22` |
| `ose-kube-rbac-proxy-rhel9` | `quay.io/openshift/origin-kube-rbac-proxy` (public upstream) |

### Issue: OLM reverts CSV patches (`oc apply` vs `oc replace`)
When using `oc apply` to patch the CSV, OLM's reconciler would reset the CSV phase back to `Pending`, fighting our changes. The CSV got stuck in an infinite `Pending` loop.

**Fix:** Use `oc replace -f -` instead of `oc apply`. This performs a full object replacement that OLM accepts without resetting the phase.

### Issue: `minKubeVersion` requirement not met
The CSV requires Kubernetes 1.35 (OCP 4.22), but EaaS clusters may run OCP 4.21 (Kubernetes 1.34). OLM refuses to proceed.

**Fix:** Remove the `minKubeVersion` field from the CSV with `jq` before applying. We're testing operator functionality, not version compatibility.

### Issue: Master `nodeSelector` on Hypershift
The CSV's deployment has `nodeSelector: {"node-role.kubernetes.io/master": ""}`. Hypershift clusters have no schedulable master nodes (control plane is hosted externally). Pods never get scheduled.

**Fix:** Remove the master `nodeSelector` from the CSV deployment spec with `jq` before applying.

---

## 4. Certsuite Configuration

### Issue: Config file not found
Certsuite defaults to looking for `config/certsuite_config.yml`. When running in a Tekton pod, this path doesn't exist. `FATAL: Cannot load configuration file`.

**Fix:** The test bundle must include a `certsuite_config.yml`. The pipeline copies it to `config/certsuite_config.yml` AND passes `--config-file` explicitly.

### Issue: Label filter defaulting to "none"
Without `--label-filter`, certsuite uses the literal string `"none"` as the filter expression, which matches NO test labels. All 121 tests were skipped.

**Fix:** Default the `CERTSUITE_LABELS` pipeline parameter to `"common"`. The pipeline passes `--label-filter` with this value.

### Issue: Wrong label selectors in config
Used native operator labels (`name: ptp`, `control-plane: controller-manager`) for certsuite discovery. Certsuite couldn't find any matching resources.

**Fix:** Changed `podsUnderTestLabels` to `"redhat-best-practices-for-k8s.com/generic: target"` and `operatorsUnderTestLabels` to `"redhat-best-practices-for-k8s.com/operator: target"` to match certsuite discovery labels injected by the deploy step.

---

## 5. Certsuite Cluster Access

### Issue: In-cluster SA token overrides KUBECONFIG
The certsuite container has a service account token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token` (read-only, can't remove). Go's `client-go` prefers in-cluster config over `KUBECONFIG`, causing certsuite to authenticate as the Tekton SA (`experimental-ptp-tenant:konflux-integration-runner`) instead of the EaaS cluster admin.

**Fix:**
- `unset KUBERNETES_SERVICE_HOST KUBERNETES_SERVICE_PORT` — disables in-cluster detection
- `cp "${KUBECONFIG}" ~/.kube/config` — default location `client-go` checks
- Pass `--kubeconfig` explicitly to `certsuite run`

---

## 6. Certsuite Discovery

### Issue: Subscription not found for CSV
Certsuite searches for Subscriptions and matches them to CSVs via `status.installedCSV`. Without a proper Subscription chain, operator tests skip.

**Fix:** Create a real CatalogSource + Subscription with `installPlanApproval: Manual`. OLM creates the InstallPlan, we approve it, and OLM installs the CSV. After CSV reaches `Succeeded`, patch the Subscription status with `installedCSV` pointing to the CSV.

### Issue: CSV metadata label needed
Certsuite's `findOperatorsByLabels` uses a label selector on CSV `.metadata.labels`. The CSV needs the `redhat-best-practices-for-k8s.com/operator: target` label.

**Fix:** `oc label csv` with the certsuite operator label after CSV reaches `Succeeded` state.

### Issue: DaemonSet pods need certsuite labels
The `linuxptp-daemon` DaemonSet is created by the operator (not OLM), so it doesn't have certsuite discovery labels.

**Fix:** After operator is ready, discover all DaemonSets in the install namespace, patch their pod template with certsuite labels, and wait for rollout.

---

## 7. Tooling and Image Issues

### Issue: `git` command not found
The `origin-cli` image used for `deploy-operands` doesn't include `git`. Cannot clone the test bundle repo.

**Fix:** Replaced `git clone` with `curl`-based download: `curl` the tarball from GitHub, extract to temp dir, copy the test bundle path. No git dependency needed.

### Issue: `oc` not available in certsuite image
The certsuite image only has the `certsuite` binary — no `oc`, `kubectl`, or other tools.

**Fix:** All `oc`/`kubectl` commands are in earlier steps. The certsuite step only runs `certsuite` itself.

### Issue: `@branch` syntax not supported in `TEST_BUNDLE_REF`
The `deploy-operands` step only parsed `url#path` format, not `url@branch#path`.

**Fix:** Added branch parsing to support `https://github.com/user/repo.git@branch#path` format.

---

## Architecture Summary

The final pipeline flow on Hypershift EaaS:

| Step | Action | Key Detail |
|---|---|---|
| 1 | Parse SNAPSHOT metadata | Extract FBC fragment image from Konflux ApplicationSnapshot |
| 2 | Provision EaaS Space | Konflux EaaS namespace for cluster lifecycle |
| 3 | Get unreleased bundle | Resolve bundle image reference from FBC fragment |
| 4 | Pick cluster params | Select OCP version + architecture from FBC metadata |
| 5 | Provision Hypershift cluster | Ephemeral AWS Hypershift cluster via EaaS |
| 6a | Render + patch FBC | `opm render` FBC, `sed`-replace bundle ref to `quay.io` |
| 6b | Deploy `opm serve` pod | ConfigMap (`subPath`) + Deployment + Service for patched catalog |
| 6c | CatalogSource + Subscription | `Manual` approval; OLM creates InstallPlan + CSV |
| 6d | Approve InstallPlan | `oc patch installplan` with `approved: true` |
| 6e | Patch CSV | Replace `registry.redhat.io` images, remove `minKubeVersion` + `nodeSelector` |
| 6f | Wait CSV Succeeded | `oc replace` ensures OLM doesn't reset phase |
| 6g | Certsuite discovery labels | Label CSV + patch DaemonSet pod templates |
| 7 | Deploy operands | `curl` test bundle, apply prerequisites + operands |
| 8 | Run certsuite | Disable in-cluster config, run with `--label-filter` |

---

## Root Cause Analysis: Why `registry.redhat.io` References Exist

This section explains the full picture: what Konflux nudging is, why the apps are split the way they are, why consolidating them wouldn't help, and what would actually fix the problem.

### Background: What Is Konflux Nudging?

Konflux builds container images from source code. When an operator has multiple images (controller, daemon, sidecar, bundle, catalog), they form a dependency chain — the bundle references the operator images, and the catalog references the bundle. When one image is rebuilt, all downstream images need to update their references to point to the new version.

**Nudging** is Konflux's mechanism for automating this. When component A is rebuilt, Konflux automatically opens a PR (via Renovate) to update component B's source files with the new digest of A. This triggers a rebuild of B, which in turn nudges C, and so on.

Concretely for the PTP operator:

```
1. ptp-operator-mono-4-22 is rebuilt
      ↓ (nudge: updates pin_images.in.yaml with new digest)
2. ptp-operator-bundle-mono-4-22 is rebuilt
      ↓ (nudge: updates bundle.builds.in.yaml with new digest)
3. ptp-operator-fbc-4-22 is rebuilt
```

Nudging only works between components within the same Konflux Application, or across Applications if explicitly configured. The nudge target files (like `pin_images.in.yaml`) are regular files in the Git repo — Renovate opens a PR that changes them, which triggers a new build.

### Background: What Is IDMS (ImageDigestMirrorSet)?

An `ImageDigestMirrorSet` is an OpenShift cluster-level resource that tells the container runtime: "when someone asks to pull an image from registry A, actually pull it from registry B instead." It's a transparent redirect — the pods and manifests still reference the original registry, but the cluster silently pulls from the mirror.

Example:
```yaml
apiVersion: config.openshift.io/v1
kind: ImageDigestMirrorSet
metadata:
  name: ptp-operator-mirrors
spec:
  imageDigestMirrors:
    - source: registry.redhat.io/openshift4/ptp-rhel9-operator
      mirrors:
        - quay.io/redhat-user-workloads/experimental-ptp-tenant/ptp-operator-mono-4-22
```

With this IDMS applied, a pod referencing `registry.redhat.io/openshift4/ptp-rhel9-operator@sha256:abc123` would transparently pull from `quay.io/redhat-user-workloads/.../ptp-operator-mono-4-22@sha256:abc123` instead.

**The problem:** On Hypershift EaaS clusters, a `ValidatingAdmissionPolicy` blocks IDMS creation. Only the management cluster admin (the Hypershift platform team) can configure image mirrors. Tenant users cannot create or modify IDMS resources on guest clusters. When we tried applying IDMS, we got: `ValidatingAdmissionPolicy 'mirror' denied request: This resource cannot be created, updated, or deleted.`

### Konflux Application Structure

The `experimental-ptp-tenant` has **two** Konflux Applications:

| Konflux Application | Components |
|---|---|
| **`ptp-operator-mono-4-22`** | `ptp-operator-mono-4-22`, `linuxptp-daemon-mono-4-22`, `cloud-event-proxy-mono-4-22`, `ptp-must-gather-mono-4-22`, `ptp-operator-bundle-mono-4-22` |
| **`ptp-operator-fbc-4-22`** | `ptp-operator-fbc-4-22` (catalog fragment only) |

The FBC **must** be in a separate Application — this is a [Konflux requirement for OLM operators](https://konflux-ci.dev/docs/end-to-end/building-olm/). The reason is that the FBC (File-Based Catalog) is the final release artifact that contains the operator's entry in the OLM catalog. It has a different release lifecycle, a different build pipeline (`fbc-pipeline` vs `build-pipeline`), and is the component that Konflux IntegrationTestScenarios (like our certsuite pipeline) target.

### How the Nudging Chain Works in Detail

Here's exactly what happens when a new operator image is built:

**Step 1: Operator image built → `pin_images.in.yaml` updated**

When `ptp-operator-mono-4-22` is rebuilt on Konflux, nudging (via Renovate) opens a PR to the monorepo that updates `.konflux/overlay/pin_images.in.yaml`:

```yaml
- key: manager
  source: quay.io/openshift/origin-ptp-operator:4.22
  target: quay.io/redhat-user-workloads/experimental-ptp-tenant/ptp-operator-mono-4-22@sha256:8b86529d...
```

This file maps each operator role (`manager`, `linuxptp-daemon`, `cloud-event-proxy`, etc.) to the actual Konflux-built image on `quay.io`. The `source` is the upstream development image reference. The `target` is the Konflux-built image with its exact digest.

**Step 2: Bundle built → CSV generated with `registry.redhat.io` paths**

The bundle build reads `pin_images.in.yaml` and runs `konflux-bundle-overlay.sh`. This script does something critical: it takes the `quay.io` digests from `pin_images.in.yaml` but writes them into the CSV with `registry.redhat.io` paths:

```
Input:  quay.io/redhat-user-workloads/experimental-ptp-tenant/ptp-operator-mono-4-22@sha256:8b86529d...
Output: registry.redhat.io/openshift4/ptp-rhel9-operator@sha256:8b86529d...
```

**Why?** Because in production, when the operator is officially released, the images will be published to `registry.redhat.io`. The bundle must reference the production registry so that customers can pull from it. The digest (`sha256:8b86529d...`) is the same image content regardless of which registry hosts it.

**Step 3: Bundle digest → `bundle.builds.in.yaml` updated → FBC rebuilt**

Nudging then updates `.konflux/catalog/bundle.builds.in.yaml` with the new bundle digest, triggering an FBC rebuild. The FBC pipeline also applies its own IDMS (`catalog-idms.yaml`) to rewrite the bundle reference from `registry.redhat.io` to `quay.io` within the catalog.

### The Core Problem: Same Digest, Wrong Registry

After all the nudging, the state is:

| What | Where it references | Where the image actually lives |
|---|---|---|
| FBC fragment image | `quay.io/redhat-user-workloads/.../ptp-operator-fbc-4-22@sha256:...` | `quay.io` (accessible) |
| Bundle image ref in FBC | `registry.redhat.io/openshift4/ptp-operator-bundle@sha256:...` | `quay.io` only (inaccessible path) |
| Operator image refs in CSV | `registry.redhat.io/openshift4/ptp-rhel9-operator@sha256:...` | `quay.io` only (inaccessible path) |
| Daemon image refs in CSV | `registry.redhat.io/openshift4/ptp-rhel9@sha256:...` | `quay.io` only (inaccessible path) |

The digests are correct — the same image bytes exist on `quay.io`. But the CSV tells the Hypershift cluster to pull from `registry.redhat.io`, which it can't access. It's like having the right address for a package but the wrong zip code — the package exists, but the delivery service can't find it.

### Why Consolidating Apps Won't Help

The initial thought was: "if all components were in one Konflux Application, nudging would rewrite everything to `quay.io`." But investigating the actual build pipeline reveals this is wrong:

1. **The apps are already structured correctly.** The operator images and the bundle ARE in the same Application (`ptp-operator-mono-4-22`). Nudging between them works fine — digests flow correctly.

2. **Nudging updates digests, not registry paths.** Nudging's job is to ensure component B uses the latest digest of component A. It does this by updating files like `pin_images.in.yaml`. It doesn't change the registry path in the final output.

3. **The `registry.redhat.io` path is intentional.** The bundle build script is designed to produce production-ready references. Changing this would break the release process. The script reads `quay.io` inputs and maps them to `registry.redhat.io` outputs on purpose.

4. **The FBC must be in a separate Application.** Even if it weren't, the core problem (CSV → `registry.redhat.io` → Hypershift can't pull) would remain because it's the bundle build that generates those references, and the bundle is already in the same Application as the operator images.

In short: consolidating the apps changes nothing because the apps are already consolidated where it matters. The problem is not about app boundaries — it's about the bundle build script hardcoding `registry.redhat.io` paths for production use.

### The `images-mirror-set.yaml` Already Exists

Here's the ironic part: the monorepo already contains `.tekton/images-mirror-set.yaml` with a complete IDMS mapping every `registry.redhat.io` image to its `quay.io` equivalent:

```yaml
imageDigestMirrors:
  - source: registry.redhat.io/openshift4/ptp-rhel9-operator
    mirrors:
      - quay.io/redhat-user-workloads/experimental-ptp-tenant/ptp-operator-mono-4-22
  - source: registry.redhat.io/openshift4/ptp-rhel9
    mirrors:
      - quay.io/redhat-user-workloads/experimental-ptp-tenant/linuxptp-daemon-mono-4-22
  # ... and so on for all components
```

This IDMS is used by Conforma/FIPS compliance tests (which run on clusters where IDMS IS allowed). If this exact same IDMS could be applied to the Hypershift cluster, every image pull from `registry.redhat.io` would transparently redirect to `quay.io` and succeed. The mapping is already maintained by nudging.

But we can't apply it because Hypershift blocks it.

---

## Proposed Fix: Eliminate Image Workarounds

> **This is the single biggest improvement that could simplify the pipeline.** Currently, 5 of our 8 workarounds exist solely because of the `registry.redhat.io` → `quay.io` image mapping problem. All three options below eliminate the same 5 workarounds — they differ in where the change is made.

### Option A: Allow IDMS on EaaS Hypershift clusters (Platform change — recommended)

**What:** Request the EaaS / Hypershift platform team to do one of:
- Relax the `ValidatingAdmissionPolicy` for ephemeral test clusters, allowing tenants to create IDMS resources
- Pre-apply IDMS from the tenant's `.tekton/images-mirror-set.yaml` during cluster provisioning (EaaS could accept an IDMS file as a parameter)
- Inject the IDMS into the HostedCluster spec (management-cluster-side, bypasses the guest policy)

**How it would work:** The certsuite pipeline would simply `oc apply -f images-mirror-set.yaml` on the Hypershift cluster (or it would be pre-applied). From that point on, every pod that tries to pull `registry.redhat.io/openshift4/ptp-rhel9-operator@sha256:abc` would transparently pull from `quay.io/redhat-user-workloads/.../ptp-operator-mono-4-22@sha256:abc` instead. OLM would install the operator normally with no patching needed.

**Impact:** Eliminates 5 workarounds at once:

| Workaround eliminated | Why it's no longer needed |
|---|---|
| FBC content patching (bundle ref → `quay.io`) | IDMS transparently redirects the bundle pull to `quay.io` |
| `opm serve` pod + ConfigMap + Service | Standard CatalogSource with `image:` pointing to the FBC fragment works directly |
| CSV image replacement (`sed` for each component) | IDMS handles the redirect for all operator images |
| Manual InstallPlan approval | No need to intercept — OLM can install directly |
| `oc replace` for CSV | No CSV modification needed at all |

**Why it makes sense:**
- EaaS clusters are ephemeral (destroyed after testing) — no security risk
- The `images-mirror-set.yaml` already exists in the monorepo and is kept up-to-date by nudging
- This is the same approach used by Conforma/FIPS tests, just on a different cluster type
- Zero code changes to the monorepo or the pipeline (beyond removing the workarounds)

**Who to ask:** EaaS / Hypershift platform team.

**Effort:** Low for the platform team (policy change or provisioning parameter). Zero for us.

### Option B: Bundle build flag for `quay.io` paths (Monorepo change)

**What:** The bundle build is invoked in `.konflux/Dockerfile.bundle` with the flag `--set-mapping-production`, which calls the shared script `telco5g-konflux/scripts/bundle/konflux-bundle-overlay.sh` (from the [openshift-kni/telco5g-konflux](https://github.com/openshift-kni/telco5g-konflux) submodule). That flag triggers the `map_images` step that rewrites `quay.io/redhat-user-workloads/...` → `registry.redhat.io/openshift4/...` using rules from `.konflux/overlay/map_images.in.yaml`. The fix would be to skip `--set-mapping-production` (or add a `--skip-mapping` flag) to produce a CSV that keeps the `quay.io` paths. The certsuite pipeline would use this "test" bundle; the production pipeline keeps the default.

**How it would work:** The bundle build would skip the `quay.io → registry.redhat.io` path rewrite. The CSV would directly reference `quay.io/redhat-user-workloads/experimental-ptp-tenant/ptp-operator-mono-4-22@sha256:...`. The Hypershift cluster can pull from `quay.io` without issues.

**Impact:** Same 5 workarounds eliminated.

**Downsides:**
- The "test" bundle differs from the production bundle (different image paths, same content). Certsuite would be testing a slightly different artifact than what ships to customers.
- Requires maintaining two build modes in the monorepo.
- Every operator monorepo would need to add this flag — not just PTP.
- The script is shared across all telco5g operators via the `openshift-kni/telco5g-konflux` submodule, so changes have broad impact.

**Effort:** Medium. Requires changes to the shared build scripts and possibly a separate Konflux component for the "test" bundle.

### Option C: Keep pipeline patching (Current state)

**What:** Continue doing everything in the certsuite pipeline: render FBC, patch bundle refs, deploy `opm serve` pod, create CatalogSource, create Subscription with Manual approval, wait for CSV, patch CSV images, replace CSV.

**Impact:** Works today.

**Downsides:**
- ~200 lines of fragile bash in the pipeline
- PTP-specific hardcoded `sed` patterns (would need to be generalized for other operators via `COMPONENT_REPOS`)
- Any change to the operator's image names or structure requires pipeline changes
- Multiple failure points (opm serve pod crash, CSV stuck in Pending, etc.)

**Effort:** Zero (already done). But ongoing maintenance cost is high.

---

## Remaining Workarounds (Cannot Remove Regardless of Fix Above)

These workarounds are structurally required for Hypershift/certsuite and cannot be eliminated by any image-related fix.

| Workaround | Why It's Required |
|---|---|
| Remove `minKubeVersion` | EaaS may provision a slightly older OCP version than the operator requires. |
| Remove master `nodeSelector` | Hypershift has no schedulable master nodes — control plane is hosted externally. |
| Certsuite discovery labels | Operator-created resources (DaemonSets) don't have certsuite labels by default. |
| Disable in-cluster SA config | Certsuite container's mounted SA token overrides the EaaS kubeconfig. |

---

## Open Questions

### 1. Why are we testing an unreleased operator?
The PTP operator monorepo (`release-4.22`) is built by Konflux in the `experimental-ptp-tenant`. These are pre-release images that have never been published to `registry.redhat.io`. Is the intent to run certsuite as a **pre-release gate**? If so, the registry workarounds (or one of the proposed fixes above) are permanently necessary for this flow. If we wait for release, the images exist on `registry.redhat.io` but the Hypershift cluster still needs a pull secret.

### 2. When should certsuite run: on every PR, merge, or release?
Currently this pipeline is triggered by Konflux IntegrationTestScenarios on the FBC component. This means it runs whenever the FBC fragment is rebuilt (typically on merges to the monorepo). Should it also run on PRs? On every component build (operator, linuxptp, cloud-event-proxy)? Running on PRs provides faster feedback but consumes EaaS cluster resources. Running only on merges is cheaper but delays failure detection.

### 3. Can we make the image mapping generic instead of PTP-hardcoded?
Currently the CSV image replacement uses hardcoded `sed` patterns specific to the PTP operator (`ptp-rhel9-operator`, `ptp-rhel9`, `cloud-event-proxy-rhel9`, etc.). The `COMPONENT_REPOS` pipeline parameter could be used to dynamically discover which `quay.io` repos to check. Iterate `COMPONENT_REPOS`, check each `registry.redhat.io` image digest against `quay.io/redhat-user-workloads/<tenant>/<repo>`, and replace if found. This would make the pipeline reusable for other operators.

### 4. Should we merge the OLM-native approach upstream?
The OLM-native approach produces better certsuite results (passes `operator-install-source`) but is more complex than the direct-apply approach. Should we propose it as a replacement for the upstream pipeline, or keep both as options? The upstream pipeline could support both approaches via a parameter.

### 5. Can Konflux/EaaS inject `registry.redhat.io` credentials into Hypershift clusters?
A `redhat-registry-pull-secret` exists in the Konflux tenant namespace. If it could be automatically injected into the Hypershift guest cluster's `pull-secret`, the cluster could pull **released** images from `registry.redhat.io` directly. This wouldn't help with unreleased images (they don't exist on `registry.redhat.io`), but it would eliminate workarounds for operators that ARE released. This would be a platform-level improvement request to the EaaS team.
