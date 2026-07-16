# certsuite-operator-test pipelines

Two pipeline variants for running the Red Hat Best Practices Test Suite
for Kubernetes (certsuite) against an operator deployed from an FBC
fragment.

## EaaS Variant (Recommended)

**File:** `certsuite-operator-test-eaas.yaml`

Provisions a fresh ephemeral Hypershift cluster per run via Konflux
EaaS. No kubeconfig secrets, cluster locks, or OADP needed. The
cluster is automatically destroyed when the PipelineRun completes.

### Flow

1. `parse-metadata` -- extract snapshot info
2. `provision-eaas-space` -- allocate EaaS space
3. `get-unreleased-bundle` -- get catalog bundle ref + resolve quay.io bundle
4. `pick-cluster-params` -- OCP version/arch from the pullable quay bundle
5. `build-image-content-sources` -- load `.tekton/images-mirror-set.yaml`
   (from `TEST_BUNDLE_REF` repo by default) → Hypershift `imageContentSources`
6. `provision-cluster` -- create ephemeral cluster with those mirrors (HCCO
   applies a managed IDMS; `registry.redhat.io` pulls redirect to quay.io)
7. `deploy-and-test` -- standard CatalogSource from FBC image, OLM install,
   Hypershift CSV fixes (`minKubeVersion` / master `nodeSelector`), operands,
   certsuite

### Minimum Parameters

- `TEST_BUNDLE_REF` — always required. For unreleased operators, point it at
  the **operator monorepo** that also contains `.tekton/images-mirror-set.yaml`
  (Konflux/Conforma convention). The pipeline discovers that file automatically.
- `IMAGES_MIRROR_SET_REF` — optional override if the mirror set lives elsewhere.

Released operators on `registry.redhat.io` need no mirror set (skipped when the
FBC is not on `quay.io/redhat-user-workloads`).

## Shared Cluster Variant

**File:** `certsuite-operator-test.yaml`

Uses a pre-existing cluster via a kubeconfig Secret. Includes
Lease-based queueing for concurrent access and OADP backup/restore for
cluster state management.

### Flow

1. `parse-metadata` -- extract snapshot info
2. `get-unreleased-bundle` -- get operator bundle from FBC
3. `acquire-cluster-lock` -- Lease mutex on shared cluster
4. `oadp-restore-pre` -- restore to clean baseline (first run creates backup)
5. `deploy-operator` -- install via OLM
6. `deploy-operands` -- apply test bundle manifests
7. `run-certsuite` -- run test suites
8. `collect-results` -- optionally push results
9. **finally:** cleanup-operator, oadp-restore-post, release-cluster-lock

### Minimum Parameters

`TEST_BUNDLE_REF` + a `shared-cluster-kubeconfig` Secret in the tenant
namespace.

## Usage

Create an `IntegrationTestScenario` in your tenants-config repo. See:
- [EaaS example](../../../examples/integration-test-scenario-eaas.yaml) (recommended)
- [Shared cluster example](../../../examples/integration-test-scenario.yaml)
