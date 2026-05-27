# 05 — MCP Agent Experience

Feedback from my perspective as an AI agent that used the Estoflow MCP to build and iterate on a
workbench (`app.yaml`) across two sessions.

---

## Orientation cost

I read a substantial number of resources before writing a single line of YAML in the first session.
The essential ones were:

- `estoflow://docs/models` — absolutely required; defines the field annotation vocabulary
  (`x-geometry-relation`, `x-geometry-types`, `x-measured`, `format: geometries`) without which I
  couldn't have written a correct model schema
- `estoflow://docs/tools` — required; the `template` Jinja context and `pointsToMm()` helper are
  not guessable
- `estoflow://docs/calculators` — required for the `x-measured` auto-computation pattern
- `estoflow://docs/lifecycle` — required to understand the one-fire-per-`app.id` behavior

Resources I read that I didn't strictly need:
- `estoflow://docs/aggregations` — read in full; ended up needing only 20% of it (the `groupBy`
  shape)
- `estoflow://docs/panels` — read in full; most of the CSS/layout detail was trial-and-error anyway

Resources I NEEDED but couldn't find (or found too late):
- **`autoRevert` build option** — I don't recall finding this in the docs before hitting the SVG
  failure. I discovered it by inspecting the `estoflow_build` tool schema. A single sentence in
  `estoflow://docs/panels` ("Panels with complex HTML or inline SVG may cause render_preview to
  fail; set `autoRevert: false` to prevent revert in this case") would have been immediately
  actionable.
- **Lifecycle re-fire semantics (per-`app.id`)** — The docs described the lifecycle firing "once
  per project" but the key detail — that "once" is scoped to the `app.id`, not the project ID — was
  not prominently stated. I inferred it by experimentation.

---

## Tool surface coherence

**Names vs. behavior — mostly good.** `estoflow_build`, `estoflow_inspect`, `estoflow_validate`,
`estoflow_seed_geometry`, `estoflow_render_preview` all do what their names suggest.

**Surprises:**

- `estoflow_build` silently succeeded on geometry ID conflicts (duplicate IDs across models). I
  expected it to at least warn. The tool's contract feels like "manifest is syntactically valid →
  build succeeds," but semantic cross-references (geometry IDs, model references) aren't validated.

- `estoflow_inspect` returns a flat dump of all instances; there's no filter parameter. When a
  project has 15+ instances across 6 models, reading the full dump to find one model's instances
  is verbose. A `model` filter parameter (e.g., `estoflow_inspect({ model: "MoleProtConfig" })`)
  would reduce token usage significantly.

- `estoflow_seed_geometry` returns the new geometry ID, which is correct. But it doesn't return
  the full geometry object (type, coordinates). If I need to verify the seed, I must call
  `estoflow_inspect` separately — that's an extra round-trip for confirmation.

**Tool I didn't realize existed until late:**
- `estoflow_inspect_calculator` and `estoflow_inspect_geometry` and `estoflow_inspect_panel` — I
  saw these in the tool list but used `estoflow_inspect` generically for most of the session.
  If the specialized inspectors return richer type-specific data (e.g., `inspect_geometry` shows
  coordinates + bounding box), they should be the recommended entry points, not the generic one.
  This wasn't obvious from the generic inspector alone.

**One tool that should be ONE but is several:**
- `estoflow_save_and_apply` vs `estoflow_build` — I don't recall if both exist with distinct
  semantics or if one is an alias. The distinction (if any) was not clear to me. If they are
  functionally the same, one should be deprecated.

---

## Error surfaces

The worst unhelpful error I hit verbatim:
```
render_preview: the SVG could not be decoded as an image
```

What would have helped:
```
render_preview failed: the panel thumbnail renderer does not support inline SVG or complex CSS
gradients in HTML panel bodies. The manifest was NOT applied because autoRevert defaults to true.
To apply the manifest without a thumbnail preview, set autoRevert: false in your build options.
```

The current message implies the SVG is broken. The actual cause is a rasterizer limitation. An
agent (or human) author's first instinct is to debug the SVG, not to look for a build flag.

Second-worst:
- When lifecycle doesn't re-fire after a code change, there is NO error. The tool succeeds, the
  build applies, and instances remain unchanged. The silent no-op is the worst kind of failure
  for an agent — it looks like success. A `estoflow_inspect` result that includes
  `lifecycle_last_fired: { app_id: "krtekstop-v2", version: 11 }` would make the state visible.

---

## Round-trip cost

- **`estoflow_inspect` then `estoflow_build`** — standard pattern, two calls, reasonable.
- **`estoflow_seed_geometry` then manual ID extraction** — one call to seed, then I have to read
  the return value, copy the ID, and embed it in the next YAML draft. Not bad but error-prone (ID
  typos). A `estoflow_get_geometry({ id })` call to verify the seed would be useful.
- **Lifecycle change cycle** — `Read app.yaml` → `Edit app.yaml` → `estoflow_build` → `estoflow_inspect`.
  Four calls to verify one lifecycle change. The inner loop feels right; no obvious compression.
- **`estoflow_validate` before `estoflow_build`** — I used this pattern but it doesn't catch
  runtime errors (wrong variable names in JS bodies, geometry ID mismatches). So validate → build
  → inspect is always three calls minimum, and validate doesn't always buy much.

---

## Stateful surprises

- **`lifecycle.initialized` one-fire-per-`app.id`** — the biggest stateful surprise. Once the
  lifecycle fires, changing its body has no effect until the `app.id` changes. This is correct
  behavior for production but very surprising during development.

- **Applied-vs-buffer divergence** — On one build that reverted, the local `app.yaml` file was
  already updated but the applied workbench in the session was the pre-SVG version. I had to be
  careful to understand which version the running session was using. A clearer "applied version"
  indicator in `estoflow_inspect` results would help.

- **`estoflow_open_session` / `estoflow_select_session` semantics** — I don't recall which I used
  first, or whether the session was already open from a prior run. The session resumption behavior
  was opaque — I assumed the active session was already set. See `02-session.jsonl` for actual
  call sequence.

---

## One change that would help me most

**Make `lifecycle.initialized` re-fire on body change during development mode (or add an explicit
`estoflow_rerun_lifecycle` tool).** The `app.id` bump trick works but it's a workaround, not a
feature. It forces me to maintain a meaningless version counter in the manifest and to remember to
increment it every time I change lifecycle logic. A `--force-lifecycle` build flag or an explicit
`estoflow_rerun_lifecycle` tool would eliminate this entire category of confusion, saving 3–5
minutes per lifecycle-change iteration.
