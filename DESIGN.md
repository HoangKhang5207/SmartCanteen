# Project Design System (Documented Assumptions — S1)

## Source of Truth
- Approved spec: `docs/superpowers/specs/2026-11-14-smart-canteen-design.md`.
- Existing design tokens and components: Zalo Mini App UI (`zmp-ui`) + Tailwind CSS v3.4 utility classes.
- Documented assumptions below apply while no Figma specification is provided.

## Visual Tokens (Documented Assumption)
- Typography: System UI stack from ZaUI (`text-xs` 12px, `text-sm` 14px, `text-base` 16px, `text-lg` 18px, `text-xl` 20px).
- Colors:
  - Primary brand/action: ZaUI Blue (`#007aff` / `blue-600`, price accent `blue-700`).
  - Surface: `bg-gray-50` page background, `bg-white` card surface, `border-gray-200` subtle borders.
  - Status badges: Available (`bg-green-100 text-green-800`), Unavailable (`bg-gray-200 text-gray-700`), Error (`text-red-700`).
- Spacing: 4px (`0.25rem`) scale (`p-3` 12px, `p-4` 16px, `gap-2` 8px, `gap-3` 12px).
- Radius: `rounded-md` (6px) thumbnails, `rounded-lg` (8px) cards, `rounded-full` category chips.
- Shadows: Flat bordered cards (`border-gray-200`) for mobile performance inside Zalo WebView.
- Breakpoints: Mobile-first (`320px`–`480px`), centered container `max-w-2xl` on larger viewports.

## Component System
- Existing library: `zmp-ui` (`Page`, `Header`, `Box`, `Text`, `Button`, `Input.Search`, `Spinner`).
- Reusable components: `miniapp/src/components/state-views.tsx` (`LoadingState`, `ErrorState`, `EmptyState`).
- Variants and states: Every data-driven screen implements `Loading`, `Error` (with retry button), `Empty`, and `Success` states.

## Layout Rules
- Container widths: `max-w-2xl mx-auto p-4`.
- Page shell conventions: ZaUI `<Page>` + `<Header>` + semantic `<main>`.

## Accessibility
- Semantic HTML (`main`, `nav`, `section`, `article`, `ul/li`, `button` with `type="button"`).
- Keyboard navigation & visible focus (`focus:ring-2 focus:ring-blue-600`).
- ARIA attributes (`aria-pressed` on filter chips, `role="status"` / `role="alert"` on state views, `alt` on product images).
