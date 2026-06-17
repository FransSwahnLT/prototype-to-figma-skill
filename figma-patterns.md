# Figma Plugin API Patterns Reference

Common patterns for building prototype-to-Figma output. Read this before your first
`use_figma` call in a session.

> **Write-tier clients only.** These patterns require `use_figma` (Figma Plugin API access).
> If your client only supports Inspect tools, refer to the Prototype Spec Document section in
> `SKILL.md` instead.

## Table of Contents
1. Font loading and text style binding
2. Creating frames
3. Importing DS components (matched elements)
4. Building from primitives (unmatched elements)
5. The "No DS match" badge
6. Native Figma Dev Mode annotations
7. Flow arrows and connectors
8. Section containers
9. Positioning and spacing
10. Annotation category reference
11. Defensive annotation helpers (platform-safe, includes DS Drift category)
12. Prototype Spec Document template (Inspect-only clients)
13. Rocket 3.0 variable + text-style helpers

---

## 1. Font loading and text style binding

### Rocket 3.0 text styles (preferred)

Bind text nodes to Rocket 3.0 named styles instead of manually loading fonts. This keeps
text styles connected to the DS across all files that have the Rocket 3.0 library linked.
See Section 13 for the `applyTextStyle` helper.

| Role | Rocket 3.0 style name |
|------|----------------------|
| Body default | `body/body1` (16px Regular) |
| Small body / metadata | `body/body3` (12px Regular) |
| Section label / emphasis | `subtitle/subtitle1` (16px Semi Bold) |
| Eyebrow / overline | `caption/section-title` (10px Medium) |
| CTA / button label | `button/button-sm` (12px Medium) |

```javascript
// Preferred: bind to Rocket 3.0 style by name (see Section 13 for helper)
const label = figma.createText();
await figma.loadFontAsync({ family: "Inter", style: "Regular" }); // still required before setting .characters
label.characters = "Submit";
await applyTextStyle(label, "button/button-sm"); // binds DS style
```

### Font loading fallback

When no Rocket 3.0 text styles are found (primitives in a file without the R3 library), load
Inter directly:

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
container.resize(1440, 1);               // ← resize() FIRST
container.primaryAxisSizingMode = 'FIXED';  // ← sizing modes AFTER resize()
container.counterAxisSizingMode = 'AUTO';   // height auto-expands
container.paddingTop = 16;
container.paddingBottom = 16;
container.paddingLeft = 24;
container.paddingRight = 24;
container.itemSpacing = 12;
container.fills = [{ type: 'SOLID', color: { r: 0.97, g: 0.97, b: 0.97 } }];
```

> **Gotcha: always call `resize()` before setting sizing modes.**
> Calling `resize()` on an auto-layout frame silently resets `primaryAxisSizingMode` and
> `counterAxisSizingMode` back to `'FIXED'`. If you set them before `resize()`, the frame
> collapses to 1px tall (or wide) and renders as invisible. The fix: call `resize()` first,
> then set sizing modes. This applies to every auto-layout frame — including the flow overview
> frame and any container that should grow with its content.

> **Rocket 3.0 layout gotchas:**
> - `layoutSizingHorizontal = 'FILL'` must be set **after** `parent.appendChild(node)` — the
>   property is ignored if set before the node has a parent with auto-layout.
> - For component sets (e.g. Badge, IconButton): use `figma.importComponentSetByKeyAsync(key)`,
>   not `importComponentByKeyAsync` — the latter only works for individual variants.
> - Badge / pill instances: set `layoutSizingHorizontal = 'HUG'` and
>   `primaryAxisSizingMode = 'AUTO'` after creating the instance to get the correct pill shape.
> - `figma.variables.getLocalVariablesAsync()` may return an empty array in some Figma clients
>   even when library variables exist. Fall back to `figma.importVariableByKeyAsync(key)` if
>   name-based lookup returns null. See Section 13 for the helper that handles both.

### Scrollable frames

When a prototype state scrolls vertically, the frame must be tall enough to show all content —
never clip at the viewport height. Set `overflowDirection` so Figma knows it scrolls, and add
a fold marker line so reviewers can see where the visible viewport ends.

```javascript
const VIEWPORT_HEIGHT = 900; // or 844 for mobile
const fullContentHeight = 1800; // estimated from prototype — how tall the full page is

const frame = figma.createFrame();
frame.name = "2.1 — Settings page (scrollable)";
frame.resize(1440, fullContentHeight); // full content height, not viewport height
frame.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
frame.overflowDirection = 'VERTICAL'; // marks this as a scrolling frame in Figma

// ... build all content into the frame at their natural positions ...

// Fold marker — dashed line at the viewport height showing where the screen cuts off
const foldLine = figma.createLine();
foldLine.name = "— viewport fold —";
foldLine.resize(1440, 0);
foldLine.x = 0;
foldLine.y = VIEWPORT_HEIGHT;
foldLine.strokes = [{ type: 'SOLID', color: { r: 0.6, g: 0.2, b: 0.9 } }]; // purple
foldLine.strokeWeight = 1.5;
foldLine.dashPattern = [8, 4];
frame.appendChild(foldLine);

// Annotate the fold line so reviewers understand it
await annotateNode(
  foldLine,
  `**Viewport fold** — content below this line requires scrolling. ` +
  `Visible area: ${1440}×${VIEWPORT_HEIGHT}px. Full page: ${1440}×${fullContentHeight}px.`,
  stateCat?.id
);
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

Primitives use Rocket 3.0 token names for colors and radii via the helpers in Section 13
(`bindColor`, `bindRadius`, `applyTextStyle`). This ensures primitives visually match the DS
even when no DS component exists. Fall back to hardcoded values only if the R3 library is not
linked in the file.

```javascript
// ─── Button (no DS match) ─────────────────────────────────────────────────
async function buildPrimitiveButton(label, variant = 'primary', parent, x, y) {
  await figma.loadFontAsync({ family: "Inter", style: "Semi Bold" });

  const btn = figma.createFrame();
  btn.name = `Button [no DS match]: ${label}`;
  btn.layoutMode = 'HORIZONTAL';
  btn.primaryAxisSizingMode = 'AUTO';
  btn.counterAxisSizingMode = 'AUTO';
  btn.paddingTop = 10; btn.paddingBottom = 10;
  btn.paddingLeft = 20; btn.paddingRight = 20;

  // Rocket 3.0 tokens — fall back to hex if variable not found
  if (variant === 'primary') {
    const bound = await bindColor(btn, 0, 'color/brand/primary');
    if (!bound) btn.fills = [{ type: 'SOLID', color: { r: 0, g: 0.77, b: 0.70 } }]; // #00C4B3
  } else {
    const bound = await bindColor(btn, 0, 'color/bg/surface');
    if (!bound) btn.fills = [{ type: 'SOLID', color: { r: 0.93, g: 0.93, b: 0.93 } }];
  }
  const radiusBound = await bindRadius(btn, 'radius/md');
  if (!radiusBound) btn.cornerRadius = 6;

  const text = figma.createText();
  text.fontName = { family: "Inter", style: "Semi Bold" };
  text.characters = label;
  text.fontSize = 14;
  await applyTextStyle(text, 'button/button-sm');
  if (variant === 'primary') {
    const bound = await bindColor(text, 0, 'color/bg/default');
    if (!bound) text.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
  } else {
    const bound = await bindColor(text, 0, 'color/text/primary');
    if (!bound) text.fills = [{ type: 'SOLID', color: { r: 0.1, g: 0.1, b: 0.1 } }];
  }
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
  const radiusBound = await bindRadius(input, 'radius/sm');
  if (!radiusBound) input.cornerRadius = 4;
  const bgBound = await bindColor(input, 0, 'color/bg/surface');
  if (!bgBound) input.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
  const strokeBound = await bindColor(input, 0, 'color/border/default'); // strokes use same helper
  if (!strokeBound) input.strokes = [{ type: 'SOLID', color: { r: 0.8, g: 0.8, b: 0.8 } }];
  else input.strokes = [{ type: 'SOLID' }]; // color bound via variable
  input.strokeWeight = 1;
  input.paddingLeft = 12; input.paddingRight = 12;
  input.paddingTop = 10; input.paddingBottom = 10;
  input.layoutMode = 'HORIZONTAL';
  input.primaryAxisAlignItems = 'CENTER';

  const text = figma.createText();
  text.fontName = { family: "Inter", style: "Regular" };
  text.characters = placeholder;
  await applyTextStyle(text, 'body/body1');
  const textBound = await bindColor(text, 0, 'color/text/secondary');
  if (!textBound) text.fills = [{ type: 'SOLID', color: { r: 0.65, g: 0.65, b: 0.65 } }];
  input.appendChild(text);

  input.x = x; input.y = y;
  parent.appendChild(input);
  return input;
}

// ─── Card / container (no DS match) ──────────────────────────────────────
async function buildPrimitiveCard(name, width, parent, x, y) {
  const card = figma.createFrame();
  card.name = `${name} [no DS match]`;
  card.resize(width, 1); // height auto via auto-layout
  card.layoutMode = 'VERTICAL';
  card.primaryAxisSizingMode = 'AUTO';
  card.counterAxisSizingMode = 'FIXED';
  card.paddingTop = 20; card.paddingBottom = 20;
  card.paddingLeft = 20; card.paddingRight = 20;
  card.itemSpacing = 12;
  const radiusBound = await bindRadius(card, 'radius/lg');
  if (!radiusBound) card.cornerRadius = 8;
  const bgBound = await bindColor(card, 0, 'color/bg/surface');
  if (!bgBound) card.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
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

  // Semantic color fallbacks (R3 doesn't expose semantic alert colors as variables)
  const fallbackColors = {
    info:    { bg: { r: 0.92, g: 0.96, b: 1.00 }, text: { r: 0.1, g: 0.3, b: 0.7 } },
    success: { bg: { r: 0.90, g: 0.98, b: 0.91 }, text: { r: 0.1, g: 0.5, b: 0.2 } },
    warning: { bg: { r: 1.00, g: 0.97, b: 0.88 }, text: { r: 0.6, g: 0.4, b: 0.0 } },
    error:   { bg: { r: 1.00, g: 0.93, b: 0.93 }, text: { r: 0.7, g: 0.1, b: 0.1 } },
  };
  const c = fallbackColors[type] ?? fallbackColors.info;

  const banner = figma.createFrame();
  banner.name = `Banner [no DS match]: ${type}`;
  banner.resize(width, 1);
  banner.layoutMode = 'HORIZONTAL';
  banner.primaryAxisSizingMode = 'FIXED';
  banner.counterAxisSizingMode = 'AUTO';
  banner.paddingTop = 12; banner.paddingBottom = 12;
  banner.paddingLeft = 16; banner.paddingRight = 16;
  const radiusBound = await bindRadius(banner, 'radius/md');
  if (!radiusBound) banner.cornerRadius = 6;
  // Banner bg: use brand primary for info, raw hex for success/warning/error (no R3 semantic tokens)
  if (type === 'info') {
    const bound = await bindColor(banner, 0, 'color/brand/primary');
    if (!bound) banner.fills = [{ type: 'SOLID', color: c.bg }];
  } else {
    banner.fills = [{ type: 'SOLID', color: c.bg }];
  }

  const text = figma.createText();
  text.fontName = { family: "Inter", style: "Regular" };
  text.characters = message;
  await applyTextStyle(text, 'body/body1');
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
| DS Drift | `'red'` | Gap between code behavior and Figma DS component/variable |

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

**Usage — full category setup:**

```javascript
// Set up all categories once before building any frame
const interactionCat = await getOrCreateAnnotationCategory('Interaction',    'blue');
const navigationCat  = await getOrCreateAnnotationCategory('Navigation',     'violet');
const stateCat       = await getOrCreateAnnotationCategory('State Change',   'teal');
const validationCat  = await getOrCreateAnnotationCategory('Validation',     'orange');
const errorCat       = await getOrCreateAnnotationCategory('Error Handling', 'red');
const edgeCaseCat    = await getOrCreateAnnotationCategory('Edge Case',      'pink');
const dataCat        = await getOrCreateAnnotationCategory('Data / API',     'green');
const a11yCat        = await getOrCreateAnnotationCategory('Accessibility',  'yellow');
const dsDriftCat     = await getOrCreateAnnotationCategory('DS Drift',       'red');

// Interaction annotation
await annotateNode(saveButton, '**On click →** Submits form via POST /api/items', interactionCat?.id);

// DS Drift annotation — DS component with a missing variant
await annotateNode(
  tableInstance,
  '**DS Drift:** Code uses `sortable` prop — column headers are clickable to sort ' +
  '(ascending → descending → default). The Figma DS Table component has no sortable ' +
  'variant. Designers: this state needs a DS component update.',
  dsDriftCat?.id
);

// DS Drift annotation — no DS match, built from primitives
await annotateNode(
  customWidgetFrame,
  '**DS Drift:** No DS match found for `<CustomWidget>`. ' +
  'Searched: Widget, Card, Panel, Tile — none matched. ' +
  'Built from primitives. Design team: consider adding this to the DS.',
  dsDriftCat?.id
);
```

> **Note:** Always use `categoryId?.id` (optional chaining) so a `null` category (returned
> when the API is unavailable) degrades gracefully to an uncategorized annotation rather than
> throwing.

---

## 12. Prototype Spec Document template (Inspect-only clients)

Use this when Write tools (`use_figma`) are unavailable. Produce a structured markdown document
the team can read and comment on directly.

```markdown
# [Feature Name] — Prototype Spec

**Generated from:** [prototype file path or description]
**Target Figma file:** [URL if provided, or "not specified"]
**Date:** [today]

---

## Overview

[2–3 sentence description of what the prototype demonstrates]

**Open questions for reviewers:**
- [List ambiguous behaviors or design decisions needing input]

---

## Flows

### Flow 1: [Flow Name]

**Goal:** [What the user is trying to accomplish]
**Entry point:** [What triggers this flow]

#### States

**1.1 — [State name]**
- **Frame size:** 1440×900 (desktop) / 390×844 (mobile)
- **Layout:** [Describe the overall layout]
- **Components:**
  - [Component name] (`<Button variant="primary">`) → DS match: Button/Primary [verified] or [unverified]
  - [Component name] → No DS match: build from primitives

- **Interactions:**
  | Element | Trigger | Result | Notes |
  |---|---|---|---|
  | Save button | Click | Validates form → loading (→ 1.2) | Debounced 300ms |

- **Annotations:**
  - **Interaction:** [click targets and results]
  - **Validation:** [form rules]
  - **Data / API:** [endpoints, data sources]
  - **Error Handling:** [errors, recovery]
  - **Edge Cases:** [non-obvious behavior]
  - **Accessibility:** [tab order, keyboard shortcuts]

---

## Component inventory

| Code component | Props | DS match | Confidence | Build approach |
|---|---|---|---|---|
| `<Button>` | variant="primary", size="md" | Button/Primary | High | Import + setProperties |
| `<DataTable>` | columns, rows | Table | Medium | Import + setProperties |
| `<CustomWidget>` | (custom) | None | — | Primitives |

---

## Figma build guide

*For whoever builds this in Figma:*

**Page:** "[Feature Name] — Prototype Flows"
**Frame sizes:** 1440×900 desktop / 390×844 mobile
**Layout:** Left-to-right per flow, branches stacked vertically, 200px gaps

**Components to import from DS:** [list from inventory above]
**Components to build from primitives:** [list from inventory above]

**Important:** For components without DS matches, build from primitives (frames, rectangles,
text, auto-layout). Do NOT call `figma.createComponent()`. Do NOT skip the element.
```

---

## 13. Rocket 3.0 variable + text-style helpers

These helpers are used by the Section 4 primitive builders. Include them at the top of every
`use_figma` script that builds primitives.

**Before calling these helpers**, run the Phase 0 library check to build `ROCKET3_VAR_KEY_MAP`
by calling `search_design_system(fileKey=ROCKET3_DS_KEY, includeVariables=true)` and extracting
a `{ "token/name": "variableKey" }` map from the results. Pass it into the script so
`getRocket3Variable` can resolve variables even when the target file has no R3 library linked.

```javascript
// ─── Rocket 3.0 library file keys ────────────────────────────────────────
const ROCKET3_DS_KEY    = "pafe2Ef03rlBZDgJT1A49G"; // 🚀 Rocket 3.0 — UI Kit
const ROCKET3_ICONS_KEY = "FPtbVqunkX6NbtnC38pYTS"; // Rocket 3.0 — Icons

// ROCKET3_VAR_KEY_MAP is built at Phase 0 from search_design_system results:
// { "color/brand/primary": "<varKey>", "radius/md": "<varKey>", ... }
// Pass it into your use_figma script as a constant.

// ─── Look up a Rocket 3.0 variable by token path name ────────────────────
// 1. Fast path: local variables (works when R3 library is already linked in target file)
// 2. Library path: import by key using ROCKET3_VAR_KEY_MAP built from Phase 0 search
async function getRocket3Variable(namePath) {
  try {
    const vars = await figma.variables.getLocalVariablesAsync();
    if (vars.length > 0) {
      const found = vars.find(v => v.name === namePath);
      if (found) return found;
    }
  } catch (_) {}
  // Target file doesn't have R3 linked — import directly from library using key map
  try {
    if (typeof ROCKET3_VAR_KEY_MAP !== 'undefined' && ROCKET3_VAR_KEY_MAP[namePath]) {
      return await figma.importVariableByKeyAsync(ROCKET3_VAR_KEY_MAP[namePath]);
    }
  } catch (_) {}
  return null; // caller falls back to hardcoded hex
}

// ─── Bind a Rocket 3.0 color variable to a node's fills ──────────────────
// Returns true if variable was found and bound; false if caller should set raw fill.
// Note: for stroke color binding, use node.setBoundVariableForPaint('strokes', index, v)
// instead — this helper targets fills only.
async function bindColor(node, fillIndex, tokenName) {
  const v = await getRocket3Variable(tokenName);
  if (!v) return false;
  try {
    node.setBoundVariableForPaint('fills', fillIndex, v);
    return true;
  } catch (_) {
    return false;
  }
}

// ─── Bind a Rocket 3.0 radius variable to a node's cornerRadius ──────────
// Returns true if bound; false if caller should set raw cornerRadius number.
async function bindRadius(node, tokenName) {
  const v = await getRocket3Variable(tokenName);
  if (!v) return false;
  try {
    node.setBoundVariable('cornerRadius', v);
    return true;
  } catch (_) {
    return false;
  }
}

// ─── Apply a Rocket 3.0 named text style to a text node ──────────────────
// Must be called AFTER setting node.characters (font must already be loaded).
// No-ops silently if the style name is not found in the file.
async function applyTextStyle(textNode, styleName) {
  try {
    const styles = await figma.getLocalTextStylesAsync();
    const match = styles.find(s => s.name === styleName);
    if (match) textNode.textStyleId = match.id;
  } catch (_) {}
}
```

**Token name reference (Rocket 3.0):**

| Token | Name |
|-------|------|
| Brand teal | `color/brand/primary` |
| Page background | `color/bg/default` |
| Card / surface background | `color/bg/surface` |
| Primary text | `color/text/primary` |
| Secondary text / icons | `color/text/secondary` |
| Default border | `color/border/default` |
| Strong border | `color/border/strong` |
| Corner radius — small | `radius/sm` |
| Corner radius — medium | `radius/md` |
| Corner radius — large | `radius/lg` |
| Corner radius — extra large | `radius/xl` |
| Corner radius — pill | `radius/full` |
