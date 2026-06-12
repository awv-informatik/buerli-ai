# @buerli.io/ai

AI agent for buerli/ClassCAD apps. Drops a chat panel into your app that creates and
modifies 3D geometry through natural language — any tool-calling LLM, all CAD operations
executed locally in the browser.

## Install

```bash
npm install @buerli.io/ai
```

Peers (your buerli app has them already): `@buerli.io/classcad`, `@buerli.io/core`,
`@react-three/fiber` ≥8, `react` ≥18, `zustand` ≥4.

## Setup — provider & models

The agent talks to any LLM through a small `LLMProvider` interface. Pick one:

| You have | Use |
| --- | --- |
| GitHub Copilot subscription | bundled `copilot-proxy` + `createAutoProvider` (below) |
| OpenAI / Azure key | `createAutoProvider({ apiKey, endpoint? })` |
| Anthropic key | `createAnthropicProvider({ apiKey })` |
| Local AI (Ollama, LM Studio, vLLM) | `createOpenAIProvider({ endpoint: 'http://localhost:11434/v1/chat/completions', model })` |
| Something else | implement `LLMProvider` (one `chat()` method — see Custom integration) |

`createAutoProvider` is the recommended default: it reads the endpoint's `/models` and
routes each model to the right API surface automatically (`gpt-5.x`/codex → Responses,
Claude/Gemini → Chat Completions), and discovers per-model context windows and thinking
levels — so the panel's model/thinking pickers just work. Endpoints without `/models`
fall back to Chat Completions. (The single-surface adapters it wraps —
`createOpenAIProvider`, `createResponsesProvider` — are exported too if you want to pin
one surface.)

### GitHub Copilot (development)

The browser can't call `api.githubcopilot.com` (no CORS, short-lived session tokens), so
the package ships a tiny local proxy:

```bash
npx copilot-proxy auth     # one-time GitHub device-flow login; scaffolds ./.env.local
npx copilot-proxy models   # list models your plan exposes
npx copilot-proxy          # run on http://localhost:8788
```

`auth` writes a ready-to-edit `./.env.local`:

```bash
AI_AGENT_API_KEY=copilot-proxy            # placeholder; the proxy injects the real token
AI_AGENT_ENDPOINT=http://localhost:8788/v1
AI_AGENT_MODEL=gpt-5.5                    # default; the panel picker can switch any time
AI_AGENT_REASONING_EFFORT=medium          # default thinking level
COPILOT_OAUTH_TOKEN=ghu_…                 # SERVER-ONLY secret — never bundle into the client
```

> `COPILOT_OAUTH_TOKEN` must never reach the browser. If your app forwards `.env.local`
> into client code (Vite `import.meta.env`, Docusaurus `customFields`), expose an
> **allowlist** of safe keys — never the whole file. Keep `.env.local` gitignored.
> Port override: `COPILOT_PROXY_PORT`.

`createCopilotProvider({ token })` also exists for Node/VS Code contexts where you
already hold a Copilot session token (it does not work in the browser).

## Usage (React / react-three-fiber)

```tsx
import { createCadAgent, createAutoProvider, initAgentAsync } from '@buerli.io/ai'
import { useBuerli, BuerliGeometry } from '@buerli.io/react'
import { Canvas } from '@react-three/fiber'

await initAgentAsync() // once at startup — loads bundled ClassCAD docs + method registry

const { AgentPanel, AgentCanvas } = createCadAgent({
  provider: createAutoProvider({ apiKey: ENV.AI_AGENT_API_KEY, endpoint: ENV.AI_AGENT_ENDPOINT }),
  modelName: ENV.AI_AGENT_MODEL,            // default model (picker can change it)
  reasoningEffort: 'medium',                // default thinking level
  // maxTokens, contextLimit — optional fallbacks; per-model values are auto-discovered
  // maxIterations: 25                      — tool-loop cap
  // systemPrompt / extraContext            — see Custom prompts
})

function App() {
  const drawingId = useBuerli((s) => s.drawing.active || '')
  const [open, setOpen] = useState(true)
  return (
    <div style={{ position: 'relative', width: '100%', height: '100%' }}>
      <Canvas>
        <AgentCanvas />                      {/* invisible — lets the agent snapshot the view */}
        <BuerliGeometry drawingId={drawingId} />
      </Canvas>
      <AgentPanel drawingId={drawingId} open={open} onClose={() => setOpen(false)} />
    </div>
  )
}
```

`<AgentPanel>` props: `drawingId` (required), `open`, `onClose`,
`position` (`'right' | 'left' | 'bottom'`), `className`, `theme`
(`{ bg, text, accent, userBubble, assistantBubble, width }`), and `extraContext`
(per-mount domain prompt — overrides the factory default, useful when one agent serves
several screens).

What you get, all capability-driven (each control only appears when the provider/model
actually supports it):

- **Model picker + thinking picker + context ring** in the footer — fed by the
  provider's `/models` discovery.
- **Attachments** — images (vision) and CAD files (STEP/IGES/STL…) via the `+` button.
- **Stop** to abort a running turn; **per-tool status chips** with results and errors.
- **Source panel** (`</>` in the header) — the session as a runnable, syntax-highlighted
  buerli script: runtime IDs threaded into variables, preconditions and failures as
  comments, one-click copy.

## Custom prompts

The default system prompt makes the agent a general ClassCAD expert. Add your domain on
top with `extraContext` (recommended — keeps the base expertise), or replace the whole
prompt with `systemPrompt`:

```tsx
createCadAgent({
  provider,
  extraContext: `## This app: parametric pipe runs
Segments are named Default, Pipe1, …; expressions: length, outerDiam, thickness.
Angles in radians (UI shows degrees). Always recalc after edits.`,
})
```

Teach it your model's structure, naming, units, and the exact calls for common
operations — the more concrete, the fewer tool-discovery turns the agent needs.
When replacing `systemPrompt`, you can compose with the exported
`DEFAULT_SYSTEM_PROMPT` to keep the base ClassCAD expertise.

## What the agent can do (tools)

| Tool | Purpose |
| --- | --- |
| `call_api` | Any `v1.<domain>.<method>` ClassCAD call (also `facade.*` and drawing APIs) |
| `call_api_batch` | Many calls in one turn; later calls reference earlier results (`"$0.id"`) |
| `tree` / `find` / `inspect` | Read the structure tree, search nodes, full node detail |
| `get_selection` / `set_selection` | Read or set the user's 3D selection |
| `list_methods` / `describe_method` | Discover and document API methods (bundled docs) |
| `snapshot` | See the 3D viewport (PNG → vision) |
| `load_file` | Import a user-attached CAD file |
| `download` | Export STEP/STL/OFB as a download button in the chat |
| `delegate` | Hand a sub-task to a specialist sub-agent |

Everything executes in the browser against the buerli API — no extra server for CAD.
(`TOOL_SCHEMAS` and `executeTool` are exported for tests or custom executors.)

## Production

Never ship API keys in the browser. Point the provider at your backend and authenticate
your users there:

```ts
// client
createAutoProvider({ apiKey: userSessionToken, endpoint: 'https://your-app.com/api/ai' })
// server: verify the user, attach the real provider key, forward the body verbatim.
```

The bundled Copilot proxy is a development convenience, not a multi-user production
gateway.

## Custom integration (your own UI)

`<AgentPanel>` is a thin view over `createAgentStore()` — build your own panel on the
same store and keep the full agent (tools, batching, attachments, cancel):

```tsx
import { createAgentStore, createAutoProvider, initAgentAsync } from '@buerli.io/ai'
import type { AgentConfig, UIMessage } from '@buerli.io/ai'

await initAgentAsync()
const useAgent = createAgentStore() // zustand — React hook AND vanilla store

const config: AgentConfig = {
  provider: createAutoProvider({ apiKey: '…', endpoint: '…' }),
  drawingId,
  model: 'gpt-5.4',            // optional per-message override
  reasoningEffort: 'none',     // optional
  extraContext: MY_PROMPT,     // optional
}

function MyPanel() {
  const messages = useAgent((s) => s.messages)   // UIMessage[]
  const isRunning = useAgent((s) => s.isRunning)
  const error = useAgent((s) => s.error)
  const usage = useAgent((s) => s.usage)         // { inputTokens, outputTokens }

  const send = (text: string) => useAgent.getState().sendMessage(text, config)
  // attachments: sendMessage(text, config, images?: ImageInput[], files?: FileAttachment[])

  return (
    <>
      {messages.map((m: UIMessage, i) => {
        if (m.type === 'assistant') return <Md key={i} text={m.text} />
        if (m.type === 'thinking') return <Collapsed key={i} text={m.text} />
        // m.type === 'tool': { name, label, status: 'running'|'done'|'error',
        //                      detail?, image? (snapshot data-url), download? }
        return <ToolChip key={i} {...m} />
      })}
      {error && <Error text={error} />}
      <Input onSubmit={send} disabled={isRunning} />
      {isRunning && <button onClick={() => useAgent.getState().stop()}>Stop</button>}
    </>
  )
}
```

- Non-React: same store via `useAgent.getState()` / `useAgent.subscribe()`.
- `useAgent((s) => s.codeLog)` (`CodeEvent[]`) holds the API-call log for a source view.
- `reset()` clears the conversation.

**Snapshots without `<AgentCanvas>`** — register a capturer once and the `snapshot` tool
works with any viewer:

```ts
import { setSnapshotCapturer, createCanvasCapturer } from '@buerli.io/ai'
setSnapshotCapturer(createCanvasCapturer(myCanvasElement)) // or a custom SnapshotCapturer → base64 PNG
```

**Model/thinking pickers** — `provider.getCapabilities()` returns
`{ models: ModelOption[] }` (ids, context limits, reasoning levels); render your own
picker and pass the choice as `config.model` / `config.reasoningEffort`.

**Fully headless** — drive the raw event loop, no store, no React:

```ts
import { runAgentLoop } from '@buerli.io/ai'
for await (const ev of runAgentLoop('Create a 50mm box', [], config)) {
  // ev.type: 'text' | 'thinking' | 'tool_start' | 'tool_end'
  //        | 'subagent_start' | 'subagent_end' | 'usage' | 'error' | 'done'
}
```

**Custom provider** — one method; must support tool-use blocks. Optionally add
`getCapabilities()` to power the pickers:

```ts
const myProvider: LLMProvider = {
  async chat({ system, messages, tools, max_tokens, model, reasoningEffort }) {
    // call your LLM; return { content: ContentBlock[], stop_reason: 'end_turn' | 'tool_use' }
  },
}
```

`initAgent({ skillBundle, methodRegistry })` is the synchronous init variant — pass the
JSONs yourself (`import bundle from '@classcad/skill/bundle.json' with { type: 'json' }`)
if you can't await at startup or run in pure-Node ESM.

## Development

```bash
npm run typecheck
npm run build        # clean + tsc → dist/ — runs automatically on publish
```

All ClassCAD knowledge (the doc bundle and the method registry, both generated in the
classcad-skill repo from `@classcad/api-js`) ships in the **`@classcad/skill`** runtime
dependency — `initAgentAsync()` imports it from there, so there is nothing to re-bundle
or vendor here. Bump that dependency to pick up new docs.
