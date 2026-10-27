# TRLC Preview v2: Structure Editing, TOC, Markdown — Design

**Status:** agreed with the user on 2026-10-28 (brainstorming session; mockups were reviewed in the browser).
**Builds on:** the TRLC preview from `docs/superpowers/plans/2026-10-27-trlc-preview.md` (branch `feat/trlc-preview`). Read that plan's "Design decisions" and "Global Constraints" first. Everything there still holds unless this document says otherwise.

## Intent

The preview already shows a `.trlc` file with sections as `#` headings and lets the user edit fields and add records. The user wants to work on the **structure** of a requirements file in the preview too, without hand-editing braces and indentation:

- put new sections in;
- reorganise records and sections by dragging them;
- dissolve a section without losing its content;
- find their way around long files.

They also want rich-text descriptions: certain string fields are Markdown, and they should be shown and edited WYSIWYG instead of as raw text.

Success means a user can restructure a large requirements file and write formatted descriptions entirely in the preview. The `.trlc` file stays valid and readable, and every change is an ordinary editor edit, so Ctrl+Z undoes it.

## Scope

In scope (user's numbering from the request):

| # | Feature |
|---|---------|
| 2 | Show which attributes are mandatory |
| 4 | Insert sections (empty, or by moving records in); dissolve sections, keeping their content |
| 5 | Indent nested content visually in the preview |
| 6 | A table of contents (TOC) built from the sections, always on top, click to jump |
| 7 | Markdown string fields, rendered, with a small WYSIWYG editor (bold, italic, table, …) |

Dropped by the user (they already work): adding missing attributes to existing records (1), removing optional attributes (3).

## Goals

### Mandatory attributes (2)

- Every required component (not `optional`) shows a red `*` after its name. This applies to set and unset fields alike, and frozen fields are no exception.
- A record that lacks a required component also shows a short hint at the record, e.g. `Pflichtfeld „description“ fehlt`. The UI language is English, so the actual text is `Required field "description" is missing`.

### Sections: insert (4)

- Every section, and the file level, has a "+ Section" button next to "+ Add record". It asks for a title and inserts an empty `section "<title>" {` / `}` at the end of that section (or the file).
- The title is a TRLC string. Use the existing string formatting (`format_value` "string" rules), so quotes and other special characters are handled.

### Drag & drop (4, chosen variant "B")

- Every record and every section has a drag handle (⠿).
- A record can be dropped:
  - between two items (records or sections) of any section;
  - into an empty section;
  - at file level.

  The drop target is shown as an insertion line. This also covers reordering inside a section.
- A section can be dragged the same way, **with all its content** (records and nested sections). Dropping a section into itself or into one of its descendants is not allowed. Show no drop target there.
- In the source, the moved item is cut and pasted at the target position. Its indentation is adjusted to the target nesting level (see "Re-indentation" below).
- Comments belong to the item and move with it:
  - comment lines directly above the item, with no blank line in between;
  - trailing comments on the item's lines.

  *Planner's call:* blank lines are not carried.
- "Moving records into a new section" is done by inserting an empty section ("+ Section") and dragging records into it. No separate "wrap selection" feature.

### Sections: dissolve and rename (4)

- Each section heading has a "dissolve" action (🗑). It removes only the `section "…" {` line part and its closing `}`. All contained records and nested sections stay where they are, move up one nesting level, and are re-indented one level outward.
- Comments on dissolve (*planner's call*):
  - Comment lines directly above the `section` line stay where they are.
  - A trailing comment on the `section "…" {` line or on the closing `}` is kept as a standalone comment line at the dedented position.
  - No comment is ever dropped.
- Each section heading has a rename action (✎). It replaces the title string.
- *Planner's call:* dissolve asks for no confirmation, because it loses no content and Ctrl+Z restores it.

### Re-indentation (applies to move and dissolve)

- The indent step is derived from the file: the difference between a section's indent and its first child's indent. If that can't be determined, use 4 spaces (the plan's existing default).
- Lines are re-indented by adding or removing the step per level changed.
- **Lines inside a multi-line string token (`'''…'''` / `"""…"""`)** are shifted along with the item, by the same amount on every continuation line. This keeps the Markdown body aligned with the field after a move or dissolve. TRLC dedents those lines by their common indentation, so a uniform shift leaves the value unchanged.
  - When dedenting, if any continuation line of that string has less leading whitespace than the step, that string's lines are left untouched instead. Shifting only some of them would change the value. The source looks misaligned, but the value is correct.
  - *Planner's call.*
- A line outside strings with less leading whitespace than the step is dedented as far as it can go. This should only happen for unusual hand formatting.
- **Items that share a line** (e.g. `MyRecord r2 { x = 3 y = 4 }`, or `}` followed by the next item on the same line) are cut by token range, not by whole lines. The paste puts the item on fresh lines at the target indentation, and a line left empty by the cut is removed. *Planner's call.*
- **Correctness check:**
  - Tests: real-parse tests apply each structural edit and re-parse. They assert that every moved record's fields and values are identical and that only the intended section path changed.
  - Runtime: the server applies the edit to the text in memory and re-lexes and re-scans it before returning the WorkspaceEdit. It checks for no lexer errors, balanced braces, and the same set of record names and assignments as before the edit. If any check fails, it refuses the edit.
  - A full semantic re-parse at runtime is not required. It would need the whole scope.
  - *Planner's call.*

### Preview indentation (5)

- Each nesting level's content (records and sections) is visually indented, with a thin vertical line on the left that marks the section's extent.
- The `#` headings from v1 stay as they are.

### TOC (6)

- At the top of the preview, sticky, so it doesn't scroll away.
- It lists all sections nested by depth.
- Clicking an entry scrolls the preview to that section's heading.
- It can be collapsed (▾/▸). *Planner's call:* the collapsed state is remembered per webview session only.
- With no sections in the file, the TOC is hidden.

### Markdown fields (7)

- A new setting, `trlcServer.preview.markdownComponents`, decides which string components are Markdown. It is an array of component names, default `["description"]`, scope `resource`. Model it on `trlcServer.hover.descriptionComponent`: the `package.json` description is the single source of truth for its wording; there is a `ServerConfig` tuple field and a `from_dict` mapping. Only components of TRLC string kind are affected. Other kinds ignore the setting.
- **Display:** a Markdown field is shown rendered. Support GitHub Flavored Markdown:
  - paragraphs, **bold**, *italic*, ~~strikethrough~~;
  - headings, bullet and numbered lists;
  - links, inline code and code blocks;
  - tables.
- **Edit:** clicking into the field makes it editable in place, still rendered (WYSIWYG). A small toolbar sits on top. Toolbar actions:

  | Action | Detail |
  |--------|--------|
  | Formatting | Bold, Italic, Strikethrough |
  | Heading | — |
  | Lists | Bullet list, Numbered list |
  | Link | — |
  | Table | Insert table; add/remove row and column |
  | Code | Inline code / code block |

  The value is committed when the field loses focus, like every other field. Leaving without changes commits nothing.
- **Storage:**
  - The value is written as a TRLC triple-quoted string. Use `'''…'''` unless the text needs `"""…"""`; the existing quote-choice and re-lex guard logic applies.
  - The text starts on its own line after the opening quotes and is indented one step deeper than the field, as in the reviewed mockup:

    ```
    description = '''
        Das Fahrzeug **muss** innerhalb von *40 m* stehen.

        - trockene Fahrbahn
        - Beladung bis `zGG`

        | v [km/h] | Weg [m] |
        | -------- | ------- |
        | 100      | 40      |
    '''
    ```
- **Normalisation is accepted:** the editor may re-serialise the Markdown. For example it may change `_x_` to `*x*` or re-pad table columns, even if only one word changed. The user explicitly accepted noisier diffs.
- **But meaning must survive the round trip.** Markdown depends on indentation: nested lists and code blocks. TRLC's triple-quote normalisation strips the first line and dedents by the common indentation of the later lines (`trlc/lexer.py` `triple_quoted_string_value`). That can flatten nesting, e.g. when every line after the first is a nested list item.
  - The serialiser must produce a literal whose TRLC value equals the Markdown text, ignoring trailing whitespace per line and leading/trailing blank lines.
  - If no representation achieves that, refuse the edit with a clear error, keep the editor content, and do not write anything.
  - This needs tests with nested lists, code blocks, and a first line that is a list item.

## Non-goals

- No multi-select or "wrap selected records in a section" command. Use "+ Section" plus drag & drop.
- No keyboard alternative for drag & drop in this iteration. *Planner's call;* the editor's text remains the fallback.
- No Markdown for `.rsl` doc comments or the hover. The preview only.
- No `Markup_String` support. The user doesn't use it. Markdown fields are chosen only by the setting.
- No drag & drop between files.
- No undo inside the preview beyond VS Code's normal editor undo.

## Anti-goals (failure even if it "works")

- Any structural edit that changes a record's field values, drops a comment that belonged to a moved item, or leaves a file that no longer parses.
- A Markdown edit that silently changes the meaning of the text, e.g. flattened nesting or lost table rows. Reformatting is fine; meaning changes are not.
- HTML or script injection through Markdown content. Rendered Markdown must be sanitised: no raw HTML execution and no event handlers. Links must not navigate the webview; open them externally or not at all.
- Loading anything from the network. The webview CSP forbids it, and the extension must work offline. The editor library must be bundled.

## Constraints

- Everything from v1's Global Constraints still applies:
  - one `context.snapshot()` per request;
  - edits are refused when stale or when the file is locked by syntax errors;
  - edits are versioned and serialised on the client;
  - UTF-16 positions;
  - the file's EOL is used;
  - BMW GPL headers;
  - ruff at 79 columns.
- The server stays the only place that knows TRLC. Structural operations (insert section, move, dissolve, rename) are computed by the server as WorkspaceEdits from the token stream, like v1's `preview_edits.py`. The client only sends intents, e.g. "move record `r1` to section 3 before item X", and applies the returned edit.
- A move must be a single WorkspaceEdit (cut + paste in one), so Ctrl+Z undoes it in one step.
- Item identity on the wire:
  - Records are identified by name (unique per package).
  - Sections are identified by the existing ordinal `sectionId`.
  - Drop positions are given as target section id (or null for the file level) plus the item to insert before (or null for the end).
  - The server re-validates everything against the current parse.
- The Markdown editor library and the Markdown renderer must be bundled into `preview.js` (esbuild, `iife`, browser). They must work under the existing CSP: nonce scripts, no `eval`, `style-src 'unsafe-inline'` is allowed. Bundle size should stay reasonable. *Planner's call:* under ~1 MB minified is fine for a local extension.
- The Bazel `preview_bundle` target and the e2e copy already include `preview.js`. A new npm dependency must be added to `package.json` / `pnpm-lock.yaml`, so the Bazel `npm_link_all_packages` picks it up. Do not commit `package-lock.json`.

## Left to the builder

- Choice of WYSIWYG/Markdown library (e.g. TipTap with a Markdown extension, Milkdown, or similar). Pick for GFM tables, CSP compatibility, offline bundling and size, and say why in the plan.
- Exact wire format and method names for the new requests, e.g. `trlc/previewInsertSection`, `trlc/previewMove`, `trlc/previewDissolveSection`, `trlc/previewRenameSection`, and how the `markdown` flag reaches the client. A per-field `markdown: true` in the model is the obvious route.
- Drag & drop implementation in the webview (HTML5 DnD or pointer events), the drop-indicator styling, and how dragging interacts with the existing focus/pending-render logic. A model must never re-render mid-drag and swallow the drop.
- How the TOC is rendered (part of `renderPreview` or separate) and its exact styling, using VS Code theme variables.
- Test strategy details, following v1: pure server modules with real-parse tests that re-parse after applying edits; pure client helpers under mocha.
