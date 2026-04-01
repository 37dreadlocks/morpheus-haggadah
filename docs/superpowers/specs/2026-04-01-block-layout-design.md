# Block Layout Design
**Date:** 2026-04-01
**Status:** Approved

## Problem

Blocks inside the chapter card have conflicting layout needs:
- The focused block should appear centered in the card on arrival
- As content grows (options, feedback, chat replies), it must scroll
- The chat input (`block-input`) must never scroll off screen — it must stay pinned to the bottom of the block
- The `↓` continue button (`material-next-btn`) must also stay pinned — it currently floats as a feed sibling and can be pushed off screen

## Design

### Core principle

The in-focus block fills all remaining card height (`flex: 1`). Inside the block, only a designated **scroll zone** scrolls. Everything else — the input and the next button — lives in a **sticky footer** that never moves.

### DOM structure

#### Q-group (questions)

```
.q-group.in-focus                    — flex: 1, flex-direction: column
  .question-block                    — flex-shrink: 0 (label/header area)
  .scroll-zone                       — flex: 1, overflow-y: auto
    .scroll-content                  — margin: auto 0  ← centering trick
      .q-text
      .options
      .q-user-bubble (on answer)
      .q-feedback-row (on answer)
  .sticky-bottom                     — flex-shrink: 0
    .block-input (.block-input-wrap + textarea + send btn)
```

#### Content block (overview / material with willPause)

```
.msg.content-block.in-focus          — flex: 1, flex-direction: column
  .content-block-label               — flex-shrink: 0
  .scroll-zone                       — flex: 1, overflow-y: auto
    .scroll-content                  — margin: auto 0
      .msg-body
      .material-chat (grows with replies)
  .sticky-bottom                     — flex-shrink: 0
    .block-input
    .next-btn-row (centered ↓ button)
```

### Centering mechanism

`scroll-content` uses `margin: auto 0` inside the flex `scroll-zone`. When content height is less than the scroll zone height, equal auto margins push it to vertical center. As content grows past the zone height, margins collapse to 0 and the browser enables scrolling — no JS needed, no threshold detection.

### Sticky footer

`sticky-bottom` is a `flex-shrink: 0` sibling to `scroll-zone` inside the block's flex column. It is always visible regardless of scroll position. For content blocks it contains both the chat input and the `↓` next button stacked vertically.

### `↓` next button placement

`awaitMaterialNext` currently inserts the button as a **feed sibling** after the focused block. This must change: the button is appended inside the block's `sticky-bottom` area, so it can never be pushed off screen by growing content.

### Feed-level layout

The feed remains a flex column with `overflow-y: auto`. Only the focused block has `flex: 1`. Past blocks remain their natural height (dimmed via `.past`). No padding tricks or JS scroll calculations are needed for centering — the `margin: auto` inside the scroll zone handles it.

## CSS changes

| Selector | Change |
|---|---|
| `.feed > .in-focus.q-group` | `flex: 1; display: flex; flex-direction: column` (keep) |
| `.feed > .in-focus.content-block` | add `flex: 1; display: flex; flex-direction: column` |
| `.q-group.in-focus .question-block` | remove sticky positioning override (already `position: static`) |
| `.scroll-zone` (new) | `flex: 1; overflow-y: auto; min-height: 0; display: flex; flex-direction: column` |
| `.scroll-content` (new) | `margin: auto 0; display: flex; flex-direction: column; gap: …` |
| `.sticky-bottom` (new) | `flex-shrink: 0; padding: …; border-top: …` |

## JS changes

| Function | Change |
|---|---|
| `showQuestion` | wrap `.q-full` + options in `.scroll-zone > .scroll-content`; move `.block-input` into `.sticky-bottom` |
| `showFreeTextQuestion` | same restructure as `showQuestion` |
| `appendMoMessage` (willPause) | wrap `.msg-body` + `.material-chat` in `.scroll-zone > .scroll-content`; put `.block-input` in `.sticky-bottom` |
| `awaitMaterialNext` | append `↓` button into the block's `.sticky-bottom`, not as feed sibling |
| `renderOptions` | append options into `.scroll-content` inside `.scroll-zone` |
| `appendFeedbackToGroup` | append feedback rows into `.scroll-content` |
| `centerFocusedBlock` | remove — centering is handled by CSS `margin: auto 0`, no JS needed |
| `setActive` | keep dimming logic; remove the scroll calculation (no longer needed) |
