Nice — the shape of this is solid and the audience (tab hoarders) is real. Pocket's wind-down left a gap, and "AI tags my mess for me" is a sharp wedge against Raindrop/Pinboard/Linkding. A few things worth pushing on before you start writing code.

## Product gaps to close before you build

**Define what "tag" means.** Free-form tags from an LLM drift fast — you'll get `js`, `javascript`, `JavaScript`, `JS-language` across three weeks. Decide upfront: does the model pick from a controlled vocabulary that grows over time (you maintain a tag table and ask the model to reuse existing tags where appropriate), or is everything free-form and you do clustering later? The first option gives much better search; the second is easier to ship. I'd start with "controlled, but the model can propose new tags" — pass the user's top-N existing tags into the prompt.

**Capture more than tags.** While you have the page open, also store: a one-line summary, a longer description, estimated reading time, primary language, and a content type (article / video / repo / docs / paper / tool / other). Search becomes much better when you can full-text the summary, not just the title. Costs barely change since you're already paying for the fetch + token in.

**Snapshot at ingest.** Links rot. At minimum, capture title, meta description, og:image, and the extracted main text at the moment you add it. Optionally archive to the Wayback Machine via their save API. That's the single biggest reason power users pay for Pinboard.

**Canonicalize URLs.** Strip UTM/tracking params, normalize `www.`/trailing slash, dedupe. Otherwise the same article saved from Twitter and from a newsletter will be two entries.

**Plan for unfetchable pages.** Paywalls, login walls, JS-only SPAs. You need: (1) a real-browser-like user-agent + a Readability pass before sending to the LLM, and (2) a graceful "couldn't fetch — tag from URL/title only" fallback. Mozilla Readability or Trafilatura before the LLM saves a lot of tokens.

## Tech stack — things I'd reconsider

**Background jobs.** BullMQ is great, but it forces Redis into a stack that's otherwise zero-dependency. For "small footprint" goals, do an in-process SQLite-backed queue: a `jobs` table with `status`, `attempts`, `next_run_at`, and a worker loop in the same Bun process. You get persistence, retries, and visibility for free, no extra container. Reach for BullMQ only if you actually outgrow it.

**OpenRouter is right, but pick a cheap default model.** Tagging + summarizing is squarely in Haiku / GPT-4o-mini / Gemini Flash territory. Don't burn frontier-model tokens on this. Also: budget caps per user (or globally) so a 5000-tab import doesn't surprise you.

**PocketID is heavy for v1.** It's an OIDC IdP — you'd be running another service to log into your bookmark app. If you actually want SSO across self-hosted apps (Authelia / Authentik / PocketID world), keep it. Otherwise start with something embedded — Lucia, better-auth, or just sessions in the DB — and add OIDC later as an option. Self-hosters appreciate "single binary, single DB" more than "ships with an IdP."

**TanStack Start is still pre-1.0.** Fine for a personal project, but expect occasional breaking changes. If you'd rather not, Remix/React Router v7 or Next.js are safer; Astro + a few React islands is also a great fit for something this read-heavy.

**SQLite in Docker = volume + backup story.** Mount a named volume, and add Litestream from day one to replicate to S3/R2. Otherwise one bad `docker compose down -v` and your hoard is gone.

**Chrome extension → cross-browser extension.** Use [WXT](https://wxt.dev/) or Plasmo so the same codebase ships to Chrome, Firefox, and Edge. Manifest V3 from the start.

## Things missing from the spec

- Rate limits and cost caps on the AI side, with a visible per-user budget.
- An admin/maintenance view: failed jobs, broken links, retry button. Tiny effort, huge sanity.
- Structured logging and a `/healthz` for the Docker side.
- Re-tagging — when you improve the prompt, you'll want to re-process old links. Make the job runner idempotent and re-runnable.
- Public/shared lists (later) — this is the lightweight "social" feature that makes hoarders feel less alone and is great viral surface.
- Mobile experience — at minimum a good iOS Share Sheet path. A PWA + the Web Share Target API gets you most of the way without a native app.

## On the names

`bookm` feels like a placeholder. `linky` is generic and crowded. `hoardy` is the most memorable and leans into the audience — but check the namespace; there's at least one tab-management tool using a similar name. If `hoardy` is taken, "Hoardly" / "Stash" / "Squirrel" are in the same vein.

## What I'd actually build first

A two-week MVP: paste-a-URL + bulk Netscape-bookmark-HTML import → Readability fetch → cheap LLM does `{summary, tags[], content_type}` → SQLite + FTS5 for search → single-user, no auth → Docker compose with a Litestream sidecar. No queue lib, no Redis, no extension, no PocketID. Get the tagging quality right first — that's the whole product. Layer the extension, multi-user/auth, and snapshots after you've used it on your own tabs for a week and felt where it hurts.
