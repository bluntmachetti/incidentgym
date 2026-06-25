# IncidentGym — Field Log

The public build blog for **IncidentGym** — the gym for incident response (multi-tenant B2B
incident-response training, built for the #H0Hackathon on Vercel + AWS Databases).

- **Live site:** https://bluntmachetti.github.io/incidentgym
- **Product demo:** https://incidentgym.redoubtlabs.dev

A Jekyll "Field Log" — reverse-chronological build dispatches. No theme; custom layouts in
`_layouts/` + `assets/css/blog.css`. The product source code is private; this repo is the blog only.

## Local preview

```bash
bundle install
bundle exec jekyll serve   # → http://localhost:4000/incidentgym/
```

## Deploy (GitHub Pages)

Push to `bluntmachetti/incidentgym` and enable Pages (Settings → Pages → Build from branch `main`,
root). The `baseurl: /incidentgym` in `_config.yml` matches the project-pages path.
