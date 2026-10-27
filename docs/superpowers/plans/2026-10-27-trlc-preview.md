# TRLC Preview Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a side-by-side preview of the active `.trlc` file that shows sections as Markdown-style headings, lets you edit record attributes, and lets you add new records.

**Architecture:** The Python server owns all TRLC knowledge. It gets three new custom requests: `trlc/previewModel` returns a JSON tree of sections, records, and fields built from the last parse's token stream. `trlc/previewSetField` and `trlc/previewAddRecord` return LSP `WorkspaceEdit`s. The TypeScript client applies those edits with `workspace.applyEdit`, so undo, dirty state, and the reparse→diagnostics loop work exactly as for keyboard edits. It renders the model in a VS Code webview whose script is bundled by a second esbuild target.

**Tech Stack:**
- Server: Python 3.10–3.13, pygls 2.1.1, lsprotocol, trlc ≥ 3.0.0, pytest / pytest-lsp.
- Client: TypeScript, vscode-languageclient, VS Code Webview API, mocha + sinon.
- Build: Bazel (`aspect_rules_esbuild`, `aspect_rules_ts`).

**Spec:** None. The user asked for this directly ("Preview zum Editieren von TRLC-Attributen, neue Records hinzufügen, Sections als Markdown-Überschriften, je tiefer desto mehr '#'") and skipped the design Q&A. The design calls made on their behalf are listed below; review them first.

## Design decisions (made by the planner, not the user)

1. The preview shows **one file**: the active `.trlc` editor. It follows the active TRLC editor, like VS Code's Markdown preview. `.rsl` files are not previewed.
2. A section at nesting depth *d* (top-level section = 1) renders as `<h{min(d,6)}>`. A visible, muted prefix of *d* `#` characters is shown so depth stays readable beyond 6.
3. Each record renders as a card with the header `Type name`, followed by one row per component of its type. The rows follow declaration order and include inherited components and unset optional components.
4. Editors by field kind:

   | Kind | Editor |
   |------|--------|
   | string | Textarea; multi-line allowed |
   | integer, decimal | Text input |
   | boolean | Select (`true` / `false`) |
   | enum | Select of the literals |
   | reference, other (arrays, tuples) | Raw TRLC source text |

   Frozen components are read-only. A set optional component gets a remove button (×).
5. A "+ Record" button sits at the end of every section and at file level. It opens an inline form with a type select and a name input. The new record gets placeholder values for required components:

   | Component type | Placeholder |
   |----------------|-------------|
   | Integer | `0` |
   | Decimal | `0.0` |
   | Boolean | `false` |
   | String | `""` |
   | Enum | first literal |
   | reference, other | omitted, so TRLC reports it as missing until filled in |
6. Edits are never applied when the server's last parse is stale relative to the live buffer, or when the file failed to parse. The preview then keeps the last good tree, shows a banner with the reason, and disables all inputs.
7. UI text is English, matching the rest of the extension.

## Global Constraints

- Every new source file starts with the repo's BMW GPL header (copy it from a neighbouring file of the same language; `bazel test //:copyright` enforces it).
- Python: ruff line length 79, `known-first-party = ["server"]`, and pylint clean (`bazel build --config=lint //...`).
- Client TS files live flat in `src/client/src/` (the `ts_project` glob is `src/client/src/*.ts`, non-recursive). Tests go in `src/client/src/__tests__/*.test.ts`.
- Handlers read the parse result through exactly one `context.snapshot()` per request (see `docs/ARCHITECTURE.md` § Concurrency) and never call `ParseEngine.validate()`.
- Inserted text uses the file's own line ending: `"\r\n"` if `lexer.content` contains `"\r\n"`, else `"\n"`.
- Positions are LSP 0-based line/character, derived via `token_utils.get_location()`, consistent with the existing handlers.

## Review Focus

1. **Typing in the editor, then editing in the preview before the reparse finishes.** The server must refuse with an error instead of applying offsets from a stale AST. Test: Task 5, `test_set_field_rejected_when_stale`.
2. **String values containing `"`, ending in `\`, or spanning lines.** They must round-trip through TRLC's lexer to the exact same value. Test: Task 2, `test_format_string_round_trips`.
3. **Comments between fields or after a value.** Replacing a value must not delete the comment. Test: Task 1, `test_value_span_excludes_trailing_comment`.
4. **Two sections with the same title (nested or sibling).** "+ Record" must insert into the section the user clicked. Test: Task 1, `test_duplicate_section_titles_get_distinct_ids`; Task 4, `test_add_record_into_second_duplicate_section`.
5. **A file with a syntax error,** which TRLC drops from `vsm.all_files`. The model must say "not editable" with a reason instead of crashing or showing an empty tree. Test: Task 5, `test_model_unavailable_for_unparsed_file`.

---

### Task 1: Token-stream scanner (`preview_scan.py`)

Locates every section, record, and explicit field assignment in a parsed file's token list, which later tasks use to compute edit ranges. Assignments are recognised through `ast_link`:

| Token(s) | `ast_link` |
|----------|------------|
| `section` keyword and its closing `}` | the `trlc.ast.Section` |
| record name `IDENTIFIER`, its `{` and `}` | the `Record_Object` |
| a component name `IDENTIFIER` and its following `ASSIGN` | the `Composite_Component` |

See `trlc/parser.py` `parse_section_declaration` / `parse_record_object_declaration`.

**Files:**
- Create: `src/server/preview_scan.py`
- Test: `tests/unit/test_preview_scan.py`

**Interfaces:**
- Produces:
  ```python
  @dataclass(frozen=True)
  class FieldSpan:
      name: str
      name_tok: int       # index of the component-name IDENTIFIER
      value_first: int    # first token after ASSIGN
      value_last: int     # last non-COMMENT token before the next field / record '}'

  @dataclass(frozen=True)
  class RecordSpan:
      obj: "trlc.ast.Record_Object"
      name_tok: int
      close_tok: int                 # the record's C_KET
      fields: "dict[str, FieldSpan]" # explicit assignments only, source order
      section_id: Optional[int]      # innermost enclosing section, None at file level

  @dataclass(frozen=True)
  class SectionSpan:
      section_id: int        # 0-based ordinal of the `section` keyword in the file
      title: str
      depth: int             # 1 = top-level
      parent_id: Optional[int]
      keyword_tok: int
      close_tok: int

  @dataclass(frozen=True)
  class FileStructure:
      tokens: list
      sections: "list[SectionSpan]"   # source order
      records: "list[RecordSpan]"     # source order
      def record(self, name: str) -> Optional[RecordSpan]: ...
      def section(self, section_id: int) -> Optional[SectionSpan]: ...

  def scan_file(tokens: list) -> FileStructure: ...
  ```

- [ ] **Step 1: Write failing tests.** Build tokens with a real parse: mirror the `fake_ls` helper in `tests/unit/handlers/conftest.py`, or use `trlc.trlc.Source_Manager` directly with `register_file(name, content)` + `process()`, then `sm.all_files[name].lexer.tokens`. Fixture TRLC (RSL = `RSL_CONTENT` from the handler conftest, plus an optional `String` component `text` on `MyRecord`):
  ```
  package Types
  section "A" {
      MyRecord r1 {
          x = 1  // keep me
          y = 2
      }
      section "A" {
          MyRecord r2 { x = 3 y = 4 }
      }
  }
  MyRecord r3 { x = 5 y = 6 }
  ```
  Tests:
  - `test_sections_have_depth_and_parent`: two sections, ids `0`,`1`, depths `1`,`2`, `sections[1].parent_id == 0`.
  - `test_duplicate_section_titles_get_distinct_ids`: both titled `"A"`, ids differ, `record("r2").section_id == 1`, `record("r1").section_id == 0`, `record("r3").section_id is None`.
  - `test_value_span_excludes_trailing_comment`: `tokens[r1.fields["x"].value_last].value == 1` and kind `INTEGER`.
  - `test_inline_record_fields`: `record("r2").fields` keys `["x","y"]`, each span one token.
  - `test_unknown_record_returns_none`: `record("nope") is None`.

- [ ] **Step 2: Run, expect FAIL.** `bazel test //tests:unit_test_preview_scan --test_output=errors`. It fails with an import error.

- [ ] **Step 3: Implement `scan_file`.** Make a single pass over `tokens` with a stack of open `SectionSpan` builders:
  - A `KEYWORD "section"` token pushes a section.
  - A `C_KET` linked to a `Section` pops one.
  - An `IDENTIFIER` linked to a `Record_Object` and immediately followed by `C_BRA` linked to the same object opens a record, which runs until the `C_KET` linked to it.
  - Inside a record, an `IDENTIFIER` linked to a `Composite_Component` followed by `ASSIGN` starts a field. Its value ends at the last non-`COMMENT` token before the next field start or the record's `C_KET`.

- [ ] **Step 4: Run, expect PASS.** Same command.

- [ ] **Step 5: Commit.** `git add src/server/preview_scan.py tests/unit/test_preview_scan.py && git commit -m "feat(server): scan TRLC token stream for preview"`

---

### Task 2: Value kinds and literal formatting (`preview_values.py`)

**Files:**
- Create: `src/server/preview_values.py`
- Test: `tests/unit/test_preview_values.py`

**Interfaces:**
- Produces:
  ```python
  class PreviewEditError(Exception): """User-facing message in str(e)."""

  def field_kind(n_typ) -> str
      # "string" | "integer" | "decimal" | "boolean" | "enum" | "reference" | "other"
  def type_ref(entity, cur_package) -> str
      # entity.name, or "Pkg.Name" when entity.n_package is not cur_package
  def format_value(kind: str, raw: str, n_typ, cur_package) -> str   # TRLC source text, raises PreviewEditError
  def placeholder(kind: str, n_typ, cur_package) -> Optional[str]    # None = omit
  def visible_record_types(cu) -> "dict[str, trlc.ast.Record_Type]"
      # non-abstract Record_Types of cu.package and cu.imports, keyed by type_ref(...)
  ```

- [ ] **Step 1: Write failing tests.**
  - `test_field_kind_maps_builtins`:
    - `Builtin_String` and `Builtin_Markup_String` → `"string"`
    - `Builtin_Integer` → `"integer"`
    - `Builtin_Decimal` → `"decimal"`
    - `Builtin_Boolean` → `"boolean"`
    - `Enumeration_Type` → `"enum"`
    - `Record_Type` → `"reference"`
    - `Array_Type` / `Tuple_Type` → `"other"`
  - `test_format_string_round_trips`: for each of `'plain'`, `'say "hi"'`, `'ends with \\'`, `'line1\nline2'`, `'a\\"b'`, take `format_value("string", v, ...)`, lex it with `trlc.lexer.TRLC_Lexer(mh, "t", text)`, and assert the single `STRING` token's `.value == v`.
  - `test_format_string_rejects_unrepresentable`: a value containing both `'''` and `"""` and a newline raises `PreviewEditError`.
  - `test_format_integer`: `" 42 "` → `"42"`; `"-1_000"` → `"-1_000"`; `"4.2"` raises.
  - `test_format_decimal`: `"1.5"` → `"1.5"`; `"x"` raises.
  - `test_format_boolean`: only `"true"` / `"false"`.
  - `test_format_enum_qualifies`: literal `"Red"` of `Types.Color` from package `Types` → `"Color.Red"`; from another package → `"Types.Color.Red"`; unknown literal raises.
  - `test_format_raw_kinds`: `"reference"` / `"other"` return `raw.strip()`; empty raises.
  - `test_placeholders`: integer `"0"`, decimal `"0.0"`, boolean `"false"`, string `'""'`, enum first literal qualified as above, reference/other `None`.
  - `test_visible_record_types_skips_abstract_and_qualifies_imports`: local `MyRecord` → key `"MyRecord"`; imported `Other.T` → key `"Other.T"`; an `abstract type` is absent.

- [ ] **Step 2: Run, expect FAIL.** `bazel test //tests:unit_test_preview_values --test_output=errors`

- [ ] **Step 3: Implement.** String rule:
  - No `"\n"` and not ending in `\` → `'"' + raw.replace('"', '\\"') + '"'`.
  - Otherwise, if `'''` is not in `raw` → `"'''" + raw + "'''"`.
  - Otherwise, if `"""` is not in `raw` → `'"""' + raw + '"""'`.
  - Otherwise raise.

  Validation regexes:

  | Kind | Regex |
  |------|-------|
  | integer | `^[+-]?\d[\d_]*$` |
  | decimal | `^[+-]?\d[\d_]*\.\d[\d_]*$` |

  Enum literals come from `n_typ.literals.values()`.

- [ ] **Step 4: Run, expect PASS.**

- [ ] **Step 5: Commit.** `git commit -m "feat(server): preview value kinds and TRLC literal formatting"`

---

### Task 3: Preview model (`preview_model.py`)

**Files:**
- Create: `src/server/preview_model.py`
- Test: `tests/unit/test_preview_model.py`

**Interfaces:**
- Consumes: `scan_file`, `FileStructure` (Task 1); `field_kind`, `type_ref`, `visible_record_types` (Task 2).
- Produces (JSON-ready dicts, camelCase keys):
  ```python
  def build_preview_model(uri: str, parsed) -> dict   # parsed = vsm.all_files[path] (a trlc Parser)
  def unavailable_model(uri: str, reason: str) -> dict
  ```
  Model shape (Task 6 mirrors it in TS):
  ```
  {"uri", "package": str|None, "editable": bool, "reason": str|None,
   "recordTypes": [str],                       # sorted(visible_record_types(cu))
   "nodes": [Node] | None}                     # None only in unavailable_model
  Node = {"kind":"section","sectionId":int,"title":str,"depth":int,"children":[Node]}
       | {"kind":"record","name":str,"type":str,"line":int,"fields":[Field]}   # line: 0-based, of name token
  Field = {"name","kind","value": str|None, "optional": bool, "editable": bool,
           "enumValues": [str]|None}
  ```
  `value` by case:

  | Case | `value` |
  |------|---------|
  | `Implicit_Null` | `None` |
  | string | `String_Literal.value` |
  | enum | literal name only |
  | otherwise | source text from `value_first` start to `value_last` end, via `lexer.content` slicing on `location.start_pos` / `end_pos` |

  `editable` is `False` for frozen components (`obj.n_typ.is_frozen(comp)`).

- [ ] **Step 1: Write failing tests** using the Task 1 fixture.
  - `test_tree_nests_sections_and_records`: `nodes` = `[section A(depth 1, children=[record r1, section A(depth 2, children=[record r2])]), record r3]`.
  - `test_fields_follow_declaration_order_and_include_unset_optional`: the `r1` field names equal `[c.name for c in MyRecord.all_components()]`, and the unset `text` has `value is None` and `optional is True`.
  - `test_enum_field_lists_literals`: with a `c Color` component set to `Color.Green`, the field has `kind == "enum"`, `value == "Green"`, and `enumValues == ["Red","Green","Blue"]`.
  - `test_record_types_lists_local_and_imported`: a second RSL package `Other` (with type `T`) imported by the file → `recordTypes == ["MyRecord", "Other.T"]`.
  - `test_unavailable_model`: `nodes is None`, `editable is False`, reason passed through.

- [ ] **Step 2: Run, expect FAIL.** `bazel test //tests:unit_test_preview_model --test_output=errors`

- [ ] **Step 3: Implement** by folding `FileStructure.sections` / `records` into the tree using `parent_id` / `section_id`, preserving source order among siblings.

- [ ] **Step 4: Run, expect PASS.**

- [ ] **Step 5: Commit.** `git commit -m "feat(server): build preview model from parse result"`

---

### Task 4: Edit computation (`preview_edits.py`)

**Files:**
- Create: `src/server/preview_edits.py`
- Test: `tests/unit/test_preview_edits.py`

**Interfaces:**
- Consumes: Task 1 spans; Task 2 `format_value`, `placeholder`, `type_ref`, `PreviewEditError`; `token_utils.get_location`.
- Produces:
  ```python
  def set_field_edits(structure: FileStructure, content: str, cur_package,
                      record: str, field: str, value: Optional[str]) -> "list[TextEdit]"
  def add_record_edits(structure: FileStructure, content: str, cu,
                       section_id: Optional[int], type_name: str, name: str) -> "list[TextEdit]"
  ```
  Both raise `PreviewEditError` on any rejection.

  `set_field_edits` rules:

  | Situation | Result |
  |-----------|--------|
  | Unknown record or field | Raise |
  | Frozen component | Raise |
  | `value is None` on a required component | Raise |
  | Explicit field, `value` not `None` | Replace the range from `value_first` start to `value_last` end with `format_value(...)` |
  | Explicit field, `value is None` (optional) | Delete the assignment. If only whitespace precedes the name token on its line and only whitespace/comment follows `value_last` on its line, delete those whole lines including the line break; otherwise delete just name-token start → `value_last` end |
  | Implicit field (unset) | Insert `name = <formatted>` before the record's `C_KET` (insertion rule below) |

  The field indent is the indent of the first explicit field's line, else the record line's indent plus 4 spaces.

  **Insertion rule.** This applies to both a field before a record `}` and a record before a section `}`. If only whitespace precedes the `C_KET` on its line, insert `indent + text + EOL` at column 0 of that line. Otherwise insert `EOL + indent + text + EOL + closing_indent` at the `C_KET` start.

  `add_record_edits` rules:
  - Reject when `name` does not match `^[A-Za-z][A-Za-z0-9_]*$`, when `name in cu.package.symbols`, when `type_name` is not a key of `visible_record_types(cu)` (Task 2), or when `section_id` is unknown.
  - The text is `f"{type_name} {name} {{"`, then one line per required, non-frozen component whose `placeholder` is not `None` (`comp = value`), then `}`.
  - At file level, append `EOL + text + EOL` at the end of the document (position after the last character of `content`).
  - In a section, use the insertion rule before the section's `C_KET`, with indent = the section-keyword line's indent plus 4 spaces.

- [ ] **Step 1: Write failing tests.** Apply the returned edits to `content` with a small test helper (`apply_text_edits(content, edits) -> str`, defined in the test file and working from LSP positions), then re-parse the result. Assert there are no errors and check the new field value via `record_obj.field[...]`.
  - `test_replace_integer_value_keeps_comment`: `r1.x` set to `"7"` → `x = 7  // keep me` is in the output.
  - `test_set_unset_optional_inserts_line`: `r1.text` set to `'he said "hi"'` → the reparsed value equals the input.
  - `test_insert_into_inline_record`: `r2.text` set to `"t"` → the reparse succeeds.
  - `test_remove_optional_deletes_whole_line`: set, then unset → output is identical to the original.
  - `test_remove_required_rejected`: `r1.x` set to `None` raises `PreviewEditError`.
  - `test_add_record_at_file_level`: `add_record_edits(..., None, "MyRecord", "r4")` → the reparse has `r4` with `x == 0` and `y == 0`.
  - `test_add_record_into_second_duplicate_section`: `section_id=1` → `r4.section[-1]` is the inner `"A"` (depth 2).
  - `test_add_record_rejects_bad_or_duplicate_name`: `"1abc"`, `"r1"`.
  - `test_crlf_file_keeps_crlf`: the fixture with `\r\n` → inserted text contains no lone `\n`.

- [ ] **Step 2: Run, expect FAIL.** `bazel test //tests:unit_test_preview_edits --test_output=errors`

- [ ] **Step 3: Implement.**

- [ ] **Step 4: Run, expect PASS.**

- [ ] **Step 5: Commit.** `git commit -m "feat(server): compute preview edits (set field, add record)"`

---

### Task 5: LSP handler (`handlers/preview.py`)

**Files:**
- Create: `src/server/handlers/preview.py`
- Modify: `src/server/language_server.py` (import `preview`, add it to the `handler_modules` tuple)
- Test: `tests/unit/handlers/test_preview.py`, `tests/integration/test_preview.py`

**Interfaces:**
- Consumes: Tasks 3 and 4; `token_utils.path_from_uri`, `token_utils._ast_tokens_are_stale`, `parse_guard.normalize_fs_path`.
- Produces three requests, with params as dataclasses in the style of `status.ScopeStatusParams`:

  | Method | Params | Result |
  |--------|--------|--------|
  | `trlc/previewModel` (`PREVIEW_MODEL_METHOD`) | `{uri}` | model dict (Task 3) |
  | `trlc/previewSetField` (`PREVIEW_SET_FIELD_METHOD`) | `{uri, record, field, value: str \| null}` | `{"edit": <WorkspaceEdit unstructured>} \| {"error": str}` |
  | `trlc/previewAddRecord` (`PREVIEW_ADD_RECORD_METHOD`) | `{uri, sectionId: int \| null, type, name}` | same as `previewSetField` |

  Shared resolution `_resolve(ls, uri) -> tuple[parsed, None] | tuple[None, str]` takes one `context.snapshot()` and returns:

  | Condition | Reason |
  |-----------|--------|
  | No context | `"File is not part of an active TRLC scope."` |
  | File missing from `vsm.all_files` | `"File has errors that prevent parsing; fix them in the editor."` |
  | `_ast_tokens_are_stale` | `"Waiting for the parser to catch up with your last edit."` |

  `previewModel` returns `unavailable_model(uri, reason)` in those cases. Both edit methods return `{"error": reason}`.

  The `WorkspaceEdit` is `WorkspaceEdit(changes={uri: edits})`, serialized with `lsprotocol.converters.get_converter().unstructure(...)`.

- [ ] **Step 1: Write failing unit tests** with `fake_ls` (`tests/unit/handlers/conftest.py`). Register the module via `preview.register(fake_ls)` and fetch handlers with `get_handler(...)`.
  - `test_model_for_open_file`: `nodes[0]["name"] == "example"`.
  - `test_model_unavailable_for_unparsed_file`: an opened uri whose content has a syntax error → `editable is False` and the reason contains `"errors"`.
  - `test_set_field_returns_workspace_edit`: `result["edit"]["changes"][uri][0]["newText"] == "9"`.
  - `test_set_field_rejected_when_stale`: change `_uri_contents[uri]` after the parse → result has `"error"` and no `"edit"`.
  - `test_add_record_returns_edit_or_error`: valid → `"edit"`; duplicate name → `"error"`.

- [ ] **Step 2: Write a failing integration test** `tests/integration/test_preview.py`, following `test_smoke.py` / `lsp_helpers.py`. Open the `sample.trlc` fixture, wait for diagnostics, then send `trlc/previewModel` and assert the record `example` with field `x` value `"1"`. Send `trlc/previewSetField` (`x` → `"5"`) and assert the edit's `newText == "5"`.

- [ ] **Step 3: Run, expect FAIL.** `bazel test //tests:unit_handlers_test_preview //tests:integration_test_preview --test_output=errors`

- [ ] **Step 4: Implement and register.**

- [ ] **Step 5: Run, expect PASS,** then the whole Python suite: `bazel test //tests:all_tests` and `bazel build --config=lint //...`.

- [ ] **Step 6: Commit.** `git commit -m "feat(server): trlc/preview* custom requests"`

---

### Task 6: Client protocol types and pure renderer

**Files:**
- Create: `src/client/src/previewProtocol.ts`, `src/client/src/previewRender.ts`
- Modify: `tsconfig.json` (add `"DOM"` to `lib`; the webview script in Task 8 needs it)
- Test: `src/client/src/__tests__/previewRender.test.ts`

**Interfaces:**
- Produces in `previewProtocol.ts`:
  - Method constants `PREVIEW_MODEL_METHOD`, `PREVIEW_SET_FIELD_METHOD`, `PREVIEW_ADD_RECORD_METHOD`.
  - `PreviewModel`, `PreviewNode`, `PreviewField`, mirroring Task 3 exactly.
  - `EditResult = { edit: unknown } | { error: string }`.
  - Message unions:
    ```ts
    type HostToWebview = { type: "model"; model: PreviewModel } | { type: "error"; message: string };
    type WebviewToHost =
      | { type: "ready" }
      | { type: "setField"; record: string; field: string; value: string | null }
      | { type: "addRecord"; sectionId: number | null; recordType: string; name: string }
      | { type: "reveal"; line: number };
    ```
- Produces in `previewRender.ts`:
  ```ts
  export function escapeHtml(s: string): string;
  export function renderPreview(model: PreviewModel, lastNodes: PreviewNode[] | null): string; // body HTML
  export function previewShellHtml(nonce: string, cspSource: string, scriptUri: string): string;
  ```
  `renderPreview` uses `model.nodes ?? lastNodes`. When `!model.editable` it prepends `<div class="banner">${reason}</div>` and gives every input/button the `disabled` attribute.

  Markup contract (the webview script in Task 8 relies on these attributes):

  | Element | Markup |
  |---------|--------|
  | Section heading | `<h{min(depth,6)} class="section"><span class="hashes">{"#".repeat(depth)}</span> {title}</h…>` |
  | Record card | `<div class="record" data-record="{name}">` with `<a class="record-name" data-line="{line}">` |
  | Field input | `data-field="{name}"` and `data-kind="{kind}"` |
  | Remove button | `button.remove` |
  | Add-record button | `<button class="add-record" data-section-id="{id or ''}">` |

  `previewShellHtml` produces this CSP: `default-src 'none'; style-src ${cspSource} 'unsafe-inline'; script-src 'nonce-${nonce}'`. It contains `<div id="root"></div>` and the script tag. Styles use VS Code theme variables (`--vscode-editor-foreground` etc.).

- [ ] **Step 1: Write failing tests.**
  - `"renders nested sections with one # per depth"`: depth 3 → contains `<h3 class="section"><span class="hashes">###</span> Inner`.
  - `"caps heading level at h6 but keeps all hashes"`: depth 8 → `<h6` and `########`.
  - `"escapes titles and values"`: title `<b>&` → `&lt;b&gt;&amp;`.
  - `"enum field renders a select with the current literal selected"`.
  - `"unset optional field shows empty input and no remove button"`; `"set optional field has a remove button"`.
  - `"non-editable model shows banner, disables inputs, and falls back to lastNodes"`.
  - `"add-record button per section and at file level"`: `data-section-id="0"` and `data-section-id=""`.

- [ ] **Step 2: Run, expect FAIL.** `bazel test //:client_test --test_output=errors` (or `pnpm test`).

- [ ] **Step 3: Implement.**

- [ ] **Step 4: Run, expect PASS,** plus `bazel build //:typecheck`.

- [ ] **Step 5: Commit.** `git commit -m "feat(client): preview protocol types and HTML renderer"`

---

### Task 7: Preview controller (host-side logic, no VS Code UI)

**Files:**
- Create: `src/client/src/previewController.ts`
- Test: `src/client/src/__tests__/previewController.test.ts`

**Interfaces:**
- Consumes: Task 6 types.
- Produces:
  ```ts
  export interface PreviewDeps {
      sendRequest<T>(method: string, params: object): Promise<T>;
      applyEdit(lspWorkspaceEdit: unknown): Promise<boolean>;
      showError(message: string): void;
      reveal(uri: string, line: number): void;
  }
  export class PreviewController {
      constructor(deps: PreviewDeps, post: (msg: HostToWebview) => void);
      readonly uri: string | undefined;
      setDocument(uri: string | undefined): Promise<void>; // no-op if unchanged; refreshes otherwise
      refresh(): Promise<void>;                             // requests model, posts {type:"model"}
      handleMessage(msg: WebviewToHost): Promise<void>;
  }
  ```
  Behaviour:
  - `ready` → `refresh()`.
  - `setField` / `addRecord` → send the request with `uri` (mapping `recordType` → `type`).
    - `{error}` → `showError("TRLC: " + error)`, then `refresh()` (so the view reverts the rejected input).
    - `{edit}` → `applyEdit(edit)`. A `false` result → `showError("TRLC: Could not apply edit.")`.
  - `reveal` → `deps.reveal(uri, line)`.
  - A rejected request promise → `post({type:"error", message})`.
  - With `uri === undefined` every handler is a no-op.

- [ ] **Step 1: Write failing tests** with sinon stubs for `PreviewDeps`:
  - `refresh posts model`
  - `setField applies returned edit`
  - `setField error shows message and refreshes`
  - `addRecord maps recordType to type param`
  - `applyEdit false shows error`
  - `setDocument same uri does not re-request`
  - `no document → handleMessage is a no-op`

- [ ] **Step 2: Run, expect FAIL.** `bazel test //:client_test --test_output=errors`

- [ ] **Step 3: Implement.**

- [ ] **Step 4: Run, expect PASS.**

- [ ] **Step 5: Commit.** `git commit -m "feat(client): preview controller"`

---

### Task 8: Webview script, panel, command, build wiring

**Files:**
- Create: `src/client/src/previewWebview.ts` (browser entry), `src/client/src/previewPanel.ts` (VS Code glue)
- Modify:
  - `src/client/src/extension.ts`
  - `package.json` (`contributes.commands`, `contributes.menus`, `files`)
  - `BUILD.bazel` (new `esbuild` target `preview_bundle`, added to `vsix` `srcs`)
  - `eslint.config.mjs`, only if lint flags browser globals in `previewWebview.ts`

**Interfaces:**
- Consumes: Tasks 6 and 7.
- Produces:
  - `export function registerPreview(context: ExtensionContext, client: LanguageClient): { refresh(): void }` in `previewPanel.ts`.
  - Command `extension.openTrlcPreview`, titled `"TRLC: Open Preview to the Side"`, with icon `$(open-preview)`. It appears under `editor/title` with `"when": "resourceExtname == .trlc"` and `"group": "navigation"`.

- [ ] **Step 1: `previewWebview.ts`.** It calls `acquireVsCodeApi()` (declare its type locally) and posts `{type:"ready"}` on load. On a `model` message it sets `#root.innerHTML = renderPreview(model, lastNodes)` and keeps `lastNodes` when `model.nodes` is non-null. It uses event delegation on `#root`:
  - Text inputs commit on `change` (blur/Enter); textareas commit on Ctrl+Enter or blur. A commit is posted only when the value differs from the rendered one.
  - Selects commit on `change`.
  - `button.remove` posts `setField` with `value: null`.
  - `button.add-record` swaps in an inline form (type `<select>` from `model.recordTypes`, name input, Create/Cancel) that posts `addRecord`.
  - `a.record-name` posts `reveal`.

  The webview does not re-render while an input inside it has focus. It stores the pending model and renders it on `focusout`.

- [ ] **Step 2: `previewPanel.ts`.** One `WebviewPanel` (`ViewColumn.Beside`, `enableScripts`, `localResourceRoots: [Uri.joinPath(extensionUri, "src", "client", "out")]`, `retainContextWhenHidden: false`). The HTML comes from `previewShellHtml` with a random nonce and `asWebviewUri(.../out/preview.js)`. It creates a `PreviewController` with deps:
  - `sendRequest` → `client.sendRequest`
  - `applyEdit` → `workspace.applyEdit(client.protocol2CodeConverter.asWorkspaceEdit(edit))`
  - `showError` → `window.showErrorMessage`
  - `reveal` → `showTextDocument(uri, {viewColumn: ViewColumn.One, selection: new Range(line,0,line,0)})`

  It calls `setDocument` on `onDidChangeActiveTextEditor` when the new editor's document has `languageId === "TRLC"` and a `.trlc` path. It ignores other editors, including the webview gaining focus. The panel title is `Preview <basename>`.

- [ ] **Step 3: Wire into `extension.ts`.**
  - Call `registerPreview(context, client)` after `client.start()`.
  - Call `preview.refresh()` in the `handleWorkDoneProgress` "end" branch, next to `refreshScopeStatus`. Guard it because the preview is registered after the middleware is created.
  - Call it in the `onDidChangeDiagnostics` listener when the event touches the previewed uri.

- [ ] **Step 4: Build wiring.**
  - Add to `BUILD.bazel`: `esbuild(name = "preview_bundle", srcs = glob(["src/client/src/*.ts"]), entry_point = "src/client/src/previewWebview.ts", format = "iife", platform = "browser", minify = True, output = "src/client/out/preview.js")`.
  - Add `":preview_bundle"` to `vsix` `srcs`.
  - Add `"src/client/out/preview.js"` to `package.json` `files`.
  - Add the command and menu entries to `package.json`.

- [ ] **Step 5: Verify.**
  - `bazel build //:vsix //:typecheck` succeeds.
  - `bazel test //:test` passes.
  - `bazel build --config=lint //...` is clean.
  - `unzip -l bazel-bin/trlc-vscode-extension.vsix | grep preview.js` lists the file.

- [ ] **Step 6: Manual check.** Install the vsix and open `tests/fixtures/sample.trlc` together with a copy that has nested sections. Then:
  - Click the preview icon; headings show `#` / `##`.
  - Edit `x` and see the editor change and become dirty; Ctrl+Z reverts it.
  - Add a record inside the inner section.
  - Type a syntax error in the editor; the banner appears and inputs disable.
  - Fix the error; the preview recovers.

- [ ] **Step 7: Commit.** `git commit -m "feat(client): TRLC preview webview and command"`

---

### Task 9: Docs and e2e smoke

**Files:**
- Modify:
  - `docs/USER_GUIDE.md` (new "Preview" section: open, edit, add record, why it can turn read-only)
  - `docs/ARCHITECTURE.md` (add `trlc/preview*` to the LSP Features section, one paragraph on "server computes `WorkspaceEdit`, client applies")
  - `CHANGELOG.md` (an Unreleased entry)
  - `tests/e2e/suite/smoke.e2e.ts`

- [ ] **Step 1: Add an e2e assertion.** After the fixture `.trlc` file is open and diagnostics have arrived, `await commands.executeCommand("extension.openTrlcPreview")` resolves, and `window.tabGroups.all.flatMap(g => g.tabs)` contains a tab labelled `Preview sample.trlc`.

- [ ] **Step 2: Run.** `bazel test //tests/e2e:vscode_smoke_e2e --test_output=errors` passes.

- [ ] **Step 3: Write the docs.**

- [ ] **Step 4: Commit.** `git commit -m "docs: TRLC preview"`
