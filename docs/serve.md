# caldrix serve

`caldrix serve <model>` compiles a model for this machine and serves it over an OpenAI-compatible HTTP API
on `http://127.0.0.1:8425`. It takes the same model argument as `caldrix run`, and every serving flag of
Inferno's server; the endpoints are in [api.md](api.md).

## Usage

```text
caldrix serve <model> [--port 8425] [--host 127.0.0.1] [--ctx N] [--backend auto|hip|metal|cpu] [serving flags]
```

The model comes first: a directory, an alias or a Hugging Face repo id `owner/name[@revision]`
([run.md](run.md#the-model-argument), [models.md](models.md)). `--backend` picks the lane as in
`caldrix run` ([doctor.md](doctor.md)), and `--ctx N` is `--max-context N`.

## An exchange

```sh
caldrix serve qwen3-0.6b --backend cpu
```

Once the server is up, stderr ends with the line `caldrix: serving Qwen/Qwen3-0.6B on http://127.0.0.1:8425`.
From another terminal:

```sh
curl http://localhost:8425/v1/models
```

```text
{"data":[{"id":"Qwen/Qwen3-0.6B","object":"model","owned_by":"inferno"}],"object":"list"}
```

```sh
curl http://localhost:8425/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3-0.6B", "messages": [{"role": "user", "content": "What is the capital of France?"}], "max_tokens": 16}'
```

```text
{"choices":[{"finish_reason":"length","index":0,"message":{"content":"<think>\nOkay, the user is asking for the capital of France. I know","role":"assistant"}}],"created":1792079067,"id":"chatcmpl-1","model":"Qwen/Qwen3-0.6B","object":"chat.completion","usage":{"completion_tokens":16,"prompt_tokens":15,"total_tokens":31}}
```

Both were captured on a Linux x86-64 machine on the CPU lane. The model name in a request must be the
name the server answers to: the repo id for an alias or a repo id, the directory's name for a directory,
or `--served-name`. Ctrl-C stops the server.

## Defaults and your flags

`caldrix serve` starts Inferno's server with its own defaults first and your flags after them:

```text
--model-dir <the model's directory> --backend <the lane> --served-name <the repo id or directory name> --max-context 2048 --port 8425
```

Where a flag is given twice, the last one wins, so any of your flags overrides the default. `--model-dir`
is the one you cannot give: the model is the first argument.

## Flags

Every flag Inferno's server parses, with the default `caldrix serve` runs it at. Each is checked before any
download starts.

| Flag | Meaning | Default |
| --- | --- | --- |
| `--port N` | the HTTP port; `0` takes any free port | `8425` |
| `--host ADDR` | the address to listen on | `127.0.0.1` |
| `--served-name NAME` | the model name the API answers to | the repo id, or the directory's name |
| `--max-context N` (or `--ctx N`) | the compiled context length, in tokens | `2048` |
| `--dtype bf16\|f32` | the type the model is compiled in | `bf16`; `f32` for a quantized model, which refuses `bf16` |
| `--queue-depth N` | requests that may wait for the engine; past it a request gets `429` | `8` |
| `--max-batch 1\|8` | requests decoded together; `8` only on the AMD lane | `8` on the AMD lane; `1` on `metal` and `cpu`, with `--weight-budget`, with `--admin` and for a quantized model, which refuses `8` |
| `--admission fill\|one` | how prompts are admitted between decode steps: `fill` admits into every free slot, `one` at most one per loop iteration | `fill` |
| `--prefill-grid off\|on` | AMD lane only: run each prompt's prefill as one persistent train beside the decode step; `on` needs `--prefill-attention tiles`, `--prefill-gemm inferno`, `--max-batch 8`, bf16, resident weights and a dense Llama model, and is refused otherwise, naming the other flag | `off` |
| `--threads N` | `cpu` only: the size of the thread pool | the physical cores, capped by a cgroup CPU quota |
| `--cpu-isa ROW` | `cpu` only: the instruction-set row to compile for (`x86-64-v2`, `x86-64-v3`, `x86-64-v4`), at or below the probed one | the probed row |
| `--weight-budget BYTES` | stream the weights through this much memory instead of holding them all (`5g`, `4500m` or a byte count); refused on the AMD lane, and on the `nvidia` lane for any model but a mixture of experts | every weight resident |
| `--expert-host-budget BYTES` | with `--weight-budget` on `--backend nvidia`: host memory that keeps the expert weights already read from disk, so an expert the card no longer holds is not read again (`18g`, a byte count); it must hold at least one token's experts, else refused | none: an expert the card does not hold is read from disk |
| `--weight-ring N` | with `--weight-budget`: slots in the streaming ring, at least `2` | `2` |
| `--weight-pin N` | with `--weight-budget`: at most this many layers held resident | no cap |
| `--weight-pins spread\|prefix` | with `--weight-budget`: which layers are held resident, spread over the stack or the first ones | `spread` |
| `--weight-digest every-fill\|at-open` | with `--weight-budget`: when a streamed segment's checksum is checked | `every-fill` |
| `--weight-codec raw\|planes` | with `--weight-budget`: how the streamed layout file stores the weights | `planes` |
| `--layout-dir DIR` | with `--weight-budget`: where the streamed layout file is kept | `layout` under Inferno's cache: `$INFERNO_CACHE_DIR`, else `$XDG_CACHE_HOME/inferno`, else `~/.cache/inferno` |
| `--telemetry off\|host\|sampled` | how much of each request's timing is recorded for `/metrics` and `/debug/requests` | `sampled` |
| `--device-sample-period N` | with `sampled`: device timing on one step in `N` | `8` |
| `--device-sample-offset N` | with `sampled`: which step of each period, below the period | `4` |
| `--telemetry-log FILE` | append each request's telemetry record to `FILE` | none |
| `--batch-gemv rows\|x2\|x4` | AMD lane only: the batched GEMV kernel's spelling; `x4` is refused when the plan compiles | `rows` |
| `--tail generated` | AMD lane only: the output-layer kernel's spelling | `generated` |
| `--prefill-attention rows\|tiles` | AMD lane only: the prefill attention kernel's spelling; `tiles` needs `--max-batch 8`, a bf16 Llama model and, with `--prefill-chunk N`, `N` a multiple of 32, and is refused otherwise | `rows` |
| `--batch-attention rows\|grouped` | AMD lane only: the batched attention kernels' spelling | `rows` |
| `--sampler device\|host` | AMD lane only, with `--max-batch 8`: where a sampled token is picked, on the GPU or on the CPU ([api.md](api.md#sampling)) | `device` |
| `--denoiser-weights FILE` | an image pipeline only: read the denoiser from this GGUF file instead of `transformer/`; it runs in f32 ([An image pipeline](#an-image-pipeline)) | the pipeline's `transformer/` |
| `--draft-model DIR\|ALIAS\|REPO` | speculate with this draft model on a lane that can check a draft (`speculation=yes` in [doctor.md](doctor.md)); a draft that fails its checks falls back to n-gram lookup, with a stderr line ([run.md](run.md#speculative-decoding)) | none |
| `--admin` | open the variant-swap admin endpoint, on loopback only; needs `--admin-port` | off |
| `--admin-port N` | the admin endpoint's port, different from `--port` | none |
| `--shadow-ring N` | with `--admin`: recent requests kept to check a candidate variant against | `8` |
| `--shadow-requests N` | with `--admin`: requests a candidate must match before it serves | `4` |
| `--shadow-timeout SECONDS` | with `--admin`: how long a candidate has to match | `600` |
| `--temperature T`, `--top-k N`, `--top-p P`, `--min-p M`, `--repetition-penalty R`, `--seed N` | the server's default sampling settings, the same knobs as [run.md](run.md#sampling); a request's own fields override them ([api.md](api.md#sampling)) | greedy (`0`, `0`, `1`, `0`, `1`, fresh entropy) |

The kernel flags (`--batch-gemv`, `--tail`, `--prefill-attention`, `--batch-attention`, `--sampler`) are
refused on the `metal` and `cpu` lanes. The `--admin` flags are refused on the `cpu` lane and with `--weight-budget`.
The sampling flags set the server's defaults for every request; a request's own `temperature`, `top_p`,
`top_k`, `min_p`, `repetition_penalty` and `seed` override them per call ([api.md](api.md#sampling)).

## A vision model

`caldrix serve smolvlm-256m --backend cpu` serves SmolVLM-256M, which takes image parts in a chat message
([api.md](api.md)); at startup stderr carries the line `vision: siglip 12×768, 64 tokens/tile, tower f32`. It runs on the
`cpu`, `hip` and `metal` lanes, one request at a time, from its safetensors weights held resident; on `hip` a startup line
says the default `--max-batch` is lowered from 8 to 1. On the `hip` lane `inferno serve` served it on one MI300X;
`caldrix serve` itself has not been exercised there. On the `metal` lane `inferno serve` served it on one Apple M1 Max;
`caldrix serve` itself has not been exercised there ([models.md](models.md#aliases)).

- `--dtype bf16` (the default) runs the text model in bf16; `--dtype f32` runs it in f32. The vision tower
  runs in f32 at either, and its output enters the text model in the text model's type.
- Refused by name at startup, each naming the vision tower: `--max-batch` above `1`, `--weight-budget`, a
  kernel flag (`--batch-gemv`, `--tail`, `--prefill-attention`, `--batch-attention`, `--sampler`) at other than
  its default, `--admin`, `--speculate draft` and `--speculate ngram`, and a GGUF file
  or quantized weights.
- It decodes without speculation, and `caldrix` adds no speculation flag or draft for it. If `--speculate auto`
  is passed, a startup line says speculation is off and why; `--speculate draft` and `--speculate ngram` are refused.
- Its request body limit is 32 MiB, on every route, and at most 2 requests decode images at once
  ([api.md](api.md)).

## An image pipeline

`caldrix serve` on a directory holding `model_index.json` (a diffusers FLUX.2 klein 4B pipeline) serves
[`POST /v1/images/generations`](api.md#post-v1imagesgenerations) instead of the text endpoints. The
pipeline's recorded lanes are `cpu` and `metal`, and `--backend auto` takes `cpu` on Linux and `metal` on a
Mac where the Metal lane is usable ([doctor.md](doctor.md)). The
Metal lane's record is on one Apple M1 Max ([run.md](run.md#generating-an-image)): there this endpoint
answered one request at 256×256, once, and its image was byte for byte the one `caldrix run` wrote. It has
not been run there with more than one request or with requests at once, nor on another Apple chip or on a
Mac with less memory than that one. `caldrix serve flux2-klein-4b` downloads the
alias as `caldrix run` does ([run.md](run.md#generating-an-image)), its memory check counting the text encoder
kept resident beside the denoiser, then serves the download as a pipeline directory; once downloaded it is
served from the cache with no request. `caldrix serve owner/name` does the same for a Hugging Face repo
whose file list holds `model_index.json` at its top, at the commit that list named. `caldrix` tells a
pipeline from a text model by the first file list it reads for a text model anyway, so serving a text
model makes no extra request. A pipeline directory
takes the same memory check before any weight loads; the check is a lower bound ("needs at least"), and a
pipeline that cannot fit the lane is refused by name. With `--denoiser-weights` a directory's check is
skipped and one line says so, because the check counts the `transformer/` files that flag replaces. The alias and repo-id paths are tested against a local stand-in for the
Hub only; no real download of the alias has run.

- The text encoder loads on the first request and stays resident beside the denoiser, so later requests
  reload nothing; one request is generated at a time. The startup line names the lane, the dtype and the
  weights on disk.
- `--dtype bf16` (the default) or `f32` sets the text encoder's and the denoiser's type; the autoencoder
  always runs in f32.
- `--denoiser-weights FILE` reads the denoiser from a GGUF file in Black Forest Labs' layout (unsloth's
  files) instead of `transformer/`. Its rows are widened to f32 when they load, so it runs in f32: it is
  the default for it, and an explicit `--dtype bf16` is refused. A Q8_0 file's header, byte order and
  widened rows are checked against the safetensors at the desk; no image has been made from a GGUF
  denoiser of the real checkpoint. A file holding a tensor type inferno does not read (a Q4_K_M file's
  Q5_K tensors) is refused by name.
- Refused by name at startup: a `--backend` other than `cpu` or, on a Mac build, `metal`, `--threads` or
  `--cpu-isa` with `--backend metal`, `--max-batch` above `1`, a draft model, `--speculate draft` and
  `--speculate ngram`, `--weight-budget`, `--admin`, and the sampling flags, whatever their value
  (`--temperature 0` included; a request's `seed` seeds its image). A tokenizer that can produce ids
  past the text encoder's vocabulary is refused before anything loads.
- `INFERNO_HIDDEN_STATE_LAYERS=a,b,c` names the text encoder's hidden-state layers for a reduced test
  pipeline whose encoder is shallower than klein's: three integers, strictly ascending, each at least 1
  and below the encoder's layer count, else refused by name. Unset, klein's layers are used.

## Binding and proxies

The server listens on `127.0.0.1`, so only this machine can reach it. `--host 0.0.0.0` (or any address
that is not loopback) exposes it to that network, and the server prints a warning: there is no
authentication and no TLS.

To serve other machines, keep the default and put a reverse proxy in front that terminates TLS and
authenticates clients, forwarding to `http://127.0.0.1:8425`. Answers stream as server-sent events, so turn
off response buffering for `/v1/` in the proxy. A request body over 4 MiB is refused with `413` by the
server itself (32 MiB for a vision model).
