# buildex.py — Architecture Documentation Generator

You describe the **essence** (registers, bits, codes, state machines, memory maps, timings) in YAML.

Images (SVG) and tables (Markdown) are generated automatically. In the documentation text, you reference them using `{{...}}` directives.

Requires Python 3 and `pip install pyyaml`.

## 1. Directory Structure

```text
buildex.py
src/<project>/spec/*.yaml      specifications (multiple documents per file, separated by ---)
src/<project>/docs/**/*.md     your documentation text
src/<project>/docs/images/     manually created images (committed)
src/<project>/docs/gen/        GENERATED: NAME.svg, NAME.md, index.md (in .gitignore)
src/<project>/buildex.yaml     optional: lang: en | ru (captions: "Fig." / "Figure")
out/<project>/docs/            mirror: ready-to-use .md + gen/ + images/ (in .gitignore)
docs/                          copy of out/ created by --deploy (committed)
```

Source `.md` files reference images using ordinary relative paths, so previews work in the editor and on GitHub without running the build:

```markdown
![INSTRUCTION · 32 BIT](gen/insn.svg)      generated
![schematic](images/board.png)             manually created
```

The generator validates `gen/NAME.svg` references (an unknown name is an error) and adds a caption: "Fig. N — Title", with automatic numbering. If the line immediately following the image starts with `*`, the caption is considered manually provided and is not added.

`gen/NAME.md` is the entity table (formerly `NAME-table.md`; if no table exists, the image is used instead).

Legacy `[include](gen/x-table.md)` and `[include](gen/x.svg)` references continue to work.

An empty `.gitignore` containing `out/` and `src/*/docs/gen/` is created on the first run. If the file already exists, the generator tells you what to add.

## Demo Project

The `demo/` directory contains a ready-to-use project demonstrating almost all features:

- `demo/src/demo/spec/*.yaml` — source specifications
- `demo/src/demo/docs/*.md` — documentation text with directives
- `demo/out/demo/docs/` — generated output

Run it with:

```bash
cd demo && python3 buildex.py
```

## 2. Running

```bash
python3 buildex.py                  build all projects
python3 buildex.py gpu              build only the gpu project
python3 buildex.py --watch          rebuild when files change
python3 buildex.py --deploy         copy out/ to docs/ after building
python3 buildex.py kinds            list entity kinds
python3 buildex.py explain bitfield show schema and example for an entity
python3 buildex.py new fsm ctl      print a YAML template to stdout
```

## 3. Understanding Errors

Format: `file:line kind 'name': what is wrong — suggestion`.

- `spec` errors (`isa.yaml:63 enum 'x': ...`) indicate a problem in the YAML. Nothing is rendered while these errors remain.
- `docs/…md: {{table:NAME}}: ...` means the documentation references something that does not exist. There are two possible cases:
  - `'NAME' is not defined in any spec file ... (loaded specs: ...)` — this name does not exist in the specifications. Check the `loaded specs` list: if the required file is missing, it was not placed in `src/<project>/spec/` or has been overwritten.
  - `'NAME' (enum) has no diagram; its views: table` — the entity exists, but does not provide the requested view.
- Warnings (⚠️) do not prevent the build: uncovered bits, unreachable states, etc.

## 4. Entity Kinds

| kind | What it describes | Default views |
|---|---|---|
| `bitfield` | register/word consisting of fields | diagram, table |
| `enum` | code table (any columns) | table |
| `encoding` | family of word formats (shared fields + selector-based variants) | diagram, table |
| `regmap` | registers at offsets | table (diagram enabled with `view:`) |
| `memmap` | address space | diagram, table |
| `timing` | timing diagram | diagram |
| `fsm` | finite-state machine | diagram, table |
| `pins` | pins: symbol block + table | diagram, table |
| `const` | named constants | table |
| `connector` | physical connector layout: header / dip / sil (**plugin**) | diagram, table |

For an example and the list of keys supported by any entity, run:

```bash
python3 buildex.py explain <kind>
```

The `output:` field is no longer required. Use it only if you want to change the default views:

```yaml
view: [diagram, table]
```

### Fields in bitfield / encoding / regmap

```yaml
- {name: OP, bits: [17,13], enum: alu_ops, access: rw, reset: 0, description: "..."}
- {name: F,  bit: 18}
- {name: RSV, bits: [6,0], reserved: true}          # hatched
- {name: J,  bit: 30, value: 1}                     # constant, rendered as "J / 1"
- {name: REG, bits: [12,7], overlay: true}          # alternative meaning of the same bits
```

The legacy `msb: 31, lsb: 24` format is also supported.

### Reuse

- `enum: alu_ops` on a field — the field's table links to the codes; values are checked against the field width.
- `extends: base_bitfield` + `fixed: {J: 0}` — field inheritance.
- `encoding`: define shared fields once in `common`; each variant uses `when: {J: 0, I: 1}`. A variant can override a common field by name or reuse an existing bitfield through `ref:`.
- `regmap`: a register can use `ref: bitfield_name` or inline `fields`.

### Connectors (`connector` plugin)

This adds physical geometry on top of an already defined `pins` entity. Each pin needs a `pin: N` field (for a bus, use a list such as `pin: [3,4,5,6]`).

```yaml
kind: connector
name: j1
pins: uart_pins            # single source of truth: names, directions, descriptions
type: header               # header (1–2 rows) | dip | sil | edge
rows: 2
per_row: 10
numbering: zigzag          # zigzag | rowwise | ccw (ccw is the default for dip)
pin1: top-left              # top-left | top-right | bottom-left | bottom-right
key: {side: bottom, kind: notch}    # notch | tab | semicircle | none
orientation: vertical       # horizontal | vertical (or rotate: 0 | 90 | 180 | 270)
```

The `edge` type represents a card-edge connector (for example, PCIe):

- `sides: [B, A]`
- `per_row: 18`
- `gap_after: 11` (key slot)

Pin numbers in `pins` use the `pin: B14` format. Power and ground signals occupying multiple positions are specified as lists, for example `pin: [B4, B7, ...]`.

See `demo/src/demo/spec/demo_pcie.yaml` for an example.

Rotation is specified in the specification, not in the documentation text. The key position and pin 1 angle are defined in the `rotate: 0` coordinate system; clockwise rotation is then applied to the entire connector body. Positions without a signal are rendered as NC (with a warning). The same `pins` entity can be displayed on multiple connectors.

## 5. Markdown Directives

| Directive | Result |
|---|---|
| `{{auto:NAME}}` | diagram + table |
| `{{figure:NAME}}` | diagram with caption "Fig. N — …" |
| `{{table:NAME}}` | table with caption "Table N — …" |
| `{{ref:NAME}}` / `{{ref:NAME.FIELD}}` | link to an entity / field, code, register, or state |
| `{{const:NAME}}` | constant value |
| `{{include:file.md}}` | include another `.md` file (path relative to `docs/`) |

To display a directive literally in the text, prefix it with a backslash:

```text
\{{auto:NAME}}
```

This will be rendered as `{{auto:NAME}}`. The same applies to `\[include](...)`.

Legacy `[include](gen/x-table.md)` and `[include](gen/x.svg)` references continue to work as before.

Figure and table numbering is automatic and maintained separately for each file.

## Plugins

New entity kinds and directives can be added using files, without modifying `buildex.py`:

```text
plugins/*.py                 global (for all projects)
src/<project>/plugins/*.py   project-specific
plugins/_template.py         template (files starting with "_" are not loaded)
```

A plugin is a function called `register(api)`:

```python
def register(api):
    api.register_kind("mykind", validate=..., diagram=..., table=..., anchors=..., help=(description, example_yaml))
    api.register_directive("shout", lambda arg, ctx, P: arg.upper())     # {{shout:hi}} -> HI
```

`api` provides `Svg`, `svg_doc`, `tw`, `md_table`, `check_keys`, `as_int`, `W_`, etc. (see `plugins/_template.py` for the full list). Therefore, a plugin does not need to import anything from `buildex`.

Errors are reported using `P.diag.err(where, msg)`.

A broken plugin does not crash the build: it produces a warning, and an exception inside a plugin is converted into an ordinary error associated with the entity name.

A plugin that depends on other entities (such as `connector`, which depends on `pins`) must specify `order=1`.

Loaded plugins are listed at the beginning of the build. `buildex.py kinds` marks them with `[plugin]`, and `--watch` reloads plugins when they change.

## 6. Validation

- All `![](path)` references in documentation point to existing files. Paths are resolved relative to the directory containing the `.md` file. The following are skipped: `http(s)://`, `#`, `data:`, and code enclosed in backticks.
- Duplicate names (with the location of the first definition), misspelled keys ("did you mean…").
- Missing `---` separators between YAML documents (duplicate keys).
- Overlapping fields, fields outside the word width, uncovered bits.
- Duplicate and overlapping codes in `enum`; values that do not fit the specified width.
- Nonexistent `enum`, `ref`, and `extends` references.
- Overlapping `memmap` regions and duplicate register offsets.
- Unknown FSM states, unreachable states, and unknown signals in timing arrows.

Copying is based on md5: `images/`, `gen/`, and `--deploy` only touch files that have changed (modification times remain unchanged, so Git does not report unnecessary changes). Stale files are removed from `out/`, `src/.../gen/`, and `docs/`.

## 7. What Was Promised vs. What Has Actually Been Implemented

The list from my first response, with an honest status for each item.

| Item | Status |
|---|---|
| View selected by `kind`; `output` not required | ✅ |
| Validator with clear error messages | ✅ |
| bitfield: access, reset, reserved, constants, enum reference | ✅ |
| enum, encoding, regmap, memmap, timing, fsm, pins, const | ✅ |
| `extends`, `ref`, `enum:` on fields, `{{ref:}}` links, automatic numbering | ✅ |
| Index of all entities | ✅ (`gen/index.md`; not inserted into documentation) |
| msb/lsb displayed above the field, rather than all bit numbers in sequence | ✅ |
| Splitting long words into rows (`split`) | ✅ |
| Rotation and fitting labels into narrow fields | ✅ (based on estimated text width, not actual font metrics) |
| Constants in fields, hatching for reserved bits, access/reset below fields | ✅ |
| Category-based color palette | ⚠️ partially: constants are blue, reserved fields are hatched, groups are shown in memmap; no legend |
| `new` / `explain` / `--watch` / errors with file and line | ✅ (no browser preview) |
| Compatibility with legacy formats (`code_list`, `memspace`, `output`) | ✅ (tested with synthetic examples, not with your entire project) |
| Renderer registry instead of `if kind == ...` | ✅ (`KINDS`) |
| Schema using JSON Schema / pydantic | ❌ validation is implemented manually in code |
| `block` (flowchart), `pipeline` | ❌ |
| `listing` (code examples with automatic encoding) | ❌ |
| `mixins` | ❌ (replaced by `common` in `encoding` and `extends`) |
| Export to Verilog / C from the same specifications | ❌ |
| setup/hold/delay in timing | ⚠️ only labeled arrows (`arrows`) |
| `{{ref:}}` links inside YAML values | ❌ supported only in Markdown and tables |

I visually checked the `encoding`, `fsm`, and `timing` diagrams. The other views (`bitfield`, `regmap`, `memmap`, `pins`) build without errors, but I have not inspected them manually.

## 8. Common Issues

- **Images are not visible** — check `src/<project>/docs/gen/*.svg` (and the copy in `out/`).
- **Empty `specs:` list** — there are no files in `src/<project>/spec/`, or their extensions are not `.yaml` / `.yml`.
- **`duplicate key 'title'`** — a `---` separator is missing between two YAML documents.
- **`duplicate name`** — the same name is defined in two files; keep only one.
- **Do not copy `example/` into your project** — its file and entity names may match yours and overwrite them.
