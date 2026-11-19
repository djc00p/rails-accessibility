# ♿ Rails Accessibility

[![ClawHub Skill](https://img.shields.io/badge/ClawHub-Skill-blue)](https://clawhub.ai/djc00p/skills/rails-accessibility) [![Agent Skill](https://img.shields.io/badge/Agent-Skill-blue)](#) [![Rails](https://img.shields.io/badge/Rails-v7+-red.svg)](https://rubyonrails.org/) [![WCAG AA](https://img.shields.io/badge/WCAG-AA-4285F4.svg)](https://www.w3.org/WAI/standards-guidelines/wcag/) [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**Rails Accessibility** is a set of practical accessibility patterns for Rails 7+/8+ apps using Stimulus, Hotwire (Turbo), and Tailwind CSS. Not WCAG theory — actual ERB partials, Stimulus controllers, and CI guardrails you drop in.

---

## 🚀 Quick Start

The axe-core + Capybara + system-spec setup is the one file worth reading end to end on first use — it walks Gemfile additions, `AxeHelper`, Capybara driver config, the smoke test pattern, and the Devise sign-in gotcha.

```ruby
# Gemfile
group :test do
  gem 'axe-core-rspec', '~> 4.10'
  gem 'axe-core-capybara', '~> 4.10'
  gem 'capybara', '~> 3.40'
  gem 'selenium-webdriver', '~> 4.20'
end
```

---

## 🧭 Philosophy

**Catch it in CI, not in production.** The patterns here are useful, but the real win is the axe-core RSpec matcher that runs on every `bin/rspec` — same as any other test failure. If it's not in CI, it doesn't exist.

**Patterns over gems.** These ERB partials and Stimulus controllers are meant to be copied into your app and adapted to your design system. They're not a gem — they're a starting point. Your Tailwind config, error styling, and i18n setup will differ.

**Test what users experience.** System specs with `driven_by(:selenium_headless)` catch real a11y issues that static analysis misses. The axe-core matcher runs in the browser, not on the server.

---

## 📚 Reference Docs

Each reference covers one task. Read the one that matches what you're building — not all six.

| Reference | Covers |
|---|---|
| `references/forms.md` | Accessible form fields, error states, labels, Turbo gotchas |
| `references/modals.md` | Focus trap, `aria-modal`, `role="dialog"`, ESC to close, Turbo Frame modal pattern |
| `references/navigation.md` | Skip links, landmark roles, current-page indicator, mobile menu, breadcrumbs |
| `references/dynamic-content.md` | `aria-live`, Turbo Frame/Stream announcements, flash messages, error summary focus |
| `references/color-contrast.md` | oklch chroma rules, Tailwind shades that fail AA, dark mode |
| `references/testing.md` | axe-core + RSpec + Capybara setup, smoke test pattern, per-rule skips, Devise sign-in gotcha |

### Examples

`examples/modal_controller.js` — Stimulus controller for accessible modals (focus trap, ESC to close, backdrop click, focus return). Drop it straight into `app/javascript/controllers/`.

The accessible field partial, skip link, and full axe system spec live as code blocks inside `references/forms.md`, `references/navigation.md`, and `references/testing.md`, since each is tightly coupled to the explanation around it.

---

## 🧩 Core Patterns

### The `_accessible_field` partial

One partial that renders a label, the input, the error message, and the `aria-describedby` / `aria-invalid` wiring together — so the linkage cannot be forgotten.

```erb
<%= render "shared/accessible_field",
           form: f, field: :email, label: t(".email") %>
```

### Key form rules

1. Every input needs a **visible** label.
2. Error messages must be **linked to the input** via `aria-describedby`.
3. Required fields need **both** a visual and a programmatic indicator.
4. `aria-invalid` on error.

### Accessible modals

Focus trap, `aria-modal`, `role="dialog"`, ESC to close, and focus returned to the trigger on close. The Stimulus controller is in `examples/modal_controller.js`; the ERB template and the Turbo Frame pattern are in `references/modals.md`.

### Accessible navigation

Skip link, landmark roles (`aria-label` on `<nav>` when there is more than one), current-page indicator, mobile hamburger menu, and breadcrumbs.

---

## 🚨 Common Gotchas

### oklch chroma threshold

Tailwind's oklch color system can produce colors that look fine but fail WCAG AA.

> **Keep chroma ≤ 0.03 on text colors over light backgrounds.** Above 0.04 almost certainly fails.

The full rule and the chroma-threshold table live in `references/color-contrast.md`.

### `aria-expanded` without `role="combobox"`

`aria-expanded` on an input signals a combobox pattern, but axe-core requires `role="combobox"` on the same element. Having `aria-expanded` alone is a WCAG violation.

**Fix:** add `role="combobox"` to any input with `aria-expanded`.

### Turbo Frame redirects break outside frame context

When a form inside `<turbo-frame id="modal">` redirects to a URL outside the frame, Turbo generates broken `GET /original-path/undefined` requests. Two related rules:

- Use `data: { turbo: false }` on forms that do full-page navigation (password changes, sign-out).
- **`data-turbo-frame` overrides `data-turbo="false"`** — if both are present, Turbo processes within the frame and ignores the opt-out.

Full patterns in `references/modals.md` and `references/forms.md`.

### System spec `sign_in` failures in headless Chrome

Devise's `sign_in` test helper can silently fail in `driven_by(:selenium_headless)` — the session cookie doesn't transfer to the browser process. Use form-based sign-in instead:

```ruby
visit new_user_session_path
fill_in "Email", with: user.email
fill_in "Password", with: "password"
click_button "Log in"
```

Full diagnosis in `references/testing.md`.

### Accessibility specs break when page copy changes

Screen-reader UX specs often assert on exact page titles, `h1` text, or landmark labels. Change an i18n key or marketing copy and those assertions fail even though the page is still accessible.

**Procedure when changing public-page copy:**

1. Search for the old text in `spec/system/` and `spec/requests/`:
   ```bash
   grep -R -n "old title text" spec/
   ```
2. Update any screen-reader, accessibility, or page-title specs that assert on that exact string.
3. Run the affected accessibility / screen-reader specs before pushing.

> Do not rely on the general request spec suite to catch title/`h1` mismatches — those specs often live in dedicated accessibility files and may not be included in a scoped run.

### Turbo Frame `click_link` race in system specs

`click_link` on a `data: { turbo_frame: "..." }` link returns before the async frame-swap resolves. Capybara's default 2s wait doesn't cover it, so the next `fill_in` runs against a frame that still has `aria-busy="true"` and no body — the failure looks like "field missing" but the controller is fine.

Confirm with a direct rack-test request to the same endpoint (returns the full form), then wait for frame **content** (`have_selector("turbo-frame#modal form", wait: 10)`) instead of the frame element — or switch modal-flow tests to `:rack_test`.

Full pattern in `references/testing.md`.

---

## 📦 Install

```bash
clawhub install rails-accessibility
```

Or clone this repo and add the skill folder to your agent's skills path.

### Repository layout

```
SKILL.md                              # entry point and gotcha index
examples/
  modal_controller.js                 # Stimulus controller for accessible modals
references/
  forms.md                            # form fields, error states, Turbo gotchas
  modals.md                           # focus trap, dialog role, Turbo Frame modal
  navigation.md                       # skip links, landmarks, current-page indicator
  dynamic-content.md                  # aria-live, Turbo announcements, flash messages
  color-contrast.md                   # oklch chroma rules, dark mode, CI enforcement
  testing.md                          # axe-core + RSpec + Capybara setup and patterns
LICENSE                               # MIT-0 / MIT
README.md                             # this file
```

### Versions

Releases track the published ClawHub versions.

| Version | ClawHub |
| --- | --- |
| 1.0.0 | [rails-accessibility v1.0.0](https://clawhub.ai/djc00p/skills/rails-accessibility) |

---

**Published by [@djc00p](https://github.com/djc00p)**
