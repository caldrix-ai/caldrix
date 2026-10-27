# Models

`caldrix run` and `caldrix serve` take a model as a directory, an alias or a Hugging Face repo id; an alias
or a repo id is downloaded into the standard Hugging Face cache and read from there after. The import
accepts dense Llama, Mistral, Qwen2 and Qwen3 checkpoints and the Qwen3 mixture-of-experts architecture
(`Qwen3MoeForCausalLM`) in safetensors, bf16 or f32; quantized safetensors checkpoints are refused. Quantized Ollama tags (Q4_K_M and
Q8_0) run on the `metal` lane and, at the desk, on the `cpu` lane (`x86-64-v3`); the `hip` lane compiles
them and has not run them on an AMD device ([Ollama tags](#ollama-tags)). Mistral is accepted
by the import but has not been verified on hardware.

## Aliases

Each alias names a model Inferno has run on hardware, pinned to the revision it ran at.
`smollm2-instruct` (from v0.1.2) is verified on the CPU lane (x86-64-v3), not on Metal or AMD GPUs: its tokens on a
plain-text prompt; its chat template is not exercised by any gate. `qwen3-30b-a3b` (the mixture-of-experts
checkpoint) is verified by its token gates — eight greedy tokens of one prompt against the PyTorch reference —
on one AMD MI300X with every expert resident, on one Apple M1 Max (64 GiB) streamed under a 56 GiB
`--weight-budget`, and on the CPU lane (x86-64-v3) streamed under a 20 GiB budget; without a budget the
Metal and CPU lanes load every expert resident, which no run has exercised at this size, and `--max-batch`
above 1 is refused for it.

| Alias | Hugging Face repo | Revision |
| --- | --- | --- |
| `smollm2` | `HuggingFaceTB/SmolLM2-135M` | `93efa2f097d58c2a74874c7e644dbc9b0cee75a2` |
| `smollm2-instruct` | `HuggingFaceTB/SmolLM2-135M-Instruct` | `12fd25f77366fa6b3b4b768ec3050bf629380bac` |
| `llama-8b` | `deepseek-ai/DeepSeek-R1-Distill-Llama-8B` | `6a6f4aa4197940add57724a7707d069478df56b1` |
| `qwen3-0.6b` | `Qwen/Qwen3-0.6B` | `c1899de289a04d12100db370d81485cdf75e47ca` |
| `qwen3-30b-a3b` | `Qwen/Qwen3-30B-A3B` | `ad44e777bcd18fa416d9da3bd8f70d33ebb85d39` |

## Ollama tags

An Ollama model is named `ollama:<name>[:<tag>]`, or `ollama:<namespace>/<name>[:<tag>]` outside the
library. `caldrix` downloads its GGUF from the Ollama registry.

```sh
caldrix run ollama:smollm2:135m-instruct-q8_0 -p "What is the capital of France?" --backend metal
```

A Q4_K_M or Q8_0 tag is read and sized before any download. It runs at an f32 plan, the default when
`--dtype` is not given; an explicit `--dtype bf16` and `--weight-budget` are refused by name.

- `metal`: on one Apple M1 Max, `smollm2:135m-instruct-q4_K_M`, `smollm2:135m-instruct-q8_0`,
  `qwen2.5:0.5b-instruct-q4_K_M` and `deepseek-r1:8b-llama-distill-q4_K_M` were token-exact in f32, and
  `caldrix run` of `deepseek-r1:8b-llama-distill-q4_K_M` completed.
- `cpu`: at `x86-64-v3`, `smollm2:135m-instruct-q4_K_M`, `smollm2:135m-instruct-q8_0`,
  `qwen2.5:0.5b-instruct-q4_K_M` and `qwen2.5:0.5b-instruct-q8_0` were token-exact in f32 at the desk on
  one machine — a desk reading, not a run.
- `hip`: the lane compiles quantized tags and holds a quantized model to one request at a time
  (`--max-batch` lowers from 8 to 1 with a startup line, and an explicit `--max-batch 8` is refused by
  name); nothing quantized has run on an AMD device.

Quantized safetensors checkpoints stay refused ([What the import refuses](#what-the-import-refuses)).

## caldrix models

`caldrix models` asks the downloaded model index, and the Hugging Face Hub for one repo the index lacks.

```text
caldrix models search <text>... [--json]
caldrix models info <repo> [--json]
caldrix models drafts <repo> [--json]
caldrix models related <repo> [--finetunes|--quantizations|--twins] [--json]
caldrix models supported [--all] [--json]
caldrix models update
```

`--json` prints the result as JSON. `caldrix models update` asks for the index again, even after a failed
update.

## Repo ids and revisions

Any other Hugging Face model is named `owner/name`, or `owner/name@revision` for a branch, a tag or a
commit. Without `@revision` the revision is `main`.

```sh
caldrix run Qwen/Qwen3-0.6B@main -p "What is the capital of France?"
```

The revision is resolved to its commit first, and every file is fetched at that commit. A directory
holding `config.json`, `tokenizer.json` and `*.safetensors` is used as it is, with nothing downloaded.

## What is downloaded

From the top level of the repo, at the resolved commit:

- `config.json`, first, alone. It must pass the import's checks (below) before anything else is fetched,
  so a refused model costs one small download.
- `tokenizer.json`, and `tokenizer_config.json` and `generation_config.json` when the repo has them.
- `model.safetensors.index.json` when the repo has one, then every `*.safetensors` file.

A repo without `config.json`, `tokenizer.json` or any `*.safetensors` file is refused; GGUF, `.bin` and other
weight formats are not supported. Before downloading, `caldrix` prints the number of files and their total
size, then each file. Files already in the cache are kept, so an interrupted download resumes at the
first missing file.

Downloads are not checked against a content digest: integrity rests on TLS and on the import's
validation of every file.

## The cache

Models go where Python's `huggingface_hub` puts them, so a model either tool downloaded is shared:

1. `HF_HUB_CACHE`, when set;
2. else `HF_HOME/hub`, where `HF_HOME` defaults to `$XDG_CACHE_HOME/huggingface`, else
   `~/.cache/huggingface`.

A commit's files are in `models--<owner>--<name>/snapshots/<commit>/` under the cache. To free the space,
delete a model's `models--<owner>--<name>` directory.

## Environment

| Variable | Effect |
| --- | --- |
| `HF_TOKEN` | the Hugging Face token, for gated or private repos (else the `token` file under `HF_HOME`) |
| `HF_HUB_OFFLINE` | `1`, `on`, `yes` or `true`: no network request; the model must already be complete in the cache |
| `CALDRIX_OFFLINE` | the same values; covers every network edge caldrix has (the Hub, the model index and the Ollama registry) |
| `HF_ENDPOINT` | the Hub's base URL, for a mirror |
| `HF_HUB_CACHE`, `HF_HOME` | where the cache is (above) |

With `HF_HUB_OFFLINE`, a commit is looked up as given, and a branch or tag (`main` included) through the
commit the cache recorded for it when it was downloaded; a model that is not complete there is refused.

A gated repo needs its terms accepted on the Hub with your account, then `HF_TOKEN` set.

## What the import refuses

`config.json` must declare exactly one architecture, one of `LlamaForCausalLM`, `MistralForCausalLM`,
`Qwen2ForCausalLM` and `Qwen3ForCausalLM` (`MistralForCausalLM` is accepted, not verified on hardware). Every other architecture is refused by name, which covers MoE
models. Within those four, the import refuses:

- a `torch_dtype` other than bf16 or f32 (f16 is refused by name), and any tensor stored as a type other
  than f32, f16 or bf16, which covers quantized checkpoints;
- any tensor the architecture does not use, since the compiler would ignore it;
- an activation other than SiLU, an MLP bias, an attention bias outside Qwen2, interleaved RoPE, an active
  sliding window, and any `rope_scaling` other than Llama 3's (Llama 3's is applied to the RoPE table in
  code, and a line says so at load; the Metal lane has read a scaled table on hardware, token-exact with
  the scaling taken from a `config.json` and from a GGUF's `rope_freqs.weight`, in a run on an Apple M1
  Max, the AMD lane likewise in a run on an MI300X, and the CPU lane at the desk on the 1B — a desk
  reading, not a run; the verification files say what has run);
- dimensions out of bounds or inconsistent with each other.

Each refusal names the field and the value. For a model from the Hub, the refusal comes after
`config.json` alone and says nothing else was downloaded.
