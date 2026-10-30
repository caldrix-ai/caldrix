# aerix.js

aerix.js runs Aerix's turn loop in the browser: an agent with tools, hooks and named limits, running on
a [caldrix.js](caldrix-js.md) engine. It is one npm package, `@caldrix-ai/aerix`. aerix.js loads nothing
itself: you load a model with caldrix.js and hand its `Engine` to an `Agent`. The loop asks the engine
for one turn at a time, runs the tools the turn calls, and feeds the results back until the model
answers with a final or a limit is reached.

`@caldrix-ai/caldrix` is a peer dependency (`^0.2.0`, the first release with constrained decoding).

## Install and quick start

```sh
npm install @caldrix-ai/caldrix @caldrix-ai/aerix
```

```ts
import { Caldrix } from "@caldrix-ai/caldrix";
import { Agent, tool } from "@caldrix-ai/aerix";

const engine = await Caldrix.load("smollm2-instruct");

const weather = tool({
  name: "weather",
  description: "Today's forecast for a city.",
  input: {
    city: { type: "string", description: "City name", required: true },
    unit: { type: "string", enum: ["c", "f"] },
  },
  run: async ({ city, unit }) => `Sunny in ${city}, 21°${unit ?? "c"}`,
});

const agent = new Agent({
  model: engine,
  instructions: "You are a travel helper. Use the tools to answer.",
  tools: [weather],
  limits: { turns: 8, retries: 2, maxTokens: 384, resultChars: 4000 },
  generation: { temperature: 0 },
  hooks: { beforeCall: (call) => (call.tool === "weather" ? undefined : { deny: "not allowed" }) },
});

const controller = new AbortController();
for await (const ev of agent.run("Should I pack an umbrella for Lisbon?", { signal: controller.signal })) {
  if (ev.type === "final") console.log(ev.text);
}
const { text } = await agent.ask("And for Porto?"); // the same conversation
console.log(text);
agent.reset();
```

`agent.run` yields the run's events; `agent.ask` runs to the end and resolves the final's text. Both
continue the agent's conversation; `agent.reset()` forgets it.

## The API

The package exports `Agent`, `tool` and `AerixError`, and the types `AgentOptions`, `Limits`, `Hooks`,
`AgentEvent`, `Call`, `EndReason`, `Model`, `Tool`, `ToolDef`, `Field` and `FieldType`. As the package's
sources have them:

```ts
class Agent {
  constructor(o: AgentOptions);
  get messages(): readonly ChatMessage[];
  reset(): void;
  run(prompt: string, o?: { signal?: AbortSignal }): AsyncGenerator<AgentEvent, void, undefined>;
  ask(prompt: string, o?: { signal?: AbortSignal }): Promise<{ text: string; events: AgentEvent[] }>;
}
function tool<I = Record<string, unknown>>(def: ToolDef<I>): Tool;
```

`ChatMessage`, `Delta`, `GenerateOptions` and `GenerationOptions` are caldrix.js's types; see
[its API](caldrix-js.md#the-api).

- **`agent.run(prompt, { signal })`** yields `AgentEvent`s and continues the agent's conversation.
- **`agent.ask(prompt, { signal })`** runs to the end and resolves `{ text, events }` on a final; any
  other ending rejects with `AerixError` kind `end`, carrying `reason` and `events`.
- **`agent.reset()`** forgets the conversation. **`agent.messages`** is the conversation so far.

One run at a time: a second concurrent `run` throws `AerixError` kind `busy`, and bad options throw
kind `options`.

### The model

```ts
interface Model {
  chat(messages: ChatMessage[], opts?: GenerateOptions): AsyncIterable<Delta>;
}
```

`Model` is what the loop generates with, over caldrix.js's types. A caldrix.js `Engine` satisfies it.

### Tools

```ts
interface ToolDef<I> {
  name: string;
  description: string;
  input: Record<string, Field>;
  run(input: I, cx: { signal: AbortSignal; callId: string }): unknown | Promise<unknown>;
}

interface Field {
  type: FieldType;
  description?: string;
  required?: boolean;
  enum?: readonly (string | number | boolean)[];
}

type FieldType = "string" | "number" | "integer" | "boolean";

/** A defined tool: the frozen definition and its compiled input schema. */
interface Tool extends Readonly<ToolDef<any>> {
  readonly schema: object;
}
```

`tool(def)` defines a tool:

- `name` matches `^[a-z][a-z0-9_]{0,63}$` and is unique in the agent (a duplicate is refused when the
  agent is built).
- `description` is the text the model sees.
- `input` is a flat field map. Each field has a `type` (`string`, `number`, `integer` or `boolean`)
  and an optional `description`, `required` and `enum`. An empty map means no input.
- `run(input, { signal, callId })` returns a string, or a value that goes through `JSON.stringify`.
  The input is checked against the fields before `run`; a failed check or a throw becomes the result
  `error: …`, and a tool error never ends the run.

### Events

```ts
/** Why a run ended without a final. */
type EndReason = "turns" | "retries" | "abort" | "error";

/** One tool call of a turn. */
interface Call {
  id: string;
  tool: string;
  input: Record<string, unknown>;
}

/** One event of a run. Every run ends with exactly one `final` or `end`. */
type AgentEvent =
  | { type: "text"; text: string }
  | { type: "usage"; promptTokens: number; completionTokens: number }
  | { type: "turn"; turn: number; thought?: string; calls: Call[] }
  | { type: "call"; id: string; tool: string; input: Record<string, unknown> }
  | { type: "result"; id: string; tool: string; text: string; error: boolean }
  | { type: "retry"; turn: number; reason: string }
  | { type: "final"; text: string; thought?: string }
  | { type: "end"; reason: EndReason; message?: string };
```

- `{ type: "text", text }`: raw streamed model text.
- `{ type: "usage", promptTokens, completionTokens }`: one per generation.
- `{ type: "turn", turn, thought?, calls }`, then per call `{ type: "call", id, tool, input }` and
  `{ type: "result", id, tool, text, error }`.
- `{ type: "retry", turn, reason }`: a cut or bad turn is asked again.
- Exactly one ending: `{ type: "final", text, thought? }`, or `{ type: "end", reason, message? }`
  where `reason` is `"turns"`, `"retries"`, `"abort"` or `"error"` (an engine failure or a throwing
  hook, with its message). An engine failure is this event, never a throw.

## Options

`new Agent(options)` takes `AgentOptions`:

```ts
interface AgentOptions {
  model: Model;
  instructions?: string;
  tools?: Tool[];
  limits?: Limits;
  generation?: Omit<GenerationOptions, "maxTokens" | "constraint">;
  /** Send the turn schema as `constraint.json` on each generation (default `true`). */
  constrain?: boolean;
  hooks?: Hooks;
}

interface Limits {
  /** Generations per run, retries and cut turns included. */
  turns?: number;
  /** Extra attempts after a bad or cut turn, per turn. */
  retries?: number;
  /** Tokens per generation. */
  maxTokens?: number;
  /** Characters of one tool result; the rest is cut with a note. */
  resultChars?: number;
}

interface Hooks {
  /** A deny becomes the result `refused: <reason>`; `input` replaces the call's input. */
  beforeCall?(c: Call): Awaitable<BeforeCall>;
  /** `result` replaces the tool's result. */
  afterCall?(c: Call, result: string): Awaitable<AfterCall>;
  /** `continue` gives the agent another turn with that text as a user message (bounded by `turns`). */
  onFinal?(text: string): Awaitable<OnFinal>;
}

// Not exported: the types Hooks is written with.
type Awaitable<T> = T | Promise<T>;
type BeforeCall = void | { deny: string } | { input: Record<string, unknown> };
type AfterCall = void | { result: string };
type OnFinal = void | { continue: string };
```

- **`model`** (required): the `Model` to generate with, such as a caldrix.js `Engine`.
- **`instructions`** (default `""`).
- **`tools`** (default none; then the only answer is a final).
- **`limits`**, each a named bound: `turns` (8, generations per run, retries included), `retries` (2,
  extra attempts after a bad or cut turn), `maxTokens` (512, per generation) and `resultChars` (4000,
  one tool result is cut to this with a note).
- **`generation`**: caldrix.js's `GenerationOptions` without `maxTokens` and `constraint`, which
  aerix.js sets.
- **`constrain`** (default `true`): send the turn schema as the `constraint` of every generation. Set
  `false` for a `Model` without constrained decoding; then only parse-and-retry applies.
- **`hooks`**, each optional and awaited: `beforeCall(call)` may return `{ deny: reason }` (the result
  becomes `refused: <reason>`) or `{ input }` (replaces the call's input); `afterCall(call, result)`
  may return `{ result }`; `onFinal(text)` may return `{ continue: text }`, which gives the agent
  another turn with that text as a user message, within `turns`.

## The turn contract

Every turn is one JSON object, either `{"calls":[{"tool":"…","input":{…}}]}` or `{"final":"…"}`, with an
optional `thought`. The system message states this contract and lists the tools; it is the same
contract and the same tool rendering as Aerix's.

Every turn is constrained (unless `constrain: false`): the engine decodes under a schema built from
the tools, so a turn that ends on its own is a valid turn with a known tool and an input of the right
shape, with one exception: an empty `calls` list passes the constraint, and parsing refuses it
and retries. A turn cut by `maxTokens` before its JSON closes is dropped and asked again, shorter, as a
retry. At most 8 calls of one turn are kept, a `thought` is cut at 600 characters, and calls run in
order, one at a time.

Each turn's prompt is the system message plus every earlier turn and result. The engine keeps what the
previous generation processed, so a turn that only appends to the conversation prefills the new part;
the done delta's `usage.cachedTokens` (caldrix.js's field) says how much was reused; aerix.js's own
`usage` event does not carry it.

## What has run

- The unit tests, against a scripted fake `Model`: the turn parsing, the system rendering and the turn
  schema (golden), the turn contract pinned to Aerix's, input checks, each hook, each ending and each limit.
- One end-to-end test in Playwright's headless Chromium on the wasm tier: an `Agent` with one tool
  on the built caldrix.js, loading a tiny test model whose output is noise, from a loopback server.
  It checks that the engine receives the turn schema on every generation, that every retry follows a
  cut turn, and that the run ends with a named outcome.

Nothing more is claimed: no real model, no other browser or tier, and no quality or speed figure.

## License

aerix.js (the npm package `@caldrix-ai/aerix`) is under Apache-2.0; the package carries `LICENSE` and
`NOTICE`.
