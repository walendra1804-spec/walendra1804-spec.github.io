# panca-articles

Personal article site of Panca Walendra Putra, co-written with Claude Opus 5.5.
Hosted on GitHub Pages (Jekyll, `minima` theme): https://walendra1804-spec.github.io

## Add a new article
1. Create `_posts/YYYY-MM-DD-short-title.md` with front matter:
   ```yaml
   ---
   title: "Article title"
   description: "One or two sentences (used by Google and AI search)."
   date: 2026-10-30 13:00:00 +0700
   tags: [tag1, tag2]
   ---
   ```
2. Write the article in Markdown below it. Remove private details (account numbers, usernames, emails).
3. `git add . && git commit -m "Add: <title>" && git push`. GitHub Pages rebuilds in about 1 minute.

## Indexing
- `sitemap.xml` and `feed.xml` are generated automatically.
- `robots.txt` allows search engines and AI crawlers (GPTBot, ClaudeBot, PerplexityBot, …).
- `llms.txt` lists every article for AI tools.
- Submit `https://walendra1804-spec.github.io/sitemap.xml` once in Google Search Console.
