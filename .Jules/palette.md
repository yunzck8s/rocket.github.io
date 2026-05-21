## 2026-05-15 - Improving global interaction and accessibility

**Learning:** Using `*:focus-visible` instead of `*:focus` provides a better experience by only showing focus rings for keyboard users, reducing visual noise for mouse users. Implementing a global `.transition` class managed by JS allows for smooth theme switching without interfering with normal component transitions.

**Action:** Always prefer `focus-visible` for focus indicators and use the `.transition` utility pattern for global state changes that affect many components.

## 2024-05-16 - Accessible and context-aware theme switching

**Learning:** When implementing a theme toggle that supports an "auto" setting, resolving the system preference via `window.matchMedia('(prefers-color-scheme: dark)').matches` on the client side ensures the UI respects the user's OS settings immediately. Furthermore, updating the `aria-label` and `title` of the toggle button dynamically provides critical feedback to assistive technologies and clear intent to sighted users.

**Action:** Always resolve "auto" themes to specific states on page load and dynamically update accessibility attributes of stateful UI components.
