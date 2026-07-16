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
3. `get-unreleased-bundle` -- get operator bundle from FBC
4. `pick-cluster-params` -- select OCP version and architecture
5. `provision-cluster` -- render FBC/bundle, match `registry.redhat.io`
   digests to `COMPONENT_REPOS` on quay → pass as Hypershift
   `imageContentSources` → create ephemeral cluster (HCCO applies a managed
   IDMS so pulls redirect to quay.io)
6. `deploy-and-test` -- standard CatalogSource from FBC image, OLM install,
   Hypershift CSV fixes (`minKubeVersion` / master `nodeSelector`), operands,
   certsuite

### Minimum Parameters

- `TEST_BUNDLE_REF` — always required
- `COMPONENT_REPOS` — required for **unreleased** operators (images only on
  `quay.io/redhat-user-workloads`). Comma-separated quay repo names under the
  FBC tenant, e.g.
  `ptp-operator-mono-4-22,linuxptp-daemon-mono-4-22,cloud-event-proxy-mono-4-22,ptp-must-gather-mono-4-22`.
  Omit for released operators on `registry.redhat.io`.

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
