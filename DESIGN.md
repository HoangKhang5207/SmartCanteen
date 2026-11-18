# Project Design System — Smart Canteen

## Source of Truth
- Brand asset: `canteen_logo.jpg` (`miniapp/src/static/canteen-logo.jpg`) — Fresh Leaf Green cutlery (`#5DBE46`) with bold Charcoal ink script `"Canteen"` (`#111827`) on pure white (`#FFFFFF`).
- Approved spec: `docs/superpowers/specs/2026-11-14-smart-canteen-design.md`.
- Existing design tokens and components: Zalo Mini App UI (`zmp-ui`) + Tailwind CSS v3.4 utility classes (`brand` scale).

## Visual Tokens
- **Typography:** System UI stack (`Inter`, `-apple-system`, `Roboto`, sans-serif) with clear mobile hierarchy (`text-xs` 12px, `text-sm` 14px, `text-base` 16px, `text-lg` 18px, `text-xl` 20px).
- **Colors (`brand` scale & semantic tokens):**
  - `brand-50` (`#EEF8EB`): Soft leaf tint for active surfaces, banners, and subtle highlights.
  - `brand-100` (`#D8F0D2`): Soft border/badge background for brand accents.
  - `brand-500` (`#5DBE46`): Exact Logo Leaf Green — used for brand accents, decorative highlights, and icon badges.
  - `brand-600` (`#3FA32B`): Mid green for borders and secondary interactive states.
  - `brand-700` (`#2E7D32`): **Primary Interactive Color** (CTAs, active filter chips, price callouts, focus rings, `--zmp-primary-color`). Provides **5.14:1** contrast ratio against `#FFFFFF` (exceeds WCAG 2.1 AA 4.5:1 requirement).
  - `brand-800` (`#236B27`): Hover/active pressed state for primary buttons.
  - `brand-900` (`#14532D`): Deep forest green for high-emphasis headings and badges.
  - **Surfaces & Neutrals:**
    - Page background: `#F6F8F5` (`surface-muted` — warm sage-tinted off-white).
    - Card surface: `#FFFFFF` with `border-gray-200` (`#E2E8DF`).
    - Primary text: `#111827` (`gray-900`, matching logo lettering).
    - Secondary text: `#4B5563` (`gray-600`).
  - **Order Status Badges:**
    - `PENDING`: `bg-amber-100 text-amber-800` ("Chờ xác nhận")
    - `PREPARING`: `bg-blue-100 text-blue-800` ("Đang chế biến")
    - `READY`: `bg-brand-100 text-brand-900` ("Sẵn sàng nhận món")
    - `COMPLETED`: `bg-gray-100 text-gray-700` ("Đã hoàn thành")
    - `CANCELLED`: `bg-red-100 text-red-800` ("Đã hủy")
- **Spacing:** 4px (`0.25rem`) scale (`p-3` 12px, `p-4` 16px, `gap-2` 8px, `gap-3` 12px).
- **Radius:** `rounded-md` (6px) thumbnails, `rounded-lg` (8px) cards, `rounded-xl` (12px) hero/summary panels, `rounded-full` category chips and pills.
- **Breakpoints:** Mobile-first (`320px`–`480px`), centered container `max-w-2xl` on larger viewports.

## Component System
- Library: `zmp-ui` (`Page`, `Header`, `Box`, `Text`, `Button`, `Input.Search`, `Spinner`).
- Custom components:
  - `miniapp/src/components/brand-logo.tsx` (`BrandLogo` — renders `canteen-logo.jpg` with accessible `alt` and optional tagline).
  - `miniapp/src/components/product-image.tsx` (`ProductImage` — renders uniform-dimension product photos across `card` `80×80px`, `cart` `56×56px`, and `detail` `aspect-[4/3] max-h-64` with `object-cover object-center`, `loading="lazy"`, and automatic `onError` fallback to `canteen-logo.jpg`).
  - `miniapp/src/components/pagination-bar.tsx` (`PaginationBar` — accessible, mobile-ergonomic pagination bar with `from–to / total` summary, windowed page buttons, `aria-current="page"`, and smooth scroll-to-top).
  - `miniapp/src/components/state-views.tsx` (`LoadingState`, `ErrorState`, `EmptyState`).
  - `miniapp/src/components/bottom-nav.tsx` (`BottomNav` — persistent navigation between Thực đơn, Giỏ hàng, Đơn của tôi).

## Accessibility
- Semantic HTML (`main`, `nav`, `section`, `article`, `ul/li`, `button` with `type="button"`).
- Keyboard navigation & visible focus (`focus:ring-2 focus:ring-brand-700`).
- WCAG 2.1 AA contrast ratios (`#2E7D32` on `#FFFFFF` = 5.14:1; `#14532D` on `#D8F0D2` = 8.4:1).
