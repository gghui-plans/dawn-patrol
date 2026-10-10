# Dawn Patrol

A phone-first web app showing a realistic sky over open water for any moment of the day. It covers the sun's and moon's real positions, the twilight phases, and how hard the wind is blowing, and rates each day **Calm**, **Manageable** or **Rough** with its calmest stretch of daylight.

It is a personal project. The owner opens it from a home-screen icon to check the light and wind before heading out. Keep the language generic: no references to any particular sport or activity.

## What matters most (in priority order)

1. **The visuals.** The sky and water should look as close to a photograph as a real-time render can get.
2. **The key datapoints.** Show these accurately and at a glance:
   - every twilight phase
   - sun altitude
   - sustained wind, gusts and direction
3. **The verdict.** It must be honest about its data. Never present estimated data as live.

## Architecture

- **One file, no build step.** `index.html` holds all HTML, CSS and JavaScript. Keep it that way unless a change clearly needs more. It is opened directly in a browser and hosted as static files.
- **No frameworks or npm dependencies.** Google Fonts (Archivo, JetBrains Mono) and the Esri map tiles are the only external loads, apart from the Open-Meteo data.
- **Map view** (the Sky | Map toggle at the top of the scene) is a small hand-written tile map of Esri's World Dark Gray Canvas: a base layer plus a label layer, combined per tile. There's no library.
  - **Gestures:** one finger pans and two fingers pinch-zoom. On desktop, ctrl+scroll (or trackpad pinch) and double-click zoom.
  - **Controls:** a Recentre button appears once you've moved away. The time strip works in both views.
  - **Choosing a location:** a pin is fixed at the map centre and the map moves under it, which is easier on a phone than dragging a small pin under a finger. The picking bar shows live coordinates, an optional name ("Pinned spot" if left blank), and Cancel / Use this spot, which saves it like typed coordinates.
    - Entry points: the map's "Set location" button, tapping the location chip, or "Pick on map" in settings.
    - Switching to the sky view cancels picking.
  - **Drawing:** the map redraws only when something changes, and the WebGL sky pauses while it shows.
  - **Legibility:** overlay text sits on frosted dark panels (time, wind readout, and one panel holding the layer switch, legend and credit), so it reads over streaks, colours and place names.
    - The sun-angle and cloud lines are hidden on the map; they stay in the sky view.
    - The sky view stays unboxed to keep the photographic look.
  - **Tile provider:** Esri works without a key for personal, non-commercial use, and the credit line ("Esri, HERE, Garmin, © OpenStreetMap contributors") must stay. CARTO, the first choice, now watermarks keyless tiles. Stadia (free account, domain registered) is the next fallback. Change the provider in `BASEMAP` and the `.attrib` credit.
  - **Commercial use** would need an ArcGIS account and key (or Stadia Starter), plus a paid Open-Meteo plan, hosting other than GitHub Pages, and removing `previewData()`.
  - **Wind layer:** one Open-Meteo request for a 7×7 grid (49 points) covering the view plus a 25% margin, fetched once panning or zooming has settled (600 ms).
    - **Caching:** grids are cached for an hour, at most 6. A grid is reused while it covers the view and is no more than 3× wider than it.
    - **Interpolation:** node winds are interpolated between forecast hours at the selected time, then bilinearly across the grid.
    - **Streaks:** about 1 particle per 220 px², each drifting with the local wind (about 36 px/s at 20 km/h). They're coloured by speed with a legend under the credit, and their trails fade each frame. Moving the map clears the trails.
    - **Reduced motion:** static arrows instead.
    - **Status:** the centre note says "Loading wind…", "The wind map didn't load" or "No wind forecast for this time" when relevant.
  - **Wave layer** (the Wind / Waves switch; hidden in land mode, and the viewer's choice is remembered for when they're back on the water):
    - **Data:** a 7×7 marine grid for the view, using the same fetch, cache and sampling path as the wind (the `GRID` table). It uses the marine model's default cell selection, so unmodelled lakes such as Lake Simcoe return no data and stay uncoloured, while points just inland take the nearest offshore cell.
    - **Colours:** node colours are smoothly scaled up and cut to a water mask taken from the tiles themselves. Esri Dark Gray water is a flat (35,34,39) and land (78,78,80); labels count as land, so they stay readable. Tiles load with `crossOrigin` so their pixels can be read. If they can't be read, the colours fall back to a fainter, unmasked wash.
    - **Direction:** white lines drift the way the waves travel, clipped to water every frame. There's no swell display, since ECMWF's free feed has none (see Wave data).
    - **Following the time:** the colours redraw once per minute of selected-time change. Only real map moves restart the grid-settle timer; time-only redraws resetting it once stopped the wave fetch from ever starting.
- **Sky renderer.** A WebGL1 fragment shader draws the scene.
  - Physically based single-scattering atmosphere: Rayleigh and Mie scattering, with an Earth-shadow test so the sky darkens correctly as the sun drops below the horizon.
  - Added on top: a twilight "fill" term and a horizon glow toward the sun, because single scattering alone makes twilight too dark.
  - Also drawn: stars, a moon with the correct phase, and a distant shoreline silhouette.
  - **Night sky:** airglow that brightens toward the horizon (darkest overhead, as in reality), so the horizon and shoreline stay visible at night. An optional warm city glow sits low on the horizon (a setting, on by default) and washes out the faintest stars. Moonlight scales with phase (steeply: a quarter moon is far dimmer than half a full moon) and with the moon's height. It adds a blue scattered-light sky and a halo, a moon glitter path on the water using the same model as the sun's, and a disc and path that redden near the horizon.
  - **Clouds:** three layers from the forecast's low, mid and high cover (shells at 1.6, 4.5 and 9.5 km): roughly 1 km cumulus, smaller mid-level cloud, and cirrus streaked along the wind. Detail too small for a pixel fades to its average, so clouds near the horizon blend into a band. They drift with the wind, offset by the selected time, so scrubbing shows them move.
    - **Lighting:** sunlight per layer, set in JS so higher clouds stay lit longer after sunset; light diffused through the cloud; direct light on cloud bases when the sun is low; bright edges near the sun; sky light from a JS port of the atmosphere (`atmoJS`, keep it in step with the shader); and city glow on the bases at night.
    - **Effects:** clouds block the sun, moon and stars, appear in water reflections, and dim the sun and moon glitter when they pass in front.
    - **Noise:** the lattice hash uses exact integer arithmetic (a mod-289 permutation). Float hashes produced straight seams once the GPU compiler reordered the maths. Don't swap it back.
  - **Stars:** a random field with many faint stars and few bright ones, coloured by temperature, each at least about a pixel wide at any render scale. It turns about the true celestial pole with local sidereal time (checked against Vega's real position), so dragging through the night sweeps the stars from east to west.
  - **Water:** a sum of 12 wind-driven waves with analytic slopes, moving at deep-water speed. Steepness and wavelength grow with the wind. Waves smaller than about 2 pixels fade out, and their slope variance moves into a microfacet roughness term instead of aliasing. That roughness blurs the shoreline reflection and shapes the sun glitter path, a Beckmann highlight lit by the same sun as the atmosphere. Gust patches roughen and darken the surface, and whitecaps appear above roughly 25 km/h. Don't go back to value-noise heightfields or hard reflection tests: they produce blocky slabs and hard-edged black patches.
  - **Exposure:** rises as the sun sets, the way a camera would adapt.
  - **Performance:** the render scale drops if frames are slow and climbs back when they're steady, but never back to a scale that already proved too slow. The first 1.5 seconds are ignored. The frame rate is capped at about 30 fps.
- **Sun and moon math** is an inline port of SunCalc (BSD licence, credited in the code), with two upgrades:
  - **Sun position** uses the NOAA solar calculator equations, not SunCalc's low-precision sun, which drifts by about 0.2°.
  - **Event times** (−18°, −12°, −6°, −0.833°, −4° blue hour, +6° golden hour) start from SunCalc's `getTimes`, then `timesFor()` refines each one by bisection on the sun's actual altitude. They match NOAA to the second.
  - Moon position, illumination and phase are still SunCalc.
- **Wind data** comes from the Open-Meteo hourly forecast: free, no API key, and free for non-commercial use.
  - It fetches 3 days of hourly speed, gusts, direction, precipitation chance, air temperature, apparent ("feels like") temperature and cloud cover (total, low, mid, high), with wind in km/h. The preview fallback has no cloud data, so it shows a clear sky and the HUD says "no data in preview."
  - **Model switch** (tap the "Live wind" chip): Auto (`best_match`, the default; NOAA HRRR then GFS around Toronto), Canada (`gem_seamless`: HRDPS 2.5 km for 2 days, then GEM regional) and ECMWF (`ecmwf_ifs`, Windy's default). Saved as `dp.wmodel`.
    - **Why not HRDPS alone:** it stops after about 51 hours and has no rain chance.
    - **Scope:** the model drives the spot forecast (wind, gusts, rain, temperature, clouds) and the map's wind grid, and map grids are cached per model. Waves stay on ECMWF WAM.
    - **Switching** clears the old numbers first, so one model's forecast is never shown under another's name.
    - **How they differ at the owner's spots:** steady wind agrees within a few km/h at Humber and Pickering; Lake Wilcox differs more. ECMWF gusts run about 10 km/h higher than the other two.
  - **Value timing:** Open-Meteo gives wind speed as an instant at each hour, so it's interpolated. Gusts are the maximum over the preceding hour, and rain chance covers the preceding hour, so `windAt()` gives any moment in hour [i, i+1) the value stamped i+1, unblended. Blending gusts shaved peaks off the chart and made the verdict slightly too generous.
  - While the first fetch is in flight, the UI shows a loading state, not preview data.
  - If the fetch fails, `previewData()` supplies a fixed Toronto estimate pinned to Oct 9–11 2026, and the UI labels it as preview data. Days outside those dates show no wind.
  - It refetches when the forecast is over 30 minutes old or didn't load. This is checked every minute and whenever the app returns to the foreground. If a refresh fails, the app keeps the last live forecast and flags it as possibly out of date. The day tabs roll over at midnight.
- **Wave data** comes from the Open-Meteo Marine API, pinned to **ECMWF WAM** (`models=ecmwf_wam`, 9 km grid, hourly), the model Windy uses for waves. It fetches 3 days of hourly wave height, direction and period, alongside the wind.
  - **No swell:** Open-Meteo's ECMWF feed carries total waves only, even on the ocean. Windy shows ECMWF "Swell 1" from ECMWF's full commercial data. On a lake, swell is mostly leftover wind waves with about a 3 s period, and it's already included in the total. The owner chose hourly ECMWF without swell over MeteoFrance MFWAM, which has swell but is 3-hourly. If an ocean location ever matters, add MFWAM back for swell only, since one request can name both models.
  - **Model choice:** don't pin NCEP GFS Wave, which returns zeros on the Great Lakes.
  - **Cell selection:** `cell_selection=sea` takes the nearest water cell. If that cell is more than 30 km away, the location counts as not near modelled water, and the app says "no forecast near here" instead of borrowing it.
  - **Direction:** given as "from," like wind.
  - **No preview fallback:** without live data, no wave numbers are shown.
  - **Display only:** waves appear in the scene HUD, in the verdict sentence and as a line on the timeline. They never change the rating, because the forecast describes open water a few km out, not the shoreline.
- **Saved locations** (`dp.locs`): tapping the location chip opens a menu to switch, add ("Add a spot on the map" with the pin picker, or "Use my location") and remove spots (×, with a confirm; the last one can't be removed).
  - **Defaults** are the owner's spots, each checked against the tiles to be on water: just off the Humber River mouth (43.6285, −79.4705), the lake in front of Pickering's Beachfront Park (43.8080, −79.0770) and the centre of Lake Wilcox (43.9493, −79.4365).
  - **Upgrade:** on first run of this version, an existing custom location is kept, added to the list and stays current.
  - **Editing:** Settings "Save location" edits the current spot in place, and a learned time zone is written back to it.
  - **Lake Wilcox** has no wave forecast, since the models don't cover small inland lakes, and says so.
- **Settings** (current location, saved locations, mode, each mode's gust limits, city glow) are saved in `localStorage`. Note that the desktop app's file preview blocks `localStorage`, so test persistence over a local web server. "Use my location" calls the browser's geolocation, which needs HTTPS.

## Verdict rules

- **Two modes**, "On the water" and "On land," switched with the pills above the day tabs.
  - Each mode keeps its own limits: water defaults to 25 / 35 km/h, land to 35 / 50 km/h. Older single limits (`dp.lim`) became the water mode's.
  - Land mode hides all wave information.
  - Keep mode names generic: no activity names.
- **Calm:** gusts at or below the calm limit and sustained wind at or below about 60% of it.
- **Rough:** gusts above the rough limit or sustained wind above about 62% of it.
- **Manageable:** anything in between.
- **Rain:** a precipitation chance of 50% or more downgrades Calm to Manageable.
- **Daylight only:** only civil daylight counts (civil dawn to civil dusk). For today, counting starts from now.
- **Windows:** the reported window is the longest run of Calm hours (at least 30 minutes). If there is none, it is the longest run of Manageable-or-better hours.

## Conventions

- Canadian context: metric units (km/h, °C), en-CA dates, 12-hour times written "6:55 am".
- All times and day boundaries use the location's own time zone, not the device's. The zone comes from the Open-Meteo response and is saved with the location. Until it's known, the device's zone is used and the location chip says "device time." When the location's zone differs from the device's, the chip shows its abbreviation. Use `wall()`, `zoned()`, `fmt()` and `fmtDate()` for anything time-related, never `Date#getHours()` or `toLocaleDateString()` without a time zone.
- Copy is plain and direct, with no exclamation marks. Wind uses Beaufort names (e.g. "gentle breeze").
- The dark palette is deliberate in every theme. Colours live as CSS tokens at the top of the `<style>` block.
- The layout must work at 400px wide with no horizontal scrolling. Respect `prefers-reduced-motion`.
  - `.app` uses `grid-template-columns:minmax(0,1fr)`, and the header chips can shrink and cut long text off with "…". On iPhone Safari, the wrapping chip row once widened the whole page, which Chromium doesn't reproduce. Keep both guards.
- **Time can be changed on the scene itself**, so the sky stays in view while scrubbing:
  - A horizontal drag on the sky nudges time. The full scene width is 6 hours, about a minute per pixel on a phone. Vertical drags still scroll the page (`touch-action: pan-y`).
  - The slim strip along the bottom of the scene maps the whole day and marks the six twilight moments.
  - The full timeline below stays as well, for the wind chart.
- **Air temperature** shows under the time of day in the scene, in both views. "Feels like" is added only when it differs by 2° or more. Water temperature isn't available: Open-Meteo's marine sea-surface temperature is empty for Lake Ontario.
- **Timeline chart scale:**
  - The top is at least 5 km/h above the day's strongest gust (and above the rough limit), rounded up to 10.
  - Each gridline is labelled just above itself. The dashed limit lines carry "calm N" and "rough N" tags, and the wave scale reads "waves N m."
  - The labels are outlined in the panel colour so bars can't hide them.
- **Timeline numbers:**
  - The header shows the exact wind, gusts, (on the water) waves and air temperature at the selected time.
  - Under the chart, one column per 3-hour block (equal columns line up with the chart) shows the average wind, coloured by the worst rating in that block, the strongest gust, on the water the highest waves, and the average air temperature.
  - The selected block is highlighted and night blocks are faded. Hourly numbers on every bar would be too cramped at phone width.

## Testing

- **Open it:** `index.html` works straight from disk, or serve it with `python3 -m http.server` and visit localhost:8000.
- **Test a moment:** in the browser console, call `selectDay(1); setTime(dayStart(1)+6.5*HOUR)` to jump to 6:30 am tomorrow.
- **Check the visuals:** after any shader change, compare screenshots at these sun altitudes:
  - −20° (night)
  - −15° (astronomical twilight)
  - −10° (nautical twilight)
  - −4° (blue hour)
  - 0° (sunrise)
  - +10° (low daylight)
  - noon
- **Check clouds:** run `cloudAt=()=>({low:.4,mid:.2,high:.3,total:.6})` in the console to fake cover. Check a sunny afternoon, a sunset at +1°, dusk at −4°, overcast (`low:1`) and night with city glow. Open a fresh tab to restore the forecast.
- **Check moonlight:** the real moon may be new, so fake one in the console. Run `SC.getMoonIllumination=()=>({fraction:1,phase:0.5}); SC.getMoonPosition=()=>({azimuth:-45*DEG,altitude:18*DEG})`, then jump to a night time. This gives a full moon 18° up in the south-east. Open a fresh tab afterwards to restore the real moon.

## Hosting

Static files only. GitHub Pages, Netlify or the owner's own website all work. It must be served over HTTPS for geolocation and for installing to the home screen. `manifest.webmanifest` and the icons make it installable via "Add to Home Screen."

## Planned: map view, waves and modes

A Windy-style map as a second view in the top scene, wave data, and two usage modes. The owner agreed the plan on 2026-10-09 and is building it step by step.

**Decisions:**
1. **Map tiles:** Esri World Dark Gray Canvas, with no key. This was changed from CARTO after CARTO turned out to need a key. If tiles fail, draw the layers on plain dark with a note.
2. **Switching:** a **Sky | Map** toggle with page dots at the top centre of the scene. Swiping isn't used, because a horizontal drag on the sky already scrubs time and the map needs drag to pan. The bottom time strip works in both views.
3. **Waves never change the Calm / Manageable / Rough rating.** They appear in the verdict text and on the timeline.
4. **Two modes**, "On the water" and "On land," built as part of this feature:
   - each mode keeps its own wind limits
   - land mode hides wave information
   - the mode switch sits near the verdict
   - existing limits become the water mode's; ask the owner for land-mode defaults
   - keep the wording generic: no activity names
5. **Map area:** opens about 60 km around the saved location, then free pan and pinch-zoom from the whole lake down to local.
   - Refetch the data grid for the view once movement settles, and cache recent grids.
   - Add a recentre button.

**Data budget:** Open-Meteo counts each point in a multi-point request. A fresh map area (7×7 wind plus 7×7 marine) costs about 100 of the 10,000 free daily points. Cache areas, and fetch only while the map is open.

**Build order:**
1. ~~Wave data at the spot: HUD readout, verdict sentence, timeline line, honest no-data states.~~ Done.
2. ~~Two modes, with land at 35 / 50 km/h (the owner's choice).~~ Done.
3. ~~Map shell: toggle and dots, tiles, pan and zoom, recentre, location marker, credits. Pause the WebGL sky while the map shows.~~ Done.
4. ~~Wind layer: grid for the visible area, animated streaks coloured by speed, following the selected time. With reduced motion, show static arrows.~~ Done.
5. ~~Wave layer: wave-height colours over water only, a legend, and a Wind / Waves layer switch.~~ Done, including the direction lines below.
   - **Wave direction**, as Windy shows it: slow-moving lines travelling the way the combined waves go, drawn lighter than the wind streaks so the layers can't be confused.
   - With reduced motion, show static arrows instead.
   - The marine grid already includes direction, so this costs no extra data.
6. Polish: loading and failure states per layer, a phone check, and moving this section into Architecture.

## Ideas not built yet

- An hourly temperature and precipitation strip under the wind chart.
- Several saved locations with quick switching.
- A service worker for offline use with the last forecast cached.
- Remove `previewData()` once the live feed is proven, or keep it only as an offline fallback.
