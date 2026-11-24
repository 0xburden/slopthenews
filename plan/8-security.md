## 8. Security Rules
- Provider API keys: AES-256-GCM (`packages/core/src/crypto.ts`), stored `iv:tag:ciphertext` (base64 each). Decrypted only in
  the worker and the admin Test connection action; never sent to the browser.
- `requireAdmin()` in every admin page/action. First admin only via `make-admin`.
- Provider URLs must be `https://`.
- Comments render as plain text.
- Model Markdown renders only through `ArticleBody` (`skipHtml`, `allowedElements`, `urlTransform`, `remarkPolicy`); raw HTML never rendered.
- Links must match a stored packet source URL; media must be an `mN` id resolved to a worker-extracted URL. Outgoing links `rel="noopener noreferrer nofollow ugc"`.
- Images `referrerPolicy="no-referrer"`; YouTube via `youtube-nocookie.com`, click-to-load.
- CSP in `next.config.ts` `headers()`, all routes:
  `img-src 'self' https: data:; media-src https:; frame-src https://www.youtube-nocookie.com; object-src 'none'; base-uri 'self'; frame-ancestors 'none'`.
  Do **not** add `default-src`/`script-src` (breaks Next/next-themes inline scripts).
- Media is hotlinked (never re-hosted), credited and linked to the outlet. `MEDIA_ENABLED=false` disables it.
