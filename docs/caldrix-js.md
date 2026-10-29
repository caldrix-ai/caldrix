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
  static load(alias: string, opts?: LoadOptions): Promise<Engine>;
}
class Engine {
  chat(messages: ChatMessage[], opts?: GenerateOptions): AsyncIterable<Delta>;
  complete(input: string | { ids: number[] }, opts?: GenerateOptions): AsyncIterable<Delta>;
  doctor(): DoctorReport;
  dispose(): void;
}
function probe(): Promise<Capabilities>;
```

- **`chat`** formats the messages with the model's chat template; **`complete`** continues a plain-text
  prompt, or a prompt given as token ids. Both yield deltas until one has `done: true`.
- **`doctor()`** reports the load: the tier that runs (`backend`), `device`, `dtype`, `storage`,
  `adapter`, on WebGPU only `maxContext` and `contextClamp`, where each model file came from and whether
  its digest was checked (`source`), where the package's own files came from (`assets`: `bundle`, `cdn`
  or `page`, with the URL), and `why` the tier was chosen.
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
  onProgress?: (p: { file: string; loaded: number; total: number | null }) => void;
  hubBaseUrl?: string;
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
  fails to open) is served by the wasm tier, and `doctor().why` says so.
- **`onProgress`**: called with each model file's progress: the file, the bytes loaded and the total
  (`null` when the size is not known).
- **`dtype`**: the plan's dtype, `"f32"` or `"bf16"`.
- **`storage`**: the WebGPU tier's weight storage. `"auto"` stores the block GEMM weights as f16 when the
  adapter has `shader-f16`; `"f32"` keeps them f32 on any adapter. `doctor().storage` reports it (`null`
  on wasm). `storage: "f32"` with `backend: "wasm"` is refused.
- **`maxContext`**: the context length to load at; see [the WebGPU default context](#the-webgpu-default-context).
- **`hubBaseUrl`**: where model files are fetched from (default `https://huggingface.co`). **`offline`**
  uses only the browser's cache and fails with `fetch` for a file it lacks.
- **`signal`**: cancels the load. Before the model is loaded, an abort terminates the engine's worker
  and the load rejects with `fetch`; after the load resolves the signal is ignored.
- **`assets`**: where the package's own files come from; see [Network](#network).

## What has run, and where

Only these browser runs are recorded (the browser-gate record, entries 5, 8 and 9); nothing else is
claimed.

- **Model:** SmolLM2-135M-Instruct, one prompt, 8 greedy tokens. No other model or prompt.
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
- **Not run:** any other adapter or GPU; a headed browser; the scalar (non-SIMD) wasm build in a
  browser; real-Hub digest behaviour (the runs used a loopback Hub stand-in); any browser other than
  Playwright's headless Chromium; any other model. No performance figure is recorded or claimed.

## The WebGPU default context

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
