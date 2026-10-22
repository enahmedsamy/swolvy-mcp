# Swolvy MCP server and skill: publish HTML from Claude, ChatGPT, Gemini and coding agents

[Swolvy](https://swolvy.com) turns an HTML file into a shareable link. Drop a file on swolvy.com, or let your AI assistant (Claude, ChatGPT, Gemini) or coding agent publish it for you, and get a short link with optional password protection. No signup is needed. A page can be replaced later without changing its link, and Swolvy Pro adds pages that never expire, files up to 100 MB and your own domain.

This repository is the public home for the two ways AI assistants publish through Swolvy:

- the remote **MCP server** at `https://swolvy.com/api/mcp`, for Claude, ChatGPT, Gemini, Claude Code, Cursor and any other MCP client
- the **`swolvy` skill** for Claude Code, in [`skills/swolvy`](./skills/swolvy), built on the HTTP API

Swolvy itself is not in this repository. `server.json` describes the server in the [MCP Registry](https://github.com/modelcontextprotocol/registry) format.

## Add the MCP server

One address, OAuth sign-in, no API key. The server works without a Swolvy account: anonymous uploads return a link and an edit token. During sign-in you can also link the connector to a Swolvy account, so pages land in your dashboard and the account tools below are added.

```
https://swolvy.com/api/mcp
```

The transport is Streamable HTTP with OAuth 2.1 (dynamic client registration and PKCE).

### Claude (claude.ai and desktop)

1. Open Settings, then Connectors.
2. Choose Add custom connector.
3. Paste `https://swolvy.com/api/mcp` and confirm.
4. Sign in when asked: continue without an account, or link your Swolvy account.

### Claude Code

```sh
claude mcp add --transport http swolvy https://swolvy.com/api/mcp
```

Run `/mcp` inside Claude Code to complete the OAuth sign-in. Add `--scope user` to make the server available in every project.

### ChatGPT

On chatgpt.com: Plugins, then +, then Add custom MCP server. Paste the address, choose OAuth as the authentication, then install the plugin. The desktop app's Add MCP server form (Streamable HTTP) works too, but without account linking.

### Cursor

Add the server to `.cursor/mcp.json` in your project, or to `~/.cursor/mcp.json` for every project:

```json
{
  "mcpServers": {
    "swolvy": {
      "url": "https://swolvy.com/api/mcp"
    }
  }
}
```

### Gemini

In the Gemini app: Settings & help, then Connected Apps, then Add a custom app, and paste the address.

For Gemini CLI, add the server to `~/.gemini/settings.json`:

```json
{
  "mcpServers": {
    "swolvy": {
      "httpUrl": "https://swolvy.com/api/mcp"
    }
  }
}
```

### Any other MCP client

Add a remote server (Streamable HTTP) with the same address. Clients that support OAuth sign in through the browser; the server advertises its metadata at `https://swolvy.com/.well-known/oauth-protected-resource/api/mcp`.

## Tools

Every connector gets the four publishing tools. A connector linked to a Swolvy account also gets the account tools. The team tools and the `workspace` parameter appear when the account belongs to a Swolvy Team.

### Publishing tools (always available)

| Tool | Purpose | Key parameters |
| --- | --- | --- |
| `upload_to_swolvy` | Publish a small HTML page (under about 80 KB) in one call and get the link | `html_content` (required), `filename` (required, 1-100 chars), `password` (4-100 chars), `generate_password`, `replace_id` + `replace_edit_token`, `remove_password`, `path`, `title`, `description`, `workspace` |
| `start_swolvy_upload` | Start an upload session for a large page; returns `upload_id`, `upload_url` and `max_bytes` | `filename` (required), plus the same optional fields as `upload_to_swolvy` |
| `append_swolvy_chunk` | Fallback for clients that cannot make HTTP requests: send the page as base64 chunks | `upload_id` (required), `chunk_base64` (required, about 50 KB of raw bytes per chunk), `is_final` |
| `finish_swolvy_upload` | Finish a session after the content was sent; returns the link, `expires_at` and `edit_token` | `upload_id` (required) |

The preferred large-file flow is: `start_swolvy_upload`, then one `PUT` of the whole file to `upload_url` (for example `curl -X PUT -H 'Content-Type: text/html' --data-binary @page.html <upload_url>`), then `finish_swolvy_upload`.

Password handling: pass `password` when the user named one, or `generate_password: true` when they only asked for protection. Swolvy then picks a memorable password (like `amber-river-42`) and returns it as `password`; the assistant tells the user. Pages can be replaced up to 10 times with `replace_id` and `replace_edit_token`; the new version keeps the current password unless a new `password` or `remove_password: true` is given.

### Account tools (connector linked to a Swolvy account)

| Tool | Purpose | Key parameters |
| --- | --- | --- |
| `list_my_files` | List the account's pages with id, link, path, name, views, size, password state and expiry | none |
| `update_my_file` | Replace a page's HTML; the link stays the same, no edit token needed | `id`, `html_content` (required), `filename` |
| `rename_my_file` | Change a page's display name | `id`, `filename` (required) |
| `set_my_file_path` | Change the page's path on the account's custom domain | `id`, `path` (required) |
| `set_my_file_metadata` | Pro: title, description and image for link previews | `id` (required), `title`, `description`, `preview_image_base64`, `remove_preview_image` |
| `set_my_file_password` | Set, change or remove a page's password | `id` (required), `password` (string or null), `generate_password` |
| `delete_my_file` | Permanently delete a page | `id` (required) |
| `get_my_plan` | Show the plan (Free or Pro), storage used, renewal and the upgrade link; checks a coupon | `coupon_code` |
| `get_my_domain` | Pro: show the custom domain, whether it is live, and DNS records still to add | none |
| `connect_my_domain` | Pro: connect a domain you own; returns the CNAME and TXT records to add | `hostname` (required) |
| `set_my_branding` | Pro: default link-preview image, and with a custom domain the site name and favicon | `site_name`, `favicon_base64`, `remove_favicon`, `preview_image_base64`, `remove_preview_image` |
| `set_my_name` | Set the name shown in the dashboard | `name` (required) |

### Team tools (Swolvy Teams)

| Tool | Purpose | Key parameters |
| --- | --- | --- |
| `list_my_teams` | List the teams the account belongs to, with role, plan, domain and dashboard link | none |
| `get_my_workspace` | Show the active workspace: personal or a team | none |
| `set_my_workspace` | Switch the active workspace for the website and every connected AI | `workspace` (required): `"personal"`, a team id such as `org_abc123`, or a team slug |

## Upload with the HTTP API

For scripts, CI jobs and coding agents that would rather make one HTTP request than add an MCP server.

### Request

`POST https://swolvy.com/api/v1/upload` with a JSON body, or a `multipart/form-data` body with the same field names (plus `file` for the HTML file itself).

| Field | Required | Meaning |
| --- | --- | --- |
| `html` | yes (or `file` in a form) | The complete HTML document, up to 2 MB |
| `filename` | yes | The page's name, up to 100 characters |
| `password` | no | 4-100 characters; viewers must enter it |
| `generate_password` | no | `true`: Swolvy picks the password and returns it |
| `replace_id`, `replace_edit_token` | no | Replace an existing page and keep its link |
| `remove_password` | no | With a replace: drop the page's password |
| `path` | no | With an API key: the page's path in the account |
| `title`, `description` | no | Pro accounts: link-preview text |
| `workspace` | no | With an API key in a team: `"personal"`, `"team"`, or a team id or slug (also the `X-Swolvy-Workspace` header) |

Authentication is optional. Without it the upload is anonymous. With `Authorization: Bearer swk_...` (an API key from the dashboard, under AI & API) the page is saved to your account.

### curl

```sh
curl -X POST https://swolvy.com/api/v1/upload \
  -H "Content-Type: application/json" \
  -d '{"html":"<!DOCTYPE html><h1>Hello</h1>","filename":"Hello.html"}'
```

Or send a file from disk:

```sh
curl -X POST https://swolvy.com/api/v1/upload \
  -F "file=@index.html" -F "filename=index.html"
```

### Response

```json
{
  "url": "https://swolvy.com/abc123",
  "id": "abc123",
  "expires_at": "2026-11-21T10:00:00.000Z",
  "edit_token": "...",
  "password_protected": false
}
```

Other fields appear when they apply: `password` (when `generate_password` was used), `never_expires: true` (Pro), `replaced: true`, `path`, `title`, `description`, `custom_url` (the page's link on a custom domain), `workspace`, and `viewer_warning` (visitors see a short notice before a page that looks like a sign-in form). Errors are JSON with one field: `{"error": "..."}`.

Keep the `edit_token`: it is the only way to replace an anonymous page later.

### Python

```python
import requests

r = requests.post(
    "https://swolvy.com/api/v1/upload",
    json={"html": open("index.html").read(), "filename": "index.html"},
)
r.raise_for_status()
page = r.json()
print(page["url"], page["edit_token"])
```

### Node

```js
import { readFile } from "node:fs/promises";

const res = await fetch("https://swolvy.com/api/v1/upload", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ html: await readFile("index.html", "utf8"), filename: "index.html" }),
});
const page = await res.json();
if (!res.ok) throw new Error(page.error);
console.log(page.url, page.edit_token);
```

### Replace a page

Send the new HTML with the id and edit token from the first response. The link does not change and the 30-day clock restarts.

```sh
curl -X POST https://swolvy.com/api/v1/upload \
  -H "Content-Type: application/json" \
  -d '{"html":"<!DOCTYPE html><h1>Hello again</h1>","filename":"Hello.html","replace_id":"abc123","replace_edit_token":"..."}'
```

### Large files (upload sessions)

One request carries up to 2 MB. Pro accounts can send pages up to 100 MB through a session; the response's `max_bytes` says what the current plan allows.

1. `POST https://swolvy.com/api/v1/upload/session` with `{"filename": "report.html"}` and the same optional fields as above (`password`, `generate_password`, `replace_id`, `replace_edit_token`, `remove_password`, `path`, `title`, `description`, `workspace`). Send `Authorization: Bearer swk_...` for a Pro account. Response: `{"upload_id", "upload_url", "finish_url", "max_bytes"}`.
2. `PUT` the raw HTML to `upload_url`: `curl -X PUT -H "Content-Type: text/html" --data-binary @report.html <upload_url>`.
3. `POST` to `finish_url` (`/api/v1/upload/session/{id}/finish`). The response has the same fields as `/api/v1/upload`.

### Rate limits

Uploads are limited to 10 per hour per IP address (HTTP 429), with a global cap across all callers. Bodies over the size limit return 413.

## Claude Code skill

The `swolvy` skill teaches Claude Code the API call above so that "publish this page" or "give me a link to this HTML" works in one step.

Install it by copying the skill folder into your Claude Code skills directory:

```sh
git clone https://github.com/enahmedsamy/swolvy-mcp.git
mkdir -p ~/.claude/skills
cp -r swolvy-mcp/skills/swolvy ~/.claude/skills/swolvy
```

Or add it to one project only, under `.claude/skills/swolvy/`. Then ask Claude Code to publish an HTML file, or run `/swolvy`. The skill file is [`skills/swolvy/SKILL.md`](./skills/swolvy/SKILL.md).

If you would rather use tool calls than curl, add the MCP server to Claude Code instead (see above). Both give the same link.

## Limits and plans

| | Free | Swolvy Pro |
| --- | --- | --- |
| Price | $0, no signup | $3 a month or $18 a year |
| File size | up to 2 MB | up to 100 MB (2 GB of storage in total) |
| Expiry | 30 days after the last upload; a new version restarts the 30 days | never |
| Password protection | yes | yes |
| Replace on the same link | yes, with the edit token (up to 10 times) or from the dashboard | yes |
| Custom domain | no | yes, one per account |
| Link previews (title, description, image) | generated | custom |

A free account (sign in with an emailed code) adds a dashboard, view counts, API keys and linking your AI assistants to the account. Passwords are stored only as a hash. Pages on swolvy.com are not listed in search engines. Swolvy hosts a single HTML file per page; it does not host multi-file sites, ZIP archives or server-side code.

## Links

- Website: https://swolvy.com
- MCP server guide: https://swolvy.com/mcp
- API guide: https://swolvy.com/developers
- Publish from Claude Code, Cursor or Codex: https://swolvy.com/share-html-from-claude-code
- llms.txt: https://swolvy.com/llms.txt
- Pricing: https://swolvy.com/pricing

## License

MIT. See [LICENSE](./LICENSE).
