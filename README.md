# figma-component-audit — Figma Component Structure Audit Skill

A Claude Code skill that evaluates whether a Figma component is structured correctly for variant management, developer handoff, and AI code generation. Pass a component URL and get a scored report across 5 dimensions, with concrete suggestions to improve the component before implementation.

---

## What this is

Figma components look fine on the canvas, but what matters for AI code generation is the structure beneath — variant naming, component properties, slot definitions, layer hierarchy, and token usage. A component with generic slot names, hardcoded spacing, and missing interactive states will produce ambiguous code even if it renders correctly in Figma.

This skill makes those structural issues visible. It runs three Figma API calls in parallel (`get_design_context`, `get_metadata`, `get_variable_defs`), analyzes the result across 5 dimensions, and returns a 25-point report with specific, actionable fixes.

```
Figma Component URL  →  figma-component-audit  →  Component score (X / 25)
                                               →  Variant map
                                               →  Dimension breakdown
                                               →  Improvement suggestions per issue
```

---

## The 5 dimensions

| # | Dimension | What is checked |
|---|-----------|-----------------|
| A | **Variant definition & naming** | Clear property names, consistent casing, coverage of standard interactive states (Default / Hover / Pressed / Disabled) |
| B | **Component properties & slots** | Text props, Boolean props, Instance Swap props, slot names, slot defaults, slot container sizing |
| C | **Layer naming & structure** | Role-based names, consistent layer tree across variants, no auto-generated names |
| D | **Auto-layout & spacing** | Auto-layout applied, spacing via tokens (`var(--spacing-*)`) rather than hardcoded px |
| E | **Color & typography tokens** | `var(--color-*)` fills, named text styles, semantic state colors |

Each dimension is scored 0–5. Total: **25 points**.

---

## Slot detection

This skill automatically detects Figma's **slot** feature. Slots appear in the generated code as `React.ReactNode` props — for example:

```tsx
type ModalProps = {
  headerSlot?: React.ReactNode | null;
  contentSlot?: React.ReactNode | null;
  footerSlot?: React.ReactNode | null;
};
```

When slots are found, the skill checks:
- **Names** — are slot names semantic (`headerSlot`) or generic (`slot2`, `slot3`)?
- **Defaults** — do slots that need a fallback have a default component, or are they `null`?
- **Sizing** — do slot containers have an explicit size so the layout doesn't collapse when empty?

Slot checks are included in Dimension B.

---

## Repo structure

```
skills/
  figma-component-audit/
    SKILL.md          — skill definition loaded by Claude Code
```

---

## Getting started

**1. Clone this repo**

```bash
git clone https://github.com/gaspanik/figma-component-audit-skill
```

**2. Install the skill into Claude Code**

Copy the skill directory into your Claude Code skills folder:

```bash
cp -r skills/figma-component-audit ~/.claude/skills/
```

**3. Verify the Figma MCP is connected**

This skill requires the official [Figma MCP server](https://github.com/figma/mcp-server-guide). Confirm it is connected in your Claude Code settings before running the skill.

**4. Run the audit**

Right-click the component in Figma → **Copy link to selection**, then pass the URL to the skill:

```
/figma-component-audit https://www.figma.com/design/<fileKey>/...?node-id=1-2
```

Natural language also works:

```
このFigmaコンポーネントをチェックして: https://www.figma.com/design/...
```

```
Can you audit this Figma component? https://www.figma.com/design/...
```

> **Note:** The URL must point to a component or component set, not a frame. If you pass a frame URL, the skill will detect this and tell you how to find the correct URL.

The report is written in whatever language you are using in the conversation — no configuration needed.

---

## Example output

```
## Figma Component Audit Report

Overall score: 18 / 25

| Dimension                        | Score | Rating |
|----------------------------------|-------|--------|
| A. Variant definition & naming   | 4 / 5 | 🟢 Good |
| B. Component properties & slots  | 3 / 5 | 🟡 Room for improvement |
| C. Layer naming & structure      | 4 / 5 | 🟢 Good |
| D. Auto-layout & spacing         | 3 / 5 | 🟡 Room for improvement |
| E. Color & typography tokens     | 4 / 5 | 🟢 Good |

### Variant map
| State   | Size   |
|---------|--------|
| Default | Medium |
| Hover   | Medium |
| Pressed | Medium |
Missing: Disabled variant

### Improvement suggestions
...
```

---

## Agent settings

- Use the most capable Claude model available. The skill calls three Figma API tools in parallel and synthesizes a multi-section report — a stronger model produces more precise layer-level suggestions.
- The `SKILL.md` format and `allowed-tools` frontmatter are Claude Code-specific. To use this with another agent (Cursor, Windsurf, etc.), copy the prompt body from `SKILL.md` into that agent's rule format — the audit logic itself is plain markdown and transfers without modification.

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| `fileKey or nodeId could not be extracted` | Make sure the URL contains `?node-id=X-Y`. File-root URLs are not supported. |
| `This looks like a frame URL, not a component` | Right-click the component in Figma → Copy link to selection → use that URL. |
| `get_design_context returned an error` | Check that your Figma MCP is connected and has access to the file. |
| Report is in the wrong language | The skill mirrors the conversation language. Switch your message language and re-run. |

---

## Related

- [figma-audit-skill](https://github.com/gaspanik/figma-audit-skill) — the companion skill for auditing **frames** (page-level layouts) rather than components

---

Built by Masaaki Komori · [@cipher](https://x.com/cipher) · Skill for [Claude Code](https://claude.ai/code) + [Figma MCP](https://github.com/figma/mcp-server-guide)
