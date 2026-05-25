# Sandkasten – Advanced sandbox SDK for AI coding agent environments

Sandkasten wraps multiple sandbox providers like Daytona, Fly.io Sprites or simple SSH boxes into a unified API for provisioning machines, preparing environments and executing code in a secure, isolated environment.

> [!WARNING]
> This project is currently in private alpha and not available for public use yet (which is why its currently only re-exporting a private @grips package). The full implementation and source will be available here soon. If you're interested in early access or contributing, please reach out!

## Table of Contents

- [Overview](#overview)
- [Packages](#packages)
  - [`@grips/format`](#gripsformat)
  - [`@grips/harness`](#gripsharness)
  - [`@grips/sdk`](#gripssdk)
  - [`@grips/cli`](#gripscli)
  - [`@grips/react`](#gripsreact)
  - [`website`](#website)
- [Installation](#installation)
- [Quick Start](#quick-start)
  - [CLI](#cli)
  - [SDK](#sdk)
  - [React Components](#react-components)
- [Architecture](#architecture)
- [Development](#development)
- [Contributing](#contributing)

---

## Overview

Modern AI coding agents all store session history differently. GRIPS unifies them under one schema and one API:

- **One schema** for nodes (messages, tool calls, compactions, checkpoints, config changes, and custom events)
- **One tree model** for branching, forking, and navigating session history
- **One streaming format** for real-time output (text, reasoning, tool calls, tool execution stdout/stderr)
- **One SDK** to list, read, create, fork, send messages to, and listen to sessions across every supported harness
- **One CLI** to inspect and manage sessions from the terminal
- **One React library** to render session trees, tables, paths, and streaming UIs

---

## Packages

### Package Dependency Graph

```
cli ──> sdk ──> harness ──> format
                 react ───> format
```

| Package | Purpose | Tech |
|---------|---------|------|
| `@grips/format` | Pure types and schemas — zero I/O | Effect-TS Schema |
| `@grips/harness` | Per-agent adapters (Pi, OpenCode, Claude Code, Codex, Cursor) | Effect-TS |
| `@grips/sdk` | Unified consumer API via Effect `Layer` and `Service` | Effect-TS |
| `@grips/cli` | Terminal interface and TUI | Effect-TS + citty |
| `@grips/react` | React components and hooks for rendering sessions | React + Tailwind |
| `website` | Marketing and documentation site | Vite + React |

---

### `@grips/format`

> Pure types and schemas. No I/O, no Effect runtime.

Defines the canonical data model used by every other package.

**Core types:**

- `Node` — Discriminated union of all node types:
  - `message` — User / assistant / system messages with usage metadata
  - `tool_execution` — Tool call with arguments, output, and exit code
  - `compaction` — Session compaction / summarization event
  - `branch_summary` — Summary of a branched conversation path
  - `checkpoint` — Code snapshot checkpoint
  - `config_change` — Agent configuration change
  - `custom` — Extensible custom event type
- `Session` — Session metadata (id, title, cwd, source harness, timestamps)
- `SessionListItem` — Lightweight session list entry
- `Tree` — Built from an array of nodes; provides `getPath`, `getChildren`, `findLeaves`, `validate`
- `StreamEvent` — Real-time streaming events (`text.start`, `text`, `tool_call.delta`, `tool_execution.stdout`, `done`, `error`, etc.)

**Validation:**

```ts
import { buildTree } from "@grips/format";

const tree = buildTree(nodes);
const result = tree.validate(); // { valid, brokenReferences, orphanedNodes, cycles }
```

---

### `@grips/harness`

> Per-harness adapters that bridge native agent storage and runtime APIs into the GRIPS format.

Each supported agent has two implementations:

- **SessionSDK** — Read/write session storage (`list`, `get`, `create`, `append`, `delete`, `fork`)
- **RuntimeSDK** — Drive a live agent (`send`, `listen`, `stop`)

**Supported harnesses:**

| Harness | SessionSDK | RuntimeSDK |
|---------|------------|------------|
| Pi (`pi`) | `PiSessionSDK` | `PiRuntimeSDK` |
| OpenCode (`opencode`) | `OpenCodeSessionSDK` | `OpenCodeRuntimeSDK` |
| Claude Code (`claude-code`) | `ClaudeCodeSessionSDK` | `ClaudeCodeRuntimeSDK` |
| Codex (`codex`) | `CodexSessionSDK` | `CodexRuntimeSDK` |
| Cursor (`cursor`) | `CursorSessionSDK` | `CursorRuntimeSDK` |

**ACP Harness base class:**

Agents that expose a JSON-RPC stdio interface can subclass `AcpHarness` in `runtime-sdk/acp.ts` to inherit standardized process handling.

**Error types:**

- `SessionSDKError` / `RuntimeSDKError` — Tagged errors from adapter operations
- `AdapterError` — Harness not found or misconfigured

---

### `@grips/sdk`

> Unified consumer API. All harnesses, one interface.

The `GripsClient` is an Effect `Service` that aggregates every registered harness and routes calls to the right adapter automatically.

**Registry services:**

- `SessionSDKRegistry` — Holds `SessionSDK` instances
- `RuntimeSDKRegistry` — Holds `RuntimeSDK` instances

**Client API:**

```ts
import { GripsClient } from "@grips/sdk";

// List sessions across all harnesses
yield* GripsClient.list({ cwd: "/project", harness: "opencode", limit: 10 });

// Get a session and its tree
const { session, tree } = yield* GripsClient.get("opencode:session-123");

// Create a new session
yield* GripsClient.create({ harness: "claude-code", title: "Fix login bug" });

// Send a message and receive a stream of events
const stream = yield* GripsClient.send("pi:session-456", { content: "Refactor auth.ts" });

// Listen to an already-running session
const liveStream = yield* GripsClient.listen("codex:session-789");

// Fork a session from a specific node
yield* GripsClient.fork("cursor:session-abc", "node-xyz", { title: "Exploration branch" });

// Stop a running session
yield* GripsClient.stop("opencode:session-123");
```

Session IDs are namespaced: `{harness}:{nativeId}`. The client parses the harness prefix and routes to the correct adapter.

---

### `@grips/cli`

> Terminal interface and interactive TUI for managing sessions.

**Commands:**

```bash
# Interactive TUI (default when no args provided)
grips

# List sessions
grips sessions list [--cwd DIR] [--harness NAME] [--json]

# Show a session
grips sessions get <id> [--json | --tree | --nodes] [--thinking]

# Create a session
grips sessions create [--harness NAME] [--cwd DIR] [--title TITLE] [--json]

# Delete a session
grips sessions delete <id>

# Fork a session from a node
grips sessions fork <id> <nodeId> [--title TITLE] [--json]

# Send a message to a running session
grips send <id> --content "MESSAGE"

# Listen to a running session
grips listen <id>

# Stop a running session
grips stop <id>
```

**Auto-detection:** The CLI automatically detects installed agents by checking for their executables (`opencode`, `claude`, `codex`, `cursor-agent`) or home-directory markers (`~/.pi`) and wires only the available harnesses into the Effect dependency layer.

---

### `@grips/react`

> React components, hooks, and headless primitives for rendering GRIPS sessions.

**Exports:**

- `/` — Components + hooks + mock data
- `/components` — Styled UI components
- `/headless` — Unstyled, stateful primitives
- `/hooks` — Reusable state and stream logic

**Components:**

| Component | Description |
|-----------|-------------|
| `SessionTree` | Static hierarchical tree of nodes with type legend |
| `InteractiveSessionTree` | Expandable / collapsible interactive tree |
| `SessionTable` | Tabular session list with sorting and filtering |
| `SessionHeader` | Session metadata header (title, harness, timestamps) |
| `SessionPath` | Breadcrumb path from root to selected node |
| `NodeDetail` | Detail view for any node type with type-specific rendering |
| `SessionView` | Composite view combining tree, path, and detail |

**Hooks:**

| Hook | Purpose |
|------|---------|
| `useTree` | Tree state (expand/collapse, select, paths, visibility) |
| `useStreamEvents` | Collect stream events into state |
| `useStreamContent` | Accumulate streaming text / reasoning content |
| `useStreamingNodes` | Merge committed nodes with live draft nodes from a stream |
| `useSelection` | Single / multi selection state |
| `usePath` | Compute ancestor path for a node ID |

**Headless primitives:**

| Primitive | Purpose |
|-----------|---------|
| `Tree` | Render-agnostic tree component (pass your own node renderer) |
| `SessionList` | Render-agnostic list component |

**Peer dependencies:** `react >=19`, `react-dom >=19`, `tailwindcss >=4`

---

### `website`

> Vite + React marketing and documentation site.

Uses TanStack Router and Tailwind CSS. Built with `vite build` and previewed with `vite preview`.

---

## Installation

This repository uses **Bun** workspaces.

```bash
# Install dependencies
bun install

# Type-check all packages
bun run typecheck

# Run tests
bun test
```

### Using packages in your own project

```bash
# Install the SDK
bun add @grips/sdk

# Install React components
bun add @grips/react
```

---

## Quick Start

### CLI

```bash
# Start the interactive TUI
cd /your/project && grips

# List all sessions across all agents
grips sessions list

# View a session transcript
grips sessions get opencode:abc123

# View the branching tree
grips sessions get opencode:abc123 --tree

# Create a session
grips sessions create --harness claude-code --title "Bugfix"

# Send a message
grips send claude-code:xyz789 --content "Fix the race condition in queue.ts"
```

### SDK

```ts
import { Effect } from "effect";
import { GripsClient, SessionSDKRegistry, RuntimeSDKRegistry } from "@grips/sdk";
import { OpenCodeSessionSDK, OpenCodeRuntimeSDK } from "@grips/sdk";

const layer = Layer.mergeAll(
  Layer.succeed(SessionSDKRegistry, { sdks: [new OpenCodeSessionSDK()] }),
  Layer.succeed(RuntimeSDKRegistry, { sdks: [new OpenCodeRuntimeSDK()] })
).pipe(Layer.provide(GripsClient.Default));

const program = Effect.gen(function* () {
  const sessions = yield* GripsClient.list({ limit: 5 });
  const { session, tree } = yield* GripsClient.get(sessions[0].id);
  const stream = yield* GripsClient.send(session.id, { content: "Hello" });

  yield* Stream.runForEach(stream, (event) =>
    Effect.sync(() => console.log(event.type))
  );
});

Effect.runPromise(Effect.provide(program, layer));
```

### React Components

```tsx
import { SessionTree, SessionView, useStreamingNodes } from "@grips/react";
import { mockNodes } from "@grips/react/data/mock";

// Static tree
<SessionTree nodes={mockNodes} />

// Full interactive view with streaming
function MyApp({ nodes, streamEvents }) {
  const { nodes: liveNodes, draftNodeIds } = useStreamingNodes(nodes, streamEvents);
  return <SessionView nodes={liveNodes} autoExpandIds={draftNodeIds} />;
}
```

---

## Architecture

### The Streaming Draft Node Lifecycle

When driving a running agent through `RuntimeSDK`, the event stream merges live output into the tree:

1. **Stream Intake** — `useStreamingNodes` merges committed nodes with incoming `StreamEvent`s.
2. **Draft Node Initialization** — When an active event starts (e.g., `text.start`), a transient "draft node" is created with `_draft: true`.
3. **Delta Merging** — Incremental events continuously merge into the draft node's payload.
4. **Live Auto-Expansion** — Draft node IDs are returned as `draftNodeIds` and mapped to `autoExpandIds` in the `Tree` component, keeping the active streaming turn visible.
5. **Commitment** — On completion events (`text.done`, `tool_execution.end`, `done`), the draft node closes. If the runtime appends it to storage, it becomes a permanent committed node.

### Node Format

Every node shares a common shape:

```ts
interface Node {
  id: string;
  parentId: string | null;
  timestamp: number;
  type: "message" | "tool_execution" | "compaction" | "branch_summary" | "checkpoint" | "config_change" | "custom";
  version?: number;
  payload: /* type-specific */;
  metadata: Record<string, unknown>;
}
```

The `parentId` enables a full directed acyclic graph of session history, supporting branching, forking, and non-linear exploration.

---

## Development

### Scripts

| Script | Description |
|--------|-------------|
| `bun run dev` | Start dev servers (Turbo) |
| `bun run typecheck` | TypeScript check all packages |
| `bun test` | Run all tests |
| `bun run fmt` | Format with oxfmt |
| `bun run lint` | Lint with oxlint |
| `bun run grips` | Run the CLI locally |

### Testing Conventions

- **Colocation required** — Test files (`*.test.ts`, `*.test.tsx`) and Storybook stories (`*.stories.tsx`) must live in the same directory as the source file they cover. No `__tests__/` or `stories/` directories.
- Unit tests: `vitest run` in `packages/react/` and `packages/website/`; `bun test` elsewhere.
- Storybook `play()` tests are real tests and must pass.

### Adding a New Harness Adapter

See [`AGENTS.md`](./AGENTS.md) for the full workflow. The short version:

1. Implement `SessionSDK` in `packages/harness/src/session-sdk/my-agent.ts`
2. Implement `RuntimeSDK` in `packages/harness/src/runtime-sdk/my-agent.ts` (or subclass `AcpHarness`)
3. Export both from `packages/harness/src/index.ts`
4. Register in `packages/sdk/src/registry.ts`
5. Add auto-detection in `packages/cli/src/layer.ts`
6. Add colocated tests

### Adding a New Node Type

See [`AGENTS.md`](./AGENTS.md) for the full workflow. The short version:

1. Define the payload schema in `packages/format/src/node.ts`
2. Add streaming events to `packages/format/src/stream.ts` if applicable
3. Add colors, background, and symbol in `packages/react/src/lib/node-rendering.tsx`
4. Create a detail component in `packages/react/src/components/NodeDetail/details/`
5. Wire it into `NodeDetail.tsx`
6. Add tests and stories

---

## Contributing

1. Follow the existing code style (oxfmt, oxlint).
2. Keep tests and stories colocated with source files.
3. Ensure all tests pass before submitting changes.
4. Make minimal changes — prefer small, focused PRs.

---

## License

MIT
