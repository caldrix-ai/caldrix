# Kubernetes

The `caldrix` Helm chart runs one `caldrix serve` of one model on Kubernetes, from the
[CPU container image](install.md#the-cpu-container-image). The server it runs is the same OpenAI-compatible
one as on your machine, so the [HTTP API](api.md) and the [`caldrix serve` flags](serve.md) apply unchanged.
The chart's source is [caldrix-ai/helm-charts](https://github.com/caldrix-ai/helm-charts); its package is
`oci://ghcr.io/caldrix-ai/charts/caldrix`.

## Install

With Helm 3.8 or newer:

```sh
helm install caldrix oci://ghcr.io/caldrix-ai/charts/caldrix --version 0.1.0 --namespace caldrix --create-namespace
```

The pod turns Ready only once the model is downloaded, compiled and loaded, because the server binds its port
after the load. The chart's startup probe allows for that. Then `helm test` checks `/healthz` and `/v1/models`
from inside the cluster:

```sh
helm test caldrix --namespace caldrix
```

From another terminal, forward the Service and read the served model; the `id` it returns is the `model` to send:

```sh
kubectl --namespace caldrix port-forward svc/caldrix 8425:80
curl http://localhost:8425/v1/models
curl http://localhost:8425/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "<id>", "messages": [{"role": "user", "content": "What is the capital of France?"}]}'
```

## What it installs

With the defaults, `helm install` creates:

- one Deployment with one replica, whose container runs `serve smollm2-instruct --host 0.0.0.0 --port 8425 --backend cpu`
  as the image's non-root user;
- one PersistentVolumeClaim, mounted at `/data/huggingface`, the model cache, so a restart does not download the
  model again;
- one ClusterIP Service on port 80 to the container's port 8425;
- one ServiceAccount with no RBAC: the server talks to no Kubernetes API;
- the `helm test` pod.

## Values worth knowing

| key | default | meaning |
|---|---|---|
| `model.name` | `smollm2-instruct` | the model: an alias, a hub id `owner/name[@revision]`, or `ollama:<tag>`, as in [Models](models.md) |
| `model.context` | `""` | the context length, when set |
| `model.extraArgs` | `[]` | further `caldrix serve` arguments, appended verbatim |
| `persistence.size` | `20Gi` | the model cache's PersistentVolumeClaim |
| `huggingface.existingSecret` | `""` | an existing Secret holding a Hugging Face token, for gated models |
| `resources` | requests `cpu: 2`, `memory: 4Gi`; no limits | a CPU limit sets the server's thread count |
| `ingress.enabled` | `false` | an Ingress to the Service |
| `networkPolicy.enabled` | `false` | a NetworkPolicy admitting traffic to the server's port only |

Every value, with the GPU rule, storage and exposure, is in the
[chart's README](https://github.com/caldrix-ai/helm-charts/blob/main/charts/caldrix/README.md).

## Exposure

The server has no authentication. The Service is ClusterIP; put a gateway that authenticates in front of it before
you expose it, through `ingress.enabled` or otherwise. `networkPolicy.enabled: true` admits traffic to the server's
port only, and `networkPolicy.ingressFrom` narrows it to the peers you list, your gateway's among them.

## GPUs

The chart carries the values a GPU deployment needs (`gpu.enabled`, `model.backend: hip`) and refuses them while the
only published image is the CPU one: `helm install` fails with one sentence saying the ROCm image is not published
yet. They render only with `image.gpu: true`, your own statement that the image you named carries a GPU lane.

## What the chart does not do

No auth, no rate limiting, no TLS of its own, no autoscaling: the server is single-process and stateful per model. No
PodDisruptionBudget, no CRDs, no operator, and no model pre-pull job.

## Verified

Verified: installed with kind in a virtual machine, at the desk and on the release runner, on the CPU image
`ghcr.io/caldrix-ai/caldrix:0.2.0-cpu` with the default model; each time `helm test` passed and one chat request
answered. Not yet run on a cluster of any other kind, and not on any GPU.
