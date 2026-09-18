# pot toy

An interactive tool for designing pottery forms for clay 3D printing, in the
browser. Drag control points on a profile curve, revolve it into a pot, add
facets, twist, squash or a sine-wave surface texture, and export a spiral
G-code toolpath.

Live version: http://www.randomvariables.studio

## Run it

There is no build step. Clone the repo and serve the directory with any
static file server. With Python 3:

```
git clone https://github.com/nikputnam/pot-toy.git
cd pot-toy
python3 -m http.server
```

Then open http://localhost:8000 in a browser. A server is needed because the
page uses ES modules; opening `index.html` directly from the filesystem will
not work. Any recent Chrome, Firefox or Safari is fine.

## Using it

- **Control points** are the red spheres. Drag them to change the profile.
  `addPoint` and `removePoint` in the panel add or remove one at the top.
- **facet / n_facets / twist / puff / squash** change the cross-section.
- **Texture** adds a radial sine wave to the toolpath: amplitude in
  millimeters, whole periods per turn, and a fractional period that shifts
  the phase between successive turns of the spiral. It shows only in the
  toolpath preview and the G-code, not on the shaded surface.
- **Show spiral toolpath** draws the path the printer will follow.
- **Export** writes OBJ, JSON (a record of the control points and settings;
  the same record is written as a comment at the top of the G-code), or
  G-code.
- **(Expert) Printer Settings** sets bed height, layer height, line width,
  and optionally a fixed printed height instead of the default autoscale.

## G-code

The exported G-code is a single continuous spiral: the bottom is printed as
an inward spiral, a serpentine raster fill and an outward spiral, then the
wall rises one layer height per turn with no seams. By default the model is
scaled to fit within a 100 mm box. The header and footer at the bottom of
`exportGcode()` in `index.html` are written for a specific clay printer;
check them against your machine before printing. `;LAYER_CHANGE` markers are
emitted so slicers such as PrusaSlicer can preview the file by layer.

## Modifying it, with or without an AI assistant

The whole app is one file, `index.html`, and every option follows the same
small pattern, so it is a friendly place to try changes. `CLAUDE.md` in the
repo root is a guide written for AI coding assistants such as Claude Code
and Claude Desktop, and it is just as useful for people: it explains how the
code is organized, the units, the recipe for adding a new slider or option,
and how to check that a change worked.

If you are asking an assistant to make a change for you, a prompt like this
works well:

> Help me run this project locally, then modify it: add an optional mode,
> off by default, where a slider sets the diameter of the top opening of the
> pot. Read CLAUDE.md first. Show me the running app before and after.
>
> https://github.com/nikputnam/pot-toy

Tips that make this go smoothly:

- Fork the repo on GitHub first, so the assistant can clone your copy and
  you can keep your changes.
- Ask it to run the app and show you it working before it changes anything.
- Be specific about defaults. "Off by default" or "should not change the
  current behavior unless the checkbox is on" is what keeps existing
  projects printing the same.
- Ask it to turn on "Show spiral toolpath" after the change, so you can see
  the printer path agrees with the pot.
- If the page goes blank, ask it to open the browser console and read the
  error. A single typo in `index.html` stops the whole page.
- The app needs a local web server (the assistant will start one). Opening
  `index.html` directly from a folder does not work.

## Layout

```
index.html    the whole app: page, styles, and the script
main.css      shared page styles
CLAUDE.md     guide to the code for AI assistants and humans
textures/     HDR environment map for the preview lighting
vendor/       three.js and lil-gui, copied unmodified; see vendor/README.md
```

## Credits

- Built on [three.js](https://threejs.org) and
  [lil-gui](https://lil-gui.georgealways.com), both MIT licensed.
- `textures/venice_sunset_1k.hdr` is from [Poly Haven](https://polyhaven.com),
  CC0, distributed with the three.js examples.
- Pottery by [@nik.ceramics](https://www.instagram.com/nik.ceramics/).

## License

MIT, see `LICENSE`. Third-party licenses are in `vendor/`.
