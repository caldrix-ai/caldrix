# Changelog

Every caldrix release, newest first. Each release's notes on GitHub are its section here.

## [0.2.0] - 2026-10-27

### Added

- `caldrix pool` and `caldrix pool --protocol-version`: a daemon on a Unix socket that holds loaded models
  and serves them to local clients over a framed protocol ([caldrix pool](https://caldrix.ai/docs/pool)).
  It has run on the CPU lane, and on Metal on one Apple M1 Max at max context 2048 and one request at
  8192; it has not been run on AMD GPUs.
- `caldrix models`: search, inspect and list models from the downloaded model index
  ([caldrix models](https://caldrix.ai/docs/models#caldrix-models)).
- `ollama:<name>[:<tag>]` model references, downloaded from the Ollama registry
  ([Ollama tags](https://caldrix.ai/docs/models#ollama-tags)).
- Ollama Q4_K_M and Q8_0 tags, read and sized before any download and run at an f32 plan; an explicit
  `--dtype bf16` and `--weight-budget` are refused for them by name
  ([Ollama tags](https://caldrix.ai/docs/models#ollama-tags)). On Metal, on one Apple M1 Max, three small
  tags and an 8 B Q4_K_M tag were token-exact in f32 and `caldrix run` of the 8 B tag completed. On the CPU
  lane at `x86-64-v3`, four small tags were token-exact in f32 at the desk on one machine, a desk reading,
  not a run. On AMD GPUs the `hip` lane compiles them and holds a quantized model to one request at a
  time (`--max-batch` lowers from 8 to 1 with a startup line, and an explicit `--max-batch 8` is refused
  by name); nothing quantized has run on an AMD device.
- Sampling for `caldrix run`: `--temperature`, `--top-k`, `--top-p`, `--min-p`, `--repetition-penalty` and
  `--seed`, and the chat's commands for them ([Sampling](https://caldrix.ai/docs/run#sampling)).
- Sampling for `caldrix serve`: the same six flags set the server's defaults, and `/v1` requests accept
  `temperature`, `top_p`, `top_k`, `min_p`, `repetition_penalty` and `seed` per request, where v0.1.2
  refused a non-neutral `temperature`, `top_p`, `top_k`, `min_p` or `repetition_penalty` with a `400`
  naming the field and accepted `seed` while decoding greedily
  ([Sampling](https://caldrix.ai/docs/api#sampling)). On the AMD lane with batching on (`--max-batch 8`,
  its default there) the GPU picks each sampled token, so the same seed can give a different reply than a
  pick on the CPU; `--sampler host` moves that pick back to the CPU. Every other lane, and
  `--max-batch 1`, always picks on the CPU.
- `--system TEXT|@FILE` to set a system prompt, `--raw` to send the prompt as plain text, and `--json` for
  `caldrix run`'s output as JSON.
- The `qwen3-30b-a3b` alias and the `Qwen3MoeForCausalLM` architecture, verified by its token gates on
  one AMD MI300X with every expert resident, on one Apple M1 Max streamed under a 56 GiB
  `--weight-budget`, and on the CPU lane streamed under a 20 GiB budget; `--max-batch` above 1 is refused
  for it ([Aliases](https://caldrix.ai/docs/models#aliases)).
- `CALDRIX_OFFLINE`, like `HF_HUB_OFFLINE` but covering every network edge caldrix has: the Hub, the model
  index and the Ollama registry ([Environment](https://caldrix.ai/docs/models#environment)).
- `MistralForCausalLM` is accepted by the import; it is not verified on hardware.
- Llama 3 rope scaling, applied from a `config.json` and from a GGUF's `rope_freqs.weight`.

### Changed

- Generation is no longer greedy-only: it is greedy unless a sampling flag asks otherwise.
- The Metal prefill and decode attention are tiled over the context. On one Apple M1 Max a 6002-token
  prompt ran at max context 7680, and max context 8192 loaded and ran.
- `caldrix run` opens the chat screen on a terminal; `--plain` keeps the line-by-line chat.

### Known limitations

- Quantized tags have not been run on AMD GPUs; on the CPU lane they have only a desk reading, not a run;
  quantized safetensors checkpoints are refused.
- `caldrix pool` has not been run on AMD GPUs.
- NVIDIA GPUs are not supported.

## [0.1.2] - 2026-10-17

### Added

- The `smollm2-instruct` alias (`HuggingFaceTB/SmolLM2-135M-Instruct`), a SmolLM2 with a chat template
  for `caldrix run`. Its tokens are verified on the CPU lane (x86-64-v3) on a plain-text prompt; it is
  not verified on Metal or AMD GPUs, and no check covers the chat template.

### Changed

- `caldrix run` with a model that has no chat template (such as `smollm2`) stops each reply at 256
  tokens unless `--max-tokens` is given; before, a base model could run until the context filled.

## [0.1.1] - 2026-10-16

### Added

- A release archive for macOS on Apple silicon (`aarch64-darwin`, macOS 14.0 or newer), installed with
  the same `curl -fsSL https://caldrix.ai/install.sh | sh`.
- The installer refuses a Mac below macOS 14.0 by name, before downloading anything.

## [0.1.0] - 2026-10-16

The first release.

### Added

- `caldrix run <model>`: chat with a model in the terminal, or answer one prompt with `-p`.
- `caldrix serve <model>`: an OpenAI-compatible HTTP server, on port 8425 by default, with
  `POST /v1/chat/completions` (streaming or not), `POST /v1/completions` and `GET /v1/models`.
- `caldrix doctor`: reports which lanes this machine can run: `hip` (AMD GPUs through ROCm),
  `metal` (Apple silicon) and `cpu`.
- Models by alias (`smollm2`, `llama-8b`, `qwen3-0.6b`), by Hugging Face name
  (`owner/name[@revision]`) or from a local directory, for the Llama, Qwen2 and Qwen3 architectures.
- A release archive for Linux on x86_64 (glibc 2.38 or newer), installed with
  `curl -fsSL https://caldrix.ai/install.sh | sh`.

### Known limitations

- There is no macOS archive in this release yet; it will be added to this release when it is built.
- Generation is greedy: sampling parameters are refused.
- NVIDIA GPUs, mixture-of-experts models and quantized checkpoints are not supported.
