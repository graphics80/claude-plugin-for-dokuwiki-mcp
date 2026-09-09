---
name: dokuwiki-setup
description: >
  Connect the DokuWiki plugin to a wiki and diagnose failed connections.
  Use when the user says the DokuWiki tools are missing or failing, mentions
  "connect my wiki", "DokuWiki login", "wiki token", "No API token was sent",
  "credentials were not accepted", or "not authorized to call method".
---

# Connect DokuWiki

The plugin talks to the `mcp` plugin inside a DokuWiki instance over Streamable
HTTP. Two values, both entered in the plugin's settings when it is enabled:

| Setting | Value |
| --- | --- |
| Wiki MCP endpoint | The wiki address plus `/lib/plugins/mcp/mcp.php` |
| DokuWiki API token | From the wiki: **User Profile → Authentication Token** |

Both are stored by Claude, not by this plugin; the token goes to the system
keychain. Never write either value into a file in the plugin directory, and
never echo the token back to the user.

## Check the connection

Call `core_aclCheck` on a page the user names (or `start`). A numeric answer
means the endpoint, the token and the wiki's remote API all work.

Tool names use underscores, not dots: the remote API method `core.getPage`
is exposed as `core_getPage`.

## Diagnose a failure

Work through the symptom the wiki reports, in this order.

**The URL or token shows up literally as `${user_config.wiki_url}` or
`${user_config.api_token}`** — the plugin's settings were never filled in.
Send the user to the plugin's configuration and have them enter both values,
then reconnect.

**`No API token was sent`** — the `Authorization` header never reached PHP.
Apache withholds it from CGI, FastCGI and FPM backends by default, so the token
is invisible to the wiki even though the client sent it. This is a wiki-side
fix, not a plugin setting. In `lib/plugins/mcp/.htaccess` on the wiki:

```apache
<IfModule mod_setenvif.c>
    SetEnvIf Authorization "(.*)" HTTP_AUTHORIZATION=$1
</IfModule>
```

It needs `AllowOverride FileInfo` for that directory; where `.htaccess` is
disabled, `CGIPassAuth On` in the server configuration does the same. Verify
with a plain `initialize` call and read the `instructions` field of the answer:
a successful handshake alone proves nothing, because the server answers it
anonymously too.

If the wiki's configuration cannot be changed, DokuWiki also accepts the token
in an `X-DokuWiki-Token` header, which no server strips. Claude's connector
settings reject custom header names, so this only helps in Claude Code, where
the header can be added to the `headers` block by hand.

**`The credentials sent in the … header were not accepted`** — the token
arrived and is wrong. It was reset in the user profile, or copied with
whitespace. Have the user re-copy it from User Profile → Authentication Token.

**`not authorized to call method …`** — the token is valid but the account
behind it is not covered by the `remoteuser` setting, or lacks ACL rights for
that page. Two wiki-side settings decide this, both in the Configuration
Manager:

- `remote` must be **on**. It is off on a fresh install.
- `remoteuser` must have been saved at least once, even when the answer is
  "everybody". It holds `!!not set!!` on a fresh install and DokuWiki refuses
  every remote request while it does. Empty allows all logged-in users; a
  value like `@user` restricts it to that group.

**404, or an HTML page instead of a tool result** — the endpoint URL is wrong.
The server-side plugin must live in `lib/plugins/mcp/`; the directory name is
part of the URL, so a renamed directory breaks it.

**The connection hangs until it times out** — an `npx mcp-remote` bridge is in
the way. On missing auth the wiki answers `401` with `WWW-Authenticate: Bearer`,
which `mcp-remote` reads as a cue to start an OAuth flow that cannot finish
unattended. This plugin speaks HTTP directly and returns a real error instead.

**The wiki is only reachable inside a network** — in Cowork, connectors reach
external services through Anthropic's cloud, not from the user's machine. A
wiki on `localhost` or behind a company firewall is not reachable that way; it
needs a public HTTPS address. Claude Code runs the connection locally, so an
internal address works there.

## ACL levels

`core_aclCheck` returns DokuWiki's permission level for the current token:
`0` none, `1` read, `2` edit, `4` create, `8` upload, `16` delete, `255` admin.
Writing a page needs at least `2`. When a write fails, check this before
assuming the token is broken.
