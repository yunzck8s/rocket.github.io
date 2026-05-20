## 2026-05-15 - Improving global interaction and accessibility

**Learning:** Using `*:focus-visible` instead of `*:focus` provides a better experience by only showing focus rings for keyboard users, reducing visual noise for mouse users. Implementing a global `.transition` class managed by JS allows for smooth theme switching without interfering with normal component transitions.

**Action:** Always prefer `focus-visible` for focus indicators and use the `.transition` utility pattern for global state changes that affect many components.

## 2026-05-20 - Skip-to-content and decorative symbol handling

**Learning:** Implementing a "Skip to main content" link is a high-impact, low-effort accessibility improvement for keyboard users. Combined with hiding decorative symbols (like arrows) using `aria-hidden="true"`, it significantly cleans up the experience for screen reader users by reducing noise and providing efficient navigation.

**Action:** Include skip links in every multi-page layout and audit for decorative symbols that should be hidden from assistive technology.
