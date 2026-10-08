# Harvest: what four A2UI renderers teach the Jetpacs renderer

**Status: Claude-authored reference sketch. Not code, not spec — the owner's
spec and implementation supersede this document wherever they diverge.**
Sketch policy: tables, file:line pointers and posed decisions; the decisions
this harvest needs are listed at the end of each take and are *not yet
settled* — nothing here was agreed with the owner beyond "write it up".

**Sources & verification note.** Every A2UI-side fact below was read on
2026-10-07 from the four checkouts under `~/workspace/a2ui/`:

| Repo | Commit | What it is |
|---|---|---|
| `a2ui-compose` (uny) | `cefac98`, 2026-09-29 | Kotlin Multiplatform renderer: `a2ui-core`, `a2ui-compose`, `a2ui-material3`, `a2ui-material3-markdown`, `a2ui-gallery` |
| `generative-mui` | `260b0f2`, v1.0.1 | React renderer onto Material UI, v0.9.1 protocol |
| `material3-expressive-a2ui` | `3f5b7ec`, v0.2.1 | React renderer onto Material 3 Expressive, on `@a2ui/web_core` |
| `prefab` (Max Health) | `3f20b61`, v0.3.14 | TypeScript authoring that emits `$prefab` (own vanilla-DOM renderer) or A2UI v1.0 |

Cited as `file:line` under the repo root. Every ebp fact is from `SPEC.md` /
`contract.json` at ebp `01d3aae` (protocol 3, document 3.1.0-draft, contract
format 12, last amendment #198); Jetpacs facts are from `jetpacs-poc`
`4566db04`, `ebp-kmp` `c84b6d0`, `ebp-compose` `146dda1`. Nothing is from
memory of any repo's documentation; where a judgment is mine rather than a
sourced fact it is phrased as a recommendation or a question.

**Why this harvest, and what it is not.** The 2026-10-05 survey of Google's
`androidx.a2ui` gave Jetpacs its renderer *architecture* (typed members from
the contract, per-node registries, headless replay) and its *protocol*
borrowings (#194 data document, #195 templates, #196 `insert`, #198
`visible_when`). This document asks a narrower question of four independent
A2UI renderers: **what do they do at render time that Jetpacs does not, and
which of it survives the EBP stance** — Emacs is right, the Companion is
strict, no client expressions, no client state authority (§1, §3, §23.2, W2,
and the must-not-transfer lists of the LiveView and json-render harvests).
The answer sorts into: places Jetpacs is already ahead (§1 — take the
confidence), five takes (§3–§7, none of them wire changes), two parked
entries (§8) and the usual warning (§9).

**One ground rule.** Every take below is a renderer or tooling change inside
`jetpacs-poc`, `ebp-compose` or `ebp-kmp`. None touches the wire, so none
needs a `SPEC-CHANGES.md` row. Each names the repository that owns it and the
test that pins it.

---

## 1. Convergences — take the confidence, adopt nothing

Read next to the Jetpacs pipeline, three of the four renderers converge with
it on the points that matter, and on several the Jetpacs answer is the
stronger one.

| Renderer | Jetpacs | Where |
|---|---|---|
| a2ui-compose: one immutable snapshot, no observables; `updateComponents` is an upsert into a `Map` (`a2ui-core/.../surface/SurfaceModel.kt:36-81`) | One immutable `JsonObject` end to end, "there is no parallel tree"; a patch re-materializes, re-validates and commits atomically | `ebp-compose/renderer/compose/.../ComposeNodeComponent.kt:15-23`; `ebp-kmp/wire/.../SurfaceStore.kt:533-666` |
| a2ui-compose: agent gets no style properties; only semantic enums (`variant`, `justify`, `weight`, `fit`); non-http(s) image URLs refused (`a2ui-material3/.../Image.kt:150-162`) | §16.6 roles or hex, fixed universal attributes, no typography outside the experimental design runtime; `ImageGuards.kt` | SPEC §16.6, §17.2; `ebp-kmp/wire/.../ImageGuards.kt` |
| generative-mui: `.strict()` schemas reject `sx`/`style`/`className` outright, never strip (`packages/generative-mui-core/src/catalog.ts:27-28`) | Whole request validated, 1201 on an unknown required shape, previous tree stays on screen | SPEC §13.2 |
| m3e: `structuredClone` every incoming message because web_core writes user input into the stored model (`src/runtime/useA2ui.ts:477-491`) | Drafts are id-keyed and live beside the document, never inside it; the document changes only through Emacs | SPEC §13.6; `SurfaceStore.kt:864-887` |
| m3e: template children keyed `id@basePath`, explicit children `id#index` (`src/internal/children.tsx:15-25`) | Identity path `key` > `id` > position, template roots must bind `key`; `key(resolvedPath)` around every node | SPEC §16.1; `material3/.../Renderer.kt:583-606` |
| a2ui-compose: action `context` resolved at tap time; template-scoped buttons carry their row (`a2ui-compose/.../A2uiComponentScope.kt:320-358`) | `args.item` injected by the engine from the innermost enclosing item of `sourcePath` | `ebp-kmp/wire/.../CompanionEngine.kt:808-811`, `SurfaceStore.kt:437-450` |
| m3e and a2ui-compose: markdown links reduced to labels, never live (`a2ui-material3/.../Markdown.kt`; `src/internal/markdown.tsx:222`) | No markdown at all (§16.4); `rich_text` spans with authored `on_tap` | SPEC §16.4; `material3/.../ContentNodes.kt:257-313` |
| generative-mui: catalog prompt block derived from the same Zod schemas the renderer validates with, drift-tested (`packages/generative-mui-core/src/catalog-doc.ts`) | The agent catalog export (json-render harvest take 1) | `JSON-RENDER-harvest.md` §3 |
| m3e: local mirror for two-way inputs, external update wins (`src/internal/useBoundValue.ts`) | id-keyed drafts that clear on acknowledgement; `debounce_ms` proposed | SPEC §13.6; `LIVEVIEW-harvest.md` §4 |

*Where Jetpacs is ahead, and should stay so.* a2ui-compose's `List` is an
eager scrolling `Column` because `LazyColumn` cannot answer intrinsics
(`a2ui-material3/.../ListComponent.kt:22-101`); Jetpacs has a real keyed
`LazyColumn` with batchable leaves chunked by eight
(`material3/.../LayoutNodes.kt:1173-1292`, `Attributes.kt:161-233`).
a2ui-compose resets `Tabs` to index 0 whenever a component is re-sent
(`a2ui-material3/.../Tabs.kt:63-69`) and loses tab content state on switch;
prefab tears down and rebuilds the whole DOM on every state change and keeps
tab, search and accordion state only by accident (`src/renderer/index.ts:267-276`);
Jetpacs retains presentation by identity path with incarnation fences across
remove-and-re-add (`RetainedPresentationTracker.kt:132-203`). generative-mui
re-renders the entire surface on every data write — one
`useSyncExternalStore` over the whole store, no `React.memo` anywhere
(`packages/generative-mui-react/src/A2uiSurface.tsx:82,125-137`); Jetpacs
publishes one `StateFlow` per surface so a hidden surface never recomposes the
visible one (`companion/app/.../AppSurfaceRegistry.kt:23-46`). And every one
of the four renders bindings and templates in the renderer; Jetpacs
materializes in the engine so no renderer learns about pointers
(`ebp-kmp/wire/.../Materializer.kt:55-65`).

---

## 2. Before the takes: the test these takes must pass

The LiveView harvest's field-kind test does not apply — nothing here is a
wire field. The test that does apply is the one §8 of the json-render harvest
states: json-render, and all four renderers here, stand on output that must
be *tolerated*; ebp stands on a builder whose output must be *right*. So
every rendering technique below was asked one question first: **does it
exist to tolerate a wrong tree, or to render a right tree well?** Placeholder
taxonomies, render budgets, error boundaries and reachability GC are the
first kind and are refused in §9. Structural sharing, geometry sweeps,
loading states and layout clamps are the second kind and are the takes.

---

## 3. Take 1 — structural sharing across patches, and the test that measures it

**The gap.** A `surface.data` patch applies its ops, then re-materializes the
*whole* authored tree and re-validates it (`SurfaceStore.kt:636-666`). The
materializer builds a fresh `JsonObject` for every node it walks
(`Materializer.kt:286-337`, `JsonObject(out)` at 337), so no subtree keeps
its identity from one revision to the next. Compose skipping therefore rests
on *deep* `JsonObject.equals` over every node of the surface — a linear cost
in surface size per patch even when one leaf changed. In material3 that deep
compare is what `compose-stability.conf` buys: it declares the kotlinx Json
types stable (`companion/renderer/material3/compose-stability.conf`, wired at
`build.gradle.kts:37-38`), which makes every skip check a deep `equals`
rather than enabling skipping at all. That file exists in the material3
module only. `ebp-compose`'s renderers and `jetpacs-components` have no
stability configuration, so a `JsonObject` parameter is *unstable* there and,
under strong skipping, is compared by reference — which a fresh instance per
revision never passes, so every node recomposes on every publish.

**What a2ui-compose actually does (verified).** Its state is one
`mutableStateOf(RendererState)` and every write replaces it
(`a2ui-compose/.../A2uiRenderer.kt:94`), which sounds worse; what saves it is
that `updateComponents` upserts into a persistent map
(`SurfaceModel.kt:67-70`) so an untouched `Component` keeps reference
identity, the per-instance scope is remembered on `(renderer, surfaceId,
component, evaluationScope)` (`A2uiSurface.kt:296-321`) and survives any
write that does not touch that component, and every typed read is a
`derivedStateOf` so only a changed *value* recomposes its reader
(`A2uiComponentScope.kt:420-477`). It then **proves the granularity**:
`A2uiScopeStabilityTest` installs a counting registry and asserts per-id body
runs and scope rebuilds after a data write
(`a2ui-compose/src/composeUiTest/.../A2uiScopeStabilityTest.kt:55-75`), with
a comment noting that re-providing a CompositionLocal per recomposition
would fail the `bodyRuns` assertion. Jetpacs has no recomposition-count test
anywhere (grep for `bodyRuns`, `RecomposeScope`, `compositionCount` across
`companion/renderer` and `ebp-compose/renderer`: nothing).

**The steal — proposed shape.** Two small changes and one test, all behind
the existing "no renderer changes for data" rule.

1. *Materializer sharing.* A reconcile pass, `Materializer.share(fresh,
   previous)`, runs after the normal walk, so the walk's side effects —
   `bindings`/`predicates` counters, `itemKeys`, PROBE walks — run exactly as
   today. It returns the *previous* instance for every subtree structurally
   equal to the one it replaces, and a fresh container only along changed
   paths. Objects are matched key by key with a positional key check, so a
   tree parsed in another key order is rebuilt rather than silently
   reordered. Arrays align by `key` when every element of both arrays carries
   a unique one (a keyed insert at index 0 keeps every later row), else by
   index. Data ops already copy only along the pointer path
   (`JsonPointer.kt:85-180`), so a template item whose scope element is
   untouched comes back as the same instance. `SurfaceStore` applies it in
   `patch`, in the data branch of `update`, and in the no-data branches of
   `update` (the Org whole-document push, where the pass shares directly
   against the previous record's spec). Validation still runs on the whole
   result (§13.2 unchanged); only allocation and equality cost drop.
2. *Identity everywhere.* material3's `compose-stability.conf` is removed and
   none is added to `ebp-compose` or `jetpacs-components`. kotlinx-
   serialization-json 1.11.0's `JsonObject.equals` and `JsonArray.equals`
   compare the inner collection against the *wrapper*
   (`Intrinsics.areEqual(this.content, other)`), so `AbstractMap.equals`'s
   identity shortcut never fires and even the same instance is deep-walked;
   only `JsonLiteral.equals` short-circuits. Declaring the types stable
   therefore makes every skip check O(subtree). Under strong skipping (the
   Kotlin 2.3.21 default) an unstable parameter is compared by `===`, which
   is O(1) once the materializer shares instances.
3. *The counter test.* A public `LocalNodeRenderObserver` seam, null in
   production, invoked with the node's identity path at the top of
   `RenderNodeContent`; `MaterialTestController` provides it, and a
   Robolectric test drives a real `SurfaceStore` through `update` and `patch`,
   hands each revision's spec to the controller, and asserts that a `set`
   under one binding runs the root and that one node's path and nothing
   else, and that a keyed `insert` at index 0 of a templated list runs only
   the new row. This is also the instrument that §7a of the json-render
   harvest's measurement gate needs and does not have: "how often does a
   patch still recompose the whole surface" becomes a number.

**What this take deliberately does not copy.** `derivedStateOf` per member.
Jetpacs renders a resolved tree with typed reads; the equivalent granularity
comes free from node identity plus shared instances, without a scope object
per node.

*(Settled with the owner 2026-10-07: scope is take 1 only; the sharing lives
in `Materializer` as a pure pass; identity everywhere; device gates may run on
the S21 Ultra; the `inputDisplays` CompositionLocal question — it is
republished on every accepted snapshot and carried in every `RenderCtx`,
`DeviceBridge.kt:1703`, `Renderer.kt:244-266,457-469` — is deferred until the
counter test shows churn.)*

**Pinned by:** `MaterializerShareTest` in `ebp-kmp` (`assertSame` on
untouched subtrees across `patch` and `update`); `PatchRecompositionTest` in
material3 (render-observer counts: a patch runs only the changed identity
path).

---

## 4. Take 2 — a phone-width geometry sweep over the existing goldens

**The gap.** The screenshot task renders the jetpacs-components previews at
400dp and 610dp widths against stored references at threshold 0.001
(`material3/src/screenshotTest/.../JetpacsComponentsScreenshotTest.kt:26-30`).
That catches pixel drift in what the previews cover; it does not ask
"does any text in any golden line overflow the screen at the narrowest
supported width", and the device gates are run by hand on one phone.
`MaterialWidgetsGoldenReplayTest` asserts every `widgets.golden` and
`jetpacs-material3.golden` line renders or degrades without throwing and
dispatches each descriptor once (`MaterialWidgetsGoldenReplayTest.kt:36-53`)
— nothing about geometry.

**What a2ui-compose actually does (verified).** `PhoneWidthLayoutTest`
renders each of the 43 vendored spec examples in a 320dp-wide host, walks the
semantics tree, and asserts that every non-blank text node has width > 0 and
that `positionInRoot.x + width` stays within the root — with one allowance,
text inside a horizontally scrolling ancestor may run past the edge, and one
listed known-overflow example
(`a2ui-gallery/src/composeUiTest/.../PhoneWidthLayoutTest.kt:31,66,87-92`).
Its `GalleryIntegrationTest` adds the complementary check that every example,
stepped one message at a time, ends with zero placeholder chips.

**The steal — proposed shape.** One Robolectric test beside
`MaterialWidgetsGoldenReplayTest`, over the same two corpora plus
`widget-surfaces.golden`: render each line's surface under
`MaterialTestController` in a 320dp × 640dp window; walk
`onRoot().fetchSemanticsNode()`; for every node with `Text` semantics assert
nonzero width and right edge ≤ root width unless an ancestor carries
horizontal scroll semantics (Jetpacs rows with `scroll` set `horizontalScroll`,
`LayoutNodes.kt:549`). Keep an explicit allowlist file for known overflows so
the test is red only for new ones. Runs in the unit-test task, no device.

**What this take deliberately does not copy.** Rendering at a single width
only. The Jetpacs profiles already distinguish compact and medium; run the
sweep at 320 and 600.

**Decisions this take needs:** none of substance. Whether to fold it into the
existing replay class or keep it separate is a taste call; separate keeps the
"renders without throwing" invariant and the "fits" invariant independently
red.

**Pinned by:** itself, in `companion/renderer/material3/src/test`.

---

## 5. Take 3 — a loading state distinct from a failure state for images

**The gap.** `RenderImage` loads through `produceState<Bitmap?>(null, url)`
and draws the *same* `broken_image` box whenever the bitmap is null
(`material3/.../ContentNodes.kt:121,155-168`). A healthy image on a slow
network shows a broken-image glyph for its first few hundred milliseconds; a
lazy column scrolling into fresh rows flashes a column of broken images. The
seam, `LocalMaterialImageSource`, is `suspend (String, Limits) -> Bitmap?`
(`ContentNodes.kt:107-110`), so loading and failure are indistinguishable at
the call site.

**What a2ui-compose actually does (verified).** With no host loader
installed, `Image` draws a sized, *described* placeholder — a neutral box with
the content description — and that is also what a scheme-refused URL draws
(`a2ui-material3/.../Image.kt:83,87,126,150`). The host loader is handed the
size and description and owns loading, error and success states itself
(`Image.kt:37`). m3e, for contrast, has no loading state at all (plain
`<img>`), which is the gap this take avoids.

**The steal — proposed shape.** Widen the seam's result to a three-valued
state (`Loading`, `Loaded(bitmap)`, `Failed`) or keep the suspend signature
and wrap `produceState` so the initial value is `Loading` and a null result
is `Failed`. Draw a neutral `surfaceVariant` box with the description for
`Loading` (same box as today, no glyph), and keep the `broken_image` glyph
for `Failed`. `ImageCache` already distinguishes a miss from a stored
failure; let it say which. The test controller stub stays a one-liner.

**Decisions this take needs:** whether a *refused* URL (scheme or limits) is
`Failed` (the glyph) or the neutral box, as a2ui-compose does. Recommendation:
the glyph — a refused URL is the author's mistake and should look like one.

**Pinned by:** a `MaterialTestController` test that holds the source in a
`CompletableDeferred` and asserts the glyph is absent before completion and
present after a null.

---

## 6. Take 4 — "never throw" clamps for nested scrollables and renderer identity

**The gap.** Jetpacs has no error boundary: a composition throw blanks the
surface, and the pipeline relies on the clamps in `Attributes.kt:8-12` and
friends instead. That is the right stance (§9), but it means every Compose
runtime exception reachable from a *valid* tree is a blank screen. Two are
reachable today. A `column` with `scroll` under a parent that already
scrolls vertically gives the inner `verticalScroll` infinite height
constraints; Compose throws. The code avoids this in one place only — inside
dialogs, where the host already wraps content in `verticalScroll`, column
scroll is suppressed (`LayoutNodes.kt:572-588`). An app-surface `column
scroll` inside a `lazy_column` item, or inside another scrolling column, is
legal traffic (nothing in §16 forbids it) and has no guard.

**What a2ui-compose actually does (verified).** Every scrolling container
gets `Modifier.boundedScrollAxis(horizontal)`, a `layout` modifier that clamps
an infinite main-axis constraint to 32767px before `verticalScroll` /
`horizontalScroll` sees it (`a2ui-material3/.../ListComponent.kt:67,86,120-135`;
the constant is chosen "for what `Constraints` can pack"). It also wraps each
`fun interface` renderer call in `key(componentRenderer)`
(`a2ui-compose/.../A2uiSurface.kt:342`) after a Kotlin/Native slot-reuse
segfault when a component's *type* swapped at the same position; Jetpacs's
`key(resolvedPath)` includes the type in the path (`Attributes.kt:275-284`),
so that one is already covered. Smaller clamps it carries that Jetpacs
already has: slider steps from the values array are `coerceAtLeast(0)`
(`InputNodes.kt:1120,1174,1217`); non-finite numbers are rejected at
validation.

**The steal — proposed shape.** A `boundedScrollAxis` modifier in
`ebp-compose` Foundation, applied in `LayoutNodes.kt` wherever
`verticalScroll` / `horizontalScroll` is attached (`:549,:588` and the
`flow_row`, `card` and `surface` cases if they scroll), and in the
jetpacs-components `panel`. Then an audit pass — one golden line per
scroll-inside-scroll shape (`column scroll` in `lazy_column` item, `row
scroll` in `column scroll`, `lazy_column` in `column scroll`) added to
`widgets.golden`, since the replay test's "renders without throwing"
invariant then pins them. The `lazy_column`-under-`IntrinsicSize` case
(a `SubcomposeLayout` child cannot answer intrinsic queries, which is why
a2ui-compose has its refusal cache at `Layout.kt:464-509`) is not reachable
from EBP traffic today because no node maps to `IntrinsicSize`; note it in
the component docs so a future `table` or `pane_scaffold` change does not
introduce it.

**Decisions this take needs:** none. It is additive and invisible for every
tree that renders today.

**Pinned by:** the new `widgets.golden` lines through
`MaterialWidgetsGoldenReplayTest`.

---

## 7. Take 5 — weight starvation and layout traits, behind a measurement

**The gap, if it is one.** Jetpacs `row` and `column` are Compose `Row` and
`Column`; the parent applies `Modifier.weight` from the child's `weight`
member (`Renderer.kt:769,798`). Compose measures unweighted children first
with the full remaining space and weighted children with what is left, so a
long unweighted `text` beside a `weight: 1` sibling starves the sibling.
Emacs authors every tree, so an author who sees it can size the text — the
starvation is a documented Compose behaviour, not a bug — but no Jetpacs
test or golden exercises it, and Grove's planning and Orgzly's outline rows
are exactly the shape (label beside a weighted value).

**What a2ui-compose actually does (verified).** It replaces `Row`/`Column`
with its own `Flex` layout and `FlexMeasurePolicy`
(`a2ui-material3/.../Layout.kt:54-91,389-425,445-`): children that refuse
intrinsics are measured first; askable children report preferred and minimum
widths and CSS `flex-shrink` is applied proportionally (`shrink()` at
`Layout.kt:1261-1300`); fill children take up to an equal share; `weight`
children split the remainder by cumulative rounding and are measured to
exactly their share, even below their minimum. Each renderer declares a
per-axis `LayoutTraits(fit = Content | Fill | Fixed)` so the container knows
before measuring whether a child wants a share
(`a2ui-compose/.../LayoutTraits.kt:67,171-261`), inferred through wrappers so
a `Card` around a filling image is itself `Fill`. A per-tree `QuerySession`
memoizes intrinsic answers (a comment cites a 40-second case without it) and
a refusal cache remembers which children throw on intrinsics
(`Layout.kt:464-509`). It is, by a wide margin, the largest piece of code in
the repo, and it exists because an *agent* authors A2UI rows with CSS
expectations the renderer must honour.

**The steal, conditionally.** Not the policy. Add two golden lines (a long
unweighted text beside a weighted sibling; two weighted siblings beside a
fixed icon) and look at them in the take-2 sweep and on the phone. If they
are wrong, the small fix is a2ui-compose's *ordering* idea only — measure
weighted children to their share of the space left after fixed-size siblings'
*minimum* intrinsic width, not their preferred width — which is a 40-line
custom `MeasurePolicy` for `row` only, no traits contract. If they are fine,
record that and close this entry.

**Decisions this take needs:** whether to look at all. *Recommendation:* add
the two golden lines now (free); build nothing until one is wrong.

---

## 8. Behind the measurement gate

### 8a. Per-node subscription instead of per-surface

m3e's `buildChild` returns a `DeferredChild` that subscribes to its one
component id (`src/runtime/A2uiSurface.tsx:44-62,128-132`), and each catalog
entry owns a `GenericBinder` subscription so only readers of a changed path
re-render (`src/runtime/adapter.tsx:97-120`). That granularity needs an
addressable node store — exactly the node-tree patching that
`JSON-RENDER-harvest.md` §7a parks behind measurement. Take 1's counter test
is the measurement. **Reserved sentence, unchanged from §7a:** *a future
negotiated capability may patch the node tree by identity; until measurement
demands it, trees ride in snapshots.* Nothing to add.

### 8b. The counting registry as a devtools source

Take 1's body-run counter is a test instrument. a2ui-compose's gallery
exposes the same counts live beside its message stepper
(`a2ui-gallery/.../GalleryState.kt`). json-render harvest take 2 (devtools
that see the data path) is Emacs-side by decision; a Companion-side
recomposition counter under the existing profiler toggle would be the first
Companion devtool with a reason to exist. Parked until take 2 of that harvest
has landed and wants a Companion number.

---

## 9. What must not transfer

The four renderers stand on trees that may be partial, cyclic, over-deep,
shared, or wrong; ebp's tree is validated whole before any state changes.
Everything below exists to tolerate the first kind of tree and is refused
for the second — the json-render harvest's §8 list, with the A2UI-specific
spellings added.

- **Streaming placeholders and placeholder taxonomies.** All four draw
  something for a child referenced before it arrives: a MUI `Skeleton`
  (generative-mui `registry.tsx:114-139`), a pulsing `aria-busy` span (m3e
  `A2uiSurface.tsx:69-71`), a host-rendered reason among `MissingComponent`,
  `UnknownType`, `Cycle`, `TooDeep`, `BudgetExceeded`, `TooManyChildren`
  (a2ui-compose `A2uiSurface.kt:50-96`). Progressive streaming was rejected
  with no reserved sentence (`JSON-RENDER-harvest.md` §7e); a reference that
  dangles is a 1201 at admission, so the taxonomy has nothing to name.
- **Render budgets and fan-out division.** a2ui-compose's
  `RenderLimits.maxInstances = 5000` / `maxDepth = 24`, the pre-descent
  `renderCost()` and the even budget split at template expansion
  (`a2ui-core/.../Traversal.kt:42-43,101-124`; `A2uiComponentScope.kt:246-281`)
  bound the 2^n blow-up an adjacency list allows when one child is shared by
  many parents. EBP trees cannot share children; depth and size are wire
  limits (§4.5), templates nest at most four deep (§13.8.5). Nothing to bound.
- **Per-node error boundaries.** generative-mui's `NodeErrorBoundary` resets
  on node identity (`guards.tsx:58-73`); Compose has no equivalent, and
  a2ui-compose does not attempt one. Jetpacs's answer is take 4's clamps.
- **Reachability GC.** generative-mui's manual `compact()` BFS from root
  (`store.ts:93-128,196-206`) reclaims orphans of an upsert-only map. A
  snapshot has no orphans.
- **Client-evaluated checks, functions and expressions.** All four evaluate
  `checks` and `call` nodes (`required`, `regex`, `formatCurrency`, …) or, in
  prefab's case, a whole `{{ }}` language with 22 pipes and a
  generation-invalidated `find` index (`src/renderer/rx.ts:28-34,94-352`).
  Refused by §23.2, W2 and #194; take 4 of the json-render harvest admits a
  comparison, never a value. m3e's UX rule — show a field's error only after
  it is touched, but disable the button from the first render
  (`TextField.tsx:68-73`, `Button.tsx:26`) — is a good rule that needs client
  validation to apply; the round-trip equivalent is the LiveView harvest's
  `pending_label` / `pending_disable`.
- **Companion-local write-back.** Every renderer writes the replica on each
  keystroke (a2ui-compose `A2uiRenderer.kt:122-140`; generative-mui
  `writeLocal`; m3e `setBound`). Reserved as `data.echo`
  (`JSON-RENDER-harvest.md` §7b); nothing new here, except one more data
  point that generative-mui's template-scoped write goes to the wrong path
  (`context.tsx:59-70` with `ItemScope`) — the race rules the reserved
  sentence asks for are not hypothetical.
- **Markdown.** §16.4 forbids it and `rich_text` spans are the better wire.
  If it is ever reopened, a2ui-compose's shape is the safe one: a
  parser-free inline `AnnotatedString` builder with a 16 KiB input cap, a
  scan budget and a nesting cap of 8, links reduced to their label
  (`a2ui-material3/.../Markdown.kt`), and a separate optional module for
  block markdown behind a sealed block model so the parser's types never
  leak.
- **prefab's renderer, and its skill.** The renderer rebuilds the whole DOM
  on every change (`src/renderer/index.ts:267-276`) and its `SKILL.md` has
  drifted from the code in at least six places (`{{ state.x }}` unresolved,
  `showToast` field names, `ForEach` syntax, scope variable names). Its
  authoring side — the lossy A2UI emitter with typed `unsupported | degraded
  | expression | action` diagnostics and transactional subtree rollback
  (`src/a2ui/emit.ts:316-361`), and one `AutoFormField[]` driving a form, MCP
  `input_required` elicitation and a JSON Schema (`src/auto/form.ts`,
  `src/mcp/input-required.ts`) — is worth a look from `jetpacs-shell` and
  the applet MCP, and is out of this harvest's scope.
- **User-activation provenance for `openUrl`.** m3e and a2ui-compose gate
  URL opening on a tap-time provenance flag (`src/internal/activation.ts`;
  `A2uiComponentScope.kt:320-358`) because expressions can call functions.
  No EBP node opens a URL except through an authored hook that Emacs or a
  builtin handles; there is no expression to guard against.

---

*Practical upshot: takes 1–4 are renderer and test work in `ebp-kmp`,
`ebp-compose` and `jetpacs-poc` with no normative text; take 5 is two golden
lines and a look; §8 is two parked entries that lean on takes already
parked elsewhere. The single most valuable item is take 1's counter test,
because it turns the standing "measurement gate" on tree patching from a
sentence into a number. Fold the shapes into the hand-written sections
directly — this document is the quarry, not the wall.*
