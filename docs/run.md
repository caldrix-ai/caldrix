# caldrix run

`caldrix run <model>` compiles a model for this machine and answers one prompt with `-p`, or, without
`-p`, chats in the terminal line by line. Generation is greedy unless a sampling flag asks otherwise
([Sampling](#sampling)); the answer goes to stdout and status lines go to stderr.

## Usage

```text
caldrix run <model> [-p PROMPT] [-i IMAGE]... [--system TEXT|@FILE] [--max-tokens N] [--ctx N] [--backend auto|hip|metal|cpu] [--plain]
    [--draft-model DIR|ALIAS|REPO] [--temperature T] [--top-k N] [--top-p P] [--min-p M] [--repetition-penalty R] [--seed N]
    [serving flags]
caldrix run <pipeline> -p PROMPT -o OUT.png [--size WxH] [--seed N] [--backend auto|cpu|metal] [serving flags]
```

The model comes first; flags follow it.

## The model argument

`<model>` is one of, checked in this order:

1. **A directory** holding `config.json`, a supported tokenizer ([the models page](models.md#what-is-downloaded) lists them) and
   `*.safetensors`, or an image pipeline directory holding `model_index.json`
   ([Generating an image](#generating-an-image)). An existing directory
   always wins. An argument starting with `./`, `/` or `~` that is not a directory is refused as
   `no such directory`.
2. **An alias**: `smollm2`, `smollm2-instruct` (from v0.1.2), `llama-8b`, `qwen3-0.6b`, `qwen3-8b`,
   `qwen3-30b-a3b`, `decider-0.8b`, `smolvlm-256m` or `flux2-klein-4b` (an image pipeline,
   [Generating an image](#generating-an-image)), each a Hugging Face repo pinned to a revision
   ([models.md](models.md)).
3. **A Hugging Face repo id**, `owner/name` or `owner/name@revision`. Without `@revision` it is `main`.

An alias or a repo id is downloaded into the Hugging Face cache the first time and read from there after.
Anything else is refused before any download.

## One prompt

```sh
caldrix run smollm2 --backend cpu -p "The capital of France is" --max-tokens 8
```

On a Linux x86-64 machine this printed, on stdout:

```text
 the capital of the country.
```

SmolLM2-135M is a small base model that continues text. Before the answer, `caldrix` printed status lines
on stderr: that the model was already in the Hugging Face cache (a first run lists the files it downloads
instead), the compile, the lane and its settings, and, for this model, that it has no chat template, so the
prompt is sent as plain text.

With `--backend auto` (the default) one more stderr line names the lane chosen, for example
``caldrix: backend cpu (auto; `caldrix doctor` shows why)``.

## Chat

Without `-p`, `caldrix run` reads one message per line after a `> ` prompt:

```sh
caldrix run qwen3-0.6b
```

- Each answer streams as it is generated, and the conversation is kept: each message is sent with the
  earlier turns through the model's chat template.
- `/reset` clears the conversation.
- When the conversation outgrows the compiled context (`--ctx`), the oldest turns are dropped, and
  stderr says how many. Room is kept for the answer: `--max-tokens` when given, else a quarter of the
  context, at most half of it.
- A single message too long for the context is refused, and the chat continues.
- A model without a chat template (a base model such as SmolLM2-135M) gets each line as plain text,
  without the conversation; stderr says so once. A base model continues text and rarely ends its own
  reply, so from v0.1.2, without `--max-tokens`, each reply stops at 256 tokens. To chat with a SmolLM2, use (from v0.1.2)
  `caldrix run smollm2-instruct` (SmolLM2-135M-Instruct), which has a chat template. It is verified on the
  CPU lane (x86-64-v3), not on Metal or AMD GPUs, and what was verified is its tokens on a plain-text
  prompt: the chat template is not exercised by any gate.
- When the model's chat template refuses a system message (`--system`, `/system`), the system text is
  placed at the top of the first user message instead, and stderr says so once.
- Ctrl-C while an answer is streaming stops that answer. Ctrl-C at the prompt exits, as does end of
  input (Ctrl-D).

## Images

A model that takes images (`smolvlm-256m`, on the `cpu`, `hip` and `metal` lanes; `caldrix doctor` prints `images on <lane>:`
for each lane, [doctor.md](doctor.md)) answers about a picture. `-i` attaches an image file to the first
message, and it may be given up to 8 times; in the chat, `/image PATH` attaches one to the next message.
On the `hip` lane its token gate ran through the engine and `inferno serve` on one MI300X; `caldrix run` itself
has not been exercised there. On the `metal` lane its token gate ran through the engine and `inferno serve` on one
Apple M1 Max; `caldrix run` itself has not been exercised there ([models.md](models.md#aliases)).

```sh
caldrix run smolvlm-256m -i photo.png -p "What is in this picture?"
```

On a Linux x86-64 machine on the CPU lane, `caldrix run smolvlm-256m -i wide.png -p "What is in this picture?"
--max-tokens 16 --plain` on a test image (geometric shapes on a colour gradient) printed, on stdout:

```text
 A colorful gradient background with five geometric shapes in various colors: a square, a
```

and, last on stderr, the image line:

```text
image: 512×512 ×13 tiles, 832 rows
```

- A file must be PNG or JPEG, told from its bytes, not its name, and at most 8 MiB. A WebP is refused by
  name (`WebP images are not accepted in this build; send a PNG or a JPEG`). It is read once,
  before the model is downloaded or loaded, so a refused file costs nothing; a pipe such as `-i <(curl …)`
  works.
- Each image is cut into tiles the way the HTTP API's `"detail": "auto"` does ([api.md](api.md)), and stderr
  prints one line for it, `image: 512×512 ×13 tiles, 832 rows`: the tile size, the tiles and the prompt rows
  they take. A side over 4096 pixels is refused when the image is attached.
- In the chat the images stay in the conversation: each later message is sent with them again, and they are
  decoded again for it. Past 8 images, or past `--ctx`, the oldest turns are dropped, and stderr says which
  limit dropped them. `/reset` clears the conversation and any image not yet sent.
- An image on a model that takes text only is refused (`… takes text only; -i and /image need a vision
  model`), as is `-i` with `--raw` (an image needs the chat template).

## Generating an image

An image pipeline — a directory holding `model_index.json` (a diffusers FLUX.2 klein 4B pipeline), the
`flux2-klein-4b` alias, or a Hugging Face repo id whose file list holds `model_index.json` at its top
([models.md](models.md#image-pipelines)) — makes one PNG from one prompt. Its recorded lane is `cpu`,
the only lane `--backend auto` takes for it (`caldrix doctor` prints `image generation on <lane>:` for each
lane, [doctor.md](doctor.md)); a Mac build accepts `--backend metal`, built for the Metal lane and not yet
recorded on a device.

```text
caldrix run <pipeline> -p PROMPT -o OUT.png [--size WxH] [--seed N] [--backend auto|cpu|metal] [serving flags]
```

```sh
caldrix run flux2-klein-4b -p "A watercolor of a lighthouse at dawn" -o lighthouse.png --size 256x256 --seed 0
caldrix run owner/name -p "A watercolor of a lighthouse at dawn" -o lighthouse.png
```

- `-p` and `-o` are required: there is no chat and no screen. `-o` names the PNG file to write; its
  directory must exist, and the file is written whole or not at all. Its temporary file is created
  before anything is downloaded or loaded, so a directory that cannot take it is refused up front.
  Ctrl-C (or a `SIGTERM`) before the PNG is in place removes the temporary file.
- `--size WxH` (default `1024x1024`): each side a multiple of 16 and at most 4096, and `(W/16)·(H/16)`
  at most 4096; a size outside that is refused naming `--size`, never rounded.
- `--seed N` seeds the starting noise: the same seed, prompt, size and `--dtype` give the same PNG on this
  machine and build, the same bytes as `caldrix serve`'s in-process path (tested on a reduced pipeline,
  with no HTTP between; [api.md](api.md#post-v1imagesgenerations)).
  Without it a seed is drawn, and stderr names it.
- `--backend auto` takes the `cpu` lane, and never `metal`; `hip` is refused by name, and a Mac build
  accepts an explicit `--backend metal`, built for the Metal lane and not yet recorded on a device.
  `--dtype bf16` (the default) or `f32` sets the text encoder's and the denoiser's type.
- The chat flags (`-i`, `--system`, `--max-tokens`, `--draft-model`, `--raw`, `--json`) and the text
  sampling flags (whatever their value, `--temperature 0` included) are refused by name before anything
  loads.
- One model is held at a time: the text encoder is released before the denoiser loads.
- For the alias or a repo id, the pipeline's files are downloaded by a fixed list, with the weight files
  the repo's index files name ([models.md](models.md#image-pipelines)), and its five configuration
  files are checked, with the memory and disk it needs, before any weight file is fetched; the memory
  check is a lower bound ("needs at least"), not a promise that the run fits. Once downloaded, the alias
  (pinned to a commit) and a repo id given at a commit run from the cache with no request; a repo id at
  a branch is looked up again, and nothing more is downloaded.
- A Hugging Face repo id given `-o` or `--size` is looked up once: if its file list holds
  `model_index.json` at its top, it is an image pipeline, and the whole run reads the commit that list
  named; any other repo is refused as not an image pipeline, with nothing of it downloaded. Without `-o`
  or `--size` a repo id is a text model and is not looked up. The repo-id path is tested against a local
  stand-in for the Hub only.

stderr ends with two lines: `caldrix: wrote <path> (<W>x<H>, seed <N>)`, then the time of each stage
(encode, the denoising steps, decode), the time to image, the peak resident memory, and the resident
memory after each stage, as `resident after encode/denoise/decode <A>/<B>/<C> GiB`. Each of those three
is the whole process's resident memory at that point (what earlier stages left loaded included), not
the stage's own share; one that could not be read is `unavailable`, and when none could the line ends
`resident after encode/denoise/decode unavailable`. They are this run's own readings, not a benchmark.
stdout stays empty.

What has run, at the desk on the CPU lane (x86-64-v3): a 256×256 image in bf16 compared step by step
with the PyTorch reference, and a 512×512 image made with no reference; the default 1024×1024 has not
been run end to end. No image quality or speed is claimed.

## Flags

| Flag | Meaning | Default |
| --- | --- | --- |
| `-p PROMPT`, `--prompt PROMPT` | answer this prompt and exit | chat |
| `-o OUT.png`, `--output OUT.png` | an image pipeline only: write the PNG here ([Generating an image](#generating-an-image)) | none; required for a pipeline |
| `--size WxH` | an image pipeline only: the image's width and height, each a multiple of 16 | `1024x1024` |
| `-i IMAGE`, `--image IMAGE` | attach a PNG or JPEG file to the first message; repeatable, up to 8 ([Images](#images)) | none |
| `--max-tokens N` | stop after `N` new tokens (a positive integer) | until the end-of-sequence token or a full context; from v0.1.2, 256 tokens for a model without a chat template |
| `--ctx N` | the compiled context length, in tokens | `2048` |
| `--backend auto\|hip\|metal\|cpu` | the lane: AMD GPU, Apple GPU or CPU | `auto`: the first usable of `metal`, `hip`, `cpu` ([doctor.md](doctor.md)) |
| `--draft-model DIR\|ALIAS\|REPO` | speculate with this draft model: a directory, an alias or a Hugging Face repo ([Speculative decoding](#speculative-decoding)) | none |
| `--temperature T` | sample the next token at temperature `T` (finite, `>= 0`; `0` picks the most likely token) | `0` |
| `--top-k N` | keep only the `N` most likely tokens (`0` keeps all) | `0` |
| `--top-p P` | keep the smallest set of tokens whose probability adds up to `P` (`0 < P <= 1`; `1` keeps all) | `1` |
| `--min-p M` | drop tokens whose probability is below `M` times the most likely token's (`0 <= M <= 1`) | `0` |
| `--repetition-penalty R` | divide the score of each token already in the last 64 tokens of prompt and reply by `R` (finite, `> 0`; `1` is off) | `1` |
| `--seed N` | seed the random draw, so the same prompt gives the same reply on this machine and build | fresh entropy per reply |

Any other flag goes to Inferno's serving library, which checks it: `--dtype`, `--threads`, `--cpu-isa`,
`--weight-budget` and the rest are the serving flags in [serve.md](serve.md#flags). A flag is checked, and the
lane chosen, before any download starts.

## Speculative decoding

`--draft-model` names a small model that guesses tokens ahead of the main one; the main model checks the
guesses, and a greedy reply is the same tokens plain decoding gives. One exception is recorded: on the `metal`
lane at bf16 the check runs a different order of arithmetic from plain decoding, and where plain decoding's own
top two choices lie within rounding of each other the reply can take the other one from that point on. It works
on a lane that can check a draft: `caldrix doctor` prints `speculation=yes` or `speculation=no` on each lane line
([doctor.md](doctor.md)), and today `cpu`, `hip` and `metal` say `yes`. On another lane the flag is not used,
and stderr says why.

- Before any download, a draft the model index describes must pass the architecture gate and, when the index
  also holds the main model, fit beside it: in memory with both models resident, and on disk, where the draft's
  own download is compared with the free space (the main model's download still to come is not counted). A
  draft named as a directory, or a Hub draft the index does not describe at that revision, gets neither check
  from `caldrix`; Inferno's serving library checks it when it loads it. Once the draft's files are fetched, its
  tokenizer must match the main model's.
- A suggested draft that fails the gate or the fit is passed over for the next candidate. A draft that cannot
  be fetched, whose tokenizer differs, or that fails to load is not replaced. None of this stops the run:
  stderr names the draft and why, and the run speculates with n-gram lookup (guesses taken from the text
  already in the conversation) instead.
- Without `--draft-model`, `caldrix` chooses a draft by itself from the model index only on the AMD lane, which
  can check a draft; on the `cpu` lane a draft is used only when you name it. On the `metal` lane, without
  `--draft-model`, `caldrix` fetches no draft and speculates with n-gram lookup; `--speculate off` turns that off. `caldrix models drafts <repo>` lists the candidates the index knows ([models.md](models.md)).
- A suggested draft that was turned off on this machine in an earlier run is not tried again for the same
  model. A draft you name with `--draft-model` is still tried, and stderr says it lost last time.
- A model that takes images never takes a draft step: `caldrix` fetches no draft for it, and a `--draft-model` you
  name is not used, with stderr saying why.
- Speculation runs one request at a time, so `--max-batch` other than `1` leaves it off; `--weight-budget`
  does too. Pool members do not speculate, and an Ollama model is not a draft.

Nothing here claims a speed-up: no speed has been measured.

## Sampling

Without a sampling flag every reply is greedy: each token is the most likely one, and the same
prompt gives the same reply. The six flags above turn sampling on. The penalty is applied first,
then `--top-k`, `--top-p` and `--min-p` cut the candidates, then `--temperature` reshapes what is
left and one token is drawn. `--temperature 0` with a penalty keeps the most likely token after the
penalty, with no random draw. On the AMD lane the GPU picks the token, in the same order but with
its own draw, so a seed's reply there can differ from the CPU's pick; `--sampler host` picks on the CPU
([api.md](api.md#sampling)). A value out of range is refused by name before any download starts.

In the chat, the same knobs are commands, applied from the next reply on (a reply already streaming
is not touched, and `/reset` leaves them alone):

```text
/temperature T   /top-k N   /top-p P   /min-p M   /repetition-penalty R   /seed N
```

Each takes a value, `off` (back to its default), or nothing (print its current value); `/sampling`
prints all six on one line.

## Refused flags

| Flag | Why |
| --- | --- |
| `--port`, `--host`, `--admin`, `--admin-port` | `caldrix run` serves no HTTP; use `caldrix serve` |
| `--model-dir` | give the model as the first argument |
| `--backend host` | the serving library's off-hardware test backend; `caldrix` does not expose it |

## Exit status

| Status | When |
| --- | --- |
| `0` | the answer finished, or the chat ended |
| `1` | anything refused or failing after `caldrix`'s own argument check: a model that cannot be resolved or downloaded, an unusable lane, a serving flag the server refuses (`--dtype f16`, `--threads 0`), or the engine; the reason is on stderr |
| `2` | a flag `caldrix` itself refuses ([Refused flags](#refused-flags), a bad `--backend` or `--max-tokens` value, a flag missing its value), a missing or misplaced model argument, or a refused `-i` image ([Images](#images)) |
| `130` | interrupted with Ctrl-C |
