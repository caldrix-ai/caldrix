# caldrix run

`caldrix run <model>` compiles a model for this machine and answers one prompt with `-p`, or, without
`-p`, chats in the terminal line by line. Generation is greedy; the answer goes to stdout and status
lines go to stderr.

## Usage

```text
caldrix run <model> [-p PROMPT] [--max-tokens N] [--ctx N] [--backend auto|hip|metal|cpu] [serving flags]
```

The model comes first; flags follow it.

## The model argument

`<model>` is one of, checked in this order:

1. **A directory** holding `config.json`, `tokenizer.json` and `*.safetensors`. An existing directory
   always wins. An argument starting with `./`, `/` or `~` that is not a directory is refused as
   `no such directory`.
2. **An alias**: `smollm2`, `smollm2-instruct` (from v0.1.2), `llama-8b` or `qwen3-0.6b`, each a Hugging Face repo
   pinned to a revision ([models.md](models.md)).
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
- Ctrl-C while an answer is streaming stops that answer. Ctrl-C at the prompt exits, as does end of
  input (Ctrl-D).

## Flags

| Flag | Meaning | Default |
| --- | --- | --- |
| `-p PROMPT`, `--prompt PROMPT` | answer this prompt and exit | chat |
| `--max-tokens N` | stop after `N` new tokens (a positive integer) | until the end-of-sequence token or a full context; from v0.1.2, 256 tokens for a model without a chat template |
| `--ctx N` | the compiled context length, in tokens | `2048` |
| `--backend auto\|hip\|metal\|cpu` | the lane: AMD GPU, Apple GPU or CPU | `auto`: the first usable of `metal`, `hip`, `cpu` ([doctor.md](doctor.md)) |

Any other flag goes to Inferno's serving library, which checks it: `--dtype`, `--threads`, `--cpu-isa`,
`--weight-budget` and the rest are the serving flags in [serve.md](serve.md#flags). A flag is checked, and the
lane chosen, before any download starts.

## Refused flags

| Flag | Why |
| --- | --- |
| `--temperature`, `--top-p`, `--top-k`, `--min-p`, `--seed`, `--repetition-penalty` | generation is greedy; sampling parameters are refused, never ignored |
| `--port`, `--host`, `--admin`, `--admin-port` | `caldrix run` serves no HTTP; use `caldrix serve` |
| `--model-dir` | give the model as the first argument |
| `--backend host` | the serving library's off-hardware test backend; `caldrix` does not expose it |

## Exit status

| Status | When |
| --- | --- |
| `0` | the answer finished, or the chat ended |
| `1` | anything refused or failing after `caldrix`'s own argument check: a model that cannot be resolved or downloaded, an unusable lane, a serving flag the server refuses (`--dtype f16`, `--threads 0`), or the engine; the reason is on stderr |
| `2` | a flag `caldrix` itself refuses ([Refused flags](#refused-flags), a bad `--backend` or `--max-tokens` value, a flag missing its value) or a missing or misplaced model argument |
| `130` | interrupted with Ctrl-C |
