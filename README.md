# KServe Raw Mode Operator

This repository contains a fully standalone Kubernetes Operator for installing [KServe](https://kserve.github.io/website/) exclusively in **Raw Deployment Mode** (without Knative or Istio dependencies).

## Prerequisites
- A vanilla Kubernetes cluster (v1.25+)
- Internet access for the cluster nodes to pull required images (`cert-manager`, `kserve-controller`, etc.)

## Quick Start Installation

You only need two commands to get KServe running on a completely fresh cluster. The operator image is multi-architecture (`linux/amd64`, `linux/arm64`, `linux/s390x`, `linux/ppc64le`), so it runs gracefully on macOS Apple Silicon, RHEL 9 (amd64), AWS Graviton, and more.

### 1. Deploy the Operator
Apply the operator manifest directly from GitHub. This will install the Custom Resource Definitions, RBAC, and the Operator Controller pod.

```sh
kubectl apply -f https://raw.githubusercontent.com/skyakash/kserve-raw-operator/main/kserve-raw-operator.yaml
```

Wait for the operator pod to be ready:
```sh
kubectl wait --for=condition=Available deployment/kserve-operator-deploy-controller-manager -n kserve-operator-deploy-system --timeout=120s
```

### 2. Trigger KServe Installation
Once the operator is running, trigger the installation sequence by applying the `KServeRawMode` custom resource:

```sh
kubectl apply -f https://raw.githubusercontent.com/skyakash/kserve-raw-operator/main/kserverawmode-sample.yaml
```

**What happens next?**
The operator will automatically execute the exact sequence necessary to bring up KServe in raw mode safely:
1. Installs `cert-manager`.
2. Applies KServe CRDs.
3. Configures KServe `Namespace` and RBAC.
4. Applies KServe Core Components (with the `defaultDeploymentMode: RawDeployment` ConfigMap parameter).
5. Waits for Webhook Readiness to avoid `connection refused` race conditions.
6. Installs all standard KServe `ClusterServingRuntimes`.

You can watch the operator orchestrate this in real-time by checking its logs:
```sh
kubectl logs -n kserve-operator-deploy-system -l control-plane=controller-manager -c manager -f
```

## Contributing
// TODO(user): Add detailed information on how you would like others to contribute to this project

### How it works
This project aims to follow the Kubernetes [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/).

It uses [Controllers](https://kubernetes.io/docs/concepts/architecture/controller/),
which provide a reconcile function responsible for synchronizing resources until the desired state is reached on the cluster.

### Test It Out
1. Install the CRDs into the cluster:

```sh
make install
```

2. Run your controller (this will run in the foreground, so switch to a new terminal if you want to leave it running):

```sh
make run
```

**NOTE:** You can also run this in one step by running: `make install run`

### Modifying the API definitions
If you are editing the API definitions, generate the manifests such as CRs or CRDs using:

```sh
make manifests
```

**NOTE:** Run `make --help` for more information on all potential `make` targets

More information can be found via the [Kubebuilder Documentation](https://book.kubebuilder.io/introduction.html)

## License

Copyright 2026.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

