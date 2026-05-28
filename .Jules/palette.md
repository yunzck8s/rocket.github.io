## 2026-05-15 - Improving global interaction and accessibility

**Learning:** Using `*:focus-visible` instead of `*:focus` provides a better experience by only showing focus rings for keyboard users, reducing visual noise for mouse users. Implementing a global `.transition` class managed by JS allows for smooth theme switching without interfering with normal component transitions.

**Action:** Always prefer `focus-visible` for focus indicators and use the `.transition` utility pattern for global state changes that affect many components.

## 2026-05-23 - Accessibility: Skip to Content

**Learning:** "Skip to content" links are essential for keyboard and screen reader users to bypass repetitive navigation. They should be the first focusable element in the DOM and use `tabindex="-1"` on the target container to ensure programmatic focus works correctly in all browsers.

**Action:** Implement "Skip to content" links in all multi-page layouts, ensuring the target container has a unique ID and proper `tabindex`.

## 2026-05-28 - Dynamic Interactive Labels and Decorative Hiding

**Learning:** Interactive elements like theme toggles need dynamic `aria-label` updates to reflect the specific action they will perform (e.g., "Switch to light mode"). When an 'auto' theme is used, the JS must resolve the system preference to provide an accurate label. Additionally, wrapping decorative characters (arrows, separators) in `<span aria-hidden="true">` prevents screen reader clutter.

**Action:** Always update ARIA labels on state change for icon-only buttons and hide non-informative decorative symbols.
