## 2026-04-24 - Enhancing Theme Toggle Accessibility & Delight
**Learning:** For icon-only buttons like theme switchers, `aria-label` and `aria-pressed` must be dynamically updated in sync with the state to provide a consistent experience for screen reader users. Additionally, using `:focus-visible` allows for a more prominent focus indicator for keyboard users without cluttering the UI for mouse users.
**Action:** Always verify that state-changing icon buttons reflect their current and future state via ARIA attributes, and prioritize `:focus-visible` for accessible focus states.
