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

## 5. Native Figma Dev Mode Annotations

These are REAL Figma annotations — they show up in Dev Mode, can be filtered by category,
support markdown, and stay attached to their target nodes. Do NOT create colored rectangles
on the canvas as fake annotations.

### Setting up annotation categories (do this once at the start)
```javascript
// Create categories so reviewers can filter by what they care about
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

Available colors: `'yellow'`, `'orange'`, `'red'`, `'pink'`, `'violet'`, `'blue'`,
`'teal'`, `'green'`.

### Adding annotations to nodes
```javascript
// Simple text annotation
node.annotations = [{ label: 'Opens settings modal on click' }];

// Rich markdown annotation with category
node.annotations = [
  {
    labelMarkdown: '**On click →** Submits form via `POST /api/items`\n\n' +
      '- Success: closes modal, refreshes list\n' +
      '- Error: shows inline error banner\n\n' +
      '*Debounced: 300ms*',
    categoryId: interactionCat.id
  }
];

// Multiple annotations on the same node (different concerns)
node.annotations = [
  {
    labelMarkdown: '**Validation:** Required, min 3 chars, max 100 chars',
    categoryId: validationCat.id
  },
  {
    labelMarkdown: '**Keyboard:** Tab to next field, Enter submits form',
    categoryId: a11yCat.id
  }
];

// Pinning design properties alongside notes
node.annotations = [
  {
    label: 'Responsive: 600px max on desktop, full-width on mobile',
    properties: [{ type: 'width' }, { type: 'maxWidth' }]
  }
];

// Clearing annotations
node.annotations = [];
```

### Retrieving existing categories (if file already has them)
```javascript
const existingCategories = await figma.annotations.getAnnotationCategoriesAsync();
// Find a specific one
const interactionCat = existingCategories.find(c => c.label === 'Interaction');
```

### Supported pinnable property types
`'width'`, `'height'`, `'maxWidth'`, `'minWidth'`, `'maxHeight'`, `'minHeight'`,
`'fills'`, `'strokes'`, `'effects'`, `'strokeWeight'`, `'cornerRadius'`,
`'textStyleId'`, `'textAlignHorizontal'`, `'fontFamily'`, `'fontStyle'`, `'fontSize'`,
`'fontWeight'`, `'lineHeight'`, `'letterSpacing'`, `'itemSpacing'`, `'padding'`,
`'layoutMode'`, `'alignItems'`, `'opacity'`, `'mainComponent'`

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

## 10. Annotation category reference

Native Figma annotation categories and their intended use:

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

Reviewers can filter by any of these categories in Dev Mode to focus on what's
relevant to their role.
