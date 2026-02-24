# KServe Raw Mode Manual Deployment Walkthrough

We have successfully prepared the required artifacts and tested the manual installation of KServe configured specifically for Raw deployment mode.

## 1. Directory Structure Organization
We created a clear and separated directory structure to hold the deployment manifests in `/Users/akashdeo/kserve-op/kserve-manual-deploy` to make the manual installation steps easy to follow:

- `01-cert-manager/cert-manager.yaml`: Standard cert-manager installation manifest.
- `02-kserve-crds/kserve-crds.yaml`: Extracted KServe CustomResourceDefinitions.
- `03-kserve-rbac/kserve-rbac.yaml`: Extracted KServe specific ClusterRoles, Roles, RoleBindings, and ServiceAccounts.
- `04-kserve-core/kserve-core.yaml`: Extracted Deployments, Services, and Webhook Configurations. Also contains the modified `inferenceservice-config` ConfigMap forcing `RawDeployment` as the default mode.
- `05-kserve-runtimes/kserve-cluster-resources.yaml`: Contains standard `ServingRuntimes` (sk-learn, torch, triton) required to boot predictors. This makes the manual deployment folder 100% self-contained.
- `install.sh`: A shell script demonstrating block-by-block execution of these manifests. (Updated with forced webhook waits to prevent race conditions on fresh clusters).
- `README.md`: Included instructions on how to manually apply each directory.

## 2. ConfigMap Modification
To ensure KServe starts up smoothly without Knative or Istio installed, we patched the `inferenceservice-config` JSON data inline:
```json
"defaultDeploymentMode": "RawDeployment"
```
This guarantees that standard Kubernetes Deployments/Services are provisioned when creating an `InferenceService`.

## 3. Server-Side Apply
During execution, we hit the known API server limit on annotation size (`metadata.annotations: Too long: may not be more than 262144 bytes`). We resolved this by modifying the scripts and instructions to use `kubectl apply --server-side` specifically for the KServe CRDs and Core Components.

## 4. Successful Deployment Logs
We executed the deployment script and confirmed the installation. The terminal output confirmed everything came up healthy:
```bash
...
deployment.apps/kserve-controller-manager serverside-applied
deployment.apps/kserve-localmodel-controller-manager serverside-applied
deployment.apps/llmisvc-controller-manager serverside-applied
daemonset.apps/kserve-localmodelnode-agent serverside-applied
...
Waiting for KServe Controller Manager to be ready...
pod/kserve-controller-manager-6899b656d-n5rrn condition met
pod/kserve-localmodel-controller-manager-5f64d4c9c6-f7lh7 condition met
pod/llmisvc-controller-manager-88f898488-6cn87 condition met
==========================================
   KServe Raw Mode Installation Complete
==========================================
```

## 5. Verification: Deploying a Sample Model
To verify the installation works, we deployed the standard `sklearn-iris` model. Since we are in Raw mode, we explicitly annotated the `InferenceService`.

1. **Install required Serving Runtimes:** (KServe needs these to map the predictors)
```bash
kubectl apply --server-side -f kserve-manual-deploy/05-kserve-runtimes/kserve-cluster-resources.yaml
```

2. **Deploy the `InferenceService`**:
```yaml
apiVersion: "serving.kserve.io/v1beta1"
kind: "InferenceService"
metadata:
  name: "sklearn-iris"
  annotations:
    serving.kserve.io/deploymentMode: "RawDeployment"
spec:
  predictor:
    sklearn:
      storageUri: "gs://kfserving-examples/models/sklearn/1.0/model"
```

3. **Verify Deployment**:
Because it is raw mode, KServe creates a standard `Deployment` and `Service` instead of Knative Revisions.
```bash
$ kubectl get pods -l serving.kserve.io/inferenceservice=sklearn-iris
NAME                                     READY   STATUS    RESTARTS   AGE
sklearn-iris-predictor-d46fd5cc9-p7gzw   1/1     Running   0          27s

$ kubectl get inferenceservice sklearn-iris
NAME           URL                                       READY   AGE
sklearn-iris   http://sklearn-iris-default.example.com   True    40s
```

4. **Test Inference**:
We port-forwarded the created service to test it locally:
```bash
kubectl port-forward svc/sklearn-iris-predictor 8080:80 &
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"instances": [[6.8, 2.8, 4.8, 1.4], [6.0, 3.4, 4.5, 1.6]]}' \
  "http://localhost:8080/v1/models/sklearn-iris:predict"
```

**Result:**
```json
{"predictions": [1, 1]}
```

The RawDeployment mode operates correctly without Istio or Knative overhead!

## 6. Standalone Validation
To guarantee the installation required absolutely no external dependencies or access to the original `kserve-master` repository, the entire cluster was reset. The `kserve-manual-deploy` folder was used exclusively to successfully reinstall KServe from scratch. The successful predictions validated the self-sufficiency of the exported configuration logs.

### Known Race Condition (Webhook `connection refused`)
During a completely fresh install (Step 5), you might initially see:
```text
failed calling webhook "llminferenceserviceconfig.kserve-webhook-server.validator"... dial tcp 10.98.201.216:443: connect: connection refused
```
**Why this happens**: KServe's Controller mounts Validating Webhooks that intercept the creation of `ServingRuntimes`. Even when the controller pod says it is `Ready`, it takes Kubernetes a few extra seconds to propagate the Pod IPs to the Webhook Service endpoints.
**The Fix**: Our `install.sh` and `README.md` now explicitly wait for the `kserve` namespace pods to become ready, **followed by a 15-second `sleep`**, ensuring the Service endpoints consistently map traffic before Step 5 applies the `ServingRuntimes`.

## 7. KServe Raw Mode Operator (Standalone)

We successfully developed and engineered a brand-new custom `KServeRawMode` operator. This operator acts as a fully self-contained controller to apply KServe sequentially without manual script execution.

### Operator Architecture:
- Scaffolding was initiated via **Operator SDK (`tools/operator-sdk`)** using the Go language structure.
- **Embedded Assets**: The previously generated `kserve-manual-deploy` manifests were embedded directly into the Go controller's binary via `//go:embed assets/*`. This makes the operator 100% standalone, without depending on filesystem paths.
- **Controller Logic**: The controller (`internal/controller/kserverawmode_controller.go`) utilizes KServe's `k8s.io/apimachinery/pkg/apis/meta/v1/unstructured` types and patches them via explicit **Server-Side Apply** to circumvent Kubernetes API payload constraints.
- **Race Condition Guard**: The programmatic installation loop explicitly crafts the `kserve` Namespace object first to satisfy RBAC creation and inserts a 15-second `time.Sleep` wait before submitting `ServingRuntimes`.

### Build & Deploy Execution:
The operator project is tracked in personal user Github space and deployed from DockerHub:
```bash
# 1. Image Build & Push to DockerHub (Multi-arch supporting amd64/arm64)
make docker-buildx IMG=akashneha/kserve-raw-operator:v4

# 2. Deploy the Controller Manager to the cluster
make deploy IMG=akashneha/kserve-raw-operator:v4
```

### Automated Trigger:
Once the Controller pod became `Ready`, we triggered the installation by submitting the Custom Resource:
```yaml
apiVersion: operator.akashdeo.com/v1alpha1
kind: KServeRawMode
metadata:
  name: kserverawmode-sample
spec: {}
```

### End-to-End Operator Verification:
Upon applying the CR, the Operator executed perfectly:
1. `cert-manager` Webhooks installed and registered successfully.
2. `KServe` controller deployed its RawMode `ConfigMap` logic.
3. Over a dozen `ClusterServingRuntimes` applied without webhook rejection.
4. We validated inference by port-forwarding our trusted test payload to a freshly spun `sklearn-iris` model!
