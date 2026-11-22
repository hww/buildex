Вот точный технический перевод текста на английский (включая завершение последней оборванной строки):

buildex.py — Architecture Documentation Generator

You describe the essence (registers, bits, codes, state machines, memory maps, timings) in YAML.
Diagrams (SVG) and tables (Markdown) are generated automatically. In the documentation text, they are referenced
using {{...}} directives.
Requires Python 3 and pip install pyyaml.

1. Directory Structure

buildex.py
src/<project>/spec/*.yaml     specifications (multiple documents per file, separated by ---)
src/<project>/docs/**/*.md    your text
src/<project>/docs/images/    hand-drawn images (tracked in git)
src/<project>/docs/gen/       GENERATED: NAME.svg, NAME.md, index.md (in .gitignore)
src/<project>/buildex.yaml    optional: lang: en | ru  ("Figure" / "Рис." captions)
out/<project>/docs/           mirror: built .md + gen/ + images/ (in .gitignore)
docs/                         copy of out/ via --deploy (tracked in git)
The source .md references images using standard relative paths, so previews in your editor
and on GitHub work without building:
markdown
![INSTRUCTION · 32 BIT](gen/insn.svg)      generated
![diagram](images/board.png)               hand-drawn
Используйте код с осторожностью.
The generator validates gen/NAME.svg links (an unknown name throws an error) and adds an auto-numbered
"Figure N — title" caption. If the line immediately following the image starts with *,
the caption is treated as manually written and is not added.
gen/NAME.md is the entity table (formerly NAME-table.md; if there is no table, it contains the diagram).
Legacy [include](gen/x-table.md) and [include](gen/x.svg) still work.
An empty .gitignore containing out/ and src/*/docs/gen/ is created on the first run
(if the file already exists, the generator will suggest what to add).

Demo Project

The demo/ folder contains a ready-to-use project showcasing almost all features: demo/src/demo/spec/*.yaml
(source specs), demo/src/demo/docs/*.md (text with directives), and demo/out/demo/docs/ (output).
Run with: cd demo && python3 buildex.py.

2. Usage

python3 buildex.py                  build all projects
python3 buildex.py gpu              build only the 'gpu' project
python3 buildex.py --watch          rebuild when files change
python3 buildex.py --deploy         copy out/ to docs/ after building
python3 buildex.py kinds            list available entity kinds
python3 buildex.py explain bitfield schema and example for an entity kind
python3 buildex.py new fsm ctl      YAML boilerplate (to stdout)

3. Reading Errors

Format: file:line kind 'name': what is wrong — hint.
• spec errors (isa.yaml:63 enum 'x': ...) — an issue in YAML. Nothing is rendered while these exist.
• docs/…md: {{table:NAME}}: ... — the text references something that doesn't exist. Two cases:
	• 'NAME' is not defined in any spec file ... (loaded specs: ...) — the name is missing from the
specifications. Check the loaded specs list: if the needed file isn't there, it wasn't placed in
src/<project>/spec/ or was overwritten.
	• 'NAME' (enum) has no diagram; its views: table — the entity exists, but doesn't support that view.
• Warnings (⚠️) do not block the build: uncovered bits, unreachable states, etc.

4. Entities


kind	What it describes	Default views
bitfield	register/word made of fields	diagram, table
enum	lookup table of codes (arbitrary columns)	table
encoding	family of word formats (common fields + variants by selector)	diagram, table
regmap	registers at offsets	table (enable diagram via view:)
memmap	address space	diagram, table
timing	timing diagram	diagram
fsm	finite state machine	diagram, table
pins	pinout: symbol block + table	diagram, table
const	named constants	table
connector	physical connector layout: header / dip / sil (plugin)	diagram, table

For an example and list of keys for any entity: python3 buildex.py explain <kind>.
The output: field is no longer needed. Only use view: [diagram, table] if you want to override the default views.

Fields in bitfield / encoding / regmap

yaml
- {name: OP, bits:, enum: alu_ops, access: rw, reset: 0, description: "..."}
- {name: F,  bit: 18}
- {name: RSV, bits:, reserved: true}          # hatched shading
- {name: J,  bit: 30, value: 1}                     # constant, rendered as "J / 1"
- {name: REG, bits:, overlay: true}          # alternative meaning for the same bits
Используйте код с осторожностью.
Legacy msb: 31, lsb: 24 syntax is also accepted.

Reusability

• enum: alu_ops on a field — the field's table links to the codes, and values are checked against bit width.
• extends: base_bitfield + fixed: {J: 0} — field inheritance.
• encoding: common fields are defined once under common, with when: {J: 0, I: 1} on each variant.
A variant can override a common field by name or reference an existing bitfield via ref:.
• regmap: registers can use ref: bitfield_name or inline fields.

Connectors (connector, plugin)

Geometry layered on top of an already defined pins entity: pins require pin: N (for a bus, use a list pin: [3,4,5,6]).
yaml
kind: connector
name: j1
pins: uart_pins            # single source of truth: names, directions, descriptions
type: header               # header (1-2 rows) | dip | sil | edge
rows: 2
per_row: 10
numbering: zigzag          # zigzag | rowwise | ccw (defaults to ccw for dip)
pin1: top-left             # top-left | top-right | bottom-left | bottom-right
key: {side: bottom, kind: notch}    # notch | tab | semicircle | none
orientation: vertical      # horizontal | vertical  (or rotate: 0 | 90 | 180 | 270)
Используйте код с осторожностью.
The edge type is for card-edge connectors (e.g., PCIe): sides: [B, A], per_row: 18, gap_after: 11 (key notch),
with pin numbers in pins formatted like pin: B14; multi-pin power and ground nets use a list pin: [B4, B7, ...].
See example in demo/src/demo/spec/demo_pcie.yaml.
Rotation is specified in the spec, not in the document text. Key position and pin 1 corner are defined in the
rotate: 0 coordinate system, and clockwise rotation is applied to the entire housing. Unassigned positions are
rendered as NC (with a warning). A single pins entity can be displayed across multiple connectors.

5. Markdown Directives


Directive	Result
{{auto:NAME}}	diagram + table
{{figure:NAME}}	diagram with "Figure N — …" caption
{{table:NAME}}	table with "Table N — …" caption
{{ref:NAME}} / {{ref:NAME.FIELD}}	link to an entity / to a field, code, register, or state
{{const:NAME}}	constant value
{{include:file.md}}	include another .md file (path relative to docs/)

To write a directive literally in the text, escape it with a backslash:
\{{auto:NAME}} renders as {{auto:NAME}} (same for \[include](...)).
Legacy [include](gen/x-table.md) and [include](gen/x.svg) work as before.
Figure and table numbering is automatic and scoped per file.

Plugins

New entities and directives are added via files without modifying buildex.py:
plugins/*.py                 global (available to all projects)
src/<project>/plugins/*.py   local to a specific project
plugins/_template.py         template (files starting with "_" are not loaded)
A plugin is a register(api) function:
python
def register(api):
    api.register_kind("mykind", validate=..., diagram=..., table=..., anchors=..., help=(description, example_yaml))
    api.register_directive("shout", lambda arg, ctx, P: arg.upper())     # {{shout:hi}} -> HI
Используйте код с осторожностью.
api provides Svg, svg_doc, tw, md_table, check_keys, as_int, W_, etc. (full list in
plugins/_template.py), so plugins do not need to import anything from buildex. Errors are reported via P.diag.err(where, msg).
A broken plugin does not crash the build: it emits a warning, and exceptions inside a plugin become standard
entity errors. A plugin that depends on other entities (like connector on pins) sets order=1.
Loaded plugins are printed at the start of the build; buildex.py kinds marks them with [plugin]; --watch
reloads plugins when modified.

Вот перевод оставшихся разделов на английский в том же стиле:

6. Validations

• All ![](path) links in documents point to existing files (paths are resolved relative to the .md file's directory;
http(s)://, #, data: links, and inline code in backticks are skipped);
• Duplicate names (indicating where the first definition occurred), typos in keys ("did you mean…");
• Forgotten --- separator between documents (duplicate key);
• Overlapping fields, out-of-bounds fields, uncovered bits;
• Duplicate and overlapping codes in enum, values exceeding bit width;
• Non-existent enum, ref, extends;
• Overlapping memmap regions, duplicate register offsets;
• Unknown fsm states, unreachable states, unknown signals in timing arrows.
File copying uses MD5 checksums: images/, gen/, and --deploy only touch modified files
(mtime remains unchanged, so git doesn't see unnecessary diffs); stale files in out/, src/.../gen/, and docs/ are removed.

7. What Was Promised vs. What Is Actually Done

The list from my first response, with an honest status check.

Item	Status
View selected by kind, output not needed	✅
Validator with clear error messages	✅
bitfield: access, reset, reserved, constants, enum reference	✅
enum, encoding, regmap, memmap, timing, fsm, pins, const	✅
extends, ref, enum: on fields, {{ref:}} links, auto-numbering	✅
Index of all entities	✅ (gen/index.md; not inserted into body text)
MSB/LSB above fields instead of every consecutive bit number	✅
Splitting a long word across rows (split)	✅
Rotating and fitting labels in narrow fields	✅ (based on text width estimation, not actual font metrics)
Constants inside fields, hatching for reserved, access/reset below fields	✅
Category-based color palette	⚠️ partial: constants are blue, reserved are hatched, groups in memmap; no legend
new / explain / --watch / errors with file and line numbers	✅ (no browser live-preview)
Backward compatibility with legacy format (code_list, memspace, output)	✅ (tested on synthetic examples, not across your entire project)
Renderer registry instead of if kind == ...	✅ (KINDS)
Schema validation via JSON Schema / Pydantic	❌ validations are hand-coded
block (block diagram), pipeline	❌
listing (code examples with auto-encoding)	❌
mixins	❌ (replaced by common in encoding and extends)
Export to Verilog / C from the same specs	❌
setup/hold/delay in timing	⚠️ labeled arrows (arrows) only
{{ref:}} links inside YAML values	❌ only in Markdown and tables

I have visually inspected the rendered diagrams for encoding, fsm, and timing. The remaining kinds
(bitfield, regmap, memmap, pins) build without errors, though I haven't inspected them manually.

8. Common Issues

• Can't see any diagrams — check src/<project>/docs/gen/*.svg (and the copy in out/).
• Empty specs: list — there are no files in src/<project>/spec/, or their extensions aren't .yaml/.yml.
• duplicate key 'title' — a --- separator is missing between two YAML documents.
• duplicate name — the same name is defined in two files; remove one.
• Do not copy example/ into your project — file and entity names will collide with yours and overwrite them.
Нужно ли собрать оба фрагмента в единый README.md с поправкой мелких опечаток (например, example/ вместо demo/ в разделе 8), или перевести остальные файлы проекта?
