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
cross-functional partners can review asynchronously.

**Core principle: static capture, not reconstruction.** Every frame is a pixel-perfect
screenshot of the actual rendered prototype — not a Figma component assembly built from imports.
This means what reviewers see in Figma is exactly what the prototype looks like, with no drift,
no missing details, and no randomly created components. Native Figma annotations are then layered
on top to document the interaction logic that isn't visible in a static frame.

**This skill works across all Figma MCP clients.** The output format adapts to what your client
supports — see [Client Compatibility](#client-compatibility) below.

## When to use this skill

The user has built (or is working on) a prototype and wants to:
- Get async design feedback from cross-functional partners
- Document the interaction flows in a format designers and PMs can comment on
- Bridge the gap between a working prototype and a design review artifact

---

## Client Compatibility

Different Figma MCP clients support different tool subsets. This skill detects what's available
and uses the best workflow for your client.

### Capability tiers

| Tier | Inspect tools | Write tools | Code Connect tools | Output |
|---|---|---|---|---|
| **Full** | ✅ | ✅ | ✅ | Figma file with static frames, native annotations, and Code Connect links |
| **Write** | ✅ or ❌ | ✅ | ❌ | Figma file with static frames and native annotations (Code Connect skipped) |
| **Inspect-only** | ✅ | ❌ | ❌ | Prototype Spec Document (rich markdown for async review) |
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

## Figma MCP tools used

### Inspect tier
- **`Figma:get_metadata`** — XML overview of file/page structure. Use to check what pages already
  exist before writing.
- **`Figma:get_screenshot`** — Screenshot any Figma node. Use in verification.

### Write tier
- **`Figma:use_figma`** — Executes Figma Plugin API JavaScript to create frames, place images,
  and add native annotations. This is the only tool used to build in Figma — no component
  imports, no DS searches, no reconstruction.
- **`Figma:generate_figma_design`** — Captures a live web page or prototype URL directly into
  Figma. Use this as the primary screenshot method when the prototype is running at an accessible
  URL. Produces an exact visual capture of the rendered page.
- **`Figma:whoami`** — Required before `create_new_file` to get the plan key.
- **`Figma:create_new_file`** — Create a new blank Figma file if needed.

### Code Connect tier (optional)
- **`Figma:get_code_connect_suggestions`** — AI-suggested mappings between Figma nodes and code.
- **`Figma:send_code_connect_mappings`** — Save Code Connect mappings in bulk.

---

## The workflow

### Phase 0: Detect available capabilities

Before anything else, determine which tier you're operating in and tell the user.

1. **Check for Write tools** — See if `use_figma` and `generate_figma_design` appear in your
   available tool list.
2. **Check for Inspect tools** — See if `get_metadata` is available.
3. **Check for Code Connect tools** — See if `get_code_connect_suggestions` is available.
4. **Announce the tier** and what output you'll produce. Example:
   > "I have Write tools available. I'll capture each prototype state as a static screenshot
   > and place it in Figma, then add native annotations for interaction details."

   Or for read-only:
   > "I only have Inspect tools on this client — I can't write to Figma. I'll produce a
   > Prototype Spec Document instead, suitable for sharing or handing off."

**Route:**
- Write tools available → Phases 1–6
- Write tools unavailable → Phases 1–3, then [Prototype Spec Document](#prototype-spec-document-output-for-inspect-only-clients)

---

### Phase 1: Understand the prototype

Analyze the prototype source code before touching Figma. This phase is the same at every tier.

**1. Identify every distinct UI state**

A "state" is any meaningfully different visual of the screen — not just pages, but also:
- Loading / skeleton states
- Empty states
- Error states
- Form validation states (pristine, filled-valid, filled-invalid, submitting)
- Success confirmations
- Modal / overlay open vs. closed
- Expanded / collapsed sections

For each state, note:
- **What triggers it** (user action, API response, timer, etc.)
- **What's different visually** from adjacent states
- **What the user can do from here** (branching paths)

Map states as a flow graph:
```
Flow: "Create new item"
  1. Dashboard (default) → user clicks "+ New"
  2. Modal — empty form
  3. Modal — filled, valid
  4. Modal — saving (loading)
  5a. Dashboard + success toast ← save succeeded
  5b. Modal + error banner ← validation or server error
```

**2. Note viewport and layout**
- Target viewport size (desktop 1440px wide, mobile 390px, tablet 768px, etc.)
- Whether the prototype is responsive or fixed-width

**3. Identify the prototype URL or entry point**
- Is it running locally? At what port/path?
- Is each state reachable via a direct URL or query param, or does it require user interaction?
- Are there credentials or seed data needed to reach certain states?

---

### Phase 2: Plan the Figma page structure

Decide how to organize the frames before capturing anything.

```
Page: "[Feature Name] — Prototype Flows"
  Section: "Flow 1: [Flow Name]"
    Frame: "1.1 — [State name]"
    Frame: "1.2 — [State name]"
    Frame: "1.3a — [Success]"
    Frame: "1.3b — [Error]"
  Section: "Flow 2: [Flow Name]"
    ...
```

**Naming convention:** `[flow number].[step]([branch]) — [plain-language state description]`
Examples: `1.1 — Dashboard default`, `1.3a — Save success with toast`, `2.2b — Permission denied`

**Layout:**
- Frames arranged left-to-right within a flow
- Branching paths stacked vertically
- ~200px horizontal gap between sequential frames
- ~400px vertical gap between flow sections

If Write tools are unavailable, stop here and jump to [Prototype Spec Document](#prototype-spec-document-output-for-inspect-only-clients).

---

### Phase 3: Capture each prototype state

This is the core phase. **Every frame in Figma must be a pixel-perfect screenshot of the
actual rendered prototype — never a reconstruction built from Figma components.**

Work through each state you identified in Phase 1, one at a time.

#### Method A: `generate_figma_design` with prototype URL *(preferred)*

If the prototype is running at an accessible URL (localhost, staging, or deployed):

```
generate_figma_design(url="http://localhost:3000/dashboard", ...)
```

Use this for states that are directly URL-addressable. Check if the prototype supports
URL-based state (e.g., `?modal=open`, `/items/new`, route params).

For states that require user interaction to reach (e.g., a form validation error, a loading
state mid-save), coordinate with the user:
- Ask them to navigate the prototype to that specific state
- Then call `generate_figma_design` while that state is visible

After capturing, `generate_figma_design` will create a Figma node. Move that node into the
correctly named frame in your page layout.

#### Method B: User-provided screenshots *(universal fallback)*

When `generate_figma_design` is unavailable or a state can't be URL-addressed:

1. Tell the user exactly what state you need a screenshot of:
   > "I need a screenshot of the modal in its error state — after submitting with an invalid
   > email. Could you trigger that state in the prototype and take a screenshot?"
2. Accept the screenshot as a file or URL.
3. Import it into Figma using `use_figma`:

```javascript
// Import a screenshot as an image node in a Figma frame
const frame = figma.createFrame();
frame.name = "1.3b — Modal: validation error";
frame.resize(1440, 900); // match prototype viewport

// From a URL:
const image = await figma.createImageAsync("https://...");
// Or from base64 bytes:
const image = figma.createImage(new Uint8Array(imageBytes));

const imageNode = figma.createRectangle();
imageNode.name = "Screenshot";
imageNode.resize(1440, 900);
imageNode.fills = [{ type: 'IMAGE', scaleMode: 'FILL', imageHash: image.hash }];
frame.appendChild(imageNode);
```

#### What NOT to do

- **Do not** call `importComponentByKeyAsync` to import Figma components
- **Do not** call `search_design_system` and try to match code components to DS components
- **Do not** build layouts using rectangles, text nodes, and auto-layout to recreate the UI
- **Do not** use `get_context_for_code_connect` to configure variant properties on instances

These approaches are the source of random components, missing details, and visual drift. The
prototype is the source of truth — capture it, don't reconstruct it.

---

### Phase 4: Add interaction annotations

Once all frames have their screenshots placed, annotate every interactive element. This is what
makes the Figma output useful for async review — the screenshots show what things look like,
the annotations show how they behave.

Use Figma's native annotation API (not colored rectangles on the canvas). Native annotations
appear in Dev Mode, support markdown, can be filtered by category, and stay linked to their
node.

**Step 4a: Set up annotation categories**

Do this once, at the start of your `use_figma` session:

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

Available colors: `'yellow'`, `'orange'`, `'red'`, `'pink'`, `'violet'`, `'blue'`, `'teal'`,
`'green'`.

**Step 4b: Annotate the frame and its key regions**

Since each frame contains a single screenshot image node, annotate at two levels:

1. **The frame itself** — describe the state: what triggered it, what the user can do from
   here, any time limits or loading states:

```javascript
const frame = figma.currentPage.findOne(n => n.name === "1.3 — Modal: saving");
frame.annotations = [
  {
    labelMarkdown: '**State: Saving**\n\nTriggered after "Save" click + successful validation.\n\n' +
      '- Save button shows spinner, is disabled\n' +
      '- All form fields disabled during save\n' +
      '- **Timeout:** 10s → error toast, form re-enabled (→ Frame 1.3b)\n' +
      '- **Success:** modal closes, dashboard refreshes (→ Frame 1.4a)',
    categoryId: stateCat.id
  }
];
```

2. **Transparent hotspot overlays** — for individual interactive elements, create transparent
   frames positioned over the element's region in the screenshot. Add annotations to those
   overlays so reviewers can click a specific button region and see its behavior:

```javascript
// Position a transparent hotspot over the "Save" button region in the screenshot
const saveHotspot = figma.createFrame();
saveHotspot.name = "Hotspot: Save button";
saveHotspot.resize(120, 40); // match the button's visual size
saveHotspot.x = 892; // match the button's x position in the screenshot
saveHotspot.y = 520; // match the button's y position
saveHotspot.fills = []; // fully transparent
saveHotspot.strokeWeight = 0;
frame.appendChild(saveHotspot);

saveHotspot.annotations = [
  {
    labelMarkdown: '**On click →** Validates all fields.\n\n' +
      '- Valid: shows saving state (→ Frame 1.3), then `POST /api/items`\n' +
      '- Invalid: highlights error fields, shows inline errors (→ Frame 1.3b)\n\n' +
      '*Debounced: 300ms*',
    categoryId: interactionCat.id
  }
];
```

To find the approximate position of an element in the screenshot, either:
- Read the prototype source to find the component's layout position
- Ask the user to point out the element's approximate location
- Use reasonable estimates based on the layout — exact pixel-perfect hotspot placement
  matters less than the annotation content

**Step 4c: Annotation checklist (run for every frame)**

- [ ] **Frame-level state**: What triggered this state? How does the user leave it?
- [ ] **Clickable elements**: What happens on click? Where does the user go? (use hotspots)
- [ ] **Form fields**: Validation rules, required/optional, character limits (use hotspots)
- [ ] **Loading behaviors**: Duration, timeout handling, what's disabled during load
- [ ] **Error states**: What errors are possible? How shown? Recovery paths?
- [ ] **Empty states**: What triggers this? What action moves the user forward?
- [ ] **Data dependencies**: API endpoints, what if data is missing/slow/stale?
- [ ] **Keyboard/a11y**: Tab order, keyboard shortcuts, screen reader considerations

Annotation density should be HIGH — a frame with 5 interactive areas should have at minimum
5 annotations (1 frame-level + 4 hotspots). A complex form might have 10–15.

**Step 4d: Flow arrows between frames**

Add visual connectors showing the path between states:

```javascript
const arrow = figma.createLine();
arrow.name = "Flow: 1.2 → 1.3";
arrow.resize(200, 0);
arrow.x = frame1.x + frame1.width + 10;
arrow.y = frame1.y + frame1.height / 2;
arrow.strokes = [{ type: 'SOLID', color: { r: 0.2, g: 0.4, b: 1 } }];
arrow.strokeWeight = 2;
arrow.strokeCap = 'ARROW_EQUILATERAL';
arrow.annotations = [
  {
    labelMarkdown: '**Transition:** On "Save" click with valid form',
    categoryId: navigationCat.id
  }
];
```

---

### Phase 5: Add a flow overview frame

Create a summary frame at the top of the page:

- **Feature name and description** — what the prototype demonstrates
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

---

### Phase 6: Verify, link, and present

**6a. Visual verification** *(Inspect tools available)*

```
get_screenshot(fileKey, nodeId)
```

Take screenshots of the overview frame and 2–3 representative state frames. Verify:
- Screenshots filled the frames correctly (no stretching, no missing content)
- Annotations are present and legible
- Flow arrows connect the right frames

*If Inspect unavailable, skip and note it to the user.*

**6b. Code Connect linking** *(Code Connect tier, optional)*

If the user wants to link Figma frames back to prototype source files:

1. `get_code_connect_suggestions(fileKey, nodeId)` on the built frames
2. Present suggestions for review
3. `send_code_connect_mappings` to save approved ones

**6c. Present to user**

Summarize:
- Flows documented and total state frames
- Screenshot capture method used (generate_figma_design vs. user-provided)
- Any states that couldn't be auto-captured and need follow-up
- Code Connect links created (if any)
- Open questions flagged for reviewers
- Any steps skipped due to client capability limits

---

## Prototype Spec Document — output for Inspect-only clients

When Write tools are unavailable, produce this structured markdown document directly in the
chat (or offer to write to a file). Content should be detailed enough that a designer with
Figma access could build the frames from it, or a PM could use it for async review.

```markdown
# [Feature Name] — Prototype Spec

**Prototype:** [file path or URL]
**Target Figma file:** [URL if provided]
**Date:** [today]

---

## Overview

[2–3 sentence description of what the prototype demonstrates]

**Open questions for reviewers:**
- [Ambiguous behaviors, design decisions needing input]

---

## Flows

### Flow 1: [Name]

**Goal:** [What the user is trying to accomplish]
**Entry point:** [What triggers this flow]

#### 1.1 — [State name]

**Screenshot needed:** [Describe exactly what state to capture — e.g., "Dashboard with
empty list, logged in as a standard user, no items created yet"]

**Layout:** [Describe the page structure — nav, sidebar, main content, overlays]
**Viewport:** [1440×900 desktop / 390×844 mobile]

**Interactions:**

| Element | Location | Trigger | Result | Notes |
|---|---|---|---|---|
| "+ New" button | Top-right of content area | Click | Opens creation modal (→ 1.2) | Disabled if user lacks create permission |
| Search field | Content area header | Type | Filters list in real time | 300ms debounce |

**Annotations:**
- **State:** [What caused this state, how the user got here]
- **Interaction:** [Click targets and their results]
- **Data / API:** [Data source, endpoint, what triggers a load]
- **Edge Cases:** [Non-obvious behavior, timing, conditions]
- **Accessibility:** [Tab order, keyboard shortcuts]

#### 1.2 — [Next state name]

**Screenshot needed:** [Exact state description]
**Differences from 1.1:** [Only what changed]
**Interactions:** [Only new or changed interactions]
**Annotations:** [As above]

---

## Figma build instructions

*For whoever implements this in Figma — use screenshots, not component reconstruction.*

**Page:** "[Feature Name] — Prototype Flows"
**Frame size:** [1440×900 desktop / 390×844 mobile]
**Layout:** Left-to-right per flow, branches stacked vertically, 200px gaps

**To capture each frame:**
1. Navigate the prototype to the state described under "Screenshot needed"
2. Take a full-page screenshot at the exact viewport size
3. Place the screenshot as an image fill in a Figma frame of the correct dimensions
4. Name the frame exactly as shown (e.g., "1.3b — Modal: validation error")
5. Add native annotations using the categories: Interaction (blue), Navigation (violet),
   State Change (teal), Validation (orange), Error Handling (red), Edge Case (pink),
   Data / API (green), Accessibility (yellow)
6. For each interactive element, add a transparent hotspot frame over it with an annotation
```

---

## Important principles

**Screenshots, not reconstruction.** The prototype is the source of truth. Capture it; don't
rebuild it. This is the single most important rule — it's the difference between a Figma file
that matches the prototype and one that has mysterious extra components and missing details.

**Annotate generously.** The screenshots show what things look like. The annotations show how
they behave. A frame without annotations is just a picture — the interaction logic is the whole
point. Every interactive element should have an annotation. When in doubt, annotate.

**Name everything.** Frame names and annotation labels show up in the layer panel and Dev Mode.
"Frame 47" is useless; "2.3a — Save success with toast" tells a reviewer exactly where they are.

**Be explicit about what was skipped.** If you couldn't auto-capture a state, couldn't reach
it via URL, or skipped Code Connect because your client doesn't support it — say so. The user
should know what needs follow-up, not assume it was done.

---

## Edge cases and tips

- **States that require interaction to reach** (e.g., a form mid-validation): Coordinate with
  the user. Ask them to put the prototype in that state, then capture it.
- **Responsive prototypes**: Capture at the primary breakpoint. If mobile and desktop flows
  differ significantly, create separate sections for each.
- **Micro-interactions**: Don't create separate frames for every hover state. Group
  micro-interaction behavior into annotations on the relevant state frame.
- **Data-dependent states**: Capture empty, partial, and full data states rather than just
  the happy path with perfect data.
- **Large prototypes (5+ flows)**: Ask the user which flows are highest priority and start
  there. You can always add more flows in follow-up passes.

---

## Reference: Figma Plugin API patterns

See `figma-patterns.md` for common Plugin API patterns — frame creation, image placement,
transparent hotspot overlays, flow arrows, annotation setup. Read this before your first
`use_figma` call. *(Write-tier clients only.)*
