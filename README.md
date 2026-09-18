# pot toy

An interactive tool for designing pottery forms for clay 3D printing, in the
browser. Drag control points on a profile curve, revolve it into a pot, add
facets, twist, squash or a sine-wave surface texture, and export a spiral
G-code toolpath.

Live version: https://www.randomvariables.studio

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
- **Export** writes OBJ, JSON (the control points and settings, loadable back
  into the app), or G-code.
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

## Layout

```
index.html    the whole app: page, styles, and the script
main.css      shared page styles
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
