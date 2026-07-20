# Manufacturing Analytics — Power BI Portfolio Site

A static, single-page portfolio site for a Manufacturing Analytics Power BI project. Built with
plain HTML5, CSS3, and vanilla JavaScript — no build step, no framework, no dependencies. Ready
to publish directly on GitHub Pages.

**Live structure:** Hero → Project Overview → Dashboard Features (9 pages) → Manufacturing KPIs
→ Dashboard Gallery → Technologies → About → Contact → Footer.

## Files

```
├── index.html      Page structure & content (edit copy, links, and sample KPI values here)
├── style.css        Design tokens, layout, dark/light theme, responsive rules, animations
├── script.js         Theme toggle, mobile nav, scroll-reveal, animated KPI counters
└── README.md         This file
```

Everything is self-contained: fonts load from Google Fonts CDN, icons are inline SVG (no image
requests), and the dashboard "screenshots" in the Gallery section are drawn with CSS/SVG so the
site works fully offline and needs no image assets to look complete.

## Before you publish — customize these

Search `index.html` for the following placeholders and replace them with your own details:

| Placeholder | Where | Replace with |
|---|---|---|
| `Your Name` | About section `<h2>`, footer copyright | Your actual name |
| `YN` | About avatar initials | Your initials |
| `github.com/yourusername` | Hero CTA, Contact, Footer | Your GitHub profile/repo URL |
| `linkedin.com/in/yourusername` | Contact, Footer | Your LinkedIn URL |
| `you@example.com` | Contact, Footer | Your email address |
| `yourportfolio.com` | Contact section | Your portfolio URL (or remove the button if you don't have one) |
| Sample KPI values (`data-target` attributes in the KPI section) | `#kpis` | Your dashboard's real metrics once connected to live data |

The About section bio is written from the description you gave (AI Systems Designer /
Manufacturing Analytics Developer / Maintenance Technician) — edit the two paragraphs to match
your actual experience and tone.

## Adding your real dashboard screenshots

The Gallery section currently uses CSS-drawn mockup panels (browser-window frames with abstract
chart shapes) as placeholders, so the page looks complete with zero image assets. To swap in real
screenshots once your `.pbix` is built:

1. Export screenshots from Power BI Desktop (File → Export → Export report pages to images, or
   just a browser screenshot of each page) and save them into a new `assets/` folder, e.g.
   `assets/executive-overview.png`.
2. In `index.html`, replace a `<div class="mock-window">...</div>` block with:
   ```html
   <img src="assets/executive-overview.png" alt="Executive Overview dashboard page" class="gallery-img" />
   ```
3. Add a matching `.gallery-img { border-radius: var(--radius-md); border: 1px solid var(--border); }`
   rule in `style.css` if you want it to match the existing card styling.

## Running locally

No build tools needed. Either:

- Open `index.html` directly in a browser, or
- Serve it locally to test exactly as GitHub Pages will (recommended, avoids any local file
  path quirks): `python3 -m http.server 8000` from this folder, then visit
  `http://localhost:8000`.

## Deploying to GitHub Pages

**Option A — User/organization or project site via repo settings (simplest):**

1. Create a new GitHub repository (or use an existing one) and push these files to the root of
   the default branch (`main`):
   ```bash
   git init
   git add index.html style.css script.js README.md
   git commit -m "Manufacturing analytics portfolio site"
   git branch -M main
   git remote add origin https://github.com/yourusername/your-repo-name.git
   git push -u origin main
   ```
2. On GitHub: **Settings → Pages**.
3. Under **Build and deployment → Source**, choose **Deploy from a branch**.
4. Branch: `main`, folder: `/ (root)` → **Save**.
5. GitHub will publish the site at `https://yourusername.github.io/your-repo-name/`
   (or `https://yourusername.github.io/` if the repo is named `yourusername.github.io`).
   The first deploy can take a minute or two.

**Option B — `docs/` folder:** if you'd rather keep the site alongside other repo content, move
these four files into a `docs/` folder and choose folder `/docs` instead of `/ (root)` in step 4.

No further configuration is needed — there's no backend, no environment variables, and no build
step, so once Pages is enabled the site is live.

## Accessibility & performance notes

- Semantic landmarks (`header`, `nav`, `main`, `section`, `footer`) and a "Skip to content" link
  are included for screen-reader and keyboard users.
- All interactive controls (`nav-toggle`, `theme-toggle`) expose `aria-*` state attributes.
- `prefers-reduced-motion` is respected — the ticker, scroll-reveal, and counters all fall back
  to a static state for users who've asked for reduced motion at the OS level.
- No external JS libraries and no render-blocking scripts; the only network requests are two
  Google Fonts stylesheets. To go fully offline-capable, download the font files and self-host
  them, then update the `<link>` tags in `index.html` accordingly.
- Color contrast for both themes targets WCAG AA for body text against its background.

## License

Use and modify freely for your own portfolio. No attribution required.
