# Batch inference with BatchGateway: installation handoff

## Purpose

This document compares the Red Hat AI Inference 3.6 BatchGateway installation boundary with the current `odh-gitops` Helm charts. It is intended for the documentation team and covers the complete installation boundary for both:

- `charts/rhai-on-xks-chart` — non-OpenShift Kubernetes with the xKS chart.
- `charts/rhai-on-openshift-chart` — ODH/RHOAI on OpenShift with a DataScienceCluster.

BatchGateway is **user opt-in**. Enabling the AI Gateway module does not automatically enable BatchGateway.

> Batch inference is a Technology Preview feature in the referenced Red Hat documentation. Follow the support statement for the exact RHAII 3.6 product release being documented.
> **Version matrix:** the target RHAII/RHOAI product procedure is 3.6. The OpenShift chart metadata is `3.6.0`; the xKS chart metadata remains `3.5.0`, so this document does not claim that the xKS chart itself is a 3.6 product bundle. Before deployment, verify that the pinned AI Gateway and BatchGateway operator images provide the RHAII/RHOAI 3.6 CRD described here.

## Contents

- [Purpose](#purpose)
- [Versions used](#versions-used)
- [Pending 3.6 GA alignment](#pending-36-ga-alignment)
- [Executive summary](#executive-summary)
- [Three deployment paths](#three-deployment-paths)
  - [RHAII on xKS](#1-rhaii-on-xks)
  - [RHAII on OpenShift through the chart](#2-rhaii-on-openshift-through-the-chart)
  - [RHOAI through OpenShift AI](#3-rhoai-through-openshift-ai)
- [Ownership matrix](#ownership-matrix)
- [Terminology: Batch API versus Async Processor](#terminology-batch-api-versus-async-processor)
- [Prerequisites](#prerequisites)
  - [Cluster and tooling](#cluster-and-tooling)
  - [Backing services](#backing-services)
- [Step 1: Enable BatchGateway through the Helm chart](#step-1-enable-batchgateway-through-the-helm-chart)
  - [xKS without MaaS](#xks-without-maas)
  - [xKS with MaaS](#xks-with-maas)
  - [RHAII on OpenShift without MaaS](#rhaii-on-openshift-without-maas)
  - [RHAII on OpenShift with MaaS](#rhaii-on-openshift-with-maas)
  - [How RHOAI deploys BatchGateway through the DataScienceCluster](#how-rhoai-deploys-batchgateway-through-the-datasciencecluster)
  - [Verify operator enablement](#verify-operator-enablement)
- [Migration warning: sync to async is breaking](#migration-warning-sync-to-async-is-breaking)
- [Step 2: Prepare namespaces and access labels](#step-2-prepare-namespaces-and-access-labels)
- [Step 3: Configure model priority and flow control](#step-3-configure-model-priority-and-flow-control)
- [Step 4: Create the internal model gateway](#step-4-create-the-internal-model-gateway)
- [Step 5: Route the processor to the model](#step-5-route-the-processor-to-the-model)
- [Step 6: Configure authorization](#step-6-configure-authorization)
- [Step 7: Create storage credentials](#step-7-create-storage-credentials)
- [Step 8: Create the `LLMBatchGateway` workload resource](#step-8-create-the-llmbatchgateway-workload-resource)
  - [Async processor dispatch (required for RHAII 3.6)](#async-processor-dispatch-required-for-rhaii-36)
  - [SQL transport variant](#sql-transport-variant)
  - [Filesystem storage variant](#filesystem-storage-variant)
  - [TLS variants](#tls-variants)
  - [Optional async processor configuration](#optional-async-processor-configuration)
- [Step 9: Expose the external Batch API](#step-9-expose-the-external-batch-api)
- [Step 10: Verify the installation](#step-10-verify-the-installation)
- [Step 11: Submit a test batch](#step-11-submit-a-test-batch)
- [Gaps in the current `odh-gitops` charts](#gaps-in-the-current-odh-gitops-charts)
- [Recommended documentation split](#recommended-documentation-split)
- [Primary references](#primary-references)

## Versions used

| Item | Version or source | Evidence |
|---|---|---|
| Target RHAII/RHOAI procedure | 3.6 | This handoff scope |
| xKS chart | `3.5.0` for both chart `version` and `appVersion` | `charts/rhai-on-xks-chart/Chart.yaml` |
| OpenShift chart | `3.6.0` for both chart `version` and `appVersion` | `charts/rhai-on-openshift-chart/Chart.yaml` |
| ODH operator | Channel `fast-3`, `community-operators`, `opendatahub-operator` | `charts/rhai-on-openshift-chart/values.yaml` |
| RHOAI operator | Channel `beta`, `redhat-operators`, `rhods-operator` | `charts/rhai-on-openshift-chart/values.yaml` |
| xKS cert-manager chart dependency | `1.1.0` | `charts/rhai-on-xks-chart/Chart.yaml` |
| xKS cert-manager bundle | `v1.18.1` in values; rendered snapshots currently show operand image version `1.18.4` | `charts/rhai-on-xks-chart/values.yaml` and chart snapshots |
| OpenShift cert-manager operator | Channel `stable-v1` | `charts/rhai-on-openshift-chart/values.yaml` |
| OpenShift LeaderWorkerSet operator | Channel `stable-v1.0` | `charts/rhai-on-openshift-chart/values.yaml` |
| OpenShift RHCL operator | Channel `stable` | `charts/rhai-on-openshift-chart/values.yaml` |
| BatchGateway CRD | `batch.llm-d.ai/v1alpha1` | BatchGateway operator CRD |
| OpenShift DSC | `datasciencecluster.opendatahub.io/v3` | `charts/rhai-on-openshift-chart/templates/operator/datasciencecluster.yaml` |
| OpenShift DSCInitialization | `dscinitialization.opendatahub.io/v2` | `charts/rhai-on-openshift-chart/templates/operator/dscinitialization.yaml` |
| xKS AI Gateway, BatchGateway, and `llm-d-async` images | No default version pins in `values.yaml`; release `RELATED_IMAGE_*` values supply them | `charts/rhai-on-xks-chart/values.yaml` and `scripts/sync-e2e-images.sh` |
| xKS guide PR #319 head | `b70a29b` | Upstream PR head |
| MaaS guide PR #312 head | `241c6c6` | Upstream PR head |
| Async-only operator PR #125 head | `74c558c` | Upstream PR head |

## Pending 3.6 GA alignment

The docs team should track these pending upstream changes when finalizing the RHAII 3.6 GA procedure:

- [batch-gateway#319](https://github.com/opendatahub-io/batch-gateway/pull/319) aligns the xKS guide with the chart-managed AI Gateway path and the async-only `LLMBatchGateway` example. Its async example uses model-specific gateways, Redis queues, `prometheus-budget`, and an operator-managed `llm-d-async` processor.
- [batch-gateway#312](https://github.com/opendatahub-io/batch-gateway/pull/312) aligns MaaS integration with the operator-managed `LLMBatchGateway` CR and SQL async transport. It documents PostgreSQL-only stateful dependencies for that path, `sql.maxConnections` budgeting, operator-managed TLS/HTTPRoute resources, and MaaS namespace variance.
- [llm-d-batch-gateway-operator#125](https://github.com/opendatahub-io/llm-d-batch-gateway-operator/pull/125) is the dependency that removes the synchronous dispatch API. Its GA CRD requires `processor.asyncConfig`, removes `dispatchMode` and `globalInferenceGateway`, and retains `modelGateways` only for pool/objective/queue fields.

These PRs are not merged yet. Keep the chart opt-in and operator ownership statements here, then re-check the final operator CRD, image pins, route ownership, and Secret keys after the changes merge. Do not mix a post-#125 manifest with a pre-#125 operator release without validating that release's CRD.

## Executive summary

The Helm charts only enable the operator layer. They do not install or configure the complete batch inference data path.

The installation has four layers:

1. **Platform/operator enablement** — configured by this repository; the platform and AI Gateway operators perform the actual reconciliation.
2. **BatchGateway workload configuration** — an `LLMBatchGateway` custom resource, managed separately by the BatchGateway operator.
3. **Inference routing and flow control** — internal Gateway, HTTPRoutes, `InferenceObjective` resources, and authorization policies.
4. **Backing services** — PostgreSQL, Redis/Valkey, and S3-compatible storage or RWX filesystem storage.

The linked Red Hat AI Inference guide includes all four layers. The current charts cover only layer 1 and selected platform dependencies. The remaining resources must be supplied by the deployment or by a future GitOps layer.

## Three deployment paths

### 1. RHAII on xKS

The xKS chart does not use OLM and does not create a `DataScienceCluster` or `DSCInitialization`. Helm installs the `rhai-operator` Deployment directly, creates the `config.opendatahub.io/v1alpha2` `Platform` custom resource and component custom resources, and deploys the cloud-manager operators. The directly installed operator reconciles the `Platform` and component resources.

### 2. RHAII on OpenShift through the chart

The OpenShift chart uses `profile=rhaii` and `operator.type=rhoai`. Helm configures the RHOAI OLM subscription and renders `DSCInitialization` and `DataScienceCluster` resources. The `rhods-operator`, AI Gateway operator, and BatchGateway operator perform the reconciliation.

### 3. RHOAI through OpenShift AI

The platform team installs or manages RHOAI through the OpenShift AI operator and configures the `DataScienceCluster` directly. The RHOAI operator, AI Gateway operator, and BatchGateway operator then use the same DSC-based reconciliation path as the chart-driven RHAII-on-OpenShift option.

The second and third options differ in the installation/configuration entry point, not the underlying OpenShift operator architecture. The xKS option is the distinct non-OpenShift path: it uses the directly installed `rhai-operator` and `Platform` resources instead of OLM and DSC.

## Ownership matrix

| Resource or setting | xKS chart | OpenShift chart | Required action |
|---|---|---|---|
| AI Gateway operator | Creates through the Platform/module flow | ODH/RHOAI operator creates through DSC/OLM | Set BatchGateway to `Managed` |
| BatchGateway operator | Deployed by the AI Gateway operator when enabled | Deployed by the AI Gateway operator when enabled | Do not install a second copy |
| `LLMBatchGateway` CR | Not created by this chart | Not created by this chart | Create separately |
| PostgreSQL | Not managed | Not managed | Provide production PostgreSQL |
| Redis/Valkey | Not managed | Not managed | Required only for the `asyncConfig.redis` transport; SQL transport reuses PostgreSQL |
| S3/MinIO or RWX PVC | Not managed | Not managed | Provide file storage |
| Internal ClusterIP Gateway | Not created for BatchGateway | Not created for BatchGateway | Create separately |
| Internal model `HTTPRoute` | Not created | Not created | Create separately for each model |
| External batch API `HTTPRoute` | Not created by this chart | Not created by this chart | Use `spec.httpRoute` when the installed operator manages it; otherwise create separately |
| `InferenceObjective` resources | Not created | Not created | Create interactive and batch priorities |
| Kuadrant `AuthPolicy` / `RateLimitPolicy` | Not created | Not created | Optional; required if Kuadrant is the chosen policy layer |
| Batch API TLS / DestinationRule | Not created | Not created | Configure according to the gateway and certificate model; the operator may create the API Certificate |
| Related images | xKS chart receives digest-pinned related images | ODH/RHOAI bundle owns the images | Do not add manual image overrides unless the release requires them |

## Terminology: Batch API versus Async Processor

These terms describe different layers:

- **Asynchronous batch inference** is the user-facing OpenAI-compatible workflow: upload a JSONL file to `/v1/files`, create a job at `/v1/batches`, poll the job, and download the output file.
- **BatchGateway** is the API server, core processor, additional async processor, and garbage collector implementing that workflow.
- **Async dispatch** is required by the RHAII 3.6 product procedure. In the current chart/operator path it uses a Redis/Valkey or SQL queue plus dispatch gates and worker pools. It is not enabled merely by setting `batchGateway.managementState: Managed`.
- **The BatchGateway API is asynchronous even though the operator also runs a core processor and an additional async-processor**. Do not confuse the user-facing Batch API with the internal async processor.

The current `main` and the pending async-only operator release are transitional APIs. The primary examples below use the async-only field set and omit `dispatchMode` and `globalInferenceGateway`. If deploying against a pre-#125 operator release, add `dispatchMode: async`; after operator PR #125 merges, `processor.asyncConfig` is required and `processor.modelGateways` accepts only async model/queue fields. Use the CRD installed in the target 3.6 release as the source of truth.

## Prerequisites

### Cluster and tooling

- Kubernetes or OpenShift cluster supported by the target RHAI/RHOAI release.
- `helm` and `kubectl`; use `oc` on OpenShift.
- Gateway API CRDs and the inference extension CRDs required by the model-serving release.
- A deployed `LLMInferenceService`, `InferencePool`, and Endpoint Picker (EPP).
- The EPP must support `InferenceObjective` priorities for interactive versus batch traffic.
- `jq` for discovering the `InferencePool` associated with an `LLMInferenceService`.
- Kuadrant is optional. Use it for gateway authentication, authorization, and rate limiting when it is the selected policy layer.

### Backing services

Provide these before creating the `LLMBatchGateway` resource:

- PostgreSQL for BatchGateway metadata and job state.
- Redis or Valkey only for the `asyncConfig.redis` transport. With `asyncConfig.sql`, PostgreSQL carries both metadata and async queue state.
- S3-compatible object storage for input/output files, or an RWX PVC for filesystem storage.

Use production-grade persistence, backups, credentials, TLS, and high availability. The charts in this repository do not provision these services.

## Step 1: Enable BatchGateway through the Helm chart

### xKS without MaaS

Create an overlay such as `values-batch-gateway-xks.yaml`:

```yaml
components:
  kserve:
    gateway:
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              llm-d.ai/gateway-route: "true"
  aigateway:
    enabled: true
    spec:
      modelsAsAService:
        managementState: Removed
      batchGateway:
        managementState: Managed
    modelsAsAService:
      gateway:
        create: false
```

Install or upgrade the xKS chart with the cloud provider configuration required by the cluster:

```bash
PULL_SECRET_FILE=~/pull-secret.json

helm upgrade --install rhai-xks ./charts/rhai-on-xks-chart \
  --namespace rhai-system \
  --create-namespace \
  --set-file imagePullSecret.dockerConfigJson="${PULL_SECRET_FILE}" \
  -f values-batch-gateway-xks.yaml \
  -f values-cloud-provider.yaml
```

If you do not want a separate values file, use the equivalent inline configuration:

```bash
PULL_SECRET_FILE=~/pull-secret.json

helm upgrade --install rhai-xks ./charts/rhai-on-xks-chart \
  --namespace rhai-system \
  --create-namespace \
  --set-json 'components.kserve.gateway.allowedRoutes.namespaces={"from":"Selector","selector":{"matchLabels":{"llm-d.ai/gateway-route":"true"}}}' \
  --set components.aigateway.enabled=true \
  --set components.aigateway.spec.modelsAsAService.managementState=Removed \
  --set components.aigateway.modelsAsAService.gateway.create=false \
  --set components.aigateway.spec.batchGateway.managementState=Managed \
  --set-file imagePullSecret.dockerConfigJson="${PULL_SECRET_FILE}" \
  -f values-cloud-provider.yaml
```

The chart creates the `Platform` and `AIGateway` resources. The AI Gateway operator then deploys the BatchGateway operator from the release's pinned manifests and images.

Do **not** also apply the standalone operator overlay from the generic Red Hat AI Inference guide. That would create a second operator ownership path.

### xKS with MaaS

Use this profile when Models as a Service owns the external gateway and API-key authentication. Enable both MaaS and BatchGateway in the AI Gateway component:

```yaml
components:
  kserve:
    gateway:
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              llm-d.ai/gateway-route: "true"
  aigateway:
    enabled: true
    spec:
      modelsAsAService:
        managementState: Managed
      batchGateway:
        managementState: Managed
    modelsAsAService:
      gateway:
        create: true
        allowedRoutes:
          namespaces:
            from: Selector
            selector:
              matchLabels:
                maas-gateway-access: "true"
```

Install or upgrade with a separate values file, for example:

```bash
PULL_SECRET_FILE=~/pull-secret.json

helm upgrade --install rhai-xks ./charts/rhai-on-xks-chart \
  --namespace rhai-system \
  --create-namespace \
  --set-file imagePullSecret.dockerConfigJson="${PULL_SECRET_FILE}" \
  -f values-batch-gateway-xks-maas.yaml \
  -f values-cloud-provider.yaml
```

The MaaS gateway and BatchGateway API must use compatible route-attachment labels. The MaaS integration uses API keys and forwards `Authorization` plus any required subscription header through `passThroughHeaders`.

The xKS chart's relevant default is in `charts/rhai-on-xks-chart/values.yaml`:

```yaml
components:
  aigateway:
    enabled: false
    spec:
      batchGateway:
        managementState: Removed
```

### RHAII on OpenShift without MaaS

Create an overlay such as `values-batch-gateway-openshift.yaml`:

```yaml
components:
  kserve:
    dsc:
      managementState: Managed
    gateway:
      spec:
        gatewayClassName: openshift-ai-inference
        listeners:
          - name: https
            port: 443
            protocol: HTTPS
            allowedRoutes:
              namespaces:
                from: Selector
                selector:
                  matchLabels:
                    llm-d.ai/gateway-route: "true"
  aigateway:
    dsc:
      managementState: Managed
      modelsAsAService:
        managementState: Removed
      batchGateway:
        managementState: Managed
```

If the required AI Gateway dependencies are not already present, configure them according to the product release:

```yaml
components:
  aigateway:
    dependencies:
      certManager: true
      leaderWorkerSet: true
      rhcl: true
```

The `rhaii` profile selects the RHAII inference stack. On the RHOAI platform, use `operator.type=rhoai`; this chart path still renders `DSCInitialization` and `DataScienceCluster` resources for the RHOAI operator:

```bash
helm upgrade --install rhoai ./charts/rhai-on-openshift-chart \
  --namespace rhoai-gitops \
  --create-namespace \
  --set profile=rhaii \
  --set operator.type=rhoai \
  -f values-batch-gateway-openshift.yaml
```

The chart renders the following DSC fragment:

```yaml
spec:
  components:
    aigateway:
      managementState: Managed
      batchGateway:
        managementState: Managed
      modelsAsAService:
        managementState: Removed
```

The RHAII profile does not bypass the RHOAI operator or DSC reconciliation. The chart renders the DSC and its dependencies; the RHOAI operator and AI Gateway operator own reconciliation.

### RHAII on OpenShift with MaaS

Use the same `rhaii` profile, but set the AI Gateway MaaS sub-component to `Managed` and configure the MaaS Gateway listener's route selector:

```yaml
components:
  kserve:
    dsc:
      managementState: Managed
    gateway:
      spec:
        gatewayClassName: openshift-ai-inference
        listeners:
          - name: https
            port: 443
            protocol: HTTPS
            allowedRoutes:
              namespaces:
                from: Selector
                selector:
                  matchLabels:
                    llm-d.ai/gateway-route: "true"
  aigateway:
    dsc:
      managementState: Managed
      modelsAsAService:
        managementState: Managed
      batchGateway:
        managementState: Managed
    modelsAsAService:
      gateway:
        spec:
          gatewayClassName: maas-gateway-class
          listeners:
            - name: https
              port: 443
              protocol: HTTPS
              tls:
                mode: Terminate
                certificateRefs:
                  - group: ""
                    kind: Secret
                    name: <maas-gateway-tls-secret>
              allowedRoutes:
                namespaces:
                  from: Selector
                  selector:
                    matchLabels:
                      maas-gateway-access: "true"
```

Install with the RHOAI operator:

```bash
helm upgrade --install rhoai ./charts/rhai-on-openshift-chart \
  --namespace rhoai-gitops \
  --create-namespace \
  --set profile=rhaii \
  --set operator.type=rhoai \
  -f values-batch-gateway-openshift-maas.yaml
```

Label every namespace that attaches routes to the MaaS Gateway with `maas-gateway-access=true`; apply that label to the BatchGateway and model namespaces before creating their HTTPRoutes. Provide the referenced `<maas-gateway-tls-secret>` before creating the HTTPS Gateway. In the MaaS profile, use the SQL async transport, keep PostgreSQL available, and configure `passThroughHeaders` for `Authorization` and any required MaaS subscription header.

### How RHOAI deploys BatchGateway through the DataScienceCluster

**Terminology:** RHAII is the inference stack/profile configured by the chart. RHOAI is the OpenShift AI platform and `rhods-operator` selected with `operator.type=rhoai`; the RHOAI operator reconciles the `DataScienceCluster`. The RHAII profile narrows which DSC components are enabled; it does not replace the DSC path.

For RHOAI on OpenShift, the chart does not install the BatchGateway workloads directly. The deployment chain is:

1. Helm installs or configures the RHOAI operator subscription with `operator.type=rhoai`.
2. Helm renders the `DataScienceCluster` resource, normally `default-dsc`.
3. The RHOAI operator watches the DSC and reconciles its managed components and dependencies.
4. `spec.components.aigateway.batchGateway.managementState: Managed` enables BatchGateway through the AI Gateway component.
5. The AI Gateway operator deploys the BatchGateway operator.
6. The BatchGateway operator watches `LLMBatchGateway` resources and creates the API server, processor, async processor, garbage collector, Certificate, and any operator-managed HTTPRoute.

The DSC fragment for a non-MaaS RHAII deployment is:

```yaml
apiVersion: datasciencecluster.opendatahub.io/v3
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  components:
    kserve:
      managementState: Managed
    aigateway:
      managementState: Managed
      modelsAsAService:
        managementState: Removed
      batchGateway:
        managementState: Managed
```

For MaaS, change `modelsAsAService.managementState` to `Managed` and configure the MaaS Gateway listener, TLS Secret, and route selector as shown above. Keep `batchGateway.managementState: Managed` in both profiles. Do not create a second DSC or install the BatchGateway operator with a standalone Kustomize overlay.

Verify each reconciliation layer independently:

```bash
oc get subscription rhods-operator -n redhat-ods-operator
oc get csv -n redhat-ods-operator
oc get datasciencecluster default-dsc -o yaml
oc get pods -n redhat-ods-applications
oc get deploy -n redhat-ods-applications
oc get crd llmbatchgateways.batch.llm-d.ai
oc get pods -A -l app.kubernetes.io/name=batch-gateway-operator
```

### Verify operator enablement

On OpenShift:

```bash
oc get datasciencecluster default-dsc -o yaml
oc get crd llmbatchgateways.batch.llm-d.ai
oc get pods -A -l app.kubernetes.io/name=batch-gateway-operator
```

On xKS:

```bash
kubectl get crd llmbatchgateways.batch.llm-d.ai
kubectl get pods -A -l app.kubernetes.io/name=batch-gateway-operator
kubectl get platform default -o yaml
kubectl get aigateway default-aigateway -o yaml
```

The exact namespace and labels can vary by release. Use the rendered resources and operator documentation as the authority.

The current async operator API puts `inferenceGateway` under `spec.processor.asyncConfig`, `modelGateways` under `spec.processor`, and the Batch API `certManager` configuration directly under `spec.tls`. Product bundles can pin an older operator version, so confirm the installed CRD before copying the workload examples:

```bash
kubectl explain llmbatchgateway.spec.processor --recursive
kubectl explain llmbatchgateway.spec.processor.asyncConfig --recursive
kubectl explain llmbatchgateway.spec.tls --recursive
```

The xKS chart snapshot confirms that the chart creates the BatchGateway operator ServiceAccount and related platform resources, not the `LLMBatchGateway` workload, backing services, or batch routes.

## Migration warning: sync to async is breaking

RHAII 3.6 supports asynchronous dispatch only. Based on [operator PR #125](https://github.com/opendatahub-io/llm-d-batch-gateway-operator/pull/125), the async-only operator change is not an in-place conversion for an existing synchronous deployment:

1. Stop submitting new batches.
2. Drain active batches and the PostgreSQL-backed job state. Confirm that no batches remain in non-terminal states and that processor leases or pending work are cleared using the Batch API, operator status, and application metrics or logs.
3. If the old deployment was already asynchronous, also drain its Redis/Valkey or SQL dispatch queue. A synchronous deployment has no async dispatch queue; do not invent one during the cutover.
4. Keep PostgreSQL and the file-storage backend. Do not manually delete PostgreSQL rows or completed results; PostgreSQL and object storage contain migration data.
5. After active work, PostgreSQL-backed pending state, and any async transport queue are drained, delete the old `LLMBatchGateway` resource.
6. Deploy the async-only operator release and apply a new CR with `processor.asyncConfig` and exactly one transport (`redis` or `sql`). Do not restore `dispatchMode: sync` or `globalInferenceGateway`.

Deleting the old resource before the drain loses in-flight batches. Preserve PostgreSQL and object storage through the cutover; they are retained data stores, not resources to empty or delete.

## Step 2: Prepare namespaces and access labels

Use a dedicated namespace for the BatchGateway workload:

```bash
export BATCH_NAMESPACE=batch-gateway
export MODEL_NAMESPACE=<llm-model-namespace>
export MODEL_NAME=<user-facing-model-name>
export GATEWAY_NAMESPACE=<gateway-namespace>

kubectl create namespace "${BATCH_NAMESPACE}"
kubectl label namespace "${BATCH_NAMESPACE}" \
  llm-d.ai/gateway-route=true --overwrite
kubectl label namespace "${MODEL_NAMESPACE}" \
  llm-d.ai/gateway-route=true --overwrite
# Add this label when using a MaaS profile:
kubectl label namespace "${BATCH_NAMESPACE}" "${MODEL_NAMESPACE}" \
  maas-gateway-access=true --overwrite
```

The `llm-d.ai/gateway-route=true` label satisfies the xKS chart's KServe Gateway attachment selector and is also used by the BatchGateway internal Gateway example below. The `maas-gateway-access=true` label is additionally required when the selected MaaS Gateway listener uses that route selector. The exact labels and listener policies must match the installed release.

## Step 3: Configure model priority and flow control

Create an interactive priority and a lower-priority batch objective for the model's `InferencePool`:

```yaml
apiVersion: <inference-objective-api-version>
kind: InferenceObjective
metadata:
  name: interactive-default
  namespace: <model-namespace>
spec:
  priority: 100
  poolRef:
    name: <inference-pool-name>
---
apiVersion: <inference-objective-api-version>
kind: InferenceObjective
metadata:
  name: batch-sheddable
  namespace: <model-namespace>
spec:
  priority: -1
  poolRef:
    name: <inference-pool-name>
```

The API group differs by platform/release. The linked Red Hat guide uses the xKS inference API for xKS and the OpenShift inference API for OpenShift. Confirm with:

```bash
kubectl api-resources | grep -i inferenceobjective
kubectl get inferencepool -n "${MODEL_NAMESPACE}"
```

The BatchGateway processor must send `batch-sheddable` as its `inferenceObjective`. Without that value, batch traffic can use the default priority and compete with interactive requests.

## Step 4: Create the internal model gateway

The internal gateway must be ClusterIP-only. It is not the public Batch API endpoint.

For xKS with the chart's Istio-based gateway stack:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: batch-internal-gateway
  namespace: <gateway-namespace>
  annotations:
    networking.istio.io/service-type: ClusterIP
spec:
  gatewayClassName: istio
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: Selector
          selector:
            matchLabels:
              llm-d.ai/gateway-route: "true"
```

Apply and wait:

```bash
kubectl apply -f batch-internal-gateway.yaml
kubectl wait --for=condition=Programmed \
  --timeout=300s \
  -n "${GATEWAY_NAMESPACE}" \
  gateway/batch-internal-gateway
```

For OpenShift, use the GatewayClass and gateway namespace selected by the RHOAI/OpenShift inference deployment. Do not assume that `istio`, `openshift-default`, and `openshift-ai-inference` are interchangeable.

## Step 5: Route the processor to the model

Create one HTTPRoute per model. The route must:

1. Match the path used by the BatchGateway processor.
2. Rewrite it to the model server endpoint.
3. Send it to the model's `InferencePool`.

Example for xKS:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: batch-llm-route
  namespace: <model-namespace>
spec:
  parentRefs:
    - name: batch-internal-gateway
      namespace: <gateway-namespace>
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /<model-namespace>/<model-name>/v1/chat/completions
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/chat/completions
      backendRefs:
        - group: inference.networking.x-k8s.io
          kind: InferencePool
          name: <inference-pool-name>
    - matches:
        - path:
            type: PathPrefix
            value: /<model-namespace>/<model-name>/v1/completions
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /v1/completions
      backendRefs:
        - group: inference.networking.x-k8s.io
          kind: InferencePool
          name: <inference-pool-name>
```

For OpenShift/RHOAI, use the OpenShift inference API group shown by the installed `InferencePool` resource, commonly `inference.networking.k8s.io`, and include the supported paths such as `/v1/chat/completions`, `/v1/completions`, `/v1/embeddings`, `/v1/responses`, and `/v1/moderations` as required by the release guide.

The BatchGateway processor URL must contain the same route prefix:

```text
http://<internal-gateway-service>.<gateway-namespace>.svc.cluster.local/<model-namespace>/<model-name>
```

A mismatch between the processor URL and the HTTPRoute path is a common failure mode.

## Step 6: Configure authorization

The internal route should preserve the caller identity when the processor invokes the model route.

At minimum, configure the API server to pass through the caller's bearer token:

```yaml
apiServer:
  config:
    batchAPI:
      passThroughHeaders:
        - Authorization
```

If using Kuadrant, create an `AuthPolicy` on the internal route. The policy should:

- Validate the Kubernetes token.
- Authorize access to the target model or `InferencePool`/`LLMInferenceService`.
- Inject the authenticated identity header expected by the BatchGateway and model policy.

The exact `AuthPolicy` resource attributes differ between xKS and OpenShift. Use the linked product guide for the installed API group and model resource. Do not copy an OpenShift policy verbatim to xKS.

Use a separate `AuthPolicy` on the external Batch API route if the API must require authentication. A `RateLimitPolicy` is optional and should protect `/v1/batches` and `/v1/files` without applying interactive token-rate limits to the internal model route.

## Step 7: Create storage credentials

Create the Secret in the same namespace as the `LLMBatchGateway` resource:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: batch-gateway-secrets
  namespace: <batch-namespace>
type: Opaque
stringData:
  redis-url: redis://<redis-host>:<redis-port>/0
  postgresql-url: postgresql://<db-user>:<db-password>@<db-host>:<db-port>/<db-name>?sslmode=disable
  s3-secret-access-key: <s3-secret-key>
```

For Redis transport, include `postgresql-url`, `redis-url`, and `s3-secret-access-key`. For SQL transport, the processor and async processor reuse `postgresql-url`; omit `redis-url` and keep `s3-secret-access-key` when using S3. The S3 endpoint, region, bucket, and access key ID are non-sensitive and belong in the `LLMBatchGateway` spec. Protect this Secret with normal Kubernetes RBAC and external secret-management controls.

## Step 8: Create the `LLMBatchGateway` workload resource

The BatchGateway operator watches this resource and deploys the API server, core processor, additional async processor, and garbage collector. The repository Helm charts do not create it.

The CRD is:

```text
apiVersion: batch.llm-d.ai/v1alpha1
kind: LLMBatchGateway
```

### Async processor dispatch (required for RHAII 3.6)

RHAII 3.6 uses the queue-based asynchronous processor. Do not use the synchronous `dispatchMode: sync` examples from older product documentation for the 3.6 product procedure. Do not configure `processor.globalInferenceGateway`; in async mode, configure `processor.modelGateways` and `processor.asyncConfig.inferenceGateway`.

For the documented 3.6 product configuration:

- `processor.asyncConfig` is required by operator PR #125 and contains exactly one of `redis` or `sql`.
- `processor.modelGateways` entries contain `inferencePoolName` plus optional objective and queue fields.
- With a pre-#125 operator release only, add `processor.dispatchMode: async`; omit that removed field for the GA async-only CRD.

The example below uses Redis/Valkey transport, Prometheus saturation gates, and Redis-quota worker pools.

The current `main` CRD uses `dbBackend: postgresql` and supports both Redis and SQL async transports. The RHAII 3.6 product path can use either transport when the selected operator image and backing services support it; the examples below show both.

```yaml
spec:
  secretRef:
    name: batch-gateway-secrets
  dbBackend: postgresql
  fileStorage:
    s3:
      region: <s3-region>
      endpoint: https://<s3-endpoint>
      accessKeyId: <s3-access-key-id>
      bucket: <s3-bucket-name>
      usePathStyle: true
      autoCreateBucket: true
  apiServer:
    replicas: 1
    config:
      batchAPI:
        passThroughHeaders:
          - Authorization
  processor:
    replicas: 1
    config:
      inputFileDiskBudgetPercent: 10
      progressUpdateInterval: 15s
      resumable:
        leaseDuration: 5m
        finalizeConcurrency: 16
        finalizeTimeout: 30m
    modelGateways:
      <model-id>:
        inferencePoolName: <inference-pool-name>
        inferenceObjective: batch-sheddable
    asyncConfig:
      resultPollTimeout: 30s
      inferenceGateway:
        url: http://<internal-inference-gateway-service>.<gateway-namespace>.svc.cluster.local
      redis:
        requestPathURL: /v1/completions
        pollIntervalMs: 1000
        batchSize: 10
        queuesConfig:
          - name: "llm-d-async:requests:<inference-pool-name>"
            workerPoolID: default
            gateType: prometheus-saturation
            gateParams:
              prometheus_url: http://<prometheus-service>.<monitoring-namespace>.svc.cluster.local:9090
              saturation_threshold: "0.8"
      workerPools:
        - name: default
          workers: 8
          gateType: redis-quota
          gateParams:
            quota_key: dispatch-quota
      prometheusURL: http://<prometheus-service>.<monitoring-namespace>.svc.cluster.local:9090
  gc:
    replicas: 1
    interval: 30m

```

The current `main` CRD exposes both `asyncConfig.redis` and `asyncConfig.sql`; they are mutually exclusive. Redis/Valkey credentials are read from the referenced Secret when the Redis transport is selected. PostgreSQL is required for the documented metadata path and for SQL transport.

On pre-#125 operator releases, `dispatchMode: async` causes the BatchGateway operator to deploy the additional `llm-d-async` processor. Operator PR #125 deploys that processor unconditionally because async is the only dispatch path. Do not install a separate `llm-d-async` deployment.

### SQL transport variant

Use SQL transport when PostgreSQL is sized for both BatchGateway metadata and async queue traffic:

```yaml
processor:
  modelGateways:
    <model-id>:
      inferencePoolName: <inference-pool-name>
      requestQueueName: "llm-d-async:requests:<inference-pool-name>"
      resultQueueName: "llm-d-async:results:<inference-pool-name>"
  asyncConfig:
    resultPollTimeout: 30s
    inferenceGateway:
      url: http://<internal-inference-gateway-service>.<gateway-namespace>.svc.cluster.local
    prometheusURL: http://<prometheus-service>.<monitoring-namespace>.svc.cluster.local:9090
    sql:
      maxConnections: 8
      pollIntervalMs: 1000
      batchSize: 10
      queuesConfig:
        - name: "llm-d-async:requests:<inference-pool-name>"
          workerPoolID: default
          gateType: prometheus-budget
          gateParams:
            pool: <inference-pool-name>
            max_concurrency: "100"
            baseline: "0.05"
```

For the pending GA SQL integration, also apply these constraints:

- `sql.maxConnections` is per BatchGateway processor replica; budget `processor replicas × maxConnections` plus the async-processor pool, API server connections, and garbage-collector connections against PostgreSQL `max_connections`.
- Set `requestQueueName` and `resultQueueName` together or omit both.
- Use a distinct `inferencePoolName` for each model gateway; the SQL transport has one destructive result queue per model.
- Ensure the pinned BatchGateway and `llm-d-async` images include SQL producer support before selecting this transport.

### Filesystem storage variant

S3-compatible storage and filesystem storage are mutually exclusive. For a pre-created RWX PVC, replace `fileStorage.s3` with:

```yaml
fileStorage:
  fs:
    claimName: batch-gateway-files
    basePath: /batch-gateway
```

The PVC, access mode, capacity, backup, and retention policy are outside the `odh-gitops` charts and must be provisioned before the `LLMBatchGateway` resource.

### TLS variants

For the BatchGateway API server, configure either cert-manager or an existing TLS Secret, never both:

```yaml
tls:
  enabled: true
  certManager:
    issuerName: <issuer-name>
    issuerKind: ClusterIssuer
    dnsNames:
      - batch-gateway-apiserver
      - batch-gateway-apiserver.<batch-namespace>.svc.cluster.local
      - localhost
```

```yaml
tls:
  enabled: true
  secretName: batch-gateway-api-tls
```

The async processor's connection to the inference gateway has a separate TLS configuration:

```yaml
processor:
  asyncConfig:
    tls:
      secretName: inference-gateway-tls
      insecureSkipVerify: false
      caCertKey: ca.crt
      certKey: tls.crt
      keyKey: tls.key
```

The Secret keys and mounted certificate paths must match the installed 3.6 operator chart. Do not enable insecure verification in production.

### Optional async processor configuration

The 3.6 operator also supports these optional async settings. Confirm field availability in the installed CRD:

```yaml
processor:
  asyncConfig:
    concurrency: 8
    drainTimeout: 2m
    resultPollTimeout: 30s
    prometheusURL: http://<prometheus-service>.<monitoring-namespace>.svc.cluster.local:9090
    prometheusCacheTTL: 30s
    transformConfig:
      requestTransforms:
        - name: <transform-name>
          type: <transform-type>
          parameters:
            <parameter-name>: <parameter-value>
    modelServerMonitor:
      enabled: true
      selector:
        llm-d.ai/role: decode
      port: modelserver
      path: /metrics
      interval: 15s
```

`asyncConfig.replicas` is reserved by the current operator and does not change the async-processor replica count. Do not use it as a scaling control without confirming a later operator version.

## Step 9: Expose the external Batch API

For the GA operator path represented by [batch-gateway#312](https://github.com/opendatahub-io/batch-gateway/pull/312), prefer the operator-managed route:

```yaml
spec:
  httpRoute:
    enabled: true
    parentRefs:
      - name: <external-gateway-name>
        namespace: <gateway-namespace>
```

The operator creates the HTTPRoute for the BatchGateway API and, in the MaaS integration, also creates the API TLS `Certificate`. Configure the external Gateway, authentication policy, hostname, and listener separately. Do not create a second route for the same API listener.

If the installed 3.6 operator does not yet support `spec.httpRoute`, create the route manually:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: batch-api-route
  namespace: <batch-namespace>
spec:
  parentRefs:
    - name: <external-gateway-name>
      namespace: <gateway-namespace>
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v1/batches
        - path:
            type: PathPrefix
            value: /v1/files
      backendRefs:
        - name: batch-gateway-apiserver
          port: 8000
```

If the API server terminates TLS with a self-signed cert, configure the Gateway's backend TLS policy/DestinationRule as described by the product guide. `insecureSkipVerify: true` is acceptable only as a development workaround. Production deployments should use a trusted CA and set verification accordingly.

Use exactly one of the operator-managed or manual route paths. The external Gateway, authentication policy, hostname, and listener configuration remain deployment-specific.

## Step 10: Verify the installation

```bash
kubectl get llmbatchgateway -n <batch-namespace>
kubectl get pods -n <batch-namespace>
kubectl get svc -n <batch-namespace>
kubectl get gateway -A
kubectl get httproute -A
kubectl get inferenceobjective -n <model-namespace>
```

Expected BatchGateway workloads:

- API server.
- Core processor StatefulSet.
- Additional async-processor Deployment.
- Garbage collector.
- PostgreSQL, Redis/Valkey, and object-storage connectivity available from the workload namespace.

Check conditions and logs:

```bash
kubectl describe llmbatchgateway batch-gateway -n <batch-namespace>
kubectl get statefulset,deployments -n <batch-namespace>
kubectl logs -n <batch-namespace> statefulset/<processor-statefulset-name>
kubectl logs -n <batch-namespace> deployment/<async-processor-deployment-name>
kubectl logs -n <batch-namespace> deployment/<api-server-deployment-name>
```

The internal Gateway and both HTTPRoutes must be programmed and accepted. A running operator alone does not prove that model routing or external API routing works.

## Step 11: Submit a test batch

Create JSONL input with one OpenAI-compatible request per line:

```json
{"custom_id":"test-1","method":"POST","url":"/v1/chat/completions","body":{"model":"<model-name>","messages":[{"role":"user","content":"Hello"}],"max_tokens":10}}
```

Upload the file:

```bash
curl -sk -X POST "https://<batch-api-host>/v1/files" \
  -H "Authorization: Bearer <token>" \
  -F "file=@test-batch.jsonl" \
  -F "purpose=batch"
```

Create the job:

```bash
curl -sk -X POST "https://<batch-api-host>/v1/batches" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "input_file_id": "<file-id>",
    "endpoint": "/v1/chat/completions",
    "completion_window": "24h"
  }'
```

Poll and download the output:

```bash
curl -sk "https://<batch-api-host>/v1/batches/<batch-id>" \
  -H "Authorization: Bearer <token>"

curl -sk "https://<batch-api-host>/v1/files/<output-file-id>/content" \
  -H "Authorization: Bearer <token>" \
  -o output.jsonl
```

A successful job transitions through states such as `validating`, `in_progress`, `finalizing`, and `completed`.

## Gaps in the current `odh-gitops` charts

The following are not currently chart-managed and are the main documentation/integration gap:

1. `LLMBatchGateway` CR generation.
2. Redis/Valkey, PostgreSQL, and S3/MinIO/RWX storage deployment.
3. Batch internal ClusterIP Gateway.
4. Per-model internal HTTPRoutes and URL rewrites.
5. External `/v1/batches` and `/v1/files` HTTPRoute when operator-managed `spec.httpRoute` is not enabled or unavailable.
6. `InferenceObjective` priority resources.
7. AuthPolicy and RateLimitPolicy resources.
8. Batch API TLS backend policy/DestinationRule; the operator-managed API `Certificate` does not replace backend trust configuration.
9. Version-specific async processor configuration.
10. End-to-end verification and sample Batch API client workflow.

The xKS chart's existing BatchGateway README section correctly documents operator opt-in but currently stops before these workload and routing resources. The OpenShift chart exposes the DSC opt-in field but does not yet have a BatchGateway installation section.

## Recommended documentation split

For the product documentation team:

- Keep the Red Hat product guide as the complete cluster-admin procedure.
- Add a short `odh-gitops` chart section showing only the xKS/OpenShift opt-in values and clearly linking to the product procedure for backing services, routes, policies, and `LLMBatchGateway` configuration.
- State that xKS chart-managed AI Gateway enablement replaces the older standalone BatchGateway operator-install step; do not instruct users to install a second operator.
- Add a version note wherever `LLMBatchGateway` processor fields differ between the product bundle and upstream `llm-d-batch-gateway-operator`.
- Reuse the existing xKS `azure-with-batch-gateway` snapshot as the chart opt-in evidence; add or update snapshots only when the opt-in rendering changes.

## Primary references

- Red Hat AI Inference 3.6 Batch inference guide for the target product release. The direct 3.6 documentation URL was not available during validation; do not substitute the 3.5 URL.
- [xKS chart values](../charts/rhai-on-xks-chart/values.yaml)
- [xKS chart BatchGateway section](../charts/rhai-on-xks-chart/README.md#batch-gateway)
- [xKS BatchGateway rendered snapshot](../charts/rhai-on-xks-chart/test/snapshots/azure-with-batch-gateway.snap.yaml)
- [OpenShift chart values](../charts/rhai-on-openshift-chart/values.yaml)
- [OpenShift chart DSC template](../charts/rhai-on-openshift-chart/templates/operator/datasciencecluster.yaml)
- [llm-d BatchGateway operator API, current main branch](https://github.com/opendatahub-io/llm-d-batch-gateway-operator/blob/main/api/v1alpha1/llmbatchgateway_types.go)
- [llm-d BatchGateway installation guide](https://llm-d.ai/docs/operations/batch-serving/batch-gateway)
- [llm-d async processing guide](https://llm-d.ai/docs/operations/batch-serving/asynchronous-processing)
