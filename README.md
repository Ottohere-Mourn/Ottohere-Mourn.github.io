# Mourn / 毛锦昊 — blog smoke

Astro 7 static blog prototype. The blog posts live in the `src/content/blog/` collection and are rendered at `/blog/:slug`.

## Local preview

```bash
npm install
npm run dev
```

The development server is available at `http://localhost:4321`.

```bash
npm run build
npm run preview
```

## Editing

- Personal details and placeholder links: `src/config.ts`
- Articles: `src/content/blog/*.md`
- Content schema: `src/content.config.ts`
- Main visual system: `src/styles/global.css`
- Avatar and site icon: `public/avatar.jpg`

## Publish

Push changes to the `master` branch. GitHub Actions will run `npm ci`, `npm run build`, and deploy the generated `dist/` directory to GitHub Pages.

The public site is `https://ottohere-mourn.github.io`.
