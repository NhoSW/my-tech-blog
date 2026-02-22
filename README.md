# OTL - Data Engineering

A bilingual (English/Korean) tech blog focused on data engineering, authored by Seungwoo Noh.

## Tech Stack

- **Static Site Generator:** [Hugo](https://gohugo.io/)
- **Theme:** [PaperMod](https://github.com/adityatelange/hugo-PaperMod)
- **Hosting:** GitHub Pages
- **CI/CD:** GitHub Actions

## Local Development

```bash
# Clone with submodules
git clone --recurse-submodules https://github.com/NhoSW/my-tech-blog.git
cd my-tech-blog

# Run local server
hugo server -D
```

The site will be available at `http://localhost:1313/`.

## Creating a New Post

```bash
hugo new posts/my-new-post.md
```

Edit the generated file under `content/posts/` and set `draft: false` when ready to publish.

## Deployment

Deployment is automatic via GitHub Actions. Pushing to the `main` branch triggers a build and deploy pipeline that publishes the site to GitHub Pages.

## Directory Structure

```
.
├── .github/           # GitHub Actions workflows
├── content/           # Blog posts and pages
│   └── posts/         # Individual blog entries
├── layouts/           # Custom layout overrides
├── static/            # Static assets (images, files)
├── themes/            # Hugo themes (PaperMod)
├── config.yml         # Hugo site configuration
└── README.md
```

## License

TBD
