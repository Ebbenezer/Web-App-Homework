# App Specification — Jobsite Weather Planner

## Purpose
The **Jobsite Weather Planner** is a single-file web application for commercial construction project managers, superintendents, and field engineers to make preliminary schedule-planning decisions for **crane lifts**, **concrete pours**, and **roofing** at one selected jobsite. For each of the next seven local calendar dates, the app compares the Open-Meteo hourly forecast with the configured activity thresholds and assigns each activity a **Go**, **Caution**, or **No-Go** planning status. Users can compare the activity statuses across the week, select a day, and inspect the forecast values, timestamps, thresholds, and comparisons behind each status.

These are forecast-based planning indicators, not safety determinations or authorization to perform work. They do not replace the approved lift plan, project requirements, manufacturer instructions, or site safety procedures.

## Inputs
- **Jobsite location:** Latitude `lat` (decimal degrees, valid range `-90` to `90`, default `40.2338`) and longitude `lon` (decimal degrees, valid range `-180` to `180`, default `-111.6585`; near Provo, Utah). Users select a point on an interactive map or enter both coordinates in numeric inputs. The map provider is not selected in this specification. Changing the location fetches a forecast for the new coordinates. Coordinates must be finite and within range; both are required.
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
- **Interactive map:** Displays the selected jobsite pin. Map clicks update `lat` and `lon`; coordinate inputs update the pin. Display the selected coordinates in decimal degrees. Map provider selection remains open.
- **Seven-day activity calendar:** Shows seven local dates and separate labeled **Go**, **Caution**, **No-Go**, or **Unavailable** statuses for **Crane Lifts**, **Concrete Pours**, and **Roofing**. Use text labels as well as color (green, amber, red); do not rely on color alone. Do not display an overall-day status.
- **Selected-day details:** For each activity, show its status, governing value and unit, local timestamp(s), configured threshold(s), comparison operator, and a reason string. Include an hourly table with local timestamp, sustained wind at 10 m (`mph`), gust at 10 m (`mph`), temperature at 2 m (`°F`), and hourly precipitation probability (`%`). Put units in each metric's label or table header.
- **Location and forecast metadata:** Show selected coordinates, jobsite timezone, and forecast retrieval time.
- **Status and error messages:** Show a loading state during fetches, a retryable readable message for network/API errors, and a distinct unavailable state for missing or incomplete forecast data. Invalid inputs show an input-specific error and do not produce ratings. Never present missing data as **Go**.

## Look and feel
- **Responsive layout:** Build a high-contrast field dashboard in a single `index.html`. On desktop, place the map and location/threshold controls in a compact side column and the activity calendar and selected-day details in the main column. On phone-sized screens, stack these sections vertically and use full-width day cards; the page must not overflow horizontally. Hourly details may scroll within their own bounded region if needed, without causing page-wide horizontal scrolling.
- **Touch and readability:** Use readable text, clear focus indicators, sufficiently large touch targets, and visible text labels for statuses in addition to color. Keep the map compact on phones so the calendar and selected-day summary remain easy to reach.
- **Initial open state:** Start with the default Provo-area coordinates and thresholds, show a loading state while fetching the forecast, then show the calendar and select the earliest date with a **Caution** or **No-Go** activity; if none exists, select the first date. Show only per-activity statuses, not an overall-day status.

## Validation
Validation is **pending implementation and capture of a forecast fixture**; do not record fabricated values. Before treating a build as validated:

1. Record the site name, `lat`, `lon`, jobsite timezone, target local date, forecast retrieval time, and exact Open-Meteo request/units.
2. Save every hourly sample in the evaluated window with timestamp, `wind_speed_10m` (`mph`), `wind_gusts_10m` (`mph`), `temperature_2m` (`°F`), and `precipitation_probability` (`%`). Include each timestamp's local offset where relevant.
3. Record the configured thresholds and hand-calculate all governing maxima/minima, their timestamps, each comparison, and each activity status using the rules in Calculations.
4. Compare the app's statuses, displayed governing values/timestamps, and reason text against the hand calculation using the same raw fixture. Ratings and boundary comparisons must match; display rounding must not change a rating.
5. Test exact boundaries for every threshold, tied governing hours, invalid threshold ordering, missing/null hourly values, API/network errors, and a current partial day. Invalid or incomplete data must not produce a false **Go**.
6. Compare against the NWS hourly forecast for the same coordinates, local date, and matching time intervals. Record both retrieval times and note differences in source, grid, interval, units, wind/gust definition, temperature, precipitation probability, and timing. Compare only fields both sources explicitly provide; do not infer NWS gusts from sustained wind. NWS comparison is contextual and is not a pass/fail oracle for Open-Meteo arithmetic.

## Out of scope
- More than one selected jobsite at a time; user accounts, cloud saving, team collaboration, local forecast history, task assignment, crew scheduling, and work-order management.
- Runtime weather sources other than the Open-Meteo Forecast API, including NWS runtime data, live observations, radar, and official alert feeds. NWS is used only for validation comparison.
- Guarantees of forecast accuracy, completeness, update timing, or service availability. Forecast values may differ from measured jobsite conditions, provider/model, forecast grid, and retrieval time.
- Crane-specific engineering or operating decisions, including boom/load-height wind estimation, crane configuration, rigging, load wind area, lift calculations, and certification. The 10 m forecast is not a crane-height measurement.
- Concrete mix design, concrete/substrate temperature, curing or protection modeling, strength/maturity calculations, and high-temperature concreting rules.
- Roofing-material-specific rules, precipitation amount/duration/type analysis, substrate conditions, and other hazards not represented by hourly precipitation probability.
- Safety certification, OSHA compliance certification, or replacement of approved project documents, manufacturer instructions, competent-person judgment, or site safety procedures. Go/Caution/No-Go labels are planning indicators only.
- Paid weather services requiring API keys, proprietary radar/telematics feeds, or a live backend. The application remains a static single-file `index.html` suitable for static hosting. Forecast access requires the Open-Meteo network service; an interactive map service/library is required, but its provider remains to be selected. Open-Meteo receives the jobsite coordinates; map-service requests may reveal the viewed area, depending on the selected provider.

## Features
*(Add at least two custom features here—with at least one being your own original idea—after the must-haves pass validation and before building those features.)*
- **Feature 1 (e.g., Crew working hours filter):** `[Define after validation]`
- **Feature 2 (Original idea):** `[Define after validation]`

## Build stages
*(Populate in Plan mode after the seven sections are complete, then build one stage at a time starting with the page layout mockup.)*