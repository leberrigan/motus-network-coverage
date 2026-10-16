# Estimated Motus tag coverage

An interactive [deck.gl](https://deck.gl) map of **modelled** detection coverage for a [Motus](https://motus.org)-tagged animal, on an H3 hex grid. It answers one question per hexagon, per radio band: *how likely is a receiver to be in range right now?*

Three networks can contribute, and up to three radio bands:

| Band | Frequencies | Networks that can contribute |
|---|---|---|
| **Lotek PPM** | 166.38, 150.1, 151.5 MHz | Motus |
| **FSK** | 434 MHz | Motus, Terra |
| **Blu** | 2400 MHz | Motus, Terra, Smartphone |

The whole app is one self-contained file, `index.html`. The Find My hex dataset is inlined, so there is no build step and no backend; the Terra and Motus networks are fetched live on load.

> **This is an estimate, not measured data.** It combines a population-based model (for the Smartphone/Find My network) with the real, current locations of Terra and Motus receiving stations, but assumes a single detection radius per band rather than modelling terrain, antenna gain, tag power or elevation. Use it to compare places, not to predict whether a particular tag will be detected.

## Running it

Open `index.html` in a modern browser that supports WebGL2. It needs an internet connection, because the libraries load from CDNs.

To serve it locally instead:

```bash
python -m http.server 8000
```

Then go to <http://localhost:8000>.

It also works as-is on GitHub Pages: enable Pages for the repo and point it at the root of the default branch.

The file is about 6.9 MB, mostly hexagon data, and takes a few seconds to load.

## Using the map

| Control | What it does |
|---|---|
| **Band** Lotek PPM / FSK / Blu | Selects the radio band to evaluate. Only the networks that actually operate on that band contribute (see the table above); the others' toggle buttons dim but stay clickable. |
| **Networks** Motus / Terra / Smartphone | Independent on/off toggles, usable on any band. |
| **Range** | Assumed detection radius for whichever network the slider currently applies to — 15/30/60 m (Smartphone, Blu band) or 10/20/40 km (Motus, Lotek PPM band). Hidden on FSK: at that band's assumed range, coverage never spreads past the hex containing the station, so the control would have nothing to change (see [Method](#method)). |
| **View** Globe / Map | Switches between deck.gl's `_GlobeView` and a flat Web Mercator `MapView`. |
| **Hover** | Shows the cell's H3 index and a line per active, contributing network. |
| **Download CSV** | Saves the underlying Smartphone/Find My table as `findmy-coverage-h3-res5.csv`. |

All three networks are folded into the same hex grid rather than drawn as separate markers, composited bottom to top as Smartphone (blue ramp) → Terra (yellow) → Motus (green), using standard alpha-over compositing — so a hex with more than one active network shows a blend rather than hiding one behind the other. Each network's own opacity scales with its own measure of confidence: the Smartphone ramp uses modelled coverage probability (as before); Terra and Motus on FSK/Blu use a count of active stations in the cell (opacity capped by 3 stations); Motus on Lotek PPM uses distance to the *nearest* active station as a fraction of the selected range, since that band's radius can span several hex widths. Terra's offline stations aren't shown at all; Motus only counts antennas with a currently-active deployment.

## Data

There are 553,335 [H3](https://h3geo.org) cells at **resolution 5**. Each cell averages about 253 km², with centres about 17 km apart. Each row has two fields:

| field | type | meaning |
|---|---|---|
| `h3` | string | H3 cell index, e.g. `852a100bfffffff` |
| `finders_per_km2` | number | estimated finder devices per km², 3 significant figures |

Cells are included if they reach at least 0.2% coverage at 60 m, which works out to at least 0.177 finders per km².

Inside `index.html`, the table is stored compactly in the `CELLS` string. Every resolution-5 index has the form `85` + 6 hex characters + `fffffff`, so only the 6 middle characters are stored. Rows are separated by `;` and fields by a space:

```js
const data = CELLS.split(";").map(r => {
  const [k, v] = r.split(" ");
  return ["85" + k + "fffffff", +v];
});
```

To use the data elsewhere, click **Download CSV**. It writes full H3 indexes that load directly into deck.gl, kepler.gl, DuckDB (`h3` extension), BigQuery or pandas with `h3-py`.

### Terra stations

A separate, much smaller dataset is fetched at runtime from `https://qa.api.clearlycrickets.io/world/stations-v2` (not inlined, since station status changes). Each entry has:

| field | meaning |
|---|---|
| `lat`, `lon` | station position |
| `status` | 1 = online, 0 = offline |
| `newTerra` | 0 = v1 station, 1 = v2 |
| `lastHeard` | seconds since epoch |

Offline (`status: 0`) stations are discarded entirely. Each remaining station is bucketed into its resolution-5 hex with `h3.latLngToCell(lat, lon, 5)`. `newTerra` isn't currently used in the visuals.

### Motus stations

Three more datasets are fetched at runtime from `motus.b-cdn.net`, Motus's public dashboard feeds:

| endpoint | gives |
|---|---|
| `stations.json` | station `stationID`, `latitude`, `longitude` |
| `stationDeployments.json` | which `sensorID` was deployed at which `stationID`, as `sensorDeployID`, with a `status` code |
| `antennaDeployments.json` | each antenna's `frequency` and the `deploymentID` (→ `sensorDeployID`) it belongs to |

They're joined `antennaDeployments.deploymentID → stationDeployments.sensorDeployID → stationDeployments.stationID → stations`. An antenna only counts if its deployment's `status` is `2`, which — cross-checked against `tsEnd` — is the code that means "currently deployed" (`status: 2` and `tsEnd: null` coincide for over 99% of deployments; `status: 1` deployments almost all have an end date). Each antenna's `frequency` sorts it into a band (166.38/150.1/151.5 MHz → Lotek PPM, 434 MHz → FSK, 2400 MHz → Blu); a station can appear in more than one band if it runs antennas on more than one frequency. As of this writing that's roughly 2,600 active Lotek PPM stations, 1,800 FSK, and 180 Blu.

Lotek PPM's much larger assumed range (tens of km) means a station can plausibly cover hexes well beyond its own, unlike Terra and Motus-FSK/Blu, where the assumed range is smaller than a hex. So instead of a station count, Lotek PPM coverage is a true radius search: `h3.gridDisk` pulls in every hex within the selected range (sized by `AVG_STEP_KM`, the resolution-5 neighbour spacing), `h3.greatCircleDistance` checks the real distance to each one, and a hex's opacity is set by `(1 - nearest_station_distance / range) ** 0.7` — so cells right next to a station are solid, fading out toward the edge of range. This is recomputed (and cached) per selected radius.

## Method

### Smartphone / Find My

1. **Finder devices per country.** Each country's count of active Apple devices with Find My enabled is estimated from its population and an Apple-market-share tier. The totals are then scaled so the world comes to about 1.7 billion devices in 2025. The model is evaluated at October 2026, which gives about 1.93 billion.
2. **Spread over space.** Each country's devices are split across its cities and rural areas. Each city is a Gaussian kernel whose width grows with city size: σ = 2 + 3.5·√(population in millions) km. Rural populations use much wider kernels, with σ from 15 to 180 km.
3. **Onto the H3 grid.** Each kernel is averaged over the area of a resolution-5 cell, then summed onto the cells within 3σ, using `gridDisk` and `greatCircleDistance` from h3-js. Fitting kernels to hexagons adds about 6% to the global total: 2.04 billion versus the model's 1.93 billion.
4. **Density to coverage.** Finders are assumed to be scattered at random (a Poisson process) with density *d* per km². The probability that at least one is within range *r* is

   ```
   P = 1 − exp(−d · π · r²)        (r in km)
   ```

This is the only network with a real density model behind it; Terra and Motus are actual station locations; opacity there is either a station count (FSK, Blu) or distance to the nearest station (Lotek PPM) rather than a modelled probability (see [Data](#data) above).

### Example values (Smartphone network, rough, default range)

| place | coverage |
|---|---|
| Rural Iowa | ~6% |
| New Jersey suburbs | ~29% |
| Lagos | ~60% |
| Boston | ~98% |
| Manhattan, central London | >99% |

### Limitations

- Device counts and market shares are rough estimates, not Apple figures.
- The model assumes finders are spread evenly at random within each kernel. In reality they gather along roads, in buildings and at certain times of day, and the map shows a single averaged moment.
- It ignores whether a finder is awake, has Bluetooth on, or has a network connection to report a sighting.
- At resolution 5 a dense downtown core is blended with its suburbs. Small towns appear as soft blobs rather than sharp points.
- The basemap is Natural Earth at 1:110m, so coastlines are coarse when zoomed in.
- The 10/20/40 km Lotek PPM and the fixed FSK/Blu station ranges are a single assumed figure per band, not a per-station model — real detection range varies hugely with antenna height, terrain, tag power and gain.
- Motus and Terra reflect whatever is currently deployed and active; the Smartphone layer is a static model evaluated at a point in time. The three aren't on the same footing, which is part of why each is independently toggleable.

## Implementation notes

- **Layers**, bottom to top:
  - an ocean `SolidPolygonLayer`;
  - a land `SolidPolygonLayer`;
  - the `H3HexagonLayer`, with `updateTriggers: { getFillColor: [band, RANGE_M, MOTUS_RADIUS_KM, networksOn.motus, networksOn.terra, networksOn.smartphone, terraByHex.size, motusVersion] }`. `getFillColor` composites whichever networks are active and relevant to the selected band, bottom to top: Smartphone ramp, then Terra, then Motus, using standard alpha-over compositing (`over()`), so overlapping networks blend rather than one hiding another;
  - `PathLayer`s for borders and coastlines.
- **Antimeridian:** in the flat map, shapes that cross ±180° (Russia, Fiji) would draw streaks right across the map. The fix is to break coastline and border paths wherever a segment jumps more than 180° of longitude. Land rings are also unwrapped into continuous longitudes, except for Antarctica, which circles the whole globe.
- **Hexagon geometry:** cells are rebuilt from their indexes in the browser. Expect a short delay on first load while deck.gl builds the polygons for about 550k cells.

## Dependencies

All are loaded from CDNs at pinned versions:

| library | version | source | licence |
|---|---|---|---|
| deck.gl | 9.3.11 | unpkg | MIT |
| h3-js | 4.4.0 | unpkg | Apache-2.0 |
| topojson-client | 3.0.2 | cdnjs | ISC |
| world-atlas `countries-110m` | inlined | Natural Earth via world-atlas | public domain / ISC |
| Archivo, IBM Plex Mono | — | Google Fonts | OFL |

## Disclaimer

This project is not affiliated with or endorsed by Apple Inc. AirTag and Find My are trademarks of Apple Inc.
