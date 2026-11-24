## 4. Packets, Articles, Markdown Rules (`packages/core`)
Authors write **Markdown**, not JSON (more reliable; formatting is part of the comparison).

### 4.1 Packet types (`packages/core/src/packet.ts`) — Zod schemas + exported types
```ts
PacketSource  = { n: number; rawArticleId: string; outlet: string; title: string;
                  publishedAt: string /* ISO */; url: string; text: string /* ≤ 1500 chars */ }
PacketMedia   = { id: string /* "m1" */; kind: 'image' | 'video' | 'youtube'; source: number /* n */;
                  caption: string; url: string; posterUrl: string | null }
PacketPayload = { eventTitle: string; sources: PacketSource[]; media: PacketMedia[] }
```
- `articleIds[i]` = `sources[i].rawArticleId`; `contentHash` = sha256 hex of `JSON.stringify(payload)` (info only).
- `text` = `content ?? excerpt`, cut to 1500 chars.
- ≤ 8 sources: newest per outlet first, then newest remaining.
- ≤ 6 media, from chosen sources in source order, ≤ 2 per source, ids `m1…m6`. Media URLs are in the payload but **never** in the prompt.
- Fixture `packages/core/test/fixtures/packet.json`: realistic, 4 sources, 2 media (used by tests and `try-llm`).

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
Rules (also shown on `/about`):
1. First line: headline as `# `.
2. Then a one-sentence summary as a blockquote (`> `).
3. Body only: paragraphs, `##`/`###`, bullet/numbered lists, `**bold**`, `*italic*`, blockquotes for short attributed quotes, tables, links, media.
4. Cite by number right after the claim: `[1]`; multiple: `[1][3]`.
5. Links: `[text](url)` only with a packet source URL, copied exactly.
6. Media: ≤ 3, `![caption](mN)` on its own line, `mN` = packet media id. Models never see/write media URLs. Renderer picks the player from `kind`.
7. No HTML, code blocks, footnotes, or headings below `###`.

Authors can't browse, so other URLs are invented: unknown links render as plain text, unknown media ids are dropped, both count in `unknownRefCount` (public metric).

### 4.3 `packages/core/src/markdown-policy.ts` (shared; must not import server-only code)
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

### 4.4 `parseArticle(raw, packet)` (`packages/core/src/article.ts`)
Returns `{ ok: true, article: ParsedArticle } | { ok: false, error: string }`. Steps:
1. **Normalize:** trim; remove `<think>…</think>` (and unclosed `<think>…` up to the first `# ` line); unwrap if the whole
   response is in a ```` ``` ````/```` ```markdown ```` fence; drop lines before the first `# ` line; delete lines matching
   `/^\s{0,3}\[\^?\d+\]:/` (definitions), each counting as a format violation.
2. **Normalize citations** by string replacement (not the parser), skipping brackets followed by `(`:
   `[1, 3]`/`[1,3]` → `[1][3]`; `[1-3]`/`[1–3]` → `[1][2][3]`; `[^1]` → `[1]`; `【1】` → `[1]`.
3. **Split:** headline = text of first `# ` line. Dek = first non-blank line after it, must start with `>`; join consecutive
   `>` lines with a space, strip `> `. Body = everything after the dek.
4. **Citations:** `findCitations(body)`; valid if 1 ≤ n ≤ `sources.length`. `citationValidRatio` = valid/all (0 if none).
   Remove invalid markers from the body.
5. **Analyze** `tree = fromMarkdown(body, { extensions: [gfm()], mdastExtensions: [gfmFromMarkdown()] })` with `visit`:
   - `wordCount` = whitespace-separated words in `toString(tree)`.
   - Counted paragraphs = `paragraph` nodes at any depth (incl. list items), except inside `blockquote` or whose only
     non-whitespace children are `image`s. Table cells don't count.
   - `paragraphCitationRatio` = counted paragraphs whose `toString` has a valid `[n]` ÷ counted paragraphs (0 if none).
   - `linkCount` = `link` nodes passing `isPacketLink`.
   - `mediaCount` = `image` nodes with non-null `resolveMedia` that are alone in their paragraph, capped at `MAX_MEDIA`.
   - `unknownRefCount` = links failing `isPacketLink` + images failing `resolveMedia`.
   - `formatViolationCount` = deleted definition lines + `html`/`code`/`definition`/`footnoteDefinition`/`footnoteReference`
     nodes + headings depth 1 or ≥ 4 + images in a paragraph with text + valid media beyond `MAX_MEDIA`.
   - Keep each counted paragraph's plain text for the copy check.

`ParsedArticle` = `{ headline, dek, bodyMarkdown, wordCount, citations: number[] (unique, sorted, valid), citationValidRatio,
paragraphCitationRatio, linkCount, mediaCount, unknownRefCount, formatViolationCount }`.
Body is stored as Markdown text and never re-serialized (would escape `[1]` to `\[1]`). Unsafe elements are removed at render (7.5).

### 4.5 `validateArticle(article, packet)` (`packages/core/src/validate.ts`) → `null` | error string
Run is `invalid` if: headline missing or not 10–140 chars; dek missing or not 20–300 chars; `wordCount` < 150 or > 1500;
no valid citations or `citationValidRatio` < 0.5; `paragraphCitationRatio` < 0.5; a counted paragraph shares ≥ 30 consecutive
words with one source's `text` (lowercased, punctuation removed). `unknownRefCount`/`formatViolationCount` never invalidate.
