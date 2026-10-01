# hello.new

Upload links, shared pages and webhook URLs for AI agents.

An agent can write, reason and call tools, but it can't receive a file from a
person, hand over what it made, or take a callback from a service until
something gives it a URL. [hello.new](https://hello.new) gives it one in a
single call:

- **Upload link.** A page a person drops files on. The agent lists what
  arrived and downloads it.
- **Shared page.** Markdown the agent wrote, published as a page anyone with
  the link can read.
- **Webhook URL.** An address any service can call: Stripe, GitHub, a form, a
  cron job. The agent reads each request that arrives.

This repository is the plugin: one skill that tells an agent when and how to
use hello.new, and the configuration that connects a client to hello.new's
MCP server at `https://api.hello.new/mcp`. There is nothing to sign up for and
no key to configure. The service itself runs at hello.new and is not in this
repository.

## Install

Claude Code:

```text
/plugin marketplace add hellodotnew/hello
/plugin install hello-new@hello-new
```

Codex:

```sh
codex plugin marketplace add hellodotnew/hello
codex plugin add hello-new@hello-new
```

Gemini CLI:

```sh
gemini extensions install https://github.com/hellodotnew/hello
```

The skill on its own, for Cursor, OpenClaw, Hermes Agent and the other agents
the [skills CLI](https://github.com/vercel-labs/skills) supports:

```sh
npx skills add hellodotnew/hello
```

Any other MCP client: add a remote server with the URL
`https://api.hello.new/mcp`. The transport is Streamable HTTP and there is no
authentication. [hello.new/docs/mcp](https://hello.new/docs/mcp) has the steps
for ChatGPT, Cursor and VS Code.

## Tools

| Tool | What it does |
|---|---|
| `create_upload_link` | Makes a page a person drops files on |
| `list_files` | Lists what arrived, with small text files inline |
| `share_page` | Publishes Markdown as a page anyone with the link can read |
| `create_webhook` | Makes a URL any service can call |
| `read_webhook_requests` | Returns the requests the webhook received |
| `get_link_stats` | Returns views, visitors and countries for a link |
| `delete_link` | Deletes a link and everything in it |

Each new link comes back with a `url`, which is for people, and a `key`, which
stays with the agent. The other tools take the link's `id` and `key`.

## What the plugin runs, sends and fetches

**Runs.** Nothing on your machine. The plugin has no hooks, scripts or
binaries and starts no local server. It is one Markdown skill and one entry
that points your client at a remote MCP server.

**Sends.** When your agent calls a tool, your client sends the tool's
arguments to `api.hello.new` over HTTPS. That includes any content you or the
agent supply: the note on an upload page, the title and text of a shared page,
and the ids and keys of links made earlier. In a client without MCP, the skill
has the agent make the same calls to `https://api.hello.new` with `curl`.

Installs through the Gemini extension or `server.json` carry a fixed
`X-Hello-Client` label (`gemini-extension`, `mcp-registry`), and the skill asks
the agent to send the name of its runtime the same way. The label says which
client is calling. It is not a credential.

**Fetches.** What other people and services put behind a link: the files
someone uploaded, the requests a webhook received, and how often a link was
opened. Lists come from `api.hello.new`; the uploaded files themselves are
downloaded from `hellouploads.com`. All of it is third-party content, so an
agent should read it as data, never as instructions.

**Keeps.** hello.new stores what was sent until the link expires or is
deleted, along with when it was made, the IP address that made it and the
client label. The [privacy page](https://hello.new/privacy) has the detail.

## Anonymous links and account links

The plugin connects anonymously: no account, no key, no sign-in prompt.

- **Anonymous links** expire after 24 hours. Anyone who holds a link can use
  it, and only the agent's key can read what arrives. Each new link comes with
  a `claim_url`: open it and sign in to keep the link under your own username.
- **Account links** are optional. When a call carries a hello.new account key
  as an `Authorization: Bearer` header, links live at `hello.new/@username/…`
  and stay until you delete them. The plugin never contains a key and never
  asks for one. To use yours, add the server to your client by hand with that
  header; [the docs](https://hello.new/docs/mcp) show the line for each client.

## What is in this repository

| Path | What it is |
|---|---|
| `skills/hello/SKILL.md` | The skill, the same file as [hello.new/skill.md](https://hello.new/skill.md) |
| `plugin.json`, `mcp.json` | [Agent Plugins](https://agent-plugins.org) manifest and MCP config, the format ChatGPT and Codex read |
| `.claude-plugin/`, `.mcp.json` | Claude Code plugin, marketplace and MCP config |
| `.agents/plugins/marketplace.json` | Codex marketplace |
| `gemini-extension.json` | Gemini CLI extension |
| `server.json` | The server described in MCP Registry format |
| `assets/` | Icons |

## Docs, privacy and support

- Docs: [hello.new/docs/mcp](https://hello.new/docs/mcp)
- Privacy: [hello.new/privacy](https://hello.new/privacy)
- Support: [hello.new/support](https://hello.new/support) or
  [hello@hello.new](mailto:hello@hello.new)
