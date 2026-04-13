# Figma Plugin API Patterns Reference

Common patterns for building prototype-to-Figma output. Read this before your first
`use_figma` call in a session.

## Table of Contents
1. Font loading
2. Creating frames with auto-layout
3. Importing and using design system components
4. Text nodes
5. Annotation callouts
6. Flow arrows and connectors
7. Section containers
8. Badge/tag overlays
9. Positioning and spacing
10. Color palette for annotations

---

## 1. Font loading

Always load fonts before setting text. Inter is the standard Figma font.

```javascript
// Load all weights you'll need upfront
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
await figma.loadFontAsync({ family: "Inter", style: "Semi Bold" });
await figma.loadFontAsync({ family: "Inter", style: "Bold" });
```

Note: "Semi Bold" has a space (not "SemiBold"). Same for "Extra Bold".

---

## 2. Creating frames with auto-layout

```javascript
// Screen-sized frame (desktop)
const frame = figma.createFrame();
frame.name = "1.1 — Dashboard default state";
frame.resize(1440, 900);
frame.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];

// Auto-layout container
const container = figma.createFrame();
container.name = "Header";
container.layoutMode = 'HORIZONTAL'; // or 'VERTICAL'
container.primaryAxisSizingMode = 'AUTO'; // hug content
container.counterAxisSizingMode = 'FIXED'; // fixed width
container.resize(1440, 64);
container.paddingTop = 16;
container.paddingBottom = 16;
container.paddingLeft = 24;
container.paddingRight = 24;
container.itemSpacing = 12;
container.fills = [{ type: 'SOLID', color: { r: 0.97, g: 0.97, b: 0.97 } }];
```

---

## 3. Importing and using design system components

```javascript
// Import a component by its key (from search_design_system results)
const component = await figma.importComponentByKeyAsync("component_key_here");
const instance = component.createInstance();

// For component sets (components with variants)
const componentSet = await figma.importComponentSetByKeyAsync("set_key_here");
// Find a specific variant
const variant = componentSet.findChild(
  n => n.name === "Type=Primary, Size=Medium"
);
if (variant && variant.type === "COMPONENT") {
  const inst = variant.createInstance();
  // Or set properties on the instance
  inst.setProperties({ "Type": "Primary", "Size": "Medium" });
}

// Position and add to parent
instance.x = 24;
instance.y = 16;
parentFrame.appendChild(instance);
```

---

## 4. Text nodes

```javascript
const text = figma.createText();
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
text.fontName = { family: "Inter", style: "Regular" };
text.characters = "Your text here";
text.fontSize = 14;
text.lineHeight = { value: 20, unit: 'PIXELS' };
text.fills = [{ type: 'SOLID', color: { r: 0.13, g: 0.13, b: 0.13 } }];
text.textAutoResize = 'WIDTH_AND_HEIGHT'; // or 'HEIGHT' for fixed width

// For multi-line text with fixed width:
text.resize(300, 1); // set width, height will auto-expand
text.textAutoResize = 'HEIGHT';
```

---

## 5. Annotation callouts

### Interaction annotation (yellow)
```javascript
async function createAnnotation(text, detail, parent, x, y) {
  const anno = figma.createFrame();
  anno.name = `Annotation: ${text}`;
  anno.layoutMode = 'VERTICAL';
  anno.primaryAxisSizingMode = 'AUTO';
  anno.counterAxisSizingMode = 'AUTO';
  anno.paddingTop = 10;
  anno.paddingBottom = 10;
  anno.paddingLeft = 14;
  anno.paddingRight = 14;
  anno.itemSpacing = 4;
  anno.cornerRadius = 6;
  anno.fills = [{ type: 'SOLID', color: { r: 1, g: 0.96, b: 0.82 } }];
  anno.strokes = [{ type: 'SOLID', color: { r: 0.9, g: 0.78, b: 0.4 } }];
  anno.strokeWeight = 1;

  await figma.loadFontAsync({ family: "Inter", style: "Semi Bold" });
  const label = figma.createText();
  label.fontName = { family: "Inter", style: "Semi Bold" };
  label.characters = text;
  label.fontSize = 11;
  label.fills = [{ type: 'SOLID', color: { r: 0.55, g: 0.38, b: 0 } }];
  anno.appendChild(label);

  if (detail) {
    await figma.loadFontAsync({ family: "Inter", style: "Regular" });
    const desc = figma.createText();
    desc.fontName = { family: "Inter", style: "Regular" };
    desc.characters = detail;
    desc.fontSize = 10;
    desc.fills = [{ type: 'SOLID', color: { r: 0.45, g: 0.35, b: 0.05 } }];
    desc.resize(260, 1);
    desc.textAutoResize = 'HEIGHT';
    anno.appendChild(desc);
  }

  anno.x = x;
  anno.y = y;
  parent.appendChild(anno);
  return anno;
}
```

### Error path annotation (red)
Same pattern but with:
```javascript
fills: [{ type: 'SOLID', color: { r: 1, g: 0.92, b: 0.92 } }]
strokes: [{ type: 'SOLID', color: { r: 0.9, g: 0.5, b: 0.5 } }]
// Text fills: { r: 0.7, g: 0.15, b: 0.15 }
```

### Success path annotation (green)
```javascript
fills: [{ type: 'SOLID', color: { r: 0.92, g: 1, b: 0.92 } }]
strokes: [{ type: 'SOLID', color: { r: 0.5, g: 0.8, b: 0.5 } }]
// Text fills: { r: 0.1, g: 0.5, b: 0.15 }
```

---

## 6. Flow arrows and connectors

```javascript
// Horizontal arrow between two frames
function createFlowArrow(fromFrame, toFrame, parent, label) {
  const startX = fromFrame.x + fromFrame.width + 10;
  const startY = fromFrame.y + fromFrame.height / 2;
  const endX = toFrame.x - 10;

  const line = figma.createLine();
  line.name = label ? `Flow: ${label}` : "Flow arrow";
  line.resize(endX - startX, 0);
  line.x = startX;
  line.y = startY;
  line.strokes = [{ type: 'SOLID', color: { r: 0.25, g: 0.45, b: 0.95 } }];
  line.strokeWeight = 2;
  line.strokeCap = 'ARROW_EQUILATERAL'; // arrowhead at end

  parent.appendChild(line);
  return line;
}
```

For branching flows (one frame leads to two outcomes), create two arrows — one straight
and one angled — with labels explaining the branch condition.

---

## 7. Section containers

Group flow frames in a large section frame:

```javascript
const section = figma.createSection();
section.name = "Flow 1: Create New Item";
// Sections auto-resize to fit children
// Add frames as children of the section
section.appendChild(frame1);
section.appendChild(frame2);
```

Alternative using a large frame if sections aren't suitable:

```javascript
const sectionFrame = figma.createFrame();
sectionFrame.name = "Flow 1: Create New Item";
sectionFrame.resize(5000, 1200);
sectionFrame.fills = []; // transparent
sectionFrame.clipsContent = false;
```

---

## 8. Badge/tag overlays

For marking components without DS matches, or tagging frame states:

```javascript
async function createBadge(text, color, parent, x, y) {
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
  badge.fills = [{ type: 'SOLID', color: color }];

  await figma.loadFontAsync({ family: "Inter", style: "Semi Bold" });
  const label = figma.createText();
  label.fontName = { family: "Inter", style: "Semi Bold" };
  label.characters = text;
  label.fontSize = 9;
  label.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
  badge.appendChild(label);

  badge.x = x;
  badge.y = y;
  parent.appendChild(badge);
  return badge;
}

// Usage examples:
// "No DS component" badge — orange
createBadge("No DS Component", { r: 0.9, g: 0.5, b: 0.1 }, frame, 8, 8);
// "Loading state" badge — blue
createBadge("Loading State", { r: 0.25, g: 0.45, b: 0.95 }, frame, 8, 8);
// "Error state" badge — red
createBadge("Error State", { r: 0.85, g: 0.2, b: 0.2 }, frame, 8, 8);
```

---

## 9. Positioning and spacing

Standard spacing between frames in a flow:

```javascript
const FRAME_GAP = 200;       // horizontal gap between sequential frames
const BRANCH_GAP = 100;      // vertical gap between branching frames
const ANNOTATION_OFFSET = 20; // gap between frame and its annotations
const FLOW_SECTION_GAP = 400; // vertical gap between flow sections

// Position frames in a horizontal sequence
function layoutFlowFrames(frames, startX, startY) {
  let currentX = startX;
  for (const frame of frames) {
    frame.x = currentX;
    frame.y = startY;
    currentX += frame.width + FRAME_GAP;
  }
}
```

---

## 10. Color palette for annotations

Consistent colors make the output scannable:

| Element | Background RGB | Stroke RGB | Text RGB |
|---|---|---|---|
| Interaction trigger | 1, 0.96, 0.82 | 0.9, 0.78, 0.4 | 0.55, 0.38, 0 |
| Success path | 0.92, 1, 0.92 | 0.5, 0.8, 0.5 | 0.1, 0.5, 0.15 |
| Error path | 1, 0.92, 0.92 | 0.9, 0.5, 0.5 | 0.7, 0.15, 0.15 |
| Info / neutral | 0.93, 0.95, 1 | 0.6, 0.7, 0.95 | 0.2, 0.3, 0.65 |
| Flow arrow | — | 0.25, 0.45, 0.95 | — |
| "No DS" badge | 0.9, 0.5, 0.1 | — | 1, 1, 1 |
| State badge | 0.25, 0.45, 0.95 | — | 1, 1, 1 |
