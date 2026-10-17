# Experimental read-only viewer

`cadcraft-viewer` is a separate native frontend for opening and reviewing DWG,
DXF and DWT drawings. It reuses the existing `io`, `doc`, `fonts` and `render`
crates. It does **not** depend on the editing engine, constraint solver or the
editor UI. The full CADCraft desktop app is unchanged.

```sh
cargo run --release -p cadcraft-viewer -- drawing.dwg
```

The interface follows the system language (English or Simplified Chinese). Use
`--lang en` or `--lang zh-CN` to choose it for one launch; the toolbar can switch
languages while running. A Chinese interface requires a suitable installed font.
On Windows the viewer tries the system YaHei, JhengHei and SimHei fonts; it never
ships copies of those fonts.

## Reviewing drawings

- Open (`Ctrl+O`) or drop a drawing into the window. One drawing per window.
- Zoom with the wheel, drag with the left/middle mouse button to pan, and
  double-click or press `Home` to fit the drawing.
- Show the Layers panel to filter names and switch visibility in memory.
- Choose Model or a layout tab. Switching spaces fits the new space.
- Measure two freely picked points. Results use drawing units, without object
  snaps; this is a visual review tool, not precision drafting.
- Export the active space to a new PDF. Existing output files are deliberately
  refused rather than truncated, including hard links to a source drawing.

The viewer provides no drawing save, entity editing, command execution or
autosave. Layer visibility, active space and camera changes are transient.
Read notes and unsupported/proxy-object warnings remain visible. Source drawings
are opened for reading only.

## Performance and limits

Parsing, display-list builds, CPU rasterization and PDF output run on a worker
thread. A cached display list is reused for pan/zoom; curves refine when their
cached chord error exceeds half a raster pixel. While a new raster is pending,
the previous image follows the camera immediately. Only one raster request is
in flight, and the UI retains the latest requested view. Layer changes are
batched. Superseded operations use per-request cancellation tokens; DWG parsing
itself is not yet interruptible. Explicit PDF exports keep their queue position.

Raster dimensions are bounded to 4096 pixels per side with a uniform density;
larger/high-DPI windows still show the whole viewport, with lower sampling
density. This first frontend uses the existing CPU rasterizer, not the editor's
GPU geometry backend. Do not infer a speed improvement for every drawing.

This is a **2D preview**, with the same reader/font/entity limitations as the
current CADCraft core. SHX/big fonts, custom objects and 3D appearance need
independent verification. It does not claim AutoCAD or DWG FastView parity.
File inputs are limited to 256 MiB. Rust panics from processing are caught, but
process aborts and allocation exhaustion are not isolated.

## Validation

```sh
cargo test -p cadcraft-viewer
cargo clippy -p cadcraft-viewer --all-targets -- -D warnings
cargo build --release -p cadcraft-viewer
```

Tests generate fresh drawings in temporary directories. They cover cursor-fixed
zoom at large world coordinates, high-DPI coverage, pan, layer visibility and
layout bounds, malformed/cancelled requests, preserving input bytes, output
hard links, and preserving an explicit PDF export while another drawing opens.

For local diagnostics (use a **new** output directory each time):

```sh
cadcraft-viewer --probe drawing.dxf new-output-directory
cadcraft-viewer --capture new-window.png --lang en drawing.dxf
```

`--probe` writes a drawing PNG, PDF and JSON timings through the same reader and
rasterizer as the GUI. It is not a GUI startup benchmark or an independent
rendering oracle. `--capture` uses egui's screenshot event and closes its test
window afterward. If no native frame is presented, an error sidecar is written
without replacing existing files. All diagnostic outputs refuse replacement.

The native app has been developed and tested on Windows; other supported desktop
platforms require their own runtime acceptance. No additional proprietary assets,
fonts or binary fixtures are included.
