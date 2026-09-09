# DokuWiki MCP — Claude plugin

A plugin for [Claude Cowork / Claude Desktop](https://claude.com/docs/plugins/overview) and
[Claude Code](https://code.claude.com/docs/en/plugins) that connects Claude to a DokuWiki instance
running **[cosmocode/dokuwiki-plugin-mcp](https://github.com/cosmocode/dokuwiki-plugin-mcp)**
by Andreas Gohr ([docs](https://www.dokuwiki.org/plugin:mcp)).

This repository is **only the client side**: a plugin definition that points Claude at that endpoint,
authenticates against it, and teaches Claude how the wiki behaves. All the actual work — exposing the
wiki's remote API as MCP tools — is done by the server-side plugin.

Once installed, Claude can search, read and edit the wiki. Every tool it offers is one of the wiki's own
remote API methods, and the wiki's ACLs decide what a call may do — Claude never sees more than the user
behind the token could see themselves.

## Login

The endpoint and the token are **plugin settings**, not file contents. Claude asks for both when the
plugin is enabled:

| Setting | Value |
| --- | --- |
| **Wiki MCP endpoint** | Your wiki address plus `/lib/plugins/mcp/mcp.php`, e.g. `https://wiki.example.com/lib/plugins/mcp/mcp.php` |
| **DokuWiki API token** | From the wiki: **User Profile → Authentication Token** |

The token is declared `sensitive`, so it is masked on entry and stored in the operating system's
keychain (`~/.claude/.credentials.json` where there is none) rather than in a settings file. The
endpoint is stored under `pluginConfigs` in your Claude settings. Neither value is ever written into
the plugin directory, so the packaged plugin carries no credentials and can be shared as-is.

At connection time the values are substituted into `.mcp.json`:

```json
{
  "mcpServers": {
    "dokuwiki": {
      "type": "http",
      "url": "${user_config.wiki_url}",
      "headers": {
        "Authorization": "Bearer ${user_config.api_token}"
      }
    }
  }
}
```

### The Authorization header has to reach PHP

Apache withholds the `Authorization` header from CGI, FastCGI and FPM backends by default, so that
scripts cannot read Basic Auth credentials. PHP then never sees the token and the wiki answers
*"No API token was sent"* even though one was provided — a confusing thing to debug, because the
message points at the client.

The fix belongs on the wiki. In `lib/plugins/mcp/.htaccess`:

```apache
<IfModule mod_setenvif.c>
    SetEnvIf Authorization "(.*)" HTTP_AUTHORIZATION=$1
</IfModule>
```

That needs `AllowOverride FileInfo` for the directory. Where `.htaccess` files are disabled, put
`CGIPassAuth On` into the server configuration for that directory instead — it is the more direct
directive, but it belongs to the `AllowOverride AuthConfig` class and returns a 500 where only
`FileInfo` is granted, so it is not safe to put in a shipped `.htaccess`.

Upstream carries this as [PR #14](https://github.com/cosmocode/dokuwiki-plugin-mcp/pull/14); once it is
merged, a current install of the server-side plugin brings the file along.

DokuWiki also accepts the token in an `X-DokuWiki-Token` header, which no server strips. This plugin
does not use it: Claude's connector settings only accept header names from an approved list, and a
custom name is rejected there. In Claude Code, where the restriction does not apply, adding

```json
"X-DokuWiki-Token": "${user_config.api_token}"
```

to the `headers` block works as a fallback for a wiki whose server configuration cannot be changed.

### OAuth instead of a shared token

The server-side plugin also offers an OAuth 2.1 authorization server and per-tool access control as part
of CosmoCode's Business Plugin Partner Program. With OAuth each user authenticates under their own wiki
identity and no token is handed around. To use it, replace the `headers` block in `.mcp.json` with
`"oauth": true` and drop the `api_token` entry from `userConfig` — Claude then runs the sign-in flow
itself. Worth considering over a token when rolling MCP access out across an organization.

## Requirements

On the **wiki** side:

1. [cosmocode/dokuwiki-plugin-mcp](https://github.com/cosmocode/dokuwiki-plugin-mcp) installed in
   `lib/plugins/mcp/` — easiest via the Extension Manager. The directory name matters: the endpoint URL
   contains that path, so a differently named directory will not work.
2. In the Configuration Manager:
   - `remote` — **on**. Off on a fresh install.
   - `remoteuser` — **must be touched**, even if the answer is "everybody". It carries `!!not set!!` on a
     fresh install and DokuWiki refuses every remote request while it does. Leave it empty to allow all
     logged-in users, or name groups such as `@user`.
3. An access token: log in → **User Profile** → **Authentication Token**.

No Node, no `npx`, no bridge process — the endpoint speaks Streamable HTTP directly.

**Reachability differs per client.** Claude Code opens the connection from your own machine, so an
intranet or `localhost` wiki works. In Cowork, connectors reach external services through Anthropic's
cloud, so the wiki needs a publicly reachable HTTPS address there.

## Installation

### Cowork / Claude Desktop

Package the plugin and upload it — no file needs editing first:

```bash
git clone https://github.com/graphics80/claude-plugin-for-dokuwiki-mcp.git
cd claude-plugin-for-dokuwiki-mcp
zip -r /tmp/dokuwiki.plugin . -x '.git/*' -x '*.DS_Store'
```

Then **Customize → Plugins → upload a plugin file**, pick `/tmp/dokuwiki.plugin`, and fill in the
endpoint and token when asked.

### Claude Code

```bash
claude plugin marketplace add graphics80/claude-plugin-for-dokuwiki-mcp
claude plugin install dokuwiki@dokuwiki-mcp
```

Claude prompts for both values on enable. To set them non-interactively:

```bash
claude plugin install dokuwiki@dokuwiki-mcp \
  --config wiki_url=https://wiki.example.com/lib/plugins/mcp/mcp.php \
  --config api_token=YOUR_TOKEN
```

### For a whole organization

Serve `.claude-plugin/marketplace.json` from a git repo or over HTTPS and point
`allowedPluginMarketplaces` at it; set `installationPreference` to `auto_install` or `required` to push
it to every device. Each user still supplies their own token, so nobody inherits somebody else's wiki
permissions. See the [organization plugin docs](https://claude.com/docs/third-party/claude-desktop/extensions).

## What the plugin ships

| Component | Purpose |
| --- | --- |
| `.mcp.json` | The DokuWiki MCP connector over Streamable HTTP |
| `skills/dokuwiki-setup` | Connect the wiki and diagnose a failing login — maps each wiki error to its cause |
| `skills/dokuwiki-basics` | Page ID and namespace conventions, search-before-read, full-page writes |

## Troubleshooting

| Symptom | Cause |
|---|---|
| URL or token appears literally as `${user_config.wiki_url}` / `${user_config.api_token}` | The plugin settings were never filled in. Re-enter them in the plugin's configuration and reconnect. |
| `No API token was sent` | The `Authorization` header did not reach PHP. Apache hides it from CGI/FastCGI/FPM backends unless configured otherwise — see *The Authorization header has to reach PHP*. |
| `The credentials sent in the … header were not accepted` | The token arrived but is invalid or was reset in the user profile. |
| `not authorized to call method …` | Token is valid, but the user is not covered by `remoteuser`, or lacks ACL permission for that page. |
| 404, or HTML instead of a tool result | Wrong endpoint URL, or the server-side plugin is not in `lib/plugins/mcp/`. |
| Calls hang until the client times out | You are using an `npx mcp-remote` bridge. On missing auth the wiki answers `401` with `WWW-Authenticate: Bearer`; `mcp-remote` takes that as a cue to start an OAuth flow, which cannot complete unattended. The direct HTTP transport in this plugin returns a real error instead. |
| `There is no tool called …` | Method names use underscores, not dots: `core.getPage` is exposed as `core_getPage`. |
| Works in Claude Code, not in Cowork | The wiki is not reachable from Anthropic's cloud. See *Requirements*. |

To check permissions for a page, ask Claude to call `core_aclCheck`. It returns DokuWiki's ACL level:
`0` none, `1` read, `2` edit, `4` create, `8` upload, `16` delete, `255` admin. Writing needs at least `2`.

## Security notes

- The token is a **full credential for the wiki's entire remote API** — treat it like a password.
- It inherits the ACL permissions of the user who created it. Consider a dedicated wiki account with
  only the rights Claude actually needs, rather than an admin account.
- The packaged `.plugin` file contains **no** credentials, because the token lives in Claude's own
  secure storage. Older, hand-edited copies of this plugin did carry the token in `.mcp.json` — those
  should not be shared. The included `.gitignore` covers `*.plugin`.
- Resetting the token in the user profile invalidates the old one and logs out every application using it.

## Upgrading from 1.x

Version 1.x required editing `.mcp.json` with the endpoint and token, then re-zipping the plugin for
every change. Nothing needs to be edited any more: reinstall the plugin and enter both values when
asked. Delete any old `.plugin` file — it contains a copy of your token.

Version 2.1 dropped the `X-DokuWiki-Token` header, because Claude's connector settings reject custom
header names. A wiki that relied on it needs the server-side change described above; until then the
header can be added back by hand in Claude Code.

## Related

- [cosmocode/dokuwiki-plugin-mcp](https://github.com/cosmocode/dokuwiki-plugin-mcp) — the server-side
  plugin this one talks to (GPL-2.0, by Andreas Gohr / CosmoCode)
- [DokuWiki Remote API](https://www.dokuwiki.org/devel:remoteapi) — the methods exposed as tools
- [Claude plugins reference](https://code.claude.com/docs/en/plugins-reference) — manifest and
  `userConfig` schema
- [Model Context Protocol](https://modelcontextprotocol.io/) — the protocol itself

## License

This plugin definition is MIT — see [LICENSE](LICENSE).
