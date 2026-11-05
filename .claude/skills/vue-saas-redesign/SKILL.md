---
name: vue-saas-redesign
description: Redesign a Vue 3 application's UI into a modern SaaS-style interface with a vertical navigation sidebar on the left instead of a top nav bar, consistent spacing, clean card layouts and a polished professional look. Use this skill whenever the user asks to modernize, restyle, redesign or "make it look like a SaaS app", wants a left sidebar or vertical nav replacing a top bar, or wants consistent spacing and card styling across a Vue app, even if they don't say "SaaS".
---

# Vue 3 SaaS-style redesign

Turn a Vue 3 app with a top navigation bar into a SaaS-style layout: a fixed left sidebar, a clean content area, and cards and spacing that look the same on every page. The goal is a restructure plus polish, not a rewrite. Keep behavior, routes, data flow and translations intact.

## Before changing anything

1. Read the app shell (usually `App.vue`) and `main.js` to find the router, the nav links, and what else lives in the header (language switcher, profile menu, filter bar).
2. Read the project's CLAUDE.md for design constraints (palette, "no emojis", chart style). Existing project rules win over this skill's defaults.
3. Skim two or three views to see the current card, table and heading patterns. You are looking for what to unify.
4. If the project has its own conventions for who edits `.vue` files (for example a required subagent), follow them.

Ask the user about choices that change the look and that you cannot read from the code (palette direction, collapsible sidebar, anything surprising). Keep the existing palette unless told otherwise, since a redesign that also recolors everything is harder to review.

## Design tokens first

Define the visual system once, as CSS custom properties on `:root` in the global stylesheet (often the unscoped `<style>` in `App.vue`). Pages then pick up consistent values instead of each inventing their own, which is the usual cause of an inconsistent look.

- Spacing scale on a 4px base: 4, 8, 12, 16, 24, 32, 48. Use only these values for padding, margin and gap.
- Radius: one for controls (6-8px), one for cards (12px).
- Borders: 1px solid in a light neutral. Prefer a border over a heavy shadow; add at most one subtle shadow level for raised surfaces.
- Type scale: page title 24px/600, section title 16px/600, body 14px, caption 12px in a muted color.
- Colors: reuse the app's existing palette as tokens (background, surface, border, text, muted text, accent, and status colors). Do not hardcode hex values in views once tokens exist.

## Layout

Replace the top bar with a two-column shell:

```
<div class="app-shell">
  <aside class="sidebar">  brand, nav links, bottom block </aside>
  <div class="main-column">
    <header class="topbar">  page-level controls (filters, language) </header>
    <main class="content"> <router-view /> </main>
  </div>
</div>
```

- Sidebar: fixed width about 240px, full viewport height, sticky, own scroll if tall. Brand at the top, nav links in the middle, and the profile/user block pinned to the bottom.
- Nav links: one row each, 40px tall, icon plus label, 8px radius. The active route gets a tinted background and stronger text color. Drive active state from the router (`router-link-active` / `$route.path`) rather than hand-set classes. Keep the existing `t()` translation keys for labels so language switching keeps working.
- Icons: inline SVG (24px viewBox, 1.5-2px stroke, `currentColor`) so there is no new dependency. Follow project rules about emojis.
- Content area: flex-grow, padded with the spacing scale, max width around 1400px, centered.
- Items that used to sit in the header (language switcher, filter bar) move to a slim top bar inside the main column or to the sidebar. Keep them one click away and keep their behavior unchanged.
- Responsive: below about 900px, hide the sidebar off-canvas and open it from a menu button as a drawer with a backdrop. Close it on route change.
- Optional when requested: a collapse toggle that shrinks the sidebar to about 72px showing icons only, with labels hidden and a `title` tooltip on each link. Persist the choice in `localStorage` inside try/catch.

## Cards and pages

- One shared card style: surface background, 1px border, 12px radius, 24px padding (16px on small screens), no nested heavy shadows.
- Card header pattern: title left, optional action or filter right, 16px gap below.
- KPI cards: small muted label, large value (28-32px, 600), optional delta in the status colors.
- Page header pattern on every page: title, one-line description in muted text, actions on the right.
- Grids: CSS Grid with `repeat(auto-fit, minmax(240px, 1fr))` and the standard gap, so layouts reflow without media queries.
- Tables: 12px vertical cell padding, muted uppercase header row, a hover row tint, a bottom border per row only.
- Status badges: one pill style, color from the status tokens.
- Replace per-view hardcoded paddings, radii and colors with the tokens. Do this by editing the CSS, not by restructuring the template, unless a page's markup blocks the layout.

## Keep it safe

- Do not change API calls, computed data, route paths, or translation keys. Add new translation keys only if you add new visible text, and add them to every locale file.
- Keep `v-for` keys unique and stable.
- Do not add UI libraries. Plain CSS and inline SVG are enough and keep the diff reviewable.
- Make changes in small passes: tokens, then shell, then shared card and table styles, then per-page cleanup.

## Verify visually

Static checks do not catch layout problems, so look at the result.

1. Make sure the dev server is running, then open each route in a browser and take a screenshot.
2. Check at desktop width (about 1440px) and at phone width (about 390px).
3. Look for: sidebar overlapping content, horizontal scroll, text clipped in nav links, active state wrong after navigating, cards with inconsistent padding, and the filter bar or language switcher unreachable.
4. Check the browser console for errors and warnings after navigating through every page.
5. Fix what you find and re-screenshot. Report anything you could not verify.

## Report

Finish with a short summary: what changed in the shell, what tokens were introduced, which pages were touched, and anything left for the user to decide.
