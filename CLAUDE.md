# CLAUDE.md

Guidance for AI coding assistants (Claude Code, Claude Desktop, and similar)
working in this repository. Humans are welcome to read it too. It is written
for people who may never have run a web project before, so it errs on the
side of spelling things out.

## What this is

pot toy is a single-page browser app for designing pottery forms for clay 3D
printing. The user drags control points on a profile curve, the app revolves
the curve into a pot and shows it in 3D, and it can export the pot as a
continuous spiral G-code toolpath. There is **no build step, no package
manager, and no framework**: everything is in `index.html`, which contains
the page, its styles, and one `<script type="module">` with all the logic.

## Running it (do this first, before changing anything)

1. Serve the repository directory over HTTP. Opening `index.html` directly
   from the filesystem will **not** work, because ES modules and the import
   map require an HTTP origin. Any static server is fine:
   ```
   python3 -m http.server 8000
   ```
   or, if Node is installed, `npx serve .`
2. Open http://localhost:8000 in Chrome, Firefox or Safari.
3. You should see a grey pot on a light background with a control panel at
   the top right and red spheres you can drag. If the page is blank, open the
   browser's developer console (F12 or Cmd-Option-I) and read the error. The
   usual causes are opening the file without a server, or a typo that broke
   the module script. A syntax error anywhere in the script stops the whole
   page from running.
4. After every edit, reload the page. There is no hot reload. If a change
   does not seem to appear, do a hard reload (Shift-Reload) to bypass cache.

To check the script parses without opening a browser:
```
python3 -c "import re;s=open('index.html').read();open('/tmp/p.mjs','w').write(re.search(r'<script type=\"module\">(.*?)</script>',s,re.S).group(1))" && node --check /tmp/p.mjs
```

## How the app works

Everything below refers to functions inside the module script in
`index.html`. Search for `function <name>(` to find them.

### Data flow

```
positions[]                 control points (THREE.Vector3; x = radius, y = height, z unused)
   |  red spheres (splineHelperObjects) are draggable handles bound to these
   v
splines.centripetal         CatmullRomCurve3 through the control points = the profile
   |  surface.spline points at this
   v
sor(spline, squash)         returns fn(u, v, pt): the pot surface as a parametric function
facet(fn, n, twist, puff)   optional wrapper that makes the cross-section polygonal
   |
   +--> updateSplineOutline()  builds the 3D mesh with ParametricGeometry(fn, 40, 90)
   |                           and, if params.bottom, a fan-triangle bottom disk
   |
   +--> spiralPath(sf)         samples the same fn along a spiral for the toolpath
           |
           v
        exportGcode()          scales to mm, swaps axes for the printer, writes G-code
```

`u` runs 0..1 along the profile from the first control point (bottom) to the
last (top). `v` runs 0..1 around the pot. `fn` writes the 3D point into `pt`.
Anything that should change the pot's **shape** belongs in this parametric
function or in how the spline is built, so that the preview mesh and the
toolpath stay in agreement. Anything that should affect only the **printed
path** (texture, layer structure, feed) belongs in `spiralPath()` or
`bottomPath()`.

### Key functions

| Function | Role |
|---|---|
| `params` (object near the top) | Every user-adjustable setting, with its default. The GUI edits these fields directly. |
| `sor()` | Surface of revolution: profile point at `u`, rotated by `v`; `squash` flattens z. |
| `facet()` | Wraps a surface function to give it `n_facets` flat sides, with `twist` and `puff`. |
| `bbox()` | Bounding box of the current pot in scene units. |
| `scale_factor()` | Millimeters per scene unit for export. Default fits the pot in a `max_height` x `max_width` box; `set_height` overrides with `target_height`. |
| `spiralPath()` | The wall toolpath: one turn per layer, rising continuously. Applies the texture. Prepends `bottomPath()` when `params.bottom`. |
| `bottomPath()` | Three bottom layers: inward spiral, `bottomRaster()` serpentine fill, outward spiral. |
| `exportGcode()` | Converts the path to mm, moves it to the bed center, computes extrusion, adds the printer header/footer and `;LAYER_CHANGE` markers, downloads the file. |
| `exportToObj()`, `exportJson()` | Other exports. `model_json()` is the settings record that also goes into the G-code header as a comment. |
| `init()` | Builds the scene, lights, camera, the lil-gui panel, the control-point handles, and loads the default shape via `load([...])`. |
| `updateSplineOutline()` | Rebuilds the mesh (and toolpath preview if `spiralize`) from the current control points and params. Call it after any change. |
| `render()` | Draws one frame. Rendering is on demand, not continuous, so call it after changes. |
| `addPoint()`, `removePoint()`, `load()` | Manage the control-point list. |
| `enforceFirstPointBottom()` | Keeps every point at or above the first point, which is the bottom. |

### Units and coordinates

- **Scene units** are arbitrary. The default pot is about 700 units tall.
  The camera and lights are placed for that scale.
- **Millimeters** appear only in `params` printer settings (`layer_height`,
  `line_width`, `bed_height`, `texture_amplitude`, `target_height`) and in
  `exportGcode()`. Convert mm to scene units by dividing by the scale
  factor, e.g. `params.layer_height / scale_fact`. `spiralPath()` and
  `bottomPath()` work in scene units and receive `scale_fact` for this.
- **Scene axes**: y is up. In the exported G-code, scene y becomes printer
  Z, scene z becomes printer -Y, and the pot is centered at X100 Y100.
- The first control point is the center of the bottom; its x is the radius
  where the wall starts. The last control point is the rim.

## How to add a new option (the standard recipe)

Every existing option follows the same four steps. A good template to copy
is the "Set height manually" feature: search for `set_height` to see all
four pieces.

1. **Add a field to `params`** with a sensible default. If the feature is
   optional, add a boolean to turn it on and a number for its value, e.g.
   ```js
   set_height: false,
   target_height: 100.0,
   ```
   Defaults must reproduce the current behavior, so the boolean defaults to
   `false`.
2. **Add controls to the GUI** in `init()`. Pick the folder that fits
   (top level for shape, `Texture`, `Export`, `(Expert) Printer Settings`)
   or create one with `gui.addFolder('Name')`. lil-gui infers the control
   type from the field: booleans become checkboxes, numbers with a range
   become sliders, functions become buttons.
   ```js
   prs.add( params, 'set_height' ).name( 'Set height manually' ).onChange( scaleChanged );
   prs.add( params, 'target_height', 10.0, 250.0 ).name( 'Height (mm)' ).step( 1.0 ).onChange( scaleChanged );
   ```
   The `onChange` handler should call `updateSplineOutline()` then
   `render()`. If the option affects only the toolpath, only do so when
   `params.spiralize` is on, as the existing handlers do.
3. **Use the field** where it belongs: `sor()`/`facet()` or the spline for
   shape, `scale_factor()` for size, `spiralPath()` for the printed path.
4. **Record it in `model_json()`** so it appears in the JSON export and the
   G-code header. One line: `ppp.my_option = params.my_option;`

Then reload the page, check the console for errors, try the control, and
toggle "Show spiral toolpath" to confirm the toolpath agrees with the mesh.
If the change affects G-code, export a file and inspect it (it is plain
text) or open it in a slicer's preview such as PrusaSlicer.

### Worked example: a shape option

Suppose the option is "flare the rim": scale the radius near the top. Shape
changes go in the parametric function, so wrap `fn` the same way `facet`
does. In `updateSplineOutline()`, `bbox()` and `spiralPath()` the surface
function is built with the same few lines; a helper that builds it in one
place, then adds the new wrapper when the option is on, keeps all three in
agreement. The wrapper reads the base point, adjusts `pt.x` and `pt.z` by a
factor that depends on `u`, and leaves `pt.y` alone. Radius is
`Math.hypot(pt.x, pt.z)` and the direction is `(pt.x, pt.z)` normalized.

## Things that trip people up

- The whole app is one `<script type="module">`. Nothing is on `window`, so
  you cannot poke at `params` from the browser console unless you add
  `window.params = params` temporarily for debugging.
- Rendering is on demand. Forgetting `render()` after a change makes it look
  like nothing happened.
- The GUI is built once in `init()`. Changing a `params` value in code does
  not move the slider; use lil-gui's `controller.updateDisplay()` if you
  need that.
- The texture feature is applied only in `spiralPath()`, not on the mesh, on
  purpose: a fractional period count cannot be shown on a single-turn
  surface. Do not "fix" that.
- The G-code header and footer in `exportGcode()` are for one specific clay
  printer. Do not change them unless asked.
- The code style is old and mixed (`var`/`let`/`const`, commented-out
  experiments). Match the surrounding style with tab indentation; do not
  reformat or clean up unrelated code in a change.
- There is no test suite. Verification is: page loads with no console
  errors, the pot looks right, the toolpath preview looks right, and if
  relevant the exported G-code looks right.
- There is no JSON import. The default shape is the hard-coded `load([...])`
  call at the end of `init()`. To make a different default, export JSON and
  paste the `points` values there.

## Dependencies

`vendor/` holds three.js r141 and lil-gui 0.16, copied unmodified; see
`vendor/README.md`. The import map in `index.html` maps the bare specifier
`three` to `vendor/three/three.module.js`. Do not add build tooling,
frameworks, or CDN links; the point of the repo is that it runs from a plain
static server with no network access.

## Committing

Commit messages are short imperative lines like the existing history. Keep
each change focused. If you add an option, the commit touches `params`, the
GUI, the logic, and `model_json()`, and nothing unrelated.
