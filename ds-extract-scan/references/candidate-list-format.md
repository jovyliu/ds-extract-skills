# Candidate List Format

The output of `ds-extract-scan` is a **single flat table** of distinct component candidates across all scanned screens. No per-screen inventory blocks.

## Format

```markdown
# Scan 結果

**掃描畫面數**: [N]
**候選元件數**: [M]
**存取問題**: [若無則寫「無」]

---

## 元件候選清單

| # | 建議命名 | 信心 | 類別 | 視覺描述 | 出現於 | 變體觀察 |
|---|---------|------|------|---------|--------|---------|
| 1 | `Button/Primary` | 高 | primitive | 紅色填色 + 白字 CTA | 畫面 1, 3, 4 | 預設 + disabled |
| 2 | `Card/Workout` | 中 | composed | 左圖右文卡片，含標題與時長 | 畫面 1, 3 | 重複出現 5 次/畫面 |
| 3 | `Icon/Arrow-Right` | 高 | icon | 16px 向右箭頭 | 畫面 1, 2, 5 | — |
| 4 | `Color/surface-default` | 高 | foundation | 卡片底色 #F5F5F5 | 全部畫面 | — |
| 5 | `Nav/Tab-Bar` | 中 | composed | 底部 4 icon + label 導覽列 | 畫面 1, 3, 5 | 跨畫面 active state 不同 |

---

## 補充說明（限值得提的觀察）

- [若有跨畫面不一致需要使用者決定的，列在這裡]
- [若有候選歸類困難的，列在這裡]
- [否則本區可省略]

---

## 下一步

[根據 candidate 數量擇一建議：]

**情境 A — 候選清單較大或複雜（>10 candidates 或含多個 composed）:**
> 建議跑 `ds-extract-consolidate` 把這份清單轉成正式的 Master Build Plan（含 batch 分組、依賴關係、進度追蹤）。

**情境 B — 候選清單小且清楚（<10 candidates 且多為 foundation/icon/primitive）:**
> 清單已經夠精簡，可以直接跑 `ds-extract-build`，跳過 consolidate 階段。

**情境 C — 候選清單有大量低信心或補充說明:**
> 建議先跟我 review 補充說明那段，確定方向再進入 consolidate / build。
```

## Field rules

### 建議命名

- Use `Category/Name` pattern (e.g. `Button/Primary`, `Icon/Arrow-Right`)
- Use `Color/`, `Spacing/`, `Typography/`, `Radius/` prefixes for foundation tokens
- Be opinionated — give a real name, not a placeholder

### 信心

- **高**: clear visual + clear function + repeats. Name is probably right.
- **中**: function clear, but the specific name is a guess. Or new pattern not seen elsewhere.
- **低**: unsure whether to extract at all, or might be decoration/one-off. User should decide.

### 類別

One of:
- `foundation` — color, typography, spacing, radius, shadow tokens
- `icon` — standalone icons (typically <32px, single-purpose vector)
- `primitive` — atomic components (Button, Input, Badge, Checkbox)
- `composed` — components built from primitives + content (Card, Nav, ListItem)

See `references/element-categories.md` for category boundaries.

### 視覺描述

**One short sentence.** Not a paragraph. Examples:
- ✅ "紅色填色 + 白字 CTA"
- ✅ "左圖右文卡片，含標題與時長"
- ❌ "這是一個 56px 高的按鈕，使用紅色填色 #E63946，含 12px 圓角，白色粗體文字..."

If you need more than one sentence to describe it visually, it's probably two candidates, or you're overthinking.

### 出現於

- List screen numbers or names where the candidate appears
- For foundation tokens that appear everywhere: write "全部畫面"
- For items appearing on a single screen: still include — the user decides if it's worth extracting

### 變體觀察

- Note any visual variation across screens (different state, size, color)
- Use "—" if no variants observed
- This field guides the build stage's variant matrix decisions

## What does NOT go in the output

- ❌ Per-screen inventory blocks ("Screen: Workout Home... Screen: Profile...")
- ❌ Existing instances ("Already DS (skipped)" lists) — silently filtered
- ❌ Decoration / one-off / page-padding / gradient backgrounds — filtered unless they look reusable
- ❌ Long descriptions for each candidate
- ❌ Build order, batch suggestions, dependencies — that's Stage 2's job
- ❌ "Should be Button/Primary" prescriptive language — just write the name directly

## Tone

- Direct and opinionated
- Confidence levels (高/中/低) carry the hedging — prose doesn't need to also hedge
- Skip "appears to be", "might be" — say it or don't list it
