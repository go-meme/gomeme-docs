# Setup Guide for GitHub Pages

This guide will help you complete the setup for hosting the documentation at https://docs.go.meme

## ✅ What's Already Done

- ✅ Documentation files created and committed
- ✅ MkDocs configuration with Material theme
- ✅ GitHub Actions workflow for auto-deployment
- ✅ CNAME file for custom domain
- ✅ Repository pushed to GitHub

## 📋 Next Steps

### 1. Enable GitHub Pages

1. Go to your repository: https://github.com/go-meme/gomeme-docs
2. Click **Settings** → **Pages** (in the left sidebar)
3. Under "Build and deployment":
   - **Source**: Select "Deploy from a branch"
   - **Branch**: Select `gh-pages` and `/ (root)`
   - Click **Save**

### 2. Configure Custom Domain in GitHub

1. Still in **Settings** → **Pages**
2. Under "Custom domain":
   - Enter: `docs.go.meme`
   - Click **Save**
3. Wait for DNS check (may take a few minutes)
4. Once verified, check **"Enforce HTTPS"**

### 3. Configure DNS in Cloudflare

1. Go to your Cloudflare dashboard
2. Select your `go.meme` domain
3. Click **DNS** → **Records**
4. Add/Update the following records:

   **Option A: Using CNAME (Recommended)**
   ```
   Type: CNAME
   Name: docs
   Target: go-meme.github.io
   Proxy status: Proxied (orange cloud)
   TTL: Auto
   ```

   **Option B: Using A Records (Alternative)**
   ```
   Type: A
   Name: docs
   IPv4 address: 185.199.108.153
   Proxy status: Proxied
   ```

   Add additional A records for:
   - 185.199.109.153
   - 185.199.110.153
   - 185.199.111.153

5. Save the DNS record

### 4. Wait for Deployment

The GitHub Action will automatically:
1. Build the MkDocs site
2. Deploy to `gh-pages` branch
3. Make it available at your custom domain

**Timeline**:
- GitHub Actions deploy: ~2-5 minutes
- DNS propagation: 5-30 minutes
- SSL certificate: 10-60 minutes

### 5. Verify Deployment

After ~30 minutes, visit:
- https://docs.go.meme (your custom domain)
- https://go-meme.github.io/gomeme-docs (GitHub Pages default)

Both should show your documentation!

## 🔄 Updating Documentation

### Local Development

1. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

2. Run local server:
   ```bash
   mkdocs serve
   ```

   Visit: http://127.0.0.1:8000

3. Make changes to `.md` files

4. Commit and push:
   ```bash
   git add .
   git commit -m "Update documentation"
   git push
   ```

### Auto-Deployment

Every push to `main` branch will automatically:
1. Trigger GitHub Actions
2. Build the site with MkDocs
3. Deploy to GitHub Pages
4. Update https://docs.go.meme

## 📁 Documentation Structure

```
docs/
├── mkdocs.yml              # MkDocs configuration
├── requirements.txt        # Python dependencies
├── CNAME                   # Custom domain file
├── .github/
│   └── workflows/
│       └── deploy.yml      # Auto-deployment workflow
├── stylesheets/
│   └── extra.css           # Custom CSS
├── javascripts/
│   └── mathjax.js          # Math rendering
├── 00-overview.md          # Program overview
├── 01-instructions.md      # Instruction reference
├── 02-state-accounts.md    # Account structures
├── 03-bonding-curve-math.md # Mathematical details
├── 04-fee-system.md        # Fee system
├── 05-workflows.md         # Workflows and examples
├── 06-migration.md         # Migration details
└── README.md               # Documentation hub
```

## 🎨 Customization

### Theme Colors

Edit `mkdocs.yml`:
```yaml
theme:
  palette:
    primary: indigo  # Change to: red, blue, green, etc.
    accent: indigo
```

### Navigation

Edit `mkdocs.yml` under `nav:` section to reorganize pages.

### Logo

Add your logo:
1. Create `docs/assets/logo.png`
2. Update `mkdocs.yml`:
   ```yaml
   theme:
     logo: assets/logo.png
   ```

### Custom CSS

Edit `docs/stylesheets/extra.css` for custom styling.

## 🔍 Features Enabled

- ✅ Material Design theme
- ✅ Dark/Light mode toggle
- ✅ Instant navigation
- ✅ Search functionality
- ✅ Code syntax highlighting
- ✅ Copy code button
- ✅ MathJax for formulas
- ✅ Table of contents
- ✅ Mobile responsive
- ✅ Social links
- ✅ Git repository link

## 🐛 Troubleshooting

### GitHub Pages not showing

1. Check GitHub Actions: https://github.com/go-meme/gomeme-docs/actions
2. Ensure workflow completed successfully
3. Check `gh-pages` branch exists
4. Verify Pages settings point to `gh-pages` branch

### Custom domain not working

1. Check DNS records in Cloudflare
2. Verify CNAME file contains `docs.go.meme`
3. Wait 30-60 minutes for DNS propagation
4. Check GitHub Pages settings shows domain as verified

### Build errors

1. Check GitHub Actions logs
2. Verify `requirements.txt` has correct versions
3. Check `mkdocs.yml` for syntax errors

### HTTPS not working

1. Wait 10-60 minutes for certificate provisioning
2. Ensure "Enforce HTTPS" is checked in GitHub Pages settings
3. Clear browser cache

## 📚 MkDocs Resources

- [MkDocs Documentation](https://www.mkdocs.org/)
- [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/)
- [GitHub Pages Documentation](https://docs.github.com/en/pages)

## ✅ Checklist

- [ ] Enable GitHub Pages in repository settings
- [ ] Configure `gh-pages` branch as source
- [ ] Set custom domain to `docs.go.meme` in GitHub
- [ ] Add CNAME record in Cloudflare DNS
- [ ] Wait for first deployment to complete
- [ ] Verify site accessible at https://docs.go.meme
- [ ] Enable HTTPS enforcement
- [ ] Test local development with `mkdocs serve`

## 🎉 You're Done!

Once all steps are complete, your documentation will be live at:

**🌐 https://docs.go.meme**

Every push to the `main` branch will automatically update the site within minutes!
