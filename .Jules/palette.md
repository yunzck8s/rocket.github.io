## 2026-05-15 - Improving global interaction and accessibility

**Learning:** Using `*:focus-visible` instead of `*:focus` provides a better experience by only showing focus rings for keyboard users, reducing visual noise for mouse users. Implementing a global `.transition` class managed by JS allows for smooth theme switching without interfering with normal component transitions.

**Action:** Always prefer `focus-visible` for focus indicators and use the `.transition` utility pattern for global state changes that affect many components.

## 2024-03-12 - Centralized accessibility hooks

**Learning:** Centralizing accessibility hooks like `id="main-content"` in the base layout ensures consistency across all page types and reduces maintenance overhead compared to per-page implementations.

**Action:** Always look for common wrapper elements in base layouts to anchor global accessibility features.
