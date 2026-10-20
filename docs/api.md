# HTTP API

`caldrix serve` speaks the OpenAI Chat Completions and Completions APIs on `http://localhost:8425`, with
no API key, and serves one model. Generation is greedy unless a request, or the server's flags, asks for
[sampling](#sampling); a feature field the server does not implement is refused with a `400` that names
it, never ignored.

## Base URL

`http://localhost:8425/v1`; `--port` and `--host` change it ([serve.md](serve.md)). Any OpenAI client works
with its base URL set to it. There is no API key.

## `POST /v1/chat/completions`

Request and response shapes follow the OpenAI Chat Completions API.

```sh
curl http://localhost:8425/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3-0.6B", "messages": [{"role": "user", "content": "What is the capital of France?"}], "max_tokens": 16}'
```

```text
{"choices":[{"finish_reason":"length","index":0,"message":{"content":"<think>\nOkay, the user is asking for the capital of France. I know","role":"assistant"}}],"created":1792079067,"id":"chatcmpl-1","model":"Qwen/Qwen3-0.6B","object":"chat.completion","usage":{"completion_tokens":16,"prompt_tokens":15,"total_tokens":31}}
```

- `model` must be the name the server answers to: for `caldrix serve qwen3-0.6b` that is the repo id,
  `Qwen/Qwen3-0.6B`; for a directory, its name; `--served-name` sets it. Any other name is a `400`.
- `messages` holds 1 to 256 messages and is rendered through the model's chat template. A model without a
  chat template (a base model such as SmolLM2-135M) refuses `messages`; use `/v1/completions`.
- `max_tokens` (or `max_completion_tokens`) caps the answer, at least `1`; without it, generation runs
  until the end-of-sequence token or until the context is full.
- `stop` takes one string or up to four, each 1 to 256 bytes.
- `prompt` is refused here; send `messages`.

## Streaming

`"stream": true` sends server-sent events, a chunk as text is decoded, then `data: [DONE]`.
`"stream_options": {"include_usage": true}` adds a final usage chunk.

```sh
curl -N http://localhost:8425/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3-0.6B", "messages": [{"role": "user", "content": "What is the capital of France?"}], "max_tokens": 2, "stream": true}'
```

```text
data: {"choices":[{"delta":{"content":"<think>"},"finish_reason":null,"index":0}],"created":1792079068,"id":"chatcmpl-2","model":"Qwen/Qwen3-0.6B","object":"chat.completion.chunk"}

data: {"choices":[{"delta":{"content":"\n"},"finish_reason":null,"index":0}],"created":1792079068,"id":"chatcmpl-2","model":"Qwen/Qwen3-0.6B","object":"chat.completion.chunk"}

data: {"choices":[{"delta":{},"finish_reason":"length","index":0}],"created":1792079068,"id":"chatcmpl-2","model":"Qwen/Qwen3-0.6B","object":"chat.completion.chunk"}

data: [DONE]
```

## `POST /v1/completions`

The OpenAI Completions API. `prompt` is a string or an array of token ids; `messages` is refused here.

```sh
curl http://localhost:8425/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3-0.6B", "prompt": "The capital of France is", "max_tokens": 8}'
```

```text
{"choices":[{"finish_reason":"length","index":0,"text":" Paris. The capital of Italy is Rome"}],"created":1792079069,"id":"cmpl-3","model":"Qwen/Qwen3-0.6B","object":"text_completion","usage":{"completion_tokens":8,"prompt_tokens":5,"total_tokens":13}}
```

`"ignore_eos": true` treats the end-of-sequence token as an ordinary token, so generation runs on to
`max_tokens` unless a `stop` sequence ends it. `model`, `max_tokens`, `stop` and streaming are as for chat.

## Fields on both endpoints

- `"return_token_ids": true` adds each generated token's id to the response (`token_ids` on a choice, or on
  each streamed chunk). It defaults to `false`.
- `seed` and `user` are accepted, and so is any field this page does not name.

## `GET /v1/models`

The one model this server serves:

```sh
curl http://localhost:8425/v1/models
```

```text
{"data":[{"id":"Qwen/Qwen3-0.6B","object":"model","owned_by":"inferno"}],"object":"list"}
```

## `GET /healthz`, `GET /metrics`, `GET /debug/requests`

`/healthz` answers `ok` (so does `/health`). `/metrics` is Prometheus text. `/debug/requests` is the
telemetry of the most recent requests, as JSON; it carries token counts, never text.

## Sampling

Six fields are honoured on both endpoints. Each overrides the server's default for that request when it
is present; absent or `null` inherits the default, which is greedy unless `caldrix serve` was started
with the matching flag ([serve.md](serve.md#flags)). The knobs compose: a request's `temperature: 0`
against a server started with `--repetition-penalty 1.2` keeps the penalty.

| Field | Accepted | Default |
| --- | --- | --- |
| `temperature` | a finite number `>= 0`; `0` is greedy | `0` |
| `top_k` | an integer `>= 0`; `0`, `-1` or `null` keeps every token | `0` |
| `top_p` | `0 < top_p <= 1`; `1` keeps every token | `1` |
| `min_p` | `0 <= min_p <= 1`; `0` is off | `0` |
| `repetition_penalty` | a finite number `> 0`; `1` is off | `1` |
| `seed` | a non-negative integer; `-1` or `null` draws a fresh seed | fresh entropy per request |

A value of the wrong type or out of range is a `400` naming the field:

```sh
curl http://localhost:8425/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3-0.6B", "messages": [{"role": "user", "content": "hi"}], "top_p": 2}'
```

```text
{"error":{"code":null,"message":"must be 0 < p <= 1, got 2","param":"top_p","type":"invalid_request_error"}}
```

The knobs apply in the order [run.md](run.md#sampling) gives: the penalty, then top-k, top-p and min-p,
then the temperature and one random draw. Where the token is picked depends on the lane. On the AMD lane
with batching on (`--max-batch 8`, its default there, for `caldrix run` as for `caldrix serve`), the GPU
picks each sampled token. It keeps every token tied with (or within rounding of) the last one a top-k
or top-p cut keeps, walks the candidates in vocabulary order for the draw, and adds in 32-bit floats,
so the same seed can give a different reply than a pick on the CPU. `--sampler host` moves that pick
back to the CPU ([serve.md](serve.md#flags)). Every other lane, and `--max-batch 1`, always picks on
the CPU, most likely token first, exactly as described in [run.md](run.md#sampling).

The record of a sampled request in `/debug/requests` carries its settings and the seed it used, so a
reply can be reproduced on the same machine and build, with the same `--sampler`.

## Not implemented

Any other sampling or feature field is refused with a `400` that names it. Each field below is accepted
only when it is absent, `null` or at the neutral value shown:

- `n`: `1`
- `presence_penalty`, `frequency_penalty`, `top_logprobs`: `0`
- `best_of`: `1`
- `logprobs`: `false` or `0`
- `logit_bias`: `{}`; `tools`: `[]`; `tool_choice`: `"none"`
- `response_format`: `{"type": "text"}`

## Errors

Error bodies follow the OpenAI shape, `{"error": {"message", "type", "param", "code"}}`, with `param`
naming the field at fault.

| Status | When |
| --- | --- |
| `400` | a malformed request, an unknown model, a prompt too long for the context, or a field the server does not implement (named in the body) |
| `413` | the request body is over 4 MiB |
| `429` | the request queue is full (`--queue-depth`) |
| `500` | the engine failed during the request; on a streamed request the `200` has already been sent, so the failure arrives instead as a server-sent event carrying the same error object |
| `503` | the engine has stopped |

Every exchange on this page was captured from `caldrix serve qwen3-0.6b --backend cpu` on a Linux x86-64
machine.

Not implemented: an Anthropic-compatible endpoint.
