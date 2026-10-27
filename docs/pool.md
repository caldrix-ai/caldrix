# caldrix pool

`caldrix pool` is a daemon on a Unix socket that holds one or more loaded models, its members, and serves
them to local clients over a framed protocol. [Aerix](https://caldrix.ai/aerix/docs) uses it to keep a model resident between
sessions. For an HTTP API on a port, use `caldrix serve` ([serve.md](serve.md)).

## Usage

```text
caldrix pool [--socket PATH] [--idle-timeout SECS] [--frames cbor|json] [--context-budget BYTES]
    [--load <model> [serving flags]]
caldrix pool stop [--socket PATH]
caldrix pool --protocol-version
```

The socket defaults to `$XDG_RUNTIME_DIR/caldrix/pool.sock`. The pool exits after `--idle-timeout` seconds
(1800; 0 means never) with no connection open, or on `caldrix pool stop`. `--protocol-version` prints the
protocol version the pool speaks.

## Members

`--load` takes the model and then the serving flags for it, as `caldrix run` does ([run.md](run.md)), and
repeats, one member each. The pool's own flags come before the first `--load`; after it, only another
`--load` ends a group.

```sh
caldrix pool --idle-timeout 600 --load qwen3-0.6b --backend cpu
```

A member is loaded only if the sum of the members' memory footprints, plus one context budget per member,
fits the limit `caldrix run`'s fit uses. A member whose footprint is unknown, such as one with streamed
weights, is the only one.

`--context-budget` caps the KV prefixes a member keeps between generations. It defaults to a quarter of the
memory limit, at most 8 GiB; 0 keeps none.

## Where it has run

`caldrix pool` has been run on the CPU lane, and on Metal on one Apple M1 Max at max context 2048, and one
request at 8192. It has not been run on AMD GPUs.
