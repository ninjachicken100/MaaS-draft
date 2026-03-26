# vLLM Runtime

| **Metadata**       | **Value**             |
|--------------------|-----------------------|
| **Status**         | Draft ⚠️              |
| **Authors**        | Ananda |
| **Created**        | 04-12-2025            |
| **Last Updated**   | 25-02-2026            |
| **Decision Date**  | NIL         |
| **Approvers**      | Ning Jie       |

## Change log

| Version | Date       | Changes                          | Author       |
|---------|------------|----------------------------------|--------------|
| v1.0    | 25-02-2026   | First Complete draft                    | Ananda         |

## Motivation

The initial LLM model serving via vLLM was through a custom Helm chart which came with a growing maintenance burden; Resource configuration, Scaling behaviour, and Model loading arguments. KServe's LLMInferenceService CRD addresses this by providing  Kubernetes-native abstraction for generative AI workloads, allowing us to express the full model serving topology in a single resource using vLLM as the underlying inference engine.

This RFC aims to evaluate against previously deployed standalone vLLM, and to propose valid deployment modalities.

### Scope

This evaluates the choice of vLLM runner, tradeoffs & implications, and standardized deployment modalities. The following API implementations have been tested:

| API | Endpoint | Spec |
|-----|----------|------|
| Chat completions | `POST /v1/chat/completions` | OpenAI-compatible |
| Embeddings | `POST /v1/embeddings` | OpenAI-compatible |
| Rerank | `POST /v1/rerank` | OpenAI-compatible |
| Audio transcriptions | `POST /v1/audio/transcriptions` | OpenAI-compatible |

Out of scope:
- Predictive AI models
- Comparison of AI Gateway and Routers
- Disaggregated Serving of LLMs
- Support for other GenAI protocols like HuggingFace TGI protocol

### Problem Context 

#### vLLM in early stages

During the introduction of vLLM, KServe did not InferenceService deployments that helped platform developers to 

1. **API Protocol limitations** - in terms of common Gen AI protocols, Triton + vLLM Backend only supported `/v1/chat/completions`. Embeddings and transcription endpoints were not supported, and the far less popular `v2/models/<model_name>[/versions/<model_version>]/infer` had to be used instead, severely impacting application portability - custom backend logic had to be implemented with the problematic `model.py`, along with client-side request changes.

    | **API Endpoint** | **Protocol** | **Triton + vLLM Backend** | **Pure vLLM** |
    |------------------|--------------|---------------------------|---------------|
    | `POST /v1/chat/completions` | OpenAI | ✅ | ✅ |
    | `POST /v1/embeddings` | OpenAI | ❌ | ✅ |
    | `POST /v1/rerank` | OpenAI | ❌ | ✅ |
    | `POST /v1/audio/transcriptions` | OpenAI | ❌ | ✅ |
    | `POST /v2/models/{model_name}/infer` | Open Inference Protocol | ✅ | ❌ |
    | `POST /v2/models/{model_name}/generate` | Open Inference Protocol | ✅ | ❌ |

2. **Complexity in Serving** - Triton's unique requirements in the model package file structure to include folder versioning and `config.pbtxt`, impacting the portability of Gen AI models pulled from HuggingFace. Such complexities require some degree of AI Engineering expertise and increases the lead time for models to be deployed.

    ```
    model_repository/
    └── vllm_model
        ├── 1
        │   └── model.json
        └── config.pbtxt
    ```

3. **Issues with Shared Tokenizer** - even though multiple models can be served on the same instance, a common tokenizer had to be used, resulting in shared servers being rarely ideal.

    ```
    python3 openai_frontend/main.py --model-repository path/to/models --tokenizer meta-llama/Meta-Llama-3.1-8B-Instruct
    ```

Given the problems faced in using Nvidia Triton with vLLM Backend, a new standard runtime and modality is proposed.

## Design

### Standalone vLLM Standardization

Given the urgency for a solution, coupled with the lack of immediate need for disaggregated serving and AI Gateway/Router, this RFC proposes the design for deploying a working and production-ready vLLM runtime only.

See the [starforge-vllm Helm chart](../../helm/starforge-vllm/) for the implementation.

The use of an Inference Platform such as KServe with the newly introduced LLMInferenceService CRD is being evaluated and will be discussed in a separate RFC.

#### Container Diagram (S3 Download Scenario)

![Container Diagram S3](artifacts/image-s3.png)

> Structurizr DSL source: [artifacts/container-diagram.dsl](artifacts/container-diagram.dsl)

#### Use of Job and initContainers

**Job** — downloads model weights to the shared PV once, as a Helm pre-install/pre-upgrade hook.

```sh
aws s3 sync s3://${S3_BUCKET}/${S3_MODEL_KEY} /models/${MODEL_NAME} --no-progress
```

**initContainer** — validates that weights are present before the vLLM container starts. Does not download.

```

```

Model weight downloading is split between a Job and an initContainer to allow safe horizontal scaling.

If a single initContainer handled both checking and downloading, scaling to multiple replicas would cause all pods to simultaneously write to the same PV path — creating a race condition that risks file corruption. Delegating the download to a Job (which runs once before the Deployment) and restricting the initContainer to a read-only check eliminates this.

#### Container Diagram (Hugging Face Scenario)

![Container Diagram HF](artifacts/image-hf.png)

> Structurizr DSL source: [artifacts/container-diagram-hf.dsl](artifacts/container-diagram-hf.dsl)

The implementation also supports direct loading of models from HuggingFace for internet connected testing.

### Choice of Image

Red Hat AI Inference Server from `registry.redhat.io/rhaiis/vllm-cuda-rhel9` was chosen to be the standardized image of choice for the following reasons:
- Fully supports vLLM APIs
- UBI base image
- Uniquely has audio libraries baked-in
- Smaller image size
- Good CVE posture

**CVE Management Comparison:**

At the time of scan (04/Dec/2025), RHAIIS was scanned with a 14-day old image whereas the OSS vLLM was scanned with a 20-hour old image.

| **CVE Source** | **RHAIIS** | **Docker-vLLM** |
| -------------- | ---------- | ------------ |
| **Base Image** | Better due to RHEL base image (1 HIGH) | Ubuntu base image (5 HIGH) |
| **vLLM & Ray** | Worse-off due to `n-1` lag in vLLM library (~2 week lag, within CVE patch window) (1 HIGH, 1 CRIT) | Latest patch (0 HIGH, 0 CRIT) |

The trade-off is deemed worth-it as the lag in vLLM library patches will be synced in the next 1-3 weeks. 

**Alternatives Considered:**

| **Image** | **Repo** | **Downsides** |
| -------------- | ---------- | ------------ |
| Docker vLLM Pre-built | docker.io/vllm/vllm-openai | Ubuntu base comes with CVEs and slight increase in image size |
| Red Hat OpenShift AI Image | registry.redhat.io/rhoai/odh-vllm-cuda-rhel9 | Same as proposed RHAIIS image, but adds the unneeded TGIS adapter |
| Dynamo vLLM Runtime | nvcr.io/nvidia/ai-dynamo/vllm-runtime | Ubuntu base comes with CVEs and slight increase in image size; Designed for use with Nvidia Dynamo |

### Key Improvements

With the support of vLLM standalone, comes the following significant improvements for users:
- Apps are portable from other environments which support OpenAI API protocols
- Models can be downloaded directly from HuggingFace without needing to modify files and add configs
- Lead time to serve models are reduced from ~days to ~minutes
- DevOps team members can easily support model deployments without AI Engineering expertise

### Implications

The largest implication moving away from Nvidia Triton for Gen AI is the need to have GPU sharing.

Since each RHAIIS instance (and other pure vLLM alternatives) can only serve 1 model per instance, it becomes extremely wasteful to allocate a 1 or more full GPU to an instance, especially if it is serving small models such as for embeddings/encoders.

This exacerbated the need for GPU sharing to be implemented in production before this vllm-service can be deployed.

### Deprecation of Triton for Gen AI

Triton with vLLM backend will be sunset for serving Gen AI models, as the dependent apps migrate away from it to the new vLLM service.
