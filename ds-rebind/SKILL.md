---
name: ds-rebind
description: Replace raw frames and layers in source screens with instances of newly-built design system components. Identifies what can be safely swapped, asks before touching ambiguous areas. Use when the user says "rebind", "把畫面接到新元件", "用新的 DS 換掉舊的", or wants to update source screens after building components. This is Stage 4 (final) of the 4-stage extract workflow. Always verify the relevant components exist before rebinding.
---

# ds-rebind

Stage 4 (final) of the design-system extract workflow. Replaces raw frames/layers in source screens with instances of components built in Stage 3.

This skill **writes to Figma** — it modifies source screens. Requires `mcp__figma__use_figma` access.

This is the highest-risk skill because it changes existing artifacts. Every change is previewed and confirmed before write.

**Design philosophy**: the designer can verify in Figma directly, faster than any screenshot diff. Screenshots are optional, user-triggered. Save tokens for the actual swaps.

## Output language

**All user-facing output must be in Traditional Chinese (繁體中文).** Component / variable names stay in English (Figma identifiers).

## When this skill triggers

Use when:
- The user says "rebind", "rebind 這個畫面", "用新元件接回去"
- A Master Plan exists with completed build batches and rebind targets
- The user finished building components and wants to update source screens

Do **not** use when:
- No components have been built yet (run `ds-extract-build` first)
- The user wants to scan / plan / build (use earlier stages)

## The 4-stage workflow

```
[Stage 1] ds-extract-scan
[Stage 2] ds-extract-consolidate
[Stage 3] ds-extract-build
[Stage 4] ds-rebind                ← you are here
```

## ⚠️ Read this first

Before doing anything, read `references/figma-mcp-patterns.md`. Most relevant for rebind:

- **Pattern 1**: `use_figma` doesn't return values — use scratch text node for verification
- **Pattern 3**: Component sets vs variants — `createInstance` only works on variants (`<symbol>`), never on the set wrapper. Walk down to the specific variant when picking the swap target.

## Pre-flight checks (mandatory)

1. **Tool surface check**: Verify `mcp__figma__use_figma` is available.
   - If not, stop. Tell user the session needs Figma write access.

2. **Plan / context check**: Locate the relevant Master Plan in `.claude/extract-plans/`, OR confirm with the user which built components map to which source elements (candidate-list mode).
   - If multiple plans exist, ask which one
   - Identify source screen(s) from the plan's Rebind Targets section

3. **Component readiness check**: For the screen being rebound, verify components it needs are already built.
   - Read source screen's candidates from plan / conversation
   - For each To-Add item, check the component now exists in the DS file
   - **Important**: for multi-variant components, find the **variant** (the `<symbol>` child of the component-set frame) whose name matches the desired property. Track variant IDs separately — never use the wrapper frame ID. See Pattern 3.
   - If a needed component is missing, stop. Tell user which batches still need to run.

4. **Source screen check**: Verify the source screen URL is still valid and accessible.

## Workflow

### Step 1: Plan the rebind

For the target screen:

1. Read current state via `mcp__figma__get_metadata` + `mcp__figma__get_screenshot`
2. For each candidate from the plan, identify its current location in the screen
3. Determine swap action for each:
   - **Replace with instance**: raw frame matches a built component → swap
   - **Replace text style**: raw text matches a typography style → apply
   - **Replace fill**: raw color matches a color variable → bind
   - **Leave alone**: marked Do Not Extract → no action
   - **Ambiguous**: unclear → flag for user

### Step 2: Preview the rebind plan

Show the user a preview:

```markdown
# Rebind Preview — [Screen Name]

## ✅ Safe replacements (will swap)
- Hero CTA frame → Button/Primary (Default)
- Workout list row 1-5 → Card/Workout (5 instances)
- Bottom tab bar → Nav/Tab-Bar

## ✅ Style bindings (will apply)
- Page title text → typography/heading-2
- Card background fill → color/surface-default

## ⚠️ Needs your confirmation
- Bottom banner: not in plan, could be one-off or new Banner component.
  Action: skip and leave as-is. OK?

## ⏭️ Leave alone
- Gradient background (Do Not Extract)
- Page padding wrapper (local layout)

Confirm to proceed? (yes / adjust / skip specific items)
```

Wait for user confirmation. Allow per-item adjustment.

### Step 3: Execute swaps

Swap **one element at a time**, in this order:

1. Foundation bindings first (color fills, typography) — lowest risk
2. Icon swaps (raw vector → Icon/* instance)
3. Primitive swaps (raw frame → Button/Primary instance, etc.)
4. Composed swaps (raw frame → Card/* or Nav/* instance) — highest risk

**Verification between swaps**: minimal. Read back the element after each swap to confirm the swap took effect (via scratch-node pattern from Pattern 1), but **do not screenshot between swaps**. The designer will look at Figma when the batch completes.

If a swap fails, flag immediately and ask the user how to proceed (don't auto-rollback, don't auto-retry).

### Step 4: Verification (lean)

After all swaps on the screen are done:

1. **Always**: read back the screen's metadata once to confirm the swap count matches what was attempted
2. **Only if user asks** ("show me the diff", "screenshot 一下", "幫我比對"): take an after-screenshot

**No before/after screenshots by default.** The designer can compare in Figma directly — that's faster and more accurate than any screenshot diff. Save the tokens.

### Step 5: Rebind report

```markdown
# Rebind Report — [Screen Name]

## ✅ Replaced
- [element location] → [DS component name]
- ...

## ✅ Style-bound
- [element location] → [variable/style name]
- ...

## ⚠️ Issues encountered
- [element] — [what happened, what was done]

## ⏭️ Left alone
- [element] — [reason]

## Master Plan updated
Checked off Rebind: [Screen Name] in [plan path]

## Next step
- Run ds-rebind for the next source screen: [next screen name], OR
- Run ds-checkup to validate rebound screens against the DS, OR
- Done — all screens rebound!
```

### Step 6: Update Master Plan

Open the Master Plan file and check off the rebind item in the Progress Tracker.

## Hard rules

1. **Pre-flight checks are mandatory.**

2. **Preview before write.** Always show the full rebind plan and wait for confirmation. Per-item granularity allowed.

3. **One element at a time.** No bulk swaps within a screen.

4. **One screen at a time.** Even if the user says "rebind all 5 screens", do them sequentially.

5. **Don't extract new components mid-rebind.** If you find something that should be a component but isn't in the plan:
   - Note it in the report under Issues
   - Suggest the user re-run scan / consolidate / build to add it
   - Do NOT silently create new components from this skill

6. **Conservative on ambiguity.** When unsure if a swap is safe (different padding, close-but-not-identical fill), default to **skip and flag**.

7. **Visual diff is optional, not mandatory.** Only screenshot when the user explicitly asks. The designer reviews in Figma directly.

8. **No silent style overrides.** When binding a variable to an element with a slightly different hardcoded value, surface the diff. User decides.

## Risk levels and confirmation policy

| Action | Risk | Confirmation |
|---|---|---|
| Apply typography style | Low | Bundled in preview |
| Bind color variable to identical fill | Low | Bundled in preview |
| Bind color variable to similar (not identical) fill | Medium | Surface the diff, ask explicitly |
| Swap raw frame to primitive instance | Medium | Bundled in preview |
| Swap composed component (large) | High | Per-item confirm |
| Touch anything not in the plan | Very high | Refuse — update plan first |

## Token budget

Per screen rebind (with lean verification):
- Reading current state: ~10-15k
- Planning swaps: minimal
- Executing swaps (per element): ~3-8k
- Final metadata sweep: ~3k
- Screenshot (only if requested): ~3k

Total per screen: **~25-50k tokens** (down from 30-60k). The savings come from skipping mandatory before/after screenshots.

If a screen has >10 elements to swap, suggest splitting into two passes (primitives first, composed next turn).

## References

- `references/figma-mcp-patterns.md` — **REQUIRED READING.** Figma MCP environment patterns.
- `references/swap-procedures.md` — How to perform each type of swap safely
- `references/safety-checklist.md` — Pre-write safety checklist
- `references/diff-format.md` — Format for visual diff (only used when user requests)
