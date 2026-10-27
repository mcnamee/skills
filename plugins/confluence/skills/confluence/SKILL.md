---
name: confluence
description: Search, read and (when writing is switched on) create and edit Confluence pages - including refreshing a designed page's figures, dates and statuses in place without breaking its layout, panels or colours, and changing a table's cells even inside a table filter or column macro - via the confluence MCP server, across one or two Confluence instances, and download a page's attachments or (writing on) upload one. Use when the user asks to find, read, summarise or pull content from Confluence (runbooks, handbooks, wiki pages, spaces), including when they ask for the tasks, action items, statuses, properties or child pages listed on a Confluence page (macro content), when they name a particular Confluence server, when they ask for a Confluence page to be saved into the local knowledge base, when they want a file attached to a page (a spreadsheet, document, deck, PDF or Markdown file) - often to open it with another plugin next - when they want a local file attached to a page or an image on a page replaced (an org chart SVG, say), or when they ask to create a page, update a page (including "update page X from this spreadsheet"), or add content (minutes, actions, a section) to a page.
---

# Confluence (via the `confluence` MCP server)

Requires the `confluence.py` MCP server (Confluence Data Center). It reads by
default; the page-writing tools exist only when the endpoint sets
`CONFLUENCE_ALLOW_WRITE=true`.
If its tools are not available, tell the user to wire it in first (see the
repo README) and to verify connectivity with `python confluence.py --check`.

## Tools

| Tool | Use for |
|---|---|
| `confluence_search` | Free-text search for pages by topic |
| `confluence_search_cql` | Advanced search with raw CQL (spaces, dates, labels) |
| `confluence_get_page` | Full content of one page by numeric ID |
| `confluence_get_page_by_title` | Full content by exact title + space key |
| `confluence_list_pages_under` | Children of a page (navigate a page tree) |
| `confluence_list_attachments` | The files attached to a page, and where each would download to |
| `confluence_download_attachment` | Save one attachment into the folder the matching plugin reads |
| `confluence_list_tables` | The tables on a page (even inside macros): number, heading, columns, rows |
| `confluence_create_page` | New page in a space, optionally under a parent *(writing on)* |
| `confluence_replace_text` | Change words, figures, dates or status lozenges in place, keeping the page's design *(writing on)* |
| `confluence_update_table` | Change cells of one table in place, and/or add rows *(writing on)* |
| `confluence_update_section` | Change one section (by heading or panel title), keeping the rest *(writing on)* |
| `confluence_append_to_page` | Add content to the end/start of a page, keeping the rest *(writing on)* |
| `confluence_update_page` | Replace a page's whole body and/or title *(writing on)* |
| `confluence_upload_attachment` | Attach a local file, or replace an attachment with a new version *(writing on)* |

The last seven are only in the tool list when writing is switched on. If the
user asks for a page to be created or changed and they are missing, say that
Confluence writing is off on this endpoint and that `CONFLUENCE_ALLOW_WRITE=true`
(in Claude Code's settings `env` block, then a restart) turns it on. Do not
work around it.

## Picking the server

The environment may have **two Confluence instances**. When it does, every tool
takes an optional `server` argument and the tool schema's `server` enum lists
the configured names (e.g. `Green` and `Blue`). No `server` argument in the
schema means only one instance is configured — ignore this section.

1. **Omit `server` by default.** The first server is the default, and an
   ordinary request ("search Confluence for the incident runbook") belongs
   there.
2. **Pass `server` when the user names one.** "find content about X on Blue",
   "check the Blue wiki", "what does Green say about Y" → `server: "Blue"` /
   `server: "Green"`. Matching is case-insensitive.
3. **Query both when the user asks for both** — "check both wikis", "is the
   policy the same on each?", or when a search on the default server comes back
   empty and the answer plausibly lives on the other one. Call the tool once per
   server and say which findings came from which. Do not silently substitute one
   server for the other.
4. **Content IDs are per-instance.** Page `393217` on Green is a different page
   from `393217` on Blue. Read a page from the same server the search that found
   it used — search results carry `server=<name>` on every line for exactly this
   reason.
5. **Always name the server in your answer** when two are configured, so the
   user knows which wiki a fact came from.

If a tool reports an unknown server, it lists the names that *are* configured —
use one of those rather than guessing, and tell the user if the instance they
asked for is not wired in.

## Macros: where a page's real content lives

A page body comes back as Markdown with its macros rendered, so most of this is
automatic. What matters is knowing when content is **missing** rather than
absent.

Confluence macros come in two kinds:

- **Stored in the page** — info/note/warning panels, expand, code, inline task
  lists, status lozenges. These always read. Inline tasks arrive as `- [ ]` and
  `- [x]` checkboxes with the assignee as `@username`, so counting or filtering
  someone's tasks is straightforward.
- **Generated when the page is displayed** — Task Report, Page Properties
  Report, Children Display, Page Tree, Jira Issues, Include Page, Excerpt
  Include. The page source holds only the macro's settings, so these have to be
  fetched by asking Confluence to render the page.

The tools do that on their own. Two things to watch:

1. **Read the `Body:` line** at the top of the output. `storage (page source)`
   plus a note means generated macros were not included; `view` or
   `export_view` means they were.
2. **A placeholder is a retry, not an answer.** Where the body says
   *"[Confluence task report (macro ...). Its content is generated ... not shown
   here]"*, the content exists and was not fetched. Call the same tool again
   with `body_format: "view"`. If it is still a placeholder, try
   `body_format: "export_view"`. Only after both should you tell the user the
   page has nothing on it.

Never report a page as empty, or a person as having no tasks, on the strength of
a placeholder — that is the one failure this argument exists to prevent.

`body_format` also takes `"storage"` (raw source, no macro output) if the user
explicitly wants to see how a page is built rather than what it shows.

## Workflow

1. Start with `confluence_search` using 2–4 topic keywords. Prefer fewer,
   more distinctive words over full sentences.
2. If the user names a space, date range or label, use `confluence_search_cql`
   instead, e.g. `space = DOCS AND text ~ "release notes" AND lastmodified >= now("-30d")`.
3. Fetch the winning result with `confluence_get_page` (by the ID from the
   search results, on the same server) and answer from the page body. Quote the
   page title and ID — plus the server, if there are two — so the user can find
   it.
4. For "everything under X" requests, walk `confluence_list_pages_under`.
5. **"What tasks are on page X?" / "what is assigned to Jane on X?"** → read the
   page and answer from the checkboxes and the task-report rows. If the body
   carries a macro placeholder instead, retry with `body_format: "view"` before
   answering (see [Macros](#macros-where-a-pages-real-content-lives)). Quote the
   task text, who it is assigned to and any due date, and say whether each one
   is ticked.

## Attachments: getting a file off a page

1. `confluence_list_attachments` on the page (by `page_id`, or `title` +
   `space`). Each line gives the name, type, size, version and the local
   folder it would go to, or why it cannot be downloaded.
2. `confluence_download_attachment` with the exact `filename` (or
   `attachment_id`). The file lands where the plugin for its type looks:

   | Type | Folder | Next tool |
   |---|---|---|
   | `.xlsx` / `.xlsm` | `documents\excel` | `excel_list_sheets`, `excel_read_table`, `excel_read_range` |
   | `.docx` | `documents\word` | `msword_open` |
   | `.pptx` | `documents\powerpoint` | `powerpoint_open` |
   | `.pdf` | `documents\pdf` | `convert_pdf_to_markdown` |
   | `.md` | `knowledge\confluence` | `kb_index` |

3. Carry straight on with the other plugin, passing the **file name** the
   download reported (not the full path - each plugin resolves names inside
   its own folder). "Get the budget from page X and total column D" is the
   download followed by an `excel_*` call, in one turn.

Images (`.svg`, `.png`, `.jpg`, `.jpeg`, `.gif`) download to
`documents\images`. No plugin opens them; it is where `/org-chart` writes.

## Attachments: putting a file on a page *(writing on)*

`confluence_upload_attachment` reads ONLY from the folders in the table above
(plus `documents\images`). Pass `file` as the file name - it is looked up in
the folder for its type - or a full path inside one of them. A file anywhere
else is refused: say so, and ask the user to save it into the right folder
rather than copying it there yourself.

**Replacing an image on a page** ("swap the org chart on page Z for this one")
is an upload, not a page edit. The page shows the attachment by name, so a new
version of it appears in place:

1. `confluence_list_attachments` on page Z to find the image's exact name.
   If more than one image could be the one meant, ask which.
2. `confluence_upload_attachment` with `file` (the local name), `attach_as`
   (the name on the page, when it differs) and `overwrite: true`. Set
   `overwrite` only when the user has asked for the replacement - without it the
   tool refuses and names the existing attachment, which is the check you want.
3. Report the new version number. Do not edit the page body: it already points
   at the attachment.

A **new** image is attached but not displayed. To show it, add
`![](<file name.svg>)` (angle brackets when the name has spaces) with
`confluence_update_section` or `confluence_append_to_page`.

- **Never overwrite by default.** If the file already exists the download is
  refused. When the user wants the newest copy ("get the latest version"),
  retry with `overwrite: true`; when they want both, use `save_as`. If it is not
  clear which, ask in one line.
- Other file types are listed but cannot be downloaded - say so rather than
  suggesting a workaround.
- Downloading changes nothing in Confluence, so it works with writing off.

## Writing pages (only when the write tools are present)

Writing to a wiki publishes as the user, so:

1. **Only write what the user asked for.** Never create or edit a page as a
   side effect of research, and never "tidy up" a page you were reading.
2. **Show the content first** - draft it in the chat, get a yes, then write -
   unless the user has already given you the exact text or said to go ahead.
3. **Change data in place; never rebuild a page to change its data.** The
   page you read is Markdown, but the page itself has layouts, panel colours,
   coloured text, highlighted cells and lozenges that Markdown cannot carry.
   Writing Markdown back over it - with `update_page`, or `update_section` in
   `replace` mode - is what breaks a page's formatting. So:
   - **Figures, dates, statuses or words anywhere** ("update the dashboard
     from this spreadsheet", "change the spend to $1.4m", "set the status to
     At risk") → `confluence_replace_text`. It swaps only the characters you
     name and keeps everything around them. See
     [Updating a page from data](#updating-a-page-from-data-a-spreadsheet-say).
   - **Cells of a table, by row and column** ("mark Travel as 120 in Q3",
     "add a row for Training") → `confluence_list_tables`, then
     `confluence_update_table`. Take the table number and exact column names
     from the listing. `match` picks rows by a column value; `set` gives the
     new values. Put every change for one table in ONE call's `updates` list.
     A `[status]` cell takes a label or `{"text": "At risk", "colour":
     "Yellow"}`, a `[date]` cell a date (`YYYY-MM-DD`): the lozenge stays.
     If it refuses (ambiguous row, merged cell, unknown column), fix that one
     thing from the error; for a `[macro]` cell, change its words with
     `confluence_replace_text` instead, or ask before passing
     `allow_macro_removal: true`. A Jira issues table or page properties
     report is generated by Confluence and cannot be edited - say so.
   - **Adding** (minutes, actions, a paragraph) → `confluence_append_to_page`,
     or `confluence_update_section` with `mode: "append"` / `"prepend"` to add
     under one heading or inside one panel.
   - **Rewriting one part in new words** ("rewrite the Director's notes") →
     `confluence_update_section` with `section: "X"` (a heading's text, or a
     panel/expand title). A heading's section runs to the next heading of the
     same or higher level, sub-headings included - if the user only meant the
     text directly under it, check first. If `section` matches nothing or
     several, the error lists the page's sections.
   - **A full rewrite or a title change** → `confluence_update_page`, after
     reading the page, passing the version you read as `expected_version`.
4. **When a rewrite is refused for losing formatting, stop and ask.**
   `update_section` (replace) and `update_page` refuse to save content that
   would drop the page's macros or formatting, and list what would go
   ("panel macro 2 -> 0; date lozenge 3 -> 0; page layout section 2 -> 0").
   That refusal is the guard doing its job, not an obstacle:
   - If you were changing data, switch to `confluence_replace_text` /
     `confluence_update_table` - the error says so.
   - If the user really wants a rewrite, either read the page with
     `body_format: "storage"` and send it back with `content_format:
     "storage"`, changing only the text and keeping every tag, or tell the user
     exactly what the error says would be lost and pass
     `allow_formatting_loss: true` only once they agree.
   - Never pass `allow_formatting_loss` (or the older `allow_macro_removal`)
     on your own initiative. A refused write because the page moved on means
     read it again, never force it.
5. **Say what you did:** title, page ID, new version and URL, plus the list of
   changes, as the tool reports them.

## Updating a page from data (a spreadsheet, say)

"Analyse this spreadsheet and update the dashboard page" is the case that used
to break pages. Do it like this:

1. **Read both.** The spreadsheet with the `excel_*` tools; the page with
   `confluence_get_page` (and `confluence_list_tables` if it has tables).
2. **Work out the changes as old -> new pairs** from what the page says now:
   `$1.2m -> $1.4m`, `ON TRACK -> At risk (Yellow)`, `2026-09-30 -> 2026-10-26`.
   Show the user that list before writing, unless they said to go ahead.
3. **Write them in one call**, cells first if a value lives in a table:
   - table cells by row/column → `confluence_update_table` (one call per table);
   - everything else → ONE `confluence_replace_text` call with every pair:
     ```json
     {"page_id": "1234", "replacements": [
       {"find": "$1.2m", "replace": "$1.4m"},
       {"find": "ON TRACK", "replace": "At risk", "colour": "Yellow"},
       {"find": "2026-09-30", "replace": "2026-10-26"}
     ]}
     ```
4. **Read the reply.** It lists each change and where it landed. If a call is
   refused, nothing was saved - fix the one item the error names and send the
   whole list again.

How `find` matches, so you can copy it straight from the page as you read it:

- **Whole words and numbers**, so `12` never matches inside `120`. Spacing,
  curly vs straight quotes and dashes are forgiven, and case too when nothing
  matches exactly.
- **Status lozenges** read as `**[ON TRACK]**`; pass the label (`ON TRACK` is
  fine) and, if the colour should change, `colour`: `Grey`, `Red`, `Yellow`
  (RAG amber), `Green` or `Blue`.
- **Date lozenges** read as `2026-10-26`; a `find` that is that date matches
  the lozenge, and `replace` must then be a date (`YYYY-MM-DD`, or `26 Oct
  2026`). Never pass an all-number date like `10/11/2026` - convert it first.
- **It must appear once.** If it appears more often the error lists each place:
  use longer text that is unique, add `section` (a heading or panel title), or
  set `all_matches: true` when every one should change.
- **Split by formatting** (`**Total spend:** $1.2m` is bold then plain): the
  error names the pieces; replace the piece that changes (`$1.2m`).
- `replace` is plain text and takes on the formatting of what it replaces -
  write `$1.4m`, not `**$1.4m**`. It must be one line.
- Text in a code block, a macro's settings, and content Confluence generates
  when the page is shown (Jira tables, page properties or task reports) cannot
  be changed this way; say so if the user expected it to change.

Bodies are Markdown by default: headings, bold/italic, `code`, links, nested
lists, tables, block quotes, fenced code (becomes a code macro), `- [ ] task`
(becomes a real Confluence task), and `> [!NOTE] Title` / `[!TIP]` /
`[!WARNING]` quotes (become panels). Write action items as `- [ ]` tasks so they
show up in Confluence task reports. Page titles must be unique within a space;
if a create is refused for that reason, ask the user for another title.

## Saving to the knowledge base

Reading a page does **not** save it. `confluence_get_page` and
`confluence_get_page_by_title` take `save_to_kb`, false by default; set it to
true **only when the user asks for the page to be kept** — "save this to the
knowledge base", "add that page to the KB", "pull the runbook in for offline
search". The file lands in `H:\Eva\knowledge\confluence` and the tool reports
the path.

- **Never set it while researching.** Pages you open to answer a question are
  not the user's filing decision, and a knowledge base full of pages nobody
  chose returns irrelevant answers later.
- **Asked after the fact** ("save that page") → call the same tool again for
  that page with `save_to_kb: true`. Re-reading is cheap; guessing is not.
- **Say what you saved** — the title and the path — so the user knows what is
  now in the index. Suggest `kb_index` if they want it searchable immediately.
- If the user says something is worth keeping and you are not sure they meant
  the KB, ask in one line rather than saving.

## Notes

- Without `CONFLUENCE_ALLOW_WRITE=true` the server is read-only and cannot
  create or edit pages or upload attachments; downloading attachments still
  works.
- With two instances configured, saved files are named
  `Confluence <server> - <title>.md`, so pages that share a title on both
  instances stay separate. Saving is off at the server if
  `CONFLUENCE_KB_DIR` is `off` — the tool says so rather than failing quietly.
- Long pages may be truncated in the returned text if `CONFLUENCE_MAX_BODY` is
  set;
  say so if an answer might sit past the truncation point.
- A rendered body is a snapshot of what Confluence showed at the moment of the
  read. A task report reflects the state of tasks right then, so say when you
  read it if the user is acting on the list.
- Saving a page keeps the same body that was read, so a page saved with its
  macro content keeps that content in the knowledge base.
