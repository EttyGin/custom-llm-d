# custom-llm-d

Transfer manifest for llm-d v1.0.0 in an air-gapped Kubernetes cluster: Gateway mode (Istio), P/D disaggregation (vLLM, NVIDIA GPU, LWS DisaggregatedSet), approximate + precise prefix-cache routing, ModelExpress P2P weight loading (kubernetes backend).

## Layout

| path | content |
|---|---|
| `charts/` | Helm charts (.tgz): llm-d-router-gateway, lws, istio base, istiod |
| `crds/gateway-api/` | Gateway API, standard channel |
| `crds/gaie/` | Gateway API Inference Extension |
| `crds/lws/` | LeaderWorkerSet, DisaggregatedSet, DisaggregatedSetRoleScaler (extracted from the LWS chart) |
| `crds/modelexpress/` | ModelExpress CRDs |
| `images/modelexpress-vllm/` | Dockerfile of the custom vLLM image |
| `images.txt`, `images.md` | pinned images; table with sources |
| `repos.md` | git repos to mirror |
| `pip.txt` | python packages for the custom image, hash-pinned to the cp312 manylinux wheels (x86_64 + aarch64) |

Istio CRDs ship inside the `base` chart and are not duplicated under `crds/`. They also cover the `DestinationRule` the router chart renders for `provider.name=istio`.

| chart | source | sha256 (.tgz) |
|---|---|---|
| `llm-d-router-gateway-v1.0.0.tgz` | `oci://ghcr.io/llm-d/charts/llm-d-router-gateway` (OCI digest `sha256:0a16af45be7457acea1d5df2a3e2b7da7583ad65efcb30a65bbd6f2fd8f6ea4e`) | `ced5cbcf3ba336ff57c14e66a55c77ace1bd08525e2ce29c45026cb1a1254cc7` |
| `lws-1.0.0.tgz` | `oci://registry.k8s.io/lws/charts/lws` (OCI digest `sha256:9444cfc199ab2aa775e683d9de64bcd903f825561d4bb0d99bb043f84423f2c8`) | `ea6cff664078688aeed8727afcb2e9cef481f3651919a3b49c0f6dfc50c9627b` |
| `base-1.29.4.tgz` | https://istio-release.storage.googleapis.com/charts | `51a5470204f68a79eb44a70a8afc6d87193fe3418ba22e69c7abc05cbe489660` |
| `istiod-1.29.4.tgz` | https://istio-release.storage.googleapis.com/charts | `b0c5bac432e4475d44fd42587227182b367cf48e98b7bbd96260bb40f54679e3` |

**Image platforms.** Every image in `images.txt` is a multi-arch index (amd64 + arm64) except the AWS DLC vLLM image, which is amd64 only. The `gke/a4x` and `gke/a4xmax` overlays (GB200/GB300) pin `kubernetes.io/arch` to arm64, so mirror all platforms of each index, not just amd64. The digests in `images.txt` are index digests.

## Versions

| component | version | source |
|---|---|---|
| llm-d | v1.0.0 (378b42d) | tag |
| llm-d-router-gateway chart | v1.0.0 | `ROUTER_CHART_VERSION` in env.sh |
| EPP image | v1.0.0 | `ROUTER_EPP_VERSION` in env.sh, chart render |
| Gateway API CRDs | v1.6.2, standard channel | `GATEWAY_API_VERSION`; `install-crds.md` uses `standard-install.yaml` |
| GAIE CRDs | v1.6.2 | `GAIE_VERSION` |
| LWS | 1.0.0, `enableDisaggregatedSet=true` | `LWS_VERSION` |
| Istio base / istiod | 1.29.4 | v1.0.0 release notes; istiod needs `ENABLE_GATEWAY_API_INFERENCE_EXTENSION=true` (istio.md) |
| ModelExpress server + client | 0.5.0 (tag v0.5.0) | `guides/modelexpress-p2p` (`MX_VERSION`, Dockerfile, server manifest) |
| vLLM | v0.31.0 | `images/gpu-vllm/release` component |
| google-crc32c | 1.8.0 | `images/modelexpress-vllm/Dockerfile` |

## Install order

1. CRDs: Gateway API, GAIE, LWS, ModelExpress (Istio CRDs come with `base`)
2. LWS, with DisaggregatedSet enabled
3. Istio: `base`, then `istiod` with the inference-extension flag on
4. GAIE (CRDs only, nothing further to install)
5. Router: `llm-d-router-gateway`, plus the Gateway resource from `guides/recipes/gateway/istio` in the llm-d repo
6. ModelExpress server
7. Model servers: pd-disaggregation vLLM overlays; modelexpress-p2p overlays use the custom image

## Open questions

- **ModelExpress version.** The v1.0.0 guide pins v0.5.0 (`MX_VERSION`, server image, client). The llm-d 1.0 release notes list v0.7.0 (tag exists, ac6d03a87a550c8188c3ed59f187b90d4f0554f5). v0.5.0 is used here; compatibility of v0.7.0 with the guide is not verified.
- **Istio version.** `docs/infrastructure/gateway/istio.md` installs 1.29.2; the release notes list 1.29.4. 1.29.4 is used here.
- **Sidecar tag.** The pd-disaggregation vLLM overlays use the `routing-sidecar/nightly` component, which renders `ghcr.io/llm-d/llm-d-router-disagg-sidecar:main` (digest at resolve time `sha256:d8d18cbd407a61ce5a7638201dfe9d2078ca37ecc8e57ea3b2a2681a509f60db`). `images.txt` carries `v1.0.0` (from the `release` component) instead, so the overlays' sidecar image must be replaced.
- **vLLM version in release notes.** The model-server table lists vllm-openai `v0.30.0`; the overlays render `v0.31.0` (used here).
- **Custom image placeholders.** The modelexpress-p2p overlays reference `<your-registry>/modelexpress-p2p-vllm:latest`; registry path and tag are yours to choose.
- **Custom image architecture.** If the ModelExpress image must run on arm64 nodes, build it for both platforms. `pip.txt` hashes cover both (Python 3.12, read from the base image build args `PYTHON_VERSION=3.12`).
- **llm-d-pd-utils** is in `repos.md` per the release notes, but no rendered manifest references its image (`ghcr.io/llm-d/llm-d-pd-utils:v1.0.0` exists), so it is not in `images.txt`.
- **Image verification** used `crane manifest <image>@<digest>`; skopeo was not available.
