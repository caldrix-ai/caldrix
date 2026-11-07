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
- A message's `content` is a string or an array of parts in the OpenAI shape, `{"type": "text", "text": …}`
  and `{"type": "image_url", "image_url": {"url": …}}`; text parts are joined with a newline before the
  template. This server's models take text only: an image part is a `400` naming it
  (`messages[i].content[j]`), as is a part of another type or a malformed one. An image URL must be
  `data:image/png;base64,…` or `data:image/jpeg;base64,…`; remote URLs are not fetched.
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

### Speculative decoding

A server started with `--speculate draft|ngram|auto` decodes single-stream with a proposer and verifies its
guesses against the loaded model; greedy output is the same tokens plain decoding gives. The flags:

- `--speculate off|draft|ngram|auto` (default `off`); `--draft <dir>` names the draft model;
  `--draft-tokens <1..=8>` fixes the draft length (absent: tuned live); `--ngram <min>:<max>` sets the
  lookup's suffix lengths (default `2:4`); `--speculation-memory <path>` keeps the tuner's state between runs.
- An explicit mode is refused at start by name when the server is batched (`restart with --max-batch 1`), on a
  backend that cannot verify a draft (`cpu` and `hip` can), or with `--weight-budget`; `auto` turns itself off with a note.
- Under `auto`, a draft that fails to load gives way to n-gram lookup with a note, and `--speculation-memory`
  is then neither read nor written, since it holds the draft's state.
- A sampled request carrying a seed (its own `seed`, or the server's `--seed`) and a request decoded under a
  constraint mask run without speculation.

When a request speculated, its `usage` gains `"speculative": {"proposed": <n>, "accepted": <n>}`, the draft
tokens proposed and accepted. No speed is claimed here.

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

## `POST /v1/systemone`

Typed decisions about a `state`, in the System One format: each question is a `choice`, a `score` or a
`noul` (yes or no), and each answer comes with a probability for every allowed answer. No text is
generated. Each question becomes one prompt through the model's chat template, with thinking turned
off, and its answer is read from the model's next-token logits over one label per option: `A` to `Z` for
`choice` and `score`, `Yes` and `No` for `noul`. The probabilities are a softmax over those labels alone.

```sh
curl http://localhost:8425/v1/systemone \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3-0.6B", "state": "Customer message: I was charged twice for my order last week and I want one of the payments back.", "questions": {"team": {"type": "choice", "instructions": "Which team should handle this message?", "criteria": {"billing": "payments, refunds and invoices", "shipping": "deliveries, tracking and lost parcels", "technical": "bugs, crashes and login problems"}}, "angry": {"type": "noul", "instructions": "Is the customer angry?"}, "urgency": {"type": "score", "instructions": "How urgent is the message?", "criteria": ["not urgent", "somewhat urgent", "very urgent"]}}}'
```

```text
{"model":"Qwen/Qwen3-0.6B","answers":{"team":{"type":"choice","choice":"billing","probabilities":{"billing":0.9976708292961121,"shipping":0.0021615091245621443,"technical":0.00016766643966548145},"confidence":0.9965062439441681},"angry":{"type":"noul","noul":0.981803297996521},"urgency":{"type":"score","score":0.10968978330492973,"legend":{"0":"not urgent","1":"somewhat urgent","2":"very urgent"},"probabilities":{"0":0.9140376448631287,"1":0.06223498284816742,"2":0.023727400228381157},"confidence":0.8354653250426054}},"usage":{"input_tokens":204,"output_tokens":0}}
```

Answers are read zero-shot from the loaded chat model and are only as good as that model. The answers
above are this small model's own. A trained decision model answers in its own format instead
([the decider reading](#the-decider-reading)).

The request:

- `model` is the served name, as for chat. `state` is a string, an object or an array.
- `questions` maps 1 to 64 names to questions; answers come back under the same names, in the same
  order.
- `choice`: `criteria` maps 1 to 26 option names to a description (a string, an object, an array or
  `null`); the option order sets the labels `A`, `B`, ….
- `score`: `criteria` is an array of 1 to 26 levels (each a string, an object or an array).
- `noul`: `criteria` is optional, `{"true": …, "false": …}`, each a string, an object, an array or `null`.
- `instructions` is optional on every question: a string, an object, an array or `null`.
- An object or array in `state`, `instructions`, a description or a level is written into the prompt as
  compact JSON: keys in the order received, no space after `,` or `:`. A server that writes it with
  spaces sends the model a different prompt, so its probabilities can differ from these. Numbers are
  written in one canonical form, not as the request spelled them: an integer exactly, at any size; any number
  with a `.`, `e` or `E` as the shortest decimal that reads back to the same 64-bit float, positional for
  exponents from -4 to 15 and otherwise with a signed exponent of at least two digits (`0.00001` is written
  `1e-05`, `1e16` is written `1e+16`); a whole number written this way keeps its fractional part (`2.0`,
  never `2`).

The response:

- `choice`: the chosen option, its `probabilities` by option name and a `confidence`,
  `max(0, (p_best - 1/n) / (1 - 1/n))` (`1` with one option).
- `score`: the expected level `score` (the sum of `i * p_i`), the `legend`, the `probabilities` by level
  index and a `confidence` from the spread around the most likely level (`1` with one level). The `legend`
  echoes each level as the request gave it, and a number inside a level keeps the request's digits:
  `1.50` is echoed `1.50` and `1e-05` is echoed `1e-05`, not `1.5` and `1e-5` as a 64-bit float would
  write them, and an integer is exact at any size.
- `noul`: the probability of yes.
- An exact tie takes the first option. `usage.input_tokens` is the sum of every question's prompt tokens;
  `output_tokens` is `0`.

Refusals are `400`s, each naming the field in `param`: an unknown field at any level, a duplicate key, more
questions, options or levels than the limits above, `assets`, `images`, `chat_template_kwargs`, a question
of `"type": "box"` and a question whose prompt does not fit the context. Three refusals are about the
server or model, not a field, and carry `"param": null`: a model whose tokenizer does not give every label
one token of its own inside the model's vocabulary (the message names the label), a model with no chat
template (`the model has no chat template; /v1/systemone needs one`), and a server started with
`--max-batch` above `1` (`/v1/systemone needs --max-batch 1`), since the route needs the single-stream
engine; `--max-batch 1` is the default everywhere except the AMD lane. Every question is checked before
any is run.

```sh
curl http://localhost:8425/v1/systemone \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen3-0.6B", "state": "s", "questions": {"q": {"type": "box", "criteria": {}}}}'
```

```text
{"error":{"code":null,"message":"box questions are not supported by this server","param":"questions.q.type","type":"invalid_request_error"}}
```

The questions then run one at a time, in order, each through the server's request queue, so other
requests can run between them. A full queue answers `429` (`the request queue is full`) for the whole
request, even when earlier questions of it have already run; their work is discarded and no answer is
returned. `503` (`the engine is unavailable`) means the engine has stopped, and `500` means the engine
failed on a question or gave no usable reading for it.

### The decider reading

A model directory that holds a `decider_config.json` beside its weights is a decider, a model trained to
answer these questions, and `/v1/systemone` reads it in its own trained format instead of the zero-shot one.
`caldrix` fetches the file with the model when the repository has one, as `decider-0.8b`'s does
([models.md](models.md)). The file is read once, when the model loads, and checked strictly: an unknown key,
a missing `temperature` or `neutralize_none`, or a form this release does not build (`neutralize_none:
true`, `schema_first: true`, a chat layout, images) stops the load with a message naming the key; a file
over 64 KiB stops it with a message naming the file and the limit. The request and the response keep the
shape described above; what changes is how each answer is read.

- **The prompt.** No chat template is used. Each prompt is `Context:` and the state, then one question with
  its options and labels, then `Answer: (`, assembled from token ids exactly as decider's own package
  builds them. A state, instructions, description or level that is an object or an array is written as JSON with a space
  after `,` and `:`, and inside the state a list of 8 or more elements carries each element's position as
  `_index`.
- **The labels.** Options are labelled `A` to `Z`, then the two-letter labels `AA`, `AB`, … that are one
  token each in the model's vocabulary, 255 labels in all. A `noul` question reads two options, `no` and `yes`.
- **The temperature.** The probabilities are a softmax of the label logits divided by the config's
  `temperature`.
- **Levels asked one at a time.** When the config sets `isolated_levels`, each level of a `score` question is
  asked as its own yes-or-no question about that level (a leading level number such as `2: ` is left out of
  it), and level `i`'s probability is its yes probability over the sum of all the levels' yes probabilities.
  Otherwise all levels are options of one question.
- **The answer.** `choice`, `score`, `noul` and `confidence` are computed as above. The values are not
  rounded (decider's own package rounds them), and `model` is the served name.

The limits differ from the zero-shot reading. The refusals of these four start with
`this model is a decider: `:

- `questions`: 1 to 64.
- `choice`: 2 options up to the config's `max_options` (at most 255).
- `score`: 2 to 10 levels.
- `instructions` is required on `choice` and `score` questions and may not be `null` or `""`; a `noul` question
  without instructions needs `criteria` describing at least one of `true` and `false`.

These size limits refuse with their own messages, which name the counts and carry no such prefix:

- **The state cap.** A state longer than the config's `max_state_tokens` (32768 when the config does not
  set it) is a `400` naming both counts. A state whose text is too long to fit the cap even at the
  vocabulary's longest token per token is refused before it is tokenized, and its count is then a lower
  bound (`the state is at least … tokens`) for text the tokenizer's normalization does not shrink. decider's own package cuts such a state short without saying
  so; this server refuses it instead, so an answer is never read from part of the state.
- Every prompt must leave room for the one read step under the compiled context, and the prompts of one
  request together must not exceed 64 times the compiled context.

The state is read once per request. The first prompt also keeps the model's state at the end of the state's
tokens, and every later prompt of the request starts from that point, so only its question is read again.
`usage.input_tokens` counts the tokens actually read: the state once plus each prompt's question. A request
with one prompt, or one whose kept state could not be taken, reads every prompt whole and counts every
prompt whole. The kept state belongs to the request and is dropped with the response.

The prompts run one at a time, in order, each through the server's request queue, as the questions do
above: other requests can run between them, and a full queue answers `429` for the whole request. A `500`
also means the reading gave a probability that is not a finite number.

`decider-0.8b` is a Qwen3.5 model, which this server runs on the CPU lane and on `--backend host` (its off-hardware test stand-in) only ([models.md](models.md)),
with `--max-batch 1` as above.

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
