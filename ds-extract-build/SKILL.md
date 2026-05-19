---
name: ds-extract-build
description: Build a batch of design system elements in Figma based on a Master Build Plan (from `ds-extract-consolidate`) or directly from a scan candidate list. Creates Foundations, Icons, Primitives, and Composed components using strict naming. Use when the user says "build Batch N", "build the foundations", "建這批元件", "extract build", or wants to execute against a plan. This is Stage 3 of the 4-stage extract workflow.
---

# ds-extract-build

Stage 3 of the design-system extract workflow. Executes a batch of items by creating actual elements in Figma using `mcp__figma__use_figma`.

This skill **writes to Figma**. Requires `mcp__figma__use_figma` to be available.

**Design philosophy**: build with minimum Figma round-trips. Verify the batch once at the end, not after every item. Skip screenshots unless the batch contains high-risk items or the user asks.

## Output language

**All user-facing output must be in Traditional Chinese (繁體中文).** Figma component/variable names stay in English.

## When this skill triggers

Use when:
- The user says "build Batch N" or names specific items to build
- The user says "建這批元件", "建 foundations", "extract build"
- A Master Plan exists at `.claude/extract-plans/` and the user wants to execute against it
- A scan candidate list is in conversation and the user wants to skip consolidate and build directly

Do **not** use when:
- No plan or candidate list exists (run `ds-extract-scan` first)
- The user wants to scan or plan (use earlier stages)
- The user wants to rebind source screens (use `ds-rebind`)

## The 4-stage workflow

```
[Stage 1] ds-extract-scan
[Stage 2] ds-extract-consolidate (optional)
[Stage 3] ds-extract-build         ← you are here
[Stage 4] ds-rebind
```

## ⚠️ Read this first

Before doing anything, read `references/figma-mcp-patterns.md`. Five environment patterns every build step depends on:

1. `use_figma` doesn't return values — use scratch text node pattern
2. `createPage()` is unreliable — build on existing Page 1
3. Component sets vs variants — `createInstance` only works on variants
4. Pre-flight inventory of existing variables — avoid duplicates
5. Cross-library variable binding can render incorrectly — try-then-verify with raw hex fallback

Skipping this read produces silent broken output.

## Pre-flight checks (mandatory)

Before any write operation:

1. **Tool surface check**: Verify `mcp__figma__use_figma` is available.
   - If not: stop. Tell the user the session needs Figma write access.

2. **Plan / candidate list check**: Locate the source of truth.
   - Master Plan at `.claude/extract-plans/` (preferred), OR
   - Scan candidate list in conversation (acceptable for direct build without consolidate)
   - If neither exists: stop. Tell user to run `ds-extract-scan` first.

3. **Batch identification**: Confirm which items are being built.
   - "Batch 1" → look up Batch 1 in the plan
   - Named items → confirm they're in the To Add list / candidate list
   - "Next batch" → look at the Progress Tracker, find the next unchecked batch
   - Direct build from candidate list (no plan): user must name the specific candidates to build this turn

4. **Dependency check**: For Master Plan batches, verify `Depends on` batches are complete.
   - Missing deps: warn the user. Offer to build deps first or proceed at user's risk.

5. **Target file check**: Confirm where to build.
   - Ask which Figma file is the DS file
   - Plan to build on an existing page (typically Page 1), in empty canvas space far from source content
   - Mark artifacts with `[DS-Extract Build Output]` titles — user will manually move them into their Master Library afterward

6. **Inventory existing variables** (Foundations batches only): Run `mcp__figma__get_variable_defs` on one representative source node to see existing tokens.
   - If file already has tokens covering this batch → ask whether to reuse or build new
   - If tokens are missing → proceed to create
   - If tokens conflict → flag as Needs Review

7. **Inventory existing components by name**: Run `mcp__figma__search_design_system` on the target file with planned names. If local matches exist, ask whether to skip, extend, or rename.

## Workflow

### Step 1: Load the source of truth

Read the Master Plan or candidate list. Extract:
- Items for this batch
- Naming convention
- Any cross-cutting notes
- (Master Plan only) Progress Tracker state

### Step 2: Preview before write

Show the build preview:

```
Building Batch [N]: [Title]

Items to create:
1. [name] — [type, e.g. "component set with 3 variants"]
2. [name] — ...

Target file: [Figma URL]
Target page: [page name]

Verify mode: [end-of-batch | per-item | screenshot]

Proceeding in 1 tool call to start. Confirm? (yes / adjust)
```

Wait for user confirmation.

### Step 3: Build items in sequence

For each item, follow the type-specific procedure:
- `references/build-foundations.md` — color/typography/spacing/radius/shadow variables
- `references/build-icons.md` — icons
- `references/build-primitives.md` — primitive components
- `references/build-composed.md` — composed components

**Build order within batch:**
1. Items with no internal dependencies first
2. Items consuming other batch items last

**One item at a time, but DO NOT verify between items.** Build → next → build → next. Verification happens once at the end of the batch (see Step 4).

### Step 4: Batch-end verification (single sweep)

After all items in the batch are built, do **one** verification pass:

1. Call `mcp__figma__get_metadata` on the parent page. Find every newly created component by name. Confirm they all exist with the right type (component / component set with expected variants).

2. **Screenshot only when**:
   - The batch contains **composed components** (high visual risk), OR
   - The batch involved **cross-library variable binding** (Pattern 5 — colors might render wrong), OR
   - The **user explicitly asked** to see a visual check
   - Otherwise: skip screenshots. Save tokens. The user can spot-check in Figma directly.

3. If something is missing or malformed, note it in the build report under "Issues" — don't loop endlessly retrying.

**Verify mode summary:**

| Batch contents | Default verify mode |
|---|---|
| Foundations (variables only) | `get_metadata` sweep, no screenshot |
| Icons (5-10 simple icons) | `get_metadata` sweep, no screenshot |
| Primitives (1-3 component sets) | `get_metadata` sweep, no screenshot |
| Composed components | `get_metadata` sweep + 1 screenshot per composed component |
| Cross-library variables involved | Add 1 screenshot of any affected component |
| User asks "let me see it" | Add screenshots, user-specified scope |

### Step 5: Update Progress Tracker (Master Plan only)

If working from a Master Plan:
- Open the plan file
- Check off the completed batch's checkbox
- If batch partially completed, add a note below the batch line indicating partial state

### Step 6: Build report

```markdown
# Build Report — Batch [N]

## ✅ Created
- [name] — [variants if applicable]
- ...

## ⚠️ Issues
- [name] — [what went wrong, what was done]

## ⏭️ Skipped
- [name] — [reason]

## Progress
Batch [N]: [X / Y items]
Master Plan updated: [path or "no plan, candidate-list mode"]

## Next step
- Run `ds-spec` to generate spec pages for the new components, OR
- Run `ds-extract-build` for the next batch ([N+1]: [title]), OR
- Run `ds-rebind` to start rebinding source screens (only after needed primitives are built)
```

## Hard rules

1. **Pre-flight checks are mandatory.** Never skip. If a check fails, stop and explain.

2. **Preview before write.** Always show the build preview, wait for confirmation. Re-confirm at the start of each new batch.

3. **No per-item verification.** Build items in sequence without intermediate `get_metadata` / `get_screenshot` calls. Verify the whole batch once at the end (Step 4).

4. **No screenshots by default.** Only screenshot when the batch contains composed components, involves cross-library variables, or the user asks.

5. **Strict naming.** Use names exactly as written in the plan / candidate list. If a name is wrong, stop and ask the user to fix it first.

6. **Check for duplicates.** Use `mcp__figma__search_design_system` in pre-flight. If duplicates exist locally, ask the user before creating.

7. **Don't go off-plan.** If you notice something the plan missed, note it in the report — don't add it.

8. **Batch size cap.** Refuse to build more than 5 items per invocation (component sets count as 1). If a batch in the plan has more, split into sub-batches and confirm.

9. **No silent failures.** Surface every tool failure immediately. No endless retries.

## Token budget

Per batch (with end-of-batch verify):
- Foundations (8-20 variables): ~20-35k tokens (down from 30-60k)
- Icons (5-10 icons): ~25-50k tokens (down from 40-80k)
- Primitives (1-3 component sets): ~35-65k tokens (down from 50-100k)
- Composed (1-2 components, includes screenshot): ~40-80k tokens (down from 50-120k)

If approaching 120k in a single invocation, stop and split. The savings vs the old per-item-verify approach come from cutting redundant `get_metadata` / `get_screenshot` round-trips.

## References

- `references/figma-mcp-patterns.md` — **REQUIRED READING.** Five critical environment patterns.
- `references/build-foundations.md` — How to create Figma Variables and Styles
- `references/build-icons.md` — How to create icon components
- `references/build-primitives.md` — How to create primitive component sets
- `references/build-composed.md` — How to create composed components
- `references/file-structure.md` — Target Figma file structure
