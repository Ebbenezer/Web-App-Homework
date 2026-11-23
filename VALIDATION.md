# Stage 5 Validation Record

## Captured Open-Meteo fixture

- Site: Provo, Utah area
- Coordinates: lat `40.2338`, lon `-111.6585`
- Timezone: `America/Denver`
- Target local date: `2026-11-23`
- Retrieval: approximately `2026-11-23 14:27 MDT`
- Exact request: `https://api.open-meteo.com/v1/forecast?latitude=40.2338&longitude=-111.6585&hourly=temperature_2m,wind_speed_10m,wind_gusts_10m,precipitation_probability&forecast_days=7&timezone=auto&timeformat=unixtime&temperature_unit=fahrenheit&wind_speed_unit=mph`
- Units: `temperature_2m` in °F, `wind_speed_10m` and `wind_gusts_10m` in mph, `precipitation_probability` in percent

The evaluated window is the current partial day, at or after the current time. Captured values below are for the 3:00 PM through 11:00 PM MST hours.

| Local timestamp | Unix timestamp | Sustained wind 10 m (mph) | Gust 10 m (mph) | Temperature 2 m (°F) | Hourly precipitation probability (%) |
|---|---:|---:|---:|---:|---:|
| 2026-11-23 15:00-07:00 | 1795471200 | 4.6 | 4.9 | 47.2 | 2 |
| 2026-11-23 16:00-07:00 | 1795474800 | 6.1 | 8.1 | 45.8 | 1 |
| 2026-11-23 17:00-07:00 | 1795478400 | 4.9 | 8.3 | 41.0 | 1 |
| 2026-11-23 18:00-07:00 | 1795482000 | 5.1 | 7.2 | 37.5 | 1 |
| 2026-11-23 19:00-07:00 | 1795485600 | 2.7 | 5.8 | 34.9 | 1 |
| 2026-11-23 20:00-07:00 | 1795489200 | 5.3 | 6.0 | 34.5 | 0 |
| 2026-11-23 21:00-07:00 | 1795492800 | 3.6 | 4.9 | 33.2 | 0 |
| 2026-11-23 22:00-07:00 | 1795496400 | 2.5 | 2.9 | 33.0 | 0 |
| 2026-11-23 23:00-07:00 | 1795500000 | 2.5 | 3.1 | 31.8 | 0 |

## Hand calculation

Configured thresholds:

- W_max = 20 mph
- G_nogo = min(20 + 5, 1.25 x 20) = 25 mph
- T_caut = 40 °F
- T_nogo = 32 °F
- P_caut = 30%
- P_nogo = 60%

### Crane Lifts

- Maximum sustained wind = 6.1 mph at 2026-11-23 16:00 MST
- Maximum gust = 8.3 mph at 2026-11-23 17:00 MST
- Neither sustained wind >= 20 mph nor gust >= 25 mph
- Sustained wind is below 15 mph and gust is <= 20 mph
- Status: **Go**

### Concrete Pours

- Minimum temperature = 31.8 °F at 2026-11-23 23:00 MST
- 31.8 <= 32
- Status: **No-Go**

### Roofing

- Maximum hourly precipitation probability = 2% at 2026-11-23 15:00 MST
- 2% < 30%
- Status: **Go**

This fixture confirms that the governing sustained and gust maxima occur at separate times and that the partial-day window excludes pre-current-hour samples.

## NWS contextual comparison

- NWS endpoint: `https://api.weather.gov/gridpoints/SLC/105,151/forecast/hourly`
- NWS grid: SLC 105,151
- Point timezone: `America/Denver`
- NWS retrieval context: generated `2026-11-23T21:29:26Z` (14:29 MDT); grid update `2026-11-23T18:31:28Z` (11:31 MDT)
- Matching window: 2026-11-23 15:00 through 23:00 MST

| Local interval | NWS temperature (°F) | NWS sustained wind (mph) | NWS precipitation probability (%) |
|---|---:|---:|---:|
| 15:00-16:00 | 46 | 2 | 1 |
| 16:00-17:00 | 47 | 2 | 1 |
| 17:00-18:00 | 45 | 2 | 1 |
| 18:00-19:00 | 43 | 2 | 1 |
| 19:00-20:00 | 42 | 2 | 1 |
| 20:00-21:00 | 39 | 2 | 1 |
| 21:00-22:00 | 37 | 1 | 0 |
| 22:00-23:00 | 36 | 1 | 0 |
| 23:00-24:00 | 35 | 1 | 0 |

The sources use different grids and retrieval times and are not required to match exactly. NWS provides forecast temperature, sustained wind, and precipitation probability for these intervals but does not provide the Open-Meteo gust field. NWS sustained wind is also an hourly text interval rather than the Open-Meteo 10 m numeric field. The NWS comparison is contextual only and is not a pass/fail oracle for the Open-Meteo arithmetic.
