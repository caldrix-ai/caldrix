# Troubleshooting

Every refusal caldrix prints, with its cause and its fix. Each quoted message is the constant part of what the binary or the installer prints; the rest of the line (a path, a value, a repo name) varies.

## Install and platform

### The platform is not supported

> `is not a supported platform; caldrix ships for Linux x86_64 and macOS on Apple silicon`

**Cause.** The installer reads `uname -s` and `uname -m` and found neither `Linux x86_64` nor `Darwin arm64`.

**Fix.** Install on one of those two platforms. The release archives are listed in [install.md](install.md#platforms).

### The host has no usable glibc

> `this host does not appear to use glibc (ldd --version gave no glibc version)`

> `this host's glibc`

**Cause.** On Linux the installer reads the glibc version from `ldd --version`. The first message means `ldd` reported none (musl, for example). The second means a glibc was found and it is below the floor the message names (2.38).

**Fix.** Use a host with glibc 2.38 or newer. The message also names the container route; those images are not published yet ([install.md](install.md#platforms)).

### The download cannot be verified

> `neither sha256sum nor shasum is installed, so the download cannot be verified`

> `the download's checksum does not match`

**Cause.** The first: the installer checks every download and has neither `sha256sum` nor `shasum` to do it with. The second: the archive does not match its `.sha256` file, so nothing was installed.

**Fix.** For the first, install `sha256sum` (coreutils) or `shasum`. For the second, run the installer again; if it repeats, do not install the archive ([install.md](install.md#what-is-checked)).

## `caldrix doctor` lanes

`caldrix doctor` prints one line per lane, and `--backend auto` takes the first lane that says `usable` ([doctor.md](doctor.md)). A line saying `no` carries one of these reasons. Nothing in `doctor` launches a kernel.

### No usable lane

> `no usable lane (`

**Cause.** `--backend auto` (the default for `run` and `serve`) tried `metal`, `hip` and `cpu` in turn and every one was refused. The message lists each lane's reason, one per line.

**Fix.** Read the reasons below; the lane closest to your hardware is the one to fix. `caldrix doctor` shows the full report, and `--backend cpu` is the lane for a machine with no supported GPU.

### The Metal lane

> `the Metal lane exists on macOS only`

> `this caldrix was built without the metal feature`

> `no Metal device: `

**Cause.** The machine is not running macOS; or the binary was built without Metal support; or the Metal device query failed (the error follows).

**Fix.** Use the macOS release archive on Apple silicon. On another system, use the `hip` or `cpu` lane.

### The AMD (HIP) lane

> `the AMD lane runs on Linux only`

> `the HIP runtime did not load: `

> `HIP loaded and reported no device`

> `a HIP call failed: `

> `libhipblaslt.so* not found under `

> `ld.lld not found beside caldrix, in INFERNO_LLD or on PATH`

**Cause.** In order: not Linux; `libamdhip64.so` did not load (the loader's error follows); it loaded and found no AMD GPU; a HIP call failed (a GPU architecture other than `gfx942` is reported here); no `libhipblaslt.so*` under `$ROCM_PATH/lib` (or `/opt/rocm/lib` when `ROCM_PATH` is unset); no `ld.lld`.

**Fix.** Install the ROCm runtime and hipBLASLt where the message says it looked, or set `ROCM_PATH`. Put `ld.lld` beside `caldrix`, on `PATH`, or point `INFERNO_LLD` at it; the release archive ships one. See [doctor.md](doctor.md#hip).

### The CPU lane

> `the CPU lane is built for Linux only`

> `this host is below x86-64-v2; the CPU lane cannot run here`

> `no C compiler: the CPU lane links its kernels with cc at load, and cc is not on PATH`

**Cause.** Not Linux; a processor below the x86-64-v2 instruction-set level; or no C compiler named `cc` on `PATH`, which the CPU lane needs when it loads a model.

**Fix.** Run on Linux x86-64 with a newer processor, and install a C compiler (`gcc` or `clang`, provided as `cc`). See [doctor.md](doctor.md#cpu).

## Model download and import

### The model argument is not a model

> `: no such directory`

> `is not a directory, an alias (`

**Cause.** The first: the argument starts with `./`, `/` or `~` and is not a directory. The second: it is not a directory, not an alias and not a Hugging Face repo id of the form `owner/name` or `owner/name@revision`.

**Fix.** Give an existing directory, an alias, or a repo id ([models.md](models.md#repo-ids-and-revisions)). The second message lists the aliases.

### The Hub refuses or cannot find the repo

> `the hub refused the request; the repo does not exist, or it is`

> `: no such repo or revision on the hub`

**Cause.** The repo is gated or private (or does not exist), or the revision does not exist.

**Fix.** Check the spelling. For a gated repo, accept its terms on the Hub and set `HF_TOKEN` ([models.md](models.md#environment)).

### Offline and not in the cache

> `HF_HUB_OFFLINE is set and `

**Cause.** `HF_HUB_OFFLINE` is set, so nothing is downloaded, and the repo at that revision is not complete in the Hugging Face cache.

**Fix.** Unset `HF_HUB_OFFLINE` and run once with network access, or fill the cache another way ([models.md](models.md#the-cache)).

### The repo is missing files

> `inferno needs config.json, tokenizer.json and *.safetensors`

> `the repo has no *.safetensors; inferno reads safetensors only`

**Cause.** The repo lacks `config.json`, `tokenizer.json` or any `*.safetensors` file. GGUF, `.bin` and other formats are not read.

**Fix.** Use a repo that publishes safetensors with a tokenizer ([models.md](models.md#what-is-downloaded)).

### The model's `config.json` is refused

> `unsupported architectures entry`

> `exactly one is required`

> `is not supported: no lane reads f16 weights`

> `attention_bias true is not implemented for this architecture`

> `nothing else was downloaded`

**Cause.** The import checks `config.json` before anything else. The architecture is not `LlamaForCausalLM`, `MistralForCausalLM`, `Qwen2ForCausalLM` or `Qwen3ForCausalLM`, or `config.json` declares more than one; the `torch_dtype` is f16; or an attention bias is set on an architecture that has none. For a Hub model the message ends by saying nothing else was downloaded.

**Fix.** Use a model in one of the four supported architectures, stored in bf16 or f32. The full list of what the import refuses is in [models.md](models.md#what-the-import-refuses).

### The download did not finish

> `the download left an incomplete snapshot at `

**Cause.** The files were fetched but the snapshot in the cache is still missing one.

**Fix.** Run the same command again. Files already in the snapshot are kept, so an interrupted download resumes at the first missing file.

## `caldrix run` refusals

`caldrix` refuses the argument errors in this section itself, before any model loads, and exits with status `2` ([run.md](run.md#exit-status)). The two entries under [While `run` answers](#while-run-answers) come later and behave differently.

### The model is missing or misplaced

> `the model is required`

> `the model comes first, got the flag`

**Cause.** No model argument was given, or a flag came before it.

**Fix.** The model is the first argument: `caldrix run <model> ...` ([run.md](run.md#the-model-argument)).

### A flag `run` does not take

> `caldrix run serves no HTTP; use caldrix serve`

> `--model-dir: give the model as the first argument`

> `--backend host is inferno-serve's off-hardware stand-in for its tests;`

**Cause.** `--port`, `--host`, `--admin` or `--admin-port` was given to `run`; or `--model-dir`; or `--backend host`.

**Fix.** Use `caldrix serve` for HTTP flags. Give the model as the first argument. Choose `auto`, `hip`, `metal` or `cpu` for `--backend` ([run.md](run.md#refused-flags)).

### A bad value

> `--backend must be auto, hip, metal or cpu, got`

> `--max-tokens must be a positive integer, got`

> `needs a value`

**Cause.** The value of `--backend` is not one of the four lanes; `--max-tokens` is not a positive integer; or a flag ended the command line without its value.

**Fix.** Correct the value ([run.md](run.md#flags)).

### An unknown command

> `unknown command`

> `takes no arguments, got`

**Cause.** The first argument is not `run`, `serve`, `doctor`, `help` or `version`; or `doctor` was given an argument.

**Fix.** Run `caldrix help` for the usage.

## While `run` answers

These happen after the model has loaded, so they are not argument errors and do not exit with status `2`.

### The prompt does not fit

> `restart with a larger --ctx or send a shorter message`

**Cause.** The prompt is at least as long as the compiled context (`--ctx`, `2048` tokens by default).

**Behaviour.** In a chat the message is printed to stderr and the chat continues, so the next message can be sent. With `-p` it ends the run with exit status `1` ([run.md](run.md#exit-status)).

**Fix.** Restart with a larger `--ctx`, or send a shorter message. In a chat, the oldest turns are dropped first to make room for the reply.

### The model has no chat template

> `this model has no chat template; each prompt is sent as plain text, without the conversation`

**Cause.** This is a notice printed to stderr, not a refusal; the run continues and the exit status is unchanged: the model's tokenizer carries no chat template, so `caldrix run` in chat mode sends each prompt alone.

**Fix.** None needed. Use a model with a chat template if the conversation should carry over ([run.md](run.md#chat)).

## `caldrix serve` flag refusals

`caldrix serve` passes your flags to Inferno's server, which refuses these before any download starts. The flags are in [serve.md](serve.md#flags).

### An unknown flag

> `unknown flag`

**Cause.** The server does not parse that flag.

**Fix.** Check the spelling against [serve.md](serve.md#flags).

### A backend this build cannot open

> `the CPU lane builds for *-unknown-linux-gnu only`

> `the Metal backend exists only on macOS, built with`

**Cause.** `--backend cpu` on a system that is not Linux, or `--backend metal` on a build that is not macOS with Metal.

**Fix.** Pick the lane `caldrix doctor` shows as usable.

### A flag that needs another

> `--admin requires --admin-port`

> `--admin-port is only meaningful with --admin`

> `is only meaningful with --weight-budget`

> `is only meaningful with --backend cpu`

**Cause.** `--admin` and `--admin-port` go together. `--weight-pin`, `--weight-ring`, `--weight-digest`, `--weight-codec`, `--weight-pins` and `--layout-dir` apply only with `--weight-budget`. `--threads` and `--cpu-isa` apply only to the `cpu` lane.

**Fix.** Add the missing flag or drop the one that depends on it.

### Flags that exclude each other

> `streamed weights are single-stream: --weight-budget cannot be combined with --max-batch 8`

> `streamed weights hold one session: --weight-budget cannot be combined with --admin`

> `batched serving on the CPU lane is C3; restart with --max-batch 1`

> `: there is no batched Metal plan (--backend metal); use --backend hip or host`

> `the CPU lane has no variant swap in first light`

**Cause.** `--weight-budget` cannot be used with `--max-batch 8` or `--admin`. The `cpu` lane serves one request at a time and has no `--admin`. The `metal` lane has no batched plan.

**Fix.** Drop one of the two flags, or use `--max-batch 1`.

### A value out of range

> `: must be 1 or 8 (the batched plan is compiled at B = 8)`

> `--weight-budget must be a positive byte count, optionally suffixed g or m`

> `--weight-ring must be >= 2`

> `must differ from --port`

> `--max-context must be >= 1`

> `--queue-depth must be >= 1`

**Cause.** The value is outside what the flag takes.

**Fix.** Use a value from the table in [serve.md](serve.md#flags).
