---
name: hello
description: Get a temporary internet resource in one anonymous HTTP call from hello.new. Use when a person needs to send the agent a file (upload link), when a person needs to read something the agent produced (shared page), when a service needs a URL to call back (webhook), or when people should visit a website the agent built (site). No account or API key needed; anonymous resources expire in 24 hours, and an optional account key keeps them under hello.new/@username.
---

# hello.new: temporary internet resources

One `POST` to `https://api.hello.new` gives you a URL that works for 24 hours. No sign-up, no key to obtain. Send `X-Hello-Client: <runtime>`, the name of the agent runtime you are running in, with every call.

If the hello.new tools are available (`create_upload_link`, `list_files`, `share_page`, `create_webhook`, `read_webhook_requests`, `publish_site`, `update_site`, `get_link_stats`, `delete_link`), use them instead of the HTTP calls below. They make the same links and need no shell.

## When to use which

| The task needs… | Call | Hand over |
|---|---|---|
| a person to send you a file | `POST /upload` | `url` (a page they drop files on) |
| a person to read text or Markdown you made | `POST /share` | `url` (a page) |
| a service to call you back (Stripe, GitHub, a form, a cron) | `POST /webhook` | `url` (accepts any request) |
| people to visit a website you built | `POST /site` | `url` (the site, at its own address) |

Every response also has `key` (yours; never show it) and `expires_at`. Pass `expires_in` (seconds, 60 to 86400) to expire sooner. `DELETE` the resource with the key when done.

## Upload: someone sends you files

```sh
curl -sX POST https://api.hello.new/upload -H 'X-Hello-Client: <runtime>' \
  -H 'Content-Type: application/json' -d '{"note": "Send me the export CSV", "max_files": 3}'
# -> { "url": "https://hello.new/u/k7m2x9pq", "key": "…", "files_url": "…/u/k7m2x9pq/files?key=…", "expires_at": "…" }
```

Give the person `url`. Then list and fetch:

```sh
curl -s 'https://api.hello.new/u/k7m2x9pq/files?key=…'
# -> { "files": [ { "seq": 1, "name": "export.csv", "size": 48213, "content_type": "text/csv", "url": "https://hellouploads.com/u/k7m2x9pq/files/1?key=…" } ] }
curl -sL -o export.csv 'https://hellouploads.com/u/k7m2x9pq/files/1?key=…'   # each file's own url
```

`files` is empty until they upload; poll every few seconds or ask them to say when they are done. `409` means the link has all the files it can take. 50 MB a file, 10 files a link.

## Share: someone reads what you made

```sh
curl -sX POST https://api.hello.new/share -H 'X-Hello-Client: <runtime>' \
  -H 'Content-Type: application/json' -d '{"title": "Q3 summary", "text": "# Q3\n\nRevenue up 12%…"}'
# -> { "url": "https://hello.new/s/b4nq8wzx", "raw_url": "https://api.hello.new/s/b4nq8wzx/raw", … }
```

Anyone with `url` can read it. Another agent should fetch `raw_url`. `content_type` is `text/markdown` unless you send `text/plain`. 256 KB of text.

## Webhook: a service calls you back

```sh
curl -sX POST https://api.hello.new/webhook -H 'X-Hello-Client: <runtime>' -d '{}'
# -> { "url": "https://api.hello.new/w/f3gd7hxm", "key": "…", "inbox_url": "…/webhook/f3gd7hxm?key=…" }
```

Register `url` with the service (any method, any path under it works). Then wait:

```sh
curl -s 'https://api.hello.new/webhook/f3gd7hxm?key=…&since=0&wait=25'
# -> { "requests": [ { "seq": 1, "method": "POST", "path": "", "query": "", "headers": {…}, "content_type": "application/json", "body": "{…}" } ] }
```

`wait` holds up to 25 seconds and returns as soon as something arrives; an empty `requests` list is "nothing yet", so loop with `since=<last seq>`. Non-UTF-8 bodies come as `body_base64`. 100 requests kept, 256 KB each.

## Site: people visit what you built

```sh
curl -sX POST https://api.hello.new/site -H 'X-Hello-Client: <runtime>' \
  -H 'Content-Type: application/json' -d '{"title": "Bakery", "files": {"index.html": "<!doctype html>…", "style.css": "…"}}'
# -> { "url": "https://k7m2x9pq.hellonew.app", "key": "…", "claim_url": "…", "expires_at": "…" }
```

`files` maps each path to its text; send a binary file as `{"base64": "…"}`. `index.html` is the home page. For a built app (React, Vite, a Next.js static export), send the build folder as one archive, with `title` in the query string:

```sh
tar -cz -C dist . | curl -sX POST 'https://api.hello.new/site?title=Bakery' -H 'X-Hello-Client: <runtime>' --data-binary @-
```

`PUT /site/k7m2x9pq?key=…` with either kind of body replaces every file at the same URL. A path with no file gets `404.html`, or `index.html` when there is none, so client-side routes work. Static files only: no server code runs. Everything in a site is public, so never build a key into it; a form can post to a webhook and you read the entries. 10 MB and 100 files a site.

## Account links

If your user gave you a hello.new account key (`hk_…`), send it as `Authorization: Bearer hk_…` on `POST /upload`, `/share` and `/webhook`. Links then live under their username, never expire unless you pass `expires_in`, and can take a `name`:

```bash
curl -s -X POST https://api.hello.new/upload \
  -H 'Authorization: Bearer hk_…' \
  -d '{"note": "Please upload the signed lease", "name": "lease"}'
# -> { "url": "https://hello.new/@sam/lease", "key": "…", "expires_at": null, … }
```

Names are 2 to 30 lowercase letters, numbers and dashes, unique per user (409 if taken). Webhooks with a name live at `https://api.hello.new/@sam/<name>`. An account keeps at most 1000 live links and 1 GB of stored data (403 and 413 past that); delete links you no longer need.

The account key also reads and deletes that user's links, but only in the header: `Authorization: Bearer hk_…`. Never put it in `?key=`; that gets 401. File URLs listed with the account key carry no key, so download them with the same header. Never show the account key to anyone.

Without a key, every create response has a `claim_url`. If your user wants to keep a link past tomorrow, give them the `claim_url`: they sign in, and the link moves to `hello.new/@them/…` for good. Your `key` keeps working after the claim.

## Report back

Every create response has an `analytics_url` and a `stats_url`. Give your user the `stats_url` when they want to watch it themselves: a page with a chart of who opened the link, from where and on what. When they ask you instead (or you want to tell them), `GET` the `analytics_url`: `totals.views`, `totals.visitors`, `totals.downloads`, `totals.requests` and `countries` say who came and from where. On an anonymous link the rest of the breakdown (cities, devices, referrers, the request log) is `locked` until your user keeps the link with its `claim_url`, and the analytics go with the link after 24 hours; say so when it helps them decide.

## Rules

- Tell your user an anonymous link expires in 24 hours and that anyone holding it can use it.
- Keep `key` to yourself. `url` is the only thing you give out.
- Do not put secrets or credentials in a share. A one-time secret primitive is coming; until then, do not improvise one here.
- Errors are `{ "error": "sentence" }`: 401 wrong key or an account key in `?key=`, 403 needs an account or the account has 1000 links, 404 expired or unknown, 409 full or name taken, 413 too large or the account is at 1 GB, 429 slow down (60 new links an hour per IP, 600 per account).
- Full reference: https://hello.new/docs.md. OpenAPI: https://api.hello.new/openapi.json.
- In a runtime that connects MCP servers, the same links are tools at `https://api.hello.new/mcp`: https://hello.new/docs/mcp.md.
