---
name: figma-component-audit
description: Accepts a Figma component or component set URL, audits variant structure, component properties, and AI-readiness across 5 dimensions, and outputs an improvement report.
argument-hint: <figma-component-url>
allowed-tools: mcp__plugin_figma_figma__get_design_context, mcp__plugin_figma_figma__get_metadata, mcp__plugin_figma_figma__get_variable_defs
---

# Figma Component Structure Audit

Analyze the Figma component URL provided in `$ARGUMENTS` and evaluate whether the component is structured correctly for variant management, code generation, and AI implementation accuracy.

**Output language:** Respond in the same language the user is using in this conversation.

## Step 1: Parse the URL

Extract from `$ARGUMENTS`:

- `figma.com/design/:fileKey/...?node-id=X-Y` → `fileKey` and `nodeId` (convert `X-Y` to `X:Y`)

## Step 2: Fetch data

Call `get_design_context`, `get_metadata`, and `get_variable_defs` **in parallel**:

- `get_design_context`: retrieves generated code, screenshot, and style info
- `get_metadata`: retrieves layer hierarchy, node types (`COMPONENT`, `COMPONENT_SET`, `VARIANT`), and component properties
- `get_variable_defs`: retrieves variable and token definitions used in the file

Parameters for `get_design_context`:
- `disableCodeConnect: true`
- `clientLanguages: "unknown"`
- `clientFrameworks: "unknown"`

## Step 3: Detect component type

From `get_metadata`, determine the root node type:
- `COMPONENT_SET` — a component with multiple variants defined
- `COMPONENT` — a single component with no variants
- `FRAME` / other — the URL likely points to a frame containing components, not the component itself

If the root node is `FRAME` or not a component, inform the user that this looks like a frame URL, not a component URL, and suggest navigating to the component definition in Figma.

## Step 4: Analyze across 5 dimensions

### Dimension A: Variant definition and naming

Applies when root node is `COMPONENT_SET`.

**Good state criteria:**
- Variant properties use clear, consistent names (e.g. `State`, `Size`, `Type`)
- Variant values are concise and PascalCase or lowercase (e.g. `Default`, `Hover`, `Pressed`, `Disabled`)
- State variants cover expected interactive states: at minimum `Default` + `Hover` + `Pressed` + `Disabled`
- Size variants exist where the component has responsive sizing needs (e.g. `Small`, `Medium`, `Large`)
- No redundant variants that duplicate structure rather than style

**Problem patterns (checklist):**
- [ ] Variant property names are ambiguous or inconsistent (e.g. `v1`, `Option`, `Type 1`)
- [ ] Missing standard states — no `Disabled`, `Focus`, or `Error` variants for an interactive component
- [ ] Variant values use inconsistent casing (`default` mixed with `Default`)
- [ ] Different variants have structurally divergent layer trees (layer count or nesting mismatch between variants)
- [ ] Variants exist that are duplicates of others with no visible or functional difference

### Dimension B: Component properties

**Good state criteria:**
- Text content is exposed as a component property (text prop), not hardcoded
- Icon or sub-component slots use Instance Swap properties
- Toggle-able elements (e.g. leading icon, badge) use Boolean properties
- Property names are clear and map to what a developer would name them (e.g. `label`, `icon`, `disabled`, `withBadge`)

**Slot detection:**
Slots appear in `get_design_context` as `React.ReactNode` props (e.g. `slot2?: React.ReactNode | null`), and as layers named "Slot N" in the design. When slots are present, check:
- Slot names are semantic (e.g. `headerSlot`, `contentSlot`, `footerSlot`) — not generic sequential names like `Slot 1`, `Slot 2`
- Slots that logically need a fallback have a meaningful default component set (not `null`)
- Slot container layers have an explicit size (height/width) so the layout doesn't collapse when empty
- The number of slots is appropriate — excessive slots may indicate the component should be split

**Problem patterns (checklist):**
- [ ] No component properties defined — all customization requires detaching or editing instances
- [ ] Text props missing for visible text nodes that callers would want to customize
- [ ] Boolean props missing for elements that are sometimes shown / sometimes hidden
- [ ] Instance Swap props missing for icon or avatar slots
- [ ] Property names are Figma-internal or cryptic (e.g. `Property 1`, `Frame 7`)
- [ ] Slots present but named generically (`Slot 1`, `Slot 2`) with no semantic role conveyed
- [ ] Slots that should have default content are set to `null` (empty by default with no fallback)
- [ ] Slot containers have no explicit size — layout will collapse when the slot is empty

### Dimension C: Layer naming and structure consistency

**Good state criteria:**
- All layers have meaningful role-based names (`Icon`, `Label`, `Badge`, `Container`)
- Layer tree is identical (or near-identical) across all variants — only style properties differ
- No auto-generated names remain (`Frame 123`, `Group 4`)

**Problem patterns (checklist):**
- [ ] Auto-generated layer names present inside component layers
- [ ] Layer structure differs between variants (different number of children or nesting depth)
- [ ] Layers with the same name appear as siblings in the same parent
- [ ] Hidden layers left in place instead of using Boolean props to toggle visibility

### Dimension D: Auto-layout and spacing tokens

**Good state criteria:**
- Auto-layout applied to all horizontal and vertical arrangements
- Padding and gap use spacing tokens (generated code shows `var(--spacing-*)` or similar)
- Fixed sizes only where explicitly needed (e.g. icon width)

**Problem patterns (checklist):**
- [ ] Children are absolutely positioned (no auto-layout on the container)
- [ ] Spacer frames used instead of gap properties
- [ ] Hardcoded px padding/gap with no token reference

### Dimension E: Color and typography tokens

**Good state criteria:**
- Fill colors reference Figma variables (generated code uses `var(--color-*)`)
- Text styles are applied (font, size, weight defined as a text style — not one-off)
- State-specific colors (e.g. hover fill, disabled text) use semantic tokens, not hardcoded hex overrides

**Problem patterns (checklist):**
- [ ] Hardcoded hex fills with no variable reference (no `var(--*)` in generated code)
- [ ] Text nodes have one-off typography (no named text style applied)
- [ ] Variant-specific color changes use raw hex instead of state-based semantic tokens

## Step 5: Output the report

Use the following format:

---

## Figma Component Audit Report

**File:** [file name]  
**Component:** [component name] (`nodeId`)  
**Type:** Component Set (N variants) / Single Component  
**Screenshot:** [display the screenshot]

---

### Overall score: X / 25

| Dimension | Score | Rating |
|-----------|-------|--------|
| A. Variant definition & naming | X / 5 | 🟢 Good / 🟡 Room for improvement / 🔴 Needs fixing |
| B. Component properties | X / 5 | 〃 |
| C. Layer naming & structure | X / 5 | 〃 |
| D. Auto-layout & spacing | X / 5 | 〃 |
| E. Color & typography tokens | X / 5 | 〃 |

Scoring criteria (5 points each):
- 5: No issues
- 3–4: Minor issues (1–2 items)
- 1–2: Multiple issues (3+ items)
- 0: Almost entirely unaddressed

---

### Variant map

If the component is a Component Set, list all variant combinations in a compact table:

| State | Size | Type | … |
|-------|------|------|---|
| Default | Medium | Primary | … |
| Hover | Medium | Primary | … |
| … | … | … | … |

Note any missing combinations (e.g. "No `Disabled` variant for `Secondary` type").

---

### Strengths

(List practices in this component that are well done and worth replicating)

---

### Improvement suggestions

For each issue found:

#### [Dimension / specific location]
- **Problem:** What is wrong
- **Impact:** How it hinders AI code generation or developer handoff
- **Suggestion:** Specific action to take in Figma (e.g. "Add a Boolean property `showIcon` to the top-level component and link the icon layer's visibility to it")

---

### Summary

(1–2 sentences on the overall component quality and the single most impactful improvement to make)

---

## Error handling

- If fileKey or nodeId cannot be extracted: ask the user for a valid Figma URL
- If the node is not a component or component set: inform the user and explain how to find the component's direct URL in Figma (right-click the component → Copy link to selection)
- If `get_design_context` or `get_metadata` returns an error: report it and ask the user to verify file access permissions
