# Stage 5 Validation Record

## Captured Open-Meteo fixture

- Site: Provo, Utah area
- Coordinates: lat `40.2338`, lon `-111.6585`
- Open-Meteo response coordinates: `40.230103`, `-111.65572`
- Jobsite timezone: `America/Denver`
- UTC offset: `-25200` seconds (`GMT-7`)
- Target local date: `2026-11-23`
- Forecast retrieval display time: `Nov 23, 2026, 2:38 PM`
- Elevation: `1392 m`
- Exact request: `https://api.open-meteo.com/v1/forecast?latitude=40.2338&longitude=-111.6585&hourly=temperature_2m%2Cwind_speed_10m%2Cwind_gusts_10m%2Cprecipitation_probability&forecast_days=7&timezone=auto&timeformat=unixtime&temperature_unit=fahrenheit&wind_speed_unit=mph`
- Units: `temperature_2m` °F, `wind_speed_10m` mph, `wind_gusts_10m` mph, `precipitation_probability` %

The evaluated window is the current partial day, 2026-11-23 from 3:00 PM MST through 11:00 PM MST. Pre-current-hour samples are excluded.

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

## Hand calculation

Configured thresholds:

- `W_max = 20.0 mph`
- `G_nogo = min(W_max + 5 mph, 1.25 × W_max) = 25 mph`
- Sustained caution floor = `W_max - 5 mph = 15 mph`
- `T_caut = 40.0 °F`
- `T_nogo = 32.0 °F`
- `P_caut = 30%`
- `P_nogo = 60%`

### Crane Lifts

- Maximum sustained wind = `6.5 mph` at 2026-11-23 16:00 MST
- Maximum gust = `8.7 mph` at 2026-11-23 16:00 MST
- `6.5 mph < 20 mph` sustained; `8.7 mph < 25 mph` gust
- Neither sustained nor gust No-Go condition applies
- Status: **Go**

### Concrete Pours

- Minimum temperature = `31.8 °F` at 2026-11-23 23:00 MST
- `31.8 °F <= 32.0 °F`
- Status: **No-Go**

### Roofing

- Maximum hourly precipitation probability = `2%` at 2026-11-23 15:00 MST
- `2% < 30%`
- Status: **Go**

## App actual output

- Crane Lifts: **Go**; sustained `6.5 mph` and gust `8.7 mph`, both at 16:00 MST
- Concrete Pours: **No-Go**; minimum `31.8 °F` at 23:00 MST
- Roofing: **Go**; maximum hourly precipitation probability `2%` at 15:00 MST

The app displays the raw hourly values, Unix timestamps, local timestamps, UTC offsets, governing values and units, thresholds, tied timestamps, and exact reason strings. The validation details panel also shows the exact request and returned source metadata.

## NWS contextual comparison

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

The sources use different grids and retrieval times and are not required to match exactly. NWS provides sustained wind as hourly text and does not provide a gust field. The comparison is contextual, not a pass/fail oracle for Open-Meteo arithmetic.
