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
