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
| `--threads N` | `cpu` only: the size of the thread pool | the physical cores, capped by a cgroup CPU quota |
| `--cpu-isa ROW` | `cpu` only: the instruction-set row to compile for (`x86-64-v2`, `x86-64-v3`, `x86-64-v4`), at or below the probed one | the probed row |
| `--weight-budget BYTES` | stream the weights through this much memory instead of holding them all (`5g`, `4500m` or a byte count); refused on the AMD lane | every weight resident |
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

## Binding and proxies

The server listens on `127.0.0.1`, so only this machine can reach it. `--host 0.0.0.0` (or any address
that is not loopback) exposes it to that network, and the server prints a warning: there is no
authentication and no TLS.

To serve other machines, keep the default and put a reverse proxy in front that terminates TLS and
authenticates clients, forwarding to `http://127.0.0.1:8425`. Answers stream as server-sent events, so turn
off response buffering for `/v1/` in the proxy. A request body over 4 MiB is refused with `413` by the
server itself.
