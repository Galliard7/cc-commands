---
name: arrows
description: "Create or update an Excalidraw diagram in the MC Arrows tab. Usage: /arrows <description>"
---

# /arrows — Push Diagrams to Mission Control Arrows

Create Excalidraw diagrams and push them to the Arrows tab in Mission Control, where the user can view, edit, and iterate on them.

## Usage

- `/arrows <description>` — create a new diagram from a description
- `/arrows update <description of changes>` — update the most recently pushed diagram in this session

The argument describes what to diagram. If no argument is provided, ask the user what they want to diagram.

## Workflow

### Creating a new diagram

1. **Analyze the description** and determine the best layout approach (flowchart, architecture, sequence, mind map, etc.)

2. **Generate Excalidraw elements** as a JSON array following the format reference below. Focus on clarity and readability.

3. **Push to Arrows** by writing the elements to a temp file and running:
   ```bash
   python3 ~/skill-backends/arrows/arrows-push.py /tmp/arrows-draft.json --name "<diagram name>"
   ```
   The script accepts MCP-style elements (with `label` shorthand) and converts them to native Excalidraw format automatically.

4. **Report to the user**: tell them the diagram name and that it's ready in the Arrows tab. Stay on the line for feedback.

5. **Iterate**: when the user gives feedback, generate updated elements and push with `--update <id>`:
   ```bash
   python3 ~/skill-backends/arrows/arrows-push.py /tmp/arrows-draft.json --update <diagram-id> --name "<name>"
   ```

### Key rules

- **Strip pseudo-elements**: the push script handles this automatically (cameraUpdate, delete, restoreCheckpoint are removed)
- **Label shorthand works**: `"label": {"text": "Hello", "fontSize": 20}` on shapes is expanded by the script
- **Track the diagram ID** returned from the create call — you need it for updates
- **Stay in the loop**: after pushing, wait for the user's feedback. Don't close out the interaction.

## Color Palette (MANDATORY)

Use the **Orange + Gray** palette for ALL diagrams. Do not deviate unless the user explicitly asks for different colors.

### Default colors

| Element | Color | Hex |
|---------|-------|-----|
| **Strokes** (all shapes, arrows) | Orange | `#f08c00` |
| **Text** (all labels, titles) | Orange | `#f08c00` |
| **Shape fill** (default) | Gray | `#3a3a3a` |
| **Font** | Normal (Helvetica) | `fontFamily: 2` |

### Semantic overrides — use ONLY when the diagram content warrants it

| Meaning | Fill color | Hex | When to use |
|---------|-----------|-----|-------------|
| **Success / complete / done** | Green | `#1a3d24` | Nodes representing completed steps, successful outcomes, or "good" states |
| **Error / blocker / warning** | Red | `#4a1a1a` | Nodes representing failures, blockers, risks, or "watch out" states |

**Strokes and text stay orange (`#f08c00`) even on green/red nodes.** Only the fill changes.

### Example element with default colors
```json
{"type": "rectangle", "id": "r1", "x": 100, "y": 100, "width": 180, "height": 70,
 "roundness": {"type": 3}, "backgroundColor": "#3a3a3a", "fillStyle": "solid",
 "strokeColor": "#f08c00", "strokeWidth": 2, "label": {"text": "Step 1", "fontSize": 20}}
```

### Example success node
```json
{"type": "rectangle", "id": "r2", "x": 100, "y": 200, "width": 180, "height": 70,
 "roundness": {"type": 3}, "backgroundColor": "#1a3d24", "fillStyle": "solid",
 "strokeColor": "#f08c00", "strokeWidth": 2, "label": {"text": "Done", "fontSize": 20}}
```

### Example error/blocker node
```json
{"type": "rectangle", "id": "r3", "x": 100, "y": 300, "width": 180, "height": 70,
 "roundness": {"type": 3}, "backgroundColor": "#4a1a1a", "fillStyle": "solid",
 "strokeColor": "#f08c00", "strokeWidth": 2, "label": {"text": "Blocked", "fontSize": 20}}
```

## Element Format Reference

### Required Fields (all elements)
`type`, `id` (unique string), `x`, `y`, `width`, `height`

### Element Types

**Rectangle**: `{"type": "rectangle", "id": "r1", "x": 100, "y": 100, "width": 200, "height": 100}`
- `roundness: {"type": 3}` for rounded corners
- `backgroundColor: "#3a3a3a"`, `fillStyle: "solid"` for filled

**Ellipse**: `{"type": "ellipse", "id": "e1", "x": 100, "y": 100, "width": 150, "height": 150}`

**Diamond**: `{"type": "diamond", "id": "d1", "x": 100, "y": 100, "width": 150, "height": 150}`

**Labeled shape (preferred)**: `{"type": "rectangle", "id": "r1", "x": 100, "y": 100, "width": 200, "height": 80, "label": {"text": "Hello", "fontSize": 20}}`
- Works on rectangle, ellipse, diamond — auto-centered, saves tokens

**Standalone text**: `{"type": "text", "id": "t1", "x": 150, "y": 138, "text": "Hello", "fontSize": 24, "fontFamily": 2}`

**Arrow**: `{"type": "arrow", "id": "a1", "x": 300, "y": 150, "width": 200, "height": 0, "points": [[0,0],[200,0]], "endArrowhead": "arrow"}`
- `startBinding: {"elementId": "r1", "fixedPoint": [1, 0.5]}` — right edge
- `endBinding: {"elementId": "r2", "fixedPoint": [0, 0.5]}` — left edge
- fixedPoint: top=[0.5,0], bottom=[0.5,1], left=[0,0.5], right=[1,0.5]
- `label: {"text": "connects"}` for labeled arrows

## Layout & Legibility (CRITICAL)

The diagram must be clean, legible, and visually organized. No tangled messes. Follow these rules strictly:

**Text sizing — non-negotiable minimums:**
- Title text: fontSize **28+**
- Node/shape labels: fontSize **20+**
- Arrow labels / annotations: fontSize **16+**
- NEVER go below 16 for any text. If a label doesn't fit, make the shape bigger — don't shrink the text.

**Shape sizing:**
- Minimum shape size: **180×70** for labeled rectangles/ellipses
- Size shapes to fit their labels comfortably — at least 30px padding on each side of the text
- All shapes of the same "type" in a diagram should be the same size (e.g., all process steps same width/height)

**Spacing & flow:**
- Minimum **60px gaps** between shapes (more is better)
- Minimum **80px** between rows/columns of shapes
- Arrows need clear runway — don't start/end arrows right at shape edges with no breathing room
- Arrow paths should not cross other shapes. If they must cross, route them around.

**Layout strategy — plan before placing:**
1. **Pick a primary flow direction** — left-to-right or top-to-bottom. Stick to one.
2. **Align shapes to a grid** — shapes in the same row share the same y; shapes in the same column share the same x. No random diagonal placement.
3. **Group related elements** visually — use consistent spacing within groups, larger spacing between groups
4. **Hierarchy matters** — inputs/starts on the left (or top), outputs/ends on the right (or bottom)
5. **Limit to 12-15 nodes max** per diagram. If the topic needs more, split into focused sub-diagrams or simplify. Density kills readability.
6. **Arrows should flow with the layout direction** — mostly left-to-right or top-to-bottom. Avoid backtracking arrows unless they represent feedback loops (and label them clearly).

**Common mistakes to avoid:**
- Shapes jammed together with tiny gaps — space them out generously
- Labels that overflow their shapes or get cut off — always size the shape to the label, not the other way around
- Arrows that zigzag through the middle of the diagram — route cleanly along edges
- Inconsistent shape sizes for same-level nodes — align and standardize
- Placing the title on top of or overlapping the diagram content

## Input format

The push script accepts either:
- A JSON **array** of elements (wraps in a scene automatically)
- A full **scene object** `{"elements": [...], "appState": {...}, "files": {}}`

Both work. The array form is simpler for generation.
