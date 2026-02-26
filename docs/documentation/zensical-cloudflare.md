---
icon: simple/cloudflare
---

# Hosting Zensical on Cloudflare Pages

## 1. Init your fresh Zensical project locally

```bash
pip install zensical
zensical new my-docs
cd my-docs
git init
git remote add origin https://github.com/yourusername/my-docs.git
git push -u origin main
```

### 2. Connect to Cloudflare Pages (one-time)

1. Cloudflare Dashboard → **Workers & Pages → Create application → Pages → Connect to Git**
2. Authorize GitHub, select your repo
3. Set build settings:
   - **Framework preset:** None
   - **Build command:** `pip install zensical && zensical build --clean`
   - **Build output directory:** `site`
4. Click **Save and Deploy**

### 3. Add your custom domain

Once the first deploy succeeds, go to your Pages project → **Custom domains → Set up a custom domain** → enter `docs.mydomain.me`. Since your domain is already on Cloudflare, it adds the DNS record for you automatically.
