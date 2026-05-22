# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Volavue — a static marketing + booking website for a "clarity advisory" service built
around the **Personal Decision Topology** framework. Plain HTML/CSS/JS: no build system,
no package manager, no framework.

## Where the real site lives

There are two parallel copies of the page files, and they are NOT equivalent:

- **`assets/`** — the **canonical, production website**. Hand-authored HTML5: responsive,
  dark mode, relative links. **Always edit these files.**
- **Repository root** (`index.html`, `booking.html`, `backup intake form.html`) — throwaway
  exports from a macOS rich-text app (`<meta name="Generator" content="Cocoa HTML Writer">`).
  They use `p.p1`-style class soup and absolute `file:///Users/mbp4br14r/...` links. These
  are content drafts, not the site. Do not edit them to change the site.

`content.rtf` is the source-of-truth copy + brand brief from the client. `testimonials.csv`
is sample data only (not wired into any page).

The repo root also contains sync-conflict temp files (`.Volavue.zip.bsXBzs`, etc.) and
`Volavue.zip` — ignore these.

## The site (`assets/`)

Three pages, each a self-contained HTML file. All three share one refined editorial-luxury
design system: Fraunces display serif, warm bone/charcoal palette, metallic-gold accents,
hairline rules, kicker section labels.

- `index.html` — homepage. Editorial numbered sections, an **interactive** Personal
  Decision Topology diagram (hover/focus previews a dimension, click locks it), a trust
  row, testimonial carousel.
- `booking.html` — booking page: session overview, Cal.com scheduler, intake-summary
  panel, Stripe payment section.
- `intake.html` — 5-question intake form, linked from the nav on every page. On submit it
  builds `booking.html?name=&email=&q1..q5=` via `URLSearchParams` and redirects.

(The repo root still holds an older `backup intake form.html`; the canonical intake page
is `assets/intake.html`.)

`volavuelogo.png` must sit next to the HTML files; pages reference it with a relative path,
so `assets/` has its own copy.

## Architecture notes

- **No shared stylesheet or script.** Every page inlines its own copy of the design system
  CSS and the theme-toggle / side-menu JS. A change to the design system, nav, or theme
  logic must be applied to **all three files** in `assets/` to stay consistent.
- **Design system** lives in `:root` CSS custom properties (with a `:root.dark` override):
  the base palette plus `--gold`/`--metal` (the metallic-gold gradient that drives the
  sheen), `--line` (hairlines), `--btn`, `--ring`, `--soft`, `--surface-2`, and shadows.
  The full token block is duplicated inline in all three pages — keep them in sync.
- **Progressive enhancement**: an inline `<script>` adds a `js` class to `<html>`;
  scroll-reveal elements are hidden only under `.js`, so content stays visible if scripts
  fail. A `prefers-reduced-motion` block disables the gold sheen and the scroll reveals.
- **Theme** follows `prefers-color-scheme` and a toggle button. It does **not** persist a
  choice to `localStorage`; reloads revert to the OS preference.
- **Booking calendar**: `assets/booking.html` embeds Cal.com via `app.cal.eu`,
  `calLink: "volavuedemotest/15min"` — still a 15-min DEMO event. Replace the namespace,
  calLink and slug with the real Cal account + a 60–90 min event before launch (see the
  `TODO` comment by the embed).
- **Payment**: `booking.html` has a Payment section whose "Pay for your session" button
  has a placeholder `href="#"`. Set it to a real Stripe Payment Link before launch (see
  the `TODO` comment).
- **Intake prefill**: `intake.html` passes answers as URL params; `booking.html` reads
  them with `URLSearchParams`, renders them into the intake-summary panel with
  `textContent` (never `innerHTML` — the values are attacker-controllable), and feeds
  name/email/notes into the Cal embed's `config` object as runtime strings. Never
  interpolate those params into the embed `<script>` source.
- Fonts load from Google Fonts: **Fraunces** (display serif, optical sizing) and
  **Hanken Grotesk** (body).

## Previewing

No build step. Open the files directly, or serve them (relative paths and the Cal embed
behave best over HTTP):

```sh
cd assets && python3 -m http.server 8000   # then visit http://localhost:8000/
```

## Copy & design guardrails (from `content.rtf` — follow when writing any text or UI)

- Tone must stay calm, minimal, precise, grounded. Target reader state: relief and
  steadiness. The client's rule: "If copy increases heart rate, revise."
- **Forbidden:** mystical / spiritual / celestial / tarot language and imagery; urgency
  language; dominance or "power dynamics" framing; hype; emotional manipulation.
- Visual palette stays neutral and warm (charcoal, stone, bone) with a single
  metallic-gold accent. No spiritual imagery.
- The **Personal Decision Topology** framework definition — its five dimensions (Forces at
  Play, Relational Dynamics, Active Pressures, Underlying Structure, Probable Consequence
  Paths) — is canonical. Do not rewrite it without explicit approval.
