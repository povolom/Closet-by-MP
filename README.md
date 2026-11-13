# Closet by MP

A closet that builds itself: one mirror photo a day, and the app finds each piece of clothing, works out whether it has seen it before, and keeps track of what I actually wear.

**Status:** coming soon at https://closet.marcantoniopovolo.com · [About the project](https://marcantoniopovolo.com/projects/closet) · Open source (MIT)

Built by [Marcantonio Povolo](https://marcantoniopovolo.com), a Computer Engineering student at Toronto Metropolitan University. I decide what it does and review and test each change; AI coding tools help write the code.

## Rules

It costs nothing, photos never leave the device, faces are never stored, and no typing is required.

## How I'm building it

1. Test the free on-device models on a phone first.
2. Build the pipeline on sample data, with tests.
3. Real models, a short inbox of yes or no questions, then install as an app.

The full plan is in [SPEC.md](SPEC.md).

## Layout

| Path | What it is |
|---|---|
| `SPEC.md` | The full build plan, milestone by milestone |
| `site/` | The website at closet.marcantoniopovolo.com: the coming-soon page (`index.html`, `style.css`), icons and favicons. `about/` and `_redirects` send the old project page to [marcantoniopovolo.com/projects/closet.html](https://marcantoniopovolo.com/projects/closet) |
| `worker.js` | The API: anything under `/api/` (for now `/api/status`). Everything else is served from `site/` |
| `wrangler.jsonc` | Cloudflare settings: `site/` plus `worker.js` at closet.marcantoniopovolo.com |
| `LICENSE` | MIT: free to use, change and share |
| `AGENTS.md` | Rules for AI coding tools working in this repo |

## Publishing

Pushing to GitHub saves the code. To put `site/` online at closet.marcantoniopovolo.com, run this in the repo folder:

```
npx wrangler deploy
```

## API

The app and its API share one address. The website is served from `site/`; requests under `/api/` go to `worker.js`. While the app is coming soon there is one endpoint:

```
curl https://closet.marcantoniopovolo.com/api/status
```

## Open source

This project is open source under the [MIT License](LICENSE). Fork it, change it and make your own version; just keep the license notice. Issues and pull requests are welcome.
