# Nahid Hasan · Portfolio

Personal portfolio and blog built with [Hugo](https://gohugo.io/) using the [hugo-coder](https://github.com/luizdepra/hugo-coder) theme.

## 🚀 Quick Start

```bash
# Clone the repo
git clone https://github.com/nahid-devops2021/portfolio-site.git
cd portfolio-site

# Run locally (requires Hugo extended)
hugo server -D
```

Visit `http://localhost:1313/` to see the site.

## 🏗️ Build for Production

```bash
hugo --minify
```

Output goes to the `public/` directory.

## 🌐 Live Sites

| Location | URL |
|----------|-----|
| GitHub Pages | https://nahid-devops2021.github.io/portfolio-site/ |
| Server (internal) | http://192.168.0.43:8081 |

## 📁 Structure

```
├── archetypes/       # Content templates
├── content/          # Site content (about, projects, contact)
├── layouts/          # Custom layout overrides
├── static/           # Static assets (images, etc.)
├── themes/hugo-coder # Theme
└── config.toml       # Site configuration
```

## 🔄 Deployment

- **GitHub Pages**: Auto-deployed via GitHub Actions on push to `main`
- **Server**: Built manually with Hugo, served via nginx Docker container
