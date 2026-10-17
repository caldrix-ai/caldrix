# Changelog

Every caldrix release, newest first. Each release's notes on GitHub are its section here.

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
