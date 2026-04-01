# Feed Scroll on Focus — Design
**Date:** 2026-04-01
**Status:** Approved

## Problem

The in-focus block has `flex: 1` so it fills remaining card height after past blocks. When several past blocks accumulate, the in-focus block is pushed to the bottom and gets only a tiny sliver of space — content is clipped and internal scroll doesn't work.

## Design

### On focus: scroll feed to in-focus block

When `setActive` runs, scroll the feed so the top of the in-focus element is 28px from the top of the card:

```javascript
feed.scrollTo({ top: Math.max(0, el.offsetTop - 28), behavior: 'smooth' });
```

This positions the in-focus block at the top of the visible area. `flex: 1` then fills the rest of the card. Past blocks sit above the scroll position — accessible by scrolling up.

The 28px offset leaves a sliver of the last past block peeking above the in-focus block, giving the user a visual cue that history exists above.

### Hover: undim past blocks

Past blocks are dimmed via `.past` CSS. When hovered, remove the dimming so the user can read past content without scrolling all the way to it:

```css
.feed .past:hover {
  opacity: 1;
  filter: none;
}
```

## Changes

| Location | Change |
|---|---|
| `setActive` JS function | Add `feed.scrollTo(...)` after the classList toggle |
| CSS `.past` section | Add `.feed .past:hover { opacity: 1; filter: none; }` |
