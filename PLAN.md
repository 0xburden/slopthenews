# slopthenews — Implementation Plan

News site where AI models ("authors") each write an article about the same current event from the
same frozen set of RSS sources. Readers react, favorite, comment, share. A public editorial benchmark.

**Rules for the implementer:** do the section 9 tasks (bottom of this file) in order; finish each *Done:* check before
the next. Add no unlisted features. If a library behaves differently from this plan, stop and report.

## 0. Fixed Decisions

### Stack
- TypeScript strict, Node ≥ 22.12, pnpm 10 workspaces.
- Web: Next.js 16 (App Router, Server Components, Server Actions).
- Styling: Tailwind v4 + `@tailwindcss/typography` + Headless UI v2 + `@heroicons/react`; `next-themes` (light/dark/system, class).
- Fonts (`next/font/google`): Newsreader (serif), Inter (sans), JetBrains Mono (mono).
- Articles: Markdown from the model, rendered with `react-markdown` + `remark-gfm` + `rehype-unwrap-images`.
- Shared Markdown rules: `packages/core/src/markdown-policy.ts` (worker + web).
- Markdown validation: `mdast-util-from-markdown`, `micromark-extension-gfm`, `mdast-util-gfm`, `mdast-util-to-string`, `unist-util-visit`.
- DB: PostgreSQL 16 + `pgvector`; Drizzle ORM + drizzle-kit, driver `pg`.
- Auth: Better Auth, email+password only, `admin` plugin.
- Jobs: pg-boss (same Postgres). LLM: `ai` + `@ai-sdk/openai-compatible`. Validation: Zod.
- Embeddings: `@huggingface/transformers`, `Xenova/all-MiniLM-L6-v2` (384 dims), local in worker.
- RSS `rss-parser`; article text `@mozilla/readability` + `linkedom`; slugs `slugify`; run TS with `tsx`; tests Vitest.
- Deploy: Docker Compose on Dokploy: `postgres`, `migrate`, `web`, `worker`.

### Version pins (install `pkg@major`; other majors have different APIs)
Pins are the newest major from ~late 2025 (implementing model's likely cutoff).
| Package | Major | Notes |
|---|---|---|
| `next` | 16 | `pnpm dlx create-next-app@16`; see section 6 notes |
| `react`, `react-dom` | 19 | |
| `tailwindcss`, `@tailwindcss/postcss` | 4 | no `tailwind.config.*` |
| `@headlessui/react` | 2 | names: `TabGroup/TabList/Tab/TabPanels/TabPanel`, `MenuButton/MenuItems/MenuItem`, `PopoverButton/PopoverPanel` |
| `next-themes` | 0.4 | |
| `pg-boss` | 12 | ESM only, `import { PgBoss } from 'pg-boss'`; API in 5.2 |
| `ai` | 5 | not 6/7. `maxOutputTokens` (not `maxTokens`); `usage.inputTokens`/`usage.outputTokens` |
| `@ai-sdk/openai-compatible` | 1 | |
| `better-auth` | 1 | |
| `drizzle-orm` / `drizzle-kit` | 0.4x / 0.3x | |
| `react-markdown` | 10 | |
| `zod` | 4 | `import { z } from 'zod'`; use `z.email()`, `z.url()` |
| `@huggingface/transformers` | 3 | not 4 |
| `pnpm` | 10 | not 11+ |

Root `package.json`: `"packageManager": "pnpm@10.x.x"` (installed version), `"engines": { "node": ">=22.12" }`.

**Out of scope:** blind mode, OAuth, email verification, profile customization, regenerating articles, search, notifications.

## 1. Repository Layout
```
PLAN.md  plan/  package.json  pnpm-workspace.yaml  tsconfig.base.json  .env.example  docker-compose.yml
apps/web      Next.js site + admin (apps/web/.env -> ../../.env symlink)
apps/worker   pg-boss jobs: ingest, cluster, select, build-packet, generate
packages/db   Drizzle schema, client, migrations, seed, make-admin
packages/core crypto, packet/article types, markdown policy, prompt, llm, config
```
- Names: `@stn/web`, `@stn/worker`, `@stn/db`, `@stn/core`.
- Packages are **not built**; they export TS source. `@stn/db`: `"exports": { ".": "./src/index.ts" }`.
  `@stn/core`: `"exports": { ".": "./src/index.ts", "./markdown-policy": "./src/markdown-policy.ts", "./packet": "./src/packet.ts" }`.
- `apps/web/next.config.ts`: `transpilePackages: ['@stn/core','@stn/db']`, `serverExternalPackages: ['pg-boss','@huggingface/transformers','pg']`,
  `output: 'standalone'`, `outputFileTracingRoot: path.join(__dirname, '../..')`.
- `'use client'` files may import only `@stn/core/markdown-policy` and `@stn/core/packet`, never `@stn/core` or `@stn/db`.
- Worker/scripts: `node --env-file=../../.env --import tsx src/<file>.ts` (or `tsx --env-file=../../.env`).
- `packages/core/src/config.ts` reads `process.env` once, validates with Zod, exports typed `config`; nothing else reads
  `process.env`. Missing/invalid vars fail startup with a clear message.

## 2. Environment Variables (`.env.example`)
Comments on their own lines; quote values with spaces.
```
# Postgres (inside docker compose the host is "postgres", not "localhost")
DATABASE_URL=postgres://stn:stn@localhost:5432/stn
POSTGRES_USER=stn
POSTGRES_PASSWORD=stn
POSTGRES_DB=stn

# Better Auth
BETTER_AUTH_SECRET=change-me-at-least-32-characters-long
BETTER_AUTH_URL=http://localhost:3000
PUBLIC_SITE_URL=http://localhost:3000

# Encrypts provider API keys. Generate with: openssl rand -base64 32
# Must decode to exactly 32 bytes (config.ts checks this).
ENCRYPTION_KEY=

# Pipeline
PIPELINE_CRON="0 */4 * * *"
EVENTS_PER_RUN=5
MIN_OUTLETS_PER_EVENT=2
CLUSTER_SIMILARITY=0.72
CLUSTER_WINDOW_HOURS=48

# Embedding model cache (the worker downloads the model here once)
HF_CACHE_DIR=./.models

# Media kill switch (server-side, read at request time; not NEXT_PUBLIC_)
MEDIA_ENABLED=true
```

## Section files
Sections 3–8 live in `plan/`. Read only the files the current task cites.

| § | File |
|---|---|
| 3 Database schema | `plan/3-schema.md` |
| 4 Packets, article parsing, validation | `plan/4-markdown.md` |
| 5 Pipeline, worker, LLM, prompt, feeds | `plan/5-pipeline.md` |
| 6 Web app routes, auth, actions | `plan/6-web.md` |
| 7 Design system, rendering | `plan/7-design.md` |
| 8 Security rules | `plan/8-security.md` |

## 9. Task List
Run `pnpm -r typecheck` and `pnpm -r test` before every commit; commit after each task.

### Phase 1 — Scaffold
- **T1. Workspace.** First `corepack enable && corepack use pnpm@10` in repo root. Root `package.json` (scripts `dev`,
  `typecheck`, `test`; `pnpm.onlyBuiltDependencies: ["onnxruntime-node", "sharp", "esbuild"]`), `pnpm-workspace.yaml`
  (`apps/*`, `packages/*`), `tsconfig.base.json` (strict, `moduleResolution: "bundler"`), `.gitignore` (`node_modules`,
  `.next`, `.env`, `.models`), `.env.example` (2). Create `packages/db`, `packages/core`, `apps/worker` with `package.json`
  (exports per 1), `tsconfig.json`, `src/index.ts`. Add `packages/core/src/config.ts` (1).
  *Done:* `pnpm install` and `pnpm -r typecheck` succeed.
- **T2. Postgres.** `docker-compose.yml` service `postgres` (`pgvector/pgvector:pg16`), env from `.env`
  (`POSTGRES_USER/PASSWORD/DB`), volume `pgdata`, port `5432:5432`, healthcheck `pg_isready -U $POSTGRES_USER`.
  *Done:* `docker compose up -d postgres` is healthy; `psql $DATABASE_URL -c 'select 1'` works.
- **T3. Next.js + design system.** `pnpm dlx create-next-app@16 apps/web` (TS, App Router, Tailwind, `src/`; "No" to ESLint
  and React Compiler). Rename to `@stn/web`. Add `@headlessui/react@2`, `@heroicons/react`, `@tailwindcss/typography`,
  `next-themes@0.4`. Symlink `apps/web/.env` → `../../.env`. `next.config.ts` (1), fonts (7.1), `globals.css` (7.2),
  Providers + ThemeSwitcher (7.4), masthead (7.6) **without user menu** (T5). `/styleguide` page: headings, prose, buttons, author chips.
  *Done:* `/styleguide` right in light+dark, theme survives reload, no wrong-theme flash, no hydration warning.

### Phase 2 — Database
- **T4. Drizzle.** `packages/db`: `src/client.ts` (`db` from `drizzle-orm/node-postgres` with `pg` Pool), `drizzle.config.ts`
  (`schema: './src/schema.ts'`, `out: './drizzle'`, dialect `postgresql`), empty `src/schema.ts`. First migration:
  `pnpm drizzle-kit generate --custom --name=enable_vector`, containing `CREATE EXTENSION IF NOT EXISTS vector;` (must be 0000).
  `src/migrate.ts` using `migrate()` from `drizzle-orm/node-postgres/migrator`; scripts `generate`, `migrate`.
  *Done:* `pnpm --filter @stn/db migrate` succeeds; `\dx` lists `vector`.
- **T5. Better Auth.** `auth.ts`, route handler, `auth-client.ts`, `session.ts` (6.1). Generate tables:
  `pnpm dlx @better-auth/cli generate --config apps/web/src/lib/auth.ts --output packages/db/src/auth-schema.ts`, re-export from
  `schema.ts`, then db `generate` + `migrate`. `make-admin.ts` + script. `/login`, `/signup`, masthead user menu.
  *Done:* sign up/in/out work; `make-admin` grants admin; "Admin" menu item only for admins.
- **T6. Content + interaction schema.** All tables/indexes from 3; `generate` + `migrate`.
  *Done:* migrations apply cleanly on an empty DB (`docker compose down -v`, up, migrate).
- **T7. Seed.** `packages/db/src/seed.ts`: feeds (5.9) + prompt v1 (5.8, `active=true`), `onConflictDoNothing()`. Script `seed`.
  *Done:* running seed twice leaves 10 feeds, 1 template.

### Phase 3 — Core library (`packages/core`, pure functions + Vitest)
- **T8. crypto.ts.** `encrypt(plain)`/`decrypt(stored)`, AES-256-GCM, `config.ENCRYPTION_KEY`.
  *Done:* round-trip passes; tampered ciphertext throws.
- **T9. packet.ts + markdown-policy.ts** (4.1, 4.3) + `test/fixtures/packet.json`.
  *Done:* tests: `normalizeUrl` (utm, hash, trailing slash, host case, http→https, invalid), `isPacketLink`, `resolveMedia`,
  `urlTransform` (keeps `m1`, https; drops `javascript:`, `mailto:`, `/relative`), `findCitations` (ignores `[2](url)`, stable on repeat calls).
- **T10. article.ts + validate.ts** (4.4, 4.5); sample responses in `test/fixtures/*.md`.
  *Done:* tests: valid article; fence-wrapped; `<think>` block; text before headline; `[1, 3]` and `[1-3]`; definition line
  `[1]: https://…`; missing headline; missing dek; dek after blank line; out-of-range citation; low paragraph coverage;
  image-only paragraph (not counted); copied text; valid packet link; invented link; valid media id; unknown media id;
  media URL instead of id; inline image; 4 media; HTML block; `####` heading.
- **T11. prompt.ts** (5.8). *Done:* test renders fixture packet; checks source numbering, `URL:` lines, media lines, `(none)`
  for empty media, no media URL in output.
- **T12. llm.ts** (5.7) + script `try-llm` (`src/scripts/try-llm.ts`): reads `TRY_BASE_URL`, `TRY_API_KEY`, `TRY_MODEL`;
  renders fixture packet with v1 template; `writeArticle`; prints text, then `parseArticle` + `validateArticle` result.
  *Done:* prints a real Venice article + its validation result.

### Phase 4 — Admin
- **T13. Admin layout + providers.** `app/admin/layout.tsx` (sidebar) + providers page (6.3): CRUD, Venice preset, Test
  connection. Verify Venice `include_venice_system_prompt` name in Venice docs.
  *Done:* normal user gets 404 on `/admin` and direct admin action calls; Venice Test lists models; API key absent from page HTML.
- **T14. Authors.** *Done:* create an author with a model from the list; invalid color rejected server-side.
- **T15. Feeds, prompts, runs, comments pages** (6.3; runs without preview).
  *Done:* each lists data; enable/disable, set-active-prompt, hide/unhide work.

### Phase 5 — Worker
- **T16. Bootstrap.** `apps/worker/src/index.ts` + `boss.ts` (5.2), empty logging handlers. Scripts `dev` (`tsx watch`), `start`.
  *Done:* starts, creates queues, logs schedule, stops cleanly on Ctrl-C.
- **T17. Embeddings + media** (5.4, 5.5). Fixtures: RSS from BBC, Guardian, NYT, France 24 + one article page HTML.
  *Done:* `download-model` caches model; test: `embed()` returns 384 numbers with norm ≈ 1; media tests pass.
- **T18. ingest** (5.3). *Done:* fills `raw_article` with embeddings and `media` with images, sets `lastFetchedAt` on every
  feed; rerun inserts no duplicates.
- **T19. cluster** (5.3). *Done:* related headlines share `eventId`; `articleCount`/`outletCount` correct; similarities logged.
- **T20. select + build-packet** (5.3). *Done:* ≤ `EVENTS_PER_RUN` events get packets; each passes `PacketPayload` with
  ≤ 8 sources, ≤ 6 media; one run per enabled author; rerunning `select` makes no new packets for the same events.
- **T21. generate** (5.3). *Done:* every run ends `succeeded` (post + citations), `invalid`, or `failed` (latter two with
  clear `error`); killing/restarting worker mid-run creates no duplicate posts.
- **T22. Run pipeline now.** `apps/web/src/lib/boss.ts` (6.4) + `/admin` button. *Done:* click produces posts end to end.

### Phase 6 — Public site
- **T23. Home** `/` (6.2, 7.6). *Done:* real events with author chips, pagination, empty state on empty DB.
- **T24. Rendering + post page.** `ArticleBody`, `remarkPolicy`, `remarkCitations`, `rehype-unwrap-images`, `CitationRef`,
  `MediaEmbed`, `MediaImage`, `YouTubeFacade` (7.5); `/posts/[slug]` with sources and **read-only** action bar (counts;
  buttons in T28); CSP header (8); rendered preview on `/admin/runs`. Add a `/styleguide` sample article using every element
  incl. one image, one mp4, one YouTube id, one invented link, one unknown media id.
  *Done:* plugin tests pass; on `/styleguide` invented link is plain text, unknown media shows nothing, citations open
  popovers; no hydration/CSP console errors.
- **T25. Event page** `/events/[id]` + `AuthorTabs`. *Done:* each tab shows its author's post and stats.
- **T26. Authors list + page** (6.2). *Done:* stats match a manual SQL check for one author.
- **T27. About.** *Done:* shows active prompt and rules.

### Phase 7 — Interactions
- **T28. Reactions** (6.5), optimistic UI (`useOptimistic`). *Done:* like→like removes, like→dislike switches, counts match
  `reaction` table, signed-out click → `/login?next=…`.
- **T29. Favorites + `/me`.** *Done:* toggling updates `favoriteCount`; `/me` lists favorites and comments.
- **T30. Comments** (6.5). *Done:* replies nest one level; reply-to-reply rejected; 6th comment in a minute rejected; hidden
  comments disappear and aren't counted.
- **T31. Share menu** (6.7). *Done:* each channel opens the right URL; `shareCount` rises (not past per-IP limit).
- **T32. OG metadata.** `generateMetadata` (title, description = dek, canonical URL) + `app/posts/[slug]/opengraph-image.tsx`
  (1200×630): headline, author chip, site name. If the post embeds an image, fetch it (3 s timeout) as background; fetch
  failure or non-jpg/png → text-only card. Fonts from bundled `.ttf` (not woff2).
  *Done:* `/posts/{slug}/opengraph-image` returns an image for posts with and without media.

### Phase 8 — Deploy
- **T33. Dockerfiles.**
  - `apps/worker/Dockerfile`: `node:22-bookworm-slim` (not alpine; onnxruntime-node needs glibc). pnpm install, run
    `download-model` at build with `HF_CACHE_DIR=/app/.models`. Start `tsx src/index.ts`.
  - `apps/web/Dockerfile`: multi-stage, `output: 'standalone'`; copy `.next/standalone`, `.next/static` →
    `apps/web/.next/static`, `public` → `apps/web/public`. Start `node apps/web/server.js`.
  - Compose: `postgres` (healthcheck); `migrate` (one-shot `pnpm --filter @stn/db migrate && pnpm --filter @stn/db seed`,
    depends_on postgres `service_healthy`); `worker`, `web` (depends_on migrate `service_completed_successfully`).
    In compose `DATABASE_URL` host is `postgres`.
  *Done:* `docker compose up --build` runs the site locally; "Run pipeline now" produces posts.
- **T34. Dokploy.** Compose deploy, env vars, domain + HTTPS (Dokploy/Traefik before `web`), daily `pg_dump` backup.
  *Done:* public URL serves the site; cron produces posts every 4 hours.

## 10. Later (not MVP)
Email verification before commenting (nodemailer + Better Auth `requireEmailVerification`), event updates/regeneration, blind
mode, leaderboard trends, side-by-side compare, search, admin similarity histogram for clustering tuning.
