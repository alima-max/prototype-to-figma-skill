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

### What "Inspect" tools means for this skill

Inspect tools (`get_design_context`, `get_metadata`, `get_screenshot`, `search_design_system`,
`get_variable_defs`) let the skill:
- Look up matching design system components for code components
- Verify the target Figma file structure before writing
- Take screenshots to verify the output
- Search for design tokens to apply to scratch-built elements

When inspect tools are unavailable, DS component matching is skipped and components are noted
as "unverified" in the spec output.

### What "Write" tools means for this skill

Write tools (`use_figma`, `create_new_file`, `whoami`) let the skill actually build frames,
import components, and add native annotations directly in Figma. Without these, the skill
produces a **Prototype Spec Document** — a detailed markdown artifact with the same information,
suitable for async review in Notion/Confluence, sharing via Slack, or handing off to someone
with Figma write access.

### What "Code Connect" tools means for this skill

Code Connect tools (`get_code_connect_map`, `get_code_connect_suggestions`,
`get_context_for_code_connect`, `add_code_connect_map`, `send_code_connect_mappings`) let the
skill verify exact code↔design component mappings and create new Code Connect links from the
Figma output back to the prototype source. These are optional — the skill produces a complete
Figma file without them.

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

### Phase 0: Detect available capabilities

Before anything else, determine which tier you're operating in and announce it to the user.

**How to detect:**

1. **Check for Inspect tools** — Attempt `Figma:get_metadata` on the target file (or any
   file). If it succeeds, Inspect tools are available.

2. **Check for Write tools** — Inspect the list of available tools for `use_figma`. If it
   appears in your tool list, Write tools are available. You can also attempt a no-op
   `use_figma` call; if it errors with "tool not found" rather than a runtime error, Write is
   unavailable.

3. **Check for Code Connect tools** — Check your tool list for `get_code_connect_map`. If
   present, Code Connect tools are available.

4. **Announce the tier** — Tell the user what you detected and what output you'll produce.
   For example:
   > "I have Inspect + Write tools but no Code Connect. I'll build the full Figma file with
   > native annotations. Code Connect linking will be skipped — you can do that separately if
   > needed."

   Or, for a read-only client:
   > "I only have Inspect tools on this client — I can't write to Figma directly. I'll produce
   > a Prototype Spec Document instead, which you can share for async review or hand off to
   > someone with Figma write access."

**Then route to the appropriate path:**

- **Write tools available** → Continue to Phases 1–6 (full Figma build)
- **Write tools unavailable** → Continue to Phases 1–3, then jump to [Prototype Spec Document](#prototype-spec-document-output-for-inspect-only-clients)

---

### Phase 1: Understand the prototype

Before touching Figma, thoroughly analyze the prototype source code. This phase is the same
regardless of which client tier you're on.

**1. Component inventory**
Read the prototype files and list every UI component used. These typically come from the user's
shared component library. For each component, note:
- The component name as used in code (e.g., `<Button variant="primary">`, `<Modal>`, `<DataTable>`)
- Props/variants being used (size, state, variant)
- Any component compositions (e.g., a Card containing a Header, Body, and ActionBar)

**2. Interaction flows**
This is the most important part. Identify every distinct user flow in the prototype:
- What triggers a state change? (click, hover, form submission, data load, etc.)
- What are the before/after states?
- Are there intermediate states? (loading, validating, animating)
- What are the error/edge-case states? (empty, error, permission denied, timeout)
- Are there branching paths? (success vs. failure, different user roles)

Map these as a flow graph in your analysis. For example:
```
Flow: "Create new item"
  1. Dashboard (default) → user clicks "+ New" button
  2. Creation modal (empty form) → user fills fields
  3. Creation modal (filled, valid) → user clicks "Save"
  4. Creation modal (saving/loading) → API responds
  5a. Dashboard (with new item, success toast) ← success
  5b. Creation modal (with error banner) ← validation error
```

**3. Layout and structure**
Note the overall page structure: navigation, sidebars, content areas, overlays.
Identify what stays constant across states vs. what changes.

---

### Phase 2: Map components to the Figma design system

This phase bridges the code world and the Figma world. Adapt based on available tools.

**If Inspect tools are available**, use both strategies below. **If not**, skip to the
fallback at the end of this phase.

**Strategy A: Design system search (component name → Figma component)**

Search the target file's linked libraries for each code component:

```
For each component in your inventory:
  1. Search the design system: search_design_system(fileKey, query="Button")
  2. If found → note the component key for later import via importComponentByKeyAsync
  3. If found but wrong variant → note the closest match and plan to adjust
  4. If not found → flag it for scratch creation
```

Search broadly — the design system might name things differently than the code. If the code
uses `<AlertBanner>`, the Figma library might call it "Alert", "Banner", "Notification", or
"Toast". Try multiple search terms before concluding it's not there.

Also search for design tokens:
- Color tokens: `search_design_system(fileKey, query="primary", includeVariables=true, includeComponents=false)`
- Spacing tokens: `search_design_system(fileKey, query="spacing", includeVariables=true, includeComponents=false)`

When you find relevant variables, call `Figma:get_variable_defs(fileKey, nodeId)` to get their
exact values for use on scratch-built elements.

**Strategy B: Code Connect reverse lookup (Code Connect tier only)**

If Code Connect tools are available:

1. Call `Figma:get_code_connect_map(fileKey, nodeId)` to check DS component ↔ code path mappings.
2. Call `Figma:get_context_for_code_connect(fileKey, nodeId)` to get property/variant metadata —
   this tells you exactly what props the Figma component supports, so you can configure
   instances to match prototype states:
   ```javascript
   instance.setProperties({ "Variant": "Primary", "Size": "sm", "State": "Disabled" });
   ```
3. After building, call `Figma:get_code_connect_suggestions(fileKey, nodeId)` on built frames
   and `Figma:send_code_connect_mappings` to save approved mappings.

**If Inspect tools are NOT available (fallback)**

Build the component mapping from code alone. For each component, record:
- The code name (e.g., `<Button>`)
- The likely Figma equivalent (infer from common naming conventions)
- Mark all entries as "unverified — DS match not confirmed"

**Build a component mapping table** (regardless of tier):

| Code component | Figma DS match | Key | Code Connect | Status |
|---|---|---|---|---|
| `<Button variant="primary">` | Button | abc123 | ✅ `src/ui/Button.tsx` | Verified |
| `<DataTable>` | Table | def456 | ✅ `src/ui/DataTable.tsx` | Verified |
| `<CustomWidget>` | — | — | — | Scratch needed |
| `<AlertBanner>` | Alert | — | ❓ | Unverified (no Inspect) |

---

### Phase 3: Plan the Figma page structure

Organize frames to tell the story of each flow. This phase is the same regardless of tier —
you're planning structure, not building yet.

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
    Frame: Components built from scratch, for reviewer context
```

**Frame layout conventions:**
- Use a consistent frame size (typically 1440×900 for desktop, 390×844 for mobile — match the
  prototype's target viewport)
- Arrange frames left-to-right within a flow, with branching paths stacked vertically
- Leave ~200px gaps between frames for connector annotations
- Use a section frame (large container) per flow to group related state frames

**If Write tools are unavailable**, skip to [Prototype Spec Document](#prototype-spec-document-output-for-inspect-only-clients).

---

### Phase 4: Build in Figma *(Write tier required)*

Execute the build using `Figma:use_figma`. Break it into manageable steps — don't try to
build the entire file in a single Plugin API call.

**Before your first `use_figma` call:** Read `figma-patterns.md` for Plugin API patterns
(font loading, auto-layout, text nodes, positioning). This avoids common gotchas.

**Step-by-step build order:**

1. **Inspect the target file** *(if Inspect tools available)* — Use
   `Figma:get_metadata(fileKey, nodeId="0:1")` to see the existing page structure. Use
   `Figma:get_design_context` on any existing frames to understand what's already there and
   avoid overwriting content. *If Inspect is unavailable, skip this step and proceed.*

2. **Set up the page** — Via `Figma:use_figma`, create (or navigate to) the target page. Set
   up the top-level layout structure.

3. **Import design system components** *(if Inspect tools found DS matches)* — For each
   component with a confirmed Figma DS key, import it using `importComponentByKeyAsync(key)`
   inside `use_figma`. Store references for reuse across frames.

   Configure instances using variant metadata from Phase 2:
   ```javascript
   instance.setProperties({ "Variant": "Primary", "Size": "sm", "State": "Disabled" });
   ```

   *If Inspect was unavailable*, build all components from primitives (rectangles, text,
   auto-layout frames) and add a "No DS match — Inspect unavailable" badge to each.

4. **Build each flow, frame by frame:**
   - Create a section frame for the flow
   - For each state in the flow:
     a. Create a frame at the correct size
     b. Compose the UI using imported component instances (`.createInstance()`)
     c. Configure instance props/variants to match the prototype state
     d. For components without DS matches, build from primitives — apply DS tokens if available
     e. Name layers clearly — reviewers inspect the layer tree

5. **Add interaction annotations using Figma's native annotation API** — This is the most
   important step. Without this, reviewers just see static screens.

   Figma has a first-class annotation system that surfaces in Dev Mode. These are NOT colored
   rectangles on the canvas — they are native Figma annotations that attach to specific layers,
   appear in the Dev Mode sidebar, support markdown, and can be filtered by category.

   **Every interactive element in every frame MUST get an annotation.**

   **Step 5a: Set up annotation categories**

   ```javascript
   const interactionCat = await figma.annotations.addAnnotationCategoryAsync({
     label: 'Interaction', color: 'blue'
   });
   const navigationCat = await figma.annotations.addAnnotationCategoryAsync({
     label: 'Navigation', color: 'violet'
   });
   const stateCat = await figma.annotations.addAnnotationCategoryAsync({
     label: 'State Change', color: 'teal'
   });
   const validationCat = await figma.annotations.addAnnotationCategoryAsync({
     label: 'Validation', color: 'orange'
   });
   const errorCat = await figma.annotations.addAnnotationCategoryAsync({
     label: 'Error Handling', color: 'red'
   });
   const edgeCaseCat = await figma.annotations.addAnnotationCategoryAsync({
     label: 'Edge Case', color: 'pink'
   });
   const dataCat = await figma.annotations.addAnnotationCategoryAsync({
     label: 'Data / API', color: 'green'
   });
   const a11yCat = await figma.annotations.addAnnotationCategoryAsync({
     label: 'Accessibility', color: 'yellow'
   });
   ```

   Available category colors: `'yellow'`, `'orange'`, `'red'`, `'pink'`, `'violet'`, `'blue'`,
   `'teal'`, `'green'`.

   **Step 5b: Annotate every interactive element**

   ```javascript
   // Example: Annotating a "Save" button
   saveButton.annotations = [
     {
       labelMarkdown: '**On click →** Validates all form fields.\n\n' +
         '- If valid: shows loading spinner (→ Frame 1.3), saves via `POST /api/items`\n' +
         '- If invalid: highlights error fields, shows inline errors (→ Frame 1.4b)\n\n' +
         '*Debounced: 300ms cooldown after last click*',
       categoryId: interactionCat.id
     }
   ];

   // Example: Annotating a form field
   emailField.annotations = [
     {
       labelMarkdown: '**Validation rules:**\n- Required\n- Must match email regex\n' +
         '- Uniqueness check via `GET /api/users/check-email`\n- Error shown on blur',
       categoryId: validationCat.id
     },
     {
       labelMarkdown: '**Keyboard:** Tab to next field. Enter submits form.',
       categoryId: a11yCat.id
     }
   ];

   // Example: Annotating a loading state frame
   loadingFrame.annotations = [
     {
       labelMarkdown: '**State: Loading**\n\nTriggered after successful validation. ' +
         'Spinner replaces button text. All fields disabled.\n\n' +
         '**Timeout:** 10s → shows error toast, re-enables form (→ Frame 1.4b)',
       categoryId: stateCat.id
     }
   ];
   ```

   You can also pin design properties alongside annotations:
   ```javascript
   someNode.annotations = [
     {
       label: 'Max-width: 600px on desktop, full-width on mobile',
       properties: [{ type: 'width' }, { type: 'maxWidth' }]
     }
   ];
   ```

   **Step 5c: Annotation checklist (run for every frame)**

   - [ ] **Clickable elements**: What happens on click? Where does the user go?
   - [ ] **Form fields**: Validation rules, required/optional, input masks, character limits
   - [ ] **State triggers**: What caused this state? How does the user leave it?
   - [ ] **Loading behaviors**: Duration, timeout handling, what's disabled during load
   - [ ] **Error states**: What errors are possible? Recovery paths?
   - [ ] **Empty states**: When does this appear? What action moves the user forward?
   - [ ] **Conditional visibility**: What determines if this element shows/hides?
   - [ ] **Data dependencies**: What API calls? What if data is missing/slow/stale?
   - [ ] **Keyboard/a11y**: Tab order, keyboard shortcuts, screen reader considerations
   - [ ] **Animations/transitions**: Duration, easing, what triggers them

   The annotation density should be HIGH — a frame with 5 interactive elements should have
   at minimum 5 annotations.

For flow connectors between frames (visual aids, separate from native annotations):

```javascript
const arrow = figma.createLine();
arrow.name = "Flow: 1.1 → 1.2";
arrow.resize(200, 0);
arrow.x = frame1.x + frame1.width + 10;
arrow.y = frame1.y + frame1.height / 2;
arrow.strokes = [{ type: 'SOLID', color: { r: 0.2, g: 0.4, b: 1 } }];
arrow.strokeWeight = 2;
arrow.strokeCap = 'ARROW_EQUILATERAL';
arrow.annotations = [
  { labelMarkdown: '**Transition:** On successful save (HTTP 200)', categoryId: navigationCat.id }
];
```

---

### Phase 5: Add a flow overview *(Write tier required)*

Create a summary frame at the top of the page:

- **Feature name and description**
- **Flow list** — numbered list of flows with brief descriptions
- **Legend** — annotation categories and what reviewers should look for:
  - Blue (Interaction) = click/tap/gesture triggers and results
  - Violet (Navigation) = page/view transitions and routing
  - Teal (State Change) = state descriptions and transition conditions
  - Orange (Validation) = form validation rules and constraints
  - Red (Error Handling) = error states, recovery paths, fallbacks
  - Pink (Edge Case) = non-obvious behaviors, race conditions, timing
  - Green (Data / API) = data sources, endpoints, loading/caching
  - Yellow (Accessibility) = keyboard behavior, screen reader, a11y
  - Note: Reviewers can **filter by category** in Dev Mode
- **Open questions** — flag any ambiguous behaviors for reviewers

---

### Phase 6: Verify, link, and present

**6a. Visual verification** *(Inspect tier required)*

If Inspect tools are available, take screenshots to verify before presenting:
- Use `Figma:get_screenshot(fileKey, nodeId)` on the overview frame and 2-3 state frames
- Check frame sizing, spacing, and annotation legibility
- Verify component instances look correct

*If Inspect tools are unavailable, skip this step and note it to the user.*

**6b. Code Connect linking** *(Code Connect tier only, optional)*

If Code Connect tools are available and the user wants to link the Figma output back to the
prototype's source code:

1. Call `Figma:get_code_connect_suggestions(fileKey, nodeId)` on the built frames.
2. Present suggestions to the user for review.
3. Use `Figma:send_code_connect_mappings(fileKey, mappings, nodeId)` to save approved mappings.
   Use `label="React"` (or appropriate framework) and set `source` to the component path.

*If Code Connect tools are unavailable, skip this step. The Figma file is complete without it.*

**6c. Present to user**

Share the Figma file URL and summarize:
- How many flows were documented
- How many total state frames
- Which components came from the design system vs. built from scratch
- Whether Code Connect mappings were created
- Any open questions flagged for reviewers
- Any steps skipped due to client capability limits (be explicit about what was skipped and why)

---

## Prototype Spec Document — output for Inspect-only clients

When Write tools are unavailable, produce a structured markdown document with the following
sections. The goal is the same as the Figma output: make implicit prototype knowledge explicit
so cross-functional partners can review asynchronously.

Output the spec document directly in the chat (or offer to write it to a file). The content
should be detailed enough that a designer with Figma access could build the frames from it, or
that a PM could use it for async feedback without needing to run the prototype.

```markdown
# [Feature Name] — Prototype Spec

**Generated from:** [prototype file path or description]
**Target Figma file:** [URL if provided, or "not specified"]
**Date:** [today]

---

## Overview

[2-3 sentence description of what the prototype demonstrates and why it exists]

**Open questions for reviewers:**
- [List any ambiguous behaviors, design decisions, or things that need input]

---

## Flows

### Flow 1: [Flow Name]

**Goal:** [What the user is trying to accomplish]
**Entry point:** [What triggers this flow — a button click, page load, etc.]

#### States

**1.1 — [State name]**
- **Frame size:** 1440×900 (desktop) / 390×844 (mobile)
- **Layout:** [Describe the overall layout — nav, sidebar, content area, overlays]
- **Key components:**
  - [Component name] ([code name: `<Button variant="primary">`]) → [Figma DS match if known]
  - [Component name] → [Figma DS match if known, or "No DS match confirmed"]

- **Interactions:**
  | Element | Trigger | Result | Notes |
  |---|---|---|---|
  | Save button | Click | Validates form → loading state (→ 1.2) | Debounced 300ms |
  | Cancel button | Click | Closes modal, returns to dashboard | No confirmation |
  | Email field | Blur | Validates email format | Shows inline error if invalid |

- **State annotations:**
  - **Interaction:** [Describe click/tap triggers and results]
  - **Validation:** [List form validation rules]
  - **Data / API:** [List API calls, endpoints, data sources]
  - **Error Handling:** [What errors can occur? How are they shown? Recovery paths?]
  - **Edge Cases:** [Non-obvious behaviors, timing, race conditions]
  - **Accessibility:** [Tab order, keyboard shortcuts, screen reader considerations]

**1.2 — [Next state name]**
- **Triggered by:** [What caused this state]
- **Differences from 1.1:** [What changed — this avoids re-documenting the whole layout]
- **Key components:** [Only components that changed]
- **Interactions:** [Only new or changed interactions]
- **Annotations:** [As above]

**1.3a — [Success path]**
[same structure]

**1.3b — [Error/failure path]**
[same structure]

---

### Flow 2: [Flow Name]
[same structure]

---

## Component Inventory

| Code component | Props used | Likely Figma DS match | Confidence | Notes |
|---|---|---|---|---|
| `<Button>` | variant="primary", size="md" | Button / Primary | High | Common component |
| `<DataTable>` | columns, rows, onSort | Table | Medium | May be named differently |
| `<CustomWidget>` | (custom) | None — build from scratch | — | Not in DS |

*Note: DS matches are inferred from code — not confirmed via search_design_system (Inspect
tools not available on this client). Verify in Figma before building.*

---

## Figma build guide

*For whoever implements this in Figma:*

**Page structure:**
```
Page: "[Feature Name] — Prototype Flows"
  Section: "Flow 1: [Name]" — frames 1.1 through 1.3b, left-to-right, branches vertical
  Section: "Flow 2: [Name]" — ...
```

**Frame sizes:** [Desktop: 1440×900 / Mobile: 390×844]
**Gap between frames:** 200px
**Gap between flow sections:** 400px

**Annotation categories to create:**
- Interaction (blue), Navigation (violet), State Change (teal), Validation (orange),
  Error Handling (red), Edge Case (pink), Data / API (green), Accessibility (yellow)

**Components to import from DS:** [list from component inventory above]
**Components to build from scratch:** [list from component inventory above]
```

**Tips for producing a good Prototype Spec Document:**
- Be as specific as possible about interaction behavior — this replaces the annotations that
  would go into a Figma file
- List every interactive element in every state, not just the primary ones
- Include API endpoint names, response shapes, and timing details when you know them
- Flag open questions explicitly — these are the most valuable part for async review
- Use the same state-numbering scheme (1.1, 1.2, 1.3a, 1.3b) that the Figma build guide
  describes, so the document and any future Figma file stay in sync

---

## Important principles

**Optimize for reviewer comprehension, not designer precision.** The point is async feedback
from PMs, engineers, and other stakeholders — not a production-ready design file. Clear labels,
obvious flow direction, and thorough annotations matter more than pixel-perfect spacing.

**Annotate generously — this is the core deliverable.** The prototype contains implicit
knowledge — state transitions, edge cases, timing, conditional logic — that isn't visible in a
static frame. Every native Figma annotation you add (or every annotation entry in the spec doc)
saves a Slack thread. When in doubt, annotate.

**Respect the design system.** Using actual Figma DS components (vs. drawing rectangles that
look like buttons) means reviewers see familiar patterns and can focus on the new interactions
rather than decoding the visual language.

**Name everything.** Frame names, layer names, annotation labels — these show up in Figma's
layer panel and comments. "Frame 47" is useless; "2.3a — Save success with toast" tells a
reviewer exactly where they are.

**Handle missing components gracefully.** When a prototype component has no DS match, build
from primitives but flag it visually (dashed border or "No DS component" badge) so reviewers
and designers know this needs design system work.

**Be explicit about tier limitations.** If you skipped DS search, visual verification, or Code
Connect because your client doesn't support those tools, say so in your summary. The user
should know what was skipped and why, not assume it was done.

---

## Edge cases and tips

- **Prototype uses responsive layouts**: Pick the primary breakpoint and document it. If
  mobile vs. desktop flows differ significantly, create separate flow sections for each.
- **Prototype has many micro-interactions**: Group related micro-interactions into a single
  annotated frame rather than exploding every hover state into its own frame. Use annotations
  to describe the micro-interaction behavior.
- **Prototype includes data-dependent states**: Create frames showing representative data
  states (empty, few items, many items, long text) rather than just the "happy path" with
  perfect data.
- **Large prototypes**: For prototypes with 5+ distinct flows, prioritize. Ask the user which
  flows are most important for review and start with those. You can always add more flows in
  follow-up passes.

---

## Reference: Figma Plugin API patterns

See `figma-patterns.md` for common Figma Plugin API patterns used in this skill, including
auto-layout configuration, text styling, component instance creation, and native annotation
setup. Read this reference before your first `use_figma` call if you're unsure about any API
pattern. *(Only needed for Write-tier clients.)*
