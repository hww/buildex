# buildex.py — architecture documentation generator

You describe the **essence** (registers, bits, codes, state machines, memory maps, timings) in YAML.
Images (SVG) and tables (Markdown) are built automatically. The documentation text refers to them
with `{{...}}` directives.

You need Python 3 and `pip install pyyaml`.

## 1. Where things live

```
buildex.py
src/<project>/spec/*.yaml      descriptions (multiple documents per file, separated by ---)
src/<project>/docs/**/*.md     your text
src/<project>/docs/images/     hand-drawn (committed)
src/<project>/docs/gen/        GENERATED: NAME.svg, NAME.md, index.md (in .gitignore)
src/<project>/buildex.yaml     optional: lang: en | ru  (captions "Figure"/"Рис.")
out/<project>/docs/            mirror: finished .md + gen/ + images/ (in .gitignore)
docs/                          copy of out/ via --deploy (committed)
```

The source `.md` refers to images by an ordinary relative path, so previews in the
editor and on GitHub work without building:

```markdown
![INSTRUCTION · 32 BIT](gen/insn.svg)      generated
![diagram](images/board.png)               hand-drawn
```

The generator checks `gen/NAME.svg` references (an unknown name = error) and adds the caption
"Figure N — title" with automatic numbering. If the line immediately after the image starts
with `*`, the caption is considered hand-written and is not added.

`gen/NAME.md` — the entity's table (formerly `NAME-table.md`; if there is no table, then the figure).
The old `[include](gen/x-table.md)` and `[include](gen/x.svg)` still work.
An empty `.gitignore` with `out/` and `src/*/docs/gen/` is created on first run
(if the file already exists, the generator will tell you what to add).

## Demo project

The `demo/` folder contains a ready-made project that shows almost all the features: `demo/src/demo/spec/*.yaml`
(source descriptions), `demo/src/demo/docs/*.md` (text with directives) and `demo/out/demo/docs/` (the result).
Run: `cd demo && python3 buildex.py`.

## 2. Running

```
python3 buildex.py                  build all projects
python3 buildex.py gpu              only the gpu project
python3 buildex.py --watch          rebuild when files change
python3 buildex.py --deploy         after building, copy out/ to docs/
python3 buildex.py kinds            list entities
python3 buildex.py explain bitfield schema and example for an entity
python3 buildex.py new fsm ctl      YAML template (to stdout)
```

## 3. How to read errors

Format: `file:line kind 'name': what is wrong — hint`.

* `spec` errors (`isa.yaml:63 enum 'x': ...`) — a problem in the YAML. While they exist, nothing is drawn.
* `docs/…md: {{table:NAME}}: ...` — the text refers to something that does not exist. Two cases:
  * `'NAME' is not defined in any spec file ... (loaded specs: ...)` — there is no such name in the
    specifications. Look at the `loaded specs` list: if the needed file is not there, it did not end up in
    `src/<project>/spec/` or was overwritten.
  * `'NAME' (enum) has no diagram; its views: table` — the entity exists, but it has no such view.
* Warnings (⚠️) do not block the build: uncovered bits, unreachable states.

## 4. Entities

| kind | What it describes | Default views |
|---|---|---|
| `bitfield` | a register/word made of fields | diagram, table |
| `enum` | a table of codes (any columns) | table |
| `encoding` | a family of word formats (common fields + variants by selector) | diagram, table |
| `regmap` | registers by offset | table (diagram enabled with `view:`) |
| `memmap` | address space | diagram, table |
| `timing` | timing diagram | diagram |
| `fsm` | finite state machine | diagram, table |
| `pins` | pins: symbol block + table | diagram, table |
| `const` | named constants | table |

Example and list of keys for any entity: `python3 buildex.py explain <kind>`.

The `output:` field is no longer needed. It is only needed to change the default view:
`view: [diagram, table]`.

### A field in bitfield / encoding / regmap

```yaml
- {name: OP, bits: [17,13], enum: alu_ops, access: rw, reset: 0, description: "..."}
- {name: F,  bit: 18}
- {name: RSV, bits: [6,0], reserved: true}          # hatching
- {name: J,  bit: 30, value: 1}                     # constant, drawn as "J / 1"
- {name: REG, bits: [12,7], overlay: true}          # alternative meaning of the same bits
```

The old `msb: 31, lsb: 24` are also accepted.

### Reuse

* `enum: alu_ops` on a field — the field's table will reference the codes, and the values are checked against the width.
* `extends: base_bitfield` + `fixed: {J: 0}` — field inheritance.
* `encoding`: common fields once in `common`, each variant has `when: {J: 0, I: 1}`.
  A variant can override a common field by name or take a ready-made bitfield via `ref:`.
* `regmap`: a register has `ref: bitfield_name` or inline `fields`.

## 5. Directives in Markdown

| Directive | Result |
|---|---|
| `{{auto:NAME}}` | figure + table |
| `{{figure:NAME}}` | figure with caption "Figure N — …" |
| `{{table:NAME}}` | table with caption "Table N — …" |
| `{{ref:NAME}}` / `{{ref:NAME.FIELD}}` | link to an entity / to a field, code, register, state |
| `{{const:NAME}}` | constant value |
| `{{include:file.md}}` | insert another .md (path from `docs/`) |

To write a directive literally in the text, put a backslash in front of it:
`\{{auto:NAME}}` is output as `{{auto:NAME}}` (same for `\[include](...)`).


The old `[include](gen/x-table.md)` and `[include](gen/x.svg)` work as before.
Figure and table numbering is automatic, separately per file.

## 6. Checks

* every `![](path)` in the documents points to an existing file (the path is resolved from the folder of the `.md`
  itself; `http(s)://`, `#`, `data:` links and code in backticks are skipped);

* duplicate names (with a note of where the first definition is), typos in keys ("did you mean…");
* a forgotten `---` between documents (duplicate key);
* overlapping fields, fields beyond the width, uncovered bits;
* duplicate and overlapping codes in `enum`, values that do not fit the width;
* non-existent `enum`, `ref`, `extends`;
* overlapping `memmap` regions, repeated register offsets;
* unknown fsm states, unreachable states, unknown signals in timing arrows.

Copying is done by md5: `images/`, `gen/` and `--deploy` only touch changed files
(mtime does not change, git sees no extra edits); stale files in `out/`, `src/.../gen/` and `docs/` are deleted.

## 7. What was promised and what was actually done

The list from my first answer, with an honest status.

| Item | Status |
|---|---|
| View is chosen by `kind`, `output` not needed | ✅ |
| Validator with clear messages | ✅ |
| bitfield: access, reset, reserved, constants, enum reference | ✅ |
| enum, encoding, regmap, memmap, timing, fsm, pins, const | ✅ |
| `extends`, `ref`, `enum:` on a field, `{{ref:}}` links, automatic numbering | ✅ |
| Index of all entities | ✅ (`gen/index.md`; not inserted into text) |
| msb/lsb above the field, not all numbers in a row | ✅ |
| Splitting a long word across lines (`split`) | ✅ |
| Rotating and fitting labels in narrow fields | ✅ (by text width estimate, not by font) |
| Constants in a field, reserved hatching, access/reset under the field | ✅ |
| Palette by category | ⚠️ partially: constants are blue, reserved is hatched, groups in memmap; no legend |
| `new` / `explain` / `--watch` / errors with file and line | ✅ (no browser preview) |
| Compatibility with the old format (`code_list`, `memspace`, `output`) | ✅ (tested on synthetic examples, not on your whole project) |
| A registry of renderers instead of `if kind == ...` | ✅ (`KINDS`) |
| Schema via JSON Schema / pydantic | ❌ checks are written by hand in the code |
| `block` (block diagram), `pipeline` | ❌ |
| `listing` (program examples with auto-encoding) | ❌ |
| `mixins` | ❌ (replaced by `common` in `encoding` and `extends`) |
| Export to Verilog / C from the same specifications | ❌ |
| setup/hold/delay in timing | ⚠️ only arrows with a label (`arrows`) |
| `{{ref:}}` links inside YAML values | ❌ only in Markdown and tables |

Visually I checked the `encoding`, `fsm` and `timing` images. The other views
(`bitfield`, `regmap`, `memmap`, `pins`) build without errors, but I did not examine them by hand.

## 8. Common problems

* **I don't see the images** — look in `src/<project>/docs/gen/*.svg` (and the copy in `out/`).
* **Empty `specs:` list** — the files are not in `src/<project>/spec/` or they do not have a `.yaml/.yml` extension.
* **`duplicate key 'title'`** — a `---` is missing between two documents in the YAML.
* **`duplicate name`** — one name is defined in two files; keep only one.
* **Don't copy `example/` into your project** — the file and entity names will clash with yours and overwrite them.
