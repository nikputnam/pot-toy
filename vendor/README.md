# Third-party code

Nothing in this directory is written by the pot toy authors. Files are copied
unmodified from their upstream releases so the app runs with no build step and
no network access.

| Path | Project | Version | License |
|------|---------|---------|---------|
| `three/three.module.js` | [three.js](https://threejs.org) | r141 (0.141.0) | MIT, see `three/LICENSE` |
| `three/controls/*`, `three/geometries/*`, `three/exporters/*`, `three/loaders/*` | three.js `examples/jsm` | r141 | MIT, see `three/LICENSE` |
| `lil-gui/lil-gui.module.min.js` | [lil-gui](https://lil-gui.georgealways.com) | 0.16.0 | MIT (header in file) |

To upgrade three.js, replace these files with the same paths from a newer
release and update the version here. The example modules import from the bare
specifier `three`, which the import map in `index.html` resolves to
`vendor/three/three.module.js`.
