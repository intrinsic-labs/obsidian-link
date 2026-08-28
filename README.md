# obsidian-link

A tiny static https bounce page that turns a `obsidian://open?...` deep link into
something Discord will actually linkify and render as a tappable link.

## Why this exists

Discord only auto-links `http(s)://` URLs. A bare or markdown-masked `obsidian://` URI
renders as dead text, and Discord's own Link buttons reject non-http(s) schemes
("Invalid URL protocol"). The vault's Discord bridge (`Tools/discord/new-task*.mjs`)
needs to hand Asher a link that opens a specific note in Obsidian from his phone, so this
page exists purely as a redirector: an https URL that (a) Discord will render as a real
link and (b) immediately bounces the browser into the `obsidian://` URI, with a big
"Open in Obsidian" button as the fallback for browsers that block a script-initiated
custom-scheme navigation.

It is deliberately static — one `index.html`, no build step, no dependencies — because
the whole job is "read a query param, build a URL, put it in an `<a href>`".

## Route

`GET /o?f=<vault-relative path, no .md, no leading slash>`

Optional: `v=<vault name>` — accepted ONLY when it exactly matches the one allowed
value, `intrinsic-labs-v01`. Any other value is rejected as a bad link rather than
silently ignored, since accepting an arbitrary vault name here would make this page an
open redirect into whatever `obsidian://` target the query string names.

`vercel.json` rewrites `/o` → `/index.html` (Vercel preserves the query string through a
rewrite), so the page itself just reads `location.search`. `/` also serves `index.html`
directly (Vercel's default static behavior) — hitting the bare root with no `f` renders
the same "bad link" state, which is fine since nobody is meant to land there.

`/o` was chosen over `/?f=` purely for readability in a Discord message and in server
logs — `/?f=...` would work identically, since both ultimately just serve `index.html`
and read the query string client-side.

## What the page does

1. Reads `f` (required) and `v` (optional) from the query string.
2. Validates `f`: must be a relative path — no leading `/`, no `..` segment, no
   `scheme:` prefix (which rules out `javascript:`, `obsidian:`, `http:`, etc. being
   smuggled in as the "path"), no control characters, and only characters that show up
   in real vault paths (unicode letters/numbers, spaces, and a small set of punctuation).
   Anything else renders "Bad link" and does nothing further — no redirect is attempted.
3. Builds **exactly** `obsidian://open?vault=<vault>&file=<encodeURIComponent(f)>` — the
   only thing this page ever navigates to. It cannot be made to redirect anywhere else:
   there is no host/URL parameter, only a file path that gets URI-encoded into the fixed
   template.
4. Attempts the jump immediately (`location.href = uri`) **and** renders a large
   "Open in Obsidian" `<a href="obsidian://...">` button as the user-gesture fallback,
   plus the plain-text file name. A `<noscript>` block covers browsers with JS disabled.

## Deploying

Preview only, through the vault's sanctioned wrapper — this project has no production
deploy path from any agent:

```
node ~/Documents/obsidian/intrinsic-labs-v01/Tools/crons/vault-loop-v2/preview-deploy.mjs ~/dev/web/obsidian-link
```

Production promotion is Asher's own hand (`vercel --prod` / promoting a deployment),
same as everywhere else in the vault's tooling.
