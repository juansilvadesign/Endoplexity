# Tasks

The living checklist. Milestone definitions and the reasoning live in
**[`ROADMAP.md`](ROADMAP.md)**; setup and the safety model live in
**[`README.md`](README.md)**.

<!-- PHASE-BLOCK: mirrored verbatim in TASKS.md and ROADMAP.md — edit both or neither -->
> ### Current phase: **A — opencode as the third provider**
> **0** shipped (install + verification). **B** is the release gate for A
> reaching daily use. **C** follows A on the same seam. **D** is deliberately
> behind the CLI lane. Last reviewed **2026-08-28**.
<!-- /PHASE-BLOCK -->

> ⛔ **Nothing below ships on a documentation claim.** Every sandbox assertion
> needs an observed refusal, and every provider needs its tool list read back.
> That cuts both ways: this checklist once recorded a provider as having no
> headless CLI on the strength of one install location, and was wrong.

---

## Completed — 0: install and verify ✅

- [x] `npm install` on node 24. ⚠️ The declared `engines` floor is real for the
      no-build-step TypeScript, but node 22.18+ strips types too — so a machine
      "below spec" may still run it, and a machine on 22.17 silently won't.
- [x] `npm test` — **175/175** pass. The README's "156 tests" badge is stale.
- [x] Bridge on WSL2, Chrome on Windows; Windows→WSL loopback on `:8787` proven
      with a throwaway listener *and* the real bridge, not assumed from docs.
- [x] `systemd --user` unit, enabled and active, `Restart=on-failure` proven by
      SIGKILL rather than by reading the unit file.
- [x] Unit runs bare `nvm use` so it tracks node 24 as a **major** via `.nvmrc`,
      and exits loudly if 24 is missing rather than degrading in silence.
- [x] Asserted the running process is really node 24 via `/proc/<pid>/exe`.
- [x] Origin pin probed with known-bad inputs: `401` for no origin, for a
      hostile origin, and for a wrong extension id; `101` only for the id pinned
      by the manifest `key`.
- [x] `/mcp` token gate: `401` missing → `401` wrong → `200` real.
      `.endo-token` and `.endo-mcp.json` both `0600`, both gitignored.
- [x] Sandbox confirmed: the spawned CLI's tool list is exactly `ToolSearch` +
      the 14 `mcp__endo__*`, with `apiKeySource: "none"`.
- [ ] **Owner step — load `extension/` unpacked** at `chrome://extensions`
      (Developer mode → Load unpacked). It must resolve to the id pinned by the
      manifest `key`, or the bridge refuses the WebSocket upgrade.
- [ ] Optional on Linux: `loginctl enable-linger "$USER"` so the bridge starts at
      boot rather than at first login.

---

## A — opencode as the third provider

Ships when an opencode-driven run completes a real browser task **and** its tool
list shows only `endo`.

### A1 — resolve the binary explicitly
- [ ] A standalone install and an npm install can both provide `opencode` at
      different versions, with `PATH` order deciding silently. Pick one
      deliberately in a `resolveOpencode()`, mirroring `resolveCursor()`.
- [ ] Record which one the bridge spawns, so upgrading the *other* one never
      looks like a silent no-op.

### A2 — build the sandbox
- [ ] Point the run at a bridge-owned config dir (`OPENCODE_CONFIG_DIR`, or
      `OPENCODE_CONFIG_CONTENT` to avoid writing a file at all), outside the repo
      and outside the operator's own config.
- [ ] Deny `bash`, `edit`, `webfetch` via `permission.deny`; allow only the
      `endo` MCP tools. `OPENCODE_PERMISSION` is the env-var form.
- [ ] Add `--pure` (no external plugins). ⛔ **Never** `--dangerously-skip-permissions`.
- [ ] Seed the `endo` MCP server at `http://127.0.0.1:<port>/mcp?token=…`,
      written `0600`, token never on a command line.

### A3 — prove the isolation (the acceptance test, not a formality)
- [ ] ⛔ Read the tool list back from a real run and assert **only `endo`** is
      present. A developer's opencode config commonly enables MCP servers with
      write access to live production systems; unsandboxed, a browser-driving
      agent inherits every one of them.
- [ ] Verify `HOME`/`XDG_CONFIG_HOME` can't leak the operator's config back in.
      This is exactly how the Cursor path failed: the config-dir override was set
      and `mcp.json` still resolved off `homedir()` — 27 tools across 7 servers.
- [ ] **Observe a refusal.** Ask the sandboxed agent to run a shell command and
      record that it could not. A passing config file is not evidence.

### A4 — the runner
- [ ] `runOpencode(prompt, model, onEvent, resume?)` returning a `ChildProcess`,
      matching `runClaude`/`runCursor` exactly.
- [ ] Headless entry point is `opencode run <message> -m <provider>/<model>
      --format json`; export the argv as `opencodeArgs()` so a test can assert on
      it without spawning, as both existing providers do.
- [ ] Normalise opencode's JSON events into the shape the panel already renders;
      guard non-object JSON the way `claude.ts` and `cursor.ts` both do.
- [ ] Briefing: no `--append-system-prompt` equivalent, so ride in front of the
      prompt like `cursor.ts`, and drop it on resume.
- [ ] Session continuity — map the bridge's `resume` onto an opencode session id.

### A5 — wire it up
- [ ] Register the model ids in `MODELS` in `bridge/src/index.ts` (that map is
      also the allowlist — an unknown id must stay refused, not passed through).
- [ ] Add the options to the panel's `#model` dropdown in `extension/panel.html`.
- [ ] Handle a missing binary as a clean panel-visible error, not a bare `ENOENT`.
      ⚠️ The two Cursor entries already fail this way wherever `cursor-agent`
      isn't installed — fix both while here.

### A6 — tests
- [ ] `bridge/test/opencode.test.ts` asserting argv, event normalisation, the
      resume branch, and that the deny list is present in the emitted config.
- [ ] Keep `node --test`, no framework, no build step.

---

## B — a gate that survives Portuguese

Structure first, lexicon second, union always on. **Release gate for A reaching
daily use.** See ROADMAP § *the gate is language-bound by construction*.

### B1 — give the gate something structural to read
- [ ] `extension/ax.js` emits `@ref [role] "name"` + a fixed flag set
      (`required`, `disabled`, `checked`, `expanded`, `selected`, `invalid`).
      **Nothing carries `type=submit` or form membership** — add a marker
      (e.g. a `submit` flag) for a button that is its form's default submit.
- [ ] Emit it only for qualifying elements. Page returns were cut **3.8×** by
      sending deltas; a flag on every actionable line spends that back.
- [ ] ⛔ This changes the snapshot wire format, which **three** parsers read: the
      full serializer, the `UNCHANGED` `+`/`-` delta, and `gate.ts`'s `LINE`
      regex. Change them together or the gate goes quietly blind.

### B2 — fold the text before matching
- [ ] NFD-normalise and strip diacritics before the lexicon runs, so
      `confirmação`, `inscrição`, and `não` are matchable at all.
- [ ] Match **stems, not whole words**: `\bremove\b` never matches `remover`, and
      `enviar`/`envie`/`enviado` all need to hit.

### B3 — the multilingual lexicon
- [ ] PT-BR first, since that is the real browsing surface: `enviar`,
      `confirmar`, `pagar`, `comprar`, `finalizar`, `concluir`, `excluir`,
      `apagar`, `remover`, `cadastrar`, `assinar`, `agendar`, `encomendar`.
- [ ] Keep the union always on — no page-language detection deciding which list
      loads. A missing `lang` attribute must not be able to disable gating.
- [ ] Re-check the existing false-positive carve-outs in the new languages:
      `accept`/`agree` were removed because every cookie wall matched them and
      blocked the page underneath. `Aceitar`/`Concordar` will do the same.
- [ ] Watch for the reverse: PT `aplicar` ("Aplicar filtros") is the same benign
      match EN `apply` already is.

### B4 — the gate itself
- [ ] `check()` gates on the structural marker **first**, lexicon second.
- [ ] Keep the `key: Enter` page-level fallback, now firing on either signal.
- [ ] Keep failing closed: an unrecognised mode still falls back to `normal`.

### B5 — tests and tuning
- [ ] Extend `bridge/test/gate.test.ts`: a PT submit button gates; an accented
      label gates; a PT cookie wall does **not**; a structural submit with an
      unlisted-language label still gates.
- [ ] ⛔ Add a **known-bad** case that fails before the change and passes after —
      otherwise "the fix" and "the assertion got looser" are indistinguishable.
- [ ] Tune against real sites, per the existing `gate.ts` note ("Phase 6 tunes
      this against 3 real job sites instead of guessing further now") — include
      at least one PT-BR checkout or form.
- [ ] Update the README's safety section: it currently states the gate matches
      "a list of English words", which this milestone makes false.

---

## C — Antigravity via its `agy` CLI

The Google lane. `agy` is the headless CLI, distinct from the Electron IDE of the
same name; the IDE bundle carries no agent entry point, which is why this was
briefly and wrongly recorded as impossible.

### C1 — find the isolation lever (do this first)
- [ ] ⛔ **Determine whether `agy` honours a config-directory override**, the way
      opencode and Cursor do. Everything else in C depends on the answer, and the
      default config already enables a **headless Playwright browser server** —
      an unsandboxed run hands a browser agent a second, invisible browser
      outside the panel and outside the approval gate.
- [ ] If there is no config-dir override, evaluate `--sandbox` and the `mcp
      disable` subcommand as the substitute, and say plainly which boundary is
      actually holding.
- [ ] Confirm where the OAuth credential lives and that the sandbox does **not**
      require copying it anywhere new.

### C2 — the runner
- [ ] `runAgy(prompt, model, onEvent, resume?)` on the same contract.
- [ ] `-p` for the single prompt, `--output-format stream-json` for events, and
      `--model` for selection (`--effort low|medium|high` maps to the panel's
      reasoning tier if it exposes one).
- [ ] Resume via `--conversation <id>`; `--continue` resumes the most recent,
      which is the wrong semantics for a bridge holding several sessions.
- [ ] ⚠️ Raise `--print-timeout`. It defaults to **5m**, and a multi-tab browsing
      task can exceed that — discovering it as a truncated run mid-task is the
      expensive way to learn it.
- [ ] ⛔ **Never** `--dangerously-skip-permissions`.

### C3 — isolation proof and wiring
- [ ] Same A3 bar, no discount: `agy mcp list` shows **only `endo`**, and a shell
      command is observed being refused.
- [ ] Register model ids in `MODELS`, add them to the panel dropdown, and handle
      a missing binary as a clean panel-visible error.
- [ ] `bridge/test/agy.test.ts` covering argv, event normalisation, the resume
      branch, and the raised print timeout.

---

## D — the raw-API lane

No agent loop to delegate to, so the bridge becomes one. A subsystem, not a
`runX()`. Deliberately behind the CLI lane.

- [ ] **D1** — decide the framing first: the README badge says *API key: not
      required*, and that is the project's stated reason to exist. Say plainly
      that subscription is the default and metered is opt-in, or don't ship D.
- [ ] **D2** — an in-bridge tool-calling loop: conversation state, tool dispatch
      against the same 14 tools, retry, and turn accounting.
- [ ] **D3** — key storage clearing the `.endo-token` bar: `0600`, gitignored,
      never on a command line, never in a model's context. One store, many
      providers.
- [ ] **D4** — per-vendor streaming normalisation into the panel's event shape:
      OpenRouter, Google AI Studio, opencode zen.
- [ ] **D5** — cost reporting. The README quotes `$0.0959 / 112,064 tokens / 10
      turns` as *equivalent* spend on a subscription. On a metered key that
      number stops being hypothetical and should be shown as real.
- [ ] **D6** — the gate must behave identically here. The approval policy lives
      in the bridge precisely so the loop can change without weakening it.

---

## E — considered, not committed

- [ ] Per-origin approval scoping — `gate.ts` keys approvals on tool name alone,
      because labels come from serialised text that carries no origin.
- [ ] Real focus tracking, to narrow the page-level `key: Enter` fallback.
- [ ] Upstream macOS/Linux autostart, now that a `systemd --user` unit is known
      to work.
- [ ] Session persistence across a bridge restart — absent **on purpose** today:
      resuming into a Chrome that has moved on hands the agent stale refs.
- [ ] Real ATS comboboxes (usually not `<select>`) need a click-then-click flow.
