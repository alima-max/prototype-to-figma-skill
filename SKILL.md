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

## When to use this skill

The user has built (or is working on) a prototype in Claude Code and wants to:
- Get async design feedback from cross-functional partners
- Document the interaction flows in a format designers and PMs can comment on
- Bridge the gap between a working prototype and a design review artifact

## Required Figma MCP tools

This skill uses the full suite of Figma MCP tools. Here's when and why each one matters:

### Writing to Figma
- **`Figma:use_figma`** — The primary workhorse. Executes Figma Plugin API JavaScript to create
  pages, frames, import components, build layouts, add annotations, and position everything.
  Every frame, text node, and arrow in the output is created through this tool. Break complex
  builds into multiple calls rather than one giant script.

### Design system discovery
- **`Figma:search_design_system`** — Search the target file's linked libraries for components,
  variables (color tokens, spacing tokens), and styles. Use this to find matching DS components
  for every code component in the prototype. Also search for variables to apply correct design
  tokens (colors, spacing, typography) to any scratch-built elements.
  - Search for components: `search_design_system(fileKey, query="Button", includeComponents=true)`
  - Search for color tokens: `search_design_system(fileKey, query="primary", includeVariables=true)`
  - Search for styles: `search_design_system(fileKey, query="heading", includeStyles=true)`

- **`Figma:get_variable_defs`** — Get variable definitions (design tokens) for a specific node.
  Use this after `search_design_system` finds relevant variables to get their exact values —
  e.g., the hex value of `color/primary/500` or the pixel value of `spacing/md`. Apply these
  values when building scratch components to stay on-system even without a DS component match.

### Code ↔ Design mapping (Code Connect)
These tools are critical because the prototype uses the user's actual component library, which
may already be mapped to Figma components via Code Connect:

- **`Figma:get_code_connect_map`** — Given a node in the Figma file, returns the mapping between
  Figma node IDs and code component locations (e.g., `Button` → `src/components/Button.tsx`).
  Use this to check if DS components already have Code Connect mappings, which tells you the
  exact code component they correspond to.

- **`Figma:get_code_connect_suggestions`** — Get AI-suggested mappings between Figma components
  and code components. Use this when you've built frames with imported DS components and want
  to verify or create the code↔design link. Call with `excludeMappingPrompt=true` for a
  lightweight list of unmapped components.

- **`Figma:get_context_for_code_connect`** — Get structured metadata for a Figma component:
  property definitions, variant options, and descendant tree. Use this to understand how a DS
  component is structured before importing it — what variants exist, what properties can be
  set, and how to configure instances to match specific prototype states.

- **`Figma:add_code_connect_map`** — Map a single Figma node to a code component. Use after
  building frames if the user wants to establish or update Code Connect links between the Figma
  output and the prototype's code components.

- **`Figma:send_code_connect_mappings`** — Save multiple Code Connect mappings in bulk. More
  efficient than `add_code_connect_map` when you need to link many components at once. Use
  after `get_code_connect_suggestions` to confirm and save approved mappings.

### Inspecting existing Figma content
- **`Figma:get_design_context`** — The primary inspection tool. Returns reference code, a
  screenshot, and contextual metadata for a node. Use this to understand the existing structure
  of the target Figma file before adding content — what pages exist, what's already there, what
  DS components look like when instantiated. Prefer this over `get_metadata`.

- **`Figma:get_metadata`** — Returns an XML overview of the file/page structure: node IDs, layer
  types, names, positions, sizes. Use this for a quick structural overview when you need to
  find specific pages or frames by name before writing. Always prefer `get_design_context` for
  detailed inspection.

### Screenshots and verification
- **`Figma:get_screenshot`** — Generate a screenshot of any node. Use this in the verification
  phase to confirm the output looks correct before presenting to the user.

### File management
- **`Figma:whoami`** — Get the authenticated user's info and plan keys. Required before
  `create_new_file` to get the `planKey`.
- **`Figma:create_new_file`** — Create a new blank Figma design file. Only use if the user
  wants a fresh file rather than adding to an existing one.

### Design system documentation (optional)
- **`Figma:create_design_system_rules`** — Generates a prompt for creating design system rules
  for the repo. Useful as a follow-up step if the user wants to formalize the design system
  conventions discovered during the prototype→Figma process (e.g., documenting which code
  components map to which Figma components, variant naming conventions, token usage patterns).

---

## The workflow

### Phase 1: Understand the prototype

Before touching Figma, thoroughly analyze the prototype source code. You're looking for:

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

### Phase 2: Map components to the Figma design system

This phase bridges the code world and the Figma world. You have two complementary strategies —
use both.

**Strategy A: Design system search (component name → Figma component)**

Search the target file's linked libraries for each code component. Use
`Figma:search_design_system` with the target file key:

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

Also search for design tokens to use on scratch-built elements:
- Color tokens: `search_design_system(fileKey, query="primary", includeVariables=true, includeComponents=false)`
- Spacing tokens: `search_design_system(fileKey, query="spacing", includeVariables=true, includeComponents=false)`

When you find relevant variables, call `Figma:get_variable_defs(fileKey, nodeId)` to get their
exact values. This lets you apply the correct design tokens (hex colors, pixel values) to any
elements you build from scratch, keeping them visually consistent with the design system.

**Strategy B: Code Connect reverse lookup (code path → Figma component)**

Since the prototype uses the user's actual component library, Code Connect mappings may already
link code components to Figma components. This is the more precise route when it's available.

1. If you know a Figma component's node ID (from search_design_system results), call
   `Figma:get_code_connect_map(fileKey, nodeId)` to see if it maps to a code component path.
   If the `codeConnectSrc` matches a component import in the prototype, that's a confirmed
   exact match.

2. For components you found in the DS, call `Figma:get_context_for_code_connect(fileKey, nodeId)`
   to get detailed property/variant metadata. This tells you exactly what props the Figma
   component supports, which is essential for correctly configuring instances to match specific
   prototype states (e.g., `variant="destructive"`, `size="sm"`, `disabled={true}`).

3. After building, optionally call `Figma:get_code_connect_suggestions(fileKey, nodeId)` on the
   built frames to check for unmapped components. Then use
   `Figma:send_code_connect_mappings` to bulk-save any new code↔design links the user
   approves.

**Build a component mapping table:**

| Code component | Figma DS match | Key | Code Connect | Variants available |
|---|---|---|---|---|
| `<Button variant="primary">` | Button | abc123 | ✅ `src/ui/Button.tsx` | Primary, Secondary, Destructive × sm, md, lg |
| `<DataTable>` | Table | def456 | ✅ `src/ui/DataTable.tsx` | Default, Compact |
| `<CustomWidget>` | — | — | — | Build from scratch |

### Phase 3: Plan the Figma page structure

Organize frames to tell the story of each flow. The structure should make it easy for a
reviewer to follow a flow from start to finish without jumping around.

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

### Phase 4: Build in Figma

Execute the build using `Figma:use_figma`. This is the most complex phase — break it into
manageable steps. Don't try to build the entire file in a single Plugin API call.

**Before your first `use_figma` call:** Read `references/figma-patterns.md` for Plugin API
patterns (font loading, auto-layout, text nodes, positioning). This avoids common gotchas.

**Step-by-step build order:**

1. **Inspect the target file** — Use `Figma:get_metadata(fileKey, nodeId="0:1")` to see the
   existing page structure. If the user specified a target page, find its ID. Use
   `Figma:get_design_context` on any existing frames to understand what's already there and
   avoid overwriting content.

2. **Set up the page** — Via `Figma:use_figma`, create (or navigate to) the target page. Set
   up the top-level layout structure.

3. **Import design system components** — For each component in your mapping table that has a
   Figma DS match, import it using `importComponentByKeyAsync(key)` inside `use_figma`. Store
   references for reuse across frames.

   To correctly configure instances for specific states, use the variant/property information
   from Phase 2's `get_context_for_code_connect` call. This tells you the exact property names
   and values the Figma component supports — for example, if a Button has
   `{ "Variant": ["Primary", "Secondary"], "Size": ["sm", "md", "lg"], "State": ["Default", "Hover", "Disabled"] }`,
   you can set the instance to match the prototype's `<Button variant="primary" size="sm" disabled>` as:
   ```javascript
   instance.setProperties({ "Variant": "Primary", "Size": "sm", "State": "Disabled" });
   ```

4. **Build each flow, frame by frame:**
   - Create a section frame for the flow
   - For each state in the flow:
     a. Create a frame at the correct size
     b. Compose the UI using imported component instances (`.createInstance()`)
     c. Configure instance props/variants to match the prototype state (see step 3)
     d. For components without DS matches, build the layout from primitives
        (rectangles, text, auto-layout frames) — apply DS variables/tokens from
        `search_design_system` where possible for consistent colors and spacing
     e. Name layers clearly — reviewers may inspect the layer tree

4. **Add interaction annotations** — This is what makes the output useful for async review.
   For each state transition, add annotations that explain:
   - **Trigger**: What the user does (e.g., "Clicks 'Save' button")
   - **Result**: What happens (e.g., "Shows loading spinner for 1-2s, then transitions to
     success state")
   - **Conditions**: Any branching logic (e.g., "If validation fails, show error state 1.3b
     instead")
   - **Edge cases**: Anything non-obvious (e.g., "Debounced — won't fire if clicked within
     300ms of last click")

**How to create annotations in Figma:**

Use native Figma annotation-style patterns. Create clearly labeled annotation nodes next to or
overlaying the relevant elements:

```javascript
// Annotation pattern — a colored callout connected to the relevant element
const annotation = figma.createFrame();
annotation.name = "Annotation: [trigger description]";
annotation.resize(320, 100);
annotation.fills = [{ type: 'SOLID', color: { r: 1, g: 0.95, b: 0.8 } }]; // light yellow
annotation.cornerRadius = 8;
annotation.layoutMode = 'VERTICAL';
annotation.paddingTop = 12;
annotation.paddingBottom = 12;
annotation.paddingLeft = 16;
annotation.paddingRight = 16;
annotation.itemSpacing = 4;

// Annotation label
const label = figma.createText();
await figma.loadFontAsync({ family: "Inter", style: "Semi Bold" });
label.fontName = { family: "Inter", style: "Semi Bold" };
label.characters = "On click → Save";
label.fontSize = 12;
label.fills = [{ type: 'SOLID', color: { r: 0.6, g: 0.4, b: 0 } }];
annotation.appendChild(label);

// Annotation body
const body = figma.createText();
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
body.fontName = { family: "Inter", style: "Regular" };
body.characters = "Validates form fields. If valid, shows loading state (Frame 1.3)...";
body.fontSize = 11;
body.fills = [{ type: 'SOLID', color: { r: 0.4, g: 0.3, b: 0 } }];
body.textAutoResize = 'WIDTH_AND_HEIGHT';
annotation.appendChild(body);
```

For flow connectors between frames, use lines or arrows:

```javascript
// Arrow connecting two frames to show flow direction
const arrow = figma.createLine();
arrow.name = "Flow: 1.1 → 1.2";
arrow.resize(200, 0);
// Position between frames
arrow.x = frame1.x + frame1.width + 10;
arrow.y = frame1.y + frame1.height / 2;
arrow.strokes = [{ type: 'SOLID', color: { r: 0.2, g: 0.4, b: 1 } }];
arrow.strokeWeight = 2;
```

### Phase 5: Add a flow overview

At the top of the page (or as the first section), create a summary frame that acts as a
table of contents:

- **Feature name and description** — what the prototype demonstrates
- **Flow list** — numbered list of flows with brief descriptions
- **Legend** — explain the annotation color coding if you used multiple colors:
  - Yellow callouts = interaction triggers
  - Blue arrows = flow direction / transitions
  - Red callouts = error/edge-case paths
  - Green callouts = success paths
- **Open questions** — if the prototype has ambiguous behaviors or things that need design
  input, call them out here explicitly. This is gold for async review.

### Phase 6: Verify, link, and present

**6a. Visual verification**
Take screenshots of the completed output to verify before presenting:
- Use `Figma:get_screenshot(fileKey, nodeId)` on the overview frame and 2-3 representative
  state frames
- Check that frames are properly sized and spaced
- Verify annotations are readable and correctly positioned
- Confirm component instances look correct (right variants, right states)

**6b. Code Connect linking (optional but recommended)**
If the user wants to maintain the code↔design link going forward (so developers can jump
from Figma components to the prototype's source code), offer to set up Code Connect mappings:

1. Call `Figma:get_code_connect_suggestions(fileKey, nodeId)` on the built frames to get
   AI-suggested mappings between the Figma output and the prototype's codebase.
2. Present the suggestions to the user for review.
3. Use `Figma:send_code_connect_mappings(fileKey, mappings, nodeId)` to bulk-save the approved
   mappings. Use `label="React"` (or the appropriate framework) and set `source` to the
   component's path in the prototype codebase.

This step is especially valuable when the prototype introduces new components that don't yet
have Code Connect mappings in the main design system.

**6c. Present to user**
Share the Figma file URL with the user and summarize what was created:
- How many flows were documented
- How many total state frames
- Which components came from the design system vs. built from scratch
- Any new Code Connect mappings created
- Any ambiguities or open questions flagged for reviewers

---

## Important principles

**Optimize for reviewer comprehension, not designer precision.** The point is async feedback
from PMs, engineers, and other stakeholders — not a production-ready design file. Clear labels,
obvious flow direction, and thorough annotations matter more than pixel-perfect spacing.

**Annotate generously.** The prototype contains implicit knowledge — state transitions, edge
cases, timing, conditional logic — that isn't visible in a static frame. Every annotation you
add saves a Slack thread. When in doubt, annotate.

**Respect the design system.** Using the team's actual Figma components (vs. drawing rectangles
that look like buttons) means reviewers see familiar patterns and can focus on the new
interactions rather than decoding the visual language.

**Name everything.** Frame names, layer names, annotation labels — these show up in Figma's
layer panel and comments. "Frame 47" is useless; "2.3a — Save success with toast" tells a
reviewer exactly where they are.

**Handle missing components gracefully.** When a prototype component has no design system match,
build a reasonable facsimile from primitives but flag it visually (e.g., a dashed border or a
small "No DS component" badge) so reviewers and designers know this needs design system work.

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

See `references/figma-patterns.md` for common Figma Plugin API patterns used in this skill,
including auto-layout configuration, text styling, component instance creation, and annotation
positioning. Read this reference before your first `use_figma` call if you're unsure about
any API pattern.
