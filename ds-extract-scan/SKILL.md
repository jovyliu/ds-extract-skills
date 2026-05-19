---
name: ds-extract-scan
description: Scan one or more Figma source screens and produce a lean **component candidate list** — a flat, deduplicated table of what could become a DS component, each with a suggested DS name and confidence level. Use whenever the user wants to extract a design system from existing screens, asks to "scan", "review", "看一下這幾個畫面", "幫我看哪些可以抽元件", or shares Figma URLs and mentions building/extracting a design system. This is Stage 1 of a 4-stage extract workflow (scan → consolidate → build → rebind). Always use this skill — never try to extract a design system without scanning first.
---

# ds-extract-scan

Stage 1 of the design-system extract workflow. Produces a **lean component candidate list** across all scanned screens: a single flat table of distinct candidates, each with a suggested DS name and confidence level.

This skill is **read-only**. Never call `mcp__figma__use_figma` or any write tool.

**Design philosophy**: scan is opinionated, not exhaustive. The user is a designer who will review and adjust — give them a working draft to react to, not a neutral inventory. Cross-screen deduplication happens here, not in Stage 2.

## Output language

**All user-facing output must be in Traditional Chinese (繁體中文).**

Internal reasoning stays in English. Category labels (`button-shaped`) and DS names (`Button/Primary`) stay in English — they become Figma layer names.

## When this skill triggers

Use this skill when:
- The user shares one or more Figma source screen URLs and mentions extracting / building a design system
- The user says "scan", "review", "看畫面", "幫我看可以抽什麼元件"

Do **not** use this skill when:
- The user asks to audit an existing design system (use `ds-checkup`)
- The user already has a Master Build Plan and wants to build (use `ds-extract-build`)

## The 4-stage workflow

```
[Stage 1] ds-extract-scan        ← you are here
[Stage 2] ds-extract-consolidate (optional — skip if candidate list is small & clear)
[Stage 3] ds-extract-build
[Stage 4] ds-rebind
```

After scanning, suggest `ds-extract-consolidate` only if the user wants a formal Master Plan. If the candidate list is small (<10 candidates) and clear, the user can skip straight to `ds-extract-build`.

## Inputs

The user provides one or more Figma URLs pointing to source screens (frames). If the URL points to a whole page or file, ask which screens to scan rather than scanning everything.

## Workflow

### Step 0: Verify scope

- Count screens. If >10, warn and suggest batching (5 at a time).
- If exactly 1 screen, mention consolidation is probably unnecessary.

### Step 1: Get metadata per screen (cheap)

For each screen, call `mcp__figma__get_metadata`. This returns structure (frame names, child counts, hierarchy) without expensive detail.

**Critical — node type handling:**
- `<instance>` nodes → **skip entirely. Do not mention in output. Do not flag as "incomplete" or "needs variant".** Assume these are intentional imports from another library (iOS UI Kit, internal library, etc.). Stage 1 does not second-guess existing instances.
- `<symbol>` nodes → existing component definitions, skip
- `<frame>` / `<text>` / `<vector>` / `<ellipse>` / `<rectangle>` → raw, candidate material

### Step 2: Get screenshot per screen

Call `mcp__figma__get_screenshot` for each screen. This is the highest-value signal — far more useful than dumping `get_design_context` for every node.

### Step 3: Targeted detail (only if needed)

Call `mcp__figma__get_design_context` **only** when:
- A region is visually ambiguous in the screenshot
- You need to confirm something is raw vs an instance

**Hard limit: max 2 `get_design_context` calls per screen.** If you feel you need more, ask the user instead of burning tokens.

### Step 4: Build the candidate list (cross-screen dedup)

This is the core step. Instead of producing one inventory per screen, build a **single flat candidate list** across all screens. For each candidate, record:

- **Suggested DS name** (e.g. `Button/Primary`) — be opinionated, see naming guidance below
- **Confidence** — 高 / 中 / 低
- **Type** — `foundation` / `icon` / `primitive` / `composed`
- **Visual gist** — one short sentence ("紅色填色 + 白字 CTA")
- **Source screens** — list of screens where it appears (e.g. "畫面 1, 3, 4")
- **Variants observed** — short note if you see variation (e.g. "預設 + disabled", or "尺寸不一致")

**Deduplication rules — be aggressive:**
- Same visual + same function across screens → **1 candidate**, list all screens
- Visually similar but different state/style (e.g. solid blue vs ghost outline) → **1 candidate with variants** (`Button/Primary` with default + ghost), not two separate entries
- Visually similar but clearly different function (e.g. one is a CTA, one is a chip) → **separate candidates**

### Step 5: Suggest DS names with confidence

Be opinionated. The user prefers a draft they can fix over a neutral list they have to interpret.

**Naming pattern**: `Category/Name` (e.g. `Button/Primary`, `Card/Workout`, `Icon/Arrow-Right`).

**Confidence levels:**
- **高**: clear function + clear visual pattern + repeats across screens. Name is probably right.
- **中**: function clear but naming is a guess (e.g. is it `Card/Workout` or `Card/Lesson`?). Or visual is clear but you've never seen this pattern before.
- **低**: not sure if it should be extracted at all, or if it's decoration, or if it's a one-off. Flag for user.

If a candidate is **clearly local/decorative/one-off**, do not include it in the list. Stage 1's output is "what's worth considering for the DS", not "everything on the screen".

### Step 6: Output the candidate list

Use the format in `references/candidate-list-format.md`. The output has three sections:

1. **Scan summary** — N screens scanned, M candidates found
2. **Candidate list** — the flat table
3. **Next step prompt** — recommend `ds-extract-consolidate` if list is large/complex, or skip to `ds-extract-build` if small

Never output per-screen inventories. The flat candidate list is the only deliverable.

## Hard rules

1. **Read-only.** Never call any write tool. If asked to "just go ahead and build", remind the user this is Stage 1.
2. **Skip existing instances silently.** Do not list them, do not flag them as incomplete. Assume they're intentional.
3. **Deduplicate aggressively at scan time.** Same shape across screens = one entry, not N entries.
4. **Be opinionated on naming.** Give a suggested DS name and a confidence level. Don't hedge with "looks like a button-shape" — say `Button/Primary` (中信心).
5. **Don't list every decoration.** Filter out gradients, dividers, background patterns, page padding wrappers, unless they look like reusable assets.
6. **No more than 2 `get_design_context` calls per screen.**
7. **No per-screen inventory blocks.** Output is one flat candidate list across all screens.

## Token budget guidance

Per screen:
- `get_metadata`: ~2-5k tokens
- `get_screenshot`: ~1.5k tokens
- `get_design_context` (if used, max 2x): ~5-10k tokens each

Output:
- Candidate list: ~2-5k tokens total (much smaller than the old per-screen inventory format)

Total for 10 screens: **~40-80k tokens**. The savings vs the old format come from (a) no per-screen inventory blocks, (b) aggressive cross-screen dedup, (c) terser per-candidate descriptions.

## References

- `references/candidate-list-format.md` — Exact format for the candidate list output
- `references/element-categories.md` — Categories for the "Type" field (foundation, icon, primitive, composed)
- `references/figma-mcp-patterns.md` — Figma MCP environment quirks (read once to understand constraints)
