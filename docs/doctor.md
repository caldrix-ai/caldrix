# caldrix doctor

`caldrix doctor` reports, for each of the four lanes (Apple GPU through Metal, NVIDIA GPU, AMD GPU through
HIP, the CPU), whether this machine can use it and why, then what models Inferno runs on which lane. It
reports only what it probed, launches no kernel, and exits with status 1 when no lane is usable. One NVIDIA
compute capability, 6.1, is accepted; the `nvidia` line says `no` for every other card.

## Run it

```sh
caldrix doctor
```

On a Linux x86-64 machine with neither an AMD GPU nor the NVIDIA driver, a build of the current source
(hence `unknown commit`) printed the output below. Three kinds of machine-specific text are replaced by
`…`: the `nvidia` line's loader message after `detail:` (it names the driver library file the loader
looked for and why it failed), the `cpu` line's path to its linker and its memory readings, and the cache
directories under the `index` and `ollama cache` blocks.

```text
caldrix 0.2.0 (unknown commit, x86_64-linux)

lanes
  metal  no      the Metal lane exists on macOS only; speculation=yes
  nvidia no      no NVIDIA driver library; detail: …; speculation=no
  hip    no      the HIP runtime did not load: dlopen libamdhip64.so: libamdhip64.so: cannot open shared object file: No such file or directory; speculation=yes
  cpu    usable  x86-64-v3 (12 physical cores, cgroup quota 8) features=sse4_2,popcnt,avx,avx2,fma,f16c,bmi1,bmi2,movbe,lzcnt; ld.lld at …; libm glibc 2.42; …; speculation=yes

index
  absent   no manifest at …

ollama cache
  …: empty

dense snapshots
  none

model support
  Inferno runs quantized GGUF (Q4_K_M, Q8_0) on the metal and hip lanes and on the cpu lane (a desk reading, not a run); quantized safetensors checkpoints other than bitsandbytes nf4 (densified at fetch) are not supported
  Inferno runs Qwen3.5 on the cpu lane (x86-64-v3; a desk reading, not a run), the hip lane (one MI300X) and the metal lane (one Apple M1 Max)
  Inferno runs a quantized Qwen3.5 file on the cpu lane (x86-64-v3; a desk reading, not a run) and the metal lane (one Apple M1 Max)
  Inferno runs a dense checkpoint on the nvidia lane, on compute capability 6.1 only: by --backend nvidia (the engine's gates, caldrix run and one caldrix serve request held on one GTX 1080 with one 135M model), and by auto for a dense Llama-family directory that fits the card's free memory, or for any dense model the lane runs when no other lane is usable, every other model keeping its other lane (auto took the card on one GTX 1080 for that model's directory); quantized, Qwen3.5 and vision models are refused there
  images on metal: yes (SmolVLM, 64 tokens per 512-px tile)
  images on nvidia: no
  images on hip: yes (SmolVLM, 64 tokens per 512-px tile)
  images on cpu: yes (SmolVLM, 64 tokens per 512-px tile)
  image generation on metal: yes (FLUX.2 klein 4B, one Apple M1 Max at 256x256)
  image generation on nvidia: no
  image generation on hip: no
  image generation on cpu: yes (FLUX.2 klein 4B, CPU desk record)

no kernel was launched; `caldrix run` is what shows a lane works
```

## Reading the report

- **The first line** is `caldrix --version`: the version, the commit the binary was built from
  (`unknown commit` for a build outside the release script) and the platform.
- **`lanes`** has one line per lane: `metal`, `nvidia`, `hip`, `cpu`. Each says `usable` with what was
  found, or `no` with the reason. `caldrix run` and `caldrix serve` with `--backend auto` (the default) take
  the first usable lane of `metal`, `hip` and `cpu`, in that order. Ahead of `hip` and `cpu` they take a
  usable `nvidia` lane only for a dense Llama-family model in a local directory that fits the GPU's free
  memory, and after them only as the last resort ([`nvidia`](#nvidia)); a usable `nvidia` line alone does
  not mean `auto` takes it.
- **`speculation=yes` or `speculation=no`** ends each lane line: whether the lane can check a draft model's
  guesses, so `--draft-model` can speculate there ([run.md](run.md#speculative-decoding)). It reports what the
  build supports, not a measurement; today `metal`, `hip` and `cpu` say `yes` and `nvidia` says `no`.
- **The `index`, `ollama cache` and `dense snapshots` blocks** say what `caldrix` found in its caches on
  disk; `doctor` reads them and makes no request.
- **`model support`** says which model formats Inferno runs on which lanes (quantized GGUF) and where Qwen3.5
  (`Qwen3_5ForCausalLM`) runs: `caldrix` runs it on the `cpu`, `hip` and `metal` lanes ([models.md](models.md#aliases)).
  The line after it is the quantized Qwen3.5 file's: the `cpu` lane, a desk reading, and the `metal` lane, where the engine's token gate ran it once on one Apple M1 Max.
  The next line is the `nvidia` lane's: how the lane is chosen, what ran on an NVIDIA card (Inferno's own tests, and `caldrix run` and `caldrix serve` by `--backend nvidia`, and `auto` choosing the GPU for that model's directory, on one GeForce GTX 1080), and what is refused there.
  Then one `images on <lane>:` line per lane, in the lanes' order: `yes` where a model that takes images
  runs on that lane (today SmolVLM, `smolvlm-256m`, on `cpu`, `hip` and `metal`: 64 prompt tokens for each 512-pixel tile), else
  `no` ([run.md](run.md#images)). Then one `image generation on <lane>:` line per lane: `yes` on a lane
  where a run on record made images from a prompt (today FLUX.2 klein 4B, the `flux2-klein-4b` alias or a
  pipeline directory, on `cpu` and `metal`), with the record named in brackets — `CPU desk record` for the
  desk's run on the CPU lane, `one Apple M1 Max at 256x256` for the Metal lane's, which is all it covers
  ([run.md](run.md#generating-an-image)) — else `no`. Like the rest of the block, it says what this build
  supports, not what was probed on this machine: a Linux machine prints the `metal` line's `yes` too.
- **The last line** is a reminder: the report comes from probes, and only `caldrix run` shows a lane
  actually works.

## What each lane needs

### `metal`

- macOS. On any other system the line reads `the Metal lane exists on macOS only`.
- A binary built with Metal support; the macOS release archive is. Otherwise the line says the binary was
  built without the `metal` feature.
- A Metal device. When the device query fails, the line reads `no Metal device:` and the error.
- When usable, the line names the device. The working-set limit is not probed.

### `nvidia`

One compute capability, 6.1, is accepted: on such a card the line says `usable`, and `--backend nvidia` runs
a dense model there. Inferno's kernel and token tests, `caldrix run` and one `caldrix serve` request passed
on one GeForce GTX 1080 with one small dense model, by `--backend nvidia`; no other card or model has been
run, and quantized, Qwen3.5 and image models are refused on this lane. For every other card the line says
`no`, and `--backend nvidia` is refused with the same reason before anything is downloaded.

**What `--backend auto` does with the GPU.** On Linux, with an NVIDIA GPU of compute capability 6.1 and the
NVIDIA driver, `caldrix run` and `caldrix serve` under `auto` use the GPU in two cases:

- **A dense Llama-family model in a local directory that fits the GPU's free memory.** This is the one
  family whose token tests have passed on an NVIDIA GPU (SmolLM2 is of it), and the directory's
  `config.json` says what it is before anything is loaded. `auto` uses the GPU for it ahead of the `hip` and
  `cpu` lanes.
- **When no other lane is usable,** any dense model the lane runs that fits the GPU's free memory: a model
  directory, or a Hugging Face repository (a repo id, or an alias of one) of the Qwen3, Qwen2 or Mistral
  class whose class the model index or the Hub names before the download. Without the GPU such a command
  would be refused with `no usable lane`.

Everything else keeps the lane it has on a machine with no NVIDIA GPU. While another lane is usable that
is: every model named by Hugging Face repository, alias or Ollama reference, and a directory of the Qwen3,
Qwen2 or Mistral class (`auto` does not prefer the GPU for those yet: Inferno's token test has run on an
NVIDIA GPU for one Qwen3 and one Qwen2 model, `caldrix` itself has run neither there, and no Mistral model
has run there). In both cases it is also:
quantized, Qwen3.5 and image models; a GGUF file; Ollama references; `caldrix pool` members;
mixture-of-experts models; a model with an adapter; a model that uses the mish activation; Hugging Face
repositories of the Llama class, because a repository does not state its activation before the download; a
repository whose class is not known before the download (no model index, offline, a pinned revision); a
command that names a draft model or passes a flag the lane refuses, such as `--weight-budget` or
`--max-batch 8`; a model whose fit cannot be worked out; and a GPU whose free memory cannot be read. For a
directory the fit counts the weight files' size, which over-counts a model stored wider than it runs, so a
directory close to the GPU's size stays off it. `--backend nvidia` forces the GPU for any dense model the
lane runs, and `--backend cpu` forces the CPU.

`auto` choosing the GPU has not run on an NVIDIA card: the choice is tested without one, and the read of
the GPU's free memory was verified on one GeForce GTX 1080 by a test, not by `caldrix run`. `auto` taking
the GPU says the lane runs the model, not that it is faster than the CPU lane. The free memory is read once,
before the load: if another process fills the GPU in between, the load fails and `--backend cpu` runs the
model.

What the line names is what `doctor` read:

- Linux. On any other system: `the NVIDIA lane exists on Linux only`.
- The NVIDIA driver library. When it does not load, the line reads `no NVIDIA driver library` and the
  loader's message after `detail:`. `INFERNO_NVIDIA_DRIVER` names the library by absolute path instead; a
  relative value is refused by name and nothing is loaded.
- When the driver loads and sees no card: `the NVIDIA driver reports no device`. When a driver call fails:
  `the NVIDIA driver failed`, with the failed call's status and name after `detail:`.
- When it sees a card of compute capability 6.1, the line says `usable` and names the card and its
  capability, then `chosen by --backend nvidia, and by auto for a dense Llama-family directory that fits its
  free memory or when no other lane is usable`, the driver version and the card's total memory.
- When it sees any other card, the line names it, its compute capability and why that capability is not
  admitted (no card of that capability has a recorded run), then the driver version and the card's memory.

`caldrix doctor` and `--backend nvidia` load the driver library to read the card. `--backend auto` loads it
only on Linux and only once it has seen that the model and the command are ones it would use the GPU for
(the two cases above); it then reads the card and, for a card of compute capability 6.1, its free memory.
For any other model `auto` loads nothing, and the other lanes never do. `doctor` does not read the free memory. Where no lane is
usable, `auto`'s refusal says on its `nvidia` line why it did not use a GPU
([troubleshooting.md](troubleshooting.md#no-usable-lane)).

### `hip`

- Linux. On any other system: `the AMD lane runs on Linux only`.
- The HIP runtime (`libamdhip64.so`) loads and reports a device. The line reports what happened:
  the runtime did not load (with the loader's error), it loaded and reported no device, or a HIP call
  failed. A GPU architecture other than `gfx942` is reported as a failed call.
- hipBLASLt: a `libhipblaslt.so*` file under `$ROCM_PATH/lib`, or `/opt/rocm/lib` when `ROCM_PATH` is
  unset. The line names the directory it searched.
- `ld.lld`: the `INFERNO_LLD` variable, else the `ld.lld` installed beside `caldrix`, else one on `PATH`.
- When usable, the line names the GPU architecture, its compute units and VRAM, the hipBLASLt files and
  where `ld.lld` is.

### `cpu`

- Linux. On any other system: `the CPU lane is built for Linux only`.
- An x86-64 instruction set at `x86-64-v2` or above. Below that: `this host is below
  x86-64-v2; the CPU lane cannot run here`.
- A linker: the lane links its kernels when the model loads, with `ld.lld` (the `INFERNO_LLD` variable, else
  the `ld.lld` installed beside `caldrix`, else one on `PATH`) or, when there is none, with a C compiler
  named `cc` on `PATH`. With neither: `no linker: the CPU lane links its kernels at load, and neither
  ld.lld (beside caldrix, in INFERNO_LLD or on PATH) nor cc is found`.
- When usable, the line names the instruction-set row, the physical cores (and the cgroup CPU quota
  when one is set, as in a container), where the linker is (`ld.lld at …`, else `cc at …`), and the libm version.

## Exit status

| Status | When |
| --- | --- |
| `0` | at least one lane is usable |
| `1` | no lane is usable |
| `2` | an argument was given; `caldrix doctor` takes none |

## Next

A lane that says `no` names what is missing; [troubleshooting.md](troubleshooting.md) has the fixes. With a
usable lane, ask a model something: [run.md](run.md).
