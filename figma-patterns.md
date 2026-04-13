# Figma Plugin API Patterns Reference

Common patterns for building prototype-to-Figma output. Read this before your first
`use_figma` call in a session.

> **Write-tier clients only.** These patterns require `use_figma` (Figma Plugin API access).
> If your client only supports Inspect tools, refer to the Prototype Spec Document section in
> `SKILL.md` instead.

## Table of Contents
1. Creating viewport-sized frames
2. Placing screenshots as image nodes
3. Transparent hotspot overlays (for annotating regions within a screenshot)
4. Native Figma Dev Mode annotations
5. Flow arrows and connectors
6. Section containers
7. State badge overlays
8. Positioning and spacing
9. Annotation category reference
10. Font loading (for labels and badges)

---

## 1. Creating viewport-sized frames

Every state in the prototype gets one frame. The frame holds the screenshot and any hotspot
overlays on top of it.

```javascript
// Desktop frame (1440×900)
const frame = figma.createFrame();
frame.name = "1.1 — Dashboard default";
frame.resize(1440, 900);
frame.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }]; // white background
frame.clipsContent = true;

// Mobile frame (390×844, iPhone 14 Pro)
const mobileFrame = figma.createFrame();
mobileFrame.name = "1.1 — Dashboard default (mobile)";
mobileFrame.resize(390, 844);
mobileFrame.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
```

Match the frame dimensions to the prototype's viewport size.

---

## 2. Placing screenshots as image nodes

This is the core operation. Each frame gets one full-frame screenshot as its base layer.
Do NOT build the UI from Figma components — place the actual rendered screenshot.

### From a URL (hosted image or data URL)

```javascript
const image = await figma.createImageAsync("https://example.com/screenshot.png");

const imageNode = figma.createRectangle();
imageNode.name = "Screenshot";
imageNode.resize(frame.width, frame.height);
imageNode.x = 0;
imageNode.y = 0;
imageNode.fills = [{
  type: 'IMAGE',
  scaleMode: 'FILL',
  imageHash: image.hash
}];
frame.appendChild(imageNode);
```

### From raw bytes (base64-decoded Uint8Array)

```javascript
// imageBytes is a Uint8Array of the PNG/JPEG binary data
const image = figma.createImage(imageBytes);

const imageNode = figma.createRectangle();
imageNode.name = "Screenshot";
imageNode.resize(frame.width, frame.height);
imageNode.x = 0;
imageNode.y = 0;
imageNode.fills = [{
  type: 'IMAGE',
  scaleMode: 'FILL',
  imageHash: image.hash
}];
frame.appendChild(imageNode);
```

### Scale modes

- `'FILL'` — fills the frame, may crop edges. Use when screenshot exactly matches frame size.
- `'FIT'` — fits entirely within frame with letterboxing. Use when aspect ratio differs.
- `'CROP'` — you control the transform. Avoid unless you need fine control.

For pixel-perfect output, always capture the screenshot at the exact frame dimensions so
`'FILL'` produces no distortion.

---

## 3. Transparent hotspot overlays

Hotspots are transparent frames placed over interactive regions within the screenshot. They
have no visible fill or stroke — they're purely annotation targets. Reviewers can click them
in Dev Mode to see interaction annotations for that specific element.

```javascript
function createHotspot(name, x, y, width, height, parentFrame) {
  const hotspot = figma.createFrame();
  hotspot.name = `Hotspot: ${name}`;
  hotspot.resize(width, height);
  hotspot.x = x;
  hotspot.y = y;
  hotspot.fills = []; // fully transparent
  hotspot.strokeWeight = 0;
  hotspot.clipsContent = false;
  parentFrame.appendChild(hotspot);
  return hotspot;
}

// Example: mark the "Save" button region
const saveHotspot = createHotspot("Save button", 892, 520, 120, 40, frame);

// Then annotate it
saveHotspot.annotations = [
  {
    labelMarkdown: '**On click →** Validates form.\n\n' +
      '- Valid: saving state (→ 1.3), then `POST /api/items`\n' +
      '- Invalid: inline errors shown (→ 1.3b)\n\n' +
      '*Debounced: 300ms*',
    categoryId: interactionCat.id
  }
];
```

**Finding hotspot coordinates:**
- Read the prototype source to find component layout positions (CSS, inline styles)
- Use approximate positions based on the layout — exact pixel-perfect placement matters
  less than the annotation content
- If unsure, ask the user to identify the element's approximate location

**Hotspot sizing:**
- Match the visible tap/click target of the element
- For buttons: the full button bounding box
- For form fields: the full input area
- For navigation items: the full clickable row
- Slightly larger than the visual element is fine — it won't affect the screenshot

---

## 4. Native Figma Dev Mode annotations

Real Figma annotations appear in Dev Mode, support markdown, can be filtered by category, and
stay attached to their node. Apply these to both frames (state-level) and hotspots (element-level).

### Setting up annotation categories (once per session)

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

If categories were already created in this file, retrieve them instead:
```javascript
const existing = await figma.annotations.getAnnotationCategoriesAsync();
const interactionCat = existing.find(c => c.label === 'Interaction');
// etc.
```

Available colors: `'yellow'`, `'orange'`, `'red'`, `'pink'`, `'violet'`, `'blue'`, `'teal'`,
`'green'`.

### Annotating nodes

```javascript
// Simple annotation
node.annotations = [{ label: 'Opens settings modal on click' }];

// Rich markdown with category
node.annotations = [
  {
    labelMarkdown: '**On click →** Submits form via `POST /api/items`\n\n' +
      '- Success: closes modal, refreshes list\n' +
      '- Error: shows inline error banner\n\n' +
      '*Debounced: 300ms*',
    categoryId: interactionCat.id
  }
];

// Multiple annotations on the same node
node.annotations = [
  {
    labelMarkdown: '**Validation:** Required, min 3 chars, max 100 chars',
    categoryId: validationCat.id
  },
  {
    labelMarkdown: '**Keyboard:** Tab to next field, Enter submits',
    categoryId: a11yCat.id
  }
];

// With pinned design properties
node.annotations = [
  {
    label: 'Responsive: 600px max on desktop, full-width on mobile',
    properties: [{ type: 'width' }, { type: 'maxWidth' }]
  }
];
```

### Supported pinnable property types

`'width'`, `'height'`, `'maxWidth'`, `'minWidth'`, `'maxHeight'`, `'minHeight'`,
`'fills'`, `'strokes'`, `'effects'`, `'strokeWeight'`, `'cornerRadius'`,
`'textStyleId'`, `'textAlignHorizontal'`, `'fontFamily'`, `'fontStyle'`, `'fontSize'`,
`'fontWeight'`, `'lineHeight'`, `'letterSpacing'`, `'itemSpacing'`, `'padding'`,
`'layoutMode'`, `'alignItems'`, `'opacity'`, `'mainComponent'`

---

## 5. Flow arrows and connectors

Visual connectors showing the path between states. These sit between frames on the canvas,
not inside a frame.

```javascript
function createFlowArrow(fromFrame, toFrame, canvasParent, transitionLabel, catId) {
  const arrow = figma.createLine();
  arrow.name = `Flow: ${fromFrame.name} → ${toFrame.name}`;
  arrow.resize(toFrame.x - (fromFrame.x + fromFrame.width) - 20, 0);
  arrow.x = fromFrame.x + fromFrame.width + 10;
  arrow.y = fromFrame.y + fromFrame.height / 2;
  arrow.strokes = [{ type: 'SOLID', color: { r: 0.25, g: 0.45, b: 0.95 } }];
  arrow.strokeWeight = 2;
  arrow.strokeCap = 'ARROW_EQUILATERAL';

  if (transitionLabel) {
    arrow.annotations = [
      { labelMarkdown: transitionLabel, categoryId: catId }
    ];
  }

  // Arrows live on the page, not inside frames
  figma.currentPage.appendChild(arrow);
  return arrow;
}

// Example
createFlowArrow(frame12, frame13, figma.currentPage,
  '**Transition:** On "Save" click with valid form', navigationCat.id);
```

For branching flows (one state → two outcomes), create two arrows with angled paths
and labels explaining the branch condition.

---

## 6. Section containers

Group related flow frames in a Figma section:

```javascript
const section = figma.createSection();
section.name = "Flow 1: Create New Item";
// Sections auto-resize to fit children
section.appendChild(frame11);
section.appendChild(frame12);
section.appendChild(frame13a);
section.appendChild(frame13b);
```

If sections cause issues, use a large transparent frame instead:

```javascript
const sectionFrame = figma.createFrame();
sectionFrame.name = "Flow 1: Create New Item";
sectionFrame.resize(8000, 1200);
sectionFrame.fills = [];
sectionFrame.clipsContent = false;
```

---

## 7. State badge overlays

Small labeled badges placed over a frame to indicate its state type at a glance:

```javascript
async function createStateBadge(text, color, parentFrame, x = 8, y = 8) {
  const badge = figma.createFrame();
  badge.name = `Badge: ${text}`;
  badge.layoutMode = 'HORIZONTAL';
  badge.primaryAxisSizingMode = 'AUTO';
  badge.counterAxisSizingMode = 'AUTO';
  badge.paddingTop = 3;
  badge.paddingBottom = 3;
  badge.paddingLeft = 8;
  badge.paddingRight = 8;
  badge.cornerRadius = 4;
  badge.fills = [{ type: 'SOLID', color }];

  await figma.loadFontAsync({ family: "Inter", style: "Semi Bold" });
  const label = figma.createText();
  label.fontName = { family: "Inter", style: "Semi Bold" };
  label.characters = text;
  label.fontSize = 9;
  label.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
  badge.appendChild(label);

  badge.x = x;
  badge.y = y;
  parentFrame.appendChild(badge);
  return badge;
}

// Common badges
createStateBadge("Loading",        { r: 0.25, g: 0.45, b: 0.95 }, frame); // blue
createStateBadge("Error state",    { r: 0.85, g: 0.20, b: 0.20 }, frame); // red
createStateBadge("Empty state",    { r: 0.55, g: 0.36, b: 0.96 }, frame); // purple
createStateBadge("Success",        { r: 0.13, g: 0.69, b: 0.30 }, frame); // green
createStateBadge("Needs screenshot",{ r: 0.90, g: 0.50, b: 0.10 }, frame); // orange
```

Use "Needs screenshot" when you've created the frame but are still waiting for the user to
provide the screenshot for that state.

---

## 8. Positioning and spacing

```javascript
const FRAME_GAP = 200;        // horizontal gap between sequential state frames
const BRANCH_V_GAP = 100;     // vertical gap between branching frames (e.g. 1.3a and 1.3b)
const FLOW_SECTION_GAP = 400; // vertical gap between flow sections

// Layout a sequence of frames left-to-right
function layoutSequence(frames, startX = 100, startY = 100) {
  let x = startX;
  for (const frame of frames) {
    frame.x = x;
    frame.y = startY;
    x += frame.width + FRAME_GAP;
  }
}

// Layout a branching pair (success above, error below)
function layoutBranch(successFrame, errorFrame, afterX, branchY) {
  successFrame.x = afterX;
  successFrame.y = branchY;
  errorFrame.x = afterX;
  errorFrame.y = branchY + successFrame.height + BRANCH_V_GAP;
}
```

---

## 9. Annotation category reference

| Category | Color | Use for |
|---|---|---|
| Interaction | `'blue'` | Click/tap/gesture triggers and their results |
| Navigation | `'violet'` | Page/view transitions, routing, deep links |
| State Change | `'teal'` | State descriptions, transition conditions, lifecycle |
| Validation | `'orange'` | Form validation rules, constraints, input formatting |
| Error Handling | `'red'` | Error states, recovery paths, fallback behavior |
| Edge Case | `'pink'` | Non-obvious behaviors, race conditions, timing quirks |
| Data / API | `'green'` | Data sources, endpoints, caching, loading behavior |
| Accessibility | `'yellow'` | Keyboard nav, screen reader text, ARIA, focus order |

---

## 10. Font loading (for labels and badges)

Only needed if you're creating text nodes for badges or overview labels — not for the
screenshot content itself.

```javascript
// Load all weights you'll use upfront
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
await figma.loadFontAsync({ family: "Inter", style: "Semi Bold" });
await figma.loadFontAsync({ family: "Inter", style: "Bold" });

// Note: "Semi Bold" has a space (not "SemiBold"). Same for "Extra Bold".

const text = figma.createText();
text.fontName = { family: "Inter", style: "Regular" };
text.characters = "Your label here";
text.fontSize = 14;
text.fills = [{ type: 'SOLID', color: { r: 0.13, g: 0.13, b: 0.13 } }];
text.textAutoResize = 'WIDTH_AND_HEIGHT';
```
