## 5. Pipeline (`apps/worker`)

### 5.1 Flow
`pipeline.tick` (cron `PIPELINE_CRON`) → `ingest` → `cluster` → `select` → `build-packet` (per selected event) → `generate`
(per run = per enabled author). Each job sends the next when done. `pipeline.tick` logs and returns early if no enabled
author has an enabled provider.

### 5.2 pg-boss v12 (`apps/worker/src/boss.ts`, `index.ts`)
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
`SIGTERM`/`SIGINT` → `await boss.stop()`. The worker owns queues and the pg-boss schema; web only sends (6.4).

### 5.3 Jobs (`apps/worker/src/jobs/*.ts`)
**`ingest`** — per enabled feed; one feed's failure never stops others:
1. Fetch with User-Agent (5.6) + `AbortSignal.timeout(15000)`, parse with `rss-parser`. Set `lastFetchedAt`; on error set `lastError`, continue.
2. Per item: `url = normalizeUrl(item.link)`, `publishedAt = item.isoDate ?? item.dcDate ?? now`. Skip items older than
   `CLUSTER_WINDOW_HOURS`. Insert `raw_article` `onConflictDoNothing({ target: url })`; continue only with inserted rows.
3. New rows (≤ 4 concurrent): fetch page (`AbortSignal.timeout(5000)`, stop after 2 MB), `linkedom` `parseHTML`. **First**
   extract media (5.5), **then** `new Readability(document).parse()` (it mutates the DOM). `content = result?.textContent?.trim()`;
   on failure `content = null`.
4. Embed `title + "\n" + excerpt` (5.4), save `embedding`.

**`cluster`** — rows with `eventId IS NULL AND embedding IS NOT NULL`, one at a time, `publishedAt ASC`, one transaction each:
1. `SELECT id, centroid <=> $v AS distance FROM event WHERE lastSeenAt >= $publishedAt - interval 'CLUSTER_WINDOW_HOURS hours' ORDER BY centroid <=> $v LIMIT 1`
   (drizzle `cosineDistance`). Similarity = `1 - distance`; log every similarity.
2. If ≥ `CLUSTER_SIMILARITY`: set `article.eventId`; `centroid = (centroid*articleCount + v)/(articleCount+1)` (in TS),
   `articleCount += 1`, `lastSeenAt = greatest(lastSeenAt, publishedAt)`,
   `outletCount = (SELECT count(DISTINCT outlet) FROM raw_article WHERE eventId = event.id)`.
3. Else new event: `title = article.title`, `centroid = v`, counts 1, `firstSeenAt = lastSeenAt = publishedAt`, `status='open'`.

**`select`**, in order:
1. `open` events with `lastSeenAt < now() - CLUSTER_WINDOW_HOURS` → `skipped`.
2. `queued` events with `queuedAt < now() - 1 hour` and no `source_packet` → `open` (recovers failed build-packet).
3. `open` events with `outletCount ≥ MIN_OUTLETS_PER_EVENT`, order `outletCount DESC, lastSeenAt DESC`, limit `EVENTS_PER_RUN`:
   set `queued`, `queuedAt=now()`, `boss.send('build-packet', { eventId }, { singletonKey: eventId })`.

**`build-packet`** — one transaction:
1. Load enabled authors with enabled providers; none → event back to `open`, stop.
2. Build payload (4.1); insert `source_packet … onConflictDoNothing({ target: eventId })`, then select it (the de-dupe: one packet per event, ever).
3. Insert a `pending` `generation_run` per author with `promptVersion` = active template version, `onConflictDoNothing()` on `(authorId, packetId)`.
4. Event → `generated`.
After commit: `boss.send('generate', { runId }, { singletonKey: runId })` for each `pending` run of this packet. On any throw:
event back to `open`, rethrow.

**`generate`**:
1. Claim: `UPDATE generation_run SET status='running' WHERE id=$runId AND status='pending' RETURNING *`; no row → return.
2. Load author, provider, packet, template with `version = run.promptVersion`. Author/provider disabled → `failed`, error `author disabled`.
3. `t0 = Date.now()`; `writeArticle` → `generateText({ model, system, prompt, temperature, maxOutputTokens, maxRetries: 0, abortSignal: AbortSignal.timeout(180_000) })`.
4. Network/API error (`APICallError` with `isRetryable`, timeout, fetch error): if `job.retryCount < job.retryLimit` → run back to
   `pending`, rethrow; else `failed`. Non-retryable (401, 404 model) → `failed` immediately.
5. Text: `parseArticle` + `validateArticle`; failure → `invalid` with error.
6. Valid, one transaction: insert `post` (slug below, `publishedAt = now()`), `post_citation` `{ postId, index: n, rawArticleId: sources[n-1].rawArticleId }`
   per n in `citations`, run → `succeeded`.
7. Always store `rawOutput` (if any), `latencyMs`, `tokensIn`, `tokensOut`, `error`, `finishedAt`.

Slug: `slugify(headline, { lower: true, strict: true }).slice(0, 60)` + `-` + `author.slug` + `-` + `run.id.slice(0, 6)`.

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
Script `pnpm --filter @stn/worker download-model` calls `embed('warm up')` once (used in Docker build).

### 5.5 Media extraction (`apps/worker/src/media.ts`)
Pure `mediaFromRssItem(item)` and `mediaFromHtml(document)`, tested on fixtures in `apps/worker/test/fixtures/`. Per article:
≤ 2 items, `https:` only, de-duped by URL, insert `onConflictDoNothing()`. No caption → article title.
1. **RSS:**
   ```ts
   new Parser({ customFields: { item: [
     ['media:content', 'mediaContent', { keepArray: true }],
     ['media:thumbnail', 'mediaThumbnail', { keepArray: true }],
     ['media:group', 'mediaGroup'],
     ['media:description', 'mediaDescription'],
     ['dc:date', 'dcDate'],
   ] } })
   ```
   Attributes under `$` (`m.$.url/medium/type/width`); `enclosure` built in (`.url`, `.type`).
   `mediaContent` with `$.medium==='video'` or `$.type` `video/*` → `video`, else `image`. `mediaThumbnail` → `image`.
   `enclosure` `image/*` → `image`. Several sizes → largest `$.width`. Caption = `mediaDescription` (string or `_`).
2. **Page** (before Readability): `meta[property="og:image"]` → `image`, caption `og:image:alt`.
   `og:video:secure_url` or `og:video` with `.mp4`/`.webm` URL or `video/*` `og:video:type` → `video`, poster = og:image.
   `iframe[src]`/`a[href]` matching `/(?:youtube(?:-nocookie)?\.com\/(?:embed\/|watch\?v=)|youtu\.be\/)([A-Za-z0-9_-]{11})/`
   → `youtube`, `url` = id, poster `https://i.ytimg.com/vi/{id}/hqdefault.jpg`.
Note: checked outlets all have RSS images and `og:image`+`og:image:alt`; video is rare, no YouTube found. Expect images.

### 5.6 HTTP
All outgoing requests send `User-Agent: slopthenews/0.1 (+{PUBLIC_SITE_URL})` (Guardian 403s browser UAs). DW is RSS 1.0
(RDF); date in `dc:date` (`dcDate`).

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
Plain `generateText` Markdown; no JSON mode/structured output.

### 5.8 Prompt template v1 (`packages/core/src/prompt.ts`, seeded by `packages/db/src/seed.ts`)
`renderPrompt(template, packet) → { system, prompt }` replaces `{{eventTitle}}`, `{{sourceCount}}`, `{{sources}}`, `{{media}}` in both `system` and `user`.

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
Each source (blank line between): `[n] Outlet — Title (publishedAt ISO date)` / `URL: url` / `text`.
Each media line: `[m1] image — caption (from source 2)`. No media → `(none)`.

### 5.9 Seed feeds (outlet: URL)
- BBC: https://feeds.bbci.co.uk/news/world/rss.xml
- NPR: https://feeds.npr.org/1001/rss.xml
- The Guardian: https://www.theguardian.com/world/rss
- Al Jazeera: https://www.aljazeera.com/xml/rss/all.xml
- NYT: https://rss.nytimes.com/services/xml/rss/nyt/World.xml
- CBS News: https://www.cbsnews.com/latest/rss/world
- Deutsche Welle: https://rss.dw.com/rdf/rss-en-world
- France 24: https://www.france24.com/en/rss
- Washington Post: https://feeds.washingtonpost.com/rss/world
- Fox News: https://moxie.foxnews.com/google-publisher/world.xml

### 5.10 Venice provider (added via admin UI, not hard-coded)
Base URL `https://api.venice.ai/api/v1`; model ids from `GET /models` (Test connection). Set `extraBody` to
`{ "venice_parameters": { "include_venice_system_prompt": false } }` (Venice's own system prompt would skew results; verify
the name in T13). Provider form has a "Venice" preset button filling base URL + extraBody.
