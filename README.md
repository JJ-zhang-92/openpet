# openpet

<p align="center">
  <b>OpenCode Agent Desktop Pet</b><br>
  <sub>4-phase state machine — your agent shows you what it's doing</sub>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/opencode-plugin-blue">
  <img src="https://img.shields.io/badge/copet-powered-green">
  <img src="https://img.shields.io/badge/license-MIT-yellow">
</p>

---

## What is this

openpet is an OpenCode plugin that turns your AI agent's internal state into **visual feedback on your desktop**. It tracks every tool execution and reasoning phase through OpenCode hooks, then drives a desktop pet (via CoPet) to reflect four distinct states in real time.

The plugin itself contains **zero GUI code**. All visual rendering is delegated to [CoPet](https://github.com/ChanceYu/CoPet), an MIT-licensed desktop pet platform built with Tauri + Rust + React.

---

## Four-Phase State Machine

OpenCode hooks provide a deterministic view into the agent's activity. openpet maps these hooks into four states with clear visual feedback:

| State | CoPet Animation | Trigger | What it means |
|---|---|---|---|
| **Executing** | Running | `tool.execute.before` | Agent is running tools (bash, read, write, etc.) |
| **Thinking** | Review | tool queue empty → `text.complete` | Agent is analyzing results, planning next steps |
| **Waiting** | Alert/Waiting | `permission.ask` | Agent is blocked, waiting for your input |
| **Idle** | Idle | no activity for 30s | Agent is done, waiting for a new prompt |

### Why Thinking State Matters

Most agent visualizations use a binary model: **working** or **not working**. But a real AI coding agent has a critical gap between tool completion and text output — the LLM is processing context, analyzing results, and formulating a response. This gap can last 5-20 seconds with no hook events.

Without a Thinking state, the pet transitions to Idle immediately after the last tool completes. The user sees a pet doing nothing and assumes the agent is stuck. Whether this assumption is correct or not, the trust erosion is real.

openpet enters Thinking state **as soon as the tool queue empties**, before the LLM returns any text, and stays there until `text.complete` fires. The pet shows a thinking animation throughout — a visual contract that says "I'm still working on it."

---

## Architecture

```
OpenCode hooks → progress-float.js plugin
                    ├─→ POST /report → progress-server.js (HTTP aggregation)
                    └─→ POST events → CoPet Runtime → Desktop Pet Window
```

The plugin is ~370 lines of JavaScript with zero dependencies beyond Node.js built-ins. It:

1. Listens to OpenCode hooks (`chat.message`, `tool.execute.before/after`, `experimental.text.complete`, `permission.ask`)
2. Maintains a deterministic state machine (priority: waiting > executing > thinking > idle)
3. Reports full state to a local HTTP aggregation server (`progress-server.js`, ~280 lines) for debugging/dashboard use
4. Sends simplified events to CoPet's local runtime endpoint for visual feedback

---

## Relationship with CoPet

openpet is a **plugin layer only**. It does not render anything — CoPet handles all visual output.

| Component | Where | Purpose |
|---|---|---|
| **openpet** | This repo | Hook → state machine → HTTP reporting + CoPet events |
| **CoPet** | [`./CoPet/`](./CoPet/) (forked, in-repo) | Visual desktop pet runtime with Thinking state |
| **CoPet upstream** | [ChanceYu/CoPet](https://github.com/ChanceYu/CoPet) | Original platform (MIT) |

### Why a CoPet fork is bundled

CoPet's `AgentState` was designed for Codex IDE's API, which has no concept of an "AI agent thinking." The `thinking` event was internally mapped to the `waiting` state, causing it to share the same visual as permission prompts.

Our fork adds 20 lines across 6 files:

1. **`PetStateId::Thinking`** — a distinct state variant
2. **30-second `idleAfter` for Thinking** — LLM processing takes 5-20s, not 1.5s
3. **Frontend mapping** — thinking → review sprite row (Row 8)

The CoPet source is included directly in this repo so users can `clone` and `pnpm tauri build` in one place. An [upstream PR](https://github.com/ChanceYu/CoPet/pull/1) has been submitted — if accepted, the bundled fork can eventually be removed.

#### Enhancements beyond upstream

This fork adds the following features on top of CoPet (by ChanceYu, MIT):

| Feature | Description |
|---|---|
| **Thinking state** | A distinct `PetStateId::Thinking` with 30s `idleAfter` — keeps the pet in thinking animation during LLM processing (5–20s gaps) |
| **Font size control** | Independent slider (8–32px) for notification text size, decoupled from pet scaling |
| **Urgency coloring** | Messages color-coded by interaction urgency — waiting=red / error=orange / running=blue / thinking=gray / done=light gray |
| **Adaptive font scaling** | Text size scales with urgency — waiting messages 1.3× larger, done messages 0.75× smaller |

All enhancements built on [CoPet](https://github.com/ChanceYu/CoPet) by ChanceYu (MIT).

### Building CoPet from this repo

```bash
cd CoPet
pnpm install
pnpm tauri build
# Binary at CoPet/src-tauri/target/release/CoPet.exe
```

Copy the binary to `~/.copet/bin/CoPet.exe`, run the setup wizard, enable OpenCode integration.

---

## Quick Start

### 1. Build and Install CoPet

```bash
cd CoPet
pnpm install
pnpm tauri build
# Copy the binary
cp src-tauri/target/release/CoPet.exe ~/.copet/bin/CoPet.exe   # Windows
# cp src-tauri/target/release/CoPet ~/.copet/bin/              # macOS/Linux
```

Run CoPet, complete the setup wizard — select a pet and enable OpenCode integration.

### 2. Register Plugin

In your `opencode.jsonc`:

```jsonc
{
  "plugin": [
    "path/to/openpet/plugin/progress-float.js"
  ]
}
```

### 3. Done

Start OpenCode. The pet reacts to your agent's activity.

---

## Custom Pets

openpet includes `tools/convert-to-copet.py` — a converter that turns 4 PNG sprites into a Codex-compatible spritesheet for CoPet:

```
python tools/convert-to-copet.py \
  --sprite-dir sprites/my-pet \
  --out-dir ~/.copet/pets/my-pet \
  --pet-name "My Pet" \
  --pet-id "my-pet" \
  --description "My custom pet"
```

Expected sprites in the source directory:

| File | CoPet Row | State |
|---|---|---|
| `idle.png` | Row 0 | Idle |
| `alert.png` | Row 6 | Waiting |
| `working.png` | Row 7 | Executing |
| `thinking.png` | Row 8 | Thinking |

---

## Configuration

All in `config.json`:

| Parameter | Default | Description |
|---|---|---|
| `port` | 19822 | Aggregation server port |
| `toolTimeoutMs` | 120000 | Max tool runtime before auto-marked done |
| `sessionTtlMs` | 600000 | Idle session lifetime |
| `reportTtlMs` | 30000 | Project report TTL |

---

## Testing

```bash
node test-build-state.js
```

Uses Node.js built-in test runner (`node:test` / `node:assert`), zero dependencies. Covers timeout, session cleanup, trimming, edge cases.

---

## License

MIT
