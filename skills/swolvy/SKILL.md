---
name: swolvy
description: Publish an HTML file to swolvy.com and get a shareable link (optional password, replace on the same link). Use when the user wants to share, host or send an HTML page, report, dashboard or prototype as a URL.
---

# Swolvy: publish an HTML file as a link

Swolvy (https://swolvy.com) hosts a single HTML file and returns a link anyone can open. No account or API key is needed. Free pages take files up to 2 MB and stay online for 30 days after the last upload; pages in a Swolvy Pro account never expire and can be up to 100 MB.

## When to use

- The user asks to share, host, publish or send an HTML page, report, dashboard, prototype or mockup as a URL.
- The user wants a link to something you just wrote as HTML.
- The user wants to update a page they published earlier and keep the same link.

Do not use it for multi-file sites, ZIP archives or server-side code: Swolvy hosts one HTML file per page. Inline CSS and JavaScript in that file are fine.

## Publish a file

Send the file as a multipart form. The response is JSON.

```sh
curl -sS -X POST https://swolvy.com/api/v1/upload \
  -F "file=@index.html" -F "filename=index.html"
```

When the HTML is in memory rather than on disk, send JSON with `html` and `filename`:

```sh
curl -sS -X POST https://swolvy.com/api/v1/upload \
  -H "Content-Type: application/json" \
  -d '{"html":"<!DOCTYPE html><h1>Hello</h1>","filename":"Hello.html"}'
```

Response:

```json
{"url":"https://swolvy.com/abc123","id":"abc123","expires_at":"2026-11-21T10:00:00.000Z","edit_token":"...","password_protected":false}
```

Rules: `filename` is required (1-100 characters, give it a descriptive name such as `Q3 report.html`); the HTML must be a complete document; one request carries up to 2 MB.

## Report the result

Tell the user, in plain words:

1. The link (`url`).
2. That the page stays online for 30 days after the last upload (unless the response has `never_expires: true`).
3. The `edit_token`, with one sentence that it lets them replace the page on the same link later, and that it should be kept private. If the user has a notes file or the project has a safe place for it, offer to save `id` and `edit_token` there.
4. If the response has `viewer_warning`, say that visitors see a short Swolvy notice first, because the page looks like it asks for a password or sign-in details.

## Passwords

- If the user named a password, add `-F "password=..."` (or `"password": "..."` in JSON). 4 to 100 characters.
- If the user only asked for protection ("protect it", "add a password"), do not ask them to think of one. Add `-F "generate_password=true"` (or `"generate_password": true`). Swolvy picks a memorable one (like `amber-river-42`) and returns it in the response as `password`. Always tell the user the password that was set.

## Replace a page (same link)

Send the new HTML with the page's `id` and `edit_token` from the first upload:

```sh
curl -sS -X POST https://swolvy.com/api/v1/upload \
  -F "file=@index.html" -F "filename=index.html" \
  -F "replace_id=abc123" -F "replace_edit_token=..."
```

The link does not change and the 30-day clock restarts. The new version keeps the current password unless you pass a new `password` or `remove_password=true`. A page can be replaced up to 10 times. The response has `replaced: true`.

## Save to the user's account

If the user has a Swolvy API key (from their dashboard under AI & API, it starts with `swk_`), add `-H "Authorization: Bearer swk_..."`. The page then appears in their dashboard, where they can rename, replace and delete it without the edit token. With a key, `path` (for example `q3-report`) sets the page's address on the account's custom domain, and Pro accounts can set `title` and `description` for link previews. Never ask for the key in chat if it is already in the environment; read it from a variable such as `$SWOLVY_API_KEY`.

## Large files (over 2 MB, Pro accounts)

Use an upload session with the API key:

```sh
curl -sS -X POST https://swolvy.com/api/v1/upload/session \
  -H "Authorization: Bearer $SWOLVY_API_KEY" -H "Content-Type: application/json" \
  -d '{"filename":"report.html"}'
# -> {"upload_id":"...","upload_url":"...","finish_url":"...","max_bytes":100000000}
curl -sS -X PUT -H "Content-Type: text/html" --data-binary @report.html "<upload_url>"
curl -sS -X POST "<finish_url>"
```

The finish response has the same fields as a normal upload. `max_bytes` says what the plan allows (2 MB without Pro). The same optional fields (`password`, `generate_password`, `replace_id`, `replace_edit_token`, `path`, `title`, `description`) go in the first request.

## Errors

Errors are JSON: `{"error": "message"}`. Relay the message.

- 400: missing `html` or `filename`, invalid HTML, password outside 4-100 characters.
- 401: invalid API key, or wrong `replace_id` / `replace_edit_token`.
- 413: the file is over the plan's limit (2 MB free, 100 MB Pro). For a free upload, suggest trimming the page (inline images are the usual cause) or Swolvy Pro.
- 429: rate limit (10 uploads per hour per IP). Wait and retry; do not loop.

## Alternative: MCP server

Claude Code can also add Swolvy as an MCP server and publish with a tool call instead of curl:
`claude mcp add --transport http swolvy https://swolvy.com/api/mcp` (OAuth sign-in, no key). Both give the same link. More at https://swolvy.com/developers and https://swolvy.com/mcp.
