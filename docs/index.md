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
| `cpu` | Linux on an `x86-64-v3` CPU, with a C compiler (`cc`) on `PATH` | supported | verified on hardware at `x86-64-v3` with small models; 8B models have not been verified on this lane |
| `metal` | other Apple silicon on macOS | accepted by the binary, not verified on hardware | the lane runs on any Metal device it finds; no other chip has been verified |
| `cpu` | Linux on an `x86-64-v2` or `x86-64-v4` CPU | accepted by the binary, not verified on hardware | `caldrix doctor` reports the lane usable at these rows; they have not been verified |
| none | NVIDIA GPUs | not supported | `caldrix doctor` names an NVIDIA device when it finds one |
| `hip` | AMD GPUs other than `gfx942` | not supported | Inferno generates GPU code for `gfx942` only |
| `cpu` | the CPU lane on macOS | not supported | the lane builds for Linux only |

Release archives are built for `x86_64-linux` (glibc 2.38 or newer) and, from v0.1.1, `aarch64-darwin`
(macOS 14.0 or newer on Apple silicon). The installer refuses any other platform.

Models: the import accepts checkpoints of five architectures (`LlamaForCausalLM`,
`MistralForCausalLM`, `Qwen2ForCausalLM`, `Qwen3ForCausalLM` and the mixture-of-experts
`Qwen3MoeForCausalLM`) in safetensors, bf16 or f32. Mistral is accepted by the import but has not been
verified on hardware. Five aliases name the models Inferno has run on hardware, each pinned to a revision:
`smollm2`, `smollm2-instruct`, `llama-8b`, `qwen3-0.6b` and `qwen3-30b-a3b` ([models.md](models.md));
`smollm2-instruct` (from v0.1.2) is verified on the CPU lane (x86-64-v3), not on Metal or AMD GPUs: its
tokens on a plain-text prompt; its chat template is not exercised by any gate. `qwen3-30b-a3b` is verified
by its token gates on one MI300X (resident), one M1 Max streamed under a 56 GiB `--weight-budget`, and
the CPU lane (x86-64-v3) streamed under a 20 GiB budget ([models.md](models.md)).
Quantized: Ollama Q4_K_M and Q8_0 tags on the `metal` lane only ([models.md](models.md#ollama-tags)); quantized safetensors checkpoints are not supported.

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
- [caldrix doctor](doctor.md): what each line of the report means.
- [caldrix run](run.md): one prompt, or a chat in the terminal.
- [caldrix serve](serve.md): the HTTP server and its flags.
- [HTTP API](api.md): the endpoints, the request fields and the errors.
- [caldrix pool](pool.md): the daemon that holds models for local clients.
- [Models](models.md): the aliases, the Hugging Face cache and what the import refuses.
- [Troubleshooting](troubleshooting.md): each refusal, its cause and its fix.

To run a model in the browser instead, see [caldrix.js](caldrix-js.md), the npm package `@caldrix-ai/caldrix`.
