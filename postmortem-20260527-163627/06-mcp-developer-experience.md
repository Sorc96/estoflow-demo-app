# 06 — MCP Developer Experience (Human Author Perspective)

Feedback on the authoring surfaces a human developer would face writing this workbench by hand.

---

## app.yaml structure

**Where the declarative grammar carries its weight:**

- **`models` section** — very natural. JSON Schema with Estoflow extensions (`x-geometry-relation`,
  `x-measured`, `x-geometry-types`, `format: geometries`) is intuitive for anyone with JSON Schema
  experience. The extensions are additive, not disruptive to the base schema shape. Enums with
  `x-label` are a particularly good pattern — the enum value is the data key; the label is UI only.

- **`aggregations` section** — readable once you understand the two-phase shape (`groupBy` +
  `compute`). The `groupBy: null` singleton pattern for project-level aggregates is a small gotcha
  but documented.

- **`tools` section** — good for simple markers (point + label). The `template` Jinja context is
  the hard part (see below), but the structural declaration (`model`, `icon`, `label`,
  `namePrefix`, `dialogFields`) is easy.

**Where the grammar feels awkward:**

- **`calculators` section with `inputs` / `outputs` / `derivations`** — the most complex section.
  The relationship between `inputs` (model fields auto-populated from `x-measured` aggregates),
  `derivations` (JS computed fields), and `outputs` (fields the panel/PDF reads from) requires
  three mental models at once. A human author needs to hold the whole data-flow graph in their head
  to write this correctly.

- **`lifecycle.initialized.body`** — inline JS in a YAML multi-line string is the most friction-
  heavy section. Three issues: (1) `this.models` API is not discoverable from the YAML itself —
  you need to read separate runtime docs; (2) error messages from runtime JS errors are opaque;
  (3) the "query all then filter by `project_id`" pattern is boilerplate that every lifecycle
  author will independently rediscover.

- **`panels[].body`** — inline HTML/CSS/Jinja2 in YAML is hard to edit in a textarea. Line
  continuation and indentation make it easy to accidentally break YAML parsing. A human author
  would want a dedicated panel body editor, not a multi-line string.

---

## Imperative blocks

### Which body context was hardest to keep straight?

**`lifecycle.initialized.body`** was hardest. The API surface is `this.models.<Model>.all.value`,
`this.models.<Model>.save({...})`, `this.project.id` — but none of this is visible in the YAML
declaration. You have to read external runtime-contract docs (or experiment) to discover that:
- `all.value` returns all instances across all projects
- `project_id` filtering is the author's responsibility (not automatic)
- `save({ id })` is update; `save({})` without `id` is create

Derivations were second hardest. The injected variables (`instance`, `models`, `compute`) have
subtle differences from lifecycle context — `instance` is the current record, not the full list.
This trips authors who copy patterns from lifecycle into derivations.

### Discoverability from the editor

Poor. There is no inline autocomplete for body context. A human author in the Studio textarea would
need to keep a browser tab open to the runtime-contract `.d.ts` at all times. The docstring pattern
(`// @param {MoleMound} instance`) that TypeScript can leverage is absent — the runtime just
injects variables without annotation.

The single most valuable addition: **hover-docs for the injected context** in the Studio editor.
Hovering over `this.models` in a lifecycle body should show the type signature.

### Compile-passes-but-runtime-fails

Yes, this happened. A lifecycle body with `this.models.MoleProtConfig.save({ pointGeometryId: '...' })`
where `pointGeometryId` didn't exist in the model schema yet compiled (validated) successfully but
silently dropped the field at runtime. `estoflow_validate` passed; the instance was created without
the field. The fix was adding the schema field to the model. A schema-aware body linter ("field
`pointGeometryId` referenced in lifecycle save but not declared in `MoleProtConfig` model") would
have caught this immediately.

### Nunjucks template surprises

- **`{% set %}` across multiple lines** — silently fails or throws; must be single-line. Not
  documented in Estoflow's template docs (at least not that I found). Caught by experimenting.

- **`pointsToMm()` is a function, not a Jinja filter** — correct Jinja idiom would be
  `{{ 88 | pointsToMm }}` but it's actually `{{ pointsToMm(88) }}`. The function-call form works
  but is easy to confuse with Jinja filters. A note in the docs ("these helpers are template
  functions, not filters") would help.

- **`g` vs `geoAll` vs individual geo arrays** — in tool templates, `g` is the primary geometry.
  But when a tool has multiple geometry fields, the naming for additional geometries isn't obvious
  from docs. I don't recall if I hit this or just avoided it by using only `pointGeometryId`. See
  `02-session.jsonl`.

- **`isTemporary` conditional** — useful for showing a ghost preview during placement. The name is
  intuitive, but the fact that it's a template-level boolean (not a model field) required reading
  the tool template docs to discover. Undiscoverable from the model schema alone.

---

## Calculator + aggregations + views

**Calculators — right in concept, verbose in execution.** The separation between the `inputs`
(auto-populated from `x-measured`), `derivations` (JS bodies), and `outputs` (what the panel sees)
is the right design — it prevents circular dependencies and makes the data flow explicit. But the
YAML is verbose: declaring a simple derived field requires 3–4 lines in `derivations` plus a
corresponding entry in `outputs`.

**Aggregations — the grammar is flexible but the `groupBy: null` singleton is a footgun.** Most
workbenches will have both per-instance aggregates (group by foreign key) and project-level totals
(group by null). The `groupBy: null` form is not intuitive — `null` as a grouping key means
"aggregate the whole project" but looks like a missing value.

**Views — I don't recall authoring views directly in this session.** See `02-session.jsonl`.

---

## The bid-panel / branded-output pattern

The panel section (`panels[].body`) was the most human-unfriendly part of the whole manifest. Reasons:

1. **HTML + CSS + Jinja2 + YAML multi-line string** — four syntaxes in one block. Any one of them
   can break the whole panel silently.

2. **No live preview in the textarea editor** (as far as I could tell). A human author is
   effectively writing blind — build, wait for apply, open Studio to see the panel. Iteration is
   slow.

3. **The `render_preview` rasterizer limitation with inline SVG** causes auto-revert unless
   `autoRevert: false` is set. A human author with no prior knowledge of this would be confused
   for a long time — their SVG is valid HTML, but the build keeps reverting.

4. **CSS class naming collision risk** — the panel HTML is injected into a broader Studio DOM. The
   `.ht-logo` class name I used was chosen arbitrarily. There's no namespace or shadow-DOM
   isolation, so a class name collision with Studio CSS would silently break the panel appearance.
   A documented pattern for scoped class names (e.g., `data-estoflow-panel` scope) would help.

---

## Error messages: human impact

**Would have HELPED a human author more than me:**
- The `render_preview` SVG error (see 04) — a human would read "SVG could not be decoded" as "your
  SVG is broken," spend hours validating the SVG, and never find the issue because the SVG is fine.
  I at least had the context to reason about the rasterizer. A human needs the explicit "set
  `autoRevert: false`" instruction in the error message.

**Would have been MORE CONFUSING for a non-LLM author:**
- Any runtime error from a lifecycle JS body — the error surface appears to be just "lifecycle
  failed" without a stack trace or line number. A human author can't diff a failing lifecycle
  against a working one without knowing which line threw. An LLM can reason about likely failure
  modes; a human cannot.
