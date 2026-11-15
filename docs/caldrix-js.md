# caldrix.js

caldrix.js runs a language model in the browser, in a worker, on WebGPU or WebAssembly. It is one npm
package, `@caldrix-ai/caldrix`: the engine's compiled browser host (WebAssembly), the worker that drives
it, and the typed API around them. The `caldrix` binary the rest of these docs describe is not involved.

To run an agent with tools on the engine, see [aerix.js](aerix-js.md).

This page states only what the browser-gate record (kept in the private Caldrix repository) holds, what
the package's own tests run in CI, and what the source does.

## Install and quick start

```sh
npm install @caldrix-ai/caldrix
```

```ts
import { Caldrix } from "@caldrix-ai/caldrix";

const engine = await Caldrix.load("smollm2-instruct", {
  onProgress: ({ file, loaded, total }) => console.log(file, loaded, total),
});

for await (const delta of engine.chat([{ role: "user", content: "Say hello." }], { maxTokens: 32 })) {
  if (!delta.done) document.body.append(delta.text);
}

console.log(engine.doctor());
engine.dispose();
```

`Caldrix.load` fetches the model's files from the Hugging Face Hub, starts the engine's worker and
resolves once the model is loaded. Inference runs only in that worker; the page's main thread never
runs the model.

## The API

The package exports `Caldrix`, `Engine`, `probe` and `CaldrixError`, and the types the signatures
below name. As the package's declarations have them:

```ts
class Caldrix {
  static load(model: string, opts?: LoadOptions): Promise<Engine>;
  static check(model: string, opts?: CheckOptions): Promise<CheckResult>;
  static fit(facts: QuantFacts, opts?: FitOptions): Promise<FitResult>;
}
class Engine {
  chat(messages: ChatMessage[], opts?: GenerateOptions): AsyncIterable<Delta>;
  complete(input: string | { ids: number[] }, opts?: GenerateOptions): AsyncIterable<Delta>;
  doctor(): DoctorReport;
  dispose(): void;
}
function probe(): Promise<Capabilities>;
```

`model` is an alias this build ships (`doctor().aliases`) or a Hugging Face repo id, `owner/name` or `owner/name@<revision>`. A repo id is resolved once through the Hub's model API to its canonical id and commit sha, and every file is fetched and cached at that sha; `doctor().model` and `doctor().revision` are the canonical id and the sha, and the doctor line says `(resolved from <id>)` when the canonical id differs from the one asked (a rename, or a different letter case). The browser build sends no token: a gated repo is refused by name, a private one reads as not found (the Hub answers it as it answers a missing repo), and the caldrix CLI, which reads `HF_TOKEN`, loads both. A repo id loads a dense Llama-family checkpoint, a bitsandbytes nf4 checkpoint (dequantized to the plan dtype at import: it runs at its dense size, not its download size; `doctor().note` says both), a PEFT LoRA adapter repo (its base, named by `adapter_config.json`, resolved once through the Hub's model API, and the adapter folded into it tensor by tensor at import), or a LoRA adapter over an nf4 base. `doctor().provenance` is the caldrix CLI's `model:` line, naming both repos and shas; progress events name the repo each file comes from. Folded weights are never cached: a repeat load fetches nothing new and folds again. A gated base, an adapter over an adapter, a pickle-only adapter and a repo carrying both a model and an adapter are refused by name before any weight byte.

- **`chat`** formats the messages with the model's chat template; **`complete`** continues a plain-text
  prompt, or a prompt given as token ids. Both yield deltas until one has `done: true`. When a model's
  chat template refuses a system message, `chat` puts the system text at the top of the first user
  message instead, and the last delta has `systemFolded: true`.
- **`doctor()`** reports the load: the tier that runs (`backend`), `device`, `dtype`, `storage`,
  `adapter`, `maxContext` and `contextClamp`, where each model file came from and whether
  its digest was checked (`source`), where the package's own files came from (`assets`: `bundle`, `cdn`
  or `page`, with the URL), `aliases` (every alias this build loads, each with its `name` and its
  `format`, `safetensors` or `gguf`), `provenance` (a densified load's `model:` line, `null` for a model read as it lies), `note` (an nf4 repo's 4-bit note, `null` otherwise), `images` (whether the tier takes image parts; see [Images](#images)), and `why` the tier was chosen.
- **`dispose()`** releases the worker and everything in it. Every later call throws `request`.
- **`probe()`** reports what this page can run, without loading anything: the WebAssembly features, the
  WebGPU adapter if there is one, which tiers are available, the tier `"auto"` would choose (`chosen`)
  and `why`.

```ts
type ChatMessage = { role: "system" | "user" | "assistant"; content: string | ContentPart[] };

interface Delta {
  text: string;
  token?: number;
  done: boolean;
  finishReason?: "stop" | "length" | "abort";
  usage?: { promptTokens: number; completionTokens: number; cachedTokens: number };
  logprobs?: { token: number; logprob: number }[];
  systemFolded?: true;
}

interface GenerationOptions {
  constraint?: { json: Record<string, unknown> } | { oneOf: string[] };
  logprobs?: number;
  temperature?: number;
  topK?: number;
  topP?: number;
  minP?: number;
  repetitionPenalty?: number;
  seed?: number;
  maxTokens?: number;
}

type GenerateOptions = GenerationOptions & {
  signal?: AbortSignal;
  onImageProgress?: (p: { tilesDone: number; tilesTotal: number }) => void;
};
```

Generation is greedy unless a sampling option is set. Aborting the `signal` ends the generation with
`finishReason: "abort"`. `constraint` limits what the model may write, on both tiers: `json` takes a
JSON Schema subset and `oneOf` a list of strings, and a schema outside the subset is refused with
`kind: "request"`; the package README's "Constrained decoding" section states the subset and what has
run under it.

A generation starts from the context the previous one on the same engine left, when the new prompt
begins with the same tokens; `usage.cachedTokens` on the last delta says how many prompt tokens that
reused. `logprobs: n` (at most 20, otherwise refused with `kind: "request"`) adds up to `n` candidates (fewer under a constraint)
as `{ token, logprob }` to each token's delta as its `logprobs`.

### Errors

The package's errors are `CaldrixError`s, whose `kind` says which stage refused:

```ts
type Kind = "probe" | "fit" | "architecture" | "fetch" | "digest" | "validate" | "lowering" | "backend" | "request";
```

## Images

This section describes caldrix.js 0.18.0, which is not yet published: the published version takes
images on the wasm tier only.

A quantized Qwen3.5 GGUF file loads by its Hub repo id and file name on the WebAssembly tier only
(`Caldrix.load("unsloth/Qwen3.5-0.8B-GGUF", { file: "Qwen3.5-0.8B-Q6_K.gguf" })`; there is no alias for it).
The WebGPU tier has none of this model's kernels: `backend: "auto"` uses the WebAssembly tier and
`backend: "webgpu"` is refused by name, as is a safetensors Qwen3.5 checkpoint on both tiers. Checked once, in
headless Chromium on a CPU, on that one Q6_K file: its reference's eight tokens, at a 64-row context and at the
default context, which is lowered to fit the browser's memory (the browser-gate record, entry 68). No device, no
other file and no speed has been measured.

`smolvlm-256m` (SmolVLM-256M-Instruct, a vision-language model) runs in the browser on both tiers:
`chat` takes images as well as text, and `backend: "auto"` no longer falls back to the wasm tier for it.
On the wasm tier it has run in Playwright's headless Chromium on Linux (the browser-gate record, entries 59
and 60); on the WebGPU tier on the operator's Apple M1 Max in Playwright's Chromium, on the Mac's own
adapter, vendor `apple`, architecture `metal-3` (entry 63). Both runs were on a test build of the browser
host that admitted the model on that tier ahead of the version that ships it; see
[What has run, and where](#what-has-run-and-where).

A message's `content` is a string or an array of parts:

```ts
type ContentPart =
  | { type: "text"; text: string }
  | { type: "image_url"; image_url: { url: string | Uint8Array; detail?: "low" | "high" | "auto" } };
```

An image is a PNG or a JPEG: a `data:` URL string (`data:image/png;base64,`, `data:image/jpeg;base64,`
or `data:image/jpg;base64,`) or the image file's own bytes as a `Uint8Array`; the engine never fetches a
URL. A WebP is refused by name in this build, as a URL or as bytes. `doctor().images` says whether the tier takes image parts, in the caldrix
CLI's words: `"yes (SmolVLM, 64 tokens per 512-px tile)"` on both tiers. On a
text-only model an image part is a `request` refusal.

`onImageProgress` is called on the page after each image tile is encoded, before the prefill; a throw
from it aborts the call as `signal` would, then, once the call has ended, the call rejects with the
callback's own error (not a `CaldrixError`) and the engine stays usable. A call aborted while its images are being encoded ends with
one `done` delta, `finishReason: "abort"`, `usage.completionTokens` 0 and no `token`; the engine stays
usable.

A malformed image request is a `CaldrixError` of kind `request`, in the caldrix server's words (`i` the
message, `j` the part), and leaves the engine usable:

| Input | Message |
|---|---|
| a remote URL | `messages[i].content[j].image_url.url: remote URLs are not fetched; send a data: URL` |
| a WebP (a `data:image/webp` URL, or a WebP's bytes under any label or as a `Uint8Array`) | `messages[i].content[j].image_url.url: WebP images are not accepted in this build; send a PNG or a JPEG` |
| another media type | `messages[i].content[j].image_url.url: unsupported media type; send data:image/png;base64, data:image/jpeg;base64 or data:image/jpg;base64` |
| a `detail` outside the three | `messages[i].content[j].image_url.detail: must be "auto", "high" or "low"` |
| more than 8 images | `messages: a request may carry at most 8 images; this one has {n}` |
| bytes over the limit (8 MiB) | `messages[i].content[j].image_url.url: the image is {n} bytes, over the limit of {limit} bytes` |
| bytes that are not a PNG or JPEG | `messages[i].content[j].image_url.url: unsupported media type: the bytes are not a PNG or JPEG` |
| a `Blob` or any other object as `url` | `messages[i].content[j].image_url.url: must be a string or a Uint8Array` |

## Options

```ts
interface LoadOptions {
  backend?: "auto" | "webgpu" | "wasm" | "webgl";
  offline?: boolean;
  onProgress?: (p: { file: string; loaded: number; total: number | null; repo: string; revision: string; provenance?: string }) => void;
  hubBaseUrl?: string;
  file?: string;
  dtype?: "f32" | "bf16";
  maxContext?: number;
  memory?: "single" | "split";
  weightMemoryBytes?: number;
  storage?: "auto" | "f32";
  signal?: AbortSignal;
  assets?: { worker?: string | URL; hostWasm?: string | URL; fallback?: boolean; cdn?: string | URL };
}
```

- **`backend`**: `"auto"` (the default) probes the browser and picks WebGPU when an adapter is present
  and `requestDevice` succeeds, wasm otherwise; a bf16 plan goes to wasm on the page, before the worker
  starts. `"webgpu"` and `"wasm"` force a tier. `"webgl"` is declared and refused: the backend is not
  built. Under `"auto"`, a load WebGPU then refuses (a larger explicit context, a limit, a device that
  fails to open) is served by the wasm tier, and `doctor().why` says so. A quantized GGUF alias under
  `"auto"` is placed as any alias: WebGPU is probed first, and a refusal of the same kinds falls back to
  wasm.
- **Quantized aliases**: `smollm2-instruct:q4_K_M` and `smollm2-instruct:q8_0` are one GGUF file each. A
  load reads the file's header with a range request first (and `rope_freqs.weight`'s small extent as a second
  range when it lies past the header), then verifies the whole file against the
  alias's pinned sha256 before parsing it. Both tiers have their kernels; on the WebGPU tier the weights
  stay in their blocks and `doctor().storage` is `"blocks"`. What has run with them is under "What has run, and where" below.
- **`onProgress`**: called with each model file's progress: the file, the bytes loaded and the total
  (`null` when the size is not known), `repo` and `revision` (the repo and commit it is fetched from: for an adapter, its own or its base's) and, once the tier that serves is settled, `provenance` (the `model:` line of a densified load).
- **`dtype`**: the plan's dtype, `"f32"` or `"bf16"`.
- **`storage`**: the WebGPU tier's weight storage. `"auto"` stores the block GEMM weights as f16 when the
  adapter has `shader-f16`; `"f32"` keeps them f32 on any adapter. `storage: "f32"` on a quantized alias
  is satisfied: nothing is narrowed, and the weights stay in their blocks. `doctor().storage` reports it
  (`"blocks"` for a quantized alias, `null` on wasm). `storage: "f32"` with `backend: "wasm"` is refused.
- **`maxContext`**: the context length to load at; see [the default context](#the-default-context).
- **`memory`**: where the wasm tier places a GGUF file's weights. Absent (the default), they go in the one
  memory whenever the model fits it; a single GGUF file that does not fit one memory has its weights placed in
  several memories (up to four) when the browser has multi-memory, and is refused before its body is downloaded when it does not (the message
  says the browser lacks multi-memory when the model's total is what does not fit). `"single"` asks for the one memory. `"split"` places the weights in
  several memories beside it, created before the file's body is requested; it needs a browser with
  multi-memory, and a weight memory the browser cannot create is a `fit` error naming it. Only a single
  GGUF file on the wasm tier can be split: any other load given `"split"` is a `request` error naming it;
  under `backend: "auto"` the WebGPU tier's refusal moves the load to the wasm tier, as its other refusals do (this ran
  on a software adapter, not a GPU). A split load's `doctor().line` ends with `, memory: split, ` and the number of weight memories; a one-memory
  load's line is unchanged. Placing weights in several memories has run in headless Chromium on a small test
  file only (the browser-gate record, entry 71); no real model and no file past 4 GiB has been loaded this way.
- **`weightMemoryBytes`**: for diagnosis, the bytes each weight memory of a `memory: "split"` load may hold
  (a multiple of 64 from 128 bytes to 4 GiB; the default is 4 GiB). Refused without `memory: "split"`.
- **`file`**: the GGUF file to load from a repo id, by its name at the repo's top level (`"model-Q4_K_M.gguf"`). A
  repo whose listing has no `config.json` with safetensors weights, and a GGUF file at its top level, is a GGUF repo, and loading it without `file` is a
  `request` error that names the candidate files (`mmproj*` files and sharded `-NNNNN-of-NNNNN.gguf` files are never
  candidates). The file is fetched at the listing's resolved commit and checked against the listed size and, when the
  listing carries one, its LFS sha256 (else the Hub's `X-Linked-Etag`); the config, tokenizer and chat template come
  from the file's own metadata, and a `config.json` in the repo is ignored. The weights stay in their blocks. A dense
  GGUF (`F32`, `F16`, `BF16`) defaults to an f32 plan. `file` on an alias or a safetensors repo is a `request` error.
- **`hubBaseUrl`**: where the model files and, for a repo id, the Hub listing (`/api/models/…`) are fetched from (default `https://huggingface.co`); a mirror serving only files loads aliases but not repo ids. **`offline`**
  uses only the browser's cache and fails with `fetch` for a file it lacks.
- **`signal`**: cancels the load. Before the model is loaded, an abort terminates the engine's worker
  and the load rejects with `fetch`; after the load resolves the signal is ignored.
- **`assets`**: where the package's own files come from; see [Network](#network).

## Checking a model before loading it

```ts
type CheckOptions = Pick<LoadOptions, "backend" | "dtype" | "maxContext" | "storage" | "hubBaseUrl" | "file" | "signal" | "assets">;
type CheckResult = { ok: true; backend: "webgpu" | "wasm" } | { ok: false; kind: Kind; message: string };

const r = await Caldrix.check("smollm2-instruct");
if (r.ok) console.log(`a load would run on ${r.backend}`);
else if (r.kind !== "fetch") console.log(`a load would be refused: ${r.message}`);
```

`Caldrix.check` says which tier a load of the model would end on in this tab, or why the load would
be refused, without loading it. It takes the same model and the same tier-deciding options as `load`,
and `assets`, so it starts the same worker and `.wasm` the load would, under the same fallback rule;
it runs the same option checks, the same probe and the same tier choice. Its worker then runs the
load only until the tier is settled: for a repo id the Hub listing (for a LoRA adapter repo, its
`adapter_config.json` and its base's Hub listing too), then `config.json` and the safetensors index
(for a GGUF file, an alias's or one named by `file`, the file's header range instead, and
`rope_freqs.weight`'s extent, at most 512 bytes, as a second range when it lies past the header), the same precheck as `load`, and on the WebGPU
tier the device open, after which the device is destroyed and the worker ends. It never fetches a
tokenizer file or a weight, sends no `onProgress` events, and is never `offline`.

Under `backend: "auto"` the answer is the tier the load would end on: a WebGPU refusal of the kinds
`load` falls back on (a limit, a larger explicit context, a device that fails to open) answers
`{ ok: true, backend: "wasm" }`, and a bf16 request answers wasm as `load` places it. An explicit
`"webgpu"` or `"wasm"` answers that tier or its refusal.

`check` never rejects. Every refusal is `{ ok: false, kind, message }`, with the kind and message
`load` would give: an option it refuses (an unknown backend, a `hubBaseUrl` that is not an absolute
http(s) URL, an empty model, `storage: "f32"` with `backend: "wasm"`) is `request`, a model past this
tab's limits is `fit`, and a worker that cannot start or answers out of protocol is `backend`.
`kind: "fetch"` means the answer is unknown: a file could not be fetched, or the check was aborted
through `signal` (`checking <model> was aborted`), so a load might still succeed. The CDN fallback
for the package's own files applies as on a load (off with `assets.fallback: false`), warned once per check on
the console. The files a check fetches are written to the cache, and a later load reuses them.

## Fitting a GGUF file from its recorded facts

```ts
type FitOptions = Pick<CheckOptions, "backend" | "maxContext" | "signal" | "assets">;
type FitResult = { ok: true; backend: "webgpu" | "wasm"; maxContext: number | null; contextClamp: NonNullable<DoctorReport["contextClamp"]> | null } | { ok: false; kind: Kind; message: string };

// `facts` is a `QuantFacts`: what the nightly records for one GGUF file of a repo.
const r = await Caldrix.fit(facts);
if (r.ok) console.log(`a load would run on ${r.backend} at context ${r.maxContext}`);
else if (r.kind !== "fetch") console.log(`a load would be refused: ${r.message}`);
```

`Caldrix.fit` says whether a GGUF file would load in this tab, on which tier and at what context,
from the facts recorded for it (`QuantFacts`: the file's name, size and digest, its architecture and
dimensions, the element count per tensor type, the base model (`base_model`), `general.file_type`, and the tensor table, each
tensor's name, type and dims), and it fetches nothing: no Hub listing, no range and no file. It takes
`check`'s tier and context options and `assets`, runs the same option checks, probe and tier choice,
and in its worker the same precheck a load of the file runs, with the file's header replaced by the
facts; on the WebGPU tier it opens the device and destroys it again, and under `backend: "auto"` a
WebGPU refusal answers wasm as `check`'s does. `{ ok: true }` carries the tier, `maxContext` (the
context a load would plan) and `contextClamp` (the clamp's line when the default context was
clamped, else `null`), as a load's `doctor()` reports them.

A caller holding a larger record (the nightly's `web-quants.json` entry, say) passes only these keys:
an unknown key is refused as `validate`, so a page projects each entry onto them. `fit` answers the
default weight storage for the tier; `FitOptions` has no `storage`, so a page that loads a dense file
with `storage: "f32"` gets `fit`'s answer for the default storage (quantized files are unaffected).

The tensor types `fit` admits are the ones this build runs: a tensor of any other type is
`{ ok: false, kind: "architecture" }`, its name and type in the message. Facts that are not a plain
object, or are past the size a fit takes once serialised, are `request`; facts the host refuses (an
unknown or missing key, a table past the GGUF reader's caps) are `validate`; a file past this tab's
limits is `fit`, and a worker that cannot start or answers out of protocol is `backend`. `fit` never rejects; `kind: "fetch"` (an abort through `signal`, `the fit was
aborted`) leaves the answer unknown.

`fit` judges one memory only: it can refuse a GGUF file that `load` would place in several memories
(see `memory` under "Options"); `check` agrees with a load that names no `memory`.

A facts admit is never a load guarantee. `check` and `load` judge the file's own header at the commit
the listing pins, and the facts cannot show what only the file holds: the values of
`rope_freqs.weight`, a missing `attention.layer_norm_rms_epsilon`, or a tensor whose bytes or element
count disagree with the table. Facts recorded from an older commit of a repo answer for that file.

## What has run, and where

Only these runs are recorded (the browser-gate record: entries 5, 8 and 9 in the browser, and for the
quantized aliases entries 10 and 11 off-browser and entries 12 and 14 in the browser, and for the adapter and
nf4 repos entry 16 off-browser and entry 17 in the browser, and for `Caldrix.check` entry 20 in the browser, and for the Try page entry 21 in the browser, and for a GGUF file in a repo entry 22, and for `Caldrix.fit` entry 24, and for images entries 59, 60 and 63, and for weights in several memories entry 71); nothing else is claimed.

- **Model:** SmolLM2-135M-Instruct, one prompt, 8 greedy tokens; the adapter and nf4 repos below add a
  LoRA adapter over it and SmolLM2-360M-Instruct's nf4 repo, on their fixtures' prompt. No other model or prompt.
- **Browser:** Chromium driven by Playwright, headless only: the headless shell 153.0.8010.12 on Linux
  (entries 5 and 8) and on an Apple M1 Max (entry 9), and on that Mac Playwright's full Chromium (new
  headless mode, userAgent `HeadlessChrome/153.0.0.0`) for the WebGPU tier (entry 9). No other browser.
- **wasm tier** (entry 5 on Linux, entry 9 on the Mac, with the same lines): both plan dtypes, f32 and
  bf16. The tokens equal the references' tokens. The f32 logits stayed inside the 1e-2 bound on every
  row, worst 7.796401e-4. The bf16 result rests on token equality alone; the logits were recorded, not
  gated.
- **WebGPU tier on a real adapter** (entry 9): the Mac's own, vendor `apple`, architecture `metal-3` (its
  own strings; `device` and `description` empty), with `shader-f16`. The f32 plan at f32 weight storage,
  token-exact, logits worst 6.6792103e-4 against the same 1e-2 bound; at f16 weight storage, a
  diagnostic gated by no bound, every row finite, worst 5.0272985e-4, token-exact. The product path's
  tokens equal the test worker's at both storage forms.
- **WebGPU tier on SwiftShader** (entry 8): Dawn on SwiftShader, Google's CPU Vulkan implementation,
  which is a CPU and not a GPU. The f32 plan at f32 weight storage, token-exact, logits worst
  5.874168e-4 against the same bound.
- quantized aliases: off-browser at the desk on both tiers (the browser-gate record, entries 10 and 11); in headless Chromium through `Caldrix.load` on both tiers on the pinned files, WebGPU on Dawn/SwiftShader, a CPU (entry 12); on the operator's Mac through `Caldrix.load` on both tiers, WebGPU on the Mac's own adapter, vendor `apple`, architecture `metal-3`, token-exact (entry 14).
- adapter and nf4 repos by id (a LoRA adapter over SmolLM2-135M-Instruct, and SmolLM2-360M-Instruct's bitsandbytes nf4 repo, each at a pinned sha): off-browser at the desk on both tiers (the browser-gate record, entry 16: wasm both pairs and WebGPU the LoRA pair token-exact, the f32 plan at f32 weight storage; the WebGPU nf4 pair stopped at lavapipe's storage-buffer binding limit); in headless Chromium through `Caldrix.load` by repo id on both tiers, both pairs token-exact, the f32 plan at f32 weight storage, WebGPU on Dawn/SwiftShader, a CPU (entry 17).
- `Caldrix.check`: in headless Chromium against the loopback hub (the browser-gate record, entry 20): on a repo id, check and load agreed on the wasm tier and, with WebGPU enabled in the browser, on the WebGPU tier on Dawn/SwiftShader, a CPU, and the check fetched no tokenizer file and no weight; an index past the wasm ceiling checked as `fit`, an unknown repo id as `fetch`.
- the website's Try page: in Brave on an Apple M1 Max, headed, every model on its list loaded by repo id from the real Hub on the WebGPU tier, answered one chat message and ran the agent's first chip, as the operator reported it (entry 21).
- a GGUF file in a repo by name (`file`): in headless Chromium through `Caldrix.load` against the loopback hub's tiny fixtures, on the wasm tier and on the WebGPU tier on Dawn/SwiftShader, a CPU, token-exact to the test harness's own tokens, with the name selecting the file, a digest mismatch refused as `digest`, and a header context past the wasm32 heap clamped and reported by `doctor`; off-browser, a rope-scaled quantized model token-exact on both tiers and the quantized row kernels past the grid dimension limit on lavapipe, a CPU (entry 22).
- `Caldrix.fit`: in headless Chromium against the loopback hub's tiny fixtures, on the wasm tier and on the WebGPU tier on Dawn/SwiftShader, a CPU: the committed facts of the hub's Q8_0 file and of its long-context copy answered `ok` with the tier, and the `maxContext` and `contextClamp` a load of that file reported in `doctor()`, with no request to the hub; the facts with one tensor's type changed to one this build lacks answered `architecture` naming it; off-browser, the fit from facts equals the fit from the header on every tiny fixture (entry 24).
- images on the WebAssembly tier (the browser-gate record, entries 59 and 60), in headless Chromium on Linux. SmolVLM-256M-Instruct's browser check ran on a test build of the browser host that admitted the model ahead of this version. Its first run stopped where a JPEG's image rows, from the package's own decode of the file, were further from transformers' than the check's bound allows (entry 59). The owner then amended the JPEG check to hold to that bound the rows of the reference's own decode of the file, the real file's tokens still checked, and the second run passed (entry 60): on the test page at f32, four low-detail images, the reference decode and a 13-tile high-detail image gave transformers' tokens, with every checked tile's image rows within the bound, in tile order; at bf16, two low-detail images gave transformers' tokens; through `Caldrix.load` in its worker, the four low-detail images and a `Uint8Array` gave the test page's tokens. On the build the package ships: a tiny test vision model through `Caldrix.load` on the WebAssembly tier, token-exact to transformers' tokens at both details, with tile progress, a `Uint8Array` image, a mirrored image giving other tokens, a second turn, a cancel during the tiles and the request refusals (its tokens do not show the order of the tiles or of the rows within one), and the same model on the WebGPU tier on Dawn/SwiftShader, a CPU, token-exact at both details, `backend: "auto"` serving it there.
- images on the WebGPU tier (the browser-gate record, entry 63), on the operator's Apple M1 Max in Playwright's full Chromium (new headless mode), on the Mac's own adapter, vendor `apple`, architecture `metal-3`. SmolVLM-256M-Instruct's browser check ran on a test build of the browser host that admitted the model on WebGPU ahead of this version; the version that ships it differs from that build in where the admission comes from. At f32 weight storage, checked: in the test worker three low-detail images, the reference's decode of the JPEG and a 13-tile high-detail image gave transformers' tokens, with every checked tile's image rows within the check's bound, and a WebP was refused by name; through `Caldrix.load` the three low-detail images and a `Uint8Array` gave the test worker's tokens, a call aborted at tile 2 of 13 ended with one `abort` delta, and the engine was usable after. At f16 weight storage the `Caldrix.load` cases ran as a diagnostic: their tokens were recorded, not compared.
- **Not run:** images on any other WebGPU adapter, browser or device, or at f16 weight storage as a checked form; `backend: "auto"` on a real vision model; a multi-tile image of a real model through `Caldrix.load`; any other high-detail image or a bf16 high-detail image; any real GGUF repo by id in a browser; Llama 3.2 or any model with a 128k vocabulary in a browser; a 2D-grid dispatch on a real adapter (the WebGPU row kernels past the grid dimension limit ran on lavapipe only); the WebGPU heap clamp in a browser; any other WebGPU adapter or GPU; the WebAssembly tier in a headed browser;
  the scalar (non-SIMD) wasm build in a
  browser; real-Hub digest behaviour (the runs used a loopback Hub stand-in); a real Hub listing for a repo id
  (the loopback hub served the pinned snapshots) beyond the Try page's own loads (entry 21); any browser other than
  Playwright's Chromium and Brave (entry 21); any model beyond SmolLM2-135M-Instruct and the ones in entries 12, 16, 17, 18, 19, 21, 59 and 60;
  `Caldrix.check` against a real adapter's buffer limit (its unit tests only), against the real Hub, or in any other
  browser or on any other device; `Caldrix.fit` on facts the nightly recorded for a real repo, or on a real adapter. No performance figure is recorded or claimed.

## The default context

On both tiers, a default context whose fit does not stay 64 MiB under the wasm32 heap's 4 GiB ceiling is clamped to the largest multiple of 256 whose fit does (a margin for the fit's stated, unmeasured terms; the margin itself is not yet measured; an explicit `maxContext` is held to the ceiling itself), and on WebGPU the smaller of that and the grid bound applies (`doctor()` reports it as above). The heap clamp ran in a browser on the wasm tier only, on a tiny fixture with a very large header context; on the WebGPU tier that run printed only the grid clamp (entry 22).

Without `maxContext`, the WebGPU tier clamps the context so the attention kernel's grid stays within
`maxComputeWorkgroupsPerDimension`. At the standard 65535 workgroups per dimension, SmolLM2's 8192
default context loads at 7282 (entries 8 and 9); `doctor()` reports the clamp in `contextClamp` and the
value in `maxContext`. A larger `maxContext` that the adapter's limits cannot hold is refused by WebGPU,
and `"auto"` then serves the load on wasm.

## Network

No telemetry. The network is the Hub, for the model files you asked for, and jsdelivr, for the
package's own assets when the bundle's copy fails.

- **When the fallback fires:** the worker falls back if it fails before it is ready (its script, a chunk
  or the host glue did not load, or the page refused to construct it); the `.wasm` falls back, inside
  the worker, if its fetch fails. Each asset is tried locally first and falls back independently.
- **What it fetches:** `https://cdn.jsdelivr.net/npm/@caldrix-ai/caldrix@<installed version>/dist/worker.js`
  and `.../host/caldrix_web_bg.wasm`, at the version you installed, so the `.wasm` matches the glue.
- **It is never silent:** it is warned once per load on the console, and `doctor().assets` reports the
  source and URL of each asset.
- **Opt out:** `assets: { fallback: false }`. An explicit `assets.worker` or `assets.hostWasm` is used as
  given, with no fallback for that asset. `assets.cdn` points the fallback at your own mirror.
- **What it trusts:** the fallback runs jsdelivr's copy of that version with your page's origin (the
  worker can fetch your origin with its credentials and read its IndexedDB and caches), and no integrity
  check is possible for a worker's module import, so a page that cannot accept that sets
  `assets.fallback: false`.
- **What has been checked:** jsdelivr serves the published package's worker, glue and WebAssembly files
  byte-identical to the npm tarball, with CORS open to any origin. After the first release was published,
  the browser fallback ran once against jsdelivr itself in headless Chromium, with the default `cdn`: a
  page whose bundle lacked the worker, and one whose bundle lacked the `.wasm`, each chatted to done
  with `doctor()` naming the jsdelivr URLs and one warning on the console. That was a one-off check at
  the desk; the committed tests point `assets.cdn` at a loopback origin and never reach jsdelivr.
- **Content-Security-Policy:** a page that uses the fallback needs `worker-src blob:` (a cross-origin
  worker starts through a same-origin module shim), `script-src` and `connect-src` for the CDN origin,
  and `'wasm-unsafe-eval'` for any WebAssembly.

## Bundlers

What ran: Vite 8.3.5 and Next.js 16.4.3 (Turbopack and webpack) with `output: "export"`, a minimal app
each, on the wasm tier with a tiny Llama fixture served from a loopback Hub, in headless Chromium. The
package's own bundler test installs the packed package into those apps and runs in CI. Those apps needed
no bundler configuration. Nothing else is claimed: other versions, other bundlers and the WebGPU tier
under a bundler have not been run. Production builds only (`vite build`, `next build`); `vite dev` and
`next dev` were not run.

- Each app ships an unused copy of the worker and (Next only) an unused copy of the host glue. The worker
  copy comes from the package's `new URL("./worker.js", import.meta.url)` literal, which it keeps so
  bundlers find the worker; the glue copy comes from wasm-bindgen's own `.wasm` URL literal inside the
  glue, which the package does not control.
- By the package's design, not run: with assets on another origin (Vite `base`, Next `assetPrefix`) the
  bundle try is cross-origin, the shim imports the raw copy of the worker whose `../host/` is not beside
  it, and every load falls back to jsdelivr. Set `assets.worker` and `assets.hostWasm` instead, serving
  the package's `dist/` and `host/` together on that origin.
- Next pages that import the package are client components.

## License

The npm package `@caldrix-ai/caldrix` (caldrix.js) is under Apache-2.0; the package carries `LICENSE`
and `NOTICE`. The compiler itself is not open source; the package ships its compiled browser host under
the package's license. Third-party components compiled into the host remain under their own licenses;
see `THIRD_PARTY_NOTICES.md` in the package.
