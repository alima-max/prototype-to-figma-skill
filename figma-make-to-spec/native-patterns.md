# Native agent patterns (figma-make-to-spec)

Net-new mechanics the native surface unlocks. For shared build primitives — frame creation,
`importComponentByKeyAsync` + `setProperties`, primitive helpers, node annotations — see
`../figma-patterns.md`. Everything here uses the Figma Plugin API (`figma.*`) available to the
native agent.

> Confirm each capability at runtime (`typeof node.setReactionsAsync`, `figma.variables`,
> branch support) and degrade gracefully — annotations still carry the intent if an API is absent.

---

## 1. Bind text styles and variables (not raw values)

The variables API lives under `figma.variables.*` (NOT the `figma` global), and
`setBoundVariableForPaint` takes a *paint* and returns a new one you must reassign.

```javascript
// Color fill → variable
const colorVar = await figma.variables.importVariableByKeyAsync(colorVarKey);
let fills = node.fills;
if (fills && fills !== figma.mixed && fills.length && fills[0].type === 'SOLID') {
  fills = [...fills];                        // copy — assigning into the live array is a no-op
  fills[0] = figma.variables.setBoundVariableForPaint(fills[0], 'color', colorVar);
  node.fills = fills;                        // reassign — required
}

// Scalar fields (radius / border-width / spacing) → node.setBoundVariable
const radiusVar = await figma.variables.importVariableByKeyAsync(radiusVarKey);
node.setBoundVariable('cornerRadius', radiusVar);   // also 'strokeWeight', 'itemSpacing'; fallback: .id

// Typography → library text style (styles use figma.importStyleByKeyAsync, NOT the variables API)
const textStyle = await figma.importStyleByKeyAsync(textStyleKey);
await textNode.setTextStyleIdAsync(textStyle.id);   // font/size/weight/lineHeight; fill stays separate
```

> ⚠ `figma.importVariableByKeyAsync(...)` (no `.variables`) and
> `node.setBoundVariableForPaint('fills', 0, ...)` THROW — use the forms above.

---

## 2. Wire real prototype interactions

Each state-advancing transition becomes a reaction from the trigger node to the destination frame.
This is what makes the spec *clickable* instead of a static board.

```javascript
// Navigate from a CTA to the next state frame on click
async function wire(triggerNode, destFrame) {
  await triggerNode.setReactionsAsync([
    {
      trigger: { type: 'ON_CLICK' },
      actions: [{
        type: 'NODE',
        destinationId: destFrame.id,
        navigation: 'NAVIGATE',
        transition: null,                 // or { type:'SMART_ANIMATE', easing:{type:'EASE_OUT'}, duration:0.3 }
        preserveScrollPosition: false,
      }],
    },
  ]);
}
```

Branches (success/error) are two reactions from the same origin to two different destination
frames — annotate each with its condition. Set the entry frame of each flow as a starting point so
the flow is playable from the top:

```javascript
figma.currentPage.flowStartingPoints = [
  { nodeId: flow1Entry.id, name: 'Flow 1: Create' },
  { nodeId: flow2Entry.id, name: 'Flow 2: Edit' },
];
```

Pair every reaction with an Interaction annotation on the trigger node (`../figma-patterns.md`):
the reaction is the behavior, the annotation is the explanation.

---

## 3. Round-trip: patch an existing spec in place

When a `[Spec] <feature>` page already exists and the prototype changed, diff instead of rebuild.

```javascript
// Index existing state frames by their "1.2 — …" name prefix
const page = figma.currentPage;
const existing = new Map();
for (const n of page.children) {
  const key = (n.name.match(/^\d+\.\d+[a-z]?/) || [])[0];
  if (key) existing.set(key, n);
}

for (const state of newStateList) {          // from Phase 1
  const match = existing.get(state.key);
  if (!match) {
    buildStateFrame(state);                   // new state → add it
  } else {
    reconcileFrame(match, state);             // changed → update instances/props in place
    existing.delete(state.key);
  }
}
for (const [key, orphan] of existing) {
  orphan.name = `${orphan.name} [removed in prototype — verify]`;  // gone → flag, don't silently delete
}
```

Keep names stable (`1.2 — …`) across runs — the diff keys on them. Prefer updating an instance's
`setProperties` / overrides over deleting and re-adding, so reviewer comments stay attached.

---

## 4. DS-gap proposals (never pollute the live file)

Collect drift notes (no-match primitives, missing variants, unbound values) and stage proposals
away from the spec.

```javascript
const proposals = figma.createPage();
proposals.name = '[DS proposals] <feature>';
```

- **On a proposals page:** represent each gap as a **primitive mockup + a note** describing the
  proposed component/variant and where it's used. Do **not** call `figma.createComponent()` here —
  masters are file-global and would pollute the library's assets (Rule 3).
- **On a branch (if the surface supports it):** a branch is isolated, so real proposed components
  can be built there for the DS team to review and merge. Prefer this when available.

Each proposal note should name the code element, the search terms that found no match (or the
missing variant), and the frames that needed it — so the gap is actionable, not just logged.

---

## 5. Attach annotations — set once, never append via the getter

`node.annotations = [...node.annotations, entry]` **throws on the second annotation**: the getter
returns normalized objects the setter rejects when re-assigned. In a real run this silently dropped
every node's 2nd annotation and (via a fallback) renamed frames. Collect all of a node's entries
and assign the full array **once**:

```javascript
// WRONG — throws on the 2nd call, drops the annotation
// node.annotations = [...(node.annotations||[]), entry];

// RIGHT — build the complete list, assign once
const entries = [];
entries.push({ labelMarkdown: '**Tap →** navigate to Detail', categoryId: interCat?.id });
if (isPrimitive) entries.push({ labelMarkdown: '**DS Gap:** no match — built from primitives', categoryId: gapCat?.id });
try { node.annotations = entries; } catch (e) { node.name += ' [note: ' + entries[0].labelMarkdown.slice(0,60) + ']'; }
```

Native annotations **do** work through `use_figma` (this was wrongly assumed MCP-only) — just set
them in one shot.

---

## 6. Bring Make images and SVGs into the Figma file

`get_design_context` on a Make file lists image resources
(`file://figma/make/image/<makeKey>/<hash>.png`). Pipeline, per image:

1. **Read the bytes** — `ReadMcpResource(server:"figma", uri:<imageUri>)`. Binary is saved to a
   local file and the path is returned. Large reads (multi-MB) can drop the connection — retry once.
2. **Get an upload URL** — `upload_assets(fileKey, count:1, nodeId:<receivingNode>, scaleMode:"FILL")`
   returns a `submitUrl`. Pass `nodeId` to set the image as that node's fill; omit it to create new
   frames.
3. **POST the bytes** — to `submitUrl`, either `multipart/form-data` with a `file` field (the
   filename becomes the layer name) or raw bytes with the right `Content-Type`. Response returns
   `imageHash` + `placedOnNodeId`. Single-use URL, 10-minute expiry, 10MB max per asset.

```bash
curl -sS -X POST -F "file=@/local/path/bg.png;type=image/png;filename=IntroBackground.png" \
  "https://mcp.figma.com/mcp/upload/<id>/submit?scaleMode=FILL"
# → {"success":true,"imageHash":"…","placedOnNodeId":"9:2"}
```

Typical use: create a full-bleed rect (or frame) in `use_figma`, insert it behind content
(`frame.insertChild(0, rect)`), read its id via `get_metadata`, then upload onto that node id.

**SVGs are not supported by `upload_assets`** — bring vector art in through `use_figma` with
`figma.createNodeFromSvg(svgString)` (read the `.svg` resource for the markup).

Never ship placeholder rectangles where a real asset exists — importing them is the difference
between a mockup and parity.

---

## 7. Scope design-system discovery to the file's libraries

Unscoped `search_design_system` searches every library the account can see and buries the file's
own DS in org-wide noise (this is why an enabled library looks "unrecognized"). Always scope:

```javascript
// 1) enumerate the libraries actually subscribed to the file
//    get_libraries(fileKey) → libraries_added_to_file[].libraryKey
const keys = ['lk-…MOSF…', 'lk-…other…'];
// 2) scope every discovery call to those keys
//    search_design_system(fileKey, query:'button', includeLibraryKeys: keys,
//                          includeComponents:true, includeVariables:true, includeStyles:true)
// 3) list_file_components_for_code_connect(libraryFileKey) enumerates published components in bulk
```

Run broad + category queries (`'nav'`, `'button'`, `'card'`, `'audio'`, plus the prototype's own
component names). In a real run, scoping surfaced a `Nav Button` component set that unscoped search
returned nothing for.

---

## 8. `use_figma` returns nothing — verify by reading back

A script's trailing value is not surfaced by the tool. To confirm an effect landed, either read the
state in a follow-up call (`node.reactions.length`, `node.annotations.length`,
`figma.currentPage.flowStartingPoints.length`) or write a short report string into a text node on
the canvas and screenshot it. Don't assume success from "executed with no return value".
