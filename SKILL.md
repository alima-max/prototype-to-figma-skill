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

This skill takes a working Claude Code prototype (a sandbox environment templated from the user's
live app, using their real component library) and produces a structured Figma file that
cross-functional partners can review asynchronously.

**Primary success metric:** visual and structural parity with the running localhost prototype —
layout, spacing, type scale, colors, component hierarchy, and per-step UI copy — within the
constraints of the MCP toolchain and the chosen breakpoint. Annotations are a required layer on
top of that parity, not a substitute for it.

**This skill works across all Figma MCP clients.** The output format adapts to what your client
supports — see [Client Compatibility](#client-compatibility) below.

## When to use this skill

The user has built (or is working on) a prototype in Claude Code and wants to:
- Get async design feedback from cross-functional partners
- Document the interaction flows in a format designers and PMs can comment on
- Bridge the gap between a working prototype and a design review artifact

---

## Four non-negotiable rules

These rules exist to prevent the bugs most commonly reported by users of this skill:

### Rule 0: Never use browser capture or HTML-to-design tools

**Do not** call `generate_figma_design`, `upload_assets` with a localhost URL, or any other
tool that opens a browser, renders the prototype, or captures a screenshot of the running app
to import into Figma.

This skill achieves pixel fidelity by **reading source code** (CSS modules, Tailwind, inline
styles, React component trees) and building Figma frames programmatically via `use_figma`. A
browser capture produces a flat image with no layers, no DS components, no variable bindings,
and no annotations — the opposite of what this skill is for.

If you are tempted to use `generate_figma_design` to "get a head start", don't. Read the code.

### Rule 1: Never create new Figma components

**Do not** call `figma.createComponent()` or `figma.createComponentSet()` under any
circumstances. These calls create new master components inside the target Figma file, polluting
the design system with components that don't belong there.

When a prototype element has no match in the design system:
- Build it from **primitives** (`figma.createFrame()`, `figma.createRectangle()`,
  `figma.createText()`) using the prototype's visual structure as reference
- Add a **"No DS match"** badge overlay so reviewers and designers know this element needs
  a DS component created for it
- List it in your final summary so the design team knows what's missing from the DS

### Rule 2: Never omit a prototype element

**Every visible element** in the prototype must appear in the Figma output — either as a DS
component instance (if a match was found) or as a primitive approximation (if not).

Do not skip an element because you couldn't find its DS match. Skipping elements is what causes
reviewers to see a Figma file that's missing whole sections of the prototype UI.

### Rule 3: Every interactive element must be annotated — on every platform

**Annotations are the primary deliverable of this skill.** A frame with no annotations is just a
screenshot of a UI, not a design spec. Do not skip the annotation phase under any circumstances,
regardless of which AI editor or client you are running on.

**This rule applies on every platform** — Claude Code, Codex, Cursor, VS Code, Copilot CLI,
Augment Code, and all others. Annotations must be present in the output.

Use the defensive helpers in `figma-patterns.md` Section 11 — they handle API errors (e.g.,
categories that already exist) and fall back to canvas text overlays if the native
`figma.annotations` API is unavailable. **Never silently drop annotations because of an API
error.** If native annotations fail, use the text fallback. If text placement fails, embed the
annotation text directly in the frame or layer name. Something is always better than nothing.

**Every frame must have at minimum:**
- One annotation per clickable/interactive element (trigger + result)
- One annotation per state frame (what caused this state, how the user leaves it)
- One annotation per element with a DS drift note (no DS match, or DS component missing a required variant/behavior)

Annotate only to communicate interaction behavior or DS drift — do not annotate for validation rules, API details, accessibility notes, or other concerns. Always use `node.annotations = [...]` on actual interactive elements — not colored rectangles on the canvas.

---

## Client Compatibility

Different Figma MCP clients support different tool subsets. This skill detects what's available
and automatically uses the best possible workflow for your client.

### Capability tiers

| Tier | Inspect tools | Write tools | Code Connect tools | Output |
|---|---|---|---|---|
| **Full** | ✅ | ✅ | ✅ | Figma file with DS components, native annotations, and Code Connect links |
| **Write** | ✅ or ❌ | ✅ | ❌ | Figma file with native annotations (Code Connect phase skipped) |
| **Inspect-only** | ✅ | ❌ | ❌ | Prototype Spec Document (rich markdown, shareable, built for async review) |
| **None** | ❌ | ❌ | ❌ | Prototype Spec Document from code analysis alone |

### Which clients support what

| Client | Inspect | Write | Code Connect |
|---|---|---|---|
| Claude Code | ✅ | ✅ | ✅ |
| Claude Desktop | ✅ | ✅ | ✅ |
| Cursor | ✅ | ✅ | ✅ |
| VS Code | ✅ | ✅ | ✅ |
| Copilot CLI | ✅ | ✅ | ✅ |
| Augment Code | ✅ | ✅ | ✅ |
| Factory | ✅ | ✅ | ✅ |
| Firebender | ✅ | ✅ | ✅ |
| Codex by OpenAI | ✅ | ✅ | ✅ |
| Android Studio | ✅ | ❌ | ❌ |
| Gemini CLI | ✅ | ❌ | ❌ |
| Kiro | ✅ | ❌ | ❌ |
| Amazon Q | ✅ | ❌ | ❌ |
| Openhands | ✅ | ❌ | ❌ |
| Replit | ❌ | ✅ | ❌ |
| Warp | varies | varies | ❌ |

---

## Figma MCP tools — by capability tier

### Inspect tier tools
- **`Figma:get_design_context`** — Primary inspection tool. Returns reference code, a screenshot,
  and contextual metadata for a node. Use to understand existing Figma file structure.
- **`Figma:get_metadata`** — XML overview of file/page structure: node IDs, layer types, names,
  positions, sizes. Use for quick structural overview.
- **`Figma:get_screenshot`** — Generate a screenshot of any node. Use only if the user explicitly asks to see a visual preview of a frame.
- **`Figma:search_design_system`** — Search the target file's linked libraries for components,
  variables, and styles.
  - Components: `search_design_system(fileKey, query="Button", includeComponents=true)`
  - Color tokens: `search_design_system(fileKey, query="primary", includeVariables=true)`
  - Styles: `search_design_system(fileKey, query="heading", includeStyles=true)`
- **`Figma:get_variable_defs`** — Get exact values for design tokens (hex colors, pixel values).
  Apply these to primitive elements to keep them visually on-system.

### Write tier tools
- **`Figma:use_figma`** — Primary workhorse. Executes Figma Plugin API JavaScript to create
  pages, frames, import components, build layouts, and add native annotations. Break complex
  builds into multiple calls rather than one giant script.
- **`Figma:whoami`** — Get authenticated user info and plan keys. Required before `create_new_file`.
- **`Figma:create_new_file`** — Create a new blank Figma design file.
- **`Figma:create_design_system_rules`** — Generate design system rules for the repo (optional
  follow-up).

### Code Connect tier tools
- **`Figma:get_code_connect_map`** — Returns mapping between Figma node IDs and code component
  locations. Use to confirm DS component ↔ code component matches.
- **`Figma:get_code_connect_suggestions`** — AI-suggested mappings between Figma and code
  components.
- **`Figma:get_context_for_code_connect`** — Structured metadata for a Figma component: property
  definitions, variant options, descendant tree. Use to understand exact variant props before
  importing.
- **`Figma:add_code_connect_map`** — Map a single Figma node to a code component.
- **`Figma:send_code_connect_mappings`** — Save multiple Code Connect mappings in bulk.

---

## The workflow

### Phase 0: Resolve target file and detect capabilities

#### Step 0a: Resolve the target Figma file

Do this before any other step.

**If the user provided a Figma file URL:**
Extract the `fileKey` from the URL and use it throughout the workflow.
- `figma.com/design/:fileKey/...` → use `:fileKey`
- `figma.com/board/:fileKey/...` → FigJam file, use `:fileKey`

**If the user did NOT provide a Figma file URL:**
Create a new file for them automatically — do not ask them for a link.

```
1. Call Figma:whoami to get the authenticated user's planKey
2. Call Figma:create_new_file(name="[Feature Name] — Prototype Flows", planKey=<planKey>)
3. Use the returned fileKey for all subsequent operations
4. Share the new file URL with the user at the end of Phase 6
```

Never block on a missing file URL. If the user hasn't provided one, create the file.

#### Step 0b: Detect available capabilities

1. **Check for Inspect tools** — Attempt `Figma:get_metadata` on the target file (either
   the one the user provided or the newly created one). If it succeeds, Inspect tools are
   available.
2. **Check for Write tools** — Check your tool list for `use_figma`.
3. **Check for Code Connect tools** — Check your tool list for `get_code_connect_map`.
4. **Announce what you're doing.** Example (new file):
   > "No Figma file provided — I've created a new one for you. I'll build the prototype
   > flows there and share the link when done."

   Example (existing file):
   > "Got it — I'll add the prototype flows to your existing file. I have Inspect + Write
   > tools; Code Connect linking will be skipped."

**Route:**
- Write tools available → Phases 1–6 (full Figma build)
- Write tools unavailable → Phases 1–3, then [Prototype Spec Document](#prototype-spec-document-output-for-inspect-only-clients)

---

### Phase 1a: Analyze the prototype

Before touching Figma, thoroughly analyze the prototype source code.

**1. Component inventory**
Read the prototype files and list every UI component used. For each component, note:
- The component name as used in code (e.g., `<Button variant="primary">`, `<Modal>`, `<DataTable>`)
- Props/variants being used (size, state, variant)
- Any component compositions (e.g., a Card containing a Header, Body, and ActionBar)

**This inventory is the completeness checklist for Phase 4.** Every component listed here must
appear in the Figma output.

**2. Interaction flows**
Identify every distinct user flow in the prototype:
- What triggers a state change? (click, hover, form submission, data load, etc.)
- What are the before/after states?
- Are there intermediate states? (loading, validating, animating)
- What are the error/edge-case states? (empty, error, permission denied, timeout)
- Are there branching paths? (success vs. failure, different user roles)

Map these as a flow graph:
```
Flow: "Create new item"
  1. Dashboard (default) → user clicks "+ New" button
  2. Creation modal (empty form) → user fills fields
  3. Creation modal (filled, valid) → user clicks "Save"
  4. Creation modal (saving/loading) → API responds
  5a. Dashboard (with new item, success toast) ← success
  5b. Creation modal (with error banner) ← validation error
```

For each flow, also note: the **feature category** it belongs to (e.g., "Core Purchase",
"Account", "Order Management"), a **one-sentence purpose** describing what a reviewer would
learn from it, and a **state count** including all branches and error paths. This is the input
to Phase 1b.

**3. Layout and structure**
Note the overall page structure: navigation, sidebars, content areas, overlays.
Identify what stays constant across states vs. what changes.

**4. Scrollable states**
For each state frame, note whether the content is taller than the viewport:
- Look for `overflow-y: scroll/auto`, `overflow: scroll/auto`, or body-level page scroll in the prototype CSS/styles
- Look for long lists, feeds, tables, or multi-section pages where content clearly exceeds one screenful
- Estimate the full content height (e.g. 1800px for a two-screen-tall page)

Mark each scrollable state in your inventory — these frames need special handling in Phase 4.

---

### Phase 1c: Pixel parity — measure from code

This phase is mandatory. Run it immediately after Phase 1a, before Phase 2 and before building
anything in Figma. Its output is the source of truth for every frame dimension, spacing value,
and color in Phase 4.

**Viewport / frame dimensions**

Locate the outermost container class that defines the prototype's viewport for the chosen
breakpoint — e.g., `.phone`, `.container`, `max-w-*`, the root layout div's inline style, or
the Tailwind responsive prefix. Read the prototype's CSS modules, Tailwind config
(`tailwind.config.js`), and any inline styles to extract:
- The frame width (and fixed height if set). Example: `.phone { width: 390px; height: 844px }` →
  every state frame must be **390×844** in Figma unless the user specifies otherwise.
- `max-width` constraints for content areas (e.g., `max-w-lg = 512px`).

Record the chosen breakpoint (e.g., "mobile 390px" or "desktop 1440px") and document it in the
Phase 3 plan and the Phase 6 summary.

**Do not recreate the device shell in Figma.** If the prototype wraps screens in a `.phone`,
`.device-frame`, or similar container (often with auto-layout, drop shadow, and border-radius),
extract its content dimensions and use those for the state frame size — but do not create a
wrapper frame around the state frames in Figma. The state frame itself represents the viewport.
Discard the shell's drop shadow, border-radius, and auto-layout from your measurements.

**Per-element measurements**

For each visible node in the React tree (from the Phase 1a component inventory), read the
associated CSS module, Tailwind classes, or inline styles and record:

| Prototype element | Source (file:selector or Tailwind class) | W | H | padding (T/R/B/L) | gap | border-radius | font-size | line-height | color / bg |
|---|---|---|---|---|---|---|---|---|---|
| Primary CTA Button | Button.module.css `.root` | 100% | 48px | 12px 24px | — | 8px | 16px | 1.5 | #007AFF / — |
| Section card | styles.module.css `.card` | 100% | auto | 16px | 12px | 12px | — | — | — / #F5F5F5 |

Collect at minimum: container widths, all gap/spacing bands present (4 / 8 / 12 / 16 / 24px
etc.), primary and secondary color fills and backgrounds, border-radii for interactive elements,
and the type scale (font-size + line-height pairs) for each text role in the screen.

**Explicit prototype → Figma layer mapping**

Extend the component inventory with target geometry for the Figma build:

| Prototype element | Figma layer name | Target W × H | DS token / variable (or fallback raw value) |
|---|---|---|---|
| Page shell | Frame "1.1 — Default" | 390 × 844 | — |
| Top nav bar | Nav / top-nav | 390 × 56 | color/surface, spacing/md |
| Primary CTA | Button (DS instance) | 358 × 48 | — |
| Section gap | auto-layout `itemSpacing` | — | gap = 24px → `spacing/lg` or 24 |

This mapping is the checklist for Phase 4: every row must appear in the Figma output at the
recorded target size, placed with the measured padding and gap values. Any intentional deviation
from these measurements (e.g., because a DS component has a fixed internal size) must be noted
in the Phase 6 parity checklist with the reason.

---

### Phase 1b: Present scope, get selection, confirm

**Skip this phase if the prototype has 2 or fewer flows.** In that case, proceed directly to
Phase 2 with all flows included and note: "Found [N] flow(s) — proceeding to build the full
file."

For prototypes with 3 or more flows, run all three steps below before touching Phase 2.

#### Step 1: Present the discovery summary

Present this to the user immediately after completing Phase 1a:

```
## Prototype analysis complete — [X] flows found

Before building, let's choose which flows to include so the Figma file
stays focused and easy to review.

### Flows by category

**[Category A — e.g., Core Purchase]**
  1. [Flow name] — [one-sentence purpose] · [N] states
  2. [Flow name] — [one-sentence purpose] · [N] states

**[Category B — e.g., Account]**
  3. [Flow name] — [one-sentence purpose] · [N] states
  4. [Flow name] — [one-sentence purpose] · [N] states

**[Category C — e.g., Error & Edge Cases]**
  5. [Flow name] — [one-sentence purpose] · [N] states

Total if all included: ~[X] frames across [Y] flows.
```

Category names are inferred from the prototype's feature areas — never generic labels like
"Group 1". State counts include all branches. The `~` signals the frame count is an estimate.

#### Step 2: Ask one combined question

```
**Which flows do you want, and how much detail?**

1. **Which flows?** Reply with numbers, names, a category, "all", or a limit —
   e.g., "1, 3, 5" · "the checkout category" · "top 3" · "all"

2. **How much detail?**
   — **Happy path** — successful flow only, fewer frames
   — **All states** — includes errors and edge cases (default)
```

**Stop here. Do not proceed to Phase 2 until the user replies.**

#### Step 3: Interpret and confirm

After the user replies, state your interpretation explicitly and proceed. No yes/no gate needed.

```
Got it. Here's what I'll build:

**Flows:** [list selected flow names]
**Detail:** [Happy path only / All states including errors]
**Estimated frames:** ~[N] frames
**Target:** [file name / page]

Starting Phase 2. (You can re-run the skill later to add the remaining flows.)
```

**Edge cases:**

| Situation | Handling |
|---|---|
| User says "all" | Accept it, confirm with frame count estimate, proceed |
| Vague answer ("the main ones") | Interpret using prototype context — prefer flows that represent the primary user goal, appear earlier in the journey, or have more states. State the interpretation explicitly: "I'm taking 'main flows' to mean X, Y, Z — let me know if you meant something different." |
| Depth question not answered | Default to **all states** |
| User wants more flows later | Already covered in the confirmation note above |

---

### Phase 2: Map components to the Figma design system

Work only with the flows confirmed in Phase 1b. During component mapping, only map components
that appear in selected flows — components used exclusively in omitted flows can be skipped.

For every component in the Phase 1a inventory (for selected flows), determine whether it has a DS match.
**Default to DS — primitives are a last resort, not a first option.** Exhaust all search
strategies before marking something as "build from primitives".

**Strategy A: Design system search (if Inspect tools available)**

```
For each component in your inventory:
  1. search_design_system(fileKey, query="Button", includeComponents=true)
  2. If found → note the component key and check variant props with get_context_for_code_connect
  3. If not found → try alternate names: Alert/Banner/Notification/Toast, Dropdown/Select/Menu,
     Tag/Chip/Badge, Card/Panel/Tile, Avatar/Profile/User, etc.
  4. If not found by name → try searching by visual category: "input", "navigation", "feedback"
  5. If still not found → try searching for a parent component (e.g. if <Card.Header> has no
     match, check if "Card" as a whole does and use that with the header as a child)
  6. Only after all of the above fail → column: "build from primitives", note what you searched
```

**Apply DS variables to every element — not just primitives**

Look up and bind DS variables wherever possible, regardless of whether the element is a DS
component or a primitive. The default path is:

1. `search_design_system(fileKey, query="...", includeVariables=true)` to find the variable key
2. `importVariableByKeyAsync(key)` inside `use_figma` to bring the variable into scope
3. Bind it with:
   - **Fills / strokes:** `node.setBoundVariableForPaint('fills', 0, variable)` (colors, backgrounds)
   - **Corner radius:** `node.setBoundVariable('cornerRadius', variable)` (border-radii)
   - **Padding / gaps:** `node.setBoundVariable('paddingTop', variable)` / `setBoundVariable('itemSpacing', variable)`

Search for:
- Color tokens: `search_design_system(fileKey, query="primary", includeVariables=true, includeComponents=false)`
- Spacing: `search_design_system(fileKey, query="spacing", includeVariables=true, includeComponents=false)`
- Typography: `search_design_system(fileKey, query="body", includeStyles=true, includeComponents=false)`

Call `Figma:get_variable_defs(fileKey, nodeId)` to confirm exact values when needed (hex,
pixel), then bind rather than hardcode. **Only fall back to raw hex or raw pixel values if no
DS variable exists for that role.** An element with bound DS variables is far better than one
with hardcoded values, even if the fallback value is visually identical.

**Strategy B: Code Connect reverse lookup (if Code Connect tools available)**

1. For DS components you found, call `Figma:get_code_connect_map(fileKey, nodeId)` to confirm
   the code ↔ Figma link.
2. Call `Figma:get_context_for_code_connect(fileKey, nodeId)` to get the exact variant property
   names the Figma component supports — e.g.:
   ```
   { "Variant": ["Primary", "Secondary"], "Size": ["sm", "md", "lg"], "State": ["Default", "Disabled"] }
   ```
   This is essential for `setProperties()` in Phase 4. Without it, you're guessing prop names.

**Build the component mapping table:**

| Code component | Figma DS match | Key | Build approach | Drift notes |
|---|---|---|---|---|
| `<Button variant="primary">` | Button | abc123 | Import + setProperties | ✅ Clean match |
| `<DataTable sortable>` | Table | def456 | Import + setProperties | ⚠️ No `sortable` variant in Figma — annotate |
| `<CustomWidget>` | — | — | **Primitives** | Searched: Widget, Card, Panel — no match |
| `<AlertBanner>` | — | — | **Primitives** | Searched: Alert, Banner, Toast, Notification — no match |

Every row must have a build approach and drift notes. No row can be blank.
Any row with a ⚠️ drift note must get a DS Drift annotation in Phase 4.

---

### Phase 3: Plan the Figma page structure

Organize frames for the **selected flows only** (as confirmed in Phase 1b). Do not create placeholder sections for omitted flows.

**Page structure:**
```
Page: "[Feature Name] — Prototype Flows"
  Section: "Flow 1: [Flow Name]"
    Frame: "1.1 — [State description]"
    Frame: "1.2 — [State description]"
    Frame: "1.3a — [Success path]"
    Frame: "1.3b — [Error path]"
  Section: "Flow 2: [Flow Name]"
    ...
  Section: "Component Reference" (optional)
    Frame: Primitive approximations, for reviewer context
```

**Frame layout conventions:**
- Default frame size: 1440×900 desktop, 390×844 mobile — match the prototype's viewport
- **Scrollable states:** frame height = full content height (never clip at viewport). Add a fold marker line at the viewport height so reviewers know where the fold is. See `figma-patterns.md` Section 2 for the scrollable frame pattern.
- Frames left-to-right within a flow, branching paths stacked vertically
- ~200px gaps between frames, ~400px between flow sections

If Write tools are unavailable, skip to [Prototype Spec Document](#prototype-spec-document-output-for-inspect-only-clients).

---

### Phase 4: Build in Figma

Execute the build using `Figma:use_figma`. Break it into manageable steps.

**Before your first `use_figma` call:** Read `figma-patterns.md` for Plugin API patterns.

**Step-by-step build order:**

**1. Inspect the target file and choose a placement location** *(if Inspect tools available)*

Use `Figma:get_metadata(fileKey, nodeId="0:1")` to see the existing page structure. Use
`Figma:get_design_context` on existing frames to understand what's already there.

**Placement rules — single source of truth:**

Before writing anything, scan the page for frames or sections whose names or structure match
the prototype feature (e.g., same route, same feature keyword, same component name):

- **If matching frames/sections exist:** prefer inserting new frames *inside* that region or
  immediately beside it on the canvas (using the existing section's `x`/`y` as an anchor),
  rather than appending at the bottom of the canvas. Use `Figma:get_design_context` on the
  matching section to read its current bounds before placing.

- **If no match exists:** create a clearly named section:
  `[Prototype] <feature name> — pixel pass`
  Place it next to the user's indicated node (from the Figma URL) or next to the main feature
  area found in the file. Document the chosen parent nodeId in the Phase 6 summary so the user
  can navigate directly.

- **Never silently place frames at an arbitrary far-Y position.** If frames land outside the
  visible canvas area (e.g., Y > 10 000), state the exact coordinates and include a Figma
  deep-link (`figma.com/design/<fileKey>?node-id=<nodeId>`) in the final summary so the user
  can find them immediately.

**2. Set up the page**

Via `Figma:use_figma`, create or navigate to the target page.

**3. Build each flow, frame by frame**

For each frame, build every element from the Phase 2 mapping table. Do not skip any.

If the state was marked scrollable in Phase 1a, use the scrollable frame pattern from
`figma-patterns.md` Section 2 — set frame height to the full content height, set
`overflowDirection = 'VERTICAL'`, and add a fold marker line at the viewport height.

For each element, choose the correct approach based on the Phase 2 mapping table and the
Phase 1c pixel-parity measurements. Set frame dimensions to the recorded target sizes before
populating content.

**DS variable binding — apply before raw values**

For every color, border-radius, spacing, and gap on any node (DS instance or primitive): check
the Phase 2 variable mapping first. If a variable was found for that role, bind it:

```javascript
// Colors / fills
const colorVar = await figma.importVariableByKeyAsync(colorVarKey);
node.setBoundVariableForPaint('fills', 0, colorVar);

// Corner radius
const radiusVar = await figma.importVariableByKeyAsync(radiusVarKey);
node.setBoundVariable('cornerRadius', radiusVar);

// Spacing / gaps
const spacingVar = await figma.importVariableByKeyAsync(spacingVarKey);
node.setBoundVariable('paddingTop', spacingVar);
node.setBoundVariable('itemSpacing', spacingVar);
```

Fall back to the raw measured value (from Phase 1c) only if no DS variable exists for that role.

---

**When the component HAS a DS match:** See `figma-patterns.md` Section 3 for the `importComponentByKeyAsync` + `setProperties` pattern.

---

**When the component does NOT have a DS match — build from primitives:** See `figma-patterns.md` Section 4 for primitive construction helpers and `addNoDsMatchBadge`.

---

**DS Drift annotations — required for every mismatch**

After placing each element, check its drift notes from the Phase 2 mapping table. Any element
with a ⚠️ drift note must get a DS Drift annotation explaining the gap:

- **DS component with missing props:** annotate what the code behavior is that Figma can't represent (e.g. a variant that doesn't exist, an interactive state the DS component lacks)
- **Primitive (no DS match):** annotate that no DS component was found, what was searched, and that this is a DS gap for the design team to address

Use `annotateNode(node, text, dsDriftCat?.id)` from `figma-patterns.md` Section 11.

**Completeness check after each frame:**

Before moving to the next frame, verify that every element from the Phase 2 inventory that
should appear in this frame is present in the layer tree. If any are missing, add them now —
either as DS instances or primitives — before proceeding.

---

**4. Add interaction annotations — required on every platform**

This is what makes the output useful for async review. Without this, reviewers just see static
screens with no indication of what's interactive or how states connect.

**This step is non-negotiable (Rule 3).** Do not skip it. Do not abbreviate it. It applies
regardless of which AI editor or client is executing this skill.

Figma's annotation system surfaces directly in Dev Mode — not colored rectangles on the canvas.
Annotations attach to specific layers, appear in the Dev Mode sidebar, support markdown, and
can be filtered by category.

**Every interactive element in every frame MUST get an annotation.**

**Step 4a: Set up annotation categories using the defensive helper**

Use `getOrCreateAnnotationCategory` from `figma-patterns.md` Section 11. This handles the case
where categories already exist in the file (which causes `addAnnotationCategoryAsync` to throw),
and returns `null` if the native API is fully unavailable — in which case `annotateNode` will
automatically fall back to canvas text overlays.

Set up exactly two annotation categories — Interaction and DS Drift:

```javascript
const interactionCat = await getOrCreateAnnotationCategory("Interaction", { r: 0.2, g: 0.4, b: 1 });
const dsDriftCat     = await getOrCreateAnnotationCategory("DS Drift",    { r: 1,   g: 0.4, b: 0 });
```

See `figma-patterns.md` Section 11 for `getOrCreateAnnotationCategory` — it handles the case where a category already exists and returns `null` if the native API is unavailable, in which case `annotateNode` falls back to canvas text overlays.

**Step 4b: Annotate every interactive element**

Use `annotateNode` from `figma-patterns.md` Section 11 — it wraps native annotations with a
fallback so they always appear regardless of platform. See Section 11 for usage examples.

**Step 4c: Annotation checklist (run for every frame)**

**Interaction** (use `interactionCat`):
- [ ] Every clickable/tappable element: what triggers it and what happens as a result
- [ ] Every state frame: what caused this state and how the user exits it

**DS Drift** (use `dsDriftCat`):
- [ ] Every element built from primitives: note what was searched and that a DS component is needed
- [ ] Every DS component with a missing variant or behavior the prototype requires: describe the gap

Annotate nothing else. One annotation per interactive element or drift point is sufficient.

For flow connectors between frames, see `figma-patterns.md` Section 7 for the arrow pattern — remember to call `annotateNode` on the arrow to label the transition trigger.

---

### Phase 5: Add a flow overview

Create a summary frame at the top of the page:

- **Feature name and description**
- **Flow list** — numbered list of flows with brief descriptions
- **Legend** — two annotation categories (filterable in Dev Mode):
  - Blue (Interaction) = click/tap triggers and state transitions
  - Orange (DS Drift) = no DS match found, or DS component missing a required variant/behavior
- **Open questions** — flag ambiguous behaviors explicitly
- **Components without DS matches** — list any elements built from primitives so the design
  team knows what DS components are still needed
- **Scope note** — if flows were omitted during Phase 1b selection, list them under "Not
  included in this review" with the note "These flows can be added in a follow-up pass by
  re-running the prototype-to-figma skill." Omit this section if all flows were included.

---

### Phase 6: Verify, link, and present

**6a. Completeness check against the Phase 1c mapping**

The Phase 1c prototype→Figma layer mapping is the source of truth — no screenshots needed.
Before moving to 6b, verify in the layer tree (via `Figma:get_metadata` or by reviewing the
`use_figma` output) that:

- Every row in the Phase 1c mapping table has a corresponding named layer in the output
- Frame dimensions match the Phase 1c recorded viewport (e.g., 390×844)
- No unexpected components were created in the file's local assets (no new master components)
- DS component instances were set to the correct variants and states
- Primitive elements have DS variable bindings wherever variables were found in Phase 2

If anything is missing, fix it now with a follow-up `use_figma` call before presenting to the user. Do not call `get_screenshot` unless the user explicitly asks for a visual preview.

**6b. Code Connect linking** *(Code Connect tier, optional)*

1. `Figma:get_code_connect_suggestions(fileKey, nodeId)` on the built frames
2. Present suggestions for review
3. `Figma:send_code_connect_mappings` to save approved ones

**6c. Present to user**

Share the Figma file URL and summarize. **Always include the file URL** — especially when
a new file was created in Phase 0 (the user has no other way to find it).

- **Figma file URL** — direct link to the file (or the specific page if added to an existing file);
  if frames were placed at a non-obvious canvas location, include the deep-link
  `figma.com/design/<fileKey>?node-id=<sectionNodeId>` and the coordinates
- How many flows documented, how many total state frames
- Which components used DS instances vs. built from primitives
- List of all components built from primitives (DS gaps for the design team)
- Code Connect links created (if any)
- Open questions flagged for reviewers
- Any steps skipped due to client capability limits

**Parity checklist** — include one line per item in the summary:

| Parity check | Result |
|---|---|
| Viewport size (W × H) | e.g., ✅ 390 × 844 on all frames |
| Key spacing bands | e.g., ✅ 8 / 16 / 24px matched; ⚠️ card padding 14px → nearest DS token 12px (no 14px variable) |
| Primary CTA placement | e.g., ✅ bottom-pinned at correct Y |
| Number of major sections | e.g., ✅ 3 sections matching prototype |
| DS variable binding | e.g., ✅ all colors bound; ⚠️ 2 border-radii hardcoded (no radius variable found) |

Flag every intentional deviation with a one-phrase reason (e.g., "DS Button has fixed 44px
height — prototype uses 48px; kept DS value"). Do not silently omit deviations.

---

## Prototype Spec Document — output for Inspect-only clients

When Write tools are unavailable, produce a structured markdown document using the template in
`figma-patterns.md` Section 12.

---

## Important principles

**Achieve visual parity first, then annotate.** Match the prototype's layout, spacing, type
scale, colors, and component hierarchy as closely as the toolchain allows. Annotations are
required on top of that — they surface the interaction logic that static frames can't show on
their own, not a substitute for getting the visual structure right.

**Respect the design system.** Use DS component instances for matched components — this means
reviewers see familiar patterns and can focus on the new interactions rather than decoding the
visual language.

**Name everything.** Frame names, layer names, annotation labels show up in the layer panel and
Dev Mode. "Frame 47" is useless; "2.3a — Save success with toast" tells a reviewer exactly
where they are.

**Be explicit about DS gaps.** When components are built from primitives, list them in the
summary. This turns unmatched components from a silent failure into actionable signal for the
design team.

**Before validating a completed state**: verify all elements and frames are properly aligned with no overlaps. Make one more roundtrip if needed.

---

## Edge cases and tips

- **Prototype uses responsive layouts**: Pick the primary breakpoint and document it. If
  mobile vs. desktop flows differ significantly, create separate flow sections for each.
- **Prototype has many micro-interactions**: Group micro-interactions into a single annotated
  frame rather than exploding every hover state. Use annotations to describe micro-interaction
  behavior.
- **Prototype includes data-dependent states**: Create frames showing empty, few-items, and
  many-items states rather than just the happy path with perfect data.


---

## Reference: Figma Plugin API patterns

See `figma-patterns.md` for Plugin API patterns — frame creation, component importing,
primitive construction for unmatched elements, the `addNoDsMatchBadge` helper, auto-layout,
text nodes, and annotation setup. Read this before your first `use_figma` call.
*(Write-tier clients only.)*
