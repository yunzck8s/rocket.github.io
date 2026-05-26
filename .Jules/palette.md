## 2026-05-15 - Improving global interaction and accessibility

**Learning:** Using `*:focus-visible` instead of `*:focus` provides a better experience by only showing focus rings for keyboard users, reducing visual noise for mouse users. Implementing a global `.transition` class managed by JS allows for smooth theme switching without interfering with normal component transitions.

**Action:** Always prefer `focus-visible` for focus indicators and use the `.transition` utility pattern for global state changes that affect many components.

## 2026-05-23 - Accessibility: Skip to Content

**Learning:** "Skip to content" links are essential for keyboard and screen reader users to bypass repetitive navigation. They should be the first focusable element in the DOM and use `tabindex="-1"` on the target container to ensure programmatic focus works correctly in all browsers.

**Action:** Implement "Skip to content" links in all multi-page layouts, ensuring the target container has a unique ID and proper `tabindex`.

## 2026-05-26 - Dynamic labels for stateful buttons

**Learning:** Interactive elements that toggle between states (like a theme switcher) should have their `aria-label` and `title` updated via JavaScript to describe the *result* of the next interaction (e.g., "Switch to dark mode" when in light mode). This provides clearer context for screen reader users compared to a static "Toggle Theme" label.

**Action:** When implementing toggle-like components, ensure JS updates accessibility attributes to reflect the target state, while maintaining a sensible static fallback in the HTML.
