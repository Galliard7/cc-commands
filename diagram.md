---
name: diagram
description: Generate a standalone Mermaid diagram HTML file and open it in the browser. Usage: /diagram <description>
---

# /diagram Command

Generate a standalone, self-contained Mermaid diagram HTML file from a description. Opens automatically in the browser.

## Usage

`/diagram <description>` — e.g., `/diagram microservices architecture for the lending platform`

The argument after `/diagram` describes what to diagram. If no argument is provided, ask the user what they want to diagram.

## Behavior

1. **Analyze the description** and determine the best Mermaid diagram type:
   - `flowchart` / `graph` — processes, workflows, decision trees
   - `sequenceDiagram` — API calls, service interactions, protocols
   - `classDiagram` — object models, data structures
   - `stateDiagram-v2` — state machines, lifecycles
   - `erDiagram` — database schemas, entity relationships
   - `gantt` — timelines, project schedules
   - `pie` — proportions, breakdowns
   - `mindmap` — brainstorming, topic exploration
   - `timeline` — chronological events, history
   - `C4Context` / `C4Container` — system architecture (C4 model)
   - `quadrantChart` — 2x2 matrices, prioritization
   - `sankey-beta` — flow quantities, resource allocation
   - `block-beta` — block diagrams, system layouts
   - `xychart-beta` — simple charts

2. **Generate the Mermaid syntax** — clean, well-structured, with descriptive node labels. Prefer readability over density. Use subgraphs for logical grouping when the diagram has 8+ nodes.

3. **Wrap in a self-contained HTML file** using this template:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{Diagram Title}</title>
    <script src="https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.min.js"></script>
    <style>
        body {
            margin: 0; padding: 2rem;
            background: #0a0f1c; color: #e2e8f0;
            font-family: system-ui, -apple-system, sans-serif;
            display: flex; flex-direction: column;
            align-items: center; min-height: 100vh;
        }
        h1 { font-size: 1.5rem; margin-bottom: 1.5rem; font-weight: 500; }
        .mermaid { max-width: 100%; overflow-x: auto; }
    </style>
</head>
<body>
    <h1>{Diagram Title}</h1>
    <pre class="mermaid">
{mermaid syntax here}
    </pre>
    <script>
        mermaid.initialize({
            startOnLoad: true,
            theme: 'dark'
        });
    </script>
</body>
</html>
```

4. **Save** to the current working directory as `{slug}-diagram.html` (kebab-case slug from the title).

5. **Open** in browser with `open {filename}.html`.

6. **Tell the user** the file path and a brief description of what was diagrammed.

## Guidelines

- Keep diagrams focused — one concept per diagram. If the topic is broad, pick the most useful view and mention what else could be diagrammed.
- Use clear, human-readable labels (not abbreviations or IDs).
- For flowcharts, use meaningful shapes: `[rectangles]` for processes, `{diamonds}` for decisions, `([stadiums])` for start/end, `[(cylinders)]` for databases.
- For sequence diagrams, use activation boxes and alt/opt/loop blocks when they add clarity.
- Limit to ~15-20 nodes max per diagram for readability. Split into multiple diagrams if needed.
- The dark theme is default. If the user asks for light, change body background to `#ffffff`, color to `#1a202c`, and mermaid theme to `'default'`.
