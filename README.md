# fernando-cv

Personal CV and blog site built with [Astro](https://astro.build), deployed to GitHub Pages.

**Live**: [fzamperin.github.io](https://fzamperin.github.io/)

## Stack

- **Astro 5** — static site generation, zero JS by default
- **Content Collections** — blog posts in Markdown with frontmatter schema validation
- **@astrojs/sitemap** — automatic sitemap generation
- **GitHub Actions** — CI/CD to GitHub Pages on push to `main`

## Structure

```
src/
├── components/       # Astro components (Hero, Experience, Skills, etc.)
├── content/blog/     # Blog posts as Markdown files
├── data/             # CV content as JSON (experience, education, skills, projects)
├── layouts/          # Base HTML layout with SEO
├── pages/            # index.astro (single-page CV) + blog routes
└── styles/           # Global CSS and design tokens
```

## Development

```bash
npm install
npm run dev        # local dev server at localhost:4321
npm run build      # production build to dist/
npm run preview    # preview the production build
```

## Adding a Blog Post

Create a new `.md` file in `src/content/blog/`:

```markdown
---
title: "Post Title"
date: 2026-01-01
description: "Short description for SEO and previews."
tags: ["tag1", "tag2"]
draft: false
---

Post content here.
```

## Deployment

Pushes to `main` automatically trigger the GitHub Actions workflow at `.github/workflows/deploy.yml`, which builds the site and deploys to GitHub Pages.

## License

[MIT](LICENSE)
