---
name: pipeline-design-gate
description: Paper design exploration for new UI components. Creates 3 design options per component for user selection.
disable-model-invocation: true
user-invocable: false
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, AskUserQuestion, mcp__paper__get_basic_info, mcp__paper__get_selection, mcp__paper__get_node_info, mcp__paper__get_children, mcp__paper__get_screenshot, mcp__paper__get_jsx, mcp__paper__get_tree_summary, mcp__paper__get_computed_styles, mcp__paper__get_fill_image, mcp__paper__get_font_family_info, mcp__paper__write_html, mcp__paper__create_artboard, mcp__paper__delete_nodes, mcp__paper__set_text_content, mcp__paper__rename_nodes, mcp__paper__update_styles, mcp__paper__duplicate_nodes, mcp__paper__finish_working_on_nodes, mcp__paper__get_guide
---

# Pipeline Design Gate

Create 3 design options per new UI component using Paper. User picks one; chosen design is exported for implementation.

**Input:** `$ARGUMENTS` = path to the plan file containing tagged tasks

## Process

### 1. Extract ui-new tasks

Read the plan file at `$ARGUMENTS`. Find all tasks tagged with `<!-- tag: ui-new -->`.

For each, extract:
- Task number (e.g., "Task 3")
- Component name
- Description and requirements (layout, data displayed, interactions)

If no `<!-- tag: ui-new -->` tasks found, report "No new UI components to design" and exit.

### 2. Load design context

**Check for Brand.md:**
- If exists in project root → use its color palette, typography, spacing tokens
- If not → generate a design brief from project context:
  - Read CLAUDE.md for project description
  - Read existing components in `docs/COMPONENTS.md` for style consistency

**Check Paper availability:**
- Call `mcp__paper__get_basic_info`

**If Paper MCP is not available** (tool calls fail):
- Report "Paper MCP not available — skipping design gate"
- Write a text-only design brief to `.gsd/designs/task-{N}-{name}.md` instead (color palette, layout description, component hierarchy in prose)
- Exit gracefully — the pipeline continues without visual designs

### 3. For each ui-new task

#### a. Create 3 design options in Paper

Use Paper to create 3 meaningfully different design options for the component. For each option, create an artboard named "[Component] — Option A/B/C".

The 3 options should be tangibly different — not just color swaps:
- **Option A:** Clean, balanced baseline
- **Option B:** Different information hierarchy, density, or composition
- **Option C:** Bolder or more distinctive take — different typography scale, asymmetric layout, stronger visual personality

Paper's system prompt handles all design quality standards, incremental building, typography, spacing, and review checkpoints. Just describe what the component should do and what content it displays — Paper handles the craft.

#### b. Present to user

Use `AskUserQuestion`:
- Question: "Which design do you prefer for [component name]?"
- Header: "Design"
- Options:
  - "Option A" with description of its approach
  - "Option B" with description of its approach
  - "Option C" with description of its approach

Take a screenshot of each option before asking so the user can see them on the canvas.

#### c. Export chosen design

For the chosen option:

1. `mcp__paper__get_jsx` with `format: "tailwind"` on the chosen artboard
2. `mcp__paper__get_computed_styles` for key nodes (container, text, buttons, etc.)
3. Write to `.gsd/designs/task-{number}-{slugified-name}.md`:

````markdown
# [Component Name]

Task: [task number from plan]
Chosen: Option [A/B/C]
Generated: [date]

## Description

[What this component does and why this option was chosen]

## JSX

```jsx
[exported JSX from get_jsx]
```

## Style Tokens

- **Primary color:** #...
- **Background:** #...
- **Text color:** #...
- **Font family:** ...
- **Font sizes:** heading Xpx, body Ypx, caption Zpx
- **Spacing:** section Xpx, group Ypx, element Zpx
- **Border radius:** Xpx
- **Container width:** Xpx

## Design Notes

[What makes this option work — layout decisions, hierarchy, visual personality]
````

#### d. Clean up

Call `mcp__paper__finish_working_on_nodes` to release the working indicator.

### 4. Report

For each designed component:
- Component name
- Chosen option
- Export path

If pipeline state exists (`.claude/pipeline-state.json`), read it and update with design artifact paths:
```json
{
  "designArtifacts": {
    "task-3": ".gsd/designs/task-3-user-profile-card.md"
  }
}
```

## Done

Report which components were designed and where exports were saved. The caller handles the next phase.
