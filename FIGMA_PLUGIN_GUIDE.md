# Figma Plugin Development Guide

## Overview

This guide serves as the reference for building a professional design system plugin for Figma. Follow this strictly when implementing features.

---

## File Structure & Architecture

### Required Files

```
plugin-root/
├── manifest.json          # Plugin configuration (REQUIRED)
├── code.js               # Main plugin logic (runs in sandbox)
├── ui.html               # UI interface (runs in iframe)
└── package.json          # Dependencies and scripts
```

### Supported File Types in Figma

#### ✅ Supported in Sandbox (code.js)

- **JavaScript (.js)** - Main plugin code
- **TypeScript (.ts)** - Compiles to JS
- No Node.js modules (fs, path, etc.)
- No DOM access
- Figma Plugin API only

#### ✅ Supported in UI (ui.html)

- **HTML** - UI structure
- **CSS** - Styling (inline or `<style>` tags)
- **JavaScript** - UI logic (inline `<script>` tags)
- **SVG** - Icons and graphics (inline only)
- External resources via CDN (with network access)

#### ❌ NOT Supported

- External file imports (must bundle)
- Node.js built-in modules
- Direct file system access
- External CSS/JS files (must inline or bundle)

---

## Communication Architecture

### Two-Context System

```
┌─────────────────┐         ┌──────────────────┐
│   Sandbox       │ ←────→  │   UI (iframe)    │
│   (code.js)     │  msgs   │   (ui.html)      │
│                 │         │                  │
│ - Figma API     │         │ - DOM access     │
│ - No DOM        │         │ - User input     │
│ - Plugin logic  │         │ - No Figma API   │
└─────────────────┘         └──────────────────┘
```

### Message Passing

```javascript
// From code.js to UI
figma.ui.postMessage({ type: "action", data: {} });

// From UI to code.js
parent.postMessage({ pluginMessage: { type: "action" } }, "*");

// Receive in code.js
figma.ui.onmessage = (msg) => {
  if (msg.type === "action") {
    /* handle */
  }
};

// Receive in UI
window.onmessage = (event) => {
  const msg = event.data.pluginMessage;
  if (msg.type === "action") {
    /* handle */
  }
};
```

---

## Manifest.json Structure

```json
{
  "name": "Design System Manager",
  "id": "unique-plugin-id",
  "api": "1.0.0",
  "main": "code.js",
  "ui": "ui.html",
  "editorType": ["figma", "figjam"],
  "networkAccess": {
    "allowedDomains": ["none"]
  },
  "permissions": []
}
```

### Key Properties

- **name**: Plugin display name
- **main**: Entry point (sandbox code)
- **ui**: UI file path (optional)
- **editorType**: Where plugin runs
- **networkAccess**: External API access
- **permissions**: Special capabilities needed

---

## Figma Plugin API - Core Concepts

### 1. Node Types

```typescript
// Common node types
SceneNode; // Base for all visible nodes
FrameNode; // Frames and groups
ComponentNode; // Components
InstanceNode; // Component instances
TextNode; // Text layers
RectangleNode; // Rectangles
VectorNode; // Vector shapes
```

### 2. Selection

```javascript
// Get current selection
const selection = figma.currentPage.selection;

// Set selection
figma.currentPage.selection = [node];

// Listen to selection changes
figma.on("selectionchange", () => {
  const selected = figma.currentPage.selection;
});
```

### 3. Styles (Design Tokens)

```javascript
// Paint styles (colors)
const paintStyle = figma.createPaintStyle();
paintStyle.name = "Primary/500";
paintStyle.paints = [{ type: "SOLID", color: { r: 0, g: 0, b: 1 } }];

// Text styles (typography)
const textStyle = figma.createTextStyle();
textStyle.name = "Heading/H1";
textStyle.fontSize = 32;
textStyle.fontName = { family: "Inter", style: "Bold" };

// Effect styles (shadows)
const effectStyle = figma.createEffectStyle();
effectStyle.name = "Shadow/Medium";
effectStyle.effects = [
  {
    type: "DROP_SHADOW",
    color: { r: 0, g: 0, b: 0, a: 0.25 },
    offset: { x: 0, y: 4 },
    radius: 8,
    visible: true,
  },
];

// Get all styles
const paintStyles = figma.getLocalPaintStyles();
const textStyles = figma.getLocalTextStyles();
const effectStyles = figma.getLocalEffectStyles();
```

### 4. Components

```javascript
// Create component
const component = figma.createComponent();
component.name = "Button/Primary";

// Create instance
const instance = component.createInstance();

// Get all components
const components = figma.root.findAll((node) => node.type === "COMPONENT");
```

### 5. Variables (Design Tokens - Modern)

```javascript
// Create variable collection
const collection = figma.variables.createVariableCollection("Colors");

// Create variable
const variable = figma.variables.createVariable(
  "primary-500",
  collection,
  "COLOR"
);

// Set value for mode
variable.setValueForMode(collection.modes[0].modeId, {
  r: 0,
  g: 0,
  b: 1,
});
```

---

## Plugin Lifecycle

### Initialization

```javascript
// code.js
figma.showUI(__html__, { width: 400, height: 600 });

// Load data
const data = await figma.clientStorage.getAsync("designSystem");

// Send to UI
figma.ui.postMessage({ type: "init", data });
```

### Closing

```javascript
// Close plugin
figma.closePlugin();

// Close with message
figma.closePlugin("Design system updated!");
```

### Storage

```javascript
// Save data (async)
await figma.clientStorage.setAsync("key", value);

// Load data
const value = await figma.clientStorage.getAsync("key");

// Delete data
await figma.clientStorage.deleteAsync("key");
```

---

## Best Practices for Design System Plugin

### 1. Organization

- Group styles by category (Colors, Typography, Spacing, etc.)
- Use naming conventions: `Category/Subcategory/Name`
- Example: `Color/Primary/500`, `Typography/Heading/H1`

### 2. Performance

- Batch operations when possible
- Use `figma.skipInvisibleInstanceChildren = true` for large files
- Avoid unnecessary traversals
- Cache frequently accessed data

### 3. Error Handling

```javascript
try {
  // Figma API operations
} catch (error) {
  figma.notify("Error: " + error.message, { error: true });
  console.error(error);
}
```

### 4. User Feedback

```javascript
// Success notification
figma.notify("✓ Design system updated");

// Error notification
figma.notify("✗ Failed to update", { error: true });

// Timeout (default 3s)
figma.notify("Processing...", { timeout: 5000 });
```

### 5. Async Operations

```javascript
// Always use async/await for Figma operations
async function loadFont() {
  await figma.loadFontAsync({ family: "Inter", style: "Regular" });
}

// Handle fonts before text operations
await figma.loadFontAsync(textNode.fontName);
textNode.characters = "New text";
```

---

## Common Patterns for Design System

### Pattern 1: Export Design Tokens

```javascript
function exportTokens() {
  const tokens = {
    colors: {},
    typography: {},
    spacing: {},
  };

  // Export paint styles
  figma.getLocalPaintStyles().forEach((style) => {
    const paint = style.paints[0];
    if (paint.type === "SOLID") {
      tokens.colors[style.name] = rgbToHex(paint.color);
    }
  });

  return tokens;
}
```

### Pattern 2: Import Design Tokens

```javascript
async function importTokens(tokens) {
  // Create or update paint styles
  for (const [name, value] of Object.entries(tokens.colors)) {
    let style = figma.getLocalPaintStyles().find((s) => s.name === name);

    if (!style) {
      style = figma.createPaintStyle();
      style.name = name;
    }

    style.paints = [
      {
        type: "SOLID",
        color: hexToRgb(value),
      },
    ];
  }
}
```

### Pattern 3: Apply Styles to Selection

```javascript
function applyStyleToSelection(styleId) {
  const selection = figma.currentPage.selection;

  selection.forEach((node) => {
    if ("fillStyleId" in node) {
      node.fillStyleId = styleId;
    }
  });

  figma.notify(`Applied to ${selection.length} layers`);
}
```

### Pattern 4: Sync Components

```javascript
function syncComponentInstances(componentId) {
  const component = figma.getNodeById(componentId);
  if (!component || component.type !== "COMPONENT") return;

  // Find all instances
  const instances = figma.root.findAll(
    (node) => node.type === "INSTANCE" && node.mainComponent?.id === componentId
  );

  // Update instances
  instances.forEach((instance) => {
    // Sync properties
  });
}
```

---

## UI Development Guidelines

### 1. Figma Design System

Use Figma's design tokens for consistency:

```css
:root {
  --color-bg: #ffffff;
  --color-bg-secondary: #f5f5f5;
  --color-text: #000000;
  --color-text-secondary: #666666;
  --color-border: #e5e5e5;
  --color-primary: #0d99ff;
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --radius: 4px;
}
```

### 2. Responsive Layout

```css
body {
  margin: 0;
  padding: 16px;
  font-family: "Inter", sans-serif;
  font-size: 12px;
}

/* Scrollable content */
.content {
  max-height: calc(100vh - 80px);
  overflow-y: auto;
}
```

### 3. Accessibility

- Use semantic HTML
- Provide keyboard navigation
- Add ARIA labels
- Ensure sufficient contrast

---

## Development Workflow

### 1. Setup

```bash
npm init -y
npm install --save-dev typescript @figma/plugin-typings
```

### 2. TypeScript Config (tsconfig.json)

```json
{
  "compilerOptions": {
    "target": "ES6",
    "module": "commonjs",
    "lib": ["ES2015"],
    "outDir": "./dist",
    "typeRoots": ["./node_modules/@types", "./node_modules/@figma"]
  }
}
```

### 3. Build Script (package.json)

```json
{
  "scripts": {
    "build": "tsc",
    "watch": "tsc --watch"
  }
}
```

### 4. Testing

- Test in Figma desktop app
- Use `console.log()` for debugging
- Check DevTools (Plugins > Development > Open Console)

---

## Security & Limitations

### Sandbox Restrictions

- No access to file system
- No Node.js modules
- No external HTTP requests (unless networkAccess enabled)
- Limited to Figma Plugin API

### Data Storage

- Use `figma.clientStorage` for plugin data
- Max 10MB per plugin
- Data persists across sessions
- Async operations only

### Network Access

- Requires manifest permission
- Specify allowed domains
- Use for API integrations only

---

## Common Pitfalls

1. **Forgetting to load fonts** before text operations
2. **Not handling async operations** properly
3. **Accessing DOM in code.js** (use UI instead)
4. **Not bundling external dependencies**
5. **Exceeding storage limits**
6. **Not handling errors gracefully**

---

## Resources

- [Figma Plugin API Docs](https://www.figma.com/plugin-docs/)
- [Plugin Samples](https://github.com/figma/plugin-samples)
- [Community Forum](https://forum.figma.com/c/plugin-api/)

---

## Development Checklist

Before implementing any feature:

- [ ] Read this guide
- [ ] Understand sandbox vs UI context
- [ ] Plan message passing structure
- [ ] Handle errors appropriately
- [ ] Test with real Figma files
- [ ] Optimize performance
- [ ] Add user feedback
- [ ] Document code clearly

---

**Last Updated**: December 2025
**Plugin Architecture**: Sandbox + UI (iframe)
**Target**: Professional design system management
