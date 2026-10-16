# Models

`caldrix run` and `caldrix serve` take a model as a directory, an alias or a Hugging Face repo id; an alias
or a repo id is downloaded into the standard Hugging Face cache and read from there after. The import
accepts dense Llama, Mistral, Qwen2 and Qwen3 checkpoints in safetensors, bf16 or f32, and refuses
mixture-of-experts (MoE) architectures and quantized checkpoints. Mistral is accepted by the import but has
not been verified on hardware.

## Aliases

Each alias names a model Inferno has run on hardware, pinned to the revision it ran at:

| Alias | Hugging Face repo | Revision |
| --- | --- | --- |
| `smollm2` | `HuggingFaceTB/SmolLM2-135M` | `93efa2f097d58c2a74874c7e644dbc9b0cee75a2` |
| `llama-8b` | `deepseek-ai/DeepSeek-R1-Distill-Llama-8B` | `6a6f4aa4197940add57724a7707d069478df56b1` |
| `qwen3-0.6b` | `Qwen/Qwen3-0.6B` | `c1899de289a04d12100db370d81485cdf75e47ca` |

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
  sliding window, and any `rope_scaling` other than Llama 3's (accepted and not applied: the RoPE table is
  unscaled, and a line says so);
- dimensions out of bounds or inconsistent with each other.

Each refusal names the field and the value. For a model from the Hub, the refusal comes after
`config.json` alone and says nothing else was downloaded.
