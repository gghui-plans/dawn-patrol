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
- **No frameworks or npm dependencies.** Google Fonts (Archivo, JetBrains Mono) is the only external load.
- **Sky renderer.** A WebGL1 fragment shader draws the scene.
  - Physically based single-scattering atmosphere: Rayleigh and Mie scattering, with an Earth-shadow test so the sky darkens correctly as the sun drops below the horizon.
  - Added on top: a twilight "fill" term and a horizon glow toward the sun, because single scattering alone makes twilight too dark.
  - Also drawn: stars, a moon with the correct phase, and a distant shoreline silhouette.
  - **Night sky:** airglow that brightens toward the horizon (darkest overhead, as in reality), so the horizon and shoreline stay visible at night. An optional warm city glow sits low on the horizon (a setting, on by default) and washes out the faintest stars. Moonlight scales with phase (steeply: a quarter moon is far dimmer than half a full moon) and with the moon's height. It adds a blue scattered-light sky and a halo, a moon glitter path on the water using the same model as the sun's, and a disc and path that redden near the horizon.
  - **Stars:** a random field with many faint stars and few bright ones, coloured by temperature, each at least about a pixel wide at any render scale. It turns about the true celestial pole with local sidereal time (checked against Vega's real position), so dragging through the night sweeps the stars from east to west.
  - **Water:** a sum of 12 wind-driven waves with analytic slopes, moving at deep-water speed. Steepness and wavelength grow with the wind. Waves smaller than about 2 pixels fade out, and their slope variance moves into a microfacet roughness term instead of aliasing. That roughness blurs the shoreline reflection and shapes the sun glitter path, a Beckmann highlight lit by the same sun as the atmosphere. Gust patches roughen and darken the surface, and whitecaps appear above roughly 25 km/h. Don't go back to value-noise heightfields or hard reflection tests: they produce blocky slabs and hard-edged black patches.
  - **Exposure:** rises as the sun sets, the way a camera would adapt.
  - **Performance:** the render scale drops if frames are slow and climbs back when they're steady, but never back to a scale that already proved too slow. The first 1.5 seconds are ignored. The frame rate is capped at about 30 fps.
- **Sun and moon math** is an inline port of SunCalc (BSD licence, credited in the code). It provides positions, twilight times (−18°, −12°, −6°, −0.833°, −4° blue hour, +6° golden hour), moon position, illumination and phase.
- **Wind data** comes from the Open-Meteo hourly forecast: free, no API key, and free for non-commercial use.
  - It fetches 3 days of hourly speed, gusts, direction, precipitation chance and temperature in km/h.
  - While the first fetch is in flight, the UI shows a loading state, not preview data.
  - If the fetch fails, `previewData()` supplies a fixed Toronto estimate pinned to Oct 9–11 2026, and the UI labels it as preview data. Days outside those dates show no wind.
  - It refetches when the forecast is over 30 minutes old or didn't load. This is checked every minute and whenever the app returns to the foreground. If a refresh fails, the app keeps the last live forecast and flags it as possibly out of date. The day tabs roll over at midnight.
- **Settings** (location, gust limits, city glow) are saved in `localStorage`. "Use my location" calls the browser's geolocation, which needs HTTPS.

## Verdict rules

- **Calm:** gusts at or below the calm limit (default 25 km/h) and sustained wind at or below about 60% of it.
- **Rough:** gusts above the rough limit (default 35 km/h) or sustained wind above about 62% of it.
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
- **Check moonlight:** the real moon may be new, so fake one in the console. Run `SC.getMoonIllumination=()=>({fraction:1,phase:0.5}); SC.getMoonPosition=()=>({azimuth:-45*DEG,altitude:18*DEG})`, then jump to a night time. This gives a full moon 18° up in the south-east. Open a fresh tab afterwards to restore the real moon.

## Hosting

Static files only. GitHub Pages, Netlify or the owner's own website all work. It must be served over HTTPS for geolocation and for installing to the home screen. `manifest.webmanifest` and the icons make it installable via "Add to Home Screen."

## Ideas not built yet

- Clouds driven by the forecast `cloud_cover` (the biggest realism gain).
- An hourly temperature and precipitation strip under the wind chart.
- Several saved locations with quick switching.
- A service worker for offline use with the last forecast cached.
- Remove `previewData()` once the live feed is proven, or keep it only as an offline fallback.
