## 2026-05-15 - Improving global interaction and accessibility

**Learning:** Using `*:focus-visible` instead of `*:focus` provides a better experience by only showing focus rings for keyboard users, reducing visual noise for mouse users. Implementing a global `.transition` class managed by JS allows for smooth theme switching without interfering with normal component transitions.

**Action:** Always prefer `focus-visible` for focus indicators and use the `.transition` utility pattern for global state changes that affect many components.

## 2026-05-16 - i18n-Safe Accessibility for Links
**Learning:** When adding accessibility features to dynamic components (like navigation links where the text is user-configurable), avoid hardcoding ARIA labels in a specific language (e.g., "Back to home"). This creates a regression for internationalized sites. Instead, wrap decorative elements like arrows in `<span aria-hidden="true">` so the screen reader correctly announces only the localized text.

**Action:** For links with icons, hide the icon from screen readers via `aria-hidden="true"` and rely on the text node for context, ensuring the feature remains accessible across all languages.
