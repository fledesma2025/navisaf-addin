# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **Geotab Add-In** (fleet telematics platform) called "Analizador de Viajes" (Trip Analyzer) for Navisaf's `fleet_colombia` account. It is a **single-file application** — the entire add-in lives in `analizador-viajes.html` (HTML + CSS + JavaScript, ~1600 lines, no build system).

There is no `package.json`, no bundler, no test suite, and no build step. Development means editing the HTML file directly.

## Deployment context

This file runs inside the Geotab web platform as an embedded Add-In. Geotab injects a `geotab` global and loads the file in an iframe. The entry point is the registration at line 280:

```js
geotab.addin['transito-geocercas'] = function() {
  return {
    initialize: function(api, state, cb) { ... },
    focus: function() {},
    blur: function() {}
  };
};
```

The `api` object (stored as `geotabApi`) is the only way to fetch data — it exposes `geotabApi.call(typeName, search, successCb, errorCb)` against the Geotab database. Key types used:
- `Zone` — geofences (geocercas); has `.id`, `.name`, `.points[]` (polygon with `.x`/`.y` for lng/lat), and `.radius` (for circular zones)
- `Device` — vehicles; has `.id`, `.name`
- `LogRecord` — GPS position logs; has `.device.id`, `.dateTime`, `.latitude`, `.longitude`, `.speed` (in km/h)

## Core data flow

1. **Init** (`cargarDatosIniciales`): fetches all Zones (up to 5000) and Devices (up to 2000); populates the filter dropdowns.

2. **Query** (on "Consultar" click): fetches `LogRecord` for the date range (up to 100,000 records), optionally filtered by device. Groups records by vehicle, sorts each vehicle's points by `dateTime`.

3. **Trip detection** (`detectarViajes`): per-vehicle state machine (`buscando_salida` → `buscando_llegada`). A trip starts when a vehicle leaves the origin zone and ends when it enters the destination zone. Uses:
   - Circular zones: `haversineM` distance ≤ `zona.radius`
   - Polygon zones: ray-casting (`raycast`) — **note**: zone points use `.x` for longitude and `.y` for latitude (Geotab convention, opposite of the usual lat/lng order).

4. **Metrics** (`calcMetricas`): computes per-trip distance (Haversine sum), max/avg speed, duration. Status classification:
   - `velMaxKmh > 100` → `'velocidad'`
   - `duracionMin > 600` (>10 h) → `'demorado'`
   - otherwise → `'a tiempo'`

5. **Map** (`dibujarMapa`): Leaflet.js loaded dynamically from unpkg CDN on first use. Uses `window._lmap` as the singleton map instance and `window._lmap_layers` for the current trip's layers. Track color encodes status: red (speed), orange (delayed), dark blue (normal). Speed-violation points (>100 km/h) get individual red circle markers.

6. **Optimal route** (`calcularRutaOptima`): calls the public OSRM demo API (`router.project-osrm.org`) with the trip's first and last GPS point. Result is cached on `f.rutaOptima` and drawn as a green dashed polyline.

7. **Export**: CSV with UTF-8 BOM (for Excel) generated client-side via `Blob` + `URL.createObjectURL`.

## Critical structural quirk

The file contains **multiple stacked overrides** of the same functions (`mostrarLayout`, `setCargando`, `renderTabla`, `renderKPIs`, `seleccionarViaje`, `dibujarMapa`, `mostrarError`, `dbg`, etc.). These are stacked from line ~727 onward in four successive blocks. **The last definition wins.** The canonical, active implementations are the final ones (roughly lines 1303–1599). When modifying any of these functions, edit only the last occurrence and consider consolidating or removing the dead earlier versions.

## Brand/style constants

- Primary navy: `#0d3c6e`
- Accent teal: `#00a884`
- Speed-violation red: `#d62c2c`
- Delayed orange: `#d68a00`

## How to test locally

Open `analizador-viajes.html` directly in a browser to inspect HTML/CSS. The JavaScript will not function outside Geotab (the `geotabApi` object is not present), but you can stub it for UI development:

```js
// Minimal stub for local development
window.geotab = { addin: {} };
// Then call initialize manually after defining a fake api object
```

To test with real data, the file must be loaded through the Geotab Add-In system (either the Geotab Marketplace or a custom Add-In configuration pointing to the hosted file URL).
