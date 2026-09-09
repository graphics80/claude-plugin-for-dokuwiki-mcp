---
name: dokuwiki-basics
description: >
  Conventions for working with a DokuWiki over its remote API — page IDs,
  namespaces, search, and safe edits. Use when reading, searching, creating
  or changing wiki pages, or when the user talks about wiki namespaces,
  page IDs, or DokuWiki syntax.
---

# Working with DokuWiki

Every tool here is one of the wiki's own remote API methods, and the wiki's
ACLs decide what a call may do. The user behind the token sets the ceiling.

Method names are exposed with underscores: `core.getPage` is `core_getPage`.

## Page IDs

A page ID is a colon-separated path, lowercase, no leading colon:
`projects:2026:kickoff`. Namespaces are the segments before the last colon.
Spaces become underscores. `start` is the default page of a namespace.

Never guess an ID. Resolve it first — search for the title, or list the
namespace — then act on the ID that comes back.

## Reading

Search before reading when the user names a topic rather than a page. Search
covers page content; a hit list gives the exact IDs to fetch.

## Writing

Before changing an existing page, read it. DokuWiki page writes replace the
whole page — send the full new text, not a fragment, or the rest of the page
is lost.

Preserve the page's existing markup style: `====== Heading ======` down to
`== Heading ==`, `[[page:id|label]]` for internal links, `  * ` for lists.

Say what changed after a write, and name the page ID.

## When a call is refused

A refusal is usually permissions, not a broken connection — see the
`dokuwiki-setup` skill for the ACL levels and the wiki-side settings.
