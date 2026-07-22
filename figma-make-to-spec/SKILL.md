---
name: figma-make-to-spec
description: >
  Turns a Figma Make prototype (or any working prototype) into a review-ready spec inside Figma —
  exploding each interaction flow into one frame per state, instancing real design-system
  components, binding tokens and text styles, wiring live prototype interactions between states,
  and flagging design-system gaps as proposals. Built for the Figma-native agent, which has
  first-class access to the canvas, the full library, variables, and prototyping. Use this skill
  whenever the user wants to make a Make output reviewable, generate a handoff or spec from a
  prototype, explode a prototype into states, wire a prototype into clickable flows, or hand a
  generated prototype to PMs, designers, or engineers for async review. Also trigger on "make
  this reviewable", "turn this Make prototype into a spec", "explode into frames", or "prep this
  for design review".
---

# Figma Make → review-ready spec

This skill runs **inside Figma** as a capability of the native agent. It takes a working
prototype — typically a Figma Make output, or a Code-Connected repo — and produces a spec file
that cross-functional partners can review async. Two equal goals:

1. **Pixel-accurate visual parity** — layout, color, spacing, typography, content, and component
   hierarchy, read from the prototype source.
2. **Interaction legibility** — every state and transition is a real, clickable prototype flow
   *and* is annotated on its node, so a reviewer understands what's tappable and where it leads.

This is the same proven method as the external `prototype-to-figma` skill.

## What the Figma MCP surface already supports (field-tested)

A live run against a real Make file corrected several assumptions. **The current Figma MCP
(`use_figma`) already does most of what was assumed to be "native-only":**

| Capability | Reality via `use_figma` today |
|---|---|
| Real prototype interactions | ✅ `node.setReactionsAsync(...)` + `figma.currentPage.flowStartingPoints` both work |
| Node-attached Dev Mode annotations | ✅ `node.annotations = [...]` works (see the append gotcha in `native-patterns.md` §5) |
| Full design-system access | ✅ **but you must scope it** — `get_libraries` → `search_design_system(..., includeLibraryKeys)`. Unscoped search returns org-wide noise and hides the file's own library |
| Import Make images/assets | ✅ read the Make image resource → `upload_assets` (see §6) |
| Round-trip / diff-in-place | ⚠️ possible by reading + patching nodes (§3); no first-class diff API |
| `use_figma` return values | ❌ the tool does **not** surface a script's return value — read state back or render a report to canvas to verify |

So the genuine wins of running natively are **fewer round-trips and richer library/context access**,
not new capability. Build with `use_figma` and don't assume anything is impossible until a call
fails. The *method* is portable; mechanics live in `native-patterns.md`, with shared build
primitives in `../figma-patterns.md`.

---

## Non-negotiable rules

**Rule 1 — Instance first, primitive last.** Never draw an element from `createFrame`/
`createRectangle`/`createText` until the discovery pass (Phase 2) confirms the library has nothing
for it. Exact match → import + set variants. Component exists but variant missing → instance it and
override the differing property per-instance (fill, text, size), plus a DS-gap note. No match at
all → primitive, plus a DS-gap note. Repeated elements (badges, nav, inputs, avatars) are all
instances, never hand-made copies.

**Rule 2 — Bind, don't hardcode.** Typography binds to library **text styles**; color, spacing,
radius, and border-width bind to **variables**; page/surface backgrounds bind to a surface
variable. Raw hex, ad-hoc font sizes, and raw stroke weights are drift, not parity.

**Rule 3 — Never create master components.** Do not call `figma.createComponent()` /
`createComponentSet()` in the working file. DS-gap *proposals* go on a dedicated proposals page or
branch (Phase 6), never into the live spec.

**Rule 4 — Never omit an element.** Every visible element appears in the output — instance or
primitive.

**Rule 5 — Interactions are real and annotated.** Each state-advancing transition is wired as an
actual prototype reaction (Phase 5) *and* carries a node-attached annotation. Keep annotations
flow-critical: 2–4 per frame (state + primary action + non-obvious tap targets); skip
close/cancel/nav/decorative. Never use floating text boxes as a substitute for `node.annotations`.

---

## The workflow

### Phase 0: Resolve source and target; enumerate libraries

**Source.** Identify the prototype: a Figma Make file, a linked repo, or code the user points at.

**Target.** Default to a new page in the target file named `[Spec] <feature>`. If a matching spec
page exists, Phase 7 (round-trip) applies instead of a fresh build.

**Enumerate the file's libraries — do not skip this.** Call `get_libraries(fileKey)` and read
`libraries_added_to_file`. These are the libraries actually subscribed to the target file; capture
their `libraryKey`s. Phase 2 scopes discovery to these keys. **A plain `search_design_system`
query with no `includeLibraryKeys` searches everything the account can see and buries the file's
own library in org-wide noise — that is what makes the skill "miss" a DS that is right there.**
If `libraries_added_to_file` is empty, say so; only then is a primitive-heavy build expected.

### Phase 1: Analyze the prototype

Read all source before touching the canvas.

**Reading a Figma Make source (important — Make is not a design file):**
- `get_design_context(fileKey=<makeKey>, nodeId="0:1")` is the **only** reader that works on a
  `/make/` file — it returns the React source tree as resource links. Read `App.tsx`, `routes.tsx`,
  and each screen/component with `ReadMcpResource`.
- `get_metadata`, `get_screenshot`, and `get_variable_defs` **all reject `/make/` URLs** — don't
  rely on them for a Make source.
- **No viewport dimensions are available** (screens are usually `size-full`). Infer the canvas size
  from imported asset dimensions or the largest fixed layout, or ask the user. State the assumption.

Capture:
- **Component inventory** — for each UI element: name, props/variants, visual structure, interactive
  or not. This is the Phase 6 completeness checklist.
- **Interaction flows** — trigger → state change → before/after → branches. Record state count;
  this becomes the Phase 5 wiring. (`react-router` `navigate()` + `setState` map directly to reactions.)
- **Layout** — fixed vs. scrollable, what persists (nav/header). Extract content dimensions only;
  don't recreate a device shell as a wrapper frame.
- **Tokens** — read the source's own token file (e.g. `theme.css`: colors, radii, type scale, font
  families). These are CSS variables, **not** Figma variables yet — Phase 2 reconciles them.
- **Assets** — list every image/SVG the source references (`get_design_context` returns Make image
  resource links). Phase 1b brings them across.

### Phase 1b: Bring over images and assets

Every image and SVG the prototype uses must land in the Figma file — placeholders are drift. See
`native-patterns.md` §6. In short: read each Make image resource (→ local file; large reads can
drop, retry), then `upload_assets` and POST the bytes onto the receiving node. SVGs go through
`figma.createNodeFromSvg()` in `use_figma`, not `upload_assets`.

### Phase 2: Discovery pass over the file's libraries

Do this **before drawing anything** — default is *instance*, not primitive. Scope every call to the
`libraryKey`s from Phase 0.
- **Components** — `search_design_system(fileKey, query, includeLibraryKeys=[...])` with broad and
  category queries; `list_file_components_for_code_connect` enumerates published components. Record
  variant prop names for each match. (A real run surfaced a `Nav Button` set only once scoped —
  unscoped it was invisible.)
- **Variables** — same scoped search with `includeVariables:true` for color / spacing / radius /
  border-width. Record keys.
- **Text styles** — the library's typography styles. Body, labels, headings bind to these.
- **Token reconciliation** — map the source's CSS variables (Phase 1) to these DS variables/styles
  by role (`--primary` → the DS brand color var, etc.). Bind the DS var in Phase 4; where no DS var
  exists, use the raw source value and note the gap.

Build the mapping table: `code element → DS match (scoped) → build approach → bound style/variables
→ drift note`. A "primitive" row is valid only when scoped discovery found nothing — never because a
variant was missing, and never because the search wasn't scoped.

### Phase 3: Plan the page structure

```
Page: "[Spec] <Feature>"
  Section "Flow 1: <name>"
    Frame "1.1 — <state>"   ← left-to-right within a flow
    Frame "1.2 — <state>"
    Frame "1.3a — Success"  ← branches stacked vertically
    Frame "1.3b — Error"
  Section "Flow 2: <name>" …
```

Frame size = measured viewport. Scrollable states → full content height with a fold marker at the
viewport line. ~200px between frames, ~400px between sections.

### Phase 4: Build the frames

- **DS instances** — import by key, `setProperties` with exact variant names. Missing variant →
  per-instance override + DS-gap note (Rule 1).
- **Bind** text styles and variables per Rule 2 (`native-patterns.md` §1, `../figma-patterns.md`).
- **Primitives** — only for confirmed no-match elements; still bind variables/text styles where
  they exist. Fixed-size square/circle containers (avatars, count badges) use fixed sizing on
  **both** axes so they don't collapse.
- **Annotate as you build** — attach flow-critical annotations to the node itself, in scope, right
  after creating it. Two categories: Interaction (blue), DS Gap (orange). **Set the full
  annotations array in one assignment** — do NOT append by reading the getter
  (`node.annotations = [...node.annotations, entry]` throws on the 2nd annotation; see
  `native-patterns.md` §5). Collect a node's entries and assign once.
- **Verify by reading back, not by return value** — `use_figma` does not surface a script's return
  value. To confirm reactions/annotations landed, read the state back in a later call (e.g.
  `node.reactions.length`) or render a short report onto the canvas, then screenshot it.

### Phase 5: Wire the interactions (native)

For each transition mapped in Phase 1, add a **real prototype reaction** from the trigger node to
the destination frame, and set flow starting points per section. See `native-patterns.md` §2.
The annotation describes intent; the reaction makes it clickable. Branch conditions (success/error)
become separate reactions to the branch frames.

### Phase 6: DS-gap proposals

Collect every drift note (no-match primitives, missing variants, unbound values). On a dedicated
`[DS proposals] <feature>` page — or a branch if the surface supports it — recreate each gap as a
*proposed* component/variable with a note describing the need and where it's used. Never add
masters to the live spec. This turns drift into actionable signal for the DS team.

### Phase 7: Round-trip (re-run against an existing spec)

When a spec page already exists and the prototype changed: read the existing frames, match by name
to the new state list, and **patch in place** — update changed props/instances, add new state
frames, mark removed ones. Don't rebuild from scratch. See `native-patterns.md` §3.

### Phase 8: Verify and hand off

- **Completeness** — every mapping-table row has a named layer; DS matches (scoped to the file's
  libraries) are instances, incl. repeated elements; **every source image/SVG imported — no
  placeholders**; text/variable bindings present; square containers square; no masters created in
  the live file; every transition has a reaction. Verify by reading state back (§8), not by trusting
  a silent `use_figma` return.
- **Review lifecycle** — add a flow-overview frame (feature, flow list, legend, DS-gap list, open
  questions), mark frames dev-ready, and post the entry point as a comment for reviewers.
- **Present** — file/page link + deep-link to the first flow, flow/frame counts, instances vs.
  primitives, DS-gap list, any deviations from measurements.

---

## Reference

- `native-patterns.md` — §1 variable/text-style binding · §2 prototype reactions & flow starts ·
  §3 round-trip diff · §4 DS-gap proposals · §5 annotation set-once fix · §6 image/SVG import ·
  §7 scoped library discovery · §8 reading results back.
- `../figma-patterns.md` — shared build primitives (frames, component import, primitives,
  node annotations) from the `prototype-to-figma` skill.
