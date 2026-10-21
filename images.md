# Images

Digests resolved with `crane digest` (skopeo is not available in the build environment). Digest = index/manifest-list digest of the tag. All are amd64 + arm64 indexes except the AWS DLC image (single amd64 manifest); `lws` also ships ppc64le/s390x.

| image | tag | digest | used by | source |
|---|---|---|---|---|
| docker.io/istio/pilot | 1.29.4 | sha256:b93740f69b0b67abb9e56abf8b28bf53e4f2a410ef93bab25ebed4b05a721bdd | istiod | `helm template istiod` (1.29.4) |
| docker.io/istio/proxyv2 | 1.29.4 | sha256:3c279352793d0c53e0a3561f0329db15dbed63a4dae5c46c391fae7220a31ddf | Istio-managed Gateway proxy (auto-deployed) + sidecar injection | `helm template istiod` (`global.hub`/`global.tag` + `proxyv2` name in injection template) |
| docker.io/vllm/vllm-openai | v0.31.0 | sha256:c1c9f6fd5c109ba7f0546a59f5b2f15fb87f64c77782e90a27b648b42a8e67c3 | vLLM prefill/decode (pd-disaggregation: cks-mooncake, coreweave, gke, gke/a4x, gke/a4xmax, gke/base); base image of the custom ModelExpress vLLM image; modelexpress-p2p fastsafetensors overlays | `kustomize build` pd-disaggregation/modelserver/gpu/vllm/*; `guides/recipes/modelserver/components/images/gpu-vllm/release` |
| ghcr.io/llm-d/llm-d-router-disagg-sidecar | v1.0.0 | sha256:83cdf20199485329baca89b69d78f695735855ed93412aeaeec325d2374ced04 | P/D routing sidecar in every pd-disaggregation vLLM overlay | `guides/recipes/modelserver/components/images/routing-sidecar/release` (the overlays as shipped render `:main`, see README Open questions) |
| ghcr.io/llm-d/llm-d-router-endpoint-picker | v1.0.0 | sha256:024188e998f516879f68532016ca24fabca00afbdcc2af9a61b17634e197df2c | EPP (router) | `helm template llm-d-router-gateway` with pd-disaggregation values; identical image with precise-prefix-cache-routing values and modelexpress-p2p values |
| nvcr.io/nvidia/ai-dynamo/modelexpress-server | 0.5.0 | sha256:a48e7085740a1cb594426a914f5ec85029c02e01e692a4b182457b82795644ae | ModelExpress server | `guides/modelexpress-p2p/modelexpress/modelexpress-server.yaml` |
| public.ecr.aws/deep-learning-containers/vllm | 0.31.0-gpu-py312-cu130-ubuntu24.04-ec2 | sha256:15883a4beb0b734a26cc1bb99296d47e7b2d86bc6330d5194c197ac2bf4a1197 | vLLM prefill/decode, AWS overlay only | `kustomize build` pd-disaggregation/modelserver/gpu/vllm/aws |
| registry.k8s.io/lws/lws | v1.0.0 | sha256:7aa4f6d26609d9f9650054227600337d240181f9d84cb2450474de3bd1e962a9 | LWS controller (incl. DisaggregatedSet) | `helm template lws --set enableDisaggregatedSet=true` |

## Not in images.txt

| image | reason |
|---|---|
| `<your-registry>/modelexpress-p2p-vllm:latest` | placeholder in the modelexpress-p2p overlays; built from `images/modelexpress-vllm/Dockerfile` with `BASE_IMAGE=docker.io/vllm/vllm-openai:v0.31.0` and `pip.txt`, then pushed to the private registry |
| `REPLACE_MODEL_SERVER_IMAGE` | unresolved placeholder in the pd-disaggregation `vllm/base` overlay; provider overlays resolve it to the vLLM images above |
| `busybox:1.28` | only in the istiod `grpc-simple` (proxyless gRPC) injection template, unused by this stack |

## Router render findings

- `precise-prefix-cache-routing` router values render only the EPP image: the `router.tokenizer` sidecar is off by default and tokenization runs on the vLLM pods (`/v1/*/render`). No tokenizer/render image is needed for vLLM.
- The Istio gateway proxy image is not a rendered workload; it is derived from istiod `global.hub`/`global.tag`/`proxyv2`.
