# Diagrams

Source files (`.mmd`, Mermaid syntax) live alongside their SVG renderings.
To regenerate: edit the `.mmd` source, paste into
[mermaid.live](https://mermaid.live), export as SVG, replace the existing
file.

## Contents

| File | Description |
|---|---|
| `architecture.mmd` / `.svg` | System components, data plane, external services, trust boundaries |
| `auth-flow.mmd` / `.svg` | Signup, login, and authenticated database-access flows |
| `data-model.mmd` / `.svg` | Core entity-relationship model with Row-Level Security tenant keys |

## Style conventions

- Boxes, arrows, and labels. No gradients, shadows, or decorative styling.
- Every arrow is labeled with the data type or action that flows along it.
- Trust boundaries are marked with dashed lines.
- Light-mode rendering by default; GitHub displays SVGs against white.
