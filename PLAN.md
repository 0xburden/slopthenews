# slopthenews — Implementation Plan

A news aggregation site where AI models ("authors") each write their own article about the
same current event from the same frozen set of RSS sources. Readers react, favorite, comment,
and share. Think of it as a public editorial benchmark.

> **How to use this plan:** Work through the tasks in section 9 in order. Each task lists what
> to build and how to check it is done ("Done when"). Do not start a task until the previous
> one passes its check. Do not add features that are not listed. When a library behaves
> differently from what this plan says, stop and report it; do not guess.

---

## 0. Fixed Decisions

### Stack
| Topic | Decision |
|---|---|
| Language | TypeScript (strict), Node 22 (≥ 22.12) |
| Package manager | pnpm 10 workspaces |
| Web | Next.js 16 (App Router, Server Components, Server Actions) |
| Styling | Tailwind CSS v4 + `@tailwindcss/typography` + Headless UI v2 + `@heroicons/react` |
| Theming | `next-themes` (light / dark / system, class strategy) |
| Fonts | `next/font/google`: Newsreader (serif), Inter (sans), JetBrains Mono (mono) |
| Article format | Markdown written by the model, rendered with `react-markdown` + `remark-gfm` + `rehype-unwrap-images` |
| Shared Markdown rules | `packages/core/src/markdown-policy.ts`, imported by both worker and web |
| Markdown parsing (validation) | `mdast-util-from-markdown` + `micromark-extension-gfm` + `mdast-util-gfm` + `mdast-util-to-string` + `unist-util-visit` |
| Database | PostgreSQL 16 + `pgvector` |
| ORM | Drizzle ORM + drizzle-kit, driver `pg` (node-postgres) |
| Auth | Better Auth, email + password only, `admin` plugin for roles |
| Jobs | pg-boss (on the same Postgres) |
| LLM client | `ai` (Vercel AI SDK) + `@ai-sdk/openai-compatible` |
| Validation | Zod |
| Embeddings | `@huggingface/transformers`, model `Xenova/all-MiniLM-L6-v2` (384 dims), runs locally in the worker |
| RSS | `rss-parser` |
| Article text | `@mozilla/readability` + `linkedom` |
| Slugs | `slugify` |
| Run TS (worker, scripts) | `tsx` |
| Tests | Vitest |
| Deploy | Docker Compose on Dokploy: `postgres`, `migrate`, `web`, `worker` |

### Version pins (required)
Install these **major versions** (`pnpm add <pkg>@<major>`). Other majors have different APIs
from what this plan describes.

The implementing model (Inkling 2 Mini, released 2026-11-20) has no published knowledge cutoff;
it is estimated at late 2025 to mid 2026. A major is pinned to the newest version released by
about **late 2025**, so the model has probably seen it. Anything newer stays one major back.

| Package | Major | First release | Notes |
|---|---|---|---|
| `next` | 16 | 2025-10 | create with `pnpm dlx create-next-app@16`. See the Next 16 notes in section 6 |
| `react`, `react-dom` | 19 | 2024-12 | |
| `tailwindcss`, `@tailwindcss/postcss` | 4 | 2025-01 | no `tailwind.config.*` file (section 7) |
| `@headlessui/react` | 2 | 2024-05 | v2 names: `TabGroup/TabList/Tab/TabPanels/TabPanel`, `MenuButton/MenuItems/MenuItem`, `PopoverButton/PopoverPanel` |
| `next-themes` | 0.4 | — | |
| `pg-boss` | 12 | 2025-11 | ESM only, **named** export `import { PgBoss } from 'pg-boss'`; API in section 5.2 |
| `ai` | 5 | 2025-07 | v6 (2025-12) and v7 (2026-06) are too new. Use `maxOutputTokens` (not `maxTokens`); usage is `usage.inputTokens` / `usage.outputTokens` |
| `@ai-sdk/openai-compatible` | 1 | 2025-07 | matches `ai@5` |
| `better-auth` | 1 | 2024-11 | |
| `drizzle-orm` / `drizzle-kit` | 0.4x / 0.3x | — | |
| `react-markdown` | 10 | 2025-02 | |
| `zod` | 4 | 2025-07 | `import { z } from 'zod'`. Prefer `z.email()`, `z.url()` over `z.string().email()` |
| `@huggingface/transformers` | 3 | 2024-10 | v4 (2026-03) is too new |
| `pnpm` | 10 | 2025-01 | v11+ is too new |

Root `package.json` sets `"packageManager": "pnpm@10.x.x"` (use the installed version) and
`"engines": { "node": ">=22.12" }`.

**Out of scope for MVP:** blind mode, OAuth, email verification, user profile customization,
regenerating articles when an event gets new sources, search, notifications.

---

## 1. Repository Layout

```
slopthenews/
├─ PLAN.md
├─ package.json                # workspace root scripts
├─ pnpm-workspace.yaml
├─ tsconfig.base.json
├─ .env.example                # copy to .env (repo root)
├─ docker-compose.yml          # local dev + Dokploy
├─ apps/
│  ├─ web/                     # Next.js site + admin panel
│  │  └─ .env -> ../../.env    # symlink (Next.js only reads its own folder)
│  └─ worker/                  # pg-boss jobs: ingest, cluster, select, build-packet, generate
└─ packages/
   ├─ db/                      # Drizzle schema, client, migrations, seed, make-admin
   └─ core/                    # crypto, packet/article types, markdown policy, prompt, llm, config
```

### Workspace wiring
- Package names: `@stn/web`, `@stn/worker`, `@stn/db`, `@stn/core`.
- `packages/*` are **not built**. They export TypeScript source:
  - `@stn/db`: `"exports": { ".": "./src/index.ts" }`
  - `@stn/core`: `"exports": { ".": "./src/index.ts", "./markdown-policy": "./src/markdown-policy.ts", "./packet": "./src/packet.ts" }`
- `apps/web/next.config.ts`:
  `transpilePackages: ['@stn/core', '@stn/db']`,
  `serverExternalPackages: ['pg-boss', '@huggingface/transformers', 'pg']`,
  `output: 'standalone'`, `outputFileTracingRoot: path.join(__dirname, '../..')`.
- **Client components** (`'use client'`) may import only `@stn/core/markdown-policy` and
  `@stn/core/packet`. Never import `@stn/core` (the barrel) or `@stn/db` in client code;
  they pull in server-only code (crypto, database, LLM).
- The worker and scripts run with `tsx`. Scripts load the root env with
  `node --env-file=../../.env --import tsx src/<file>.ts` (or `tsx --env-file=../../.env`).
- Config: `packages/core/src/config.ts` reads `process.env` once, validates it with Zod, and
  exports a typed `config` object. Everything else imports `config` instead of reading
  `process.env`. Startup fails with a clear message if a variable is missing or invalid.

---

## 2. Environment Variables (`.env.example`)

Comments go on their own lines. Values with spaces are quoted.

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

---

## 3. Database Schema (`packages/db/src/schema.ts`)

- All ids are `text` primary keys set with `crypto.randomUUID()` unless noted.
- All tables have `createdAt timestamp with time zone default now()`. All timestamps are `with time zone`.
- `vector(384)` columns use drizzle's `vector('name', { dimensions: 384 })` from `drizzle-orm/pg-core`.

### Auth tables (generated; do not hand-write)
Generated by the Better Auth CLI into `packages/db/src/auth-schema.ts` (task T5) and
re-exported from `schema.ts`. With the `admin` plugin, `user` gets `role`, `banned`, `banReason`,
`banExpires`, and `session` gets `impersonatedBy`.

### Content
```
provider
  id, name text unique, baseUrl text (https only),
  apiKeyEncrypted text,
  extraBody jsonb default '{}',        -- merged into every chat request body (provider-specific options)
  enabled bool default true

author
  id, providerId -> provider.id, slug text unique, displayName text, modelId text,
  bio text null, color text default '#64748b',   -- hex /^#[0-9a-f]{6}$/i, author chip dot
  temperature real default 0.7, maxOutputTokens int default 4000,
  enabled bool default true

feed
  id, name text, outlet text, url text unique, enabled bool default true,
  lastFetchedAt timestamp null, lastError text null

raw_article
  id, feedId -> feed.id, outlet text, url text unique (normalized), title text, excerpt text,
  content text null, publishedAt timestamp, embedding vector(384) null,
  eventId -> event.id null

media
  id, rawArticleId -> raw_article.id,
  kind text  -- 'image' | 'video' | 'youtube'
  url text            -- https image/video file URL, or the 11-char video id for 'youtube'
  posterUrl text null, caption text null
  UNIQUE(rawArticleId, url)

event
  id, title text, centroid vector(384),
  articleCount int default 0, outletCount int default 0,
  status text default 'open'  -- 'open' | 'queued' | 'generated' | 'skipped'
  queuedAt timestamp null,
  firstSeenAt timestamp, lastSeenAt timestamp

source_packet
  id, eventId -> event.id unique, articleIds text[], payload jsonb, contentHash text

prompt_template
  id, version int unique, system text, user text, active bool default false

generation_run
  id, authorId -> author.id, packetId -> source_packet.id, promptVersion int,
  status text  -- 'pending' | 'running' | 'succeeded' | 'failed' | 'invalid'
  latencyMs int null, tokensIn int null, tokensOut int null,
  rawOutput text null, error text null, finishedAt timestamp null
  UNIQUE(authorId, packetId)

post
  id, runId -> generation_run.id unique, eventId -> event.id, authorId -> author.id,
  slug text unique, headline text, dek text, bodyMarkdown text, wordCount int,
  publishedAt timestamp,
  likeCount int default 0, dislikeCount int default 0, commentCount int default 0,
  favoriteCount int default 0, shareCount int default 0,
  citationValidRatio real, paragraphCitationRatio real,
  linkCount int default 0, mediaCount int default 0,
  unknownRefCount int default 0,      -- links/media the model used that are not in the packet
  formatViolationCount int default 0  -- disallowed Markdown (see section 4)

post_citation
  postId -> post.id, index int, rawArticleId -> raw_article.id
  PK(postId, index)
```

### Interactions
```
reaction   userId -> user.id, postId -> post.id, value smallint (1 = like, -1 = dislike), PK(userId, postId)
favorite   userId -> user.id, postId -> post.id, PK(userId, postId)
comment    id, postId -> post.id, userId -> user.id, parentId -> comment.id null, body text,
           hidden bool default false
share      id, postId -> post.id, userId null, ipHash text null,
           channel text  -- 'copy' | 'x' | 'reddit' | 'email' | 'native'
```

### Indexes
`raw_article(publishedAt)`, `raw_article(eventId)`, `media(rawArticleId)`,
`event(status, lastSeenAt)`, `post(eventId)`, `post(authorId)`, `comment(postId)`,
`generation_run(status)`, and an HNSW index on `event.centroid`:
`index('event_centroid_idx').using('hnsw', t.centroid.op('vector_cosine_ops'))`.

### Count columns
The `*Count` columns on `post` are caches. Every action that changes reactions, favorites,
comments (including hide/unhide), or shares recomputes the affected count with a
`SELECT count(*)` in the same transaction. Never increment blindly.

---

## 4. Packets, Articles, and Markdown Rules (`packages/core`)

Authors write **Markdown**, not JSON. Models write Markdown more reliably than long escaped
JSON strings, and Markdown keeps formatting (subheads, lists, tables, media). Formatting is
part of what we are comparing.

### 4.1 Packet types (`packages/core/src/packet.ts`)
Zod schemas plus exported types:
```ts
PacketSource  = { n: number; rawArticleId: string; outlet: string; title: string;
                  publishedAt: string /* ISO */; url: string; text: string /* ≤ 1500 chars */ }
PacketMedia   = { id: string /* "m1" */; kind: 'image' | 'video' | 'youtube'; source: number /* n */;
                  caption: string; url: string; posterUrl: string | null }
PacketPayload = { eventTitle: string; sources: PacketSource[]; media: PacketMedia[] }
```
- `source_packet.articleIds[i]` = `payload.sources[i].rawArticleId`.
- `contentHash` = sha256 hex of `JSON.stringify(payload)` (for information only).
- `text` = `raw_article.content ?? raw_article.excerpt`, cut to 1500 characters.
- At most 8 sources: one article per outlet first (newest per outlet), then the newest remaining.
- At most 6 media items, from the chosen sources in source order, max 2 per source, ids `m1…m6`.
- Media `url` / `posterUrl` are stored in the payload for rendering but **never** shown in the prompt.
- Add `packages/core/test/fixtures/packet.json`: a realistic packet with 4 sources and 2 media items,
  used by the tests and by the `try-llm` script.

### 4.2 Required response shape
```markdown
# Headline goes here

> One-sentence summary (the "dek").

Opening paragraph with facts and citations [1][3].

![Rescue teams at the site on Tuesday](m1)

## Optional subhead

More paragraphs. Where sources disagree, say so [2][4]. The
[BBC's coverage](https://www.bbc.co.uk/news/articles/abc123) has more detail [1].
```

Rules given to the model (the same text appears on `/about`):
1. The first line is the headline as a level-1 heading (`# `).
2. Next comes a one-sentence summary as a blockquote (`> `).
3. The body uses only: paragraphs, `##` / `###` subheads, bullet or numbered lists, `**bold**`,
   `*italic*`, blockquotes for short attributed quotes, tables, links, and media.
4. Cite sources by number in square brackets right after the claim: `[1]`. For multiple
   sources use `[1][3]`.
5. **Links:** `[text](url)` only with a source URL from the packet, copied exactly.
6. **Media:** up to 3 items, `![caption](mN)`, each on its own line, where `mN` is a media id
   from the packet. Models never see or write media URLs, only ids. This avoids typos in long CDN
   URLs and makes invented media impossible. Images and videos use the same syntax; the
   renderer picks the player from the media `kind`.
7. No HTML, code blocks, footnotes, or headings below `###`.

**Why links and media are restricted to the packet:** authors cannot browse the web, so any
other URL is invented. Unknown links render as plain text, unknown media ids are dropped, and
both are counted in `unknownRefCount`, a public benchmark metric.

### 4.3 Shared rules (`packages/core/src/markdown-policy.ts`)
Both the worker (validation) and the web app (rendering) import this file, so they always agree.
It must not import anything server-only.
```ts
export const ALLOWED_ELEMENTS = ['p','br','h2','h3','ul','ol','li','strong','em','blockquote',
  'table','thead','tbody','tr','th','td','hr','del','sup','a','img'];
export const MAX_MEDIA = 3;
export const MEDIA_REF = /^m([1-9]\d*)$/;            // no "g" flag: safe to reuse with .test()

// Citations. Never export a /g regex (it keeps lastIndex state between calls).
export function findCitations(s: string): number[];   // all [n] in order, excluding "[n](" (a link)
export function normalizeUrl(url: string): string;    // try/catch new URL(); lowercase host; http→https;
                                                      // drop hash, utm_* params, trailing "/"; on error return input
export function isPacketLink(url: string, packet: PacketPayload): boolean;
export function resolveMedia(ref: string, packet: PacketPayload): PacketMedia | null;
export function urlTransform(url: string): string;    // keep "mN" refs and http(s) URLs; everything else → ""
```
`urlTransform` is stricter than react-markdown's default: it also drops relative URLs that are
not media ids, and `mailto:` and other schemes.

### 4.4 Parsing: `parseArticle(raw, packet)` (`packages/core/src/article.ts`)
Returns `{ ok: true, article: ParsedArticle } | { ok: false, error: string }`. Steps, in order:

1. **Normalize the response.**
   - Trim. Remove `<think>…</think>` blocks (and an unclosed `<think>…` up to the first `# ` line).
   - If the whole response is wrapped in a code fence (```` ``` ```` or ```` ```markdown ````), unwrap it.
   - Drop every line before the first line that starts with `# `.
   - Delete lines matching `/^\s{0,3}\[\^?\d+\]:/` (footnote / reference definitions; they would
     turn citations into links). Count each deleted line as a format violation.
2. **Normalize citations** with string replacement (not the Markdown parser), skipping
   brackets followed by `(`:
   `[1, 3]` / `[1,3]` → `[1][3]`; `[1-3]` / `[1–3]` → `[1][2][3]`; `[^1]` → `[1]`; `【1】` → `[1]`.
3. **Split.**
   - Headline = text of the first `# ` line.
   - Dek = the first non-blank line after it, which must start with `>`. Join consecutive `>`
     lines with a space and strip the `> `.
   - Body = everything after the dek.
4. **Check citations.** `findCitations(body)`. A number is valid when 1 ≤ n ≤ `sources.length`.
   `citationValidRatio` = valid / all (0 if there are none). Remove invalid markers from the body string.
5. **Analyze** `tree = fromMarkdown(body, { extensions: [gfm()], mdastExtensions: [gfmFromMarkdown()] })`
   with `visit` from `unist-util-visit`:
   - `wordCount` = number of whitespace-separated words in `toString(tree)`.
   - **Counted paragraphs** = `paragraph` nodes at any depth (including inside list items), except
     paragraphs inside a `blockquote` and paragraphs whose only non-whitespace children are `image`
     nodes. Table cells are not counted.
   - `paragraphCitationRatio` = counted paragraphs whose `toString` contains a valid `[n]` ÷ counted
     paragraphs (0 if none).
   - `linkCount` = `link` nodes where `isPacketLink(url)`.
   - `mediaCount` = `image` nodes where `resolveMedia(url)` is not null and the image is alone
     in its paragraph, capped at `MAX_MEDIA`.
   - `unknownRefCount` = `link` nodes failing `isPacketLink` + `image` nodes failing `resolveMedia`.
   - `formatViolationCount` = deleted definition lines (step 1) + `html`, `code`, `definition`,
     `footnoteDefinition`, `footnoteReference` nodes + headings with depth 1 or ≥ 4 + images in a
     paragraph that also has text + valid media beyond `MAX_MEDIA`.
   - Keep the plain text of each counted paragraph for the copy check.

`ParsedArticle` = `{ headline, dek, bodyMarkdown, wordCount, citations: number[] (unique, sorted, valid only),
citationValidRatio, paragraphCitationRatio, linkCount, mediaCount, unknownRefCount, formatViolationCount }`.

The body is stored **as Markdown text** (`post.bodyMarkdown`). It is never re-serialized
(re-serializing can escape `[1]` into `\[1]`). Unsafe elements are removed at render time (section 7.5).

### 4.5 Validation (`packages/core/src/validate.ts`)
`validateArticle(article, packet)` returns `null` or an error string. The run is `invalid` if any rule fails:
1. The headline is missing or not 10–140 characters.
2. The dek is missing or not 20–300 characters.
3. `wordCount` is under 150 or over 1500.
4. There are no valid citations, or `citationValidRatio` < 0.5.
5. `paragraphCitationRatio` < 0.5.
6. A counted paragraph shares 30 or more consecutive words with one source's `text` (the packet
   text the model saw). Compare lowercased words with punctuation removed.

`unknownRefCount` and `formatViolationCount` never make a run invalid. They are stored on the
post and shown publicly, so readers can see which models follow the rules.

---

## 5. Pipeline (`apps/worker`)

### 5.1 Flow
```
pipeline.tick (cron PIPELINE_CRON)
  └─> ingest ──> cluster ──> select ──> build-packet (one per selected event)
                                              └─> generate (one per run = per enabled author)
```
Each job sends the next one when it finishes. `pipeline.tick` stops early (logs and returns) if
there is no enabled author with an enabled provider.

### 5.2 pg-boss v12 setup (`apps/worker/src/boss.ts`, `apps/worker/src/index.ts`)
```ts
import { PgBoss } from 'pg-boss';
const boss = new PgBoss(config.DATABASE_URL);
boss.on('error', console.error);
await boss.start();

// Queues must exist before send/work/schedule.
await boss.createQueue('pipeline.tick', { policy: 'singleton' });
await boss.createQueue('ingest',        { policy: 'singleton', expireInSeconds: 1800 });
await boss.createQueue('cluster',       { policy: 'singleton', expireInSeconds: 900 });
await boss.createQueue('select',        { policy: 'singleton' });
await boss.createQueue('build-packet',  { retryLimit: 2, retryDelay: 30 });
await boss.createQueue('generate',      { retryLimit: 2, retryDelay: 30, retryBackoff: true, expireInSeconds: 600 });

await boss.schedule('pipeline.tick', config.PIPELINE_CRON);

// Handlers receive an ARRAY of jobs. Use batchSize 1 and destructure.
await boss.work('ingest', { batchSize: 1 }, async ([job]) => { await ingest(); await boss.send('cluster', {}); });
// generate: 2 jobs at a time.
await boss.work('generate', { batchSize: 1, includeMetadata: true, localConcurrency: 2 }, async ([job]) => generate(job));
```
- Graceful shutdown: on `SIGTERM`/`SIGINT` call `await boss.stop()`.
- **The worker owns the queues and the pg-boss schema.** The web app only sends jobs (section 6.4).

### 5.3 Jobs

**`ingest`** (`apps/worker/src/jobs/ingest.ts`). For each enabled feed (one feed's failure
never stops the others):
1. Fetch with `User-Agent` (5.6) and `AbortSignal.timeout(15000)`, then parse with `rss-parser`.
   Set `feed.lastFetchedAt`; on error set `feed.lastError` and continue.
2. For each item: `url = normalizeUrl(item.link)`, `publishedAt = item.isoDate ?? item.dcDate ?? now`.
   Skip items older than `CLUSTER_WINDOW_HOURS`. Insert `raw_article` with
   `onConflictDoNothing({ target: url })`. Continue only with rows that were actually inserted.
3. For new rows (at most 4 page fetches at a time): fetch the page (`AbortSignal.timeout(5000)`,
   stop reading after 2 MB). Parse with `linkedom`'s `parseHTML`. **First** extract media (5.5),
   **then** run `new Readability(document).parse()` (Readability changes the DOM). Save
   `content = result?.textContent?.trim()`. Page failures are fine: keep `content = null`.
4. Embed `title + "\n" + excerpt` (5.4) and save `embedding`.

**`cluster`** (`apps/worker/src/jobs/cluster.ts`). Process articles with `eventId IS NULL AND
embedding IS NOT NULL`, **one at a time**, ordered by `publishedAt ASC`, one transaction each:
1. Find the nearest event:
   `SELECT id, centroid <=> $v AS distance FROM event WHERE lastSeenAt >= $publishedAt - interval 'CLUSTER_WINDOW_HOURS hours' ORDER BY centroid <=> $v LIMIT 1`
   (drizzle: `cosineDistance(event.centroid, v)`). Similarity = `1 - distance`. Log every similarity
   (for tuning `CLUSTER_SIMILARITY`).
2. If similarity ≥ `CLUSTER_SIMILARITY`: set `article.eventId`, then update the event:
   - `centroid = (centroid * articleCount + v) / (articleCount + 1)` (compute in TS on the number arrays)
   - `articleCount = articleCount + 1`
   - `lastSeenAt = greatest(lastSeenAt, publishedAt)`
   - `outletCount = (SELECT count(DISTINCT outlet) FROM raw_article WHERE eventId = event.id)`
3. Otherwise create an event: `title = article.title`, `centroid = v`, `articleCount = 1`,
   `outletCount = 1`, `firstSeenAt = lastSeenAt = publishedAt`, `status = 'open'`.

**`select`** (`apps/worker/src/jobs/select.ts`), in order:
1. Set `status='skipped'` on `open` events with `lastSeenAt < now() - CLUSTER_WINDOW_HOURS`.
2. Set `status='open'` on `queued` events with `queuedAt < now() - 1 hour` and no `source_packet`
   (recovers events whose build-packet failed).
3. Pick `open` events with `outletCount ≥ MIN_OUTLETS_PER_EVENT`, ordered by
   `outletCount DESC, lastSeenAt DESC`, limit `EVENTS_PER_RUN`. Set `status='queued'`,
   `queuedAt=now()`, and `boss.send('build-packet', { eventId }, { singletonKey: eventId })` for each.

**`build-packet`** (`apps/worker/src/jobs/build-packet.ts`). In one transaction:
1. Load enabled authors whose provider is also enabled. If there are none: set the event back to
   `open` and stop.
2. Build the payload (4.1) and insert `source_packet … onConflictDoNothing({ target: eventId })`,
   then select the packet for this event. **This is the de-dupe:** an event gets one packet, ever.
3. Load the active prompt template. Insert a `generation_run` (`status='pending'`,
   `promptVersion` = active version) for each author with `onConflictDoNothing()` on `(authorId, packetId)`.
4. Set event `status='generated'`.

After commit: `boss.send('generate', { runId }, { singletonKey: runId })` for every `pending` run of
this packet. If anything throws, set the event back to `open` and rethrow.

**`generate`** (`apps/worker/src/jobs/generate.ts`):
1. **Claim the run:** `UPDATE generation_run SET status='running' WHERE id=$runId AND status='pending' RETURNING *`.
   If no row comes back, return (another worker has it, or it is finished).
2. Load author, provider, the packet, and the prompt template with `version = run.promptVersion`.
   If the author or provider is now disabled, set the run to `failed` with error `author disabled` and return.
3. `const t0 = Date.now()`. Call `writeArticle` (T11), which uses
   `generateText({ model, system, prompt, temperature, maxOutputTokens, maxRetries: 0, abortSignal: AbortSignal.timeout(180_000) })`.
4. **Network/API error** (`APICallError` with `isRetryable`, timeout, or fetch error): if
   `job.retryCount < job.retryLimit`, set the run back to `pending` and rethrow (pg-boss retries).
   Otherwise set `failed`. Non-retryable API errors (e.g. 401, 404 model) → `failed` immediately.
5. **Got text:** `parseArticle` + `validateArticle`. If either fails, set `invalid` with the error.
6. **Valid:** in one transaction, insert `post` (slug rule below, `publishedAt = now()`) and
   `post_citation` rows `{ postId, index: n, rawArticleId: sources[n-1].rawArticleId }` for each
   n in `citations`, and set the run to `succeeded`.
7. In every outcome, store `rawOutput` (if any), `latencyMs`, `tokensIn`, `tokensOut`, `error`, `finishedAt`.

**Slug rule:** `slugify(headline, { lower: true, strict: true }).slice(0, 60)` + `-` + `author.slug`
+ `-` + `run.id.slice(0, 6)`.

### 5.4 Embeddings (`apps/worker/src/embed.ts`)
```ts
import { pipeline, env } from '@huggingface/transformers';
env.cacheDir = config.HF_CACHE_DIR;
let extractor: any;                                   // singleton, loaded once
export async function embed(text: string): Promise<number[]> {
  extractor ??= await pipeline('feature-extraction', 'Xenova/all-MiniLM-L6-v2');
  const out = await extractor(text, { pooling: 'mean', normalize: true });
  return Array.from(out.data as Float32Array);        // length 384
}
```
Add a script `pnpm --filter @stn/worker download-model` that calls `embed('warm up')` once
(used in the Docker build).

### 5.5 Media extraction (`apps/worker/src/media.ts`)
Pure functions with unit tests on saved fixtures in `apps/worker/test/fixtures/`:
`mediaFromRssItem(item)` and `mediaFromHtml(document)`. Per article, keep at most 2 items, only
`https:` URLs, de-duplicated by URL. Insert with `onConflictDoNothing()`.

1. **From RSS.** Configure `rss-parser`:
   ```ts
   new Parser({ customFields: { item: [
     ['media:content', 'mediaContent', { keepArray: true }],
     ['media:thumbnail', 'mediaThumbnail', { keepArray: true }],
     ['media:group', 'mediaGroup'],
     ['media:description', 'mediaDescription'],
     ['dc:date', 'dcDate'],
   ] } })
   ```
   Attributes are under `$`: `m.$.url`, `m.$.medium`, `m.$.type`, `m.$.width`. `enclosure` is built in
   (`item.enclosure.url`, `.type`).
   - `mediaContent` with `$.medium === 'video'` or `$.type` starting with `video/` → `video`; otherwise → `image`.
   - `mediaThumbnail` → `image`. `enclosure` with an `image/*` type → `image`.
   - With several sizes, pick the largest `$.width`.
   - Caption: `mediaDescription` (string or `_` text) if present.
2. **From the article page** (before Readability runs):
   - `meta[property="og:image"]` → `image`; caption = `meta[property="og:image:alt"]`.
   - `meta[property="og:video:secure_url"]` or `og:video` with an `.mp4`/`.webm` URL or a `video/*`
     `og:video:type` → `video`, poster = og:image.
   - `iframe[src]` or `a[href]` matching
     `/(?:youtube(?:-nocookie)?\.com\/(?:embed\/|watch\?v=)|youtu\.be\/)([A-Za-z0-9_-]{11})/` → `youtube`,
     `url` = the 11-character id, poster = `https://i.ytimg.com/vi/{id}/hqdefault.jpg`.
3. If there is no caption, use the article title.

Feed check (done while planning): BBC, Guardian, NYT, France 24, and Fox News include images in
RSS; every article page checked had `og:image` + `og:image:alt`. **Video is rare.** These
outlets use their own JavaScript players, and no YouTube embeds were found. Expect almost all
media to be images.

### 5.6 HTTP rules
Every outgoing request (RSS, article pages, OG image fetch) sends
`User-Agent: slopthenews/0.1 (+{PUBLIC_SITE_URL})`. The Guardian returns 403 to browser-like
user agents. DW serves RSS 1.0 (RDF); its date is in `dc:date` (`dcDate`).

### 5.7 LLM client (`packages/core/src/llm.ts`)
```ts
createModel(provider, apiKey, modelId) =
  createOpenAICompatible({
    name: 'provider',
    baseURL: provider.baseUrl,
    apiKey,
    transformRequestBody: (body) => ({ ...body, ...provider.extraBody }),
  }).chatModel(modelId)

listModels(baseUrl, apiKey): GET {baseUrl}/models with Bearer auth, AbortSignal.timeout(10000) → string[] of ids
writeArticle({ model, author, template, packet }) → { text, usage: { inputTokens, outputTokens }, latencyMs }
```
`generateText` returning Markdown works with every OpenAI-compatible endpoint. No JSON mode or
structured output is needed.

### 5.8 Prompt template v1 (`packages/core/src/prompt.ts`, seeded by `packages/db/src/seed.ts`)
`renderPrompt(template, packet) → { system, prompt }` replaces `{{eventTitle}}`, `{{sourceCount}}`,
`{{sources}}`, and `{{media}}` in both `system` and `user`.

**System:**
```
You are a news writer for a news site. Write one neutral, factual news article that combines
the provided sources into a single story.

Rules:
- Use only facts found in the sources. Do not add outside knowledge.
- Cite sources by number in square brackets right after each claim, like [1] or [2][3].
  Only use numbers from 1 to {{sourceCount}}.
- Every paragraph must have at least one citation.
- Where sources disagree, say so and cite each side.
- Do not copy sentences from the sources. Short quotes are allowed if attributed.
- Length: 300 to 800 words.

Format your answer as Markdown, exactly like this:
# Headline

> One-sentence summary.

Body paragraphs. You may use ## and ### subheads, lists, **bold**, *italic*, tables, and
blockquotes for short attributed quotes.

Links: you may link to a source with [text](url), using only the source URLs given below,
copied exactly.

Media: you may add up to 3 images or videos from the media list below. Put each one on its own
line as ![caption](m1), using the media id. Never write a media URL. If no media is listed,
do not add any.

Do not use HTML, code blocks, or footnotes.

Reply with the article only.
```

**User:**
```
Event: {{eventTitle}}

Sources:
{{sources}}

Media:
{{media}}
```
Each source in `{{sources}}`, separated by a blank line:
```
[n] Outlet — Title (publishedAt ISO date)
URL: url
text
```
Each media item in `{{media}}` is one line: `[m1] image — caption (from source 2)`. If there is
no media, `{{media}}` is `(none)`.

### 5.9 Seed feeds (all checked working)
| Outlet | URL |
|---|---|
| BBC | https://feeds.bbci.co.uk/news/world/rss.xml |
| NPR | https://feeds.npr.org/1001/rss.xml |
| The Guardian | https://www.theguardian.com/world/rss |
| Al Jazeera | https://www.aljazeera.com/xml/rss/all.xml |
| NYT | https://rss.nytimes.com/services/xml/rss/nyt/World.xml |
| CBS News | https://www.cbsnews.com/latest/rss/world |
| Deutsche Welle | https://rss.dw.com/rdf/rss-en-world |
| France 24 | https://www.france24.com/en/rss |
| Washington Post | https://feeds.washingtonpost.com/rss/world |
| Fox News | https://moxie.foxnews.com/google-publisher/world.xml |

### 5.10 Venice provider (added in the admin UI, not hard-coded)
- Base URL `https://api.venice.ai/api/v1`, API key from the Venice account. Model ids come from
  `GET /models` (the admin "Test connection" button).
- Venice adds its own system prompt by default, which would skew the benchmark. Set the provider's
  `extraBody` to `{ "venice_parameters": { "include_venice_system_prompt": false } }`. Check this
  parameter name against the Venice API docs in T13.
- The provider form has a "Venice" preset button that fills in the base URL and this `extraBody`.

---

## 6. Web App (`apps/web`)

Next.js 16 notes:
- `params`, `searchParams`, `headers()`, and `cookies()` are **async**
  (`const { slug } = await params`). This includes `opengraph-image.tsx` and `generateMetadata`.
- Turbopack is the default for `next dev` and `next build`. Keep it.
- `next lint` no longer exists. Lint with `eslint` directly (or skip lint in MVP).
- Do not enable `cacheComponents` or use `'use cache'`. This app needs no caching layer (6.6).
- There is no `middleware.ts` in this app. Auth checks happen in pages and actions (6.1). Pages with Headless UI interactivity use small client
components; do **not** put `'use client'` on a page.

### 6.1 Auth (`apps/web/src/lib/`)
- `auth.ts`: `betterAuth({ database: drizzleAdapter(db, { provider: 'pg', schema }), emailAndPassword: { enabled: true }, plugins: [admin(), nextCookies()] })`.
  `nextCookies()` must be the last plugin.
- `app/api/auth/[...all]/route.ts`: `export const { GET, POST } = toNextJsHandler(auth)`.
- `auth-client.ts`: `createAuthClient({ plugins: [adminClient()] })`, used by the login/signup forms.
- `session.ts`:
  - `getSession()` = `auth.api.getSession({ headers: await headers() })`
  - `requireUser()` redirects to `/login?next=…` when signed out
  - `requireAdmin()` calls `notFound()` unless `session.user.role === 'admin'`
- **Making the first admin:** no automatic promotion. After signing up, run
  `pnpm --filter @stn/db make-admin you@example.com` (`packages/db/src/make-admin.ts`
  sets `role='admin'` for an existing user, matching email case-insensitively).

### 6.2 Public routes
Only events with at least one post are shown anywhere public.

| Route | Content |
|---|---|
| `/` | Latest events with ≥ 1 post (paginated, 20, `?page=`). Each item: event title, source outlet names, one author chip per post linking to it. |
| `/events/[id]` | Event title + outlets, `AuthorTabs` (client wrapper around Headless UI `TabGroup`) with one tab per post. Each panel gets server-rendered `<ArticleBody>` as children plus a stats strip. Shared source list. |
| `/posts/[slug]` | Full post (section 7.6), sources list, action bar, comments. `generateMetadata` for OG. |
| `/authors` | All authors with stats: posts, likes, dislikes, approval % (likes / (likes + dislikes)), average `citationValidRatio`, average latency, average `unknownRefCount`, average `formatViolationCount`, % of posts with media. Also: run success rate (succeeded / all finished runs). |
| `/authors/[slug]` | Author info + the same stats + their posts. |
| `/about` | Methodology: the pipeline, the active prompt template text, the rules from 4.2 and 4.5. |
| `/login`, `/signup` | Email + password forms (Better Auth client). After login, go to `?next=` or `/`. |
| `/me` | `requireUser()`. Signed-in user's favorites and comments. Sign-out button. |

### 6.3 Admin routes
Every admin `page.tsx` **and** every admin server action calls `requireAdmin()` first. The layout
check alone is not enough.

| Route | Content |
|---|---|
| `/admin` | Counts (events by status, runs by status, posts) + last 20 runs + "Run pipeline now" button. |
| `/admin/providers` | List / create / edit / delete. Fields: name, baseUrl (must be `https://`), API key (write-only: never sent to the client; on edit an empty field keeps the existing key), extraBody (JSON textarea, validated). "Venice" preset button. "Test connection" lists the model ids. |
| `/admin/authors` | List / create / edit / enable / disable. Model picker filled from the provider's `/models`. Color (`<input type="color">`, validated `/^#[0-9a-f]{6}$/i` on the server). Slug auto-filled from display name, editable. |
| `/admin/feeds` | List / create / enable / disable, showing `lastFetchedAt` and `lastError`. |
| `/admin/prompts` | List versions, create a new version (next version number), set active (one transaction: all others `active=false`). |
| `/admin/runs` | Paginated runs with status filter. Detail view: `rawOutput` in a `<pre>` (plain text), `error`, metrics. (A rendered preview is added in T24.) |
| `/admin/comments` | Latest comments, hide/unhide (recomputes `commentCount`). |

### 6.4 Sending jobs from the web app (`apps/web/src/lib/boss.ts`)
A lazy singleton for sending only:
`new PgBoss({ connectionString: config.DATABASE_URL, supervise: false, schedule: false, migrate: false })`,
started once on first use, exposing `sendJob(name, data)`. The worker must have run once first (it
creates the schema and queues). "Run pipeline now" calls `sendJob('pipeline.tick', {})`.

### 6.5 Server actions (`apps/web/src/actions/`)
All inputs are validated with Zod. After writing, call `revalidatePath` for the post page and its event page.

- `react(postId, value: 1 | -1)`: `requireUser()`. Same value again removes the reaction; the other
  value switches it. Recompute `likeCount`/`dislikeCount` in the same transaction.
- `toggleFavorite(postId)`: `requireUser()`. Recompute `favoriteCount`.
- `addComment(postId, body, parentId?)`: `requireUser()`. Body is 1–2000 characters after trimming.
  If `parentId` is given, the parent must be a top-level comment (`parentId IS NULL`) on the **same
  post**; otherwise reject. Rate limit: max 5 comments per user per minute (count in the DB).
  Recompute `commentCount` (only `hidden = false`).
- `recordShare(postId, channel)`: no login needed. `channel` must be one of the enum values. Store
  `ipHash` = sha256(ip + BETTER_AUTH_SECRET), from the `x-forwarded-for` header. Ignore more than
  10 shares per ipHash per hour. Recompute `shareCount`.

### 6.6 Viewer state and caching
Post and event pages read the session to show "you liked this" / "favorited", so they are
dynamic. Do not add `revalidate` or other caching config. Rendering Markdown per request is cheap
at this scale.

### 6.7 Share
Use `navigator.share` when available. Otherwise open a Headless UI `Menu` with: Copy link,
X (`https://x.com/intent/post?url=`), Reddit (`https://www.reddit.com/submit?url=`), Email
(`mailto:?subject=&body=`). URL-encode the parameters. Each option calls `recordShare`.

---

## 7. Design System (`apps/web`)

The look is a calm, newspaper-like reading page. Keep chrome minimal. Use hairline rules
instead of shadows. Use serif type for reading, sans for UI, and mono for model data.

### 7.1 Fonts (`apps/web/src/app/fonts.ts`)
| Role | Font | CSS variable from next/font | Used for |
|---|---|---|---|
| Serif | **Newsreader** (variable, `axes: ['opsz']`, `style: ['normal', 'italic']`) | `--font-newsreader` | Headlines, deks, article body |
| Sans | **Inter** (variable) | `--font-inter` | Navigation, buttons, forms, admin, metadata |
| Mono | **JetBrains Mono** | `--font-jetbrains` | Model ids, stats |

Load all three with `next/font/google` (`display: 'swap'`, `subsets: ['latin']`, `variable: '…'`)
and put their `.variable` class names on `<html>`. Newsreader was designed for on-screen news
reading. If it looks too narrow, use Source Serif 4 instead (only this file changes).

### 7.2 Tailwind v4 setup (`apps/web/src/app/globals.css`)
There is **no** `tailwind.config.*` file. `postcss.config.mjs` uses `@tailwindcss/postcss`.
```css
@import "tailwindcss";
@plugin "@tailwindcss/typography";
@custom-variant dark (&:where(.dark, .dark *));

@theme inline {
  --font-serif: var(--font-newsreader), Georgia, serif;
  --font-sans: var(--font-inter), system-ui, sans-serif;
  --font-mono: var(--font-jetbrains), ui-monospace, monospace;
  --color-paper: var(--paper);
  --color-surface: var(--surface);
  --color-ink: var(--ink);
  --color-muted: var(--muted);
  --color-rule: var(--rule);
  --color-accent: var(--accent);
}

:root { --paper: #FAF8F3; --surface: #FFFFFF; --ink: #1C1B19; --muted: #6B6760; --rule: #E4E0D8; --accent: #B4232A; }
.dark { --paper: #141311; --surface: #1C1B19; --ink: #ECE8E1; --muted: #A29D94; --rule: #2E2C28; --accent: #F0626A; }

body { @apply bg-paper text-ink font-sans; }
```
Classes like `bg-paper text-ink border-rule text-accent font-serif` then work. Set the typography
plugin colors to the tokens with a `.prose` override (`--tw-prose-body: var(--ink)` and so on).
Every text color meets WCAG AA contrast on `paper` in both modes.

### 7.3 Type scale
- Article body: `prose prose-lg dark:prose-invert font-serif max-w-[68ch]`, line height 1.7.
- Post headline: `font-serif font-semibold text-4xl md:text-5xl tracking-tight leading-tight`.
- Dek: `font-serif italic text-xl text-muted`.
- Metadata line (author · date · read time): `font-sans text-sm text-muted`.
- Stats: `font-mono text-xs tabular-nums`.

### 7.4 Light / dark mode
- `apps/web/src/components/Providers.tsx` (client) wraps `next-themes` `ThemeProvider` with
  `attribute="class"`, `defaultTheme="system"`, `enableSystem`, `disableTransitionOnChange`.
- `<html lang="en" suppressHydrationWarning>`.
- `ThemeSwitcher` (client): a Headless UI `Menu` with System / Light / Dark, using heroicons
  `ComputerDesktopIcon` / `SunIcon` / `MoonIcon`. Render a same-size placeholder until mounted
  (`useEffect` sets `mounted`), to avoid a hydration mismatch.

### 7.5 Markdown rendering (`apps/web/src/components/ArticleBody.tsx`)
A **Server Component** that renders `post.bodyMarkdown` on every request. Markdown is never
converted to HTML ahead of time, so rule changes apply to every existing post.

Props: `{ markdown: string; packet: PacketPayload }`. Pages load the packet through
`post → event → source_packet.payload`.

```tsx
<Markdown
  remarkPlugins={[remarkGfm, [remarkPolicy, { packet }], remarkCitations]}
  rehypePlugins={[rehypeUnwrapImages]}
  skipHtml
  allowedElements={ALLOWED_ELEMENTS}
  unwrapDisallowed
  urlTransform={urlTransform}
  components={components /* below */}
>{markdown}</Markdown>
```

**Plugins** (in `apps/web/src/lib/`, each with a Vitest test):
- `remarkPolicy` (ours, `remark-policy.ts`), changes the mdast tree:
  - heading depth 1 → 2, depth ≥ 4 → 3 (`h1` must not reach `allowedElements`, which would strip it).
  - Remove `image` nodes inside a paragraph that has other non-whitespace content.
  - Remove `image` nodes whose URL fails `resolveMedia`, and every valid media image after the third.
  - Remove `footnoteDefinition`, `footnoteReference`, `definition`, `html`, and `code` nodes.
- `remarkCitations` (ours, `remark-citations.ts`): walks `text` nodes (skipping text inside `link`
  and `image`) and splits each `[n]` out into this exact node:
  ```ts
  { type: 'citation', data: { hName: 'sup', hProperties: { dataN: String(n) } },
    children: [{ type: 'text', value: String(n) }] }
  ```
  `mdast-util-to-hast` turns it into `<sup data-n="n">n</sup>`. Add the TypeScript module
  augmentation: `declare module 'mdast' { interface PhrasingContentMap { citation: Citation } }`.
- `rehype-unwrap-images` (npm): unwraps a `<p>` that contains only images, so `<figure>`/`<video>`/
  `<iframe>` never end up inside `<p>` (invalid HTML that breaks hydration).

**Components.** Never pass react-markdown's `node` prop to a client component (it is not
serializable). Destructure it away: `({ node, ...props }) => …`.
```tsx
const components = {
  table: ({ node, ...p }) => <div className="overflow-x-auto"><table {...p} /></div>,
  sup:   ({ node, ...p }) => { const n = Number(p['data-n']); const s = packet.sources[n - 1];
           return s ? <CitationRef n={n} outlet={s.outlet} title={s.title} url={s.url} /> : null; },
  a:     ({ node, href, children }) => isPacketLink(href ?? '', packet)
           ? <a href={href} target="_blank" rel="noopener noreferrer nofollow ugc">{children}<ArrowTopRightOnSquareIcon className="inline size-3.5 ml-0.5" /></a>
           : <span>{children}</span>,
  img:   ({ node, src, alt }) => { const m = resolveMedia(src ?? '', packet);
           return m && config.MEDIA_ENABLED ? <MediaEmbed media={m} alt={alt ?? m.caption} source={packet.sources[m.source - 1]} /> : null; },
};
```
- `CitationRef` (client): a Headless UI `Popover`. The button shows the superscript `n` and links to
  `#source-n`. The panel shows the outlet, the title, and an external link.
- `MediaEmbed`, by `media.kind`:
  - `image` → `<figure>` containing `MediaImage` (client): `<img src loading="lazy" decoding="async" referrerPolicy="no-referrer" alt>`.
    On `onError` it hides the whole figure (some outlet CDNs block hotlinking). Use a plain `<img>`,
    not `next/image`, because hosts are unknown.
  - `video` → `<figure><video controls preload="none" playsInline poster={posterUrl ?? undefined} src={url} /></figure>`.
  - `youtube` → `<figure>` containing `YouTubeFacade` (client): shows the poster with a play button in a 16:9
    box. On click it swaps in
    `<iframe src="https://www.youtube-nocookie.com/embed/{id}?autoplay=1" allow="autoplay; encrypted-media; picture-in-picture" allowFullScreen>`.
    Nothing loads from YouTube until the reader clicks.
  - Every figure has `<figcaption>`: `{caption} · ` + the outlet name linking to the source URL.

### 7.6 Page patterns
- **Masthead:** wordmark "slopthenews" in Newsreader bold, today's date in small caps sans, a thin
  double rule underneath. Navigation on the right: Authors, About, theme switcher, user menu
  (Log in / Sign up when signed out; Me / Admin (admins only) / Sign out when signed in).
- **Home:** the first event is the "lead" (large serif headline, full width). The rest are a list
  separated by rules. Each event shows the title, the source outlet names, and one **author chip**
  per post: a colored dot (`author.color`) plus the display name, linking to that post.
- **Post page:** a small "AI-generated · {model id in mono}" label in `accent` above the headline,
  then headline, dek, metadata line, `ArticleBody`, numbered "Sources" list (`<li id="source-n">`),
  action bar (like, dislike, favorite, share: heroicon buttons with counts), and comments.
- **Event page:** headline and outlet list, then `AuthorTabs`. Each tab label is an author chip;
  each panel is that author's post plus a stats strip (words, latency, tokens, citation ratio,
  links/media used, unknown refs, format violations).
- **Admin:** plain sans UI with tables and forms, the same tokens, and no serif.
- **Empty states:** every list has a friendly empty message (e.g. "No stories yet. The next
  pipeline run is at {time}.").
- **Responsive:** mobile-first single column; the home lead plus list becomes two columns at `lg`.
- **Accessibility:** semantic `<article>`, `<header>`, `<time dateTime>`; visible
  `focus-visible:ring-2 ring-accent`; respect `prefers-reduced-motion`; icon buttons have
  `aria-label`s.

---

## 8. Security Rules
- **API keys:** provider keys are encrypted with AES-256-GCM (`packages/core/src/crypto.ts`) and
  stored as `iv:tag:ciphertext` (each part base64). They are decrypted only in the worker and in
  the admin "Test connection" action, and are never sent to the browser.
- **Admin:** `requireAdmin()` in every admin page and admin action. The first admin is created
  only with the `make-admin` script.
- **Provider URLs** must be `https://` (an admin-only feature, but still a server-side request).
- **Comments** render as plain text, never HTML.
- **Model Markdown** renders only through `ArticleBody` (`skipHtml`, `allowedElements`,
  `urlTransform`, `remarkPolicy`). Raw HTML from a model is never rendered.
- **Links and media come only from the packet.** Links must match a stored source URL. Media must
  be an `mN` id resolved to a URL the worker extracted. Models cannot inject URLs. Outgoing links
  use `rel="noopener noreferrer nofollow ugc"`.
- **Third-party loading:** images use `referrerPolicy="no-referrer"`. YouTube uses
  `youtube-nocookie.com` and loads only on click.
- **CSP header** in `next.config.ts` `headers()` for all routes:
  `img-src 'self' https: data:; media-src https:; frame-src https://www.youtube-nocookie.com; object-src 'none'; base-uri 'self'; frame-ancestors 'none'`.
  Do **not** add `default-src` or `script-src`; they break Next.js and next-themes inline scripts.
- **Copyright:** media is hotlinked (never re-hosted), always credited to the outlet, and linked to
  the source article. `MEDIA_ENABLED=false` turns it off.

---

## 9. Task List

Each task ends with a "Done when" check. Run `pnpm -r typecheck` and `pnpm -r test` before
every commit. Commit after each task.

### Phase 1 — Scaffold
- **T1. Workspace.** First run `corepack enable && corepack use pnpm@10` in the repo root, so
  every `pnpm` command uses pnpm 10 (the machine may have a newer global pnpm). Root
  `package.json` (scripts: `dev`, `typecheck`, `test`),
  `pnpm-workspace.yaml` (`apps/*`, `packages/*`), `tsconfig.base.json` (strict, `moduleResolution: "bundler"`),
  `.gitignore` (`node_modules`, `.next`, `.env`, `.models`), `.env.example` (section 2). Create
  `packages/db`, `packages/core`, `apps/worker` with `package.json` (exports as in section 1),
  `tsconfig.json`, and an `src/index.ts`. Add `packages/core/src/config.ts` (section 1).
  In the root `package.json`, set `pnpm.onlyBuiltDependencies: ["onnxruntime-node", "sharp", "esbuild"]`.
  *Done when:* `pnpm install` and `pnpm -r typecheck` succeed.
- **T2. Postgres.** `docker-compose.yml` with service `postgres` (`pgvector/pgvector:pg16`), env
  from `.env` (`POSTGRES_USER/PASSWORD/DB`), volume `pgdata`, port `5432:5432`, healthcheck
  `pg_isready -U $POSTGRES_USER`.
  *Done when:* `docker compose up -d postgres` shows the service as healthy and `psql $DATABASE_URL -c 'select 1'` works.
- **T3. Next.js app + design system.** `pnpm dlx create-next-app@16 apps/web` (TypeScript, App Router,
  Tailwind, `src/` dir; answer "No" to ESLint and React Compiler). Rename the package to `@stn/web`. Add
  `@headlessui/react@2`, `@heroicons/react`, `@tailwindcss/typography`, `next-themes@0.4`.
  Symlink `apps/web/.env` → `../../.env`. Set up `next.config.ts` (section 1), fonts (7.1),
  `globals.css` (7.2), `Providers` + `ThemeSwitcher` (7.4), and the masthead (7.6) **without the
  user menu** (added in T5). Add a `/styleguide` page with headings, prose sample text, buttons, and
  author chips.
  *Done when:* `/styleguide` looks right in light and dark, the theme choice survives a reload,
  there is no flash of the wrong theme, and there is no hydration warning in the console.

### Phase 2 — Database
- **T4. Drizzle setup.** `packages/db`: `src/client.ts` (exports `db` from `drizzle-orm/node-postgres`
  with a `pg` Pool), `drizzle.config.ts` (`schema: './src/schema.ts'`, `out: './drizzle'`, dialect
  `postgresql`), an empty `src/schema.ts`. Create the first migration with
  `pnpm drizzle-kit generate --custom --name=enable_vector` and put
  `CREATE EXTENSION IF NOT EXISTS vector;` in the generated SQL file (it must be migration 0000).
  Add `src/migrate.ts` using `migrate()` from `drizzle-orm/node-postgres/migrator`, and scripts
  `generate` (`drizzle-kit generate`) and `migrate` (runs `migrate.ts`).
  *Done when:* `pnpm --filter @stn/db migrate` succeeds and `\dx` in psql lists `vector`.
- **T5. Better Auth.** Add `apps/web/src/lib/auth.ts`, the route handler, `auth-client.ts`, and
  `session.ts` (6.1). Generate the auth tables with
  `pnpm dlx @better-auth/cli generate --config apps/web/src/lib/auth.ts --output packages/db/src/auth-schema.ts`,
  re-export them from `schema.ts`, then run `pnpm --filter @stn/db generate` and `migrate`. Add
  `make-admin.ts` + script. Build `/login`, `/signup`, and the masthead user menu.
  *Done when:* you can sign up, log in, and log out; `make-admin` gives your user the admin role; the
  user menu shows "Admin" only for admins.
- **T6. Content + interaction schema.** All tables and indexes from section 3 in `schema.ts`, then
  `generate` + `migrate`.
  *Done when:* the migrations apply cleanly on an empty database (`docker compose down -v`, up, migrate).
- **T7. Seed.** `packages/db/src/seed.ts`: the feeds (5.9) and prompt template v1 (5.8, `active=true`),
  with `onConflictDoNothing()`. Script `seed`.
  *Done when:* running `pnpm --filter @stn/db seed` twice leaves 10 feeds and 1 prompt template.

### Phase 3 — Core library (`packages/core`, all pure functions with Vitest tests)
- **T8. crypto.ts.** `encrypt(plain)` / `decrypt(stored)` with AES-256-GCM and `config.ENCRYPTION_KEY`.
  *Done when:* round-trip test passes and decrypting tampered ciphertext throws.
- **T9. packet.ts + markdown-policy.ts** (4.1, 4.3) and `test/fixtures/packet.json`.
  *Done when:* tests cover `normalizeUrl` (utm params, hash, trailing slash, host case, http→https,
  invalid URL), `isPacketLink`, `resolveMedia`, `urlTransform` (keeps `m1` and https; drops
  `javascript:`, `mailto:`, `/relative`), and `findCitations` (ignores `[2](url)`, is the same on repeated calls).
- **T10. article.ts + validate.ts** (4.4, 4.5). Sample model responses go in `test/fixtures/*.md`.
  *Done when:* tests cover: a valid article; a code-fence-wrapped response; a `<think>` block; text
  before the headline; `[1, 3]` and `[1-3]` normalization; a definition line `[1]: https://…`; a
  missing headline; a missing dek; a dek after a blank line; an out-of-range citation; low paragraph
  citation coverage; an image-only paragraph (not counted against coverage); copied text; a valid
  packet link; an invented link; a valid media id; an unknown media id; a media URL instead of an id;
  an inline image; 4 media items; an HTML block; an `####` heading.
- **T11. prompt.ts** (5.8). *Done when:* a test renders the fixture packet and checks source
  numbering, the `URL:` lines, the media lines, `(none)` for an empty media list, and that no media URL
  appears in the output.
- **T12. llm.ts** (5.7) plus a script `try-llm` (`src/scripts/try-llm.ts`) that reads `TRY_BASE_URL`,
  `TRY_API_KEY`, `TRY_MODEL` from the env, renders the fixture packet with the v1 template, calls
  `writeArticle`, prints the text, then prints the `parseArticle` + `validateArticle` result.
  *Done when:* the script prints a real article from Venice and its validation result.

### Phase 4 — Admin
- **T13. Admin layout + providers.** `app/admin/layout.tsx` (sidebar nav) and the providers page (6.3)
  with create/edit/delete, the Venice preset, and Test connection. Check the Venice
  `include_venice_system_prompt` parameter name against the Venice docs.
  *Done when:* a normal user gets a 404 on `/admin` and on direct admin action calls; adding Venice
  and clicking Test lists its models; the API key never appears in page HTML (view source).
- **T14. Authors.** *Done when:* an author can be created with a model picked from the list, and
  an invalid color is rejected on the server.
- **T15. Feeds, prompts, runs, comments pages** (6.3, runs without the rendered preview).
  *Done when:* each page lists data; enable/disable, set-active-prompt, and hide/unhide work.

### Phase 5 — Worker
- **T16. Worker bootstrap.** `apps/worker/src/index.ts` + `boss.ts` (5.2) with empty job handlers
  that log. Scripts: `dev` (`tsx watch`), `start`.
  *Done when:* the worker starts, creates the queues, logs the schedule, and stops cleanly on Ctrl-C.
- **T17. Embeddings + media extraction** (5.4, 5.5). Save fixtures: one RSS file each from BBC,
  Guardian, NYT, and France 24, and one article page HTML.
  *Done when:* `download-model` caches the model, a test checks `embed()` returns 384 numbers with a
  length (norm) of about 1, and the media tests pass on the fixtures.
- **T18. ingest** (5.3). *Done when:* sending `ingest` fills `raw_article` with embeddings and `media`
  with images, sets `lastFetchedAt` on every feed, and running it again inserts no duplicates.
- **T19. cluster** (5.3). *Done when:* related headlines share an `eventId`, `articleCount` and
  `outletCount` are correct, and similarity values are logged.
- **T20. select + build-packet** (5.3). *Done when:* at most `EVENTS_PER_RUN` events get packets,
  each packet passes the `PacketPayload` Zod schema with ≤ 8 sources and ≤ 6 media, there is one run per
  enabled author, and running `select` again creates no new packets for the same events.
- **T21. generate** (5.3). *Done when:* each run ends as `succeeded` (with a post and citations),
  `invalid`, or `failed`, always with a clear `error` for the last two; killing the worker mid-run
  and restarting does not create duplicate posts.
- **T22. Run pipeline now.** `apps/web/src/lib/boss.ts` (6.4) + the button on `/admin`.
  *Done when:* clicking it produces posts end to end.

### Phase 6 — Public site
- **T23. Home** `/` (6.2, 7.6). *Done when:* it shows real events with author chips, pagination works,
  and the empty state shows on an empty database.
- **T24. Article rendering + post page.** `ArticleBody`, `remarkPolicy`, `remarkCitations`,
  `rehype-unwrap-images`, `CitationRef`, `MediaEmbed`, `MediaImage`, `YouTubeFacade` (7.5), the post page
  `/posts/[slug]` with the sources list and a **read-only** action bar (counts only; buttons work from
  T28), the CSP header (section 8), and the rendered preview on `/admin/runs`. Add a sample article to
  `/styleguide` that uses every element, including one image, one mp4, one YouTube id, one invented
  link, and one unknown media id.
  *Done when:* plugin tests pass; on `/styleguide` the invented link is plain text, the unknown media id
  shows nothing, citations open popovers; the browser console has no hydration or CSP errors.
- **T25. Event page** `/events/[id]` with `AuthorTabs`. *Done when:* each tab shows its author's post and stats.
- **T26. Authors list + author page** (6.2). *Done when:* the stats match a manual SQL check for one author.
- **T27. About page.** *Done when:* it shows the active prompt and the rules.

### Phase 7 — Interactions
- **T28. Reactions** (6.5) with optimistic UI (`useOptimistic`). *Done when:* like → like removes it,
  like → dislike switches, counts match the `reaction` table, and a signed-out click goes to `/login?next=…`.
- **T29. Favorites + `/me`.** *Done when:* toggling updates `favoriteCount`, and `/me` lists favorites and comments.
- **T30. Comments** (6.5). *Done when:* replies nest one level, a reply to a reply is rejected, the 6th
  comment in a minute is rejected, and hidden comments disappear and are no longer counted.
- **T31. Share menu** (6.7). *Done when:* each channel opens the right URL and `shareCount` goes up
  (but not past the per-IP limit).
- **T32. OG metadata.** `generateMetadata` (title, description = dek, canonical URL) +
  `app/posts/[slug]/opengraph-image.tsx` (1200×630): headline, author chip, site name. If the post
  embeds an image, fetch it with a 3 s timeout and use it as the background; if the fetch fails or the
  format is not jpg/png, use a text-only card. Fonts are loaded from bundled `.ttf` files (not woff2).
  *Done when:* `/posts/{slug}/opengraph-image` returns an image for a post with media and one without.

### Phase 8 — Deploy
- **T33. Dockerfiles.**
  - `apps/worker/Dockerfile`: base `node:22-bookworm-slim` (not alpine; onnxruntime-node needs glibc).
    Install with pnpm, run `download-model` at build time with `HF_CACHE_DIR=/app/.models`. Start
    with `tsx src/index.ts`.
  - `apps/web/Dockerfile`: multi-stage build with `output: 'standalone'`; copy `.next/standalone`,
    `.next/static` → `apps/web/.next/static`, and `public` → `apps/web/public`. Start with `node apps/web/server.js`.
  - Compose services:
    - `postgres` (healthcheck)
    - `migrate` (one-shot: `pnpm --filter @stn/db migrate && pnpm --filter @stn/db seed`; `depends_on` postgres `service_healthy`)
    - `worker` and `web` (`depends_on` migrate `service_completed_successfully`)
    - Inside compose, `DATABASE_URL` uses host `postgres`.
  *Done when:* `docker compose up --build` runs the whole site locally and "Run pipeline now" produces posts.
- **T34. Dokploy.** Compose deployment, env vars, domain + HTTPS (Dokploy/Traefik in front of `web`),
  daily `pg_dump` backup.
  *Done when:* the public URL serves the site and the cron produces posts every 4 hours.

---

## 10. Later (not MVP)
Email verification before commenting (SMTP via nodemailer + Better Auth
`requireEmailVerification`), event updates/regeneration, blind mode, leaderboard trends,
side-by-side compare view, search, admin-visible similarity histogram for tuning clustering.
