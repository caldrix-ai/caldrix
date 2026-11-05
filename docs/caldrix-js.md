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
  prompt, or a prompt given as token ids. Both yield deltas until one has `done: true`.
- **`doctor()`** reports the load: the tier that runs (`backend`), `device`, `dtype`, `storage`,
  `adapter`, `maxContext` and `contextClamp`, where each model file came from and whether
  its digest was checked (`source`), where the package's own files came from (`assets`: `bundle`, `cdn`
  or `page`, with the URL), `aliases` (every alias this build loads, each with its `name` and its
  `format`, `safetensors` or `gguf`), `provenance` (a densified load's `model:` line, `null` for a model read as it lies), `note` (an nf4 repo's 4-bit note, `null` otherwise), and `why` the tier was chosen.
- **`dispose()`** releases the worker and everything in it. Every later call throws `request`.
- **`probe()`** reports what this page can run, without loading anything: the WebAssembly features, the
  WebGPU adapter if there is one, which tiers are available, the tier `"auto"` would choose (`chosen`)
  and `why`.

```ts
type ChatMessage = { role: "system" | "user" | "assistant"; content: string };

interface Delta {
  text: string;
  token?: number;
  done: boolean;
  finishReason?: "stop" | "length" | "abort";
  usage?: { promptTokens: number; completionTokens: number; cachedTokens: number };
  logprobs?: { token: number; logprob: number }[];
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

type GenerateOptions = GenerationOptions & { signal?: AbortSignal };
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

## What has run, and where

Only these runs are recorded (the browser-gate record: entries 5, 8 and 9 in the browser, and for the
quantized aliases entries 10 and 11 off-browser and entries 12 and 14 in the browser, and for the adapter and
nf4 repos entry 16 off-browser and entry 17 in the browser, and for `Caldrix.check` entry 20 in the browser, and for the Try page entry 21 in the browser, and for a GGUF file in a repo entry 22); nothing else is claimed.

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
- **Not run:** any real GGUF repo by id in a browser; Llama 3.2 or any model with a 128k vocabulary in a browser; a 2D-grid dispatch on a real adapter (the WebGPU row kernels past the grid dimension limit ran on lavapipe only); the WebGPU heap clamp in a browser; any other WebGPU adapter or GPU; the WebAssembly tier in a headed browser;
  the scalar (non-SIMD) wasm build in a
  browser; real-Hub digest behaviour (the runs used a loopback Hub stand-in); a real Hub listing for a repo id
  (the loopback hub served the pinned snapshots) beyond the Try page's own loads (entry 21); any browser other than
  Playwright's Chromium and Brave (entry 21); any model beyond SmolLM2-135M-Instruct and the ones in entries 12, 16, 17, 18, 19 and 21;
  `Caldrix.check` against a real adapter's buffer limit (its unit tests only), against the real Hub, or in any other
  browser or on any other device. No performance figure is recorded or claimed.

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
