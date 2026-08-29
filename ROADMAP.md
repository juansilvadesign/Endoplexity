# Roadmap

Where Endoplexity is now, what has to become true for it to drive more than one
vendor's CLI, and which later work is worth considering.

**Direction:** keep the thesis — *the agent loop is the subscription you already
pay for* — while widening what can supply that loop, and make the approval gate
stop being an English-only heuristic on a web that mostly isn't English.

Setup and the safety model live in **[`README.md`](README.md)**. This file holds
milestone reasoning; the actionable checklist is **[`TASKS.md`](TASKS.md)**.

<!-- PHASE-BLOCK: mirrored verbatim in TASKS.md and ROADMAP.md — edit both or neither -->
> ### Current phase: **A — opencode as the third provider**
> **0** shipped (install + verification). **B** is the release gate for A
> reaching daily use. **C** follows A on the same seam. **D** is deliberately
> behind the CLI lane. Last reviewed **2026-08-28**.
<!-- /PHASE-BLOCK -->

---

## Frame: one seam, then one gate

`v0.0.1` is installed and verified end to end on the bridge side. The next
increment has **one maintainer**, **no bundler or framework**, and **no backend**,
and it must not spend that capacity re-litigating the architecture. Two things
are in scope: provers that the seam generalises, and an approval gate that
survives contact with a non-English page.

| Fixed for the next increment | Open to change |
| --- | --- |
| The bridge stays the only place policy lives | How many providers ship in the release |
| Loopback-only, origin-pinned panel, token-gated `/mcp` | Whether the raw-API lane ships at all |
| Every provider proves its sandbox **by execution** | Which sandbox lever each provider uses |
| No bundler, no framework, `node --test` only | Whether the gate keeps a word list at all |
| Subscription path stays the default | Where API keys live, if they ever exist |

No external launch date. D and E stay coarse rather than pretending to predict
dates nobody has committed to.

---

## Stance: the sandbox is the feature, not the spawn

Adding a provider looks like writing a `runX()` next to `runClaude` and
`runCursor`. It isn't. The spawn is thirty lines; **the isolation is the work**,
and it is the part that fails silently.

`cursor.ts` records the near-miss in its own comments: setting
`CURSOR_CONFIG_DIR` looked like isolation and wasn't, because `mcp.json` resolves
off `homedir()`. Measured 2026-08-03, that left the browser agent holding **27
tools across 7 servers** — including another tool's browser automation and
several plugin-auth integrations — with `endo` itself absent. More reach than the
Claude path and none of the tools it needed.

That trap is **live for both providers below**, and it was confirmed by reading
each one's real config rather than assuming:

- The opencode config on a developer machine commonly enables MCP servers with
  **write access to live production systems**. An unsandboxed run hands an agent
  that is already driving a logged-in browser the ability to change them.
- The Antigravity CLI ships a **headless Playwright browser server** enabled by
  default. Unsandboxed, a browser agent gets a *second*, invisible browser —
  outside the panel, outside the approval gate, outside everything this project
  puts a boundary around.

So the acceptance test for **any** new provider is not "it answered", it is
**"list its tools and see only `endo`"**.

The second load-bearing rule: **a provider is not integrated until its refusal is
observed.** Claude's allowlist and Cursor's deny-wildcards were both confirmed by
actually trying to run a shell command, not by reading the documentation. Nothing
below is allowed to ship on a documentation claim — including the claim that a
provider *has* no headless entry point, which is a mistake this roadmap has
already made once and corrected.

---

## Stance: the gate is language-bound by construction

`gate.ts` decides irreversibility with one regex over a visible label:

```
/\b(submit|apply|pay|buy|purchase|order|checkout|confirm|delete|remove|send|book|sign up)\b/i
```

Adding `enviar|confirmar|pagar|excluir` makes it wrong in Portuguese too, just
later. Three reasons the word list can't be the whole answer:

1. **It is a permanent tail.** Every unlisted language, every unusual phrasing
   ("Finish", "Yes, place it"), and every icon-only button is a silent miss.
2. **Inflection doesn't transfer.** `\bremove\b` does not match `remover`, and
   `\bsend\b` says nothing about `enviar`, `envie`, or `enviado`. Stems, not
   words.
3. **Diacritics break the boundary.** `confirmação`, `inscrição`, and `não` need
   NFD folding before any of this matches at all.

So the fix is **structure first, lexicon second**. A `type=submit` button inside a
`<form>` is irreversible in every language on earth, and that signal costs no
translation. The word list stays as the second net for the large class of modern
buttons with no structural tell — a bare `<button>` wired to JS.

**The structural signal does not exist yet, and that is the real cost of this
milestone.** `extension/ax.js` serialises `@ref [role] "name"` plus a fixed flag
set (`required`, `disabled`, `checked`, `expanded`, `selected`, `invalid`).
Nothing carries `type=submit` or form membership. So this work starts in the
extension's serializer, changes the snapshot wire format, and only then reaches
`gate.ts` — which means it also touches the delta protocol (`UNCHANGED` `+`/`-`
lines) and the gate's own `LINE` regex. Three parsers, one format.

**The union is always on.** Detecting the page language and loading only that
lexicon trades a dismiss click for a fail-open, in the one component that must
fail closed — the same reason an unrecognised mode falls back to `normal` and
never `trust`. Over-gating costs a click; under-gating costs whatever the button
did.

---

## Milestones

### Shipped — 0: install and verify ✅

Bridge on WSL2, Chrome on Windows, `systemd --user` autostart. 175/175 tests,
MCP handshake `connected` with all 14 tools, sandbox confirmed (`ToolSearch` +
`mcp__endo__*` and nothing else), origin pin and token gate both probed with
known-bad inputs rather than assumed.

### A — opencode as the third provider

Proves the seam generalises beyond "two CLIs that happen to look alike".
opencode is open source and exposes a real `permission.{allow,ask,deny}` surface
over `bash`/`edit`/`webfetch`, so the sandbox is achievable rather than hoped
for. Ships when an opencode-driven run completes a browser task with its tool
list showing only `endo`.

**Known before starting:** `opencode run -m <provider>/<model> --format json` is
the headless entry point; `OPENCODE_CONFIG_DIR` / `OPENCODE_CONFIG_CONTENT` /
`OPENCODE_PERMISSION` and `--pure` are the isolation levers;
`--dangerously-skip-permissions` is the flag that must never appear.
⛔ A standalone install and an npm install can shadow each other on `PATH` and
report different versions. Resolve the binary explicitly, the way
`resolveCursor()` already does for Windows shims.

### B — a gate that survives Portuguese

The structure-first rework above, in three parts: a structural marker in
`ax.js`, a folded multilingual lexicon, and the gate reading both. **This is the
release gate for A reaching daily use** — every provider added while the gate is
English-only widens the blast radius of the same missed button, and the real
browsing surface here is Portuguese. Build A first if you like; don't run it in
`normal` mode on a PT checkout until B is in.

### C — Antigravity via its `agy` CLI

The Google lane. Antigravity ships a headless CLI, `agy`, distinct from the
Electron IDE that carries the same name — a distinction this roadmap originally
got wrong, having inspected only the desktop bundle. It fits the existing
contract without adaptation:

| Contract need | `agy` flag |
| --- | --- |
| Headless single prompt | `-p` / `--print` / `--prompt` |
| Streamed events | `--output-format stream-json` |
| Model selection | `--model`, plus `--effort low\|medium\|high` |
| Session continuity | `--continue`, `--conversation <id>` |
| MCP wiring + audit | `mcp add\|remove\|list\|enable\|disable` |
| Restriction | `--sandbox` (terminal restrictions) |
| ⛔ Never | `--dangerously-skip-permissions` |

Two things to design around. `--print-timeout` defaults to **5m**, and a
multi-tab browsing task can exceed that, so it needs raising deliberately rather
than discovering the truncation mid-run. And the isolation lever is the open
question: opencode and Cursor both expose a config-directory override, and
whether `agy` honours an equivalent has to be **determined, not assumed** —
that determination is the first task, because the default config already enables
a headless browser server.

### D — the raw-API lane

OpenRouter, Google AI Studio, and opencode zen have **no agent loop to delegate
to**, so the bridge would have to become one: conversation state, tool-call
dispatch, retry, and streaming normalisation, per vendor. That is a new
subsystem, not a new `runX()`.

It also inverts the pitch. The README's badge says *API key: not required*, and
"most open-source browser agents need an API key and bill you per click" is the
project's stated reason to exist. Shipping D means saying plainly that the
subscription path is the **default** and the metered path is the **opt-in**, and
answering where a key is stored — `.endo-token` is `0600` and gitignored, and any
key store has to clear that same bar without ever reaching a command line or a
model's context.

### E — considered, not committed

Per-origin approval scoping (`gate.ts` keys approvals on tool name alone). Real
focus tracking, to narrow the `key: Enter` page-level fallback. macOS and Linux
autostart upstream, now that a `systemd --user` unit is known to work. Session
persistence across a bridge restart — deliberately absent today, since resuming
into a moved-on Chrome hands the agent stale refs.
