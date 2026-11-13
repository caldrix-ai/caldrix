# caldrix doctor

`caldrix doctor` reports, for each of the four lanes (Apple GPU through Metal, NVIDIA GPU, AMD GPU through
HIP, the CPU), whether this machine can use it and why, then what models Inferno runs on which lane. It
reports only what it probed, launches no kernel, and exits with status 1 when no lane is usable. NVIDIA GPUs
are not supported: the `nvidia` line says `no` on every machine.

## Run it

```sh
caldrix doctor
```

On a Linux x86-64 machine with neither an AMD GPU nor the NVIDIA driver, a build of the current source
(hence `unknown commit`) printed the output below. Three kinds of machine-specific text are replaced by
`…`: the `nvidia` line's loader message after `detail:` (it names the driver library file the loader
looked for and why it failed), the `cpu` line's path to `cc` and its memory readings, and the cache
directories under the `index` and `ollama cache` blocks.

```text
caldrix 0.2.0 (unknown commit, x86_64-linux)

lanes
  metal  no      the Metal lane exists on macOS only; speculation=yes
  nvidia no      no NVIDIA driver library; detail: …; speculation=no
  hip    no      the HIP runtime did not load: dlopen libamdhip64.so: libamdhip64.so: cannot open shared object file: No such file or directory; speculation=yes
  cpu    usable  x86-64-v3 (12 physical cores, cgroup quota 8) features=sse4_2,popcnt,avx,avx2,fma,f16c,bmi1,bmi2,movbe,lzcnt; cc at …; libm glibc 2.42; …; speculation=yes

index
  absent   no manifest at …

ollama cache
  …: empty

dense snapshots
  none

model support
  Inferno runs quantized GGUF (Q4_K_M, Q8_0) on the metal and hip lanes and on the cpu lane (a desk reading, not a run); quantized safetensors checkpoints other than bitsandbytes nf4 (densified at fetch) are not supported
  Inferno runs Qwen3.5 on the cpu lane (x86-64-v3; a desk reading, not a run), the hip lane (one MI300X) and the metal lane (one Apple M1 Max)
  images on metal: yes (SmolVLM, 64 tokens per 512-px tile)
  images on nvidia: no
  images on hip: yes (SmolVLM, 64 tokens per 512-px tile)
  images on cpu: yes (SmolVLM, 64 tokens per 512-px tile)
  image generation on metal: no
  image generation on nvidia: no
  image generation on hip: no
  image generation on cpu: yes (FLUX.2 klein 4B, CPU desk record)

no kernel was launched; `caldrix run` is what shows a lane works
```

## Reading the report

- **The first line** is `caldrix --version`: the version, the commit the binary was built from
  (`unknown commit` for a build outside the release script) and the platform.
- **`lanes`** has one line per lane, in the order `--backend auto` tries them: `metal`, `nvidia`, `hip`,
  `cpu`. Each says `usable` with what was found, or `no` with the reason. `caldrix run` and `caldrix serve`
  with `--backend auto` (the default) take the first usable lane; the `nvidia` line is never usable, so
  `auto` never takes it.
- **`speculation=yes` or `speculation=no`** ends each lane line: whether the lane can check a draft model's
  guesses, so `--draft-model` can speculate there ([run.md](run.md#speculative-decoding)). It reports what the
  build supports, not a measurement; today `metal`, `hip` and `cpu` say `yes` and `nvidia` says `no`.
- **The `index`, `ollama cache` and `dense snapshots` blocks** say what `caldrix` found in its caches on
  disk; `doctor` reads them and makes no request.
- **`model support`** says which model formats Inferno runs on which lanes (quantized GGUF) and where Qwen3.5
  (`Qwen3_5ForCausalLM`) runs: `caldrix` runs it on the `cpu`, `hip` and `metal` lanes ([models.md](models.md#aliases)).
  Then one `images on <lane>:` line per lane, in the lanes' order: `yes` where a model that takes images
  runs on that lane (today SmolVLM, `smolvlm-256m`, on `cpu`, `hip` and `metal`: 64 prompt tokens for each 512-pixel tile), else
  `no` ([run.md](run.md#images)). Then one `image generation on <lane>:` line per lane: `yes` on a lane
  `--backend auto` may choose to make images from a prompt (today FLUX.2 klein 4B, the `flux2-klein-4b`
  alias or a pipeline directory, on `cpu`, as verified at the desk on that lane), else `no` (a lane
  `--backend` admits by flag alone, `metal` for klein, prints `no`) ([run.md](run.md#generating-an-image)). Like the rest of the block, it says what this build supports,
  not what was probed on this machine.
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

No NVIDIA GPU is supported: the line says `no` on every machine, and `--backend nvidia` is refused with the
same reason before anything is downloaded. What the line names is what `doctor` read:

- Linux. On any other system: `the NVIDIA lane exists on Linux only`.
- The NVIDIA driver library. When it does not load, the line reads `no NVIDIA driver library` and the
  loader's message after `detail:`. `INFERNO_NVIDIA_DRIVER` names the library by absolute path instead; a
  relative value is refused by name and nothing is loaded.
- When the driver loads and sees no card: `the NVIDIA driver reports no device`. When a driver call fails:
  `the NVIDIA driver failed`, with the failed call's status and name after `detail:`.
- When it sees a card, the line names it, its compute capability and why that capability is not admitted
  (no NVIDIA card has a recorded run), then the driver version and the card's memory.

`caldrix doctor` and `--backend nvidia` load the driver library to read the card; `--backend auto` and the
other lanes do not, and `auto`'s refusal lists the lane as `no NVIDIA card is admitted yet`.

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
- A C compiler, `cc`, on `PATH`: the lane links its kernels with it when the model loads.
- When usable, the line names the instruction-set row, the physical cores (and the cgroup CPU quota
  when one is set, as in a container), where `cc` is, and the libm version.

## Exit status

| Status | When |
| --- | --- |
| `0` | at least one lane is usable |
| `1` | no lane is usable |
| `2` | an argument was given; `caldrix doctor` takes none |

## Next

A lane that says `no` names what is missing; [troubleshooting.md](troubleshooting.md) has the fixes. With a
usable lane, ask a model something: [run.md](run.md).
