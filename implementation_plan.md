# KServe Raw Mode Operator Implementation Plan

## User Review Required
No immediate changes require user review. We are initializing an Operator SDK project purely configured for standalone RawMode deployment.

## Proposed Architecture
The Operator will be built using Golang and Operator-SDK v1.33+ (located in `/Users/akashdeo/kserve-op/tools`).

### Operator Components
1. **CRD API**: `RawKServe` (Group: `operator.kserve.io`, Version: `v1alpha1`).
   - A single Custom Resource that triggers the installation.
2. **Controller Logic (`internal/controller/rawkserve_controller.go`)**:
   - The controller will embed the specific raw-mode YAMLs directly or load them from a local `assets` directory to remain 100% self-sufficient.
   - It will execute the installation sequentially:
     1. Apply `cert-manager`.
     2. Wait for `cert-manager` webhook to be ready.
     3. Apply KServe `CRDs` & `RBAC`.
     4. Apply KServe `Core` components (with default RawDeployment ConfigMap).
     5. Wait for `kserve` webhook to be ready (important race condition fix).
     6. Apply KServe `ServingRuntimes`.
3. **RBAC Permissions**:
   - The `manager` role will require high-level permissions (`*` on `*`) specifically to lay down CRDs, ClusterRoles, and Namespaces across the cluster.

### Directory Structure: `kserve-operator-deploy`
#### [NEW] `assets/`
- We will copy the cleanly split manifests from `kserve-manual-deploy` into `kserve-operator-deploy/assets/` to bundle with the operator.
#### [NEW] `api/v1alpha1/rawkserve_types.go`
- Go struct definitions for our custom resource.
#### [NEW] `internal/controller/rawkserve_controller.go`
- The core reconciliation loop using `sigs.k8s.io/controller-runtime` and `k8s.io/cli-runtime` (or similar apply mechanisms) to read and apply the assets.

## Workflow
1. Use `operator-sdk init` to scaffold the Go project in `kserve-operator-deploy`.
2. Use `operator-sdk create api` to scaffold the Custom Resource `RawKServe`.
3. Implement the Go logic to parse and deploy the YAML assets sequentially.
4. Use `make docker-build docker-push` (to local registry or Minikube/Docker-desktop cache) and `make deploy`.

## Verification Plan
1. Reset the cluster and ensure Kubernetes is clean.
2. Delete or ignore `kserve-manual-deploy` entirely.
3. Deploy the operator Manager pod.
4. Apply a `RawKServe` CR to the cluster.
5. Verify the Operator successfully drives the KServe raw mode rollout by checking logs and pod readiness.
6. Run the `sklearn-iris` prediction test again.
