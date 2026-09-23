---
name: google-drive
description: Work with Google Drive through the google_drive_* tools - search, read, create, organize, share, comment, and export files. Use when creating files in specific folders, moving or renaming, converting Markdown/CSV into Docs/Sheets, managing comments, or exporting a rendered PDF.
---

# Google Drive files

## Ids come from search, never guesses

Every tool takes Drive file ids. Get them from `google_drive_search` (or
`google_drive_list_folder` for a known folder,
`google_drive_list_recent_files` for "what was I just working on"), or from
the id in a URL the user pasted. `exact_name` beats `query` for a known
filename; add `mime_type_filter` to cut noise. Never invent an id or reuse
one from a stale conversation - re-search first.

## Folder targeting

- `google_drive_create_folder` takes `parent_folder_id`; without it the
  folder lands in My Drive root. When the user names a destination folder,
  resolve its id and pass it - a file created in root when a folder was
  specified is a wrong result even if the content is perfect.
- `google_drive_create_file` also takes `parent_folder_id`, so create files
  directly where they belong instead of creating then moving.
- The Docs/Sheets/Slides providers' own create tools CANNOT target a folder
  (their APIs have no parent parameter). When placement matters, either
  create the artifact there with `google_drive_create_file` (Markdown to
  Doc, CSV to Sheet), or create it and then `google_drive_move_or_rename`
  with `destination_folder_id`.
- `google_drive_move_or_rename` does either or both in one call; renaming
  does not change the id.
- Grant limits: this app's grant is `drive.file`, so moving into or writing
  files/folders this app never created (and the user never picked) can 403.
  That 403 is a permission fact, not a transient error; do not retry it.

## Markdown and CSV conversion

`google_drive_create_file` with `target_mime_type:
application/vnd.google-apps.document` converts Markdown (headings, lists,
tables) into a real Google Doc in one call; `...spreadsheet` converts CSV
into a Sheet. This is the fastest correct way to produce a formatted Doc:
prefer it over create-then-insert when the content is known up front.
`google_drive_update_file_content` converts in place the same way.

## Copying, metadata, and sharing

- `google_drive_copy_file` duplicates a file (optionally into a folder) -
  the right way to instantiate a template without touching the original.
- `google_drive_get_file_metadata` returns the full record (mime type,
  parents, size, owners, modified time); use it to confirm a move landed.
- `google_drive_get_file_permissions` lists who has access;
  `google_drive_share_file` grants a role to an email. Share only when the
  user asked; report the link rather than widening access speculatively.
- `google_drive_download_file` returns the raw bytes base64-encoded for
  non-Google files (images, PDFs, plain text).

## Reading and exporting

- `google_drive_read_file` returns text: Docs as Markdown, Sheets as CSV per
  tab, Slides as plain text. Use it to verify content after writes.
- `google_drive_export_file` returns rendered bytes base64-encoded: `pdf`
  for Docs/Sheets/Slides (Sheets PDF renders every tab, charts included),
  `png`/`svg` for Drawings, plus office formats (docx/xlsx/pptx). Use PDF
  when the user wants to see the final look; exports cap at 10 MB.
- For one slide as PNG use the google-slides `get_slide_thumbnail` tool, not
  Drive export.

## Comments

Comments live on the Drive file, so one tool set covers Docs, Sheets, and
Slides alike:

- `google_drive_list_comments` returns each comment's text, quoted document
  text, resolved state, and replies; it pages, so follow `nextPageToken`
  before concluding a thread does not exist.
- `google_drive_add_comment` adds a file-level comment (no text anchor).
- `google_drive_reply_to_comment` replies by comment id, and its
  `action: "resolve"` (content optional) is the ONLY way to resolve a
  thread; editing document text near a comment never resolves it. Leave
  already-resolved threads alone.

## Deleting

`google_drive_trash_file` is recoverable (~30 days) and works on files and
folders (a trashed folder takes its contents). Permanent deletion is
deliberately not available; never promise it.

## Verify your work

After creates and moves, confirm placement with `google_drive_list_folder`
on the destination or `google_drive_get_file_metadata` (check `parents`).
After content writes, `google_drive_read_file` is the ground truth for text
and `google_drive_export_file` (pdf) for the rendered result.
