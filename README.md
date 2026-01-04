# Routes Documentation

This repository contains documentation for Routes - Background Location Tracking implementation.

## 🌐 Live Documentation

Visit the live documentation at: **https://amrat98.github.io/routes-docs/**

## 📚 Documentation

- [Background Mode Quick Start](./BACKGROUND_MODE_QUICK_START.md) - Implementation guide for background location tracking

## 🚀 Local Development

To run the documentation site locally:

1. Install Jekyll:
   ```bash
   gem install bundler jekyll
   ```

2. Create a Gemfile (optional):
   ```bash
   bundle init
   bundle add jekyll
   ```

3. Serve the site locally:
   ```bash
   jekyll serve
   ```

4. Visit `http://localhost:4000/routes-docs/` in your browser

## 📝 Adding New Documentation

1. Create a new `.md` file in the repository
2. Add front matter at the top:
   ```yaml
   ---
   layout: default
   title: Your Page Title
   ---
   ```
3. Add a link to the new page in `index.md`
4. Commit and push to GitHub

## 🔧 GitHub Pages Setup

This site uses GitHub Pages with Jekyll. The workflow is automatically triggered on push to the `main` branch.

### Configuration Files

- `_config.yml` - Jekyll configuration
- `.github/workflows/jekyll-gh-pages.yml` - GitHub Actions workflow
- `index.md` - Homepage

## 📦 Theme

This site uses the `jekyll-theme-cayman` theme. You can change it in `_config.yml`.

Available themes:
- jekyll-theme-cayman
- jekyll-theme-minimal
- jekyll-theme-slate
- jekyll-theme-architect
- And more...

## 🤝 Contributing

To contribute to the documentation:

1. Fork the repository
2. Create a new branch
3. Make your changes
4. Submit a pull request

---

*Built with ❤️ using Jekyll and GitHub Pages*
