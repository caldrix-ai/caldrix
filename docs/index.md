# caldrix quickstart

caldrix is one binary that downloads a model from Hugging Face, compiles it with the Inferno compiler for
the hardware it finds (an AMD GPU, an Apple silicon GPU or the CPU), and either answers a prompt or serves
an OpenAI-compatible HTTP API. Four commands take you from nothing to a running server: install, `caldrix doctor`,
`caldrix run`, `caldrix serve`.

These docs track the main branch, which can be ahead of the latest release; what each release
changed is in the [changelog](https://github.com/caldrix-ai/caldrix/blob/main/CHANGELOG.md).

## Supported hardware and models

A lane is supported only on hardware it has been verified on. Hardware the binary accepts but that has
not been verified is listed as not verified on hardware; everything else is not supported.

| Lane | Hardware | Status | Notes |
| --- | --- | --- | --- |
| `hip` | AMD Instinct MI300X (`gfx942`) on Linux, with ROCm's hipBLASLt | supported | verified on hardware |
| `metal` | Apple M1, M2 and M4 on macOS | supported | verified on hardware on M1, M2 and M4 |
| `cpu` | Linux on an `x86-64-v3` CPU; it links with the `ld.lld` shipped beside `caldrix`, or with a C compiler (`cc`) on `PATH` | supported | verified on hardware at `x86-64-v3` with small models; 8B models have not been verified on this lane |
| `metal` | other Apple silicon on macOS | accepted by the binary, not verified on hardware | the lane runs on any Metal device it finds; no other chip has been verified |
| `cpu` | Linux on an `x86-64-v2` or `x86-64-v4` CPU | accepted by the binary, not verified on hardware | `caldrix doctor` reports the lane usable at these rows; they have not been verified |
| `nvidia` | an NVIDIA GPU of compute capability 6.1 on Linux, with the NVIDIA driver | supported | verified on hardware on one GeForce GTX 1080 with three small dense models (one each of the Llama, Qwen3 and Qwen2 families), by `--backend nvidia`: Inferno's kernel and token tests, `caldrix run` and one `caldrix serve` request per model. `--backend auto` uses the GPU for a dense Llama, Qwen3 or Qwen2 model in a local directory that fits the GPU's free memory, and for any dense model the lane runs when no other lane is usable; every other model keeps its other lane. `auto` choosing the GPU is verified on that one GTX 1080 for each of those models' directories; the no-other-lane case is not verified on hardware. A quantized GGUF file runs on this lane by `--backend nvidia` only: Inferno's own tests ran Q8_0 and Q4_K_M files on that card, and `caldrix run` and one `caldrix serve` request passed there on one Q4_K_M Ollama tag of SmolLM2-135M. The safetensors Qwen3.5 model runs on this lane by `--backend nvidia` only: Inferno's own tests ran it on that card, and `caldrix run` and one `caldrix serve` request passed there on Qwen3.5-0.8B-Base. A quantized Qwen3.5 file runs on this lane by `--backend nvidia` only: Inferno's own token tests ran a Q6_K and a Q8_0 file of the 0.8B on that card, and `caldrix run` and one `caldrix serve` request passed there on the Q8_0 Ollama tag `qwen3.5:0.8b`. A model that takes images (SmolVLM) runs on this lane by `--backend nvidia` only: Inferno's own token test ran SmolVLM-256M on that card, and `caldrix serve` (four image requests) and `caldrix run -i` (one reply) gave the reference tokens there. A quantized Qwen3.5 file whose value heads outnumber its key heads (the 27B files) and image generation are refused on this lane |
| `nvidia` | an NVIDIA GPU of compute capability 8.6 on Linux, with the NVIDIA driver | supported | verified on hardware on one GeForce RTX 3060: Inferno's kernel and token tests (the small dense models, the quantized GGUF files, Qwen3.5 0.8B, SmolVLM-256M and speculative decoding), and `caldrix run` and one `caldrix serve` request per model by `--backend nvidia` for the same classes. `--backend auto` chose the GPU there for the small Llama, Qwen3 and Qwen2 models. The 30B mixture-of-experts model ran on another RTX 3060 by `--backend nvidia` with `--weight-budget` and `--expert-host-budget`: Inferno's token test, `caldrix run` and two `caldrix serve` requests |
| `nvidia` | an NVIDIA GPU of compute capability 7.5 on Linux, with the NVIDIA driver | supported | verified on hardware on one GeForce RTX 2080 Ti for the engine's tests and another for `caldrix`: Inferno's kernel and token tests (the small dense models, the quantized GGUF files, Qwen3.5 0.8B, SmolVLM-256M and speculative decoding), and `caldrix run` and one `caldrix serve` request per model by `--backend nvidia` for the same classes. `--backend auto` chose the GPU there for the small Llama, Qwen3 and Qwen2 models. The 30B mixture-of-experts model ran on a Tesla T4 by `--backend nvidia` with `--weight-budget` and `--expert-host-budget`: Inferno's token test, `caldrix run` and two `caldrix serve` requests |
| `nvidia` | an NVIDIA GPU of compute capability 8.9 on Linux, with the NVIDIA driver | supported | verified on hardware on one GeForce RTX 4070 for the engine's tests and another for `caldrix`: Inferno's kernel and token tests (the small dense models, the quantized GGUF files, Qwen3.5 0.8B, SmolVLM-256M and speculative decoding), and `caldrix run` and one `caldrix serve` request per model by `--backend nvidia` for the same classes. `--backend auto` chose the GPU there for the small Llama, Qwen3 and Qwen2 models. The 30B mixture-of-experts model ran on an RTX 4070 by `--backend nvidia` with `--weight-budget` and `--expert-host-budget`: Inferno's token test, `caldrix run` and two `caldrix serve` requests |
| `nvidia` | an NVIDIA GPU of compute capability 12.0 on Linux, with the NVIDIA driver | supported | verified on hardware on one GeForce RTX 5070 for the engine's tests and another for `caldrix`: Inferno's kernel and token tests (the small dense models, the quantized GGUF files, Qwen3.5 0.8B, SmolVLM-256M and speculative decoding), and `caldrix run` and one `caldrix serve` request per model by `--backend nvidia` for the same classes. `--backend auto` chose the GPU there for the small Llama, Qwen3 and Qwen2 models. The 30B mixture-of-experts model ran on an RTX 5070 by `--backend nvidia` with `--weight-budget` and `--expert-host-budget`: Inferno's token test, `caldrix run` and two `caldrix serve` requests |
| `nvidia` | NVIDIA GPUs of any other compute capability | not supported | `caldrix doctor`'s `nvidia` line names the card and its compute capability and refuses it |
| `hip` | AMD GPUs other than `gfx942` | not supported | Inferno generates GPU code for `gfx942` only |
| `cpu` | the CPU lane on macOS | not supported | the lane builds for Linux only |

Release archives are built for `x86_64-linux` (glibc 2.38 or newer) and, from v0.1.1, `aarch64-darwin`
(macOS 14.0 or newer on Apple silicon). The installer refuses any other platform.

Models: the import accepts checkpoints of six architectures (`LlamaForCausalLM`,
`MistralForCausalLM`, `Qwen2ForCausalLM`, `Qwen3ForCausalLM`, the mixture-of-experts
`Qwen3MoeForCausalLM` and Qwen3.5's `Qwen3_5ForCausalLM`, with `Qwen3_5ForConditionalGeneration` as text
only) in safetensors, bf16, f32 or f16. Mistral is accepted by the import but has not been verified on hardware.
Aliases name the models Inferno has run on hardware, each pinned to a revision: `smollm2`,
`smollm2-instruct`, `llama-8b`, `qwen3-0.6b`, `qwen3-30b-a3b` and `decider-0.8b` ([models.md](models.md));
`smollm2-instruct` (from v0.1.2) is verified on the CPU lane (x86-64-v3), not on Metal or AMD GPUs: its
tokens on a plain-text prompt; its chat template is not exercised by any gate. `qwen3-30b-a3b` is verified
by its token gates on one MI300X (resident), one M1 Max streamed under a 56 GiB `--weight-budget`, and
the CPU lane (x86-64-v3) streamed under a 20 GiB budget ([models.md](models.md)).
`decider-0.8b` (Qwen3.5) is a decision model, not a chat model; its token gates and the Qwen3.5 base
model's ran at the desk on the CPU lane (x86-64-v3), on one MI300X and on one Apple M1 Max (through the
engine), and `caldrix` runs Qwen3.5 on the `cpu`, `hip` and `metal` lanes, and on the `nvidia` lane by `--backend nvidia` only (there the engine's token gates, `caldrix run` and one `caldrix serve` request ran on one GeForce GTX 1080 with one 0.8B model); in the browser the wasm tier takes a quantized Qwen3.5 GGUF file (one Q6_K file of the 0.8B ran, in headless Chromium)
([models.md](models.md)).
Quantized: `caldrix` admits GGUF files of Q8_0, Q4_K_M and Q6_K, from an Ollama tag or a Hugging Face repo's file ([models.md](models.md#gguf-files-of-a-hugging-face-repo)); what has run: Ollama Q4_K_M and Q8_0 tags on the `metal` lane and, at the desk, on the `cpu` lane (`x86-64-v3`); on the `hip` lane, five tags were token-exact in f32 on one MI300X ([models.md](models.md#ollama-tags)); quantized safetensors checkpoints are not supported.

Generation is greedy unless asked: `--temperature`, `--top-k`, `--top-p`, `--min-p`, `--repetition-penalty`
and `--seed` turn sampling on, and the chat changes them with `/temperature` and its siblings
([run.md](run.md#sampling)).

## Start

Install the binary into `~/.local/bin` ([install.md](install.md)):

```sh
curl -fsSL https://caldrix.ai/install.sh | sh
```

See which lanes this machine can use ([doctor.md](doctor.md)):

```sh
caldrix doctor
```

Ask a model something ([run.md](run.md)). The first run downloads the model into the Hugging Face cache:

```sh
caldrix run qwen3-0.6b -p "What is the capital of France?"
```

Serve it over HTTP on port 8425 ([serve.md](serve.md)):

```sh
caldrix serve qwen3-0.6b
```

Then, from another terminal ([api.md](api.md)):

```sh
curl http://localhost:8425/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3-0.6B", "messages": [{"role": "user", "content": "What is the capital of France?"}]}'
```

Any OpenAI client works with its base URL set to `http://localhost:8425/v1`.

## All docs

- [Quickstart](index.md): this page.
- [Install](install.md): the installer, its options, checksums and uninstalling.
- [Kubernetes](kubernetes.md): the Helm chart, what it installs, its values and what it does not do.
- [caldrix doctor](doctor.md): what each line of the report means.
- [caldrix run](run.md): one prompt, or a chat in the terminal.
- [caldrix serve](serve.md): the HTTP server and its flags.
- [HTTP API](api.md): the endpoints, the request fields and the errors.
- [caldrix pool](pool.md): the daemon that holds models for local clients.
- [Models](models.md): the aliases, the Hugging Face cache and what the import refuses.
- [Troubleshooting](troubleshooting.md): each refusal, its cause and its fix.

To run a model in the browser instead, see [caldrix.js](caldrix-js.md), the npm package `@caldrix-ai/caldrix`.
