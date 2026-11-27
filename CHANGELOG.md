# Changelog

Every caldrix release, newest first. Each release's notes on GitHub are its section here.

## [0.3.0] - 2026-11-27

### Changed

- Breaking: `--prefill-attention` (AMD lane only) takes `rows` (the default) or `tiles`; v0.2.0's values
  `lane0` and `tree` are refused by name. `tiles` needs `--max-batch 8`, a bf16 Llama model and, with
  `--prefill-chunk N`, `N` a multiple of 32 ([caldrix serve](https://caldrix.ai/docs/serve#flags)).
- `caldrix doctor` prints a fourth lane, `nvidia`, between `metal` and `hip`; each lane line ends with
  `speculation=yes` or `speculation=no`; the `not supported` block is now `model support`, which says which
  models run on which lane, followed by a `dense snapshots` block and one `images on <lane>:` and one
  `image generation on <lane>:` line per lane ([caldrix doctor](https://caldrix.ai/docs/doctor)).
- The CPU lane links its kernels with the `ld.lld` the release archive puts beside `caldrix` (or
  `INFERNO_LLD`, or an `ld.lld` on `PATH`), and needs a C compiler (`cc`) only when it finds none
  ([cpu](https://caldrix.ai/docs/doctor#cpu)).
- f16 checkpoints are accepted on every lane, computed in bf16 by default or exactly with `--dtype f32`;
  v0.2.0 refused them by name ([Models](https://caldrix.ai/docs/models)).
- A model with an active sliding window (a Mistral `sliding_window`, a Qwen `use_sliding_window`), which
  v0.2.0 refused, runs at a context no wider than its window, and a wider `--ctx` is refused with the
  window's size. `--backend cpu` applies the window and runs at any context, and `--backend metal` applies
  it too: on one Apple M1 Max, TinyMistral-248M matched the reference's 48 tokens past its window. `--backend
  nvidia` is built to apply it and no NVIDIA card has run it, so `auto` does not take the card for such a
  model; `--backend hip` keeps the context limit ([What the import refuses](https://caldrix.ai/docs/models#what-the-import-refuses)).
- Weight files are chosen as the `transformers` library chooses them (`model.safetensors` alone when
  present, else the files its index names, else every `*.safetensors`), and a note names the files not
  read. A repo shipping full weights beside an `adapter_config.json` runs those weights, and is refused
  only when they are byte for byte the adapter's base model's
  ([What is downloaded](https://caldrix.ai/docs/models#what-is-downloaded)).
- A checkpoint whose `config.json` ties the output head to the embedding but which ships a different
  `lm_head.weight` runs with the shipped head, as `transformers` does, and says so at load.
- When a model's chat template refuses a system message (`--system`, `/system`, or a `system` message to
  `caldrix serve`), the system text is put at the top of the first user message, and stderr or the server
  log says so once ([Chat](https://caldrix.ai/docs/run#chat)).
- `generation_config.json` is downloaded when a repo has one, and its `eos_token_id` joins the ids a
  generation stops on.
- `--backend auto` can take an NVIDIA GPU (below); everything it took in v0.2.0 on a machine without one
  is unchanged ([What `--backend auto` does with the GPU](https://caldrix.ai/docs/doctor#nvidia)).

### Added

- The `nvidia` lane, `--backend nvidia`, on Linux with the NVIDIA driver, for cards of compute capability
  6.1, 7.5, 8.0, 8.6, 8.9 and 12.0; `caldrix doctor` names any other card and its capability and refuses it.
  Inferno's kernel and token tests, `caldrix run` and one `caldrix serve` request per model passed by
  `--backend nvidia` on one rented card of each capability (a GeForce GTX 1080, an RTX 2080 Ti, an A100, an
  RTX 3060, an RTX 4070 and an RTX 5070) for small dense Llama, Qwen3 and Qwen2 models, and on the GTX 1080
  for a quantized Ollama tag (SmolLM2-135M Q4_K_M), Qwen3.5-0.8B-Base, the Ollama tag `qwen3.5:0.8b` and
  SmolVLM-256M (`caldrix serve` with four image requests, `caldrix run -i` with one). Qwen3-30B-A3B runs by
  `--backend nvidia` with `--weight-budget`, its experts streamed through the card, and the new
  `--expert-host-budget` keeps experts already read in host memory: on a GTX 1080 Ti `caldrix run` and two
  `caldrix serve` requests gave the reference's tokens with a 6 GiB `--weight-budget` and an 18 GiB
  `--expert-host-budget`, and it ran the same way on an RTX 3060, a Tesla T4, an RTX 4070, an RTX 5070 and an
  A100. `--backend auto` takes the card only for a dense Llama, Qwen3 or Qwen2 model in a local directory
  that fits the card's free memory, or for a dense model the lane runs when no other lane is usable; it
  chose the card for the small Llama, Qwen3 and Qwen2 directories on the GTX 1080, RTX 2080 Ti, A100, RTX
  3060, RTX 4070 and RTX 5070. Quantized files, Qwen3.5, image-taking models, mixture-of-experts models
  and models named by Hugging Face repo or alias go on the card by `--backend nvidia` only. A quantized
  Qwen3.5 file whose value heads outnumber its key heads (the 27B files) and image generation are refused
  on this lane ([nvidia](https://caldrix.ai/docs/doctor#nvidia)).
- Qwen3.5 (`Qwen3_5ForCausalLM`, and `Qwen3_5ForConditionalGeneration` as text only) and the
  `decider-0.8b` alias, a decision model, not a chat model. Their token gates ran at the desk on the CPU
  lane (x86-64-v3, a desk reading, not a run), on one MI300X and on one Apple M1 Max; `caldrix run` ran the
  safetensors model on that MI300X (single-stream: `--max-batch` lowers to 1) and on that M1 Max.
  `--max-batch` above 1 and `--weight-budget` are refused for it ([Aliases](https://caldrix.ai/docs/models#aliases)).
- Quantized Qwen3.5 GGUF files, single-stream at f32: on the CPU lane, a desk reading, `caldrix run
  ollama:qwen3.5:0.8b` gave its reference's eight tokens; on the Metal lane, on one Apple M1 Max, so did
  `caldrix run ollama:qwen3.5:0.8b --backend metal`. The Qwen3.6 35B-A3B GGUF files with a mixture of
  experts (`qwen35moe`) are admitted on the CPU and Metal lanes only: `caldrix run` of the `UD-Q6_K` and
  `UD-Q4_K_M` files of `unsloth/Qwen3.6-35B-A3B-GGUF` gave the reference's eight tokens at the desk on the CPU
  lane and on one Apple M1 Max, where `caldrix serve --backend metal` answered one completion with them
  ([Aliases](https://caldrix.ai/docs/models#aliases)).
- `POST /v1/systemone` on `caldrix serve`: typed `choice`, `score` and `noul` decisions about a `state`,
  read from the model's next-token probabilities with no text generated, zero-shot from a chat model or in
  its own trained format from a model with a `decider_config.json` such as `decider-0.8b`
  ([POST /v1/systemone](https://caldrix.ai/docs/api#post-v1systemone)).
- Images in: the `smolvlm-256m` alias (SmolVLM-256M-Instruct, `Idefics3ForConditionalGeneration`) answers
  about PNG and JPEG images, attached with `caldrix run -i` (up to 8) or `/image` in the chat, and sent to
  `caldrix serve` as OpenAI `image_url` parts in `data:` URLs. Its token gate ran at the desk on the CPU lane
  (x86-64-v3), where `caldrix run -i` ran, and through the engine and `inferno serve` on one MI300X and one
  Apple M1 Max; `caldrix run` and `caldrix serve` themselves have not been exercised on the HIP or Metal
  lanes. A WebP is refused by name ([Images](https://caldrix.ai/docs/run#images),
  [A vision model](https://caldrix.ai/docs/serve#a-vision-model)).
- Images out: `caldrix run <pipeline> -p PROMPT -o OUT.png [--size WxH] [--seed N]` makes a PNG with FLUX.2
  klein 4B (the `flux2-klein-4b` alias, a pipeline directory holding `model_index.json`, or a Hub repo id
  whose file list holds one), and `caldrix serve` on a pipeline serves `POST /v1/images/generations`;
  `--denoiser-weights FILE` reads the denoiser from a GGUF file. At the desk on the CPU lane (x86-64-v3) a
  256×256 image in bf16 was compared step by step with the PyTorch reference and a 512×512 one made with no
  reference; on one Apple M1 Max `caldrix run --backend metal` made 256×256, 512×512 and 1024×1024 images
  (only the 256×256 compared with a reference) and the endpoint answered one 256×256 request. The default
  1024×1024 has not run end to end on the CPU lane; `hip` and `nvidia` refuse it. No image quality or speed
  is claimed ([Generating an image](https://caldrix.ai/docs/run#generating-an-image),
  [POST /v1/images/generations](https://caldrix.ai/docs/api#post-v1imagesgenerations)).
- Speculative decoding: `--draft-model DIR|ALIAS|REPO` on `caldrix run` and `caldrix serve`, the serving
  flags `--speculate off|draft|ngram|auto`, `--draft`, `--draft-tokens`, `--ngram` and
  `--speculation-memory`, and `caldrix models drafts <repo>`. A draft that fails its checks falls back to
  n-gram lookup with a stderr line; a greedy reply is plain decoding's tokens (one recorded exception on
  `metal` at bf16, near-ties). `caldrix` picks a draft by itself on the AMD lane only, and on the Metal lane
  adds n-gram lookup with no draft. Single-stream only; no speed is claimed
  ([Speculative decoding](https://caldrix.ai/docs/run#speculative-decoding)).
- One GGUF file of a Hugging Face repo, `owner/name[@revision]:<quant>` or `owner/name[@revision]:<file>.gguf`,
  checked from its header before download, downloaded into the Hugging Face cache and held to the SHA-256
  the Hub publishes; and a path to a GGUF file on this machine (`caldrix run ./model.gguf`). F32, F16, BF16,
  Q8_0, Q4_K_M and, new in this release, Q6_K are admitted from either source and from Ollama tags. The
  0.8B Qwen3.5 Q6_K file has run by its Hub reference on the CPU lane at the desk (a desk reading, not a
  run) ([GGUF files of a Hugging Face repo](https://caldrix.ai/docs/models#gguf-files-of-a-hugging-face-repo),
  [A GGUF file on this machine](https://caldrix.ai/docs/models#a-gguf-file-on-this-machine)).
- On the CPU lane, from a local GGUF file only and with `--backend cpu`: the ternary `TQ1_0` and `TQ2_0`,
  `Q1_0`, `Q2_K`, `Q4_1`, `Q5_K`, `IQ4_XS`, the `IQ1`, `IQ2` and `IQ3` kinds, `MXFP4`, `NVFP4`, and BitNet
  b1.58 files. Files of these kinds have run on one `x86-64-v3` CPU only; every other lane refuses them by
  name ([A GGUF file on this machine](https://caldrix.ai/docs/models#a-gguf-file-on-this-machine)).
- Quantized safetensors checkpoints in GPTQ, AWQ, compressed-tensors int and bitsandbytes nf4, refused in
  v0.2.0: kept compressed on the CPU lane where a CPU record holds their scheme (desk readings of one real
  checkpoint of each format, not runs), and expanded in memory at load on the `metal`, `hip` and `nvidia`
  lanes, which admit them and have run none. FP8 checkpoints are refused by name before download
  ([caldrix doctor](https://caldrix.ai/docs/doctor)).
- MLX checkpoints (kept compressed on the CPU lane where a CPU record holds their variant, else expanded in
  memory at load) and Jlama Q4 checkpoints (expanded in memory at load on every lane): the CPU lane has desk
  readings, not runs; no other lane has run one ([caldrix doctor](https://caldrix.ai/docs/doctor)).
- PEFT LoRA adapter repos, bitsandbytes nf4 repos and QLoRA pairs by Hub id, and `caldrix run <dir>` on a
  local LoRA adapter directory (an `adapter_config.json` with no `config.json`) over the Hub base it names:
  the adapter is folded in memory at load and nothing dense is written to disk. Five pinned Hub pairs (two
  LoRA adapters, two nf4 repos, one QLoRA adapter) gave their references' tokens at f32 on the CPU lane at
  the desk; `--weight-budget` with an adapter is refused by name.
- NemotronH (`NemotronHForCausalLM`) on the CPU lane only (`--backend cpu`), from a dense checkpoint, an MLX
  one or a GGUF file in the kinds its records hold: at the desk `caldrix run --backend cpu` gave the
  reference's 16 greedy tokens on one prompt for a 0.9B model with experts and a 4B one without, a desk
  reading, not a run; every other lane refuses it by name
  ([What the import refuses](https://caldrix.ai/docs/models#what-the-import-refuses)).
- Qwen2 and Qwen3 models converted to multi-head latent attention (a `config.json` with an `mha2mla` block:
  a Qwen3's `uniform` selection, and two named `2-norm` Qwen2 conversions) on the CPU lane only, desk
  readings, not runs ([What the import refuses](https://caldrix.ai/docs/models#what-the-import-refuses)).
- Repos without `tokenizer.json`: a SentencePiece `tokenizer.model`, or `vocab.json` with `merges.txt`, is
  fetched right after `config.json` and converted when inferno supports its kind; otherwise the refusal
  says why and nothing else is downloaded. `chat_template.jinja` and `chat_template.json` are read for the
  chat template ([What is downloaded](https://caldrix.ai/docs/models#what-is-downloaded)).
- The import also accepts a Llama's `attention_bias` and `mlp_bias`, `linear` and `dynamic` `rope_scaling`
  (a `dynamic` model up to its own `max_position_embeddings`; a wider context is refused by name), an
  explicit `head_dim`, and on the `hip` and `metal` lanes a `head_dim` that is not a power of two
  (h2o-danube3-500m, head_dim 96, ran on one MI300X and one Apple M1 Max).
- On the AMD lane: Ollama Q4_K_M and Q8_0 tags were token-exact in f32 on one MI300X (five tags, eight
  greedy tokens of one prompt each), where v0.2.0 had run none on an AMD device
  ([Ollama tags](https://caldrix.ai/docs/models#ollama-tags)).
- Serving flags `--admission fill|one` and, AMD lane only, `--prefill-grid off|on` and
  `--prefill-attention tiles`, with no speed claimed for any of them
  ([caldrix serve](https://caldrix.ai/docs/serve#flags)).
- `caldrix models info --json` carries the fit's `memory_bytes`, and `caldrix models` marks quantized and
  adapter repos with the route they run on ([caldrix models](https://caldrix.ai/docs/models#caldrix-models)).

### Fixed

- Llama-class `tokenizer.json` files are rebuilt as `transformers` 5.22.0 builds them, a character with no
  token is left out of the prompt as `transformers` leaves it out, and a CodeLlama prompt with `<FILL_ME>` is
  rewritten for fill-in-the-middle.
- `tokenizer.json`'s padding and truncation settings are never applied to a prompt.
- `/debug/requests` holds a request by the time its response ends.

### Known limitations

- The NVIDIA runs above were made with builds of the source on rented cards; the 0.3.0 release archive
  itself has not been run on an NVIDIA card. No other compute capability is accepted, and `auto` never puts
  a quantized, Qwen3.5, image-taking, mixture-of-experts or Hub-named model on the card.
- No device lane (`metal`, `hip`, `nvidia`) has run a quantized safetensors, MLX or Jlama checkpoint; the CPU
  lane has desk readings of them, not runs. FP8 checkpoints are refused.
- `caldrix run` and `caldrix serve` on a model that takes images have not been exercised on the HIP or Metal
  lanes; image generation is refused on `hip` and `nvidia`.
- No speed is claimed for speculative decoding, the NVIDIA lane or image generation.

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
