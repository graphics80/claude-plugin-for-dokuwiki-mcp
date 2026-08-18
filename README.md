# DokuWiki MCP — Claude plugin

A [Claude Code / Cowork plugin](https://docs.claude.com/en/docs/claude-code/plugins) that connects Claude to a
DokuWiki instance running **[cosmocode/dokuwiki-plugin-mcp](https://github.com/cosmocode/dokuwiki-plugin-mcp)**
by Andreas Gohr ([docs](https://www.dokuwiki.org/plugin:mcp)).

This repository is **only the client side**: a small plugin definition that points Claude at that endpoint
and authenticates against it. All the actual work — exposing the wiki's remote API as MCP tools — is done
by the server-side plugin.

Once installed, Claude can search, read and edit the wiki. Every tool it offers is one of the wiki's own
remote API methods, and the wiki's ACLs decide what a call may do — Claude never sees more than the user
behind the token could see themselves.

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

That's it. No Node, no `npx`, no bridge process — the endpoint speaks Streamable HTTP directly.

## Installation

```bash
git clone https://github.com/graphics80/claude-plugin-for-dokuwiki-mcp.git
cd claude-plugin-for-dokuwiki-mcp
```

Edit `.mcp.json` and replace both placeholders:

- `https://example.com/lib/plugins/mcp/mcp.php` → your wiki's endpoint
- `REPLACE_WITH_YOUR_DOKUWIKI_TOKEN` → your token (**two** places, see below)

Then zip the directory with a `.plugin` extension and install it:

```bash
zip -r dokuwiki.plugin . -x '.git/*'
```

## Why the token is sent twice

`.mcp.json` sends the token in two headers at once:

```json
"headers": {
  "X-DokuWiki-Token": "<token>",
  "Authorization": "Bearer <token>"
}
```

This is deliberate. Many web servers — Apache with CGI/FastCGI and no `CGIPassAuth On` being the usual
suspect — strip the `Authorization` header before PHP ever sees it. The wiki then reports
*"No API token was sent"* even though a token is configured, which is a confusing thing to debug.
`X-DokuWiki-Token` survives that. When both arrive, the DokuWiki plugin prefers the custom header.

## Troubleshooting

| Symptom | Cause |
|---|---|
| `No API token was sent` | Neither header reached PHP. The `Authorization` header was stripped in transit — this is what `X-DokuWiki-Token` is for. |
| `The credentials sent in the … header were not accepted` | The token arrived but is invalid or was reset in the user profile. |
| `not authorized to call method …` | Token is valid, but the user is not covered by `remoteuser`, or lacks ACL permission for that page. |
| Calls hang until the client times out | You are using an `npx mcp-remote` bridge. On missing auth the wiki answers `401` with `WWW-Authenticate: Bearer`; `mcp-remote` takes that as a cue to start an OAuth flow, which cannot complete unattended. The direct HTTP transport in this plugin returns a real error instead. |
| `There is no tool called …` | Method names use underscores, not dots: `core.getPage` is exposed as `core_getPage`. |

To check permissions for a page, ask Claude to call `core_aclCheck`. It returns DokuWiki's ACL level:
`0` none, `1` read, `2` edit, `4` create, `8` upload, `16` delete, `255` admin. Writing needs at least `2`.

## Security notes

- The token is a **full credential for the wiki's entire remote API** — treat it like a password.
- It inherits the ACL permissions of the user who created it. Consider a dedicated wiki account with
  only the rights Claude actually needs, rather than an admin account.
- `.mcp.json` holds the token in plain text. Do not commit a configured copy, and do not share the
  packaged `.plugin` file — it contains the token. The included `.gitignore` covers `*.plugin`.
- Resetting the token in the user profile invalidates the old one and logs out every application using it.

## Related

- [cosmocode/dokuwiki-plugin-mcp](https://github.com/cosmocode/dokuwiki-plugin-mcp) — the server-side
  plugin this one talks to (GPL-2.0, by Andreas Gohr / CosmoCode)
- [DokuWiki Remote API](https://www.dokuwiki.org/devel:remoteapi) — the methods exposed as tools
- [DokuWiki Token Auth](https://www.dokuwiki.org/devel:remoteapi) — how the token credential works
- [Model Context Protocol](https://modelcontextprotocol.io/) — the protocol itself

Note that the server-side plugin also offers an OAuth 2.1 authorization server and per-tool access
control as part of CosmoCode's Business Plugin Partner Program. With OAuth, each user authenticates
under their own wiki identity and no shared token is needed — worth considering instead of this
token-based setup if you are rolling MCP access out across an organization.

## License

This plugin definition is MIT — see [LICENSE](LICENSE).
