# UI and Visual Testing — V1.0

## Scope — load this standard when the task touches:

- Frontend pages or components
- Layout, spacing, typography, or color changes
- Loading, error, or empty states
- Form interactions, modals, or notifications
- Responsive breakpoints

## Required checks per UI change

### Component changes

- [ ] Render in isolation (Storybook or direct render test if available)
- [ ] Verify: default, loading, error, empty states all render without crash
- [ ] Verify: keyboard accessible (tab order, focus rings, Enter/Space activation)
- [ ] Verify: renders at 375px (mobile) and 1280px (desktop) without overflow or clipping

### Page / flow changes

- [ ] Walk the changed user flow end-to-end in a browser
- [ ] Golden path completes without JS error or layout break
- [ ] Error states surface visibly — no silent failure
- [ ] No overflow or clipping at 375px and 1280px

### Dynamic or user-generated content rendering

- [ ] Long content does not break layout
- [ ] Special characters and international text render correctly
- [ ] Status labels and state indicators are accurate

## Screenshot / visual diff

- MVP: manual screenshot comparison before/after is acceptable.
- Phase 2: automate with Playwright screenshot diff or Chromatic.
- When taking screenshots: capture both mobile and desktop for any layout change.

## Accessibility minimum

- All interactive elements keyboard-reachable.
- All images have meaningful alt text.
- Body text meets WCAG AA contrast.
- Form inputs have associated labels.

## When this standard does NOT apply

- Logic-only changes (service, worker, utility) with no UI touch
- Type-only or config-only changes with no rendered output

## Agent note

AI agents do not have browser access. Any task with UI changes must include this item in DONE WHEN:

```
- [ ] Verified in browser at 375px and 1280px (executor must do this manually — agent cannot)
```