# Block Layout Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the focused block fill remaining card height, scroll its content internally, and pin the chat input + continue button to a sticky footer that never scrolls away.

**Architecture:** The in-focus block becomes a flex column with `flex: 1`. Inside it, `.question-block` (for q-groups) or a `.scroll-zone` div (for content blocks) gets `flex: 1; overflow-y: auto` — only this area scrolls. A `.sticky-bottom` sibling holds the chat input and `↓` button with `flex-shrink: 0`. Content is centered when short via `margin: auto 0` on the inner content wrapper.

**Tech Stack:** Vanilla HTML/CSS/JS — single file `index.html`.

---

## File

- Modify: `index.html` (all changes are in this one file)

---

### Task 1: Add CSS utility classes

**Files:**
- Modify: `index.html` — `<style>` block, after the existing `/* ── IN-FOCUS BLOCK ── */` section

- [ ] **Step 1: Add `.scroll-zone`, `.scroll-content`, `.sticky-bottom` classes**

Find this comment in the CSS:
```css
/* ── IN-FOCUS BLOCK ── */
```

Add these rules immediately after the existing `/* ── IN-FOCUS BLOCK ── */` block (after `.feed > .in-focus .block-input` rule):

```css
/* Scrollable area inside in-focus block — only this region scrolls */
.scroll-zone {
  flex: 1;
  overflow-y: auto;
  min-height: 0;
  display: flex;
  flex-direction: column;
}

/* Centers content vertically when short; collapses to 0 margin when content overflows */
.scroll-content {
  margin: auto 0;
  display: flex;
  flex-direction: column;
}

/* Pinned footer — never scrolls away */
.sticky-bottom {
  flex-shrink: 0;
  padding: 10px 14px 14px;
  border-top: 1px solid var(--card-border);
}

.q-group.in-focus .sticky-bottom {
  background: var(--q-block-bg);
  padding: 10px 24px 14px;
}

/* Content block: when not in-focus, undo the in-focus wrappers */
.msg.content-block:not(.in-focus) .scroll-zone {
  display: contents;
}
.msg.content-block:not(.in-focus) .sticky-bottom {
  display: none;
}

/* In-focus content block fills card like q-group */
.feed > .in-focus.content-block {
  flex: 1;
  display: flex;
  flex-direction: column;
}
```

- [ ] **Step 2: Verify visually**

Open `index.html` in a browser. The page should look identical to before — no new classes are applied to anything yet.

---

### Task 2: Restyle `.q-group.in-focus` — make question-block the scroll zone

**Files:**
- Modify: `index.html` — CSS, around the existing `.q-group.in-focus .question-block` rule

- [ ] **Step 1: Replace the question-block centering rule**

Find and replace:
```css
/* Center question content vertically within the in-focus group */
.q-group.in-focus .question-block {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  position: static;
}
```

Replace with:
```css
/* question-block IS the scroll zone for in-focus q-groups */
.q-group.in-focus .question-block {
  flex: 1;
  overflow-y: auto;
  min-height: 0;
  display: flex;
  flex-direction: column;
  position: static;
}

/* q-full centers when short, scrolls when tall */
.q-group.in-focus .q-full {
  margin: auto 0;
}
```

- [ ] **Step 2: Replace the block-input rules**

Find and remove this rule (it's superseded by `.sticky-bottom`):
```css
.q-group.in-focus .block-input {
  background: var(--q-block-bg);
  padding: 12px 24px 32px;
}
```

Find and replace:
```css
.q-group.collapsed .block-input { display: none; }
```
with:
```css
.q-group.collapsed .sticky-bottom { display: none; }
```

Also remove this rule entirely (sticky-bottom handles the padding now):
```css
.feed > .in-focus .block-input {
  padding-bottom: 32px;
}
```

- [ ] **Step 3: Verify**

Open the page. A question block should now fill the remaining card height. At this point the textarea may be in the wrong place (still appended to group, not in sticky-bottom) — that's fixed in Task 3.

---

### Task 3: Restructure `showQuestion` — move input into sticky-bottom

**Files:**
- Modify: `index.html` — `showQuestion` function

- [ ] **Step 1: Wrap block-input in sticky-bottom**

Find in `showQuestion`:
```javascript
  group.appendChild(block);

  // Block input for typed answers (A/B/C/D)
  const { el: mcBiEl, focus: focusMcInput } = createBlockInput('Type A, B, C, or D…', (raw) => {
    const letter = raw.toUpperCase();
    const optionIdx = ['A','B','C','D'].indexOf(letter);
    if (activeQuestion && optionIdx !== -1 && raw.length === 1) {
      const { prevWrong } = activeQuestion;
      if (optionIdx < step.options.length && !prevWrong.includes(optionIdx)) {
        const userText = `${letter} — ${step.options[optionIdx]}`;
        handleAnswer(chapterIdx, stepIdx, optionIdx, prevWrong, userText);
      }
    }
  });
  group.appendChild(mcBiEl);
```

Replace with:
```javascript
  group.appendChild(block);

  // Sticky footer holds the input — never scrolls away
  const stickyBottom = document.createElement('div');
  stickyBottom.className = 'sticky-bottom';

  // Block input for typed answers (A/B/C/D)
  const { el: mcBiEl, focus: focusMcInput } = createBlockInput('Type A, B, C, or D…', (raw) => {
    const letter = raw.toUpperCase();
    const optionIdx = ['A','B','C','D'].indexOf(letter);
    if (activeQuestion && optionIdx !== -1 && raw.length === 1) {
      const { prevWrong } = activeQuestion;
      if (optionIdx < step.options.length && !prevWrong.includes(optionIdx)) {
        const userText = `${letter} — ${step.options[optionIdx]}`;
        handleAnswer(chapterIdx, stepIdx, optionIdx, prevWrong, userText);
      }
    }
  });
  stickyBottom.appendChild(mcBiEl);
  group.appendChild(stickyBottom);
```

- [ ] **Step 2: Remove `centerFocusedBlock` call**

Find in `showQuestion`:
```javascript
  return typeInto(block.querySelector('.q-text'), step.text).then(() => {
    if (group.classList.contains('collapsed')) return;
    renderOptions(chapterIdx, stepIdx, []);
    centerFocusedBlock(feed, group, block);
  });
```

Replace with:
```javascript
  return typeInto(block.querySelector('.q-text'), step.text).then(() => {
    if (group.classList.contains('collapsed')) return;
    renderOptions(chapterIdx, stepIdx, []);
  });
```

- [ ] **Step 3: Verify**

Reload. The question should appear with the text input pinned at the bottom of the card. Answering options should work normally.

---

### Task 4: Restructure `showFreeTextQuestion` — move input into sticky-bottom

**Files:**
- Modify: `index.html` — `showFreeTextQuestion` function

- [ ] **Step 1: Wrap block-input in sticky-bottom**

Find in `showFreeTextQuestion`:
```javascript
  // Block input for freetext answer
  const { el: freetextBiEl, focus: focusFreetextInput } = createBlockInput('Type your answer…', (raw) => {
    if (activeQuestion && activeQuestion.freetext) {
      handleFreeTextAnswer(chapterIdx, stepIdx, raw);
    }
  });
  group.appendChild(freetextBiEl);
```

Replace with:
```javascript
  // Sticky footer holds the input
  const stickyBottom = document.createElement('div');
  stickyBottom.className = 'sticky-bottom';

  // Block input for freetext answer
  const { el: freetextBiEl, focus: focusFreetextInput } = createBlockInput('Type your answer…', (raw) => {
    if (activeQuestion && activeQuestion.freetext) {
      handleFreeTextAnswer(chapterIdx, stepIdx, raw);
    }
  });
  stickyBottom.appendChild(freetextBiEl);
  group.appendChild(stickyBottom);
```

- [ ] **Step 2: Verify**

The freetext question (Chapter I, step 4 — "A procedural deviation…") should show the textarea pinned at the bottom of the card.

---

### Task 5: Update cleanup — target `.sticky-bottom` instead of `.block-input`

Three places currently find and remove `.block-input` from the group. They must now find `.sticky-bottom` instead.

**Files:**
- Modify: `index.html` — `handleAnswer`, `handleSkip`, `handleFreeTextAnswer`

- [ ] **Step 1: Fix `handleAnswer` (correct branch)**

Find:
```javascript
    // Clean up block-input and in-focus before collapsing
    const bi = group.querySelector('.block-input');
    if (bi) bi.remove();
    group.classList.remove('in-focus');
    activeBlockInput = null;
```

Replace with:
```javascript
    // Clean up sticky-bottom and in-focus before collapsing
    const sb = group.querySelector('.sticky-bottom');
    if (sb) sb.remove();
    group.classList.remove('in-focus');
    activeBlockInput = null;
```

- [ ] **Step 2: Fix `handleSkip`**

Find:
```javascript
  // Clean up block-input and in-focus
  const feed = document.getElementById(`feed-${chapterIdx}`);
  const bi = group.querySelector('.block-input');
  if (bi) bi.remove();
  group.classList.remove('in-focus');
  activeBlockInput = null;
```

Replace with:
```javascript
  // Clean up sticky-bottom and in-focus
  const sb = group.querySelector('.sticky-bottom');
  if (sb) sb.remove();
  group.classList.remove('in-focus');
  activeBlockInput = null;
```

- [ ] **Step 3: Fix `handleFreeTextAnswer` (correct branch)**

Find:
```javascript
    // Clean up block-input and in-focus
    const bi = group.querySelector('.block-input');
    if (bi) bi.remove();
    group.classList.remove('in-focus');
    activeBlockInput = null;
    group.classList.add('collapsed');
```

Replace with:
```javascript
    // Clean up sticky-bottom and in-focus
    const sb = group.querySelector('.sticky-bottom');
    if (sb) sb.remove();
    group.classList.remove('in-focus');
    activeBlockInput = null;
    group.classList.add('collapsed');
```

- [ ] **Step 4: Verify**

Answer a question correctly. The group should collapse cleanly. Skip a question — same. Answer a freetext question correctly — same.

---

### Task 6: Restructure `appendMoMessage` — scroll-zone + sticky-bottom for content blocks

**Files:**
- Modify: `index.html` — `appendMoMessage` function, the `if (label)` branch

- [ ] **Step 1: Wrap msg-body in scroll-zone; put block-input in sticky-bottom**

Find the entire `if (label)` branch in `appendMoMessage`:
```javascript
  if (label) {
    msg.className = `msg content-block ${kind}`;
    const labelEl = document.createElement('span');
    labelEl.className = 'content-block-label';
    labelEl.textContent = label;
    const body = document.createElement('div');
    body.className = 'msg-body';
    msg.appendChild(labelEl);
    msg.appendChild(body);
    feed.appendChild(msg);
    contentIn(msg);
    setActive(feed, msg);
    if (willPause) setBlockFocus(feed, msg);
    return typeInto(body, text).then(() => {
      setActive(feed, msg);
      if (!willPause) return;

      // Attach inline chat container
      const chatEl = document.createElement('div');
      chatEl.className = 'material-chat';
      chatEl.style.display = 'none'; // hidden until first message
      msg.appendChild(chatEl);
      activeMaterialBlock = { feedId, chatEl };

      // Add block-input with textarea for asking Mo questions
      const blockInput = createBlockInput('Ask Mo anything…', async (raw) => {
        if (!activeMaterialBlock) return;
        const { chatEl: ce } = activeMaterialBlock;
        if (ce.style.display === 'none') ce.style.display = 'flex';

        const userBubble = document.createElement('div');
        userBubble.className = 'q-user-bubble';
        userBubble.textContent = raw;
        ce.appendChild(userBubble);
        bounceIn(userBubble);
        ce.scrollIntoView({ behavior: 'smooth', block: 'nearest' });

        await delay(350);

        const reply = MO_CHAT_REPLIES[Math.floor(Math.random() * MO_CHAT_REPLIES.length)];
        const replyEl = document.createElement('div');
        replyEl.className = 'material-chat-reply';
        ce.appendChild(replyEl);
        bounceIn(replyEl);
        await typeInto(replyEl, reply);
        ce.scrollIntoView({ behavior: 'smooth', block: 'nearest' });
      });
      const { el: biEl } = blockInput;
      msg.appendChild(biEl);
    });
  }
```

Replace with:
```javascript
  if (label) {
    msg.className = `msg content-block ${kind}`;
    const labelEl = document.createElement('span');
    labelEl.className = 'content-block-label';
    labelEl.textContent = label;
    const body = document.createElement('div');
    body.className = 'msg-body';

    // scroll-zone wraps body (+ chat when it appears)
    const scrollZone = document.createElement('div');
    scrollZone.className = 'scroll-zone';
    scrollZone.appendChild(body);

    msg.appendChild(labelEl);
    msg.appendChild(scrollZone);
    feed.appendChild(msg);
    contentIn(msg);
    setActive(feed, msg);
    if (willPause) setBlockFocus(feed, msg);
    return typeInto(body, text).then(() => {
      setActive(feed, msg);
      if (!willPause) return;

      // Chat container lives inside scroll-zone so it scrolls with the content
      const chatEl = document.createElement('div');
      chatEl.className = 'material-chat';
      chatEl.style.display = 'none'; // hidden until first message
      scrollZone.appendChild(chatEl);
      activeMaterialBlock = { feedId, chatEl, scrollZone };

      // Block-input goes in sticky-bottom — always visible
      const stickyBottom = document.createElement('div');
      stickyBottom.className = 'sticky-bottom';
      const blockInput = createBlockInput('Ask Mo anything…', async (raw) => {
        if (!activeMaterialBlock) return;
        const { chatEl: ce, scrollZone: sz } = activeMaterialBlock;
        if (ce.style.display === 'none') ce.style.display = 'flex';

        const userBubble = document.createElement('div');
        userBubble.className = 'q-user-bubble';
        userBubble.textContent = raw;
        ce.appendChild(userBubble);
        bounceIn(userBubble);
        sz.scrollTop = sz.scrollHeight;

        await delay(350);

        const reply = MO_CHAT_REPLIES[Math.floor(Math.random() * MO_CHAT_REPLIES.length)];
        const replyEl = document.createElement('div');
        replyEl.className = 'material-chat-reply';
        ce.appendChild(replyEl);
        bounceIn(replyEl);
        await typeInto(replyEl, reply);
        sz.scrollTop = sz.scrollHeight;
      });
      const { el: biEl } = blockInput;
      stickyBottom.appendChild(biEl);
      msg.appendChild(stickyBottom);
    });
  }
```

- [ ] **Step 2: Verify**

The Overview block (first step, Chapter I) should show its text. When it pauses for user interaction, the text input should appear pinned at the bottom. Typing a question and getting a reply should scroll the content area upward while the input stays fixed.

---

### Task 7: Fix `awaitMaterialNext` — append `↓` button into sticky-bottom

**Files:**
- Modify: `index.html` — `awaitMaterialNext` function

- [ ] **Step 1: Replace the function**

Find the entire `awaitMaterialNext` function:
```javascript
function awaitMaterialNext(feedId) {
  // Find the current in-focus block and render a continue button below it in the feed
  const feed = document.getElementById(feedId);
  const focusedBlock = feed.querySelector('.in-focus');

  const btn = document.createElement('button');
  btn.className = 'material-next-btn';
  btn.innerHTML = '↓';

  if (focusedBlock) {
    feed.insertBefore(btn, focusedBlock.nextSibling);
  } else {
    feed.appendChild(btn);
  }

  bounceIn(btn);
  setTimeout(() => btn.scrollIntoView({ behavior: 'smooth', block: 'nearest' }), 100);
  return new Promise(resolve => {
    btn.addEventListener('click', () => {
      // Clean up: remove in-focus, clear activeMaterialBlock
      if (focusedBlock) {
        focusedBlock.classList.remove('in-focus');
        const bi = focusedBlock.querySelector('.block-input');
        if (bi) bi.remove();
      }
      btn.remove();
      activeMaterialBlock = null;
      activeBlockInput = null;
      resolve();
    }, { once: true });
  });
}
```

Replace with:
```javascript
function awaitMaterialNext(feedId) {
  const feed = document.getElementById(feedId);
  const focusedBlock = feed.querySelector('.in-focus');
  const stickyBottom = focusedBlock && focusedBlock.querySelector('.sticky-bottom');

  const btn = document.createElement('button');
  btn.className = 'material-next-btn';
  btn.innerHTML = '↓';

  // Wrap in a centered row and append inside sticky-bottom (never pushed off screen)
  const btnRow = document.createElement('div');
  btnRow.style.cssText = 'display:flex;justify-content:center;padding-top:8px';
  btnRow.appendChild(btn);

  if (stickyBottom) {
    stickyBottom.appendChild(btnRow);
  } else if (focusedBlock) {
    focusedBlock.appendChild(btnRow);
  } else {
    feed.appendChild(btnRow);
  }

  bounceIn(btn);
  return new Promise(resolve => {
    btn.addEventListener('click', () => {
      if (focusedBlock) {
        focusedBlock.classList.remove('in-focus');
        const sb = focusedBlock.querySelector('.sticky-bottom');
        if (sb) sb.remove();
      } else {
        btnRow.remove();
      }
      activeMaterialBlock = null;
      activeBlockInput = null;
      resolve();
    }, { once: true });
  });
}
```

- [ ] **Step 2: Verify**

The `↓` continue button should appear inside the block, below the chat input. Clicking it should advance to the next step. The button should never float outside the card boundary.

---

### Task 8: Simplify `setActive` — remove the scroll calculation

Now that the block fills the card and centering is done by CSS, the scroll-to-center logic in `setActive` is no longer needed.

**Files:**
- Modify: `index.html` — `setActive` function

- [ ] **Step 1: Replace `setActive`**

Find:
```javascript
function setActive(feed, el) {
  // Dim all siblings, keep el bright
  Array.from(feed.children).forEach(c => c.classList.toggle('past', c !== el));
  // If block fills the feed (in-focus), scroll its top into view with padding.
  // Otherwise center it.
  const feedRect = feed.getBoundingClientRect();
  const elRect = el.getBoundingClientRect();
  const elTopInFeed = elRect.top - feedRect.top + feed.scrollTop;
  const scrollTarget = el.offsetHeight >= feed.clientHeight * 0.6
    ? elTopInFeed - 28
    : elTopInFeed - (feed.clientHeight - el.offsetHeight) / 2;
  feed.scrollTo({ top: Math.max(0, scrollTarget), behavior: 'smooth' });
}
```

Replace with:
```javascript
function setActive(feed, el) {
  Array.from(feed.children).forEach(c => c.classList.toggle('past', c !== el));
}
```

- [ ] **Step 2: Verify**

Page still works. Dimming of past blocks still happens correctly.

---

### Task 9: Remove `centerFocusedBlock`

**Files:**
- Modify: `index.html` — remove dead function

- [ ] **Step 1: Delete the function**

Find and delete:
```javascript
// Scroll feed so the question content is centered
function centerFocusedBlock(feed, group, contentEl) {
  requestAnimationFrame(() => requestAnimationFrame(() => {
    contentEl.scrollIntoView({ behavior: 'smooth', block: 'center' });
  }));
}
```

- [ ] **Step 2: Verify no call sites remain**

Search for `centerFocusedBlock` in the file — should return zero results.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: block fills card, content scrolls, input pinned to sticky footer"
```

---

## Self-Review

**Spec coverage check:**

| Requirement | Task |
|---|---|
| Block in focus appears centered | Task 2 — `margin: auto 0` on `.q-group.in-focus .q-full` |
| Content growing → internal scroll | Tasks 2, 6 — `.question-block` / `.scroll-zone` become `overflow-y: auto` |
| Chat input never scrolls away | Tasks 3, 4, 6 — moved into `.sticky-bottom` (`flex-shrink: 0`) |
| `↓` button never pushed off screen | Task 7 — appended inside `.sticky-bottom`, not as feed sibling |
| Content block in-focus fills card | Task 1 — `.feed > .in-focus.content-block { flex: 1 }` |
| Cleanup on answer/skip/continue | Tasks 5, 7 — target `.sticky-bottom` not `.block-input` |
| Remove dead centering code | Tasks 8, 9 — `setActive` simplified, `centerFocusedBlock` deleted |

No gaps found. No placeholders. Type/selector names are consistent across all tasks.
