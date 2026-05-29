## 2026-05-15 - Improving global interaction and accessibility

**Learning:** Using `*:focus-visible` instead of `*:focus` provides a better experience by only showing focus rings for keyboard users, reducing visual noise for mouse users. Implementing a global `.transition` class managed by JS allows for smooth theme switching without interfering with normal component transitions.

**Action:** Always prefer `focus-visible` for focus indicators and use the `.transition` utility pattern for global state changes that affect many components.

## 2026-05-23 - Accessibility: Skip to Content

**Learning:** "Skip to content" links are essential for keyboard and screen reader users to bypass repetitive navigation. They should be the first focusable element in the DOM and use `tabindex="-1"` on the target container to ensure programmatic focus works correctly in all browsers.

**Action:** Implement "Skip to content" links in all multi-page layouts, ensuring the target container has a unique ID and proper `tabindex`.

## 2024-06-05 - Enhancing Interactive Labels and Decorative Hiding

**Learning:** Interactive elements that change state (like theme toggles) must have their ARIA labels and titles updated via JavaScript to provide accurate feedback. Also, wrapping decorative symbols (arrows, separators) in `aria-hidden="true"` significantly improves screen reader clarity by preventing the announcement of non-informative characters.

**Action:** Always implement state-aware label updates for toggle buttons and audit layouts for decorative characters that should be hidden from assistive technology.
