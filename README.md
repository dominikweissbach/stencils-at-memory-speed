# Stencils at Memory Speed

Accelerating AngstromCube's Poisson solver on GPUs. Talk by Dominik Weißbach, Jülich Supercomputing Centre.

- **Slides:** https://dominikweissbach.github.io/stencils-at-memory-speed/
- **PDF:** https://dominikweissbach.github.io/stencils-at-memory-speed/stencils-at-memory-speed.pdf

Use the arrow keys or space to step through the slides. Many diagrams animate step by step. Press `o` for a slide overview.

## Run locally

Built with [Slidev](https://sli.dev). Needs Node.js (any recent LTS).

```sh
npm ci
npm run dev        # http://localhost:3030
npm run export     # PDF (first run: npx playwright install chromium)
```

Every push to `main` builds the site and the PDF and deploys both to GitHub Pages (`.github/workflows/deploy.yml`).

## Contents

- `slides.md`: all slides.
- `components/`: interactive and step-through diagrams (Vue).
- `layouts/`, `global-top.vue`, `styles/`: the Jülich slide theme.
- `public/figures/`: rendered figures. The plotting scripts and measured data are not part of this repository.
- `public/fonts/`: TeX Gyre Heros (GUST Font License).
