# Figma Plugin API Patterns Reference

Common patterns for building prototype-to-Figma output. Read this before your first
`use_figma` call in a session.

> **Write-tier clients only.** These patterns require `use_figma` (Figma Plugin API access).
> If your client only supports Inspect tools, refer to the Prototype Spec Document section in
> `SKILL.md` instead.

## Table of Contents
1. Font loading
2. Creating frames
3. Importing DS components (matched elements)
4. Building from primitives (unmatched elements)
5. The "No DS match" badge
6. Native Figma Dev Mode annotations
7. Flow arrows and connectors
8. Section containers
9. Positioning and spacing
10. Annotation category reference
11. Defensive annotation helpers (platform-safe)

---

## 1. Font loading

Always load fonts before setting any text character. Inter is the standard Figma font.

```javascript
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
await figma.loadFontAsync({ family: "Inter", style: "Semi Bold" });
await figma.loadFontAsync({ family: "Inter", style: "Bold" });
```

Note: "Semi Bold" has a space (not "SemiBold"). Same for "Extra Bold".

---

## 2. Creating frames

Every state in the prototype gets one top-level frame.

```javascript
// Desktop (1440×900)
const frame = figma.createFrame();
frame.name = "1.1 — Dashboard default";
frame.resize(1440, 900);
frame.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];

// Mobile (390×844, iPhone 14 Pro)
const mobileFrame = figma.createFrame();
mobileFrame.name = "1.1 — Dashboard default (mobile)";
mobileFrame.resize(390, 844);
mobileFrame.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];

// Auto-layout container inside a frame
const container = figma.createFrame();
container.name = "Header";
container.layoutMode = 'HORIZONTAL';
container.primaryAxisSizingMode = 'FIXED';
container.counterAxisSizingMode = 'AUTO';
container.resize(1440, 1); // height auto-expands
container.paddingTop = 16;
container.paddingBottom = 16;
container.paddingLeft = 24;
container.paddingRight = 24;
container.itemSpacing = 12;
container.fills = [{ type: 'SOLID', color: { r: 0.97, g: 0.97, b: 0.97 } }];
```

---

## 3. Importing DS components (matched elements)

Use this for every component that has a confirmed DS match from `search_design_system`.

```javascript
// Import a single component by its key
const component = await figma.importComponentByKeyAsync("component_key_here");
const instance = component.createInstance();

// Set variant/property values using exact names from get_context_for_code_connect
// e.g., { "Variant": ["Primary", "Secondary"], "Size": ["sm", "md", "lg"] }
instance.setProperties({ "Variant": "Primary", "Size": "md", "State": "Default" });

instance.x = 24;
instance.y = 16;
parentFrame.appendChild(instance);
```

For component sets (variants grouped together):

```javascript
const componentSet = await figma.importComponentSetByKeyAsync("set_key_here");
const variant = componentSet.defaultVariant; // or find a specific one:
const specific = componentSet.findChild(n => n.name === "Variant=Primary, Size=Medium");
if (specific?.type === "COMPONENT") {
  const inst = specific.createInstance();
  parentFrame.appendChild(inst);
}
```

> **Never call `figma.createComponent()` or `figma.createComponentSet()`.**
> These create new master components in the Figma file and pollute the design system.
> Only use `importComponentByKeyAsync` for DS components.

---

## 4. Building from primitives (unmatched elements)

When a prototype component has no DS match, approximate it visually using plain frames,
rectangles, and text. The goal is to make reviewers understand the intent — not to be
pixel-perfect. Always follow with a "No DS match" badge (see section 5).

```javascript
// ─── Button (no DS match) ─────────────────────────────────────────────────
async function buildPrimitiveButton(label, variant = 'primary', parent, x, y) {
  await figma.loadFontAsync({ family: "Inter", style: "Semi Bold" });

  const bg = variant === 'primary'
    ? { r: 0.09, g: 0.46, b: 0.96 }   // blue
    : { r: 0.93, g: 0.93, b: 0.93 };  // light gray

  const textColor = variant === 'primary'
    ? { r: 1, g: 1, b: 1 }
    : { r: 0.1, g: 0.1, b: 0.1 };

  const btn = figma.createFrame();
  btn.name = `Button [no DS match]: ${label}`;
  btn.layoutMode = 'HORIZONTAL';
  btn.primaryAxisSizingMode = 'AUTO';
  btn.counterAxisSizingMode = 'AUTO';
  btn.paddingTop = 10; btn.paddingBottom = 10;
  btn.paddingLeft = 20; btn.paddingRight = 20;
  btn.cornerRadius = 6;
  btn.fills = [{ type: 'SOLID', color: bg }];

  const text = figma.createText();
  text.fontName = { family: "Inter", style: "Semi Bold" };
  text.characters = label;
  text.fontSize = 14;
  text.fills = [{ type: 'SOLID', color: textColor }];
  btn.appendChild(text);

  btn.x = x; btn.y = y;
  parent.appendChild(btn);
  return btn;
}

// ─── Input field (no DS match) ────────────────────────────────────────────
async function buildPrimitiveInput(placeholder, parent, x, y, width = 320) {
  await figma.loadFontAsync({ family: "Inter", style: "Regular" });

  const input = figma.createFrame();
  input.name = 'Input [no DS match]';
  input.resize(width, 40);
  input.cornerRadius = 4;
  input.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
  input.strokes = [{ type: 'SOLID', color: { r: 0.8, g: 0.8, b: 0.8 } }];
  input.strokeWeight = 1;
  input.paddingLeft = 12; input.paddingRight = 12;
  input.paddingTop = 10; input.paddingBottom = 10;
  input.layoutMode = 'HORIZONTAL';
  input.primaryAxisAlignItems = 'CENTER';

  const text = figma.createText();
  text.fontName = { family: "Inter", style: "Regular" };
  text.characters = placeholder;
  text.fontSize = 14;
  text.fills = [{ type: 'SOLID', color: { r: 0.65, g: 0.65, b: 0.65 } }];
  input.appendChild(text);

  input.x = x; input.y = y;
  parent.appendChild(input);
  return input;
}

// ─── Card / container (no DS match) ──────────────────────────────────────
function buildPrimitiveCard(name, width, parent, x, y) {
  const card = figma.createFrame();
  card.name = `${name} [no DS match]`;
  card.resize(width, 1); // height auto via auto-layout
  card.layoutMode = 'VERTICAL';
  card.primaryAxisSizingMode = 'AUTO';
  card.counterAxisSizingMode = 'FIXED';
  card.paddingTop = 20; card.paddingBottom = 20;
  card.paddingLeft = 20; card.paddingRight = 20;
  card.itemSpacing = 12;
  card.cornerRadius = 8;
  card.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
  card.effects = [{
    type: 'DROP_SHADOW',
    color: { r: 0, g: 0, b: 0, a: 0.08 },
    offset: { x: 0, y: 2 },
    radius: 8,
    spread: 0,
    visible: true,
    blendMode: 'NORMAL'
  }];
  card.x = x; card.y = y;
  parent.appendChild(card);
  return card;
}

// ─── Toast / banner (no DS match) ────────────────────────────────────────
async function buildPrimitiveBanner(message, type = 'info', parent, x, y, width = 400) {
  await figma.loadFontAsync({ family: "Inter", style: "Regular" });

  const colors = {
    info:    { bg: { r: 0.92, g: 0.96, b: 1.00 }, text: { r: 0.1, g: 0.3, b: 0.7 } },
    success: { bg: { r: 0.90, g: 0.98, b: 0.91 }, text: { r: 0.1, g: 0.5, b: 0.2 } },
    warning: { bg: { r: 1.00, g: 0.97, b: 0.88 }, text: { r: 0.6, g: 0.4, b: 0.0 } },
    error:   { bg: { r: 1.00, g: 0.93, b: 0.93 }, text: { r: 0.7, g: 0.1, b: 0.1 } },
  };
  const c = colors[type] ?? colors.info;

  const banner = figma.createFrame();
  banner.name = `Banner [no DS match]: ${type}`;
  banner.resize(width, 1);
  banner.layoutMode = 'HORIZONTAL';
  banner.primaryAxisSizingMode = 'FIXED';
  banner.counterAxisSizingMode = 'AUTO';
  banner.paddingTop = 12; banner.paddingBottom = 12;
  banner.paddingLeft = 16; banner.paddingRight = 16;
  banner.cornerRadius = 6;
  banner.fills = [{ type: 'SOLID', color: c.bg }];

  const text = figma.createText();
  text.fontName = { family: "Inter", style: "Regular" };
  text.characters = message;
  text.fontSize = 14;
  text.fills = [{ type: 'SOLID', color: c.text }];
  text.layoutGrow = 1;
  text.textAutoResize = 'HEIGHT';
  banner.appendChild(text);

  banner.x = x; banner.y = y;
  parent.appendChild(banner);
  return banner;
}
```

**General rule for primitives:** Match the visual intent (color, shape, size) of the prototype
element as closely as you can from reading the source code. Exact pixel-perfection is not
required — recognizability is.

---

## 5. The "No DS match" badge

Add this to every primitive element so reviewers and designers can identify which components
still need a DS counterpart.

```javascript
async function addNoDsMatchBadge(node) {
  await figma.loadFontAsync({ family: "Inter", style: "Semi Bold" });

  const badge = figma.createFrame();
  badge.name = "⚠ No DS match";
  badge.layoutMode = 'HORIZONTAL';
  badge.primaryAxisSizingMode = 'AUTO';
  badge.counterAxisSizingMode = 'AUTO';
  badge.paddingTop = 2; badge.paddingBottom = 2;
  badge.paddingLeft = 6; badge.paddingRight = 6;
  badge.cornerRadius = 3;
  badge.fills = [{ type: 'SOLID', color: { r: 1, g: 0.6, b: 0.1 } }]; // orange

  const label = figma.createText();
  label.fontName = { family: "Inter", style: "Semi Bold" };
  label.characters = "No DS match";
  label.fontSize = 9;
  label.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
  badge.appendChild(label);

  // Position at top-left of the node
  badge.x = node.x + 4;
  badge.y = node.y + 4;

  // Append to the same parent so it floats above the node
  node.parent.appendChild(badge);
  return badge;
}
```

Call it immediately after creating any primitive approximation:
```javascript
const alertBanner = await buildPrimitiveBanner("Something went wrong", "error", frame, 24, 120);
await addNoDsMatchBadge(alertBanner);
```

---

## 6. Native Figma Dev Mode annotations

Real Figma annotations that appear in Dev Mode, support markdown, can be filtered by category,
and stay attached to their node. Do NOT create colored rectangles on the canvas as substitutes.

> **Use the defensive helpers in Section 11** instead of calling `figma.annotations` directly.
> Direct calls to `addAnnotationCategoryAsync` will throw if the category already exists, and
> direct `node.annotations = [...]` assignment will fail silently on some platforms. The Section
> 11 helpers handle both failure modes with automatic fallbacks.

### Setting up categories (once per session)

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

If categories were already created in this file:
```javascript
const existing = await figma.annotations.getAnnotationCategoriesAsync();
const interactionCat = existing.find(c => c.label === 'Interaction');
```

Available colors: `'yellow'`, `'orange'`, `'red'`, `'pink'`, `'violet'`, `'blue'`, `'teal'`,
`'green'`.

### Adding annotations to nodes

```javascript
// Simple
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

// Multiple annotations (different concerns)
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

## 7. Flow arrows and connectors

```javascript
function createFlowArrow(fromFrame, toFrame, transitionLabel, catId) {
  const arrow = figma.createLine();
  arrow.name = `Flow: ${fromFrame.name} → ${toFrame.name}`;
  const length = toFrame.x - (fromFrame.x + fromFrame.width) - 20;
  arrow.resize(Math.max(length, 40), 0);
  arrow.x = fromFrame.x + fromFrame.width + 10;
  arrow.y = fromFrame.y + fromFrame.height / 2;
  arrow.strokes = [{ type: 'SOLID', color: { r: 0.25, g: 0.45, b: 0.95 } }];
  arrow.strokeWeight = 2;
  arrow.strokeCap = 'ARROW_EQUILATERAL';

  if (transitionLabel && catId) {
    arrow.annotations = [
      { labelMarkdown: transitionLabel, categoryId: catId }
    ];
  }

  figma.currentPage.appendChild(arrow);
  return arrow;
}
```

For branching flows (one frame → two outcomes), create two arrows with labels explaining the
branch condition.

---

## 8. Section containers

```javascript
// Preferred: Figma native sections
const section = figma.createSection();
section.name = "Flow 1: Create New Item";
section.appendChild(frame1);
section.appendChild(frame2);
// Sections auto-resize to fit children

// Fallback: transparent large frame
const sectionFrame = figma.createFrame();
sectionFrame.name = "Flow 1: Create New Item";
sectionFrame.resize(8000, 1200);
sectionFrame.fills = [];
sectionFrame.clipsContent = false;
```

---

## 9. Positioning and spacing

```javascript
const FRAME_GAP = 200;         // horizontal gap between sequential state frames
const BRANCH_GAP = 100;        // vertical gap between branching frames (e.g. 1.3a / 1.3b)
const FLOW_SECTION_GAP = 400;  // vertical gap between flow sections

function layoutSequence(frames, startX = 100, startY = 100) {
  let x = startX;
  for (const frame of frames) {
    frame.x = x;
    frame.y = startY;
    x += frame.width + FRAME_GAP;
  }
}

function layoutBranch(successFrame, errorFrame, afterX, baseY) {
  successFrame.x = afterX;
  successFrame.y = baseY;
  errorFrame.x = afterX;
  errorFrame.y = baseY + successFrame.height + BRANCH_GAP;
}
```

---

## 10. Annotation category reference

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

Reviewers can filter by any category in Dev Mode to focus on what's relevant to their role.

---

## 11. Defensive annotation helpers (platform-safe)

Use these helpers instead of calling `figma.annotations` directly. They handle two failure
modes that cause annotations to silently disappear on some platforms:

1. **Categories already exist** — `addAnnotationCategoryAsync` throws if a category with that
   label was already created in this file. The helper checks first and returns the existing one.
2. **Native API unavailable** — If `figma.annotations` is undefined or throws entirely,
   `annotateNode` falls back to a canvas text overlay so annotation content is never lost.

```javascript
// ─── Get or create an annotation category safely ─────────────────────────
async function getOrCreateAnnotationCategory(label, color) {
  try {
    const existing = await figma.annotations.getAnnotationCategoriesAsync();
    const found = existing.find(c => c.label === label);
    if (found) return found;
    return await figma.annotations.addAnnotationCategoryAsync({ label, color });
  } catch (e) {
    return null; // annotateNode will use text fallback when categoryId is null
  }
}

// ─── Annotate a node safely, with text-overlay fallback ──────────────────
async function annotateNode(node, markdownText, categoryId) {
  try {
    const entry = categoryId
      ? { labelMarkdown: markdownText, categoryId }
      : { label: markdownText };
    node.annotations = [...(node.annotations || []), entry];
  } catch (e) {
    // Native annotation API failed — place a visible text label near the node instead.
    // This ensures annotation content always appears in the output.
    try {
      await figma.loadFontAsync({ family: "Inter", style: "Regular" });
      const plain = markdownText.replace(/\*\*/g, '').replace(/\n/g, ' | ');
      const label = figma.createText();
      label.fontName = { family: "Inter", style: "Regular" };
      label.characters = `[Annotation] ${plain}`;
      label.fontSize = 11;
      label.fills = [{ type: 'SOLID', color: { r: 0.18, g: 0.36, b: 0.94 } }];
      label.x = node.absoluteBoundingBox?.x ?? node.x;
      label.y = (node.absoluteBoundingBox?.y ?? node.y) - 20;
      figma.currentPage.appendChild(label);
    } catch (_) {
      // Last resort: encode the annotation in the node's name so it's visible in the layer panel
      node.name = `${node.name} [Note: ${markdownText.substring(0, 80)}]`;
    }
  }
}
```

**Usage:**

```javascript
// Set up categories once before building any frame
const interactionCat = await getOrCreateAnnotationCategory('Interaction', 'blue');
const validationCat  = await getOrCreateAnnotationCategory('Validation',  'orange');
// ... etc

// Annotate each interactive element using the helper
await annotateNode(saveButton, '**On click →** Submits form via POST /api/items', interactionCat?.id);
await annotateNode(emailField, '**Validation:** Required, must be valid email', validationCat?.id);
```

> **Note:** Always use `categoryId?.id` (optional chaining) so a `null` category (returned
> when the API is unavailable) degrades gracefully to an uncategorized annotation rather than
> throwing.
