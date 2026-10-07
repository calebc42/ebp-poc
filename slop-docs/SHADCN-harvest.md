# Harvest: what shadcn/ui teaches the ebp rebuild

**Status: Claude-authored reference sketch. Not code, not spec — the owner's
spec and implementation supersede this document wherever they diverge.**
Sketch policy: tables, shapes, and posed decisions; the decisions this
harvest needed were settled with the owner on 2026-10-07 and are recorded
inline in italics, the way the LiveView and json-render harvests record
their resolutions.

**Sources & verification note.** Every shadcn fact below was read this
session from the source checkout at `~/workspace/ui` (commit `e8c3143`,
CLI package version 4.21.3, 2026-10-07) — cited as `file:line` under that
root, with `packages/`, `apps/v4/` and `skills/` spelled out. Every ebp fact
is from `SPEC.md` / `contract.json` at ebp commit `dee4920` (protocol 3,
document 3.1.0-draft, contract format 12, last amendment #198); Jetpacs
facts are from `jetpacs-poc` `c3785a51` and `ebp-compose` `146dda1`.
Nothing in this document is from memory of shadcn's documentation; where a
judgment is mine rather than a sourced fact, it is phrased as a
recommendation or a question.

**Why shadcn.** The two previous harvests tested the wire: LiveView's
server-driven delta model and json-render's client-evaluated expression
model. shadcn/ui tests something ebp has not been measured against — the
*tooling around a catalog*. It is not a renderer and not a protocol. It is a
registry of source items (`packages/registry/src/registry/schema.ts:179`),
a CLI that resolves, fetches, transforms and installs them
(`packages/shadcn/src/commands/add.ts`), an MCP server that lets a coding
agent search and view them (`packages/shadcn/src/mcp/index.ts:59-180`), a
skill with rules and evals (`skills/shadcn/`), and a docs site that renders
every component from the same index. ebp's Jetpacs side has a catalog, a
skill and an MCP since the json-render take 1 (`jetpacs-applet-mcp/
jetpacs-applet-catalog.el`), so shadcn is the sharpest available comparison
for what that tooling still lacks. The harvest sorts into: convergences
(§1), the field-kind test (§2), three takes worth taking now (§3–§5), three
entries behind a gate (§6), and what must not transfer (§7).

**Two ground rules before any take:**

1. **Every take below is a tooling change, never a wire change.** Nothing
   here adds a method, member, capability or feature to the SPEC. W2 keeps
   view definitions off the Companion and W8 keeps devtools off the wire
   (`W-REGISTER.md:22, 25`); the same reasoning keeps a registry off it. A
   tooling take names the repository that owns it and the gate that pins it.
2. **The Companion compiles no app catalog.** It "does not compile an app
   catalog or interpret app-specific action names"
   (`jetpacs-poc/docs/ARCHITECTURE-POC3.md:21-22`), and renderer extensions
   are "injected at its implementation boundary" (SPEC §16.2.1,
   `SPEC.md:3148`). So where shadcn ships *source into a project*, Jetpacs
   can ship Elisp applets into Emacs, and extensions are **declared
   requirements that are checked, never fetched code**.

---

## 1. Convergences — take the confidence, adopt nothing

| shadcn | ebp / Jetpacs | Where |
|---|---|---|
| One registry drives the CLI, the MCP server (seven tools, `packages/shadcn/src/mcp/index.ts:59-180`), the `docs` command (`packages/shadcn/src/commands/docs.ts:63` reads `meta.links`) and the skill (`skills/shadcn/SKILL.md`) | One contract plus three format-2 manifests drive `jetpacs-catalog.json`, `CATALOG.md`, the generated `SKILL.md` and the `jetpacs_catalog` MCP tool | `jetpacs-poc/jetpacs-applet-mcp/jetpacs-applet-catalog.el:592, 703`; `jetpacs-applet-mcp.el:174-188` |
| The zod schema is the source of truth and the public JSON Schema copies are kept in sync *by hand* (comment above `packages/registry/src/registry/schema.ts:81`) | Kotlin and Elisp vocabularies are *generated* from the manifests and drift-checked (`--check`) | `ebp-compose/tools/gen-renderer-extension-vocabulary.py`; `jetpacs-poc/jetpacs-components/tools/check-projections.sh` |
| A frozen corpus of real payloads with line-diff snapshots: "Never edit them: change the output only on purpose, and review the snapshot diff" (`packages/registry/src/utils/transformers/transform-corpus.test.ts:21-27`) | Goldens: JSON-lines fixtures replayed by every endpoint, regenerated only with an amendment row | SPEC §24.5 (`SPEC.md:5816`); `goldens/` |
| Component sources carry `cn-*` marker classes; a style map compiled at registry-build time replaces them (`packages/shadcn/src/styles/create-style-map.ts:17`) | `jetpacs.design` names closed `component-style-slot`s (`text.body`, `card.filled`, …) that a `design_scope` style targets | `jetpacs-poc/jetpacs-components/renderer-extensions/jetpacs-design.json:166, 275` |
| `components.json` is `.strict()` (`schema.ts:28-62`) | Manifests are closed mappings; unknown keys are rejected by the generator | `gen-renderer-extension-vocabulary.py` |
| A `registry:theme` item is `cssVars.light` / `cssVars.dark` records (`apps/v4/registry/themes.ts:7`) | `theme.set` carries thirteen roles plus `dark` and `syntax` | SPEC §18.4 (`SPEC.md:4079`); `contract.json:3051` |
| Items carry a `description` the agent reads (`schema.ts:158-178`) | Format-2 manifests carry a documentation-only `description` that never reaches a projection or the schema digest | `ebp-compose/tools/gen-renderer-extension-vocabulary.py:128-141` |

*One difference inside a convergence, recorded so it is not mistaken for a
gap.* shadcn resolves styles at build time so no runtime ever interprets a
style map. Jetpacs resolves `design_scope` on the Companion. That is a
*lookup* of a named style by a node that references it, not evaluation of a
peer-supplied expression; SPEC §23.2 (`SPEC.md:5548`) is untouched. No
change recommended.

---

## 2. Before the takes: the field-kind test

The LiveView harvest's test decides every wire field: cosmetic, QoS or
constraining. This harvest has no wire field to classify. The one
wire-adjacent question it raises — *does an applet installed from a registry
need a feature advertisement?* — is answered no: the registry delivers Elisp
to Emacs, and Emacs already "MUST gate every emitted node, builtin,
constraining feature, and renderer extension against the target profile"
(SPEC §10.2, `SPEC.md:1591`). An installed applet that requires an extension
the connected Companion does not advertise is gated at push time exactly as
a packaged one is (`jetpacs-apps-missing-extensions`,
`jetpacs-poc/emacs/jetpacs-apps.el:332`). So every take is tooling and
lands without a `SPEC-CHANGES.md` row.

---

## 3. Take 1 — examples as first-class catalog entries

**The gap.** Both catalogs name every member and no catalog shows a single
use. `jetpacs-catalog.json` joins the contract, the manifests and the Elisp
builders (`jetpacs-applet-catalog.el:263-329`) and emits one json-render-style
line per node (`:429`), but no node carries an example form or a built tree.
The raw material exists three times over: the component-catalog applet keeps
a canonical document per component (`jetpacs-poc/jetpacs-component-catalog/
lisp/jetpacs-component-authoring.el:1197-1255`), the Material 3 catalog pages
keep `:examples` whose `:build` is a nullary pure function returning the node
(`jetpacs-poc/emacs/apps/m3-catalog/jetpacs-m3-core.el:134`), and every
manifest ships a `.golden` of one wire node per line.

**What shadcn does.** Examples are registry items of their own type,
`registry:example` (`apps/v4/registry/bases/base/examples/_registry.ts:7`),
published next to the components; each `registry:ui` item points at its
docs, examples and upstream API through `meta.links`
(`apps/v4/registry/bases/base/ui/_registry.ts:14-20`); the docs site builds an
examples index from the same source (`apps/v4/examples/__index__.tsx:1-8`,
generated by `apps/v4/scripts/build-registry.mts:33-61`). The MCP surfaces
them as a tool, `get_item_examples_from_registries` — "Find usage examples
and demos with their complete code. Search for patterns like
'accordion-demo', 'button example', 'card-demo', etc." (`packages/shadcn/src/
mcp/index.ts:146-148`) — and the search tool's own description sends the
agent there: "After finding an item, use get_item_examples_from_registries
to see usage examples" (`:96-98`).

**The take.** In `jetpacs-poc`, tooling only:

| Change | Where |
|---|---|
| `jetpacs-applet-catalog--examples (type)` → list of `{source, elisp?, node}`: jetpacs.* pages from the component-catalog documents, material3.* from the m3 `:examples`, everything else from the first matching golden line | `jetpacs-applet-catalog.el` beside `--core-node` (:263) and `--extension-node` (:291) |
| `examples` key on every node entry; catalog format 2, generator `jetpacs-applet-catalog/2` | `jetpacs-catalog.json`, `CATALOG.md` (all), `SKILL.md` (extension nodes only) |
| MCP tool `jetpacs_node_examples {nodeType, limit?}`; `jetpacs_describe_node` gains the first example and the cross-reference sentence | `jetpacs-applet-mcp.el:144, 174, 322` |
| ERT: every extension node has one example whose `t` equals its type; the drift test already pins the renderings | `jetpacs-applet-mcp/test/jetpacs-applet-tooling-test.el:554` |

*Gate:* `tools/check-jetpacs-catalog.sh` in `test/run-tests.sh`, the
applet-MCP ERT command from its `AGENTS.md`, the component-catalog ERT.

*Resolved with the owner 2026-10-07: in scope; the closed authoring model of
the component catalog is not extended — nodes without a page take their
example from the golden.*

---

## 4. Take 2 — a skill with live context, rule pairs and evals

Three mechanisms in `skills/shadcn/`, each cheap and each missing from the
generated `skills/jetpacs-applet/SKILL.md`.

**4a. Live project context.** The skill injects the project's state at load
time with a shell directive — ``!`npx shadcn@latest info --json` `` under
`## Current Project Context` (`skills/shadcn/SKILL.md:14-17`) — and the text
below tells the agent which fields matter (`:149`). The Jetpacs skill is
static: an agent authoring against it sees the whole catalog, not what the
connected Companion advertises. The ebp analogue of "the project" is the
negotiated profile. Nothing persists it today: the welcome's
`surface_profiles` live only on the client (`ebp.el/lisp/ebp.el:1774`), read
by the gates (`jetpacs-poc/emacs/jetpacs-surfaces.el:1109-1240`) and copied
into `jetpacs-chrome--gate-signature` (`jetpacs-chrome.el:784`).

*Take:* `tools/jetpacs-info.sh` printing `{contract, extensions, applets,
last_profile?, registries?}` and a `jetpacs_info` MCP tool; a snapshot writer
on `jetpacs-ready-functions` (`jetpacs-surfaces.el:1366`) that stores the
last welcome's `profiles`/`granted`/`limits` under `~/.emacs.d/jetpacs/`;
the generated SKILL.md gains the directive and one sentence: author against
`last_profile` when present, otherwise against the catalog.

**4b. Rules as Incorrect/Correct pairs.** shadcn's `## Critical Rules`
(`SKILL.md:29-85`) are one-liners that each link to a file of code pairs:
`rules/styling.md` opens with "**Incorrect:**" `bg-blue-500` and
"**Correct:**" a semantic token (`skills/shadcn/rules/styling.md:19-30`).
The Jetpacs rules are seven prose sentences (`jetpacs-applet-catalog.el:539`).
The content is already right; the form is harder for an agent to apply.

*Take:* `rules/{ir,data,actions,theme,registry}.md` generated from a
`jetpacs-applet-catalog--rule-pairs` constant so the drift check still owns
them; the theme file carries ebp's own version of the semantic-colour rule —
a theme role over a hex literal (SPEC §16.6, `SPEC.md:3288`) and never a
derived token (§18.4).

**4c. Evals.** `skills/shadcn/evals/evals.json` is
`{skill_name, evals[{id, prompt, expected_output, files, expectations}]}`
(`:2`), the same schema skill-creator documents in its `references/
schemas.md`. The Jetpacs skill has none.

*Take:* five hand-written evals with checkable expectations (bound list,
conditional row, action with a patch and no blocking call, pinned tabs, a
registry add that asks which registry and dry-runs first); owner-run, not
CI.

**4d. Two rules that cost nothing.** "Registry must be explicit — do not
guess the registry … Never default to a registry on behalf of the user"
(`SKILL.md:186`) and "Never use `--overwrite` without the user's explicit
approval" (`:206`), with "NEVER fetch raw files from GitHub manually — always
use the CLI" (`:198`). They go into `rules/registry.md` with take 3.

*Gate:* `check-jetpacs-catalog.sh` (SKILL.md and rules are generated);
`tools/jetpacs-info.sh` prints valid JSON with and without a snapshot; ERT
for the snapshot writer in `test/run-tests.sh`.

---

## 5. Take 3 — an applet registry

**The gap.** Jetpacs has no distribution story. Components ship as source
modules mapped in `companion/settings.gradle.kts:58`; applets ship as a
hand-maintained plist (`jetpacs-poc/emacs/apps/packaged-apps/
jetpacs-packaged-apps.el:16-48`) staged by a hand-maintained list in
`companion/app/build.gradle.kts:193-210` and again in
`tools/onboard-tablet.sh:619-647`; the App Store side-loads single `.el`
files from two staging directories (`emacs/jetpacs-app-store.el:52`). The
pieces of a registry already exist in isolation: `jetpacs-defapp
:requires-extensions` is an app-to-extension dependency
(`emacs/jetpacs-apps.el:220-235`), the applet manifest
(`jetpacs.applet-manifest/2`, `jetpacs-applet-tooling.el:1888-2066`) already
records every file's sha256, and `jetpacs_validate_applet` (`:2540-2736`) is
half of `registry validate`.

**What shadcn does.** The registry item (`packages/registry/src/registry/
schema.ts:158-179`) is `{name, type, title?, description?, dependencies[],
registryDependencies[], files[{path, type, target?, content?}], docs?,
categories?, meta?}`; the index is `registry.json` with `items` or a
relative-only `include` (`schema.ts:234`; `validate.ts:281`; depth-capped,
`loader.ts:307`). Addresses come in five schemes — bare, `@namespace/item`,
URL, `file`, `owner/repo/item[#ref]` (`address.ts:15-32, 45-75`). Namespaced
registries live in `components.json#registries` with `{name}` templates and
`${VAR}` expansion in URL and headers, and a header whose variable is empty
is dropped (`builder.ts:126`). The resolver walks `registryDependencies`
breadth-first, topologically sorts with Kahn's algorithm (`resolver.ts:148,
697, 772`) and deduplicates files by target. `build` inlines every file's
content and writes `<out>/<name>.json` plus a stripped index
(`loader.ts:144-157`; `commands/build.ts:21-91`). GitHub addresses are fetched
over `raw.githubusercontent.com` (`github.ts:29, 49`). Before any write,
`add --dry-run` classifies each file `create | overwrite | skip` and keeps
the existing content for `--diff` (`src/utils/dry-run.ts:38, 237-252`).

**The take.** *Resolved with the owner 2026-10-07: plan the full thing* —
index, addresses, dependency resolution, fetch from file, URL and GitHub,
validation, build, install, preview-then-diff. Tooling only, in
`jetpacs-poc`; the design (file layout, formats, trust chain, tests, commit
order) is in the plan, and the shape is:

| shadcn | Jetpacs | Difference and why |
|---|---|---|
| `registry:*` item with `files[].content` | applet item `{format, name, version, title, description, files[{path, sha256, content?}], package_requires, registry_dependencies, requires_extensions, requires_features, applet_manifest_sha256}` | Derived by `build` from the existing applet manifest; authored keys in a `jetpacs-applet.json` read by `json-parse-string`, so no Lisp reader touches it |
| `registry.json` + relative `include` | the same, `format: 1` | — |
| five address schemes | the same five | GitHub addresses fetch the *built* `public/r/<item>.json`, not source files, so every fetch has a sha pin |
| `${VAR}` headers, dropped when empty | the same, in `jetpacs-registry-namespaces` | — |
| BFS + Kahn, cycles tolerated, duplicate targets last-wins | BFS + Kahn, **cycles refused**, duplicate basenames refused | ebp validates everything then rejects atomically (SPEC §13.2); every apps directory shares one `load-path`, so a duplicate basename is a real collision (`tools/onboard-tablet.sh:323-330`) |
| `add` installs through a ts-morph transform chain | `install` writes bytes and a lockfile; **loads nothing** | Only the App Store's existing consent gate ("installing is running code", `emacs/jetpacs-app-store.el:31, 584`) loads an applet |
| `--dry-run` / `--diff` | the same classification and a `diff-no-select` rendering; overwrite needs explicit approval | — |
| `registry validate` | the same, over the applet validator plus index-level checks | — |
| MCP search/list/view/get-add-command | `jetpacs_registry_{namespaces,list,search,view,resolve,validate,install_command}`; the MCP never installs or builds | the static MCP's read-only invariant (`jetpacs-applet-mcp/test/jetpacs-applet-mcp-test.el:105`) |

Two placement facts decide the module layout: the applet tooling is never
staged onto the device (`build.gradle.kts:150-212`), so parsing, fetch,
verification, resolution, lockfile and dry-run live under `emacs/` as a
foundation module, and build/validate/MCP live in the tooling; and the App
Store's adopt directory is not on `load-path` (`emacs/jetpacs-init.el:88-96`
adds only the installation's `apps/*`), so the registry source pushes it
once. Fetched Elisp is read with a throwaway obarray before anything else
touches it (the `jetpacs-authoring-read-one` discipline,
`jetpacs-authoring/lisp/jetpacs-authoring.el:36-71`), and no fetch runs
inside a `jetpacs-defaction` body, so the validator's blocking-call rule
(`jetpacs-applet-tooling.el:198`) stays literally true.

*Gate:* the new ERT suites (address table, resolver, sha refusal, obarray
isolation, dry-run, lockfile, a loopback HTTP fixture), the committed
`public/r/` drift check in `test/run-tests.sh`, the App Store suite
unchanged, and the device check owner-gated.

---

## 6. Behind the gate

### 6a. Design-profile preview-then-diff

`skills/migrate-radix-to-base/SKILL.md:28-57` names the pattern: fetch the
pristine "golden" of each file, classify the local copy as pristine or
customized, and three-way merge the customized ones with
`git merge-file user.tsx radix-golden.tsx base-golden.tsx` (`:54`); the
shadcn skill asks "overwrite, partial, merge, or skip?" before switching a
preset (`skills/shadcn/SKILL.md:187-193`). The Jetpacs analogue is design
profiles: presets are immutable and users keep copies through Customize
(`jetpacs-components/lisp/jetpacs-components/jetpacs-design-profiles.el:
29-44`), but the closed plist has no origin key and `jpdesign.save-as` loses
the lineage (`jetpacs-component-catalog/lisp/jetpacs-design-lab.el:403`), so
a preset update has no merge path into a user copy. *Resolved with the owner
2026-10-07: parked behind design-runtime stabilisation; `jetpacs.design` is
experimental and off by default.* The free rule — never overwrite without
approval — lands with take 2.

### 6b. Preset codes

`preset decode`, `url`, `open` and `resolve` (`packages/shadcn/src/preset/`;
`SKILL.md:188-189`) turn a theme choice into one shareable string. A design
profile is a closed plist and could encode the same way. *Resolved: parked;
useful only once profiles are stable.*

### 6c. A health-ranked registry directory

`/r/registries.json` is built from a directory plus a health snapshot and
rankings (`apps/v4/app/r/registries.json/route.ts:7, 61-63`). It matters
only once more than one registry exists. Parked without a decision.

---

## 7. What must not transfer

shadcn stands on "components are added as source code to the user's
project" (`skills/shadcn/SKILL.md:11`); ebp stands on a Companion that holds
no app code. Everything below follows from that.

- **The install transform chain.** Import rewriting, `"use client"`
  stripping, CSS-variable inlining, Tailwind prefixing, icon-library swaps
  and TSX→JSX (`packages/registry/src/utils/transformers/index.ts:46-48` and
  siblings) rewrite source on the way into a project. There is no source to
  rewrite on the way into a Companion, and the Elisp side is generated from
  the manifests already.
- **The variable set.** One shadcn theme carries about thirty oklch values
  per mode including `chart-*` and `sidebar-*` (`apps/v4/registry/themes.ts:
  7-80`). SPEC §18.4 names thirteen roles and says derived tokens "are not
  EBP names and MUST NOT be accepted as `colors` members". The role set does
  not grow because shadcn's did.
- **Tolerance in the resolver.** Dependency cycles are tolerated and
  duplicate file targets resolve last-one-wins (`resolver.ts:697-`). ebp
  validates the whole request and rejects atomically (SPEC §13.2); the
  Jetpacs resolver refuses both.
- **Hand-synced schemas.** The JSON Schema copies under `apps/v4/public/
  schema/` are maintained by hand (`schema.ts:81`, the comment above it).
  Every ebp projection is generated and drift-checked; keep generating.
- **Universal `~/` items** (`packages/registry/src/registry/utils.ts:35`)
  that install verbatim without a config, and the **v0 hand-off**
  (`apps/v4/components/open-in-v0-button.tsx:24`). No counterpart.
- **`get_audit_checklist`** (`packages/shadcn/src/mcp/index.ts:180`) — a
  static post-generation checklist for a web project. The Jetpacs
  equivalents are `jetpacs_validate_applet` and the ERT gate, which check
  rather than remind.

---

*Practical upshot: three takes, all tooling in `jetpacs-poc`, none with
normative text — examples in the catalog and MCP, a skill with live context,
rule pairs and evals, and an applet registry built as a third App Store
source that never loads what it fetches. §6 is three parked entries, two of
them closed by the owner's decision. Fold the shapes into the applet tooling
directly — this document is the quarry, not the wall.*
