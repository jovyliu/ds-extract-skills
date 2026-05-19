---
name: ds-extract-consolidate
description: Take the candidate list from `ds-extract-scan` and turn it into a formal **Master Build Plan** — confirm naming, group into build batches, identify dependencies, and save to disk. Use when the user has a scan candidate list and wants a structured plan for the build stage, or says "consolidate", "整合", "產出 build plan", "做 master plan". This is Stage 2 of the 4-stage extract workflow. Optional — if the scan candidate list is small and clear, the user may skip directly to `ds-extract-build`.
---

# ds-extract-consolidate

Stage 2 of the design-system extract workflow. **Optional stage** — only use when the scan candidate list is large or complex enough to need formal batch planning.

Takes the candidate list from `ds-extract-scan` and produces a **Master Build Plan**: confirmed names, grouped into ordered batches with dependencies, saved to disk as a markdown file.

This skill is **read-only with respect to Figma** (no `mcp__figma__use_figma` calls). It **does** write to the local filesystem — to `.claude/extract-plans/`.

**Design philosophy**: scan already did the cross-screen deduplication. This skill's job is just (1) confirm names, (2) group into batches, (3) save. Do not redo dedup work.

## Output language

**Chat output in Traditional Chinese (繁體中文).** Section headers in the saved Master Plan file can be bilingual. DS component names stay in English (Figma identifiers).

## When this skill triggers

Use when:
- The user has a scan candidate list (in conversation or pasted in) and wants a formal Master Plan
- The user says "consolidate", "merge", "整合", "合併"
- The user asks "幫我規劃 build 順序" or "做 master plan"

Do **not** use when:
- No scan candidate list exists (run `ds-extract-scan` first)
- The candidate list is small/clear and the user wants to go straight to build (skip this stage)

## The 4-stage workflow

```
[Stage 1] ds-extract-scan
[Stage 2] ds-extract-consolidate    ← you are here (optional)
[Stage 3] ds-extract-build
[Stage 4] ds-rebind
```

## Inputs

The skill expects a scan candidate list in the conversation context (typically from a prior `ds-extract-scan` run, or pasted in). If no candidate list is present, ask the user to either run `ds-extract-scan` first or paste a previous scan result.

## Workflow

### Step 1: Read the candidate list

Pull the candidate list from context. Expect the flat-table format from `ds-extract-scan` (see scan skill's `references/candidate-list-format.md`). Each row has: 建議命名, 信心, 類別, 視覺描述, 出現於, 變體觀察.

### Step 2: Confirm names (high-confidence pass-through, low-confidence triage)

For each candidate:
- **高信心**: keep the name as-is. No second-guessing.
- **中信心**: keep the name, but note alternatives in a "naming notes" section if you genuinely see a better option. Don't invent alternatives just to fill the field.
- **低信心**: bubble up to a "Needs Review" bucket for the user to decide. Do not silently include in the build plan.

Apply naming-convention sanity checks (see `references/naming-convention.md`):
- Slash-separated `Category/Name`
- Category by function, not by page
- Concise generic names (`Card/Workout` not `Card/HomePageWorkoutListItem`)
- Multi-word names: lowercase-hyphen (`Icon/Arrow-Right`)

If a high-confidence candidate violates naming rules, fix the name and add a short note ("renamed for consistency with convention").

### Step 3: Group into batches

Group the To-Add list into ordered batches using `references/batch-strategy.md`. Default order:

1. **Batch 1**: Foundations (color, typography, spacing, radius variables)
2. **Batch 2**: Icons (all icons — they're independent)
3. **Batch 3+**: Primitives, grouped by family (all Button variants together, all Input variants together)
4. **Final batches**: Composed components (depend on primitives)

Each batch should contain **3-5 items**. A component set with N variants counts as 1 item.

For each batch, record:
- Batch number + short title (e.g. "Batch 3: Button family")
- Items in this batch
- `Depends on`: which prior batches must complete first

### Step 4: Save the Master Plan

Use the template in `references/master-plan-template.md`. Save to:

```
.claude/extract-plans/[YYYY-MM-DD]-[short-slug].md
```

If `.claude/extract-plans/` doesn't exist, create it. If a plan with the same name exists, append `-v2` rather than overwriting silently.

The plan file should contain:
- Header (date, source screens, summary stats)
- Confirmed To Add list (organized by batch)
- Needs Review list (low-confidence candidates the user must triage)
- Progress Tracker (checkboxes for each batch + each rebind target)
- Source candidate list reference

### Step 5: Chat summary

In chat, output a short summary (not the full plan):

```
✅ Master Plan 已儲存

- 路徑: .claude/extract-plans/2026-05-19-foo.md
- 確認 To Add: N 個元件，分 M 個 batch
- Needs Review: K 個低信心候選需要你決定
- Rebind 目標: P 個畫面

下一步：跑 ds-extract-build Batch 1 開始建 Foundations。
```

Direct the user to the saved file for details.

## Hard rules

1. **One Master Plan per workflow.** Don't produce competing plans. If one exists, ask before overwriting.
2. **Trust the scan.** Scan already did dedup and gave confidence levels. Don't redo that analysis. Don't second-guess high-confidence candidates.
3. **Be conservative on low-confidence.** Low-confidence candidates go to "Needs Review", not auto-added to To Add.
4. **Don't invent components.** Only consolidate what the scan found.
5. **Don't build.** This skill plans. The next skill builds.
6. **No Figma calls.** This is a pure planning step.

## What changed from the old flow

Previously, this skill did the cross-screen deduplication and category classification work. **That work has moved to `ds-extract-scan`.** This skill is now intentionally lean — its only jobs are name confirmation, batching, and disk persistence.

If you find yourself needing to "compare elements across screens" or "decide if these are the same component" in this stage, you're doing scan's job. Stop and tell the user the scan candidate list was incomplete — they should re-run scan.

## References

- `references/naming-convention.md` — Sanity checks for DS names
- `references/batch-strategy.md` — How to order batches and handle dependencies
- `references/master-plan-template.md` — Exact format of the saved plan
- `references/classification-rules.md` — (deprecated, kept for reference) — classification logic now happens in scan
- `references/figma-mcp-patterns.md` — Figma MCP environment quirks (read for build-stage awareness)
