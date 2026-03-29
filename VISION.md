# action-graph

> The missing decision layer between AI agent triggers and execution.

## Problem

AI agent frameworks offer two extremes:

1. **Pure agent** — the LLM decides everything. Flexible but unreliable. The agent might skip steps, call wrong tools, or take actions the user didn't want.
2. **Rigid pipeline** — hardcoded step-by-step flow. Reliable but inflexible. Can't handle novel situations or adapt to user preferences.

Neither works for production workflows where **trust must be earned over time**.

## Vision

A new employee asks for approval on everything. Over time, they prove they make the right calls. The manager stops checking routine decisions and only reviews novel ones.

**action-graph** brings this pattern to AI agents:

1. **Trigger** — something happens (message, webhook, cron, tool result)
2. **Fan out** — enumerate possible actions from user-configured templates
3. **Evaluate** — LLM checks which actions are valid given current context
4. **Gate** — user approves (or auto-approves if confidence is high enough)
5. **Execute** — host framework runs the approved actions
6. **Learn** — approval/denial patterns build confidence over time

```
Incoming lead email
  → Possible actions:
    [x] Search CRM for existing contact    (auto-approved — 95% confidence)
    [x] Enrich from LinkedIn               (needs approval — 60% confidence)
    [ ] Schedule showing                   (invalid — no property mentioned)
    [x] Draft reply                        (auto-approved — 90% confidence)
  → User sees: "I'd like to enrich from LinkedIn. Approve?"
  → User: "yes"
  → Next time: auto-approved (confidence now 75%)
```

## Goal

Make AI agent workflows as reliable as human workflows — where autonomy is earned, not assumed.

## Design Principles

| Principle                           | Implementation                                                              |
| ----------------------------------- | --------------------------------------------------------------------------- |
| **Zero dependencies**               | Pure TypeScript, no LLM SDK, no framework imports                           |
| **Model agnostic**                  | Host provides a `ConditionEvaluator` function — any LLM works               |
| **Framework agnostic**              | 5 standard hooks that map to any agent pipeline                             |
| **Config, not code**                | Action templates are JSON/YAML — users define workflows without programming |
| **Conditions are natural language** | "when mentions property" — the LLM evaluates, not regex                     |
| **Confidence is earned**            | Starts at 0, grows with approvals, resets on denials                        |
| **Transparent**                     | Every decision is explainable — "approved because 12/12 past approvals"     |

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Host Application                  │
│  (OttoRuntime, LangChain, n8n, OpenAI SDK, custom)  │
│                                                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│  │ Trigger  │  │  Agent   │  │ Channel │             │
│  │ (cron,   │  │  (LLM,   │  │ (Slack, │             │
│  │  webhook)│  │  tools)  │  │  email) │             │
│  └────┬─────┘  └────┬─────┘  └────┬────┘             │
│       │              │              │                  │
│  ─────┼──────────────┼──────────────┼───────────────  │
│       │              │              │                  │
│       ▼              ▼              ▼                  │
│  ┌──────────────────────────────────────────────┐    │
│  │              action-graph                     │    │
│  │                                               │    │
│  │  Templates ──→ Evaluate ──→ Gate ──→ Learn   │    │
│  │  (config)     (LLM)       (approve)  (track) │    │
│  │                                               │    │
│  │  5 Hooks:                                     │    │
│  │    onInput · onActionProposed · onToolResult  │    │
│  │    onStepComplete · onBeforeResponse          │    │
│  └──────────────────────────────────────────────┘    │
│                                                      │
└─────────────────────────────────────────────────────┘
```

## API

```typescript
import { createActionGraph } from "@maxtongwang/action-graph";

const graph = createActionGraph({
  // Host provides LLM for condition evaluation
  evaluate: async (condition, context) => {
    return await myLlm.judge(condition, context);
  },
  // Action templates — user-configured
  templates: [
    { trigger: "new_email", action: "search_crm", always: true },
    {
      trigger: "new_email",
      action: "enrich_linkedin",
      when: "has linkedin url",
    },
    { trigger: "new_email", action: "draft_reply", always: true },
  ],
  // Approval history — from DB or in-memory
  approvals: loadApprovalHistory(),
  // Auto-approve after N consecutive approvals with 0 denials
  autoApproveThreshold: 10,
});

// 5 hooks — wire into your framework
const actions = await graph.onInput(triggerContext);
// → [{ action: "search_crm", valid: true, confidence: 1.0, autoApproved: true }, ...]

const decision = await graph.onActionProposed(action);
// → { approved: true, reason: "auto-approved (12/12 past approvals)" }

const followups = await graph.onToolResult(result);
// → [{ action: "draft_reply", valid: true }]
```

## Confidence Model

```
Initial:      confidence = 0.0 (always ask)
After 1 yes:  confidence = 0.1
After 5 yes:  confidence = 0.5
After 10 yes: confidence = 1.0 (auto-approve)
After 1 no:   confidence *= 0.5 (decay on denial)
After reset:  confidence = 0.0

Pattern key: hash(trigger_type + action_name + condition)
Each unique pattern tracks its own confidence independently.
```

## Visual Workflow (GUI Layer)

action-graph supports rendering workflow graphs in two formats for user review and creation:

### ASCII (Terminal / Chat)

When a user asks "show me my workflows" in a chat interface, render as ASCII:

```
┌──────────────┐
│  new_email   │ trigger
└──────┬───────┘
       │
  ┌────┴────┐
  ▼         ▼
┌─────┐  ┌─────────┐
│ CRM │  │LinkedIn │ when: "has linkedin url"
│search│  │ enrich  │ confidence: 60%
│ ✅   │  │ ⚠️ ask  │
└──┬──┘  └────┬────┘
   │          │
   ▼          ▼
┌──────────────┐
│  draft_reply │ always
│     ✅       │ confidence: 90%
└──────────────┘
```

### HTML (Web App / Dashboard)

For web applications, export workflow graphs as renderable data:

```typescript
const visual = graph.toVisual(templates);
// Returns: { nodes: [...], edges: [...], layout: "dagre" }

// Host renders with any graph library:
// - React Flow
// - D3.js
// - Mermaid
// - Cytoscape
```

The library exports **data**, not UI components. The host picks their rendering library.

### Interactive Workflow Builder

For web apps that want users to CREATE workflows visually:

```typescript
// Export current templates as editable graph
const editable = graph.toEditable();

// User modifies in UI (drag nodes, add conditions, set thresholds)
// UI sends back updated templates

// Import modified templates
graph.updateTemplates(modifiedTemplates);
```

The library handles serialization/deserialization. The host builds the UI.

### Mermaid Export

For documentation and sharing:

```typescript
const mermaid = graph.toMermaid(templates);
// Returns: "graph TD\n  A[new_email] --> B[CRM search]\n  ..."
```

Renders in GitHub READMEs, Notion, Slack, any markdown viewer.

## Web App Extension

action-graph is designed to be embeddable in web applications:

### REST API Wrapper

```typescript
import { createActionGraph } from "@maxtongwang/action-graph";
import { createServer } from "@maxtongwang/action-graph/server"; // optional

const graph = createActionGraph({ ... });
const server = createServer(graph, { port: 3200 });

// Endpoints:
// POST /decide        — evaluate actions for a trigger
// POST /approve/:id   — approve a pending action
// POST /deny/:id      — deny a pending action
// GET  /templates     — list action templates
// PUT  /templates     — update action templates
// GET  /confidence    — view confidence scores
// GET  /visual        — get graph visualization data
// GET  /history       — approval/denial history
```

### Embeddable Widget

For apps that want a drop-in approval UI:

```html
<action-graph-widget endpoint="http://localhost:3200" theme="dark" />
```

Renders pending approvals, confidence scores, and workflow visualization.
Built as a web component — works in React, Vue, Svelte, vanilla HTML.

## What This Is NOT

- **Not a workflow engine** — it doesn't execute actions. The host does.
- **Not an agent framework** — it doesn't call LLMs for reasoning. The host does.
- **Not a UI library** — it exports data for visualization. The host renders.
- **Not a database** — it takes approval history as input. The host stores it.

It's the **decision layer** — the thin, focused piece between "something happened" and "here's what should happen next, and here's why."

## Roadmap

| Phase    | Scope                                                      |
| -------- | ---------------------------------------------------------- |
| **v0.1** | Core: templates, evaluate, confidence, 5 hooks             |
| **v0.2** | Persistence adapters (in-memory, file, Postgres)           |
| **v0.3** | Visual: ASCII, Mermaid, HTML graph data export             |
| **v0.4** | Server: REST API wrapper, embeddable widget                |
| **v1.0** | Stable API, comprehensive docs, examples for 5+ frameworks |

## License

MIT
