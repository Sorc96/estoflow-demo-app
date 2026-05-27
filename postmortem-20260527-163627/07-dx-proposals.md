# 07 — DX Proposals (Ordered by Impact)

---

## 1. Make lifecycle re-fireable without `app.id` bump

**The friction it closes:** Every time I changed lifecycle logic (switching instance types, adding
fields, updating geometry IDs), I had to bump `app.id` (krtekstop-v2 → v3 → v4) to get the
lifecycle to re-fire. This is a workaround, not a feature — it forces authors to maintain a
meaningless version counter and creates confusing app-ID history in the project list.

**Sketch:** Add a `--force-lifecycle` flag to `estoflow_build` (or `estoflow_rerun_lifecycle` as a
standalone tool) that clears the "already fired for this app.id" flag for the current project and
re-runs `lifecycle.initialized`. This should be opt-in (not default) to preserve production
idempotency guarantees.

**Why this and not "always re-run on build":** Always re-running would break production idempotency
— lifecycle is supposed to be a one-time seed, not a sync. The `--force-lifecycle` flag preserves
the semantic contract while enabling the development workflow.

---

## 2. Improve render_preview failure message with actionable fix

**The friction it closes:** When the SVG panel logo caused `render_preview: the SVG could not be
decoded as an image`, the entire build reverted. The error message pointed at the SVG as broken when
the actual issue was a rasterizer limitation. I spent ~5 minutes debugging valid SVG before finding
the `autoRevert: false` workaround.

**Sketch:** In the `estoflow_build` response, when render_preview fails, include:
```
render_preview failed: the thumbnail rasterizer does not support inline SVG or complex CSS gradients
in HTML panel bodies. Your manifest was NOT applied (autoRevert defaulted to true).
To apply without thumbnail: set autoRevert: false in build options.
```
Optionally, add a `.estoflow/known-preview-limitations.md` doc page that lists unsupported patterns.

**Why this and not "fix the rasterizer":** Fixing the rasterizer (supporting inline SVG) is
non-trivial and probably not worth the effort. The workaround (`autoRevert: false`) is acceptable;
the problem is discoverability of the workaround. An improved error message is a one-line change.

---

## 3. Add `lifecycle_state` to `estoflow_inspect` output

**The friction it closes:** When lifecycle changes didn't take effect (because the lifecycle had
already fired for the current `app.id`), the only symptom was instances with old data. I had to
infer that lifecycle had already fired; the system gave no signal.

**Sketch:** Add a `lifecycle_state` block to `estoflow_inspect` output:
```json
"lifecycle_state": {
  "initialized": {
    "fired": true,
    "app_id": "krtekstop-v3",
    "fired_at_version": 11,
    "fired_at": "2026-05-26T..."
  }
}
```
This makes the "already fired" state visible in one tool call, eliminating the silent no-op.

**Why this and not a separate `estoflow_lifecycle_status` tool:** Consolidating into `estoflow_inspect`
keeps the tool count low and ensures the information is seen during the normal inspect-after-build
workflow, not only when an author remembers to call a separate tool.

---

## 4. Add `model` filter parameter to `estoflow_inspect`

**The friction it closes:** `estoflow_inspect` returns all instances for all models in a project.
With 6 models and 10+ instances, the dump is large. I repeatedly had to scan through unrelated
instances to find the one model I was checking after a lifecycle run.

**Sketch:** Add optional `model` parameter to `estoflow_inspect`:
```
estoflow_inspect({ model: "MoleProtConfig" })
```
Returns only instances of the specified model. Existing call without `model` returns all (unchanged
behavior for backward compat).

**Why this and not a dedicated per-model inspector:** The existing specialized inspectors
(`estoflow_inspect_calculator`, `estoflow_inspect_panel`) do something slightly different (they
inspect the computed/rendered output, not raw instance data). A `model` filter on `estoflow_inspect`
is the right primitive for "show me instances of this model."

---

## 5. Provide a schema-aware lifecycle body linter

**The friction it closes:** A lifecycle body that called `save({ pointGeometryId: '...' })` on a
model where `pointGeometryId` wasn't yet declared passed `estoflow_validate` and built successfully,
but the field was silently dropped at runtime. I noticed only when checking `estoflow_inspect` output
carefully.

**Sketch:** In `estoflow_validate`, parse all `lifecycle.initialized.body`, `derivation.body`, and
`aggregation.compute` JS strings and check that:
- All field names used in `save({})` calls exist in the corresponding model schema
- All model names accessed via `this.models.<Name>` are declared models

This is static analysis, not runtime execution — it can be approximate (flag unknowns, don't require
perfect resolution). Even a best-effort check would catch the "field not in schema" class of bug.

**Why this and not runtime validation:** Runtime validation would require executing the body, which
has side effects (creating instances). Static analysis at validate-time has no side effects and
catches the error before it silently corrupts data.

---

## 6. Document the `project_id` filter pattern in lifecycle docs

**The friction it closes:** The update-or-create lifecycle pattern requires filtering
`this.models.<Model>.all.value` by `project_id` to avoid operating on instances from other projects.
This boilerplate is non-obvious — `all.value` returns instances for ALL projects, not just the
current one. Every lifecycle author will independently discover this or silently corrupt other
projects' data.

**Sketch:** In `estoflow://docs/lifecycle`, add a dedicated section:
```
## Working with existing instances
this.models.GardenZone.all.value returns ALL instances across ALL projects.
Always filter by project before updating:
  var mine = (this.models.GardenZone.all.value || [])
    .filter(function(r) { return r.project_id === this.project.id; });
```
And add a convenience method: `this.models.<Model>.forCurrentProject()` that does this filtering
automatically.

**Why this and not making `all.value` project-scoped by default:** Changing `all.value` scope would
break existing lifecycles that intentionally read cross-project data (e.g., catalog lookups).
A new `forCurrentProject()` method adds the convenience without breaking existing behavior.

---

## 7. Named geometry aliases for lifecycle seeding

**The friction it closes:** Geometry IDs returned by `estoflow_seed_geometry` are opaque UUIDs
that must be hard-coded into lifecycle bodies. The lifecycle then has comments like
`// seeded via estoflow_seed_geometry on 2026-05-26` to explain where the ID came from.
If the sketch is re-created or the geometry is deleted, the hard-coded ID becomes a dangling
reference silently ignored at runtime.

**Sketch:** Add a `name` parameter to `estoflow_seed_geometry`:
```
estoflow_seed_geometry({ type: "point", x: 10500, y: 2500, name: "config-marker-default" })
```
The name creates a stable alias. In lifecycle:
```js
this.models.MoleProtConfig.save({
  pointGeometryId: this.sketch.geometryByName("config-marker-default")?.id
});
```
If the named geometry doesn't exist, `geometryByName` returns null and the lifecycle degrades
gracefully rather than using a stale UUID.

**Why this and not just documenting the current flow:** The current flow (seed → copy UUID → paste
into code → comment the source) is error-prone for human authors and fragile across sketch edits.
Named aliases create a stable contract between the sketch and the lifecycle.

---

## 8. Add a `fromFile` build shortcut in `estoflow_build` docs

**The friction it closes:** The `{ fromFile: "/abs/path" }` build option was essential for working
with large `app.yaml` files (the manifest grew to 700+ lines). But this option is not prominently
documented — I discovered it from prior session memory, not from the docs. Without it, I would have
had to pass the entire YAML content as a string argument, which is fragile with multi-line strings.

**Sketch:** Add a dedicated "Building from file" section to `estoflow://docs/build`:
```
## Building from a local file
To avoid passing large YAML strings as arguments, use the fromFile option:
  estoflow_build({ fromFile: "/absolute/path/to/app.yaml" })
The daemon reads and parses the file directly. Path must be absolute.
```
This is an existing feature that just needs documentation.

**Why this and not making inline-string build the documented path:** Inline strings break YAML
escaping for large manifests and make the tool call arguments unreadable in agent transcripts.
`fromFile` is clearly the better pattern for any non-trivial workbench.
