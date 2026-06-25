# TODO — Modernize portfolio site

Goal: transform the current single long-text page into a modern, interactive,
professional personal-presentation site. This is a **personal portfolio to "sell
myself"** (get hired / freelance work) — I am the product. NO product/SaaS stuff:
no pricing, no product features, no marketing testimonials. Focus = me, my
experience, and clear "hire me / contact" CTAs.


## Decisions (locked in — can still change later)
- **Theme:** Light + Dark with a toggle. Default to system preference (`prefers-color-scheme`),
  remember the user's choice.
- **Accent color:** Warm (orange / amber).
- **Interactivity:** Moderate — must be **fast and responsive** above all. Sticky nav,
  smooth scroll, lightweight scroll-reveal. No heavy JS, no frameworks.
- **Restyle-friendly:** everything driven by CSS variables so colors/fonts are a one-line change.
- [ ] Still open: keep `minima` (currently minima-2.5.2) + custom CSS, OR build a fully
      custom `_layouts/home.html` + `_includes/head.html` for full control.
      Recommended: go custom — minima fights a modern design.
- [ ] Pick font pairing (e.g. Inter for body/headings).

## 1. Structure — break the long page into sections
Turn `index.md` into distinct, navigable sections instead of one scroll of text:
- [ ] **Hero** — name, headline, tagline, profile photo, primary CTAs
      (Hire me / Contact, Download CV), social icons, "Available for hire" badge.
- [ ] **About** — short paragraph (intro text already exists).
- [ ] **Skills** — visual skill cards / grouped tech badges instead of a bullet list.
- [ ] **Experience** — clean timeline / cards (always shown, no click-to-expand —
      keeps it simple & fast). Tech shown as pill badges.
- [ ] **Education & Languages** — compact cards.
- [ ] **Contact** — clear CTA section with email + social links.

## 2. Navigation
- [ ] Sticky top nav with anchor links (About, Skills, Experience, Contact) + Download CV.
- [ ] Smooth scroll to anchors.
- [ ] Mobile hamburger menu.
- [ ] Light/dark toggle button in the nav.
- [ ] Active-section highlighting (scroll spy) — optional, only if it stays cheap.

## 3. Interactivity & motion (keep it light)
- [ ] Scroll-reveal animations (fade/slide in) via IntersectionObserver — cheap, no library.
- [ ] Hover effects on cards/badges.
- [ ] Dark/light mode toggle logic (respect system, persist to localStorage).
- [ ] Subtle hero polish (gradient / warm accent glow). No typed-text or heavy animation.

## 4. Design system / styling
- [ ] Replace `assets/main.scss` with a small design system: CSS variables for
      colors (light + dark), spacing, radius; modern font; consistent cards.
- [ ] Warm accent applied to CTAs, links, highlights.
- [ ] Responsive grid; verify mobile breakpoints.
- [ ] Pill-style tech badges for the "Technologies:" lines.
- [ ] Polish profile image, spacing, typography scale.

## 5. Content/data refactor (optional but cleaner)
- [ ] Move experience entries into `_data/experience.yml` and loop in the layout.
- [ ] Same for skills and languages.

## 6. Quality / housekeeping
- [ ] Keep SEO tags working (`jekyll-seo-tag`).
- [ ] Accessibility: alt text, focus states, keyboard nav, color contrast (both themes).
- [ ] Performance: minimal JS, lazy-load images, no frameworks. Prioritize speed.
- [ ] Test responsive on mobile / tablet / desktop.
- [ ] Run `bundle exec jekyll serve` locally and verify before pushing.
      NOTE: local bundle currently broken — missing gems i18n-1.14.7 &
      concurrent-ruby-1.3.5; run `bundle install` before serving.

## 7. Nice-to-have / stretch
- [ ] Featured work / selected projects section with cards + links.
- [ ] "What I do" / services section (still about me, not a product).
- [ ] Favicon / open-graph preview image refresh.
- [ ] Print-friendly stylesheet so the page doubles as a CV.
