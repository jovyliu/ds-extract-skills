這組 skills 本身是 model-agnostic 的 workflow instructions，可在 Codex 或 Claude 類型的 skill 系統中使用。前提是執行環境支援 SKILL.md-based skills，並且有相容的 Figma MCP tools。
These skills are model-agnostic workflow instructions. They can be used in Codex or Claude-style skill systems as long as the host supports SKILL.md-based skills and exposes compatible Figma MCP tools.

# ds-extract-skills

Figma design-system extraction workflow skills for Codex.

This repo contains a 4-stage skill workflow for turning existing Figma screens into a cleaner, reusable design system:

1. `ds-extract-scan`
2. `ds-extract-consolidate`
3. `ds-extract-build`
4. `ds-rebind`

The workflow is designed for teams that already have real product screens in Figma, but do not yet have a well-structured component library behind them.

## What problem does this solve?

Design systems often fail to start from a blank canvas. In real teams, the UI already exists across product screens:

- buttons are duplicated manually
- cards look similar but are not components
- icons and tokens are scattered across pages
- designers are unsure which patterns should become reusable components
- component naming and build order become inconsistent
- after components are built, the original screens are still disconnected from the new design system

`ds-extract-skills` turns that messy extraction process into a structured workflow.

Instead of asking an AI agent to "make a design system" in one vague step, this repo breaks the job into clear stages:

- scan existing screens
- deduplicate and name reusable candidates
- consolidate candidates into a build plan
- build the design system elements in Figma
- rebind source screens back to the newly built components

The goal is not to generate a design system from nothing. The goal is to help designers and Design Ops teams systematize the UI patterns that already exist.

## Who is this for?

This repo is useful for:

- product designers cleaning up mature Figma files
- design system maintainers extracting reusable components from shipped screens
- Design Ops teams standardizing component naming and build order
- teams migrating raw frames into a proper component library
- AI-assisted design workflows that need safer, staged Figma operations

## Workflow overview

```text
Stage 1: Scan
Existing Figma screens
        |
        v
Stage 2: Consolidate
Candidate list -> Master Build Plan
        |
        v
Stage 3: Build
Foundations, icons, primitives, composed components
        |
        v
Stage 4: Rebind
Source screens use the new DS components
```

## Skills

### `ds-extract-scan`

Scans one or more Figma source screens and produces a lean component candidate list.

Use this when you want to answer:

- Which raw UI patterns are worth extracting?
- Which repeated elements should become components?
- Which candidates are high-confidence vs. uncertain?
- Which existing Figma instances should be ignored because they are already imported components?

Output:

- a single flat candidate table
- suggested DS names such as `Button/Primary` or `Card/Lesson`
- confidence levels: high / medium / low
- category labels: `foundation`, `icon`, `primitive`, `composed`
- source screens where each candidate appears
- observed variants

This stage is read-only. It does not modify Figma.

### `ds-extract-consolidate`

Turns the scan candidate list into a formal Master Build Plan.

Use this when the candidate list is large, ambiguous, or needs build sequencing.

Output:

- confirmed component names
- items grouped into build batches
- dependency order
- low-confidence candidates separated into a review bucket
- a saved plan under `.claude/extract-plans/`

This stage is optional. If the candidate list is small and obvious, you can skip straight to `ds-extract-build`.

### `ds-extract-build`

Builds a batch of design system elements in Figma.

Use this when you are ready to create:

- foundation variables
- icon components
- primitive components such as buttons, inputs, badges, and checkboxes
- composed components such as cards, nav bars, and list items

This stage writes to Figma. It requires Figma MCP write access.

### `ds-rebind`

Replaces raw layers and frames in source screens with instances of the newly built design system components.

Use this after components are built and you want the original screens to actually connect back to the design system.

This stage is intentionally conservative:

- previews replacements before writing
- swaps one screen at a time
- asks before touching ambiguous areas
- does not create new components mid-rebind

This stage writes to Figma source screens.

## Recommended usage

### 1. Scan source screens

Ask Codex:

```text
Use ds-extract-scan to scan these Figma screens and tell me which components can be extracted.
```

Provide one or more Figma frame URLs.

### 2. Consolidate into a Master Build Plan

If the scan result is large or contains composed components:

```text
Use ds-extract-consolidate to turn this candidate list into a Master Build Plan.
```

### 3. Build a batch

After reviewing the plan:

```text
Use ds-extract-build to build Batch 1.
```

Then continue batch by batch.

### 4. Rebind source screens

After the components exist:

```text
Use ds-rebind to replace the raw elements in this screen with the new DS components.
```

## Repo structure

```text
.
├── ds-extract-scan/
│   ├── SKILL.md
│   └── references/
│       └── candidate-list-format.md
├── ds-extract-consolidate/
│   └── SKILL.md
├── ds-extract-build/
│   └── SKILL.md
├── ds-rebind/
│   └── SKILL.md
├── ds-extract-scan.zip
├── ds-extract-consolidate.zip
├── ds-extract-build.zip
└── ds-rebind.zip
```

The `.zip` files are packaged versions of each skill folder.

## Installation

Copy the skill folders into your Codex skills directory.

Example:

```bash
cp -R ds-extract-scan ~/.codex/skills/
cp -R ds-extract-consolidate ~/.codex/skills/
cp -R ds-extract-build ~/.codex/skills/
cp -R ds-rebind ~/.codex/skills/
```

Restart Codex after copying the folders so the skills can be discovered.

## Language and naming

The skills are optimized for a Traditional Chinese design workflow:

- user-facing responses are in Traditional Chinese
- Figma component names stay in English
- design system names follow `Category/Name`, for example:
  - `Button/Primary`
  - `Icon/Arrow-Right`
  - `Card/Lesson`
  - `Typography/Heading-2`

## Design principles

### Staged, not magical

Each stage has a narrow responsibility. This makes the workflow easier to inspect, correct, and repeat.

### Opinionated, not exhaustive

The scan stage should produce a useful candidate list, not a full inventory of every layer.

### Conservative with writes

Only the build and rebind stages write to Figma. Rebind is especially cautious because it modifies source screens.

### Existing components are respected

The scan stage skips existing Figma instances instead of treating them as incomplete work.

### The designer stays in control

Low-confidence candidates and ambiguous rebinds are surfaced for review instead of being silently changed.

## Pain points covered

This workflow helps with:

- finding reusable patterns across many Figma screens
- reducing duplicate raw UI layers
- turning scattered screen-level UI into component candidates
- creating a build order for foundations, icons, primitives, and composed components
- avoiding messy component naming drift
- separating high-confidence extraction from uncertain one-offs
- connecting finished source screens back to the design system

## Current scope

This repo contains the workflow instructions for Codex skills. It does not include a standalone CLI or app.

To use the full workflow, you need:

- Codex with custom skills support
- access to the relevant Figma MCP tools
- Figma file permissions for read operations
- Figma write access for `ds-extract-build` and `ds-rebind`

## Repository

GitHub:

https://github.com/jovyliu/ds-extract-skills
