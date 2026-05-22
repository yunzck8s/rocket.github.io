## 2026-05-15 - Improving global interaction and accessibility

**Learning:** Using `*:focus-visible` instead of `*:focus` provides a better experience by only showing focus rings for keyboard users, reducing visual noise for mouse users. Implementing a global `.transition` class managed by JS allows for smooth theme switching without interfering with normal component transitions.

**Action:** Always prefer `focus-visible` for focus indicators and use the `.transition` utility pattern for global state changes that affect many components.

## 2024-05-18 - Accessibility for Navigation and Theming

**Learning:** "Skip to main content" links are essential for keyboard users to bypass repetitive navigation. Additionally, interactive elements like theme toggles must have dynamic `aria-label` and `title` attributes that reflect the *action* they will perform next, and the site should respect system-level color preferences automatically if no user preference is stored.

**Action:** Implement "Skip to content" links in all global layouts and ensure all stateful icon-only buttons update their ARIA labels dynamically.
