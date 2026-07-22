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

This is the same proven method as the external `prototype-to-figma` skill, but the native surface
removes its biggest constraint (keyhole access to the library) and unlocks capabilities MCP can't
reach: real prototype interactions, round-trip sync, and DS-gap proposals.

---

## What's different when you're native

| Capability | External (MCP) | Native agent (this skill) |
|---|---|---|
| Design-system access | Keyhole — `search_design_system` queries | Full catalog: every component, variant, variable, text style |
| Interactions | Text annotations only | **Real prototype connections** between frames |
| Direction | One-way (code → Figma) | **Round-trip** — diff and patch existing frames in place |
| DS gaps | Listed in a summary | **Proposed** components/variables on a proposals page or branch |
| Review | Share a link | Native comments, dev-ready status, branch |

Everything below leans on these. The *method* is portable; the *mechanics* use the native Plugin
API (`figma.*`) — see `native-patterns.md`, and `../figma-patterns.md` for shared build primitives
(frames, component import, variable binding, node annotations).

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

### Phase 0: Resolve source and target

**Source.** Identify the prototype: a Figma Make file/artifact, a linked repo, or code the user
points at. For Make, read the generated component tree and interaction logic directly.

**Target.** Default to a new page in the current file named `[Spec] <feature>`. If a matching
spec page exists, Phase 7 (round-trip) applies instead of a fresh build.

**Capabilities.** Confirm library access, prototyping (`setReactionsAsync`), and whether the
surface supports branches. Announce what you have; degrade gracefully (e.g. proposals page if no
branch).

### Phase 1: Analyze the prototype

Read all source before touching the canvas.
- **Component inventory** — for each UI element: name, props/variants in use, visual structure,
  interactive or not. This is the Phase 6 completeness checklist.
- **Interaction flows** — trigger → state change → before/after → branches (success/error/edge).
  Record state count. This becomes the prototype wiring in Phase 5.
- **Layout** — fixed vs. scrollable, what persists (nav/header), viewport dimensions. Extract the
  content dimensions only; don't recreate a device shell as a wrapper frame.
- **Measurements** — exact width/height, padding, gap, radius, font, color per element. Source of
  truth for Phase 4.

### Phase 2: Discovery pass over the native library

Do this **before drawing anything** — default is *instance*, not primitive.
- **Components** — enumerate the linked libraries' full component set (not just names the code
  uses). Record variant prop names for each match.
- **Variables** — color, spacing, radius, border-width tokens. Record ids/keys.
- **Text styles** — the library's typography styles (font/size/weight/line-height). Body, labels,
  and headings bind to these.

Build the mapping table: `code element → DS match → build approach → bound style/variables →
drift note`. A "primitive" row is valid only when discovery found nothing — never because a
variant was missing.

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
  after creating it. Two categories: Interaction (blue), DS Gap (orange).

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

- **Completeness** — every mapping-table row has a named layer; DS matches are instances (incl.
  repeated elements); frames match measured viewport; text/variable bindings present; square
  containers square; no masters created in the live file; every transition has a reaction.
- **Review lifecycle** — add a flow-overview frame (feature, flow list, legend, DS-gap list, open
  questions), mark frames dev-ready, and post the entry point as a comment for reviewers.
- **Present** — file/page link + deep-link to the first flow, flow/frame counts, instances vs.
  primitives, DS-gap list, any deviations from measurements.

---

## Reference

- `native-patterns.md` — net-new native mechanics: prototype reactions & flow starting points,
  text-style/variable binding, round-trip diff, DS-proposals page.
- `../figma-patterns.md` — shared build primitives (frames, component import, primitives,
  node annotations) from the `prototype-to-figma` skill.
