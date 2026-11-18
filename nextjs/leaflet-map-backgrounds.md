# Leaflet map backgrounds: one set on every map

The set Letshego settled (its `lib/map.ts`, `docs/maps.md`) and NICE took on 18/11/2026 (`lib/mapLayers.ts`,
`components/admin-panel/map/MapBaseLayers.tsx`): a list of backgrounds, one component that draws the layers button,
dropped into every `<MapContainer>` in place of the map's own `TileLayer`s.

| Name | Source | Notes |
| --- | --- | --- |
| OpenStreetMap (light default) | `https://tile.openstreetmap.org/{z}/{x}/{y}.png` | what people here are used to |
| Google streets | `https://mt{s}.google.com/vt/lyrs=m&x={x}&y={y}&z={z}`, subdomains `0123` | |
| Satellite | `lyrs=s` | photo, no names |
| Satellite with labels | `lyrs=y` | **not `lyrs=h`**: that is the labels ONLY |
| Terrain | `lyrs=p` | |
| Dark (dark default) | Esri `Canvas/World_Dark_Gray_Base` + `World_Dark_Gray_Reference` (labels) | native to zoom 16, enlarged beyond |
| Grey | Esri `Canvas/World_Light_Gray_Base` + `_Reference` | printing |

- **The theme picks until the person does**; the pick is kept in `localStorage` (a convenience: try/catch, a private
  window forgets). Key the `LayersControl` by `chosen ? "chosen" : theme` so switching the theme with nothing chosen
  swaps the background.
- Each entry is a `LayerGroup` (background + its labels) inside `LayersControl.BaseLayer`; the group's `add` event
  records the pick.
- **No CSS filter for dark mode**: it turns a satellite photo into a negative.
- **Not CARTO's dark map**: since 2026 its tiles carry "API KEY REQUIRED" without a key, and the free key is
  non-commercial.
- Google's direct tile URLs are what both apps use; Google's terms expect the keyed Map Tiles API (billed). Moving
  there later changes only the URLs.

## Traps found while moving NICE over

- **A `TileLayer` outside the `LayersControl`** (NICE had one on 6 of 7 maps) is drawn under every background:
  OpenStreetMap kept downloading under the satellite, and NICE's "Hybrid View" (`lyrs=h`, labels only) looked like a
  street map because the street map was underneath. Remove it: the control draws the background.
- **Waiting for tiles in a test**: count `<img class="leaflet-tile">` that are not `complete`, then require some
  `.leaflet-tile-loaded`. Leaflet only adds the loaded class on success, so a tile that failed (a passing
  `ERR_NAME_NOT_RESOLVED` on `mt2.google.com`) stays "pending" for ever and the wait times out. The trace tells which:
  read `trace.zip`'s `*.network` entries for tile URLs whose status is not 200 (-1 with no `_failureText` is a request
  cancelled by a layer switch or a reload, not a failure).
- A screenshot right after switching shows grey squares: tiles still on their way. Wait as above first.

Related: [leaflet-map-rotation.md](leaflet-map-rotation.md) (the turned map loads more tiles: same wait).
