# MapLibre terrain: basemap labels bleed over an overlying raster

A minimal, self-contained reproduction (no build step, no API keys) of a MapLibre
GL JS terrain rendering issue: when a `raster-dem` terrain is enabled, the symbol
(label) layers of a vector basemap render **on top of** an opaque raster layer
that sits **above** them in the style, instead of being hidden by it.

**Live demo:** https://giswqs.github.io/maplibre-terrain-label-bleed/

## What it sets up

- Vector basemap with labels: **OpenFreeMap Liberty** (`https://tiles.openfreemap.org/styles/liberty`)
- An opaque raster layer added **above every basemap layer**: **Esri Wayback imagery, 2014-02-20** (release `10`)
- **`raster-dem` terrain** enabled (AWS Terrarium elevation tiles)
- View over the White House, Washington DC, at ~zoom 17

The imagery layer is appended with no `beforeId`, so it sits at the very top of
the style, above every basemap fill, line, and symbol layer.

## Expected vs actual

- **Expected:** an opaque raster on top of the basemap fully hides the basemap,
  including its labels (this is exactly what happens with terrain **off**).
- **Actual (terrain on, high zoom):** the basemap's symbol labels
  ("White House", "The West Wing", "West Colonnade", ...) show through on top of
  the imagery.

Use the **Toggle terrain** button to compare. With terrain off the labels are
correctly covered; with terrain on they bleed through. The effect only appears
at high zoom (roughly z16 and above).

## Why it happens

MapLibre's terrain renderer drapes `background` / `fill` / `line` / `raster` /
`hillshade` / `color-relief` layers onto the terrain mesh (render-to-texture),
but **symbol layers are never draped** - they are drawn live and occluded only by
the terrain depth buffer. At high zoom the live symbol fragments win the depth
test against the draped raster (which is composited at the terrain surface), so
the labels survive on top of a raster that is above them in the style order. In
2D (no terrain) the raster simply paints over the labels, so they are hidden.

This is related to the known behavior where basemap labels lift off the terrain
surface at high zoom (maplibre/maplibre-gl-js#4908).

## Run locally

It is a single static file. Serve the folder with any static server:

```bash
python3 -m http.server 8000
# then open http://localhost:8000/
```

(Opening `index.html` directly via `file://` also works.)

## Environment

- MapLibre GL JS 5.24.0
- Reproduces in Chromium, and originally reported on Safari 26 / macOS

## Context

Originally surfaced as [opengeos/GeoLibre#781](https://github.com/opengeos/GeoLibre/issues/781).
GeoLibre's layer ordering is correct (the raster is placed above the basemap
symbols); the bleed comes from the MapLibre terrain rendering described above.
