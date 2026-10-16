# caldrix doctor

`caldrix doctor` reports, for each of the three lanes (Apple GPU through Metal, AMD GPU through HIP, the
CPU), whether this machine can use it and why, then what Inferno does not run. It reports only what it
probed, launches no kernel, and exits with status 1 when no lane is usable.

## Run it

```sh
caldrix doctor
```

On a Linux x86-64 machine without an AMD GPU, the v0.1.0 release installed with `install.sh`
printed:

```text
caldrix 0.1.0 (1b2f29dd1c13, x86_64-linux)

lanes
  metal  no      the Metal lane exists on macOS only
  hip    no      the HIP runtime did not load: dlopen libamdhip64.so: libamdhip64.so: cannot open shared object file: No such file or directory
  cpu    usable  x86-64-v3 (12 physical cores, cgroup quota 8); cc at /usr/bin/cc; libm glibc 2.41

not supported
  Inferno does not run MoE architectures, quantized checkpoints or NVIDIA GPUs today

no kernel was launched; `caldrix run` is what shows a lane works
```

## Reading the report

- **The first line** is `caldrix --version`: the version, the commit the binary was built from
  (`unknown commit` for a build outside the release script) and the platform.
- **`lanes`** has one line per lane, in the order `--backend auto` tries them: `metal`, `hip`, `cpu`.
  Each says `usable` with what was found, or `no` with the reason. `caldrix run` and `caldrix serve`
  with `--backend auto` (the default) take the first usable lane.
- **`not supported`** always lists what Inferno does not run. When a device under `/sys/class/drm`
  reports NVIDIA's vendor id, an extra line says an NVIDIA device is present and not supported.
- **The last line** is a reminder: the report comes from probes, and only `caldrix run` shows a lane
  actually works.

## What each lane needs

### `metal`

- macOS. On any other system the line reads `the Metal lane exists on macOS only`.
- A binary built with Metal support; the macOS release archive is. Otherwise the line says the binary was
  built without the `metal` feature.
- A Metal device. When the device query fails, the line reads `no Metal device:` and the error.
- When usable, the line names the device. The working-set limit is not probed.

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
