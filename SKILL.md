---
name: prototype-to-figma
description: >
  Converts a working Claude Code prototype into a structured Figma design file — exploding each
  interaction flow into separate frames, using the target file's design system components, and
  annotating interaction details natively in Figma. Use this skill whenever the user asks to
  turn a prototype into Figma frames, generate design specs from a prototype, create a Figma
  handoff from code, explode a prototype into states, or make a prototype reviewable by
  cross-functional partners. Also trigger when the user mentions "prototype to Figma",
  "design review from prototype", "async feedback on prototype", "break prototype into flows",
  "Figma specs from code", or wants to make a working prototype legible to non-technical
  stakeholders. Even if the user just says "put this in Figma" or "make this reviewable" in the
  context of a prototype, use this skill.
---

# Prototype → Figma

This skill takes a working Claude Code prototype and produces a structured Figma file that
cross-functional partners can review asynchronously. The output has two equal goals:

1. **Pixel-perfect visual parity** — correct layout, colors, spacing, typography, content, and
   component hierarchy, derived directly from the source code (CSS, Tailwind, inline styles).
2. **Interaction legibility** — every interactive element and state transition annotated on its
   node so reviewers understand what's tappable, what it does, and how states connect.

**This skill works across all Figma MCP clients.** The output format adapts to what your client
supports — see [Client Compatibility](#client-compatibility) below.

---

## Four non-negotiable rules

### Rule 0: Never use browser capture or HTML-to-design tools

**Do not** call `generate_figma_design`, `upload_assets` with a localhost URL, or any other
tool that opens a browser or captures a screenshot to import into Figma.

Read the source code (CSS modules, Tailwind, inline styles, React component trees) and build
frames programmatically via `use_figma`. A browser capture produces a flat image with no layers,
no DS components, and no annotations — the opposite of what this skill produces.

### Rule 1: Never create new Figma components

**Do not** call `figma.createComponent()` or `figma.createComponentSet()`. These pollute the
design system with components that don't belong there.

When a prototype element has no DS match: build it from primitives (`figma.createFrame()`,
`figma.createRectangle()`, `figma.createText()`), add a **DS Drift annotation** explaining
what was missing, and list it in the Phase 6 summary.

### Rule 2: Never omit a prototype element

Every visible element must appear in the Figma output — DS instance or primitive. Skipping
elements because you couldn't find a DS match is the most common cause of incomplete outputs.

### Rule 3: Annotations must be attached to nodes, and must be flow-critical only

**Annotations are what make this output useful for review.** A frame with no annotations is
just a static image. But annotating every tap target creates noise that buries the actual flow.

Annotate using `node.annotations = [...]` on the actual layer — this surfaces in Figma Dev Mode
on the correct element. **Never create floating text boxes or separate annotation frames** as a
substitute. If the native API fails, fall back to `figma-patterns.md` Section 11.

Two categories only:
- **Interaction** (blue) — state transitions, primary CTAs, grouped controls
- **DS Drift** (orange) — missing DS component, missing variant, or unbound color/spacing

**Annotate (Interaction):**
- **Frame-level** — one annotation per frame: what state this is and what the primary action does / where it goes next
- **State-advancing CTAs** — the action(s) that trigger a state change (e.g., Submit, Confirm, Create goal, Move to completed)
- **Grouped controls** — annotate the container once, not each item (annotate the amount picker row, not each chip; annotate the story pill rail, not each pill)
- **Non-obvious tap targets** — elements that open a new state but don't look like buttons (e.g., a goal card that opens a detail sheet)

**Do not annotate:**
- Close / cancel / dismiss buttons
- Nav bar buttons and tab items
- Secondary utility actions (settings, prototype controls)
- Individual chips or pills within a grouped picker
- Backdrops and overlays
- Decorative elements

**Target: 2–4 Interaction annotations per frame.** A reviewer should be able to read all annotations on a frame in under 30 seconds and understand the complete flow.

Every element with a DS Drift note from Phase 2 must have a DS Drift annotation.

---

## Client Compatibility

| Tier | Inspect | Write | Code Connect | Output |
|---|---|---|---|---|
| **Full** | ✅ | ✅ | ✅ | Figma file with DS components, native annotations, Code Connect links |
| **Write** | ✅ or ❌ | ✅ | ❌ | Figma file with native annotations |
| **Inspect-only** | ✅ | ❌ | ❌ | Prototype Spec Document (markdown) |
| **None** | ❌ | ❌ | ❌ | Prototype Spec Document from code analysis alone |

| Client | Inspect | Write | Code Connect |
|---|---|---|---|
| Claude Code / Desktop / Cursor / VS Code / Copilot CLI / Augment / Factory | ✅ | ✅ | ✅ |
| Android Studio / Gemini CLI / Kiro / Amazon Q / Openhands | ✅ | ❌ | ❌ |
| Replit | ❌ | ✅ | ❌ |

---

## Figma MCP tools

**Inspect:** `get_design_context` (primary), `get_metadata` (structure overview),
`search_design_system` (components + variables), `get_variable_defs` (exact token values),
`get_screenshot` (only if user explicitly requests a visual preview).

**Write:** `use_figma` (primary workhorse — all frame building and annotation),
`whoami`, `create_new_file`.

**Code Connect:** `get_code_connect_map`, `get_code_connect_suggestions`,
`get_context_for_code_connect`, `send_code_connect_mappings`.

---

## The workflow

### Phase 0: Resolve file and detect capabilities

**File:** Extract `fileKey` from a user-provided URL (`figma.com/design/:fileKey/...`). If no
URL is given, call `whoami` then `create_new_file` — never block on a missing URL.

**Capabilities:** Attempt `get_metadata` to confirm Inspect tools. Check tool list for
`use_figma` (Write) and `get_code_connect_map` (Code Connect). Announce what you have.

**Route:** Write tools available → Phases 1–6. Write tools unavailable → Phases 1–3, then
[Prototype Spec Document](#prototype-spec-document--inspect-only-output).

---

### Phase 1a: Analyze the prototype

Read all prototype source files before touching Figma.

**Component inventory** — for every UI component, record: name, props/variants in use, visual
structure (what's inside it), and whether it is interactive. This list is the completeness
checklist for Phase 4.

**Interaction flows** — for each flow, map: trigger → state change → before/after states →
branching paths (success / error / edge cases). Include state count.

**Layout structure** — note what's fixed vs. scrollable, what persists across states (nav,
header), and the viewport dimensions. For the phone/device frame: if the prototype wraps screens
in a `.phone`, `.device-frame`, or similar shell, **extract only the content dimensions** and
use those for Figma frame size — do not recreate the shell as a wrapper frame in Figma.

**CSS measurement** — read the CSS modules, Tailwind config, and inline styles for every element
in your inventory. Record exact values: frame width/height, padding, gap, border-radius,
font-size, line-height, color, background. These are the source of truth for Phase 4. Do not
guess or approximate — read the file.

---

### Phase 1b: Present scope, get selection, confirm

**Skip if the prototype has ≤ 2 flows.** Proceed directly to Phase 2 with all flows.

For 3+ flows, present a categorized flow list with state counts, ask "which flows and how much
detail (happy path / all states)?", and wait for a reply before proceeding. Confirm the
interpretation explicitly before starting Phase 2.

---

### Phase 2: Map components to the design system

For every component in the Phase 1a inventory, determine DS match or primitive.

**Search strategy** (try in order before declaring "no match"):
1. `search_design_system(fileKey, query="ComponentName", includeComponents=true)`
2. Try alternate names: Alert/Banner/Toast/Notification, Dropdown/Select/Menu, Tag/Chip/Badge, etc.
3. Search by visual category: "input", "navigation", "feedback", "card"
4. Check if a parent component covers it (e.g., "Card" covers `Card.Header`)

**For found components:** call `get_context_for_code_connect` to get exact variant prop names
before setting properties in Phase 4.

**For DS variables:** `search_design_system(..., includeVariables=true)` for colors and spacing.
Record any variable keys found — bind them in Phase 4 instead of hardcoding raw values.

**Build the mapping table — every row needs a build approach and drift note:**

| Code component | DS match | Key | Build approach | Drift note |
|---|---|---|---|---|
| `<Button variant="primary">` | Button | abc123 | Import + setProperties | ✅ Clean match |
| `<CustomCard>` | — | — | Primitives | ⚠️ No DS match — searched: Card, Panel, Tile |
| `<DataTable sortable>` | Table | def456 | Import + setProperties | ⚠️ No `sortable` variant in Figma |

Any row with ⚠️ must get a DS Drift annotation in Phase 4.

---

### Phase 3: Plan the page structure

```
Page: "[Feature Name] — Prototype Flows"
  Section: "Flow 1: [Flow Name]"
    Frame: "1.1 — [State description]"   ← left-to-right within a flow
    Frame: "1.2 — [State description]"
    Frame: "1.3a — Success"
    Frame: "1.3b — Error"               ← branches stacked vertically
  Section: "Flow 2: [Flow Name]"
    ...
```

Frame size: match the CSS-measured viewport (e.g., 390×844 mobile, 1440×900 desktop). For
scrollable states, set frame height to full content height — never clip at viewport height. Add
a fold marker line at the viewport height. ~200px gaps between frames, ~400px between sections.

---

### Phase 4: Build in Figma

Break the build into multiple `use_figma` calls — one per flow or per frame group.

**Before writing any node:** confirm the CSS measurements from Phase 1a. Set every frame's
`width` and `height` to the measured values. Set every padding, gap, border-radius, font-size,
and color from the CSS — not from memory or approximation.

**Placement:** scan the target page with `get_metadata` first. If matching frames/sections exist
for this feature, insert beside them. If not, create a named section:
`[Prototype] <feature name>`. Never place frames at an arbitrary far-Y without reporting
coordinates and a deep-link in the Phase 6 summary.

**DS component instances** — see `figma-patterns.md` Section 3 for `importComponentByKeyAsync`
+ `setProperties`.

**Primitives** — see `figma-patterns.md` Section 4. Use the CSS-measured values for all
dimensions, radii, and colors.

**DS variable binding** — for every fill, radius, and spacing where a variable key was found in
Phase 2, bind it instead of hardcoding:

```javascript
const colorVar = await figma.importVariableByKeyAsync(colorVarKey);
node.setBoundVariableForPaint('fills', 0, colorVar);
const radiusVar = await figma.importVariableByKeyAsync(radiusVarKey);
node.setBoundVariable('cornerRadius', radiusVar);
```

Fall back to raw CSS values only if no DS variable exists for that role.

**Annotations — attach to nodes during build, not after**

Set up two annotation categories once at the start of your first `use_figma` call:

```javascript
let interactionCat = null, dsDriftCat = null;
try {
  const existing = await figma.annotations.getAnnotationCategoriesAsync();
  interactionCat = existing.find(c => c.label === 'Interaction')
    ?? await figma.annotations.addAnnotationCategoryAsync({ label: 'Interaction', color: 'blue' });
  dsDriftCat = existing.find(c => c.label === 'DS Drift')
    ?? await figma.annotations.addAnnotationCategoryAsync({ label: 'DS Drift', color: 'orange' });
} catch(e) { /* native API unavailable — use layer name fallback below */ }
```

As you build each node, **annotate it immediately** — do not defer annotation to a separate
pass. Keeping the node reference in scope is the only reliable way to attach to the right layer:

```javascript
// Interaction annotation — on the node itself, right after creating it
const ctaButton = figma.createFrame(); // ...build it...
ctaButton.name = 'CTA: Submit';
ctaButton.annotations = [{
  label: 'Tap → submits form; on 200 navigates to success screen; on 4xx shows inline error',
  ...(interactionCat ? { categoryId: interactionCat.id } : {})
}];

// DS Drift annotation — on any primitive or mismatched DS component
const customCard = figma.createFrame(); // ...build it...
customCard.name = 'Card: Goal (primitive)';
customCard.annotations = [{
  label: 'DS Drift: no Card/Goal component found in library (searched: Card, Panel, Tile). DS gap for design team.',
  ...(dsDriftCat ? { categoryId: dsDriftCat.id } : {})
}];

// Fallback if native API failed: encode in layer name
if (!interactionCat) ctaButton.name += ' [Interaction: tap → submit form]';
if (!dsDriftCat) customCard.name += ' [DS Drift: no Card/Goal in DS]';
```

**Annotation checklist per frame:**
- [ ] Frame itself has one annotation: current state + what the primary action does / where it leads
- [ ] Each state-advancing CTA has an annotation (trigger → result)
- [ ] Grouped controls (pickers, pill rails, option lists) have one annotation for the group, not per item
- [ ] Every primitive or mismatched DS component has a DS Drift annotation

Completeness check: before moving to the next frame, verify every row from the Phase 2 mapping
table that belongs in this frame is present in the layer tree.

---

### Phase 5: Add a flow overview frame

One frame at the top of the page:
- Feature name, flow list, frame count
- Annotation legend: blue = Interaction, orange = DS Drift (filterable in Dev Mode)
- DS gaps: list all primitives and mismatched components for the design team
- Open questions: flag any ambiguous prototype behavior
- Scope note if flows were omitted (offer to re-run skill to add them)

---

### Phase 6: Verify and present

**6a. Completeness check** — via `get_metadata` on the output, verify:
- Every row in the Phase 2 mapping table has a named layer in the correct frame
- Frame dimensions match the CSS-measured viewport
- No new master components were created in local assets
- DS variable bindings are present on at least color fills and border-radii

Fix anything missing with a follow-up `use_figma` call before presenting.

**6b. Code Connect** *(optional)* — `get_code_connect_suggestions` → review → `send_code_connect_mappings`.

**6c. Present to user:**
- Figma file URL (always) + deep-link to the section node if not at the canvas origin
- Flow count, frame count
- DS instances vs. primitives used
- Full list of DS gaps (primitives) for the design team
- Any deviations from CSS measurements and why
- Steps skipped due to capability limits

---

## Prototype Spec Document — Inspect-only output

When Write tools are unavailable, produce a structured markdown document using the template in
`figma-patterns.md` Section 12.

---

## Important principles

**Read the CSS, then build.** Every dimension, color, radius, and spacing value must come from
the source files — not from memory or approximation. Guessing is what causes visual drift.

**Annotate on the node, not the canvas.** `node.annotations = [...]` is what surfaces in Dev
Mode. Floating text boxes and separate annotation frames are noise, not annotations.

**Name everything.** "Frame 47" is useless. "2.3a — Save success with toast" tells a reviewer
exactly where they are.

**Annotate flows, not tap targets.** The goal is for a reviewer to read all annotations on a
frame in under 30 seconds and understand the complete flow. Annotate the state transition, not
the mechanism — one annotation on the amount picker row beats four identical annotations on each
chip inside it.

**Be explicit about DS gaps.** List every primitive in the summary so the design team knows what
components are missing from the DS. This turns unmatched elements from a silent failure into
actionable signal.

---

## Reference: Figma Plugin API patterns

See `figma-patterns.md` for Plugin API patterns — frame creation, component importing,
primitive helpers, `addNoDsMatchBadge`, auto-layout, text nodes, annotation setup, and the
Prototype Spec Document template. Read this before your first `use_figma` call.
*(Write-tier clients only.)*
