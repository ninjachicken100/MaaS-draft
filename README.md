# qwen3-06b Model Deployment Overview

## Pod Roles

### Ingress & Gateway

| Pod | Namespace | Role |
|-----|-----------|------|
| `router-default-*` | `openshift-ingress` | **HAProxy ingress router.** OpenShift's built-in reverse proxy. Runs with `hostNetwork: true`, binding directly to the node's port 80/443. Handles all inbound HTTP/HTTPS traffic to `*.apps-crc.testing`. For MaaS routes, it does **TLS passthrough** — reads the SNI hostname to pick the right backend, then forwards the raw encrypted bytes without decrypting. |
| `maas-default-gateway-openshift-default-*` | `openshift-ingress` | **Istio/Envoy proxy pod.** The actual MaaS gateway. Created by Istio when the `Gateway` resource (`maas-default-gateway`) is applied. Receives TLS traffic forwarded by HAProxy, terminates TLS, enforces auth policies (via Authorino), and routes requests to the correct backend based on HTTPRoute rules. |
| `istiod-openshift-gateway-*` | `openshift-ingress` | **Istio control plane.** Manages and configures all Envoy proxies including `maas-default-gateway`. Watches Gateway and HTTPRoute resources and pushes routing config to Envoy via xDS protocol. |
| `routes-controller-*` | `openshift-ingress` | Syncs OpenShift Routes from Gateway API resources. Created the `maas-gateway` OpenShift Route with `tls.termination: passthrough` so HAProxy knows to forward traffic to the Istio gateway pod. |

---

### Auth & Rate Limiting (Kuadrant)

| Pod | Namespace | Role |
|-----|-----------|------|
| `authorino-*` | `rh-connectivity-link` | **Auth enforcement engine.** Called by the Envoy gateway on every request. Evaluates `AuthPolicy` rules: runs Kubernetes TokenReview (validates the JWT), then SubjectAccessReview (checks RBAC permission to call the model), then optional metadata lookups. Returns allow/deny to Envoy. If it returns deny, Envoy sends back 403. |
| `authorino-operator-*` | `rh-connectivity-link` | Manages the Authorino deployment and its configuration. |
| `limitador-*` | `rh-connectivity-link` | **Rate limiting engine.** Enforces token-per-minute limits defined in `MaaSSubscription`. Called by Envoy via the rate limit filter. If a key exceeds its quota, requests are rejected with 429. |
| `limitador-operator-*` | `rh-connectivity-link` | Manages the Limitador deployment. |
| `kuadrant-operator-*` | `rh-connectivity-link` | Watches `AuthPolicy` and `RateLimitPolicy` CRs. Configures Authorino and Limitador by translating those CRs into their respective configurations and wiring them into Envoy via `WASMPlugin`/`EnvoyFilter`. |

---

### MaaS & Model Serving (RHOAI)

| Pod | Namespace | Role |
|-----|-----------|------|
| `maas-api-*` | `redhat-ods-applications` | **MaaS REST API.** Exposes endpoints for managing API keys (`POST /maas-api/v1/api-keys`) and listing models (`GET /v1/models`). Stores keys in Postgres. When a key is created, it provisions a Kubernetes ServiceAccount in `maas-default-gateway-tier-free` and returns a short-lived JWT bound to that SA with audience `maas-default-gateway-sa`. |
| `maas-controller-*` | `redhat-ods-applications` | **MaaS Kubernetes controller.** Watches `MaaSModelRef`, `MaaSSubscription`, and `MaaSAuthPolicy` CRs. When they are created/updated, it reconciles: creates RBAC, creates Kuadrant `AuthPolicy` and `RateLimitPolicy` on the HTTPRoute, and sets CR status. Note: in RHOAI 3.3.0 the generated `AuthPolicy` calls `/internal/v1/api-keys/validate` which does not exist — see rbac.yaml note below. |
| `postgres-*` | `redhat-ods-applications` | **Database.** Stores API keys, subscription metadata, and usage data for the MaaS API. |
| `kserve-controller-manager-*` | `redhat-ods-applications` | **KServe controller.** Watches `LLMInferenceService` CRs. When one is created, it: provisions a storage-initializer init container to download the model, creates the Deployment and Service for vLLM, and creates the `HTTPRoute` that attaches the model endpoint to the MaaS gateway. |
| `odh-model-controller-*` | `redhat-ods-applications` | RHOAI-specific extension of KServe. Handles OpenShift-specific integrations (routes, monitoring, auth sidecars). |

---

### Model Workloads

| Pod | Namespace | Role |
|-----|-----------|------|
| `qwen-qwen3-06b-kserve-*` | `llm` | **vLLM inference server.** The model serving pod. Contains two containers: (1) `storage-initializer` init container — runs at pod startup, downloads model weights from `s3://models/qwen3-06b` to `/mnt/models`, then exits. (2) `main` container — runs vLLM, serving the model over HTTPS on port 8000 using TLS certs injected by KServe at `/var/run/kserve/tls/`. Receives requests forwarded by Envoy after auth passes. |

---

## Request Flow

```
Client
  │
  │  POST https://maas.apps-crc.testing/llm/qwen-qwen3-06b/v1/chat/completions
  │  Authorization: Bearer <JWT token>
  │
  ▼
router-default (HAProxy)                          [openshift-ingress]
  │
  │  TLS passthrough via OpenShift Route.
  │  Reads SNI hostname, forwards raw encrypted bytes
  │  to the Istio gateway pod without decrypting.
  │
  ▼
maas-default-gateway (Envoy/Istio)               [openshift-ingress]
  │
  │  Terminates TLS.
  │  Calls Authorino to enforce gateway-auth-policy:
  │
  │  Step 1 — TokenReview (via Authorino)
  │    Validates the JWT against Kubernetes API.
  │    Token must have audience: maas-default-gateway-sa.
  │    (Tokens from POST /maas-api/v1/api-keys satisfy this.)
  │
  │  Step 2 — SubjectAccessReview (via Authorino)
  │    Checks if the JWT's ServiceAccount can verb:post
  │    on llminferenceservices/qwen-qwen3-06b in namespace llm.
  │    Granted by maas/rbac.yaml (Role + RoleBinding).
  │
  │  If both pass → forwards to HTTPRoute.
  │  If either fails → returns 403.
  │
  ▼
HTTPRoute: qwen-qwen3-06b-kserve-route            [llm]
  │
  │  Created automatically by kserve-controller-manager.
  │  Maps /llm/qwen-qwen3-06b/* → vLLM service.
  │
  ▼
qwen-qwen3-06b-kserve pod (vLLM)                 [llm]
  │
  │  Serves the model over HTTPS on port 8000.
  │  TLS certs injected by KServe at /var/run/kserve/tls/.
  │  Model weights at /mnt/models (downloaded from S3
  │  by storage-initializer init container at pod startup).
  │
  ▼
Response returned to client
```

---

## How to Get an API Token

```bash
# 1. Log in with your OpenShift user credentials
oc login https://api.crc.testing:6443

# 2. Exchange your OpenShift token for a MaaS API key
TOKEN=$(oc whoami --show-token)
API_KEY=$(curl -sk -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"my-key"}' \
  https://maas.apps-crc.testing/maas-api/v1/api-keys \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")

# 3. Use the API key for inference
curl -sk -X POST \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"models/qwen3-06b","messages":[{"role":"user","content":"hello"}],"max_tokens":100}' \
  https://maas.apps-crc.testing/llm/qwen-qwen3-06b/v1/chat/completions
```

---

## Files

### kustomization.yaml
The root Kustomize entry point. Sets `namePrefix: qwen-` which is prepended
to all resource `metadata.name` fields. This means:
- `qwen3-06b` (LLMInferenceService)  →  `qwen-qwen3-06b`
- `qwen3-06b` (MaaSModelRef)         →  `qwen-qwen3-06b`
- `qwen3-06b-subscription`           →  `qwen-qwen3-06b-subscription`
- `maas-qwen3-06b-access` (Role)     →  `qwen-maas-qwen3-06b-access`

Includes two resource sets: `model.yaml` and the `maas/` directory.

---

### model.yaml
Defines the `LLMInferenceService` (KServe v1alpha1). When applied,
`kserve-controller-manager` reconciles this and creates:
- A `Deployment` running the vLLM pod
- A `Service` pointing to the pod
- An `HTTPRoute` attaching `/llm/qwen-qwen3-06b/*` to the MaaS gateway

Key fields:
- `spec.model.uri: s3://models/qwen3-06b` — where to download weights.
  The storage-initializer init container downloads this to `/mnt/models`
  before vLLM starts.
- `spec.model.name: models/qwen3-06b` — the model name vLLM registers.
  Used in inference requests as `"model": "models/qwen3-06b"`.
- `spec.router.gateway.refs` — attaches the generated HTTPRoute to
  `maas-default-gateway`, so traffic flows through the Istio gateway.
- `spec.template.containers[main]` — the vLLM container config. Uses
  `--enforce-eager` to disable CUDA graph compilation (required for Tesla T4).

---

### maas/maas-model.yaml
Defines a `MaaSModelRef`. Registers this model with the `maas-controller`
so it can be referenced by subscriptions and auth policies.

The controller sets its status to `Ready` once it confirms the
LLMInferenceService's HTTPRoute is attached to the MaaS gateway.
The `MaaSSubscription` looks up models through this resource.

- Namespace: `llm` (must match the LLMInferenceService namespace).
- `spec.modelRef.name` must match the final prefixed LLMInferenceService name.

---

### maas/maas-subscription.yaml
Defines a `MaaSSubscription`. Controls who can subscribe to this model
and sets token rate limits per API key.

The `maas-controller` reconciles this and creates a Kuadrant
`RateLimitPolicy` on the HTTPRoute, wired to `limitador` for enforcement.

- Namespace: `models-as-a-service` (the controller only watches this
  namespace — configured via `MAAS_SUBSCRIPTION_NAMESPACE` env var).
- `spec.owner.groups: [system:authenticated]` — any authenticated
  OpenShift user can request an API key for this model.
- `spec.modelRefs[0].tokenRateLimits` — 100 tokens per minute per key.

---

### maas/rbac.yaml
Defines a `Role` and `RoleBinding` in namespace `llm`.

Required because Authorino's SubjectAccessReview (Step 2 in the flow above)
checks Kubernetes RBAC before allowing the request through. Without this,
every inference request returns 403.

- The `Role` grants verb `post` on `llminferenceservices/qwen-qwen3-06b`.
- The `RoleBinding` grants it to ServiceAccount `kubeadmin-cffa0fee`
  in `maas-default-gateway-tier-free`. This SA is created by `maas-api`
  when an API key is issued to the `kubeadmin` user. Each OpenShift user
  gets their own SA in that namespace.

> **Note:** Normally this RBAC is created automatically by a `MaaSAuthPolicy`
> CR (reconciled by `maas-controller`). That CR is intentionally omitted here
> because the generated Kuadrant `AuthPolicy` calls
> `/internal/v1/api-keys/validate` on `maas-api`, which does not exist in
> RHOAI 3.3.0. The manual RBAC is a workaround until a newer RHOAI build
> is available.

---

## Namespace Summary

| Resource               | Namespace               | Why                                             |
|------------------------|-------------------------|-------------------------------------------------|
| LLMInferenceService    | `llm`                   | Model workload namespace                        |
| MaaSModelRef           | `llm`                   | Must be in same namespace as LLMInferenceService |
| MaaSSubscription       | `models-as-a-service`   | Controller only watches this namespace          |
| Role + RoleBinding     | `llm`                   | RBAC must be in the model's namespace           |
