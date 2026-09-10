# Custom Theme (starter scaffold)

A minimal, unstyled starting point for a Micro.blog theme. No CSS, no
opinionated markup beyond what's needed to make Hugo's template lookup
and Micro.blog's platform features (IndieAuth, Micropub, Webmention,
plug-in hooks, replies) work correctly.

## Structure

```
theme.toml                        # meta info shown on import
config.json                       # theme-level Hugo config (kept minimal —
                                   # Micro.blog manages most site settings itself)
layouts/
  index.html                      # homepage — MUST be root-level, see note below
  index.xml                       # feed.xml — MUST be root-level
  index.json                      # feed.json — MUST be root-level
  list.archivehtml.html           # /archive/ — kept at root AND in _default/, see note below
  404.html                        # 404 page — no sidebar, root-level (Blank has none, so no shadowing risk)
  post/
    single.html                   # single post (Type "post") — MUST be under post/, two-column layout
  _default/
    baseof.html                   # base template: head + header + {{ block "main" }} + footer
    single.html                   # fallback single template (non-post content, e.g. pages) — no sidebar
    list.html                     # fallback list template (taxonomy/category pages) — two-column layout
    list.archivehtml.html         # /archive/ — see note below; this copy is the one that actually renders
  partials/
    layout-two-column.html        # content + positioned sidebar wrapper
    sidebar.html                  # sidebar content — defaults to an h-card identity block
    entry-list.html                # shared post-list markup (h-feed), used by index.html and _default/list.html
    entry-single.html               # shared single-post markup (h-entry), used by post/single.html and _default/single.html
    head.html                     # <head>: custom.css, IndieWeb tags, plug-in CSS/HTML hooks
    header.html                   # site title + subtitle
    footer.html                   # copyright line
    custom_footer.html            # empty hook for Micro.blog's custom-footer mechanism
static/
  custom.css                      # structural base styles — layout, no colors/typography/branding yet
plugin.json                       # exposes the sidebar-position toggle as a settings-page field
```

### `static/custom.css` vs. Micro.blog's "Edit CSS" button

Micro.blog's Design page shows an "Edit CSS" button next to the
active theme, editing a file also called `custom.css`. The exact
relationship between that in-browser editor and this repo's
`static/custom.css` isn't fully confirmed here — most likely the
editor is either backed by this same file (edits there would need
syncing back to git) or is a separate override layer on top of it.
Worth clarifying by testing: edit a line via "Edit CSS" on the live
site and see whether it shows up in a subsequent theme export.

### Why some templates must live at the root, not under `_default/`

This tripped us up once already, so it's worth spelling out. Hugo
resolves a template by walking an ordered list of candidate paths
(most specific first) and, for each candidate, checking your theme's
files before falling back to the base design (Blank). The key
subtlety: **path specificity is checked before layer priority.** If
Blank defines a *more specific* path than the one you used, Blank's
file wins even though your repo is layered on top of Blank overall.

This applies to the site's **home page** (Kind `home`), where Hugo
genuinely checks root-level `layouts/index.html` before falling back
to `_default/`. Micro.blog's actual Blank design defines these at the
*root* `layouts/` level:

- `layouts/index.html` (homepage)
- `layouts/index.xml` (RSS)
- `layouts/index.json` (JSON feed)
- `layouts/post/single.html` (single post, for content of Type "post" —
  which is most Micro.blog content)

If this theme only defined `_default/list.html`, `_default/single.html`,
etc., Blank's more specific root-level versions would have silently
taken precedence for the homepage, RSS, JSON feed, and every post-type
single page. So this theme mirrors Blank's own file structure exactly
for those four, and reserves `_default/list.html` / `_default/single.html`
for what they still correctly handle: taxonomy/category list pages and
non-post content (e.g. standalone pages) respectively.

### The archive page is a different mechanism entirely

Initially we assumed `/archive/` was produced the same root-vs-`_default`
way, via the `ArchiveHTML` entry in `config.json`'s `outputFormats`.
It isn't — local testing showed `/archive/` actually falls back all the
way to the generic single-page template. The real mechanism, confirmed
by inspecting an exported `content/archive.md`, is a literal content
page with explicit front matter:

```yaml
title: "Archive"
type: archive
layout: list.archivehtml
url: /archive/
```

That's Hugo's ordinary **type + explicit layout** template lookup for
a regular page (`layouts/<type>/<layout>.html`, then
`layouts/_default/<layout>.html`) — a completely different lookup
chain from the home-page one above, and it does *not* have a
bare-root fallback the way `index.html` does. So the file that
actually renders `/archive/` is `layouts/_default/list.archivehtml.html`,
not the root-level copy. This theme keeps a copy at
`layouts/list.archivehtml.html` too, matching what Blank itself
ships (both locations) — presumably for compatibility across Hugo
versions where this lookup behavior may have shifted — but the
`_default/` copy is the one doing the work. If you only edit one of
them, edit `_default/`.

The `ArchiveHTML`/`ArchiveJSON` entries in `config.json` may be
legacy/unused for this — `outputs.page` only allows `HTML` anyway, so
an `ArchiveJSON` output through that mechanism wouldn't be reachable
for a literal page. Left them in since they're harmless and Blank
ships them, but don't rely on them for anything.

When adding any new template later, don't assume root-vs-`_default`
purely from Blank's file listing — check whether the content it
renders is the actual home page (root-sensitive) or a literal content
page with its own `type`/`layout` front matter (type+layout lookup,
`_default/` is what matters, root is just a compatibility copy).

## Content + sidebar layout

Homepage, single posts, and taxonomy/category list pages render
through `partials/layout-two-column.html`, which wraps the actual
content in `<div class="content">` and adds `<aside class="sidebar">`
next to it. Position is controlled by the `sidebar_left` site param
(boolean, default `false` = sidebar on the right):

```
<div class="layout layout--sidebar-left">   <!-- or layout--sidebar-right -->
	<div class="content">...</div>
	<aside class="sidebar">...</aside>
</div>
```

Style the two arrangements with CSS, e.g.:

```css
.layout { display: flex; gap: 2rem; }
.layout--sidebar-left  { flex-direction: row-reverse; }
.layout--sidebar-right { flex-direction: row; }
```

`static/custom.css` already implements this (plus a mobile breakpoint
that stacks content above sidebar below 640px, regardless of the
configured position — content-first reads better on narrow screens
than sidebar-first). It's deliberately unopinionated beyond structure:
no color palette, typography, or spacing scale — a foundation to
design on top of, not a finished look.

`_default/single.html` (the fallback for non-post content — in
practice, standalone pages, since Blank has no more specific
`page/single.html` to shadow it) and `404.html` both deliberately skip
this wrapper and render single-column, per your spec that a page can
be sidebar-free.

Note on `404.html`: Micro.blog caches 404 responses briefly, so a
change here can take a little while to show up when testing on the
live site — this is expected, not a sign the file isn't being picked
up (confirmed by Micro.blog's own team in a help thread).

The sidebar itself (`partials/sidebar.html`) defaults to a minimal
`h-card` identity block (name, avatar, site description, URL) built
from the same `.Site.Params.author` / `.Site.Params.description`
already wired up elsewhere in this theme — replace or extend it with
whatever else belongs there (recent posts, categories, etc).

**Toggling the position:** for local development, edit
`sidebar_left` directly in `config.json`. Once this theme is
installed as a plug-in, `plugin.json` exposes it as a checkbox
("Move sidebar to the left") on the theme's settings page instead —
useful even before public distribution, since Micro.blog shows this
settings page for any theme that ships a `plugin.json`, custom or
not. If you'd rather not commit to that now, the field is easy to
delete along with `plugin.json` itself.

### Microformats used

- `h-feed` on the post-list wrapper (`partials/entry-list.html`) —
  marks it as a feed of entries for feed-reading tools.
- `h-entry` / `p-name` / `dt-published` / `u-url` / `e-content` on
  each entry (already in place from before).
- `h-card` / `p-name` / `u-photo` / `p-note` / `u-url` in the sidebar
  identity block — standard vocabulary for "who is this site".

## Getting set up

1. Push this to its own GitHub repository.
2. In Micro.blog: Posts → Design → Edit Custom Themes → New Theme, and
   give it a Clone URL pointing at the repo.
3. Select the theme for your blog, and set **Hugo version** — see note
   below.
4. Iterate: edit locally, push, bump something (e.g. a comment or a
   version string) to bust Micro.blog's build cache, then check the
   live test blog. Expect a few minutes' lag between push and rebuild.

## Hugo version caveat: `.Site.Author` → `.Site.Params.author`

`.Site.Author` was deprecated in Hugo 0.124.0 and **removed in Hugo
0.141.0** (confirmed as of a Dec 2024 report from someone running
Micro.blog on Hugo 0.140). `head.html` already uses
`.Site.Params.author.username` instead, matching current Blank
source.

The bigger risk was Micro.blog's own built-in RSS, JSON feed, and
podcast feed templates (inherited from Blank), which reportedly still
referenced `.Site.Author` as of that report. Rather than depend on
those being fixed, this theme ships its own overrides so it's
self-contained regardless of Blank's internal state:

| Output format | Public file | Template in this repo |
|---|---|---|
| RSS | `feed.xml` | `layouts/index.xml` |
| JSON | `feed.json` | `layouts/index.json` |

Both use `.Site.Params.author`, not `.Site.Author`. These now live at
root level to match Blank's own `layouts/index.xml` / `layouts/index.json`
paths — see the structure note above for why that matters. If you
later add podcast support, Blank defines those at
`layouts/list.podcastxml.xml` (→ `podcast.xml`) and
`layouts/list.podcastjson.json` (→ `podcast.json`), also root-level,
not under `_default/`.

**Verify this locally** (see workflow below) before trusting it in
production.

## Local development

Rather than waiting 5–10 minutes per push for Micro.blog to rebuild,
develop against a local Hugo instance matching the exact version set
in Design → Hugo Version on your blog.

**Installing a pinned Hugo version on macOS:** Hugo's macOS releases
are signed `.pkg` installers, not tarballs — `hugo_extended_<version>
_darwin-universal.pkg` (skip `_withdeploy` unless you use `hugo
deploy`). Find the exact asset URL for your target version via
GitHub's API rather than guessing the filename (naming has changed
across Hugo versions):

```bash
curl -s https://api.github.com/repos/gohugoio/hugo/releases/tags/v0.158.0 \
  | grep browser_download_url | grep -i darwin
```

Install it (`sudo installer -pkg <file>.pkg -target /`, or
double-click). This writes to `/usr/local/bin/hugo`, which on Apple
Silicon usually sits *after* Homebrew's `/opt/homebrew/bin` in `PATH`
— so it won't silently replace a `brew`-installed `hugo`. Verify with
`which -a hugo` and `/usr/local/bin/hugo version`. To get a
version-tagged command without relying on `PATH` order:

```bash
mkdir -p ~/bin
ln -sf /usr/local/bin/hugo ~/bin/hugo-0.158
```

Note that a later `.pkg` install of a *different* version overwrites
`/usr/local/bin/hugo` again, so `hugo-0.158` would need re-linking if
you ever pin a second version this way. If you'd rather sidestep all
of this, a pinned Docker image (e.g. `klakegg/hugo:0.158.0`) avoids
touching the local install entirely — trade-off is the usual Docker
overhead for a one-off theme project.

Once you have the right `hugo` (or `hugo-0.158`) in hand:

1. From any Micro.blog dashboard page, use the "..." menu → Export →
   "Export theme and Markdown". You'll get an emailed link to a zip.
2. Copy the `content/` folder from that export into this repo.
3. Run `hugo-0.158 serve` (or whatever you named it) and iterate
   locally with instant reloads.
4. Commit, push, then use the sync button under Design → Edit Themes
   → your theme name to pull the changes into Micro.blog.

## Hugo version

`min_version` in `theme.toml` is set to `0.91` since nothing here uses
template features newer than that. For the actual **Design → Hugo
Version** setting on the blog itself, prefer a newer option over 0.91
where you reasonably can (a couple of relevant security fixes landed
in later Hugo releases) — but see the `.Site.Author` caveat below
before jumping all the way to 0.158.

## Replies / comments

`partials/entry-single.html` already includes the official
Conversation.js hook (used by both `post/single.html` and
`_default/single.html`), gated behind the `include_conversation` site
param (the same checkbox under Posts → Design):

```
{{ if .Site.Params.include_conversation }}
<script type="text/javascript" src="https://micro.blog/conversation.js?url={{ .Permalink }}"></script>
{{ end }}
```

This renders replies client-side using Micro.blog's own script and
sanitization — nothing in this theme parses or escapes reply content
itself. Style it via the `.microblog_conversation`, `.microblog_post`,
`.microblog_avatar`, `.microblog_time` classes in custom.css.

## Social `rel="me"` links

`head.html` includes `twitter_username`, `github_username`, and
`instagram_username` params. Twitter/X cross-posting was removed from
Micro.blog in March 2023 (Twitter API changes) but Manton announced on
2026-07-13 that X support is being re-enabled (Premium tier). The
`twitter_username` param and its `rel="me"` link are kept for this
reason — the link now points at `x.com` instead of `twitter.com`. If
Micro.blog ever renames the underlying site param, this is the file
to check.

## Notes ported from Micro.blog's theming conventions

- Homepage/list pages filter to `where .Site.RegularPages "Type" "post"`
  — without this, non-post content can leak into the list.
- Most Micro.blog posts are titleless microposts, so every `.Title`
  reference is wrapped in `{{ if .Title }}`.
- Micro.blog uses **categories**, not Hugo's default tags — taxonomy
  references use `.Params.categories`.
- `/archive/` is a route Micro.blog exposes on every blog regardless
  of whether it's in the nav, so `list.archivehtml.html` should exist
  in every theme.

## On "theme" vs "plug-in" terminology

Once this is ready for wider distribution, note that Micro.blog's own
team has said the "custom theme" UI is really meant for
blog-level customizations layered on an existing design, not for
publishing a standalone theme — publishing to the community now goes
through **Design → Edit Themes → New Plug-in**. The underlying
artifact (a GitHub repo of Hugo layouts, imported the same way) is
the same; only the distribution path and label differ. Worth
checking the current Plug-ins guide on help.micro.blog when you get
to that step, since this shifted from the older docs.
