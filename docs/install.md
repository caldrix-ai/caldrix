# Install caldrix

One command installs `caldrix` and the linker it uses, `ld.lld`, into `~/.local/bin`, verifying both
checksums before anything is written and never using `sudo`. The installer runs on Linux x86_64 with glibc
2.38 or newer and on macOS 14.0 or newer on Apple silicon; the hardware each lane is supported on is listed in
[index.md](index.md#supported-hardware-and-models).

## Install

```sh
curl -fsSL https://caldrix.ai/install.sh | sh
```

The script asks GitHub for the latest release of `caldrix-ai/caldrix`, downloads the archive for this
platform, verifies it, installs the two binaries, then runs `caldrix doctor` so you see which lanes this
machine can use ([doctor.md](doctor.md)). Releases and their notes are at
[github.com/caldrix-ai/caldrix/releases](https://github.com/caldrix-ai/caldrix/releases); the macOS
archive is published from v0.1.1.

The whole script is inside one function called on its last line, so a download cut short runs nothing.

## Choose the directory

`--prefix DIR` installs into `DIR` instead of `~/.local/bin`:

```sh
curl -fsSL https://caldrix.ai/install.sh | sh -s -- --prefix "$HOME/bin"
```

The script creates the directory if needed and never uses `sudo`; pick a directory you can write to. It
takes no other option. When the directory is not on your `PATH`, the script prints
`add it to your PATH:` and the `export PATH="DIR:$PATH"` line to paste, `DIR` being the install directory.

## Platforms

| `uname -s` and `uname -m` | Archive |
| --- | --- |
| `Linux x86_64` | `caldrix-<version>-x86_64-linux.tar.gz` |
| `Darwin arm64` | `caldrix-<version>-aarch64-darwin.tar.gz` |

Any other platform is refused by name. On Linux the script reads the glibc version from `ldd --version`
and refuses a host below 2.38, or a host whose `ldd` reports no glibc (musl, for example). The refusal
names the container route instead: images `ghcr.io/caldrix-ai/caldrix:latest-rocm6.4` (AMD) and
`ghcr.io/caldrix-ai/caldrix:latest-cpu`. Those images are not published yet. On macOS the script reads
`sw_vers -productVersion` and refuses a Mac below macOS 14.0 before downloading anything.

## What is checked

Each release ships `caldrix-<version>-<platform>.tar.gz` and its `.sha256` file. The archive holds three
files: `caldrix`, `ld.lld` and `SHA256SUMS`, the checksums of the two binaries.

1. The archive's checksum is checked against its `.sha256` file.
2. After unpacking, both binaries are checked against `SHA256SUMS`.

If either check fails, the script stops before writing anything to the install directory. Checksums are
computed with `sha256sum`, or `shasum -a 256` where that is missing (macOS); with neither, the script refuses.

Each binary is copied beside its destination and renamed over it, so reinstalling while `caldrix` is
running works.

## Install with mise

In a project managed by [mise](https://mise.jdx.dev), add caldrix to `mise.toml`:

```toml
[tools]
"github:caldrix-ai/caldrix" = "latest"
```

or pin a release, such as `"0.1.1"`. `mise install` downloads this platform's release archive from GitHub
and puts `caldrix` and `ld.lld` on the project's `PATH`; `mise lock` records each archive's checksum in
`mise.lock`. mise does not run the installer's platform checks: on Linux below glibc 2.38, or a Mac below
macOS 14.0, `caldrix` fails when it starts instead of being refused before the download.

If the project already has a different `ld.lld` on its `PATH` that ours would shadow, expose only
`caldrix`. It still finds its own `ld.lld` in mise's install directory:

```toml
[tools]
"github:caldrix-ai/caldrix" = { version = "latest", filter_bins = "caldrix" }
```

## Uninstall

Remove the two binaries from the install directory:

```sh
rm ~/.local/bin/caldrix ~/.local/bin/ld.lld
```

Downloaded models stay in the Hugging Face cache ([models.md](models.md)).
