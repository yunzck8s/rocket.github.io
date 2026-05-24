## 2026-05-15 - Improving global interaction and accessibility

**Learning:** Using `*:focus-visible` instead of `*:focus` provides a better experience by only showing focus rings for keyboard users, reducing visual noise for mouse users. Implementing a global `.transition` class managed by JS allows for smooth theme switching without interfering with normal component transitions.

**Action:** Always prefer `focus-visible` for focus indicators and use the `.transition` utility pattern for global state changes that affect many components.

## 2026-05-24 - Theme Toggle Accessibility and 'Auto' Logic

**Learning:** When implementing a theme toggle for a site that supports an 'auto' (system-preference) setting, the JavaScript must resolve the actual current state (system dark or light) *before* determining the next state to toggle to. This prevents a "dead click" where the state changes internally (e.g., auto -> light) but the visual theme remains the same because the system was already light. Dynamically updating ARIA labels to reflect the *action* of the toggle (e.g., "Switch to dark theme") provides much better feedback than a static "Toggle theme" label.

**Action:** Always resolve the computed theme state in JS when using 'auto' settings and ensure interactive labels reflect the result of the next interaction.
