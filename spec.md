# App Specification — Jobsite Weather Planner

## App Overview
The Jobsite Weather Planner is a static, single-file `index.html` dashboard that helps construction project managers, superintendents, and field engineers compare a seven-local-date weather forecast for one jobsite. It shows separate Go, Caution, No-Go, or Unavailable planning statuses for crane lifts, concrete pours, and roofing, with the forecast values and thresholds behind each result. These statuses are planning indicators, not safety determinations.

## Purpose
The **Jobsite Weather Planner** is a single-file web application for commercial construction project managers, superintendents, and field engineers to make preliminary schedule-planning decisions for **crane lifts**, **concrete pours**, and **roofing** at one selected jobsite. For each of the next seven local calendar dates, the app compares the Open-Meteo hourly forecast with the configured activity thresholds and assigns each activity a **Go**, **Caution**, or **No-Go** planning status. Users can compare the activity statuses across the week, select a day, and inspect the forecast values, timestamps, thresholds, and comparisons behind each status.

These are forecast-based planning indicators, not safety determinations or authorization to perform work. They do not replace the approved lift plan, project requirements, manufacturer instructions, or site safety procedures.

## Inputs
- **Jobsite location:** Latitude `lat` (decimal degrees, valid range `-90` to `90`, default `40.2338`) and longitude `lon` (decimal degrees, valid range `-180` to `180`, default `-111.6585`; near Provo, Utah). Users select a point on an interactive Leaflet map with OpenStreetMap tiles or enter both coordinates in numeric inputs. The map requires internet access and visible OpenStreetMap attribution. Changing the location fetches a forecast for the new coordinates. Coordinates must be finite and within range; both are required.
- **Crane wind limit (`W_max`):** Site/lift-plan sustained-wind limit in `mph`, default `20.0 mph`, allowed range `5.0` to `80.0 mph`. Provide a numeric input and an optional synchronized slider. This is a user-configured planning threshold, not a universal crane limit.
- **Concrete low-temperature limits:** `T_caut`, the cold-weather caution threshold, defaults to `40.0 °F`; `T_nogo`, the No-Go threshold, defaults to `32.0 °F`. Both use numeric inputs and have an allowed range of `-20.0 °F` to `70.0 °F`. A valid configuration requires `T_nogo < T_caut`.
- **Roofing precipitation-probability limits:** `P_caut`, the caution threshold, defaults to `30%`; `P_nogo`, the No-Go threshold, defaults to `60%`. Both use numeric inputs and have an allowed range of `0%` to `100%`. A valid configuration requires `P_caut < P_nogo`.
- **Selected calendar day:** Clicking any displayed date selects it and updates the activity reasons and hourly breakdown.
- Invalid coordinates, out-of-range values, or invalid threshold ordering must produce a visible input error and must not produce ratings from invalid inputs. Valid threshold edits recalculate ratings using the currently loaded forecast; location changes request new forecast data.

## Calculations
Fetch a seven-local-date hourly forecast from the free **Open-Meteo Forecast API** (no API key required). Request `forecast_days=7`, `timezone=auto`, `timeformat=unixtime`, `temperature_unit=fahrenheit`, and `wind_speed_unit=mph`, with hourly fields `temperature_2m`, `wind_speed_10m`, `wind_gusts_10m`, and `precipitation_probability`. Do not use the daily `precipitation_probability_max` field in these hourly calculations.

Convert each returned timestamp to the jobsite's local timezone and group samples by local calendar date; do not assume every date has 24 samples. The seven dates are today and the next six local dates. For today, include only forecast samples whose timestamps are at or after the current time; for the following dates, use all hourly samples in each local date. Local dates affected by daylight-saving transitions may contain 23 or 25 hourly samples.

For each date `d`, let `H_d` be the set of included hourly samples and `i` identify a sample in that set:

- `t_i`: timestamp of sample `i`, displayed in jobsite-local time.
- `v_sust,i`: `wind_speed_10m` in `mph` (sustained wind at 10 m).
- `v_gust,i`: `wind_gusts_10m` in `mph` (wind gust at 10 m).
- `T_i`: `temperature_2m` in `°F` (ambient air temperature at 2 m).
- `P_i`: `precipitation_probability` in `%` for that hourly sample.
- `W_max`: configured crane sustained-wind limit in `mph`.
- `G_nogo`: effective crane gust No-Go threshold in `mph`, defined below.
- `T_caut` and `T_nogo`: concrete caution and No-Go thresholds in `°F`.
- `P_caut` and `P_nogo`: roofing caution and No-Go thresholds in `%`.

Calculate the daily governing values from the same included samples:

- `v_sust,max,d = max(v_sust,i)` over `i` in `H_d`, in `mph`.
- `v_gust,max,d = max(v_gust,i)` over `i` in `H_d`, in `mph`.
- `T_min,d = min(T_i)` over `i` in `H_d`, in `°F`.
- `P_max,d = max(P_i)` over `i` in `H_d`, in `%`. This is the maximum hourly precipitation probability, not the probability that precipitation occurs at any time during the entire date.

Use unrounded numeric values for comparisons; rounding is display-only. If required hourly data is missing, invalid, or incomplete for an activity's evaluation window, show that activity as **Unavailable**, not **Go**, and identify the missing data. Validate that forecast arrays align by timestamp and that returned units match the requested units. Do not silently replace missing values with zero.

### 1. Crane Lifts
Define `G_nogo = min(W_max + 5 mph, 1.25 × W_max)`. The gust cutoff is **inclusive**:
- **No-Go:** `v_sust,max,d >= W_max` or `v_gust,max,d >= G_nogo`.
- **Caution:** No No-Go condition applies, and either `W_max - 5 mph <= v_sust,max,d < W_max` or `W_max < v_gust,max,d < G_nogo`.
- **Go:** Neither No-Go nor Caution applies; equivalently, `v_sust,max,d < W_max - 5 mph` and `v_gust,max,d <= W_max`.

The sustained and gust maxima can occur at different times; report their timestamps separately. The API wind fields are at 10 m and do not estimate wind at crane boom/load height.

### 2. Concrete Pours
- **No-Go:** `T_min,d <= T_nogo` (default `<= 32.0 °F`).
- **Caution:** `T_nogo < T_min,d <= T_caut` (default `32.0 °F` to `40.0 °F`, inclusive at `T_caut`).
- **Go:** `T_min,d > T_caut` (default `> 40.0 °F`).

This rule evaluates forecast ambient air temperature only; it does not calculate concrete temperature, curing, protection, or mix performance.

### 3. Roofing
- **No-Go:** `P_max,d >= P_nogo` (default `>= 60%`).
- **Caution:** `P_caut <= P_max,d < P_nogo` (default `30%` to `< 60%`).
- **Go:** `P_max,d < P_caut` (default `< 30%`).

This rule uses hourly precipitation probability only; it does not evaluate precipitation amount, duration, type, or substrate conditions.

### Activity Rating and Explanation Rule
- Assign a status independently for each activity. Do not calculate or display a combined overall-day status.
- For each activity, generate a plain-English explanation naming the governing metric, value, unit, local timestamp, threshold, and comparison operator. Determine status from unrounded values and show enough precision to make the comparison auditable. If multiple hours tie for a governing maximum or minimum, report all tied timestamps.
- A **No-Go** or **Caution** status uses the activity's exact rule above; do not infer a different status from rounded display values.

## Outputs
- **Interactive map:** A Leaflet map with OpenStreetMap tiles displays the selected jobsite pin and visible attribution. Map clicks update `lat` and `lon`; coordinate inputs update the pin. Display the selected coordinates in decimal degrees.
- **Seven-day activity calendar:** Shows seven local dates and separate labeled **Go**, **Caution**, **No-Go**, or **Unavailable** statuses for **Crane Lifts**, **Concrete Pours**, and **Roofing**. Use text labels as well as color (green, amber, red); do not rely on color alone. Do not display an overall-day status.
- **Selected-day details:** For each activity, show its status, governing value and unit, local timestamp(s), configured threshold(s), comparison operator, and a reason string. Include an hourly table with local timestamp, sustained wind at 10 m (`mph`), gust at 10 m (`mph`), temperature at 2 m (`°F`), and hourly precipitation probability (`%`). Put units in each metric's label or table header.
- **Location and forecast metadata:** Show selected coordinates, jobsite timezone, and forecast retrieval time.
- **Status and error messages:** Show a loading state during fetches, a retryable readable message for network/API errors, and a distinct unavailable state for missing or incomplete forecast data. Invalid inputs show an input-specific error and do not produce ratings. Never present missing data as **Go**.

## Look and feel
- **Responsive layout:** Build a high-contrast field dashboard in a single `index.html`. On desktop, place the map and location/threshold controls in a compact side column and the activity calendar and selected-day details in the main column. On phone-sized screens, stack these sections vertically and use full-width day cards; the page must not overflow horizontally. Hourly details may scroll within their own bounded region if needed, without causing page-wide horizontal scrolling.
- **Touch and readability:** Use readable text, clear focus indicators, sufficiently large touch targets, and visible text labels for statuses in addition to color. Keep the map compact on phones so the calendar and selected-day summary remain easy to reach.
- **Initial open state:** Start with the default Provo-area coordinates and thresholds, show a loading state while fetching the forecast, then show the calendar and select the earliest date with a **Caution** or **No-Go** activity; if none exists, select the first date. Show only per-activity statuses, not an overall-day status.

## Validation
The validation record is maintained in `VALIDATION.md` using the captured Open-Meteo fixture below and a contextual NWS comparison.

### Captured Open-Meteo fixture

- Site: Provo, Utah area
- Coordinates: `lat = 40.2338`, `lon = -111.6585`
- Open-Meteo response coordinates: `40.230103`, `-111.65572`
- Jobsite timezone: `America/Denver`
- UTC offset: `-25200` seconds (`GMT-7`)
- Target local date: `2026-11-23`
- Forecast retrieval display time: `Nov 23, 2026, 2:38 PM`
- Elevation: `1392 m`
- Exact request: `https://api.open-meteo.com/v1/forecast?latitude=40.2338&longitude=-111.6585&hourly=temperature_2m%2Cwind_speed_10m%2Cwind_gusts_10m%2Cprecipitation_probability&forecast_days=7&timezone=auto&timeformat=unixtime&temperature_unit=fahrenheit&wind_speed_unit=mph`
- Units: `temperature_2m` in °F, `wind_speed_10m` and `wind_gusts_10m` in mph, and `precipitation_probability` in `%`

The evaluated window is the current partial day: 2026-11-23 from 3:00 PM MST through 11:00 PM MST. Pre-current-hour samples are excluded.

| Local timestamp | Unix timestamp | UTC offset | Sustained wind 10 m (mph) | Gust 10 m (mph) | Temperature 2 m (°F) | Precipitation probability (%) |
|---|---:|---|---:|---:|---:|---:|
| 2026-11-23 15:00 MST | 1795471200 | GMT-7 | 5.3 | 5.6 | 47.0 | 2 |
| 2026-11-23 16:00 MST | 1795474800 | GMT-7 | 6.5 | 8.7 | 45.7 | 1 |
| 2026-11-23 17:00 MST | 1795478400 | GMT-7 | 4.9 | 7.4 | 40.9 | 1 |
| 2026-11-23 18:00 MST | 1795482000 | GMT-7 | 4.3 | 5.6 | 37.4 | 1 |
| 2026-11-23 19:00 MST | 1795485600 | GMT-7 | 3.3 | 6.5 | 35.0 | 1 |
| 2026-11-23 20:00 MST | 1795489200 | GMT-7 | 4.7 | 4.9 | 34.2 | 0 |
| 2026-11-23 21:00 MST | 1795492800 | GMT-7 | 4.6 | 6.9 | 33.2 | 0 |
| 2026-11-23 22:00 MST | 1795496400 | GMT-7 | 0.5 | 1.8 | 32.8 | 0 |
| 2026-11-23 23:00 MST | 1795500000 | GMT-7 | 2.3 | 2.7 | 31.8 | 0 |

### Hand calculation

Configured thresholds:

- `W_max = 20.0 mph`
- `G_nogo = min(W_max + 5 mph, 1.25 × W_max) = min(25 mph, 25 mph) = 25 mph`
- Sustained caution floor = `W_max - 5 mph = 15 mph`
- `T_caut = 40.0 °F`
- `T_nogo = 32.0 °F`
- `P_caut = 30%`
- `P_nogo = 60%`

Crane calculation:

- `v_sust,max,d = 6.5 mph` at 2026-11-23 16:00 MST
- `v_gust,max,d = 8.7 mph` at 2026-11-23 16:00 MST
- `6.5 mph < 20 mph` and `8.7 mph < 25 mph`
- No sustained or gust No-Go condition applies
- Sustained wind is below the 15 mph caution floor and gust is not above 20 mph
- Hand status: **Go**

Concrete calculation:

- `T_min,d = 31.8 °F` at 2026-11-23 23:00 MST
- `31.8 °F <= 32.0 °F`
- Hand status: **No-Go**

Roofing calculation:

- `P_max,d = 2%` at 2026-11-23 15:00 MST
- `2% < 30%`
- Hand status: **Go**

### App actual output

The app currently displays the partial-day governing values and reasons using unrounded raw numeric values:

- Crane Lifts: **Go**; sustained maximum `6.5 mph` at 16:00 MST, gust maximum `8.7 mph` at 16:00 MST; thresholds `W_max = 20 mph`, sustained caution floor `15 mph`, and `G_nogo = 25 mph`.
- Concrete Pours: **No-Go**; minimum `31.8 °F` at 23:00 MST; thresholds `T_nogo = 32 °F` and `T_caut = 40 °F`.
- Roofing: **Go**; maximum hourly precipitation probability `2%` at 15:00 MST; thresholds `P_caut = 30%` and `P_nogo = 60%`.

The app additionally displays one local Unix timestamp, local timestamp, UTC offset, and all four raw hourly inputs for each included hour. It exposes the exact request URL and the returned response coordinates, timezone, UTC offset, elevation, and units in the validation details panel.

### NWS contextual comparison

- NWS endpoint: `https://api.weather.gov/gridpoints/SLC/105,151/forecast/hourly`
- NWS grid: `SLC 105,151`
- Grid timezone: `America/Denver`
- NWS retrieval context: generated `2026-11-23T21:29:26Z` (`14:29 MDT`); grid update `2026-11-23T18:31:28Z` (`11:31 MDT`)
- Matching window: 2026-11-23 15:00 through 23:00 MST

| Local interval | NWS temperature (°F) | NWS sustained wind (mph) | NWS precipitation probability (%) |
|---|---:|---:|---:|
| 15:00-16:00 MST | 46 | 2 | 1 |
| 16:00-17:00 MST | 47 | 2 | 1 |
| 17:00-18:00 MST | 45 | 2 | 1 |
| 18:00-19:00 MST | 43 | 2 | 1 |
| 19:00-20:00 MST | 42 | 2 | 1 |
| 20:00-21:00 MST | 39 | 2 | 1 |
| 21:00-22:00 MST | 37 | 1 | 0 |
| 22:00-23:00 MST | 36 | 1 | 0 |
| 23:00-24:00 MST | 35 | 1 | 0 |

The NWS and Open-Meteo sources use different grids and retrieval times and are not required to match exactly. NWS provides sustained wind as an hourly text interval and does not provide an explicit gust field; it has no gust value suitable for substituting into the crane gust rule. This comparison is contextual and is not a pass/fail oracle for Open-Meteo arithmetic.

### Verification coverage

Before completion, fixtures must cover exact threshold boundaries, tied governing hours, invalid threshold ordering, missing/null hourly values, API/network errors, and the current partial day. Invalid or incomplete data must not produce a false **Go**. The app matches the hand calculation above for the captured fixture, with ratings determined before display rounding.

## Out of scope
- More than one selected jobsite at a time; user accounts, cloud saving, team collaboration, local forecast history, task assignment, crew scheduling, and work-order management.
- Runtime weather sources other than the Open-Meteo Forecast API, including NWS runtime data, live observations, radar, and official alert feeds. NWS is used only for validation comparison.
- Guarantees of forecast accuracy, completeness, update timing, or service availability. Forecast values may differ from measured jobsite conditions, provider/model, forecast grid, and retrieval time.
- Crane-specific engineering or operating decisions, including boom/load-height wind estimation, crane configuration, rigging, load wind area, lift calculations, and certification. The 10 m forecast is not a crane-height measurement.
- Concrete mix design, concrete/substrate temperature, curing or protection modeling, strength/maturity calculations, and high-temperature concreting rules.
- Roofing-material-specific rules, precipitation amount/duration/type analysis, substrate conditions, and other hazards not represented by hourly precipitation probability.
- Safety certification, OSHA compliance certification, or replacement of approved project documents, manufacturer instructions, competent-person judgment, or site safety procedures. Go/Caution/No-Go labels are planning indicators only.
- Paid weather services requiring API keys, proprietary radar/telematics feeds, or a live backend. The application remains a static single-file `index.html` suitable for static hosting. Forecast access requires the Open-Meteo network service; Leaflet and OpenStreetMap tiles require internet access, attribution, and compliance with the tile provider's usage policy. Open-Meteo receives the jobsite coordinates; map-tile requests reveal the viewed area.

## Features
The must-have forecast, calculation, calendar, and validation behaviors remain in place. The custom features below are planned additions; they do not alter the Go, Caution, No-Go, or Unavailable rules used for individual activities.

### Feature 1: Working-Window Finder

**Purpose:** Help a superintendent or field engineer identify consecutive portions of a selected shift when crane lifts, concrete pours, and roofing can proceed under the app's planning indicators.

**Inputs:**
- Selected local calendar date. This is independent of the currently selected day and must update when the user changes the date.
- Work start time and work end time in jobsite-local time.
- Minimum uninterrupted duration of 1 to 12 full forecast hours. The default is 4 hours.
- Qualifying statuses: **Go** or **Caution**. A window may contain only hours whose selected activity is Go or Caution.
- No-Go and Unavailable hours stop or prevent qualification.

**Rules and calculations:**
- Evaluate one activity at a time and produce three separate lists: Crane Lifts, Concrete Pours, and Roofing.
- Convert the shift start and end times to the selected date's local time.
- The window-finder date is an independent input from the selected calendar day. Changing one does not change the other.
- A shift is a same-local-date interval; the end time must be later than the start time. Overnight shifts are not supported.
- If a local time is ambiguous during a daylight-saving fall-back, use its earlier occurrence. Reject nonexistent local times during a spring-forward gap with a visible input error.
- Include only complete hourly forecast samples whose full hour falls entirely inside the selected shift.
- Exclude partial overlapping edge hours rather than treating a partial hour as a full usable hour.
- If the selected shift is shorter than one complete hour or crosses no complete included forecast hour, return an explicit no-window result.
- Examine complete hourly activity statuses in chronological order and find consecutive sequences containing only Go or Caution.
- Split a window when an hour is No-Go or Unavailable, or when its status breaks the consecutive sequence.
- Report a window only when the sequence length is at least the configured minimum duration in full forecast hours.
- Duration is a count of complete qualifying forecast hours, not a wall-clock estimate within a partial edge hour.
- For each qualifying window, retain the worst included status; **Caution** is worse than **Go**.
- Crane governing evidence must preserve separate sustained-wind and gust maxima/timestamps within the window. Concrete preserves the window minimum temperature and tied timestamps. Roofing preserves the window maximum hourly precipitation probability and tied timestamps.
- Missing or invalid required hourly values make the affected hour Unavailable and therefore cannot qualify as part of a window.

**Outputs:**
- Three chronological window lists, one for each activity.
- Each window shows local start time, local end time, complete-hour count/duration, worst status, governing value and unit, and governing local timestamp(s).
- If multiple qualifying windows exist, display them separately in start-time order.
- If no qualifying window meets the minimum duration, show a readable no-window message identifying whether the reason is shift boundaries, No-Go hours, or missing data.
- Do not calculate or display a combined overall window status.

**Validation:**
- Test exact one-hour shift boundaries and shifts with partial edge hours.
- Test 1-hour and 12-hour minimum-duration boundaries.
- Test alternating Go, Caution, No-Go, and Unavailable hours.
- Test tied governing hours within a window.
- Verify that partial edge hours are excluded, Caution qualifies, and missing data never creates a false usable window.

### Feature 2: Shift Handoff Snapshot (Original idea)

**Purpose:** This is the original, construction-specific feature. It gives a superintendent or foreman one concise, source-attributed snapshot of the next shift's weather planning results for all three activities, rather than requiring them to repeat the calendar, threshold, governing-value, and hourly details manually during a handoff.

**Inputs:**
- Selected local calendar date.
- All three activities: Crane Lifts, Concrete Pours, and Roofing.
- No free-text note field. The snapshot is forecast-derived and source-attributed.
- Snapshot format selection:
  - Copy a plain-text snapshot.
  - Print or print/save to PDF using the browser print dialog.

**Rules/calculations:**
- Do not recalculate activity ratings for the snapshot.
- Use the currently loaded normalized forecast and the current valid threshold configuration.
- Include the selected date's result for each activity exactly as shown in the app.
- Include the exact reason text, status, governing value and unit, governing timestamp(s), and thresholds for all three activities.
- Include the raw hourly rows for the selected date, with local timestamp, UTC offset, Unix timestamp, sustained wind at 10 m in mph, gust at 10 m in mph, temperature at 2 m in °F, and hourly precipitation probability in %.
- Include site coordinates, jobsite timezone, forecast retrieval time, exact Open-Meteo request, returned units, and source elevation when available.
- Include an explicit planning-only disclaimer.
- If thresholds are invalid or required forecast data is unavailable, mark affected activities as Unavailable and prohibit presenting the snapshot as successful forecast evidence for those activities.
- Keep each activity independent; do not add or calculate an overall-day status.

**Outputs:**
- A copyable plain-text handoff block with headings for location, source metadata, all three activity results, and selected-date hourly data.
- A print-friendly layout containing the same source metadata and activity/hourly information.
- A visible notice in the UI that the snapshot reflects a retrieval at a specific time and is a planning aid, not authorization or a safety determination.

**Validation:**
- Verify copied text contains the same status, governing value/unit, timestamp(s), thresholds, and reason as the app.
- Verify print output includes the same values and source/retrieval metadata.
- Verify invalid thresholds and missing/null hourly values remain visibly Unavailable.
- Verify no overall-day status or safety certification language is introduced.

## Build stages
Build the application one stage at a time, beginning with the page layout mockup.

### Stage 1: Page layout mockup
**Build:** Create the page structure and responsive visual mockup with placeholder forecast data: header and location summary, map area, coordinate and threshold controls, seven-day activity cards, selected-day detail, hourly table, and loading/error states. Do not connect real forecast data yet.

**Check:** Review the information hierarchy at desktop and phone widths. Confirm there is no horizontal page overflow, day statuses are readable without color alone, and the map placeholder does not crowd ratings on mobile.

### Stage 2: Location, inputs, and forecast connection
**Build:** Implement coordinate entry, map selection with Leaflet and OpenStreetMap tiles, threshold controls and validation, and the Open-Meteo request for the specified hourly fields and units. Add loading, retryable API/network errors, selected-coordinate display, visible map attribution, and forecast retrieval metadata.

**Check:** Confirm the default Provo-area coordinates and thresholds load; map and coordinate changes stay synchronized; invalid coordinates and threshold ordering show errors; valid threshold changes remain configured for Stage 3 calculations; requests use the required fields, units, timezone, and seven-day window; and fetch failures never appear as successful data. Do not calculate activity ratings in Stage 2.

### Stage 3: Forecast normalization and activity calculations
**Build:** Convert timestamps to jobsite-local time, group the seven dates correctly (including daylight-saving days), use only future hours for today, validate response arrays and units, and implement all crane, concrete, and roofing formulas exactly. Missing or incomplete required data produces **Unavailable**.

**Check:** Use fixed hourly fixtures to verify each maximum/minimum, timestamp, and status. Exercise every exact threshold boundary, tied governing hours, invalid/missing values, and today's partial window. Verify comparisons use unrounded values and missing data never becomes **Go**.

### Stage 4: Calendar, explanations, and hourly details
**Build:** Replace placeholders with the seven-day per-activity statuses, select the earliest risk date on load (or the first date when none is at risk), and render the selected date's reasons, thresholds, governing values/times, and hourly table. Do not add a combined overall-day status.

**Check:** Confirm each day and activity is selectable and readable; explanations trace to the exact rule and local time; tied timestamps are all shown; all displayed metrics have units; and status text remains clear without color.

### Stage 5: Validation and field-use polish
**Build:** Complete the hand-calculation validation record using captured Open-Meteo data, compare matching hourly intervals with the NWS forecast as context, and refine responsive, keyboard/touch, and error-state behavior.

**Check:** Confirm the app matches hand-calculated ratings and reasons for the same fixture; document source and time-window differences in the NWS comparison; test network/API failures, incomplete data, and supported screen widths; verify there is no horizontal page overflow and no wording implies safety certification.

**Dependency:** The interactive map uses Leaflet with OpenStreetMap tiles. Verify attribution and tile-service usage requirements before public deployment.