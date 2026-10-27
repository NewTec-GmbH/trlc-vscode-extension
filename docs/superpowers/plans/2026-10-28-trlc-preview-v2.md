# TRLC Preview v2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend the TRLC preview with:
- mandatory-field markers;
- section insert, rename and dissolve;
- drag & drop of records and sections;
- nested indentation and a sticky TOC;
- WYSIWYG Markdown editing for configured string fields.

**Architecture:** The server remains the only place that knows TRLC.
- New pure modules compute structural WorkspaceEdits from the token stream:
  - `preview_layout.py`: item extents, indent step, re-indentation;
  - `preview_structure.py`: insert, rename, dissolve, move, plus a runtime verification guard.
- Markdown fields are flagged in the model by a new setting and serialised as triple-quoted strings by a round-trip-checked formatter.
- The client sends intents through the existing serialised, versioned edit queue.
- The webview gains TOC, drop zones and section forms.
- The webview hosts a bundled Milkdown editor for Markdown fields.

**Tech Stack:**
- Server: Python 3.10–3.13, pygls 2.1.1, trlc ≥ 3.0.0, pytest / pytest-lsp.
- Client: TypeScript, VS Code Webview API, HTML5 drag & drop, `@milkdown/kit` (new, bundled with esbuild), mocha + sinon.

**Spec:** `docs/superpowers/specs/2026-10-28-trlc-preview-v2-design.md` — read it in full, including its planner's calls. v1 plan: `docs/superpowers/plans/2026-10-27-trlc-preview.md`.

## Global Constraints

- All v1 Global Constraints still apply:
  - BMW GPL header on every new file; ruff at 79 columns; pylint clean.
  - Client TS files flat in `src/client/src/`.
  - One `context.snapshot()` per request.
  - Inserted text uses the file's EOL.
  - Positions converted to UTF-16 with `preview_edits._to_utf16`.
- Every structural edit is ONE WorkspaceEdit returned through `handlers/preview.py` `_edit_result`, so it is versioned and one Ctrl+Z undoes it. It is refused with `_NOT_PARSED` when the file is locked and with code `stale` when the parse is behind, exactly like `previewSetField`.
- Server re-validates every client intent against the current parse: record names, `sectionId`s, and `before` items must be direct children of the target.
- Comments are never dropped by any structural edit.
- Lines inside multi-line string tokens are shifted uniformly with their item. If a dedent would need more whitespace than any continuation line of that string has, that string's lines are left untouched.
- New setting `trlcServer.preview.markdownComponents`:
  - type `array` of `string`, default `["description"]`, scope `resource`;
  - `ServerConfig` field `preview_markdown_components: Tuple[str, ...] = ("description",)`.
- UI text is English.
- Strings:
  - missing-field hint: `Required field "<name>" is missing`;
  - section buttons: `+ Section`, rename `✎`, dissolve `🗑` (title attribute `Dissolve section`), drag handle `⠿`.
- No network loads; the Markdown editor is bundled into `src/client/out/preview.js`. The minified bundle stays under 1 MB. It must not contain `eval(` or `new Function(`, and it must work under the existing CSP.
- No Bazel locally. Verification commands:

  | Purpose | Command |
  |---------|---------|
  | Python tests | `PYTHONPATH=src .venv/Scripts/python -m pytest tests -q --ignore=tests/integration/test_bazel_mode.py` |
  | Client tests | `npx mocha` |
  | Typecheck (client) | `npx tsc -p tsconfig.test.json --noEmit` |
  | Typecheck (e2e) | `npx tsc -p tests/e2e/tsconfig.json --noEmit` |
  | Lint (TS) | `npx eslint src/client/src` |
  | Lint (Python) | `.venv/Scripts/ruff check src tests && .venv/Scripts/ruff format --check src tests` |
  | Bundles | esbuild into the scratchpad dir |

  Never commit `package-lock.json`.

## Review Focus

1. **Dragging a record that shares a line with another** (`MyRecord r2 { x = 3 y = 4 } MyRecord r3 { … }`): only that record moves, onto fresh lines. Test in Task 5: `test_move_record_sharing_a_line`.
2. **Dissolving a section that contains a record with a `'''` string** whose continuation lines are indented less than the step: the string value is unchanged and the file re-parses. Test in Task 4: `test_dissolve_keeps_underindented_multiline_string`.
3. **A Markdown value whose lines after the first are all nested** (`"- a\n  - b"`): TRLC cannot represent it, so the edit is refused and nothing is written. Test in Task 2: `test_markdown_unrepresentable_nesting_rejected`.
4. **A model refresh arriving while the user is dragging**: no re-render until the drop or cancel, so the drop lands. Test in Task 9: `"does not render while dragging"`.
5. **Moving a section into its own descendant via a crafted request**: the server rejects it even though the client never offers that drop zone. Test in Task 5: `test_move_section_into_descendant_rejected`.

---

### Task 1: Markdown setting and model flag

**Files:**
- Modify:
  - `package.json` (`contributes.configuration`)
  - `src/server/server_config.py`
  - `src/server/preview_model.py`
  - `src/server/handlers/preview.py` (pass the config into `build_preview_model`)
- Test:
  - `tests/unit/test_server_config.py`
  - `tests/unit/test_preview_model.py`
  - `tests/unit/handlers/test_preview.py`

**Interfaces:**
- Produces:
  - `ServerConfig.preview_markdown_components: Tuple[str, ...]`, read from `preview.markdownComponents` in `from_dict`. A non-list or a non-string entry falls back to the default, mirroring `hover.descriptionComponent` handling.
  - `build_preview_model(uri: str, parsed, markdown_components: "tuple[str, ...]" = ()) -> dict`. Every field dict gains `"markdown": bool`, which is True iff `kind == "string"` and the field name is in `markdown_components`.
  - The handler passes `ls.get_config_for_uri(uri).preview_markdown_components`.

- [ ] **Step 1: Write failing tests.**
  - `test_preview_markdown_components_default` → `("description",)`.
  - `test_preview_markdown_components_from_dict` → `{"preview": {"markdownComponents": ["description", "rationale"]}}` gives `("description", "rationale")`.
  - `test_preview_markdown_components_invalid_falls_back`.
  - `test_markdown_flag_only_for_configured_string_fields` (model): a string field `description` → True; the integer field `x` → False even if listed.
  - `test_handler_model_uses_config_markdown_components`.
- [ ] **Step 2: Run, expect FAIL.** `PYTHONPATH=src .venv/Scripts/python -m pytest tests/unit/test_server_config.py tests/unit/test_preview_model.py tests/unit/handlers/test_preview.py -q`
- [ ] **Step 3: Implement.** The `package.json` description is the single source of truth: `"String components rendered and edited as GitHub Flavored Markdown in the TRLC preview."`
- [ ] **Step 4: Run, expect PASS,** then the full Python suite plus ruff.
- [ ] **Step 5: Commit** `feat: preview.markdownComponents setting and markdown field flag`

---

### Task 2: Markdown serialisation on set-field

**Files:**
- Modify:
  - `src/server/preview_values.py`
  - `src/server/preview_edits.py` (`set_field_edits`)
  - `src/server/handlers/preview.py` (pass components)
- Test: `tests/unit/test_preview_values.py`, `tests/unit/test_preview_edits.py`

**Interfaces:**
- Consumes: Task 1 setting.
- Produces:
  - `format_markdown(raw: str, indent: str, eol: str) -> str` in `preview_values.py`. It returns a triple-quoted literal laid out as:
    ```
    '''<eol>
    <indent+STEP>line…<eol>
    <indent>'''
    ```
    - It uses `"""` if the text contains `'''`.
    - Empty lines are emitted without indentation.
    - The input is normalised first: CRLF/CR → LF, trailing whitespace stripped per line, leading/trailing blank lines dropped.
    - It raises `PreviewEditError("This Markdown cannot be stored as a TRLC string (indentation of the first lines would be lost).")` unless the re-lexed TRLC value of the literal (reuse `_lexes_to`) equals the normalised input exactly.
  - `set_field_edits(..., markdown_components: "tuple[str, ...]" = ())`: when the field's kind is `"string"` and its name is in `markdown_components`, it formats with `format_markdown(value, field_indent, eol)` instead of `format_value`. `field_indent` is the indent of the field's line (`_field_indent`).

- [ ] **Step 1: Write failing tests.**
  - `test_markdown_round_trip` (parametrised): a paragraph with `**bold**`; a GFM table; a code block; `"intro\n\n- a\n  - b\n- c"`. Each must re-lex to the normalised input.
  - `test_markdown_layout`: `format_markdown("a\n\nb", "    ", "\n") == "'''\n        a\n\n        b\n    '''"`.
  - `test_markdown_unrepresentable_nesting_rejected`: `"- a\n  - b"` raises `PreviewEditError`.
  - `test_markdown_uses_double_quotes_when_needed`.
  - Edits:
    - `test_set_markdown_field_writes_block`: re-parse; the value equals the input; the closing quotes sit at the field's indent.
    - `test_set_markdown_field_crlf`.
- [ ] **Step 2: Run, expect FAIL.** `PYTHONPATH=src .venv/Scripts/python -m pytest tests/unit/test_preview_values.py tests/unit/test_preview_edits.py -q`
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run, expect PASS,** then the full Python suite plus ruff/pylint.
- [ ] **Step 5: Commit** `feat(server): store markdown fields as indented triple-quoted strings`

---

### Task 3: Layout helpers (`preview_layout.py`)

**Files:**
- Create: `src/server/preview_layout.py`
- Test: `tests/unit/test_preview_layout.py`

**Interfaces:**
- Consumes: `FileStructure`, `RecordSpan`, `SectionSpan` (`preview_scan.py`).
- Produces:
  ```python
  @dataclass(frozen=True)
  class Extent:
      start: int          # offset in content (inclusive)
      end: int            # offset in content (exclusive)
      whole_lines: bool   # True: [start, end) are complete lines incl. the final EOL

  def record_extent(structure: FileStructure, content: str, rec: RecordSpan) -> Extent
  def section_extent(structure: FileStructure, content: str, sec: SectionSpan) -> Extent  # requires sec.close_tok
  def indent_step(structure: FileStructure, content: str) -> str
  def shift_block(content: str, tokens: list, start: int, end: int,
                  levels: int, step: str) -> str
  ```

  **Extent rules:**
  - An item begins at its first token: for a record, the type name, including a qualified `Pkg.Type`; for a section, the `section` keyword.
  - It ends at its closing `}`.
  - Comment lines directly above (no blank line between) are included.
  - A trailing comment after the closing `}` on the same line is included.
  - `whole_lines` is True when the item (with those comments) is alone on its lines. The extent then spans from the line start to after the EOL of the last line.
  - Otherwise the extent is the exact token range (+ trailing comment).

  **`indent_step`:** the first section's first child indent minus the section's indent; else the first record field's indent minus the record's indent; else `"    "`.

  **`shift_block`:** returns `content[start:end]` with every line moved by `levels` steps (negative = dedent).
  - Dedent removes at most the available leading whitespace.
  - Continuation lines of a multi-line STRING token are shifted by the same amount. Exception: when dedenting and any of that token's continuation lines has less leading whitespace than `|levels|*len(step)`, all of that token's lines are left as they are.

- [ ] **Step 1: Write failing tests** with real parses (reuse the tmp-file fixture pattern from `tests/unit/test_preview_scan.py`):
  - `test_record_extent_whole_lines_with_leading_comment`
  - `test_record_extent_inline_record_is_token_range`
  - `test_record_extent_qualified_type`
  - `test_section_extent_includes_trailing_comment`
  - `test_indent_step_from_file` (2-space file → `"  "`)
  - `test_indent_step_default`
  - `test_shift_block_indent_and_dedent`
  - `test_shift_block_moves_multiline_string_uniformly`
  - `test_shift_block_leaves_underindented_string_untouched`. The two string tests also assert the re-lexed STRING token values are unchanged.
- [ ] **Step 2: Run, expect FAIL.** `PYTHONPATH=src .venv/Scripts/python -m pytest tests/unit/test_preview_layout.py -q`
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run, expect PASS** plus ruff/pylint.
- [ ] **Step 5: Commit** `feat(server): preview layout helpers (extents, indent step, shifting)`

---

### Task 4: Insert, rename, dissolve section + verification guard (`preview_structure.py`)

**Files:**
- Create: `src/server/preview_structure.py`
- Test: `tests/unit/test_preview_structure.py`

**Interfaces:**
- Consumes:
  - Task 3 helpers.
  - From `preview_edits`: `_eol`, `_indent`, `_insert_before_ket`, `_doc_end`, `_to_utf16`, `_start`, `_end`. Import them; promote to public names (drop the underscore) only if pylint complains about protected access.
  - `preview_values.format_value`, `PreviewEditError`.
- Produces:
  ```python
  def insert_section_edits(structure, content, parent_id: Optional[int], title: str) -> "list[TextEdit]"
  def rename_section_edits(structure, content, section_id: int, title: str) -> "list[TextEdit]"
  def dissolve_section_edits(structure, content, section_id: int) -> "list[TextEdit]"
  def apply_text_edits(content: str, edits: "list[TextEdit]") -> str   # code-point positions (before _to_utf16)
  def verify_structure_edit(old: str, new: str) -> None                 # raises PreviewEditError
  ```
  - **Titles:** formatted with `format_value("string", title, None, None)`. A title containing a newline, or empty after strip, raises `PreviewEditError`.
  - **Insert:** appends `section "<t>" {` EOL `<indent>}` at the end of the parent:
    - in a section: `_insert_before_ket`, with indent = section indent + step;
    - at file level: at `_doc_end`, preceded by a blank line, at column 0.
    - An unclosed parent (`close_tok is None`) is rejected.
  - **Rename:** replaces the STRING token after `keyword_tok`.
  - **Dissolve:** ONE TextEdit replacing `section_extent` with the rebuilt text:
    - the header (`section` … `{`) and the closing `}` are removed;
    - their trailing comments are kept as standalone lines at the parent indent;
    - comment lines directly above the section stay in place;
    - the body goes through `shift_block(levels=-1)`.
  - Every function calls `verify_structure_edit(content, apply_text_edits(content, edits))` before returning, and returns `_to_utf16(content, edits)`.
  - **`verify_structure_edit`** lexes both texts with a silenced `Message_Handler(out=io.StringIO())` and raises unless:
    - there is no lex error;
    - `C_BRA`/`C_KET` are balanced;
    - the multiset of `(record_name, field_name, normalised value text)` is equal between old and new. Value text is the token values joined by a single space; STRING tokens use their `.value`.

    A rename or insert legitimately changes only section titles or adds a section, so section header tokens are excluded from the comparison.

- [ ] **Step 1: Write failing tests.** Each applies the edits (UTF-16-aware helper as in `test_preview_edits.py`), re-parses with zero errors, and checks values and section paths (`obj.section`):
  - `test_insert_empty_section_at_file_level`
  - `test_insert_section_into_section`
  - `test_insert_into_unclosed_section_rejected`
  - `test_rename_section_with_quote_in_title`
  - `test_dissolve_moves_children_up_one_level` (records and a nested section; the nested section's depth decreases by 1; indentation is one step less)
  - `test_dissolve_keeps_comments`
  - `test_dissolve_keeps_underindented_multiline_string`
  - `test_dissolve_crlf`
  - `test_verify_rejects_dropped_record`
  - `test_verify_rejects_unbalanced_braces`
- [ ] **Step 2: Run, expect FAIL.** `PYTHONPATH=src .venv/Scripts/python -m pytest tests/unit/test_preview_structure.py -q`
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run, expect PASS,** then the full Python suite plus ruff/pylint.
- [ ] **Step 5: Commit** `feat(server): insert, rename and dissolve sections`

---

### Task 5: Move records and sections

**Files:**
- Modify: `src/server/preview_structure.py`
- Test: `tests/unit/test_preview_structure.py`

**Interfaces:**
- Consumes: Tasks 3–4.
- Produces:
  ```python
  @dataclass(frozen=True)
  class ItemRef:
      kind: str                 # "record" | "section"
      name: Optional[str] = None        # record
      section_id: Optional[int] = None  # section

  def move_edits(structure, content, item: ItemRef,
                 target_section_id: Optional[int], before: Optional[ItemRef]) -> "list[TextEdit]"
  ```
  Semantics:
  - Unknown item or target raises; so does an unclosed target or moved section.
  - `before` must be a direct child of the target (or a file-level item when the target is None).
  - A section cannot be moved into itself or a descendant (raises).
  - Moving to the item's current position returns `[]`.
  - The result is ONE WorkspaceEdit with two non-overlapping TextEdits:
    - **Cut:** delete the item's extent. A `whole_lines` extent deletes whole lines. A token-range cut also removes one adjacent space, and the line if it becomes blank.
    - **Paste:** insert the item's text shifted by `target_depth - item_depth` levels, on fresh lines at the target indent.
      - Before `before`: at the start of `before`'s extent line.
      - At the end of a section: via the `_insert_before_ket` rule.
      - At file end: after a blank line.
  - Calls `verify_structure_edit`.

- [ ] **Step 1: Write failing tests:**
  - `test_move_record_into_section`
  - `test_move_record_reorder_within_section`
  - `test_move_record_to_file_level`
  - `test_move_record_carries_leading_comment`
  - `test_move_record_sharing_a_line`
  - `test_move_section_with_content` (nested records keep their values; depths updated)
  - `test_move_section_into_descendant_rejected`
  - `test_move_to_same_position_is_noop`
  - `test_move_before_item_not_in_target_rejected`
  - `test_move_record_with_multiline_string_shifts_it`
  - `test_move_crlf`
- [ ] **Step 2: Run, expect FAIL.**
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run, expect PASS,** then the full suite plus lint.
- [ ] **Step 5: Commit** `feat(server): move records and sections`

---

### Task 6: LSP requests for structure edits

**Files:**
- Modify: `src/server/handlers/preview.py`
- Test: `tests/unit/handlers/test_preview.py`, `tests/integration/test_preview.py`

**Interfaces:**
- Consumes: Tasks 4–5.
- Produces four requests, all using the existing `_resolve` / lock / stale / `_edit_result` path and returning `{"edit"}` or `{"error", "code"?}`:

  | Method | Params |
  |--------|--------|
  | `trlc/previewInsertSection` | `{uri, sectionId: int\|null (parent), title}` |
  | `trlc/previewRenameSection` | `{uri, sectionId, title}` |
  | `trlc/previewDissolveSection` | `{uri, sectionId}` |
  | `trlc/previewMove` | `{uri, item: ItemRefJson, targetSectionId: int\|null, before: ItemRefJson\|null}`, with `ItemRefJson = {kind: "record", name} \| {kind: "section", sectionId}` |

  Params arrive as pygls `Object`s. Read them with `getattr` like the existing handlers, and convert `item`/`before` to `ItemRef`. Malformed input returns `{"error": "Invalid request."}`, not an exception.

- [ ] **Step 1: Write failing tests.**
  - Unit (with `fake_ls`), for each method: a valid request → `edit.documentChanges[0].edits` non-empty.
  - Unit, for one representative method: locked file → `_NOT_PARSED`; stale → code `stale`; malformed `item` → `"Invalid request."`.
  - Integration: `trlc/previewInsertSection` on `sample.trlc` returns an edit whose `newText` contains `section "New"`.
- [ ] **Step 2: Run, expect FAIL.**
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run, expect PASS,** then the full suite plus lint.
- [ ] **Step 5: Commit** `feat(server): trlc/preview structure requests`

---

### Task 7: Client protocol + renderer (TOC, indentation, mandatory, section controls, drop zones)

**Files:**
- Modify: `src/client/src/previewProtocol.ts`, `src/client/src/previewRender.ts`
- Test: `src/client/src/__tests__/previewRender.test.ts`

**Interfaces:**
- Consumes: the model's `markdown` field (Task 1) and the methods from Task 6.
- Produces:
  - **`previewProtocol.ts`:**
    - constants `PREVIEW_INSERT_SECTION_METHOD`, `PREVIEW_RENAME_SECTION_METHOD`, `PREVIEW_DISSOLVE_SECTION_METHOD`, `PREVIEW_MOVE_METHOD`;
    - `PreviewField.markdown: boolean`;
    - `type ItemRef = { kind: "record"; name: string } | { kind: "section"; sectionId: number }`;
    - new `WebviewToHost` members:
      ```ts
      | { type: "insertSection"; sectionId: number | null; title: string }
      | { type: "renameSection"; sectionId: number; title: string }
      | { type: "dissolveSection"; sectionId: number }
      | { type: "move"; item: ItemRef; targetSectionId: number | null; before: ItemRef | null }
      ```
  - **`previewRender.ts` markup contract** (consumed by Tasks 9–10):

    | Element | Markup |
    |---------|--------|
    | TOC (first child of the body; omitted when the model has no sections) | `<nav class="toc"><button class="toc-toggle">▾</button><ul>` nested `<li><a class="toc-link" data-section-id="N">title</a><ul>…</ul></li>` |
    | Section | `<div class="section-block" id="section-N" data-section-id="N">`; heading as v1 plus `<span class="drag-handle" draggable="true" data-drag-kind="section" data-drag-id="N">⠿</span>`, `<button class="rename-section" data-section-id="N">✎</button>`, `<button class="dissolve-section" data-section-id="N" title="Dissolve section">🗑</button>`; children inside `<div class="section-body">` (CSS: `margin-left` + `border-left`) |
    | Record handle | `<span class="drag-handle" draggable="true" data-drag-kind="record" data-drag-id="<name>">⠿</span>` inside `.record` |
    | Drop zones (before every child, and one at the end of every closed section body and of the file level) | `<div class="drop-zone" data-target-section-id="N or ''" data-before-kind="record\|section\|''" data-before-id="…">` |
    | Required field label | `<span class="required">*</span>` when `!optional` |
    | Missing-field hint (`value === null && !optional`) | `<div class="missing">Required field "<name>" is missing</div>` per field at the top of the record |
    | Add-section button | `<button class="add-section" data-section-id="N or ''">+ Section</button>` next to every add-record button |
    | Markdown field | `<div class="md-field" data-field data-kind="string" data-optional data-markdown="1" data-value="<escaped raw>"></div>` instead of a textarea |

    When locked: no `draggable` attribute, and all buttons disabled. Everything escaped.

- [ ] **Step 1: Write failing tests:**
  - `"renders a TOC with nested links"`
  - `"omits the TOC without sections"`
  - `"marks required fields with *"`
  - `"shows a hint per missing required field"`
  - `"emits drop zones before children and at section end"`
  - `"emits drag handles only when editable"`
  - `"renders markdown fields as md-field with escaped data-value"`
  - `"emits add-section, rename and dissolve buttons"`
- [ ] **Step 2: Run, expect FAIL.** `npx mocha`
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run, expect PASS** plus tsc and eslint.
- [ ] **Step 5: Commit** `feat(client): preview TOC, nesting, required markers, structure controls`

---

### Task 8: Controller dispatch for structure intents

**Files:**
- Modify: `src/client/src/previewController.ts`
- Test: `src/client/src/__tests__/previewController.test.ts`

**Interfaces:**
- Consumes: Task 7 message types and constants.
- Produces: `dispatch` maps each new message to its method through the existing `edit(method, params)` queue, with the same stale and versioning behaviour.

  | Message | Method | Params |
  |---------|--------|--------|
  | `insertSection` | `PREVIEW_INSERT_SECTION_METHOD` | `{uri, sectionId, title}` |
  | `renameSection` | `PREVIEW_RENAME_SECTION_METHOD` | `{uri, sectionId, title}` |
  | `dissolveSection` | `PREVIEW_DISSOLVE_SECTION_METHOD` | `{uri, sectionId}` |
  | `move` | `PREVIEW_MOVE_METHOD` | `{uri, item, targetSectionId, before}` |

- [ ] **Step 1: Write failing tests:**
  - one per message, asserting method and params;
  - `"move is queued behind a pending setField"`.
- [ ] **Step 2: Run, expect FAIL.**
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run, expect PASS** plus tsc and eslint.
- [ ] **Step 5: Commit** `feat(client): controller handles structure intents`

---

### Task 9: Webview interactions (drag & drop, TOC, section forms)

**Files:**
- Create: `src/client/src/previewDnd.ts` (pure)
- Modify: `src/client/src/previewWebview.ts`, `src/client/src/previewFocus.ts`
- Test: `src/client/src/__tests__/previewDnd.test.ts`, `src/client/src/__tests__/previewFocus.test.ts`

**Interfaces:**
- Consumes: Task 7 markup, Task 8 messages.
- Produces:
  ```ts
  // previewDnd.ts
  export function sectionAncestors(nodes: PreviewNode[], sectionId: number): number[];   // nearest first
  export function canDrop(nodes: PreviewNode[], item: ItemRef, targetSectionId: number | null): boolean; // false for a section into itself/descendant
  export function isNoOpMove(nodes: PreviewNode[], item: ItemRef, targetSectionId: number | null, before: ItemRef | null): boolean;
  // previewFocus.ts
  RenderState gains `dragging: boolean`, and shouldRenderNow returns false while dragging.
  ```

  **Webview behaviour:**
  - **Drag & drop:**
    - `dragstart` on a handle sets `dragging`; the item is serialised in `dataTransfer`.
    - Drop zones become visible and highlight on `dragover` when `canDrop`.
    - `drop` posts `move` unless `isNoOpMove`.
    - `dragend` clears `dragging` and flushes a pending model.
  - **TOC:** a click on `.toc-link` calls `scrollIntoView` on `#section-N`; `.toc-toggle` collapses or expands (a module variable, kept across re-renders).
  - **Section forms:** `+ Section` and `✎` open an inline title form (render-blocking like the add-record form). It posts `insertSection` / `renameSection`.
  - **Dissolve:** `🗑` posts `dissolveSection` with no confirmation.

- [ ] **Step 1: Write failing tests:**
  - `canDrop`: record anywhere → true; section into itself → false; section into its grandchild → false; section into a sibling → true.
  - `isNoOpMove`: before its own next sibling → true; to another section → false.
  - `sectionAncestors`.
  - `"does not render while dragging"` (`shouldRenderNow`).
- [ ] **Step 2: Run, expect FAIL.**
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run, expect PASS** plus tsc, eslint and the esbuild bundle check.
- [ ] **Step 5: Commit** `feat(client): drag & drop, TOC navigation, section forms`

---

### Task 10: Markdown editor in the webview

**Files:**
- Modify: `package.json` (dependency), `pnpm-lock.yaml`
- Create: `src/client/src/previewMarkdown.ts`
- Modify: `src/client/src/previewWebview.ts`
- Test: `src/client/src/__tests__/previewMarkdown.test.ts` (pure helpers only)

**Interfaces:**
- Consumes: Task 7 `.md-field` markup; the existing `setField` message and `textCommitValue`.
- Produces:
  ```ts
  export function normalizeMarkdown(md: string): string;  // CRLF→LF, trim trailing spaces per line, trim leading/trailing blank lines (mirrors server format_markdown)
  export function markdownChanged(before: string, after: string): boolean; // compares normalised forms
  export function mountMarkdownField(el: HTMLElement, opts: { editable: boolean; onCommit(md: string): void }): { destroy(): void };
  ```

  **Library:** `@milkdown/kit` (exact version pinned) with the `commonmark`, `gfm` and `listener` plugins. The planner chose it because it is markdown-first (remark/GFM serialisation including tables), headless (no CDN, no remote CSS) and ProseMirror-based. The implementer must verify:
  - the bundle builds as `iife`/browser;
  - it contains no `eval(`/`new Function(`;
  - the minified `preview.js` is < 1 MB.

  If Milkdown fails any of these, use TipTap (`@tiptap/core`, starter-kit, table extensions + `tiptap-markdown`) and record the reason in the commit message.

  **Behaviour:**
  - The field renders read-only (rendered Markdown, no HTML passthrough; links not followed in the webview).
  - A click makes it editable and shows the toolbar: Bold, Italic, Strikethrough, Heading, Bullet list, Numbered list, Link, Table (insert; add/remove row/column), Code. Use the library's commands.
  - `focusout` leaving the field → `onCommit(markdown)`. The webview posts `setField` only if `markdownChanged`.
  - An empty value on an optional field posts null (reuse `textCommitValue`).
  - The editor counts as "input has focus" for the render gating.

  Update the lockfile with `npx pnpm@11.13.1 install --lockfile-only`. That matches `packageManager`; if the version differs, use the one in `package.json`.

- [ ] **Step 1: Write failing tests:** `normalizeMarkdown` (CRLF, trailing spaces, blank edges); `markdownChanged` (whitespace-only change → false).
- [ ] **Step 2: Run, expect FAIL.**
- [ ] **Step 3: Implement,** adding the dependency and lockfile.
- [ ] **Step 4: Verify:**
  - `npx mocha`, tsc, eslint;
  - `npx esbuild src/client/src/previewWebview.ts --bundle --format=iife --platform=browser --minify --outfile=<scratch>/preview.js`;
  - the size is < 1 MB;
  - `grep -c "new Function\|eval(" <scratch>/preview.js` → 0.
- [ ] **Step 5: Commit** `feat(client): WYSIWYG markdown fields in the preview`

---

### Task 11: Docs, changelog, e2e

**Files:**
- Modify:
  - `docs/USER_GUIDE.md` (Preview section: TOC, required markers, sections, drag & drop, Markdown fields and the new setting in the settings table)
  - `docs/ARCHITECTURE.md` (new requests; `preview_layout`/`preview_structure`; verification guard)
  - `CHANGELOG.md` (Unreleased)
  - `tests/e2e/suite/smoke.e2e.ts`

- [ ] **Step 1:** Extend the e2e. The e2e cannot reach into the webview, so the structure features stay covered by the unit and integration tests. Add a test asserting that `vscode.workspace.getConfiguration("trlcServer").get("preview.markdownComponents")` deep-equals `["description"]`. That proves the setting is contributed. The existing preview-tab test must keep passing.
- [ ] **Step 2:** `npx tsc -p tests/e2e/tsconfig.json --noEmit` passes.
- [ ] **Step 3:** Write the docs. Verify every claim against the code.
- [ ] **Step 4:** Commit `docs: preview v2`.
