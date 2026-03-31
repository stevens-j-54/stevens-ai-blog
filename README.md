# Stevens AI Blog

A blog and tutorial site about building AI agents, written by Stevens. Built with [Astro](https://astro.build) and deployed on [Render](https://render.com).

---

## Local Development

### Prerequisites

- [Node.js](https://nodejs.org) v18 or higher
- npm (comes with Node)

### Setup

```bash
# Clone the repo
git clone https://github.com/stevens-j-54/stevens-ai-blog.git
cd stevens-ai-blog

# Install dependencies
npm install

# Start the dev server
npm run dev
```

The site will be available at `http://localhost:4321`.

### Build

```bash
npm run build
```

This compiles the site to the `dist/` folder. You can preview the built output with:

```bash
npm run preview
```

---

## Project Structure

```
stevens-ai-blog/
├── public/
│   └── styles/
│       └── global.css        # Site-wide styles
├── src/
│   ├── layouts/
│   │   ├── BaseLayout.astro  # Shared HTML shell (nav, header, footer)
│   │   └── PostLayout.astro  # Layout for individual blog posts
│   └── pages/
│       ├── index.astro       # Home page
│       ├── about.astro       # About page
│       └── blog/
│           ├── index.astro   # Blog index (lists all posts)
│           └── *.md          # Individual posts (one file per post)
├── astro.config.mjs          # Astro configuration
└── package.json
```

### Adding a new post

Create a new `.md` file in `src/pages/blog/`. Every post needs this frontmatter at the top:

```markdown
---
layout: ../../layouts/PostLayout.astro
title: "Your Post Title"
date: "2025-06-01"
description: "A short summary of the post."
tags: ["agents", "tutorial"]
---

Your content goes here.
```

Commit and push — the post will appear automatically on the blog index and home page.

---

## Deploying to Render

### First-time setup

1. Log in to [Render](https://render.com)
2. Click **New** → **Static Site**
3. Connect your GitHub account and select the `stevens-j-54/stevens-ai-blog` repository
4. Configure the service:
   - **Build Command**: `npm install && npm run build`
   - **Publish Directory**: `dist`
5. Click **Create Static Site**

Render will build and deploy the site. Once deployed, you'll get an auto-generated URL (e.g. `https://stevens-ai-blog.onrender.com`).

### Update `astro.config.mjs`

Once you have the Render URL, update the `site` field in `astro.config.mjs`:

```js
export default defineConfig({
  site: 'https://stevens-ai-blog.onrender.com', // your actual Render URL
  integrations: [mdx()],
});
```

### Subsequent deploys

Render auto-deploys on every push to `main`. No manual steps needed after initial setup.

---

## Notes

- The placeholder post (`src/pages/blog/placeholder.md`) is marked `draft: true` but will still render in the current setup. Delete it before going live, or once real posts are in place.
- To add a custom domain later: go to Render → your site → **Custom Domains**.
