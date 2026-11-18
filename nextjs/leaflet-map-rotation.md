# Turning a Leaflet map like Google Maps (react-leaflet, Next.js)

Letshego, 16 Nov 2026. The ask: "in Google Maps you can rotate the map and position it to how you are seeing the
world". Leaflet 1.9 cannot turn a map; [leaflet-rotate](https://github.com/Raruto/leaflet-rotate) adds it
(0.6.2 at the time, peer `leaflet ^1.9.3 || ^2.0.0-beta.3`).

## What people get

- Shift + scroll, or Ctrl + drag / right-button drag (that one also tilts), on a computer.
- A two-finger twist on a phone (`touchRotate`, off by default).
- A compass control (`rotateControl`): shows north, a click sets it back up. Keep it always shown
  (`closeOnZeroBearing: false`) or nobody finds the feature.
- Pins and popups stay upright. **The map's own labels turn with it**: raster tiles (OpenStreetMap, Google's tile
  URLs) are pictures. Google's own app keeps labels upright because it draws vector maps. Say so before building.

## The trap: it patches `globalThis.L`, not your import

The bundle starts with `var v = globalThis.L`. With a bundler, `import L from "leaflet"` sets no global, so the plugin
finds nothing. Imports run in order and before any statement, so the global goes in a module of its own:

```ts
// leafletGlobal.ts
import L from "leaflet";
(globalThis as { L?: typeof L }).L = L;

// rotate.ts
import "./leafletGlobal";      // first
import "leaflet-rotate";       // then the plugin sees our Leaflet
import type { MapOptions } from "leaflet";
export const ROTATE: MapOptions = { rotate: true, touchRotate: true, shiftKeyRotate: true, rotateControl: { closeOnZeroBearing: false } };
```

Then `<MapContainer {...ROTATE} ...>` (react-leaflet passes unknown props to `L.Map` as options). Only in client
components (the map files are already `"use client"` and loaded without SSR).

## Types

0.6 ships none, and `@types/leaflet-rotate` describes 0.2. Declare what you use:

```ts
// types/leaflet-rotate.d.ts
import "leaflet";
declare module "leaflet" {
  interface MapOptions { rotate?: boolean; bearing?: number; touchRotate?: boolean; shiftKeyRotate?: boolean;
    rotateControl?: boolean | { closeOnZeroBearing?: boolean; position?: ControlPosition }; }
  interface Map { getBearing(): number; setBearing(degrees: number): this; resetNorth(options?: { animate?: boolean }): this; }
}
declare module "leaflet-rotate";
```

## Keep north up where it matters

Leave the options off print maps (a printed map is read north up). The patch is global but `rotate` defaults to false,
so other maps are untouched.

## Proving it (Playwright)

The turned pane is `.leaflet-rotate-pane`; its computed `transform` is the identity at north.

```ts
const turn = () => map.locator(".leaflet-rotate-pane").evaluate((el) => getComputedStyle(el).transform);
await page.mouse.move(cx, cy); await page.keyboard.down("Shift");
for (let i = 0; i < 6; i++) await page.mouse.wheel(0, 120);
await page.keyboard.up("Shift");
await expect.poll(turn).not.toMatch(/^(none|matrix\(1, 0, 0, 1,)/);   // 30°: matrix(0.866, 0.5, ...)
await map.locator(".leaflet-control-rotate").click();
await expect.poll(turn).toMatch(/^(none|matrix\(1, 0, 0, 1,)/);
```

A screenshot right after the turn shows grey: the extra tiles a turned view needs (6 became 12) were still loading.
Count `img.leaflet-tile-loaded` before deciding the plugin broke the tiles. Wait for none pending instead
(`expect.poll(() => count("img.leaflet-tile:not(.leaflet-tile-loaded)")).toBe(0)`, then loaded > 0): NICE's first
light picture was grey for exactly this reason (17/11/2026).

Three more traps from NICE (17/11/2026, seven maps):

- **A wheel turns only the map under the pointer.** A map below a table (NICE's reach maps) is off screen, so the
  pointer at its box's centre is outside the window and nothing turns: `scrollIntoViewIfNeeded()` before measuring.
- **A map that draws only with data** (a visits map on a day with no visits renders "No data" and no
  `.leaflet-container`): point its request at a day with data (`page.route(..., r => r.continue({ url }))` with the
  date param changed) rather than drive the whole filter form.
- **A picker's map behind a button** (NICE's `MapLocationPicker`: "Open Map" opens it in its own dialog): click it
  first. Check every map the options were added to, not only the pages called "map".
