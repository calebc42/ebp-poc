# Harvest: what json-render teaches the ebp rebuild

**Status: Claude-authored reference sketch. Not code, not spec — the owner's
spec and implementation supersede this document wherever they diverge.**
Sketch policy: tables, wire shapes, and posed decisions; the decisions this
harvest needed were settled with the owner on 2026-10-06 and are recorded
inline in italics, the way the LiveView harvest records its resolutions.

**Sources & verification note.** Every json-render fact below was read this
session from the source checkout at `~/workspace/json-render` (commit
`fc2a696`, version 0.21.0) — cited as `file:line` under that root, with
`packages/` and `apps/web/` spelled out. Every ebp fact is from `SPEC.md` /
`contract.json` at ebp commit `84e0d66` (protocol 3, document 3.1.0-draft,
contract format 12, last amendment #196); Jetpacs facts are from
`jetpacs-poc` `14dc2816`, `ebp-kmp` `5089eb0`, `ebp.el` `e8b6eea`,
`ebp-compose` `cb21aa0`, `grove-native` `a7d6c81`, `orgzly-native`
`379d97f`. Nothing in this document is from memory of json-render's
documentation; where a judgment is mine rather than a sourced fact, it is
phrased as a recommendation or a question.

**Why json-render.** Vercel's json-render is the LLM-authored counterpart to
the agent-authored A2UI that ebp harvested in amendments #192–#196: a model
emits a flat JSON element map constrained to a catalog, the map streams as
RFC 6902 patches, per-framework registries render it, and a client-side state
model with `$state`, `$cond`, `$template`, `$computed`, `setState`, `watch`
and `visible` carries the interactivity. It makes the opposite bet from ebp
on exactly one axis — the renderer evaluates a small expression language — so
it is the sharpest available test of §23.2 and the W2 ruling. The harvest
sorts into: things ebp already independently converged on (§1 — take the
confidence), four things worth taking now (§3–§6), five things to file behind
the measurement gate (§7), and a warning about what must not transfer (§8).

**Two ground rules before any take:**

1. **Every take below is an amendment or a tooling change, never a drift.**
   A wire take lands with a `SPEC-CHANGES.md` row in the same commit (SPEC
   §2.2, §25) and ends here with the changelog line it needs. A tooling take
   names the repository that owns it and the gate that pins it.
2. **These shapes are envelope-agnostic.** json-render's wire is JSONL
   patches over an HTTP response or an AI-SDK message stream; ebp's is
   Content-Length-framed JSON-RPC 2.0. Every take is payload-level and
   re-homes onto `surface.update`, `surface.data` and the welcome unchanged.

---

## 1. Convergences — take the confidence, adopt nothing

Read next to the SPEC after #194–#196, json-render's core is mostly a second
spelling of what ebp already has. None of these rows needs importing.

| json-render | ebp / Jetpacs | Where |
|---|---|---|
| A catalog of allowed components; the model names intent, the host implements it (`apps/web/content/docs/registry.mdx:141` — "a core guardrail") | Negotiated `surface_profiles`, per-node `node_schema`, goldens; the Companion "does not compile an app catalog or interpret app-specific action names" | SPEC §10.2, §12; `jetpacs-poc/docs/ARCHITECTURE-POC3.md:21-22` |
| `{ "$state": "/ptr" }` — a JSON Pointer into a state document (`packages/core/src/props.ts:202-332`) | `bind` — member → RFC 6901 pointer into the surface data document, resolved at materialization | SPEC §13.8.3 |
| `repeat: { statePath, key }` with `$item` / `$index` (`packages/react/src/renderer.tsx:808-903`) | `template` beside `bind.children`, bound `key`, engine-injected `item`; identity by key, never index | SPEC §13.8.5, §14.3 |
| `defineRegistry` per renderer over shared Zod definitions (`packages/react/src/renderer.tsx:1092-1170`; `packages/shadcn/src/catalog.ts:29`) | `NodeComponentRegistry` / `ComposeComponentRegistry` / `MaterialComponents`, typed members generated from the contract | `ebp-compose/renderer/model/.../NodeComponent.kt`; `jetpacs-poc/companion/renderer/material3/.../MaterialComponents.kt` |
| Unknown type → `fallback` component or null (`renderer.tsx:674-680`) | Unknown or unadvertised `t` → the children as a plain column, no dispatch | SPEC §16.2; `Renderer.kt:614-627, 667-671` |
| Events decoupled from actions: a component emits `press`, the spec binds `on.press` to a named action (`packages/core/src/actions.ts:8-52`; changelog v0.5.0, `apps/web/content/docs/changelog.mdx:863-866`) | `on_*` hooks carrying ActionDescriptors with allowlisted dotted names | SPEC §14.1 |
| Devtools cost nothing when off (`packages/core/src/devtools-flag.ts:27`) | Profiler and flight recorder off by default | `jetpacs-poc/emacs/jetpacs-devtools.el:8-67` |
| A `SKILL.md` per package for coding agents (`skills/*/SKILL.md`) | Layered `AGENTS.md` files; the applet MCP serves them as resources and prompts | `jetpacs-poc/jetpacs-applet-mcp/jetpacs-applet-mcp.el:367-464` |
| `watch` on state paths fires actions on change, not on first render (`renderer.tsx:565-646`) | `jetpacs-on-state-change` and the automations interpreter, both evaluated in Emacs; `state.edge` triggers for device state | `jetpacs-poc/emacs/jetpacs-surfaces.el:1242-1258`; SPEC §21.6 |
| Standard component definitions behind a server-safe entry point (`packages/ink/src/server.ts`) | One contract plus format-2 manifests, projected by generators into Kotlin and Elisp and drift-checked | `ebp-compose/tools/gen-renderer-extension-vocabulary.py`; `jetpacs-poc/jetpacs-components/tools/check-projections.sh` |

*Where ebp is stricter, and should stay so.* json-render has no revisions,
acknowledgements or optimistic-update protocol — an edit sends `currentSpec`
and applies whatever patches return (`packages/react/src/hooks.ts:276-292`).
With two or more components `propsOf` collapses to `z.record(z.string(),
z.unknown())`, so per-component props are not validated at runtime outside
Jev (`packages/core/src/schema.ts:533-545`). The stream compiler silently
drops any line whose trimmed text it has already seen
(`packages/core/src/types.ts:872, 885-886`); `replace` creates a missing path
"for streaming tolerance" (`types.ts:610-649`); `validateSpec` has no cycle
code (`packages/core/src/spec-validator.ts:84-352`); and the React spec schema
omits `state`, `on` and `watch`, so `jsonSchema()` can never ask a model for
them (`packages/react/src/schema.ts:10-101`). Each of these is the price of
tolerating a model's partial output. ebp's input is a deterministic Emacs
builder, so §13.2's validate-everything-then-reject-atomically stays.

---

## 2. Before the takes: three kinds of wire field

The LiveView harvest's test still decides every field: **cosmetic** (ignoring
it loses polish), **QoS** (ignoring it changes rates, never outcomes), or
**constraining** (an old Companion that ignores it over-acts, so it needs a
catalog to gate on and a skip-the-whole-thing rule; SPEC §12 rule 3, §22.4).
Applied below: the device locale of take 3 is *presentation advice* — the
Companion reports a fact and Emacs decides, the `window.changed` class (§20.1.1
"presentation advice, never an event"), so it is additive with no gate. The
`visible_when` of take 4 is **constraining**: a Companion that ignored it
would show a node its author meant to hide, so it is a §22.4 member-gating
feature with an Emacs-side fallback, exactly as `data.template` and
`data.insert` were.

---

## 3. Take 1 — the agent catalog export (`catalog.prompt()` → `jetpacs_catalog`)

**The gap.** Jetpacs has every datum an authoring agent needs and no document
that joins them. `jetpacs_describe_node` reads only `contract.json`'s 49
core nodes and never the renderer-extension manifests, so the fifteen
`jetpacs.*`, `material3.*` and `jetpacs.design` nodes are invisible to an
agent (`jetpacs-poc/jetpacs-applet-mcp/jetpacs-applet-tooling.el:1107-1155`).
The format-2 manifests carry a single extension-level `description` and none
per node or member (`ebp-compose/tools/gen-renderer-extension-vocabulary.py:444-446`).
`jetpacs-defaction`'s `:doc` and `:args` are "EMACS-SIDE ONLY: nothing here
reaches the wire" (`jetpacs-poc/emacs/jetpacs-surfaces.el:876-888`). The
"agent catalog export" has sat on the open list since the A2UI plan.

**What json-render actually does (verified).** One Zod definition per
component feeds four uses — TypeScript types, the prompt, validation and JSON
Schema (`packages/core/src/schema.ts:83-105`). `catalog.prompt()`
(`generatePrompt`, `schema.ts:588-1153`) emits, in order: a system line; the
output format; an example built from the first two *real* components
(`:659-740`); state and list rules; `AVAILABLE COMPONENTS` as one line each —
`- Name: { p: type, opt?: type } - desc [accepts children; slots: a] [events:
press]` (`:824-854`, formatter `:1331-1430`); `AVAILABLE ACTIONS` with
built-ins tagged `[built-in]`; expression, visibility and watcher sections
only when the catalog uses them; a numbered rules list (`:1123-1150`).
Per-package `skills/<name>/SKILL.md` files carry YAML frontmatter (`name`,
a trigger-style `description`), Key Concepts, copy-paste recipes and a Key
Exports table (`skills/core/SKILL.md:1-35`), installed with `npx skills add`.

**The steal — proposed shape.** No wire change. An Elisp generator in the
applet MCP, `jetpacs-applet-catalog.el`, joins (a) the contract's core nodes,
universal attributes, field types and enums, (b) every installed manifest's
nodes rendered in the format-2 type algebra, (c) each node's Elisp builders
and their docstrings — the join `describe_node` already performs for core
nodes, (d) the action descriptor schema, builtins and their constructors, and
(e) an applet checkout's `jetpacs-defaction` registrations with `:doc` and
`:args`. It renders three committed, drift-checked files — a JSON catalog, a
`CATALOG.md` in json-render's one-line-per-node style, and
`skills/jetpacs-applet/SKILL.md` — and serves them as an MCP tool and two
resources. The manifests gain an optional, documentation-only `description`
per node and member that never reaches a projection or the schema digest.

```text
- material3.assist_chip: { label: text(1..4096), icon?: identifier, on_tap?: action }
  - A compact action chip. [targets: app] [events: on_tap] [elisp: jetpacs-material3-assist-chip (label &key icon on-tap)]
```

**Why Elisp and not Python.** The only non-JSON input is builder signatures
and docstrings, which the static tooling already extracts with the Lisp
reader and whose regex parsing `jetpacs-applet-mcp/AGENTS.md` forbids; one
generator gives the live MCP answer and the committed files one code path.

**Decisions this take needed:** whether an export is wanted at all, and
whether it is Python or Elisp. *(Settled 2026-10-06: in scope; Elisp, in the
applet MCP, with a sh wrapper mirroring `tools/check-material3-projections.sh`
so CI pins it.)*

**Changelog line it needs:** none — no normative text changes. The
`ebp-compose` generator's `description` passthrough is a tooling release.

---

## 4. Take 2 — devtools that see the data path

**The gap.** `jetpacs-devtools.el` is a profiler, a bounded flight recorder
and an inspector that pretty-prints the authored IR and pushes it to the phone
through the Tier-0 renderer. It advises `jetpacs-shell--build` and
`ebp-client-surface-update` only (`jetpacs-poc/emacs/jetpacs-devtools.el:288-291,
593-598`), so `surface.data` patches, their results, `event.action` dispatch,
`state.changed` fan-out and the data document itself are never recorded, and
the materialized tree is never shown. The byte measurements behind #194–#196
were taken with an ad-hoc `process-send-string` advice. The Companion has no
inspector at all (`jetpacs-poc/companion/.../JetpacsNavKey.kt`: Pairing,
Catalog, Surface, Settings). W8 keeps devtools off the wire, so this is
Emacs-side work.

**What json-render actually does (verified).** A vanilla-TypeScript shadow-DOM
panel with six tabs — Spec (tree, raw, validation issues, a switcher across
generations), State (a flat JSON-Pointer list with inline edits written
through `store.set`), Actions, Stream, Catalog and Pick (a DOM picker mapping
clicks to spec keys through `data-jr-key`) — fed by typed events
`spec-changed`, `state-set`, `action-dispatched`, `action-settled`,
`stream-patch`, `stream-text`, `stream-usage`, `stream-lifecycle` in a ring
buffer of 500 (`packages/devtools/src/event-store.ts:46-47`). The stream is
tapped by `tee()` on the server (`packages/devtools/src/stream-tap.ts:40-58`)
and by scanning message parts on the client; renderers add `data-jr-key` only
while `markDevtoolsActive()` is held, so the panel costs nothing when absent
(`packages/core/src/devtools-flag.ts:27`).

**The steal — proposed shape.** One bounded ring of typed entries
(`push`, `patch`, `remove`, `result`, `action`, `state`, `frame`, `failure`)
with cheap metadata always and payloads only under the existing
`jetpacs-devtools-recording` toggle (the amendment-#74 discipline that keeps
action args out of records); advice on `ebp-client-surface-data`,
`jetpacs--dispatch` and `jetpacs--on-state-changed` beside the existing push
advice; a per-method wire byte log on `jsonrpc-event-hook` (present on the
Emacs 30.1 floor, jsonrpc 1.0.25) behind its own toggle; the inspector gains
the last-sent document and the materialized tree
(`jetpacs-data-materialize … :strip t`) beside the authored IR, and an
edit-the-document-then-patch command that sends the diff through
`jetpacs-shell-data-patch` — json-render's State editor, over ebp's own
`surface.data`. Two `special-mode` views, wire log and action log, reach the
phone through the Tier-0 buffer path like `*jetpacs-devtools*` does today.

**What this take deliberately does not copy.** The DOM picker: a Companion
overlay mapping a touch to an identity path is Companion-side work with no
Emacs-side equivalent yet, and nothing in the current pilots needs it.

**Decisions this take needed:** Emacs-only or Companion too. *(Settled
2026-10-06: Emacs-only first; the Companion inspector waits for a need.)*

**Changelog line it needs:** none (W8). Side effect worth recording: the
Elisp `data.golden` replay `test/jetpacs-data-test.el` joins `run-tests.sh`,
which does not run it today.

---

## 5. Take 3 — device locale hints (`$format`'s benefit without `$format`)

**The gap.** Emacs formats every date, clock and number the Companion shows
(SPEC §17.2 `date_stamp` is "presentation data, not a clock instruction";
#193 made `date_stamp.month` a display string; `jetpacs-dates.el` keeps
English month abbreviations so output is identical across host locales). But
the welcome carries no fact about the device's locale, clock cycle or time
zone — §10.2's table lists `window` nowhere and no `locale` anywhere, and the
only zone-shaped member in the whole contract is the `timezone.changed`
trigger's fire data (`contract.json` `trigger_types`). So Grove's "today"
(`grove-native/elisp/grove-document.el:588`) is the desktop's today, and a
`time_button.label` is 12- or 24-hour by Emacs's convention, not the phone's.

**What json-render actually does (verified).** The `@json-render/directives`
package supplies `$format` (`date | currency | number | percent` with
`locale`, `currency`, `notation`, `style: "relative"`), `$math`, `$concat`,
`$count`, `$truncate`, `$pluralize`, `$join` and an i18n `$t`
(`packages/directives/src`), each resolved by the renderer inside the prop
walk (`packages/core/src/props.ts:297-306`), with the *browser's* locale as
the default. The benefit is real — the device knows its own clock and zone —
and the mechanism is exactly what §23.2 forbids: a peer-supplied directive the
host interprets.

**The steal — proposed shape.** Keep the formatting in Emacs; move the
*facts* across, on the `window.changed` precedent (§20.1.1: mirrored in the
welcome so the first snapshot is authored for the right class; re-sent only
when it changes; "presentation advice, never an event"):

```json
"locale": {"tag": "de-DE", "hour_cycle": "h23", "time_zone": "Europe/Berlin"}
```

An OPTIONAL `locale` member of the welcome and a `locale.changed`
notification (Companion → Emacs, core, SYNCING and READY) carrying the same
object: `tag` a BCP 47 language tag, `hour_cycle` one of `h12` | `h23`,
`time_zone` an IANA zone identifier. Emacs MUST treat it as presentation
advice and MUST fall back to its own locale when it is absent. Jetpacs gives
builders `jetpacs-dates-today`, a device-zone civil date, and
`jetpacs-dates-format-clock`, so Grove's today and every time label follow the
phone. Emacs has no CLDR, so this take buys the clock cycle and the civil
day, not localized month names — the honest scope.

**Classification.** Presentation advice, like `window`; additive within
protocol 3; no capability, no feature. Privacy: `tag` plus `time_zone` is
coarse but identifying; it travels only inside the authenticated session,
like the §20.1 device report.

**Decisions this take needed:** take it or leave formatting desktop-bound.
*(Settled 2026-10-06: take it, as amendment #197.)*

**Changelog line it needs:** `§10.2, §20.1.2 (new), §11: the welcome gains an
OPTIONAL locale member {tag, hour_cycle, time_zone} mirrored from a new
locale.changed Companion notification; presentation advice on the
window.changed model; Emacs formats, the Companion reports.`

---

## 6. Take 4 — conditional nodes (`visible` as a bounded predicate)

**The gap.** Under #194 a changed *value* is a patch, but a changed *shape*
is still a whole snapshot. Grove's planning feedback node is chosen by kind
and the handler patches only when its `--planning-shape` is unchanged
(`grove-native/elisp/grove-document.el:643-676, 976-1011`); Orgzly's outline
template writes `""` for a missing value and drops the content preview row
because a template cannot vary shape per item
(`orgzly-native/emacs/apps/orgzly/orgzly-ui.el:102-109, 146-154`). Both are
the same missing primitive: show this node when this value holds.

**What json-render actually does (verified).** `visible` on an element is a
boolean, a `SingleCondition`, an array of them (implicit AND), or `{$and}` /
`{$or}` (`packages/core/src/types.ts:108-170`). A `SingleCondition` names
`$state` (or `$item` / `$index` in a repeat) plus at most one of `eq, neq,
gt, gte, lt, lte` and an optional `not: true`; no operator means truthiness;
numeric operators apply to numbers only; the right-hand side may itself be
`$state` (`packages/core/src/visibility.ts:237-364`). A repeat container's
`$item` condition is split into a container gate and a per-item filter
(`visibility.ts:116-143`). Elements are keyed by their map key, so hiding one
never shifts a sibling's identity.

**The steal — proposed shape.** A universal attribute, admitted only under
`data.model` with a new member-gating feature `data.visible`:

```json
{"t": "text", "key": "warn", "bind": {"text": "/planning/feedback"},
 "visible_when": [{"path": "/planning/feedback_kind", "op": "eq", "value": "warning"}]}
```

`visible_when` is a non-empty array of at most 8 predicates meaning logical
AND — flat, no OR, no nesting, the §21.3 `when` discipline. A predicate is
`{path, op, value?}`: `path` a §13.8.2 pointer (relative inside a template
item, as `bind`); `op` one of `present`, `absent`, `empty`, `non_empty` (no
`value`), `eq`, `neq` (a scalar `value`), `lt`, `lte`, `gt`, `gte` (a number).
Evaluation is total and deterministic: an unreachable pointer makes `absent`
hold and everything else fail; a type mismatch fails, never rejects. The
Companion evaluates before resolving the node's bindings and omits a hidden
node and its subtree from the materialized `children`; every `surface.data`
re-materializes, so a `set` on a flag shows or hides without a snapshot.

**Why this is not the `$cond` ebp refused.** It is the class of §21.7 state
predicates and the §14.6.1 dropdown filter: a bounded exact comparison over
inert data, O(predicates × pointer segments), no I/O, terminating, with no
value *produced* — nothing is derived, formatted, filtered or transformed, so
W2 stands. The rules that keep it there: only on an element of a `children`
array (never a root, a slot member, a `stale_spec` node or a template root —
Emacs filters arrays); every element of an array containing a conditional node
MUST carry `key`, so identity never falls back to position while siblings
appear and disappear; and no stateful node, `editor`, `variant_host`, `id` or
`capture_fields` beneath a conditional node, so §13.6 drafts are untouched in
this version. json-render's `$or`, `not`, truthiness and `$state`-valued
right-hand sides are deliberately not taken.

**Decisions this take needed:** gate it behind measurement, plan it, or rule
it out. *(Settled 2026-10-06: plan it now, as amendment #198, with the
#194–#196 Emacs-side fallback — evaluate in Emacs and omit the node when the
feature is absent.)*

**Changelog line it needs:** `§13.8.6 (new), §13.8.3, §16.5, §16.1, §17.1,
§4.5, §22.1, §22.4, §24.5, §24.6: the universal attribute visible_when — a
flat AND of bounded predicates over the data document, evaluated at
materialization before bindings; keyed children only; feature data.visible;
limits max_visible_predicates_per_node 8 and _per_snapshot 10000.`

---

## 7. Behind the measurement gate

### 7a. Node-tree patching (`surface.patch`)

json-render patches `/elements/<key>` because its map is flat and every
element is independently addressable (`packages/core/src/types.ts:175-183`,
"optimized for LLM generation"). ebp patches only the document. The LiveView
harvest §7 parked general tree patching; #194–#196 and take 4 remove most of
the need. **The reserved sentence, unchanged:** *a future negotiated
capability may patch the node tree by identity; until measurement demands it,
trees ride in snapshots.* Take 2's recorder counts how often
`jetpacs-app-defer-patch` still falls back to a re-push after #198 — that
count is the gate.

### 7b. Companion-local echo (`$bindState` / `setState`)

json-render's `useBoundProp` writes the state store on every keystroke and
every other `$state` reader re-renders (`packages/react/src/hooks.ts:454-468`;
`packages/react/src/contexts/actions.tsx:159-362`). #194 deferred the
A2UI-shaped echo by the owner's intent; json-render adds one warning to that
analysis: once client state exists, `watch` (`renderer.tsx:565-646`) and
`onSuccess` / `onError` chains (`packages/core/src/actions.ts:205-244`) follow
within two releases, and that is the mini-runtime W2 exists to refuse.
**Reserved sentence:** *a `data.echo` feature under which the Companion
materializes a stateful node's draft at its bound pointer for sibling
bindings until the next accepted revision, with race rules and a §3.1
durable-domain change, is explored only when a pilot needs it.*

### 7c. Decision-model composition (Jev)

json-render's `experimental_composeSpec` has the app supply atomic candidates
— configured component instances with props, bindings and allowed actions —
and a choice model (TypeSafe AI's Jev, `apps/web/content/docs/jev.mdx:14`)
select membership, parent and order; code assembles and validates an ordinary
spec and the model never writes JSON or prose
(`packages/core/src/experimental-compose.ts:49-77, 524-571`;
`experimental-composition-batch.ts:88-253`). The evaluator contract is
`{state, questions} → {answers: {q: {choice, confidence}}}`, the same
`/v1/systemone` shape the owner's `vulpea-jev` already speaks to a local
JevK5. The mapping onto Jetpacs is clean — candidates are authored IR nodes,
the result passes the shell gates, the Companion never knows — and it is
**not planned: the owner dropped Jev from every implementation plan on
2026-10-06.** Recorded here only so the mapping is not rediscovered.

### 7d. A web Companion as an MCP App

`@json-render/mcp` registers a `render-ui` tool whose result renders in a
sandboxed `ui://` iframe inside Claude, ChatGPT, Cursor and VS Code
(`packages/mcp/src/server.ts:31-64, 73-104, 131`). For ebp that is a browser
Companion, which `WEBSOCKET-transport-kit.md` already imagined over §5.5.
Nothing to do until a browser Companion exists.

### 7e. Progressive streaming of a surface

json-render streams one RFC 6902 patch per line so the UI fills in as the
model types (`packages/core/src/types.ts:834-930`; changelog v0.4.0). Emacs
builders are fast, §22.2 latest-wins already drops superseded snapshots, and
no pilot has shown a surface too slow to author whole. No reserved sentence.

---

## 8. What must not transfer

json-render stands on a model whose output must be *tolerated*; ebp stands on
a builder whose output must be *right*. Everything below follows from that.

- **Renderer-side expressions.** `$cond` / `$then` / `$else`, `$template`
  interpolation, `$computed` registered functions and the directives package
  (`$format`, `$math`, `$concat`, `$pluralize`, `$t`) are each the Companion
  interpreting a peer-supplied expression (`packages/core/src/props.ts:120-306`).
  §23.2 forbids it; W2 keeps transforms off the wire; #194 already declined
  A2UI's `call` nodes and `checks`. Take 4 admits a comparison, never a value.
- **A client `StateStore` as the state authority.** `setState`, `pushState`,
  `removeState` and the Redux / Zustand / Jotai / XState adapters
  (`packages/core/src/state-store.ts:59-164`) make the client the owner of
  application state. §1 and §3 give that to Emacs; the Companion keeps drafts
  and delivery records only.
- **`watch` on the Companion.** Reaction logic belongs in Emacs
  (`jetpacs-on-state-change`) or in the automations interpreter's closed
  31-operator language, which only Emacs evaluates
  (`jetpacs-poc/jetpacs-automations/lisp/jetpacs-automation-model.el:66-72`).
- **Tolerant rendering of partial or invalid specs.** Missing children render
  null while loading, unknown types fall back, `replace` creates paths, and a
  per-element error boundary swallows a crash
  (`packages/react/src/renderer.tsx:127-162, 698-711`). ebp validates the
  whole request before any state change and rejects atomically (§13.2); the
  only tolerance it keeps is §16.2's children fallback for unadvertised types.
- **Codegen from specs.** `@json-render/codegen` exports a spec as framework
  source; `DETERMINISM.md:23-24` forbids "a parallel AST or code generator" —
  the builder's plist is the one IR.
- **YAML wire and edit modes.** `yaml-spec`, `yaml-edit`, `yaml-patch` and
  unified-diff fences exist so a model can emit less (`packages/yaml/src/transform.ts:13-17`).
  No model emits ebp frames; Content-Length JSON-RPC stays.
- **Model-chosen element keys.** json-render lets the model name elements and
  mints `el-N` / `node_N` itself (`types.ts:725`; `experimental-compose.ts:654`).
  ebp identity is authored (`key` > `id` > position, §16.1) and, under
  templates, bound — never generated.
- **Living without revisions.** The absence of revisions, acknowledgements and
  `revision_seen` is json-render's single largest gap against §13 and §14.5;
  it is the reason its edits cannot be made safe against a stale client.

---

*Practical upshot: takes 1 and 2 are tooling in `jetpacs-poc` and
`ebp-compose` with no normative text; takes 3 and 4 are amendments #197 and
#198, each additive within protocol 3 and each with its Emacs-side fallback;
§7 is five parked entries, one of them closed by the owner's decision. Fold
the shapes into the hand-written sections directly — this document is the
quarry, not the wall.*
