# LitSupport — PubMed Research Assistant

Find open-access PubMed articles to support, challenge, or reframe a peer's comment. No API key. No server. Works in any browser.

---

## ✅ Option 1 — Use it instantly (recommended)

After uploading to GitHub Pages (see below), your tool lives at a permanent URL like:

```
https://YOUR-USERNAME.github.io/litsupport/
```

Open that URL in any browser. Done.

---

## 🚀 Deploy to GitHub Pages (free, 2 minutes)

1. **Create a GitHub account** at [github.com](https://github.com) if you don't have one.

2. **Create a new repository**
   - Go to [github.com/new](https://github.com/new)
   - Name it `litsupport`
   - Set it to **Public**
   - Click **Create repository**

3. **Upload the files**
   - Click **Add file → Upload files**
   - Drag both `index.html` and `README.md` into the upload area
   - Click **Commit changes**

4. **Enable GitHub Pages**
   - Go to your repo → **Settings → Pages**
   - Under *Source*, select **Deploy from a branch**
   - Branch: `main`, folder: `/ (root)`
   - Click **Save**

5. **Open your live URL**
   - After ~60 seconds, visit `https://YOUR-USERNAME.github.io/litsupport/`
   - Bookmark it — it works forever.

---

## How it works

- Searches **PubMed** via the official [NCBI E-utilities API](https://www.ncbi.nlm.nih.gov/books/NBK25497/) (NIH)
- Filters for **open-access full-text** articles in PubMed Central only
- Uses NCBI's own **MeSH query translation** for precision matching
- No data leaves your browser except the search query sent to NIH
