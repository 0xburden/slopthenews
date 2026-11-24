## 6. Web App (`apps/web`)
Next 16 notes: `params`, `searchParams`, `headers()`, `cookies()` are **async** (`const { slug } = await params`), incl. in
`opengraph-image.tsx` and `generateMetadata`. Turbopack is default; keep it. No `next lint` (use eslint or skip). No
`cacheComponents`/`'use cache'`. No `middleware.ts`; auth checks are in pages/actions. Interactivity lives in small client
components; never `'use client'` on a page.

### 6.1 Auth (`apps/web/src/lib/`)
- `auth.ts`: `betterAuth({ database: drizzleAdapter(db, { provider: 'pg', schema }), emailAndPassword: { enabled: true }, plugins: [admin(), nextCookies()] })` — `nextCookies()` last.
- `app/api/auth/[...all]/route.ts`: `export const { GET, POST } = toNextJsHandler(auth)`.
- `auth-client.ts`: `createAuthClient({ plugins: [adminClient()] })` for login/signup forms.
- `session.ts`: `getSession()` = `auth.api.getSession({ headers: await headers() })`; `requireUser()` redirects to
  `/login?next=…`; `requireAdmin()` → `notFound()` unless `session.user.role === 'admin'`.
- First admin: no auto-promotion. `pnpm --filter @stn/db make-admin you@example.com` (`packages/db/src/make-admin.ts`, sets
  `role='admin'` on an existing user, email case-insensitive).

### 6.2 Public routes (only events with ≥ 1 post appear anywhere public)
- `/`: latest events, 20/page, `?page=`. Each: title, outlet names, one author chip per post (links to post).
- `/events/[id]`: title + outlets, `AuthorTabs` (client wrapper over `TabGroup`), one tab per post; panel = server-rendered
  `<ArticleBody>` as children + stats strip. Shared source list.
- `/posts/[slug]`: full post (7.6), sources, action bar, comments, `generateMetadata`.
- `/authors`: per author: posts, likes, dislikes, approval % (likes/(likes+dislikes)), avg `citationValidRatio`, avg latency,
  avg `unknownRefCount`, avg `formatViolationCount`, % posts with media, run success rate (succeeded / all finished).
- `/authors/[slug]`: info + same stats + posts.
- `/about`: methodology: pipeline, active prompt text, rules from 4.2 and 4.5.
- `/login`, `/signup`: email+password (Better Auth client); then go to `?next=` or `/`.
- `/me`: `requireUser()`; favorites, comments, sign-out.

### 6.3 Admin routes — every admin `page.tsx` **and** server action calls `requireAdmin()` (layout check is not enough)
- `/admin`: counts (events/runs by status, posts), last 20 runs, "Run pipeline now".
- `/admin/providers`: CRUD. name, baseUrl (`https://` only), API key (write-only, never sent to client; empty on edit keeps
  key), extraBody (JSON textarea, validated). Venice preset. "Test connection" lists model ids.
- `/admin/authors`: list/create/edit/enable/disable. Model picker from provider `/models`. Color `<input type="color">`,
  server-validated `/^#[0-9a-f]{6}$/i`. Slug auto-filled from display name, editable.
- `/admin/feeds`: list/create/enable/disable, shows `lastFetchedAt`, `lastError`.
- `/admin/prompts`: list versions, create next version, set active (one transaction, others `active=false`).
- `/admin/runs`: paginated, status filter. Detail: `rawOutput` in `<pre>`, `error`, metrics (rendered preview in T24).
- `/admin/comments`: latest comments, hide/unhide (recomputes `commentCount`).

### 6.4 Sending jobs (`apps/web/src/lib/boss.ts`)
Lazy singleton: `new PgBoss({ connectionString: config.DATABASE_URL, supervise: false, schedule: false, migrate: false })`,
started on first use, exposes `sendJob(name, data)`. Worker must have run once (creates schema/queues). "Run pipeline now" →
`sendJob('pipeline.tick', {})`.

### 6.5 Server actions (`apps/web/src/actions/`) — Zod-validated; then `revalidatePath` post page + its event page
- `react(postId, value: 1 | -1)`: `requireUser()`. Same value removes; other value switches. Recompute like/dislikeCount in-tx.
- `toggleFavorite(postId)`: `requireUser()`. Recompute `favoriteCount`.
- `addComment(postId, body, parentId?)`: `requireUser()`. Body 1–2000 chars trimmed. Parent must be top-level
  (`parentId IS NULL`) on the **same post**. Max 5 comments/user/minute (count in DB). Recompute `commentCount` (`hidden = false`).
- `recordShare(postId, channel)`: no login. Channel from enum. `ipHash` = sha256(ip + BETTER_AUTH_SECRET) from
  `x-forwarded-for`. Ignore > 10 shares/ipHash/hour. Recompute `shareCount`.

### 6.6 Caching
Post/event pages read the session (viewer state) so they're dynamic. No `revalidate` or caching config.

### 6.7 Share
`navigator.share` if available; else Headless UI `Menu`: Copy link, X (`https://x.com/intent/post?url=`), Reddit
(`https://www.reddit.com/submit?url=`), Email (`mailto:?subject=&body=`). URL-encode params. Each calls `recordShare`.
