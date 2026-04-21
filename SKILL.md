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
cross-functional partners can review asynchronously. The goal is **not** pixel-perfect mockups —
it's making the interaction logic, flows, and design decisions legible to reviewers who can't
easily walk through a live prototype.

**This skill works across all Figma MCP clients.** The output format adapts to what your client
supports — see [Client Compatibility](#client-compatibility) below.

## When to use this skill

The user has built (or is working on) a prototype in Claude Code and wants to:
- Get async design feedback from cross-functional partners
- Document the interaction flows in a format designers and PMs can comment on
- Bridge the gap between a working prototype and a design review artifact

---

## Three non-negotiable rules

These rules exist to prevent the bugs most commonly reported by users of this skill:

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
- One annotation per form field (validation rules)
- One annotation per state frame (what caused this state, how the user leaves it)

The annotation density should be HIGH. When in doubt, annotate more. Always use `node.annotations = [...]` on actual interactive elements — not colored rectangles on the canvas.

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
- **`Figma:get_screenshot`** — Generate a screenshot of any node. Use in verification phase.
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
Every component must end up in one of two columns: **matched** or **build from primitives**.
Neither column can be empty just because you couldn't find something.

**Strategy A: Design system search (if Inspect tools available)**

```
For each component in your inventory:
  1. search_design_system(fileKey, query="Button", includeComponents=true)
  2. If found → note the component key for importComponentByKeyAsync
  3. If not found under that name → try alternate names (Alert/Banner/Notification/Toast, etc.)
  4. If still not found → column: "build from primitives"
```

Also search for design tokens to use on primitive elements:
- Color tokens: `search_design_system(fileKey, query="primary", includeVariables=true, includeComponents=false)`
- Spacing tokens: `search_design_system(fileKey, query="spacing", includeVariables=true, includeComponents=false)`

When you find relevant variables, call `Figma:get_variable_defs(fileKey, nodeId)` to get their
exact hex/pixel values. Apply these to primitive elements to keep them visually on-system.

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

| Code component | Figma DS match | Key | Code Connect | Build approach |
|---|---|---|---|---|
| `<Button variant="primary">` | Button | abc123 | ✅ `src/ui/Button.tsx` | Import + setProperties |
| `<DataTable>` | Table | def456 | ✅ `src/ui/DataTable.tsx` | Import + setProperties |
| `<CustomWidget>` | — | — | — | **Primitives** |
| `<AlertBanner>` | — | — | — | **Primitives** |

Every row must have a build approach. No row can be blank.

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

**1. Inspect the target file** *(if Inspect tools available)*

Use `Figma:get_metadata(fileKey, nodeId="0:1")` to see the existing page structure. Use
`Figma:get_design_context` on existing frames to understand what's already there.

**2. Set up the page**

Via `Figma:use_figma`, create or navigate to the target page.

**3. Build each flow, frame by frame**

For each frame, build every element from the Phase 2 mapping table. Do not skip any.

If the state was marked scrollable in Phase 1a, use the scrollable frame pattern from
`figma-patterns.md` Section 2 — set frame height to the full content height, set
`overflowDirection = 'VERTICAL'`, and add a fold marker line at the viewport height.

For each element, choose the correct approach based on the mapping table:

---

**When the component HAS a DS match:** See `figma-patterns.md` Section 3 for the `importComponentByKeyAsync` + `setProperties` pattern.

---

**When the component does NOT have a DS match — build from primitives:** See `figma-patterns.md` Section 4 for primitive construction helpers and `addNoDsMatchBadge`.

---

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

See `figma-patterns.md` Section 11 for the full category setup — all eight categories with names and colors.

**Step 4b: Annotate every interactive element**

Use `annotateNode` from `figma-patterns.md` Section 11 — it wraps native annotations with a
fallback so they always appear regardless of platform. See Section 11 for usage examples.

**Step 4c: Annotation checklist (run for every frame)**

- [ ] **Clickable elements**: What happens on click? Where does the user go?
- [ ] **Form fields**: Validation rules, required/optional, input masks, character limits
- [ ] **State triggers**: What caused this state? How does the user leave it?
- [ ] **Loading behaviors**: Duration, timeout handling, what's disabled during load
- [ ] **Error states**: What errors are possible? How shown? Recovery paths?
- [ ] **Empty states**: When does this appear? What action moves the user forward?
- [ ] **Conditional visibility**: What determines if this element shows/hides?
- [ ] **Data dependencies**: What API calls? What if data is missing/slow/stale?
- [ ] **Keyboard/a11y**: Tab order, keyboard shortcuts, screen reader considerations
- [ ] **Animations/transitions**: Duration, easing, what triggers them

The annotation density should be HIGH — a frame with 5 interactive elements should have
at minimum 5 annotations.

For flow connectors between frames, see `figma-patterns.md` Section 7 for the arrow pattern — remember to call `annotateNode` on the arrow to label the transition trigger.

---

### Phase 5: Add a flow overview

Create a summary frame at the top of the page:

- **Feature name and description**
- **Flow list** — numbered list of flows with brief descriptions
- **Legend** — annotation categories:
  - Blue (Interaction) = click/tap triggers and results
  - Violet (Navigation) = transitions and routing
  - Teal (State Change) = state descriptions and conditions
  - Orange (Validation) = form rules and constraints
  - Red (Error Handling) = errors, recovery, fallbacks
  - Pink (Edge Case) = timing quirks, race conditions
  - Green (Data / API) = endpoints, loading, caching
  - Yellow (Accessibility) = keyboard, screen reader, a11y
  - Note: Reviewers can **filter by category** in Dev Mode
- **Open questions** — flag ambiguous behaviors explicitly
- **Components without DS matches** — list any elements built from primitives so the design
  team knows what DS components are still needed
- **Scope note** — if flows were omitted during Phase 1b selection, list them under "Not
  included in this review" with the note "These flows can be added in a follow-up pass by
  re-running the prototype-to-figma skill." Omit this section if all flows were included.

---

### Phase 6: Verify, link, and present

**6a. Visual verification** *(Inspect tools available)*

Take screenshots of the output to verify:
- Use `Figma:get_screenshot(fileKey, nodeId)` on the overview frame and 2–3 state frames
- Verify all elements from the component inventory are present (completeness check)
- Verify no unexpected components were created in the file's local assets
- Check that DS component instances look correct (right variants, right states)
- Check that primitive approximations are visually reasonable

*If Inspect tools are unavailable, skip and note it to the user.*

**6b. Code Connect linking** *(Code Connect tier, optional)*

1. `Figma:get_code_connect_suggestions(fileKey, nodeId)` on the built frames
2. Present suggestions for review
3. `Figma:send_code_connect_mappings` to save approved ones

**6c. Present to user**

Share the Figma file URL and summarize. **Always include the file URL** — especially when
a new file was created in Phase 0 (the user has no other way to find it).

- **Figma file URL** — direct link to the file (or the specific page if added to an existing file)
- How many flows documented, how many total state frames
- Which components used DS instances vs. built from primitives
- List of all components built from primitives (DS gaps for the design team)
- Code Connect links created (if any)
- Open questions flagged for reviewers
- Any steps skipped due to client capability limits

---

## Prototype Spec Document — output for Inspect-only clients

When Write tools are unavailable, produce a structured markdown document using the template in
`figma-patterns.md` Section 12.

---

## Important principles

**Optimize for reviewer comprehension, not designer precision.** Clear labels, obvious flow
direction, and thorough annotations matter more than pixel-perfect spacing.

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
