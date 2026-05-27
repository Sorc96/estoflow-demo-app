# 04 — Aha Moments and Roadblocks

---

## Aha Moments

- **`lifecycle.initialized` fires ONCE per `app.id`, not once per deploy** — This clicked when I
  read the lifecycle docs and saw the "per workbench ID" wording. Before this I was confused about
  why re-building with updated lifecycle code didn't re-run the seeding logic — the initial build
  had already marked the lifecycle as fired. Once I understood the `app.id` trick (change the ID →
  fresh lifecycle), I could confidently migrate instances without fear of duplicates. This became a
  recurring pattern across every subsequent task that needed to re-initialize data.

- **`update-or-create` is the correct lifecycle migration pattern** — Before understanding the
  lifecycle re-fire mechanism, I expected to need a separate "migrate" tool. The aha was realizing
  that the `lifecycle.initialized` body has full access to `this.models.<Model>.all.value`, so I
  could query existing instances, find them by index, and call `save({ id: existingId, ...newData })`
  to update in-place. This eliminated the "stale old instances + new duplicates" failure mode that a
  naive "always create" lifecycle would produce.

- **`autoRevert: false` bypasses the render_preview rasterizer** — The SVG logo build failed with
  `render_preview: the SVG could not be decoded as an image` and auto-reverted. I had assumed build
  failures were always fatal. The aha: `autoRevert: false` makes the build apply the manifest even
  when render_preview fails, so the panel works correctly in Studio UI even though the thumbnail
  preview can't be generated. This is a genuine capability of the system that isn't prominently
  documented — I found it by reasoning about what "revert" meant and checking the build options.

- **`pointsToMm(n)` accepts arithmetic expressions inline** — I initially assumed it was just a
  scalar multiplier and used it as `pointsToMm(88)` for a width, then explicitly computed center
  offsets in separate variables. The aha was seeing that Jinja2 allows `{{ g.x - bw / 2 }}` where
  `bw = pointsToMm(88)` — the result is a regular float in template context, so full arithmetic
  works. This simplified the tool template considerably once understood.

- **`x-geometry-relation` is the correct binding primitive for tool geometry fields** — I initially
  looked at `format: geometries` alone and wasn't sure if that was sufficient for the drawing canvas
  to recognize a field as a geometry binding. The aha came from finding `x-geometry-relation` in the
  schema docs — the combination of `format: geometries` + `x-geometry-relation` + `x-geometry-types`
  is the complete spell, and each part has a distinct role (field type vs. binding cardinality vs.
  geometry-type filter).

---

## Roadblocks

- **SVG logo rejected by render_preview rasterizer** — Tried to embed inline SVG in the panel HTML
  template. Build applied cleanly via `estoflow_build` but then immediately reverted with
  `render_preview: the SVG could not be decoded as an image`. Spent ~5 minutes trying to understand
  whether the SVG was malformed (it wasn't) before realizing the rasterizer simply doesn't support
  inline SVG in HTML context. Unblocked by: switching to `autoRevert: false`. System-fixable:
  yes — the rasterizer should either (a) skip the SVG-in-HTML path gracefully without reverting the
  entire build, or (b) document the limitation explicitly so authors know to set `autoRevert: false`.

- **Lifecycle not re-firing after code change** — After updating lifecycle body with new sci-fi
  values, rebuilt and saw old data still in instances. Took ~3 minutes to understand why. Root
  cause: `lifecycle.initialized` had already fired for `app.id: krtekstop-v2`; changing the body
  doesn't re-fire it. Unblocked by: `app.id` bump trick. System-fixable: the error surface here is
  zero — the system is silent. A `estoflow_inspect` result showing "lifecycle last fired at version
  N for this app.id" would have cut diagnosis time from 3 minutes to 30 seconds.

- **Geometry IDs not pre-known for new instances** — When seeding new RepellerDevice instances
  (Task 1), I had 1 existing geometry point but needed 3. I had to call `estoflow_seed_geometry`
  twice, capture the returned IDs, and then hard-code them into the lifecycle body. This two-step
  process (seed → capture ID → reference in lifecycle) felt fragile. The hardcoded geometry IDs in
  the lifecycle body are brittle — they reference a specific sketch. Spent ~4 minutes planning
  the flow before executing. Unblocked by: careful reading of `estoflow_seed_geometry` return value.
  System-fixable: a "named geometry" or "geometry alias" feature would let lifecycle reference
  geometries by semantic name instead of opaque ID.

- **Jinja2 single-line `{% set %}` constraint** — Attempted a multi-line ternary `{% set %}` block
  in the tool template (for conditional SVG rendering). Estoflow/Nunjucks rejected it. Spent ~2
  minutes restructuring to nested single-line form. Unblocked by: rewriting as
  `{% set x = a %}{% if cond %}{% set x = b %}{% endif %}`. This is a known Nunjucks constraint but
  not called out in Estoflow docs — a one-line note would save the confusion.

- **`FloodPoint` geometry IDs conflicted with `MoleMound` IDs in lifecycle** — In the first draft
  of the sci-fi lifecycle, I accidentally reused a MoleMound geometry ID for a FloodPoint instance
  (both pointing to `Ci2kKVG4tsIHIhLGQYDnS`). The build applied without error — Estoflow doesn't
  validate geometry ID uniqueness across model instances. I noticed the conflict only when reviewing
  the geometry mapping manually. Spent ~2 minutes debugging. Unblocked by: manual cross-check of
  geometry ID table. System-fixable: a validation warning for duplicate geometry ID references
  across instances would catch this class of error at build time.

- **`render_preview` failure message gave no hint about HTML context** — The error
  `render_preview: the SVG could not be decoded as an image` sounds like the SVG element itself is
  broken. It took time to understand the error means "the panel thumbnail renderer tried to rasterize
  the whole panel HTML as an image, saw an SVG, and couldn't handle it" — not that my SVG markup was
  invalid. A message like "render_preview: HTML panels with inline SVG are not supported by the
  thumbnail rasterizer; set autoRevert: false to skip preview generation" would have saved ~5 minutes.
