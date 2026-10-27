# caldrix

caldrix runs and serves language models with the Inferno compiler. It is one binary: `caldrix run` chats with a model in your terminal, `caldrix serve` starts an OpenAI-compatible HTTP server, and `caldrix doctor` tells you what your machine can run.

These docs track the main branch, which can be ahead of the latest release; what each release
changed is in the [changelog](https://github.com/caldrix-ai/caldrix/blob/main/CHANGELOG.md).

## Install

```sh
curl -fsSL https://caldrix.ai/install.sh | sh
```

With [mise](https://mise.jdx.dev): `"github:caldrix-ai/caldrix" = "latest"` under `[tools]` in `mise.toml`
([install](docs/install.md#install-with-mise)).

## Start

```sh
caldrix doctor
caldrix run qwen3-0.6b -p "What is the capital of France?"
caldrix serve qwen3-0.6b
```

Then, from another terminal:

```sh
curl http://localhost:8425/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3-0.6B", "messages": [{"role": "user", "content": "What is the capital of France?"}]}'
```

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
`smollm2`, `smollm2-instruct`, `llama-8b`, `qwen3-0.6b` and `qwen3-30b-a3b` ([models.md](docs/models.md));
`smollm2-instruct` (from v0.1.2) is verified on the CPU lane (x86-64-v3), not on Metal or AMD GPUs: its
tokens on a plain-text prompt; its chat template is not exercised by any gate. `qwen3-30b-a3b` is verified
by its token gates on one MI300X (resident), one M1 Max streamed under a 56 GiB `--weight-budget`, and
the CPU lane (x86-64-v3) streamed under a 20 GiB budget ([models.md](docs/models.md)).
Not supported: quantized checkpoints.

Generation is greedy unless asked: `--temperature`, `--top-k`, `--top-p`, `--min-p`, `--repetition-penalty`
and `--seed` turn sampling on, and the chat changes them with `/temperature` and its siblings
([run.md](docs/run.md#sampling)).

## Docs, for humans and agents

The [docs](docs/index.md) are plain Markdown, one task per page: [install](docs/install.md), [doctor](docs/doctor.md), [run](docs/run.md), [serve](docs/serve.md), [HTTP API](docs/api.md), [models](docs/models.md), [troubleshooting](docs/troubleshooting.md). Agents can start at [caldrix.ai/llms.txt](https://caldrix.ai/llms.txt).

## In the browser: caldrix.js

caldrix.js runs a language model in a web page, in a worker, on WebGPU or WebAssembly. It is the npm
package `@caldrix-ai/caldrix`; see [its docs](docs/caldrix-js.md).

The first release of @caldrix-ai/caldrix is not published yet.

## License

caldrix is free for non-commercial use. Production and commercial use need a commercial license: jyothsna@caldrix.ai. See [LICENSE.md](LICENSE.md) and the [License Terms](https://caldrix.ai/terms).

## Bugs and security

File bugs as [issues](https://github.com/caldrix-ai/caldrix/issues/new/choose) and include `caldrix --version` and `caldrix doctor` output. Report security issues privately: see [SECURITY.md](SECURITY.md).
