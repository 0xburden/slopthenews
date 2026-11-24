## 7. Design System (`apps/web`)
Calm newspaper look: minimal chrome, hairline rules not shadows, serif for reading, sans for UI, mono for model data.

### 7.1 Fonts (`apps/web/src/app/fonts.ts`)
`next/font/google`, `display: 'swap'`, `subsets: ['latin']`; `.variable` classes on `<html>`.
- Newsreader (variable, `axes: ['opsz']`, `style: ['normal','italic']`) → `--font-newsreader`: headlines, deks, body.
  (If too narrow, swap to Source Serif 4; only this file changes.)
- Inter (variable) → `--font-inter`: nav, buttons, forms, admin, metadata.
- JetBrains Mono → `--font-jetbrains`: model ids, stats.

### 7.2 Tailwind v4 (`apps/web/src/app/globals.css`); `postcss.config.mjs` uses `@tailwindcss/postcss`
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
Set typography colors to tokens via a `.prose` override (`--tw-prose-body: var(--ink)` etc.). All text meets WCAG AA on `paper` in both modes.

### 7.3 Type scale
- Body: `prose prose-lg dark:prose-invert font-serif max-w-[68ch]`, line-height 1.7.
- Headline: `font-serif font-semibold text-4xl md:text-5xl tracking-tight leading-tight`.
- Dek: `font-serif italic text-xl text-muted`. Metadata (author · date · read time): `font-sans text-sm text-muted`.
- Stats: `font-mono text-xs tabular-nums`.

### 7.4 Light/dark
- `components/Providers.tsx` (client): `ThemeProvider` `attribute="class"`, `defaultTheme="system"`, `enableSystem`, `disableTransitionOnChange`.
- `<html lang="en" suppressHydrationWarning>`.
- `ThemeSwitcher` (client): `Menu` with System/Light/Dark (`ComputerDesktopIcon`/`SunIcon`/`MoonIcon`). Same-size placeholder
  until mounted (`useEffect` sets `mounted`).

### 7.5 Markdown rendering (`apps/web/src/components/ArticleBody.tsx`)
**Server Component**, renders `post.bodyMarkdown` per request (never pre-converted, so rule changes apply to old posts).
Props `{ markdown: string; packet: PacketPayload }`; pages load packet via `post → event → source_packet.payload`.
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
Plugins in `apps/web/src/lib/`, each with a Vitest test:
- `remark-policy.ts` (`remarkPolicy`): heading depth 1 → 2, ≥ 4 → 3; remove `image` in a paragraph with other non-whitespace
  content; remove images failing `resolveMedia` and valid media after the third; remove `footnoteDefinition`,
  `footnoteReference`, `definition`, `html`, `code` nodes.
- `remark-citations.ts` (`remarkCitations`): walk `text` nodes (skip inside `link`/`image`), split each `[n]` into:
  ```ts
  { type: 'citation', data: { hName: 'sup', hProperties: { dataN: String(n) } },
    children: [{ type: 'text', value: String(n) }] }
  ```
  → `<sup data-n="n">n</sup>`. Add `declare module 'mdast' { interface PhrasingContentMap { citation: Citation } }`.
- `rehype-unwrap-images` (npm): unwraps image-only `<p>` so figure/video/iframe never sit in `<p>` (hydration errors).

Components — never pass `node` to a client component; destructure it away:
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
- `CitationRef` (client): Headless UI `Popover`; button shows superscript `n`, links `#source-n`; panel: outlet, title, external link.
- `MediaEmbed` by `kind`; every figure has `<figcaption>`: `{caption} · ` + outlet name linking to source URL.
  - `image` → `<figure>` + `MediaImage` (client): plain `<img src loading="lazy" decoding="async" referrerPolicy="no-referrer" alt>`
    (not `next/image`; unknown hosts); `onError` hides the whole figure.
  - `video` → `<figure><video controls preload="none" playsInline poster={posterUrl ?? undefined} src={url} /></figure>`.
  - `youtube` → `<figure>` + `YouTubeFacade` (client): poster + play button in 16:9 box; on click swap in
    `<iframe src="https://www.youtube-nocookie.com/embed/{id}?autoplay=1" allow="autoplay; encrypted-media; picture-in-picture" allowFullScreen>`.
    Nothing loads from YouTube before the click.

### 7.6 Page patterns
- **Masthead:** "slopthenews" wordmark (Newsreader bold), today's date (small caps sans), thin double rule below. Right nav:
  Authors, About, theme switcher, user menu (signed out: Log in / Sign up; signed in: Me / Admin (admins only) / Sign out).
- **Home:** first event is the lead (large serif, full width); rest a rule-separated list. Each: title, outlets, author chips
  (dot in `author.color` + display name, links to post).
- **Post:** "AI-generated · {model id in mono}" label in `accent` above headline; headline, dek, metadata, `ArticleBody`,
  numbered Sources (`<li id="source-n">`), action bar (like/dislike/favorite/share heroicon buttons with counts), comments.
- **Event:** headline + outlets, `AuthorTabs` (tab label = author chip; panel = post + stats strip: words, latency, tokens,
  citation ratio, links/media used, unknown refs, format violations).
- **Admin:** plain sans tables/forms, same tokens, no serif.
- **Empty states** on every list (e.g. "No stories yet. The next pipeline run is at {time}.").
- **Responsive:** mobile-first single column; home lead + list → two columns at `lg`.
- **A11y:** `<article>`, `<header>`, `<time dateTime>`; `focus-visible:ring-2 ring-accent`; respect `prefers-reduced-motion`; `aria-label` on icon buttons.
