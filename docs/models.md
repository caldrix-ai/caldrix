# Models

`caldrix run` and `caldrix serve` take a model as a directory, an alias or a Hugging Face repo id; an alias
or a repo id is downloaded into the standard Hugging Face cache and read from there after. The import
accepts dense Llama, Mistral, Qwen2 and Qwen3 checkpoints, the Qwen3 mixture-of-experts architecture
(`Qwen3MoeForCausalLM`) and Qwen3.5 (`Qwen3_5ForCausalLM`, and `Qwen3_5ForConditionalGeneration` as text
only) in safetensors, bf16, f32 or f16, and SmolVLM's vision-language architecture
(`Idefics3ForConditionalGeneration`), which runs on the `cpu`, `hip` and `metal` lanes and in the browser on both tiers from caldrix.js 0.16.0 ([Aliases](#aliases)); f16 weights are computed in bf16 by default, or exactly with
`--dtype f32`; quantized safetensors checkpoints are refused. Qwen3.5 runs on the `cpu`,
`hip` and `metal` lanes, and on the `nvidia` lane by `--backend nvidia` only ([Aliases](#aliases)); `inferno serve` also serves it on `--backend host`, its off-hardware test
stand-in, which makes no hardware claim. Quantized Ollama tags (Q4_K_M and
Q8_0) run on the `metal` lane and, at the desk, on the `cpu` lane (`x86-64-v3`); on the `hip` lane five
tags were token-exact in f32 on one MI300X ([Ollama tags](#ollama-tags)). Mistral is accepted
by the import but has not been verified on hardware. FLUX.2 klein 4B, an image pipeline, makes images
from a prompt on the `cpu` and `metal` lanes ([Image pipelines](#image-pipelines)).

## Aliases

Each alias names a model Inferno has run on hardware, pinned to the revision it ran at.
`smollm2-instruct` (from v0.1.2) is verified on the CPU lane (x86-64-v3), not on Metal or AMD GPUs: its tokens on a
plain-text prompt; its chat template is not exercised by any gate. `qwen3-30b-a3b` (the mixture-of-experts
checkpoint) is verified by its token gates — eight greedy tokens of one prompt against the PyTorch reference —
on one AMD MI300X with every expert resident, on one Apple M1 Max (64 GiB) streamed under a 56 GiB
`--weight-budget`, and on the CPU lane (x86-64-v3) streamed under a 20 GiB budget; without a budget the
Metal and CPU lanes load every expert resident, which no run has exercised at this size, and `--max-batch`
above 1 is refused for it. `decider-0.8b` (Qwen3.5, `Qwen3_5ForCausalLM`) is a decision model, not a chat
model. It and `Qwen/Qwen3.5-0.8B-Base` are verified by their token gates — eight greedy tokens of one prompt
in f32 and in bf16, and of a 1,100-token prompt, against the PyTorch reference — at the desk on the CPU lane
(x86-64-v3), on the AMD lane on one MI300X and on the Metal lane on one Apple M1 Max (64 GiB), through
the engine. `inferno serve` serves Qwen3.5 on the
`cpu` lane, on the `hip` lane (one MI300X, single-stream: the default `--max-batch` is lowered to 1), on the `metal` lane (one Apple M1 Max) and on `--backend host` (its off-hardware test stand-in, which makes no hardware claim). `caldrix` runs Qwen3.5 on the
`cpu`, `hip` and `metal` lanes; the browser refuses the safetensors model. The safetensors Qwen3.5 model also runs on the `nvidia` lane, by `--backend nvidia` only: Inferno's own token tests ran decider-0.8b and the Qwen3.5 base model on one GeForce GTX 1080, and `caldrix run` and one `caldrix serve` request ran the base model there. `caldrix run` ran the safetensors model on the `hip` lane on one MI300X and on the `metal` lane on one Apple M1 Max; `--max-batch` above 1 and `--weight-budget`
are refused for it. A quantized Qwen3.5 GGUF file runs on the `cpu` lane, a desk reading and not a hardware run: the engine generated eight tokens from one Q6_K file of the 0.8B and from the Ollama registry tag `qwen3.5:0.8b` (a Q8_0 file), each token-exact against its references, `caldrix run ollama:qwen3.5:0.8b` answered one prompt there with its reference's eight tokens, and `inferno serve --backend cpu` answered that prompt from the Q6_K file in-process (no HTTP request), cold, repeated and resumed from a checkpoint. On the `metal` lane the engine's own token gate has run it: once, on one Apple M1 Max, it generated the same eight tokens from the Q6_K file. `inferno serve` and `caldrix` admit it on the `metal` lane too, single-stream at f32, and that served path has run there once, on the same M1 Max: `inferno serve --backend metal` answered the prompt from the Q6_K file in-process (no HTTP request), cold, repeated and resumed from a checkpoint, and `caldrix run ollama:qwen3.5:0.8b --backend metal` answered it, each with its reference's eight tokens. One run; no speed or memory figure. In the browser, caldrix.js loads the Q6_K file of the 0.8B by its Hub repo id on the wasm tier: in headless Chromium on a CPU it generated the same eight tokens, at a 64-row context and at the default context, which the fit clamped to 12032 rows (this is the fit's arithmetic, not a memory reading; the first run of this file, before the load streamed it into place, was clamped to 9728). That is not a device run; the WebGPU tier has none of this model's kernels, so the default `backend: "auto"` uses the wasm tier and `backend: "webgpu"` is refused by name; the Try page does not list it. No other quantized Qwen3.5 file has run in a browser. On the `nvidia` lane a quantized Qwen3.5 file runs by `--backend nvidia` only: the engine's own token gates ran the Q6_K file of the 0.8B and the Ollama tag `qwen3.5:0.8b` (a Q8_0 file) on one GeForce GTX 1080, each generating the same eight tokens, and `caldrix run ollama:qwen3.5:0.8b --backend nvidia` and one `caldrix serve` request answered the prompt there with those tokens, single-stream at f32. `inferno serve`'s own binary has served none there. One run each; no speed or memory figure. `--backend auto` never uses the GPU for it, and a file whose value heads outnumber its key heads (the 27B files) is refused on that lane. Every other lane refuses it by name. A quantized Qwen3.5 file with a mixture of experts (the Qwen3.6 35B-A3B GGUF files) is admitted by `inferno serve` and `caldrix` on the `cpu` and `metal` lanes only, and every other lane refuses such a file by name; name the file after the repo (`owner/name@revision:<file>.gguf`), since `UD-Q4_K_M` is not a quant selector. On the `cpu` lane, a desk reading and not a hardware run: from the `UD-Q6_K` file and from the `UD-Q4_K_M` file of `unsloth/Qwen3.6-35B-A3B-GGUF` the engine generated eight tokens at one prompt, each token-exact against its reference; `inferno serve --backend cpu` answered that prompt from each file in-process (no HTTP request), cold, repeated and resumed from a checkpoint, and `caldrix run` answered it from each, every reply the reference's eight tokens. On the `metal` lane both files have run once, on one Apple M1 Max: the engine's own token gate generated the same eight tokens from each, and `inferno serve --backend metal` answered that prompt from each in-process (no HTTP request), cold, repeated and resumed from a checkpoint, with those tokens; `caldrix` has not run either file there. There is no speed or memory figure.

`smolvlm-256m` (SmolVLM-256M-Instruct, `Idefics3ForConditionalGeneration`) is a vision-language model: it takes
images as well as text, in `caldrix run` with `-i` and `/image` ([run.md](run.md#images)) and in `caldrix serve`
as image parts ([api.md](api.md)). It runs on the CPU, HIP and Metal lanes (and on the NVIDIA lane by `--backend nvidia` only, where Inferno's own token test, `caldrix serve` and `caldrix run -i` ran it on one GeForce GTX 1080), and in the browser on both
tiers through caldrix.js 0.16.0 ([caldrix-js.md](caldrix-js.md#images); on the wasm tier it has run in Playwright's
headless Chromium on Linux, on the WebGPU tier in Playwright's Chromium on the operator's Apple M1 Max on its
own adapter (`apple`, `metal-3`), each on a test build that admitted it ahead of the release, as [What has run, and
where](caldrix-js.md#what-has-run-and-where) records). `caldrix doctor` prints which lanes take images. It is verified at the desk on the
CPU lane (x86-64-v3) by its token gate: greedy tokens of fixture images against the PyTorch reference at f32 and
bf16 through the engine, and through the server at f32; at bf16 the server's tokens equal the engine's own (the vision
tower runs in f32 at either `--dtype`; the text model at the `--dtype` given). On the HIP lane its token gate ran on one
AMD Instinct MI300X (`gfx942`, ROCm 6.4): greedy tokens against the PyTorch reference through the engine in the seven gated f32 cases and
in the three scored bf16 cases, and through the server (`inferno-serve --backend hip`), whose tokens equal the
engine's own at f32 and bf16; `caldrix run` and `caldrix serve` themselves have not been exercised on the HIP lane. On the Metal
lane its token gate ran on one Apple M1 Max (macOS 15.7.9): greedy tokens against the PyTorch reference through the engine in the
seven gated f32 cases and in the three scored bf16 cases, and through the server (`inferno-serve --backend metal`), whose tokens
equal the engine's own at f32 and bf16; `caldrix run` and `caldrix serve` themselves have not been exercised on the Metal lane.
It is not a draft model, and an adapter over it is refused.

| Alias | Hugging Face repo | Revision |
| --- | --- | --- |
| `smollm2` | `HuggingFaceTB/SmolLM2-135M` | `93efa2f097d58c2a74874c7e644dbc9b0cee75a2` |
| `smollm2-instruct` | `HuggingFaceTB/SmolLM2-135M-Instruct` | `12fd25f77366fa6b3b4b768ec3050bf629380bac` |
| `llama-8b` | `deepseek-ai/DeepSeek-R1-Distill-Llama-8B` | `6a6f4aa4197940add57724a7707d069478df56b1` |
| `qwen3-0.6b` | `Qwen/Qwen3-0.6B` | `c1899de289a04d12100db370d81485cdf75e47ca` |
| `qwen3-8b` | `Qwen/Qwen3-8B` | `b968826d9c46dd6066d109eabc6255188de91218` |
| `qwen3-30b-a3b` | `Qwen/Qwen3-30B-A3B` | `ad44e777bcd18fa416d9da3bd8f70d33ebb85d39` |
| `decider-0.8b` | `Mapika/decider-0.8b` | `a0a01d6f8135298f400a8c856b355793012ae971` |
| `smolvlm-256m` | `HuggingFaceTB/SmolVLM-256M-Instruct` | `7e3e67edbbed1bf9888184d9df282b700a323964` |
| `flux2-klein-4b` | `black-forest-labs/FLUX.2-klein-4B` | `e7b7dc27f91deacad38e78976d1f2b499d76a294` |

## Image pipelines

`flux2-klein-4b` (FLUX.2 klein 4B, `black-forest-labs/FLUX.2-klein-4B`, licensed Apache-2.0 per its model
card) is an image pipeline: a text encoder, a denoiser and an autoencoder in one diffusers repo, with
`model_index.json` at its top. `caldrix run` makes one PNG from a prompt with it
([run.md](run.md#generating-an-image)) and `caldrix serve` serves it
([serve.md](serve.md#an-image-pipeline)); a directory holding `model_index.json` is used as it is, by
`caldrix run` and `caldrix serve` ([serve.md](serve.md#an-image-pipeline)). It runs on the CPU and
Metal lanes: `--backend auto` takes the CPU lane on Linux and the Metal lane on a Mac where the Metal
lane is usable ([doctor.md](doctor.md)). `caldrix` refuses
it by name on `hip`, and the browser build refuses the alias by name and does not list it. It is verified
at the desk on the CPU lane (x86-64-v3): a 256×256 image in bf16 against
the PyTorch reference, step by step, and a 512×512 image with no reference. On the Metal lane, on one
Apple M1 Max, `caldrix run` made a 256×256 image, the same step-by-step comparison with the PyTorch
reference held at 256×256, and a 512×512 image was made outside `caldrix run` with no reference; on that Mac
`caldrix run` has also made a 512×512 and a default 1024×1024 image, each once, with no reference at those sizes,
so neither was compared with anything, and `caldrix serve`'s image endpoint answered one request at 256×256,
once, with an image byte for byte the one `caldrix run` wrote. It has not been run on another Apple chip or on
a Mac with less memory than that one, and on the CPU lane the default 1024×1024 has not been run end to end.
Nothing about image quality or speed is claimed.

A Hugging Face repo id is an image pipeline when its file list holds `model_index.json` at its top, and
by nothing else: `caldrix run <owner/name> -p PROMPT -o OUT.png` looks the repo up once and runs it at
the commit that list named ([run.md](run.md#generating-an-image)), and `caldrix serve <owner/name>`
serves it ([serve.md](serve.md#an-image-pipeline)); any other repo id is a text model. The files
fetched are a fixed list, at the resolved commit: `model_index.json`, the four components'
configuration files (`scheduler/scheduler_config.json`, `text_encoder/config.json`,
`transformer/config.json`, `vae/config.json`), the tokenizer's files (`tokenizer/tokenizer.json`,
`tokenizer/tokenizer_config.json`, `tokenizer/vocab.json`, `tokenizer/merges.txt`,
`tokenizer/added_tokens.json` and the chat template `tokenizer/chat_template.jinja`, whichever the
repo has),
`vae/diffusion_pytorch_model.safetensors`, and the weight files of `text_encoder/` and `transformer/`:
the ones its index file (`model.safetensors.index.json`, `diffusion_pytorch_model.safetensors.index.json`)
names, or, with no index file, the directory's one `.safetensors` file. A directory with no index that
holds more than one, such as a variant (`*.fp16.safetensors`) beside the single file, is refused.
No other file in the repo is fetched. On `caldrix serve <owner/name>`, a repo the model index does not
hold is first described as any repo is (its file list, its configuration files and the first bytes of
its weight files), so no whole weight file is downloaded before it is judged. A repo whose file list
names more than 4096 files, or a path that is not a plain relative path, is refused whole, before
anything is fetched. The five configuration files come first, each at most 64 KiB, with the index
files, each at most 16 MiB, and each tokenizer file is held to its own limit by its listed size; an index that is not valid JSON or names a file
that is not a plain `.safetensors` name in its own directory refuses the repo by name. The
configuration files must pass the import's checks, with the memory and disk the run needs ("needs at
least": the memory figure is a lower bound), before any weight or tokenizer file is fetched.

A GGUF denoiser (`--denoiser-weights`, [serve.md](serve.md#an-image-pipeline)) is read in Black Forest
Labs' layout and widened to f32 as it loads; a file holding Q5_K tensors (unsloth's Q4_K_M) is refused by
name, and there is no quantized pipeline alias.

## Ollama tags

An Ollama model is named `ollama:<name>[:<tag>]`, or `ollama:<namespace>/<name>[:<tag>]` outside the
library. `caldrix` downloads its GGUF from the Ollama registry.

```sh
caldrix run ollama:smollm2:135m-instruct-q8_0 -p "What is the capital of France?" --backend metal
```

A Q4_K_M, Q8_0 or Q6_K tag is read and sized before any download. It runs at an f32 plan, the default when
`--dtype` is not given; an explicit `--dtype bf16` and `--weight-budget` are refused by name.

- `metal`: on one Apple M1 Max, `smollm2:135m-instruct-q4_K_M`, `smollm2:135m-instruct-q8_0`,
  `qwen2.5:0.5b-instruct-q4_K_M` and `deepseek-r1:8b-llama-distill-q4_K_M` were token-exact in f32, and
  `caldrix run` of `deepseek-r1:8b-llama-distill-q4_K_M` completed.
- `cpu`: at `x86-64-v3`, `smollm2:135m-instruct-q4_K_M`, `smollm2:135m-instruct-q8_0`,
  `qwen2.5:0.5b-instruct-q4_K_M` and `qwen2.5:0.5b-instruct-q8_0` were token-exact in f32 at the desk on
  one machine — a desk reading, not a run.
- `hip`: the lane compiles quantized tags and holds a quantized model to one request at a time
  (`--max-batch` lowers from 8 to 1 with a startup line, and an explicit `--max-batch 8` is refused by
  name). On one MI300X, `smollm2:135m-instruct-q4_K_M`, `smollm2:135m-instruct-q8_0`,
  `qwen2.5:0.5b-instruct-q4_K_M`, `qwen2.5:0.5b-instruct-q8_0` and `deepseek-r1:8b-llama-distill-q4_K_M`
  were token-exact in f32 by their token gates (eight greedy tokens of one prompt each).

Quantized safetensors checkpoints stay refused ([What the import refuses](#what-the-import-refuses)).

## GGUF files of a Hugging Face repo

One GGUF file of a Hugging Face repo is named `owner/name[@revision]:<quant>`, the quant as the file's name
ends (`Q6_K`, any case), or `owner/name[@revision]:<file>.gguf`, the file's exact name. Without
`@revision` it is `main`.

```sh
caldrix run unsloth/Qwen3.5-0.8B-GGUF:Q6_K -p "Counting to ten: one, two, three," --backend cpu
```

`caldrix` lists the repo at the commit the revision names and takes the one top-level `.gguf` file whose
name carries that quant. When two files carry it (`model-Q4_K_M.gguf` and `model-UD-Q4_K_M.gguf`), or none
does, the run is refused with every file it could be; name the file then. One part of a split file
(`-00001-of-00003.gguf`, or a header that says `split.count` above 1) and a multimodal projector
(`mmproj-*.gguf`) are refused by name. There is no default quant.

`caldrix` admits GGUF files of F32, F16, BF16, Q8_0, Q4_K_M and Q6_K, from a Hugging Face repo or an
Ollama tag alike; any other quant is refused by name. Before any byte past the file's header it checks the
quant, the header (architecture and every tensor's type), the lane and the fit, as for an Ollama tag. The
file is downloaded into the Hugging Face cache (`blobs/<sha256>`, linked from `snapshots/<commit>/<file>`),
where other Hugging Face tools find it too, and is used only when its SHA-256 equals the one the Hub
publishes for it; an interrupted download resumes. Quantized files run at an f32 plan, as an Ollama tag's
do. What has run on which lane is in [Ollama tags](#ollama-tags) and `caldrix doctor`; a file is admitted
by its header, and that a repo's file is admitted does not mean it has run.

## A GGUF file on this machine

A path to one GGUF file is a model too: `caldrix run ./model.gguf`, and the same for `caldrix serve`.
The argument is taken as a file when it starts with `./`, `../` or `/`, or ends in `.gguf` and has no
`:` in it, and the file exists. So `owner/name:<file>.gguf` is always the Hub's file; put `./` in front
to name a local file with a colon in its path.

```sh
caldrix run ./my-model.Q4_K_M.gguf -p "Hello"
```

Before anything is loaded the file's header is checked as an Ollama tag's is: the architecture, every
tensor's type, the lane and the fit. The file's name is not read for a quant: the tensors' own types
decide. Nothing is downloaded and nothing is written beside the file. The
file is your own, so there is no published SHA-256 to hold it to and none is checked. One part of a
split GGUF file is refused, and so is `--draft-model` with a file. A file is admitted by its header, and
that it is admitted does not mean a file like it has run on that lane.

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

`caldrix models drafts` lists the draft candidates the index knows for a repo. `caldrix` speculates with the
first candidate that passes its checks on this machine; declared speculator heads, MoE and quantized
checkpoints are not drafts in this release. When an earlier run on this machine settled a candidate, a line
`last run on this machine: <draft> was turned off` or `<draft> stayed on` follows, with no figure
([run.md](run.md#speculative-decoding)).

`--json` prints the result as JSON. A fractional number the index stores at 32-bit precision, such as an
adapter's `lora_alpha` in `caldrix models info --json`'s `entry`, is printed as the shortest decimal that
reads back to that 32-bit value (`0.1`, not `0.10000000149011612`). `caldrix models update` asks for the index again, even
after a failed update.

## Repo ids and revisions

Any other Hugging Face model is named `owner/name`, or `owner/name@revision` for a branch, a tag or a
commit. Without `@revision` the revision is `main`.

```sh
caldrix run Qwen/Qwen3-0.6B@main -p "What is the capital of France?"
```

The revision is resolved to its commit first, and every file is fetched at that commit. A directory
holding `config.json`, a supported tokenizer (listed below) and `*.safetensors` is used as
it is, with nothing downloaded.

## What is downloaded

From the top level of the repo, at the resolved commit:

- `config.json`, first, alone. It must pass the import's checks (below) before anything else is fetched,
  so a refused model costs one small download.
- `tokenizer.json`, and `tokenizer_config.json` and `generation_config.json` when the repo has them.
  `chat_template.json` is fetched too when the repo has it, and read for the chat template when there is
  no `chat_template.jinja`.
  A repo without `tokenizer.json` needs a SentencePiece `tokenizer.model` instead, or else both
  `vocab.json` and `merges.txt`: right after `config.json` they are fetched (with `added_tokens.json`
  and `tokenizer_config.json` when the repo has them) and checked before any weights. inferno
  converts them only when it supports that kind; otherwise the refusal says why, and nothing else is
  downloaded. A repo with `tokenizer.json` never downloads its `tokenizer.model` or the pair; one with `tokenizer.model` never downloads the pair.
- `preprocessor_config.json` and `processor_config.json` when the repo has them: they say how a vision
  model (`smolvlm-256m`'s architecture) resizes an image and cuts it into tiles. A vision model must have
  both, each within its size cap, before any weight file is fetched; on a lane that does not run it, the
  model is refused after `config.json` alone.
- The weight files, chosen the way the `transformers` library chooses them: `model.safetensors` alone when
  the repo has it; otherwise the files named in `model.safetensors.index.json`; otherwise every
  `*.safetensors` file. Any other `.safetensors` file in the repo is not downloaded and not named. When a
  local directory or an already-downloaded copy holds `.safetensors` files the rule leaves out, or the repo
  ships an adapter beside a model, `caldrix` prints a note naming the files it does not read:
  `caldrix: reads model.safetensors (as transformers loads it); not read: …`.

A repo without `config.json`, a supported tokenizer (`tokenizer.json`, a SentencePiece `tokenizer.model`, or both `vocab.json` and `merges.txt`) or any
`*.safetensors` file is refused; `.bin` and other
weight formats are not supported. A repo that holds GGUF files and no `*.safetensors` is refused with the
way to choose one, `owner/name:<quant>`, and every file it could be
([GGUF files of a Hugging Face repo](#gguf-files-of-a-hugging-face-repo)). Before downloading, `caldrix` prints the number of files and their total
size, then each file. Files already in the cache are kept, so an interrupted download resumes at the
first missing file.

With a Llama tokenizer (`LlamaTokenizer`), a character the vocabulary has no token for, not even a byte token, is left out of the prompt, as transformers leaves it out. With a CodeLlama tokenizer (`CodeLlamaTokenizer`), a prompt containing `<FILL_ME>` is rewritten for fill-in-the-middle, as transformers does.

Some repos ship full weights beside an `adapter_config.json`. `caldrix` asks the Hub for the checksums of
the repo's weight files and of the adapter's base model. If they are the same files, the repo is refused
as an adapter over that base. If they differ, the shipped model is run and the adapter files are listed
as not read. If the two cannot be compared, the shipped model is run with a note saying why.
In the browser, caldrix.js makes the same comparison from the two repos' file listings, fetching no weight bytes for it: when every weight file it would read is the base's, unchanged, the load is refused naming both repos; otherwise the model runs and `doctor().notRead` names the adapter files it does not read, ending with why the base could not be compared when it could not.

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
`Qwen2ForCausalLM`, `Qwen3ForCausalLM`, `Qwen3MoeForCausalLM`, `Qwen3_5ForCausalLM`,
`Qwen3_5ForConditionalGeneration` (text only) and `Idefics3ForConditionalGeneration` (the CPU lane only)
(`MistralForCausalLM` is accepted, not verified on hardware).
Every other architecture is refused by name. Within those, the import refuses:

- a `torch_dtype` other than bf16, f32 or f16, and any tensor stored as a type other than f32, f16 or bf16,
  which covers quantized checkpoints;
- in a Qwen3 mixture-of-experts checkpoint, expert weights stored as anything but bf16;
- any tensor the architecture does not use, since the compiler would ignore it (Qwen3.5's
  multi-token-prediction tensors, `mtp.*`, and the vision tower, `model.visual.*`, are skipped by name and
  never read);
- an activation other than SiLU, an MLP bias, an attention bias outside Qwen2, interleaved RoPE, and any `rope_scaling` other than Llama 3's (Llama 3's is applied to the RoPE table in
  code, and a line says so at load; the Metal lane has read a scaled table on hardware, token-exact with
  the scaling taken from a `config.json` and from a GGUF's `rope_freqs.weight`, in a run on an Apple M1
  Max, the AMD lane likewise in a run on an MI300X, and the CPU lane at the desk on the 1B — a desk
  reading, not a run; the verification files say what has run);
- dimensions out of bounds or inconsistent with each other;
- an `lm_head.weight` beside a tied embedding whose shape or type differs from the embedding's;
- for Qwen3.5, any `config.json` key outside the set it reads, and an active sliding window.

A checkpoint whose `config.json` ties the output head to the embedding but which ships a different
`lm_head.weight` runs with the shipped head, as transformers does; the command line says so at load (the
browser build prints no note).

Each refusal names the field and the value. For a model from the Hub, the refusal comes after
`config.json` alone and says nothing else was downloaded — except the tied-head check above, which has to
read the weights and so comes after the download.

A model with an active sliding window (a Mistral model with `sliding_window` set, or a Qwen model with
`use_sliding_window: true`) runs at a context no wider than its window: within the window the window changes
nothing, and a wider `--ctx` is refused with the window's size.
