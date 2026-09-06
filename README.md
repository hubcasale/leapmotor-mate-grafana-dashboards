# LeapMotor Mate — Custom Grafana Dashboards

A set of Grafana dashboards for [LeapMotor Mate](https://github.com/) that read
**directly from its SQLite database** — no InfluxDB, no ETL, no data duplication.

Structure and panel design are modeled after
[jheredianet/Teslamate-CustomGrafanaDashboards](https://github.com/jheredianet/Teslamate-CustomGrafanaDashboards),
adapted to the tables and fields LeapMotor Mate actually exposes (`charges`, `trips`,
`positions`, `vehicles`, `offline_gaps`).

## Dashboards

| File | Title | Description |
|---|---|---|
| `CurrentState.json` | Stato Attuale | Live vehicle status: battery, range, temperature, last charge/trip, state timeline |
| `ChargingCostsStats.json` | Ricariche | Charging cost/energy stats, AC/DC split, top stations, monthly breakdown |
| `MileageStats.json` | Viaggi | Trip distance/efficiency stats, monthly mileage, long trips |
| `IncompleteData.json` | Qualità Dati | Data-quality diagnostics: incomplete records, tracking gaps, open trips/charges |
| `ChargingCurveStats.json` | Curva di Ricarica | Power/SOC curve for a selected charging session |
| `SpeedRatesAndTemperature.json` | Velocità e Temperatura | Efficiency vs. average speed and vs. outside temperature |
| `TrackingMaps.json` | Mappe | Trip route map, charging locations, recent trip starting points |
| `AmortizationTracker.json` | Ammortamento | Cumulative charging cost vs. an equivalent combustion-car fuel cost, break-even tracking |
| `CurrentChargeView.json` | Ricarica Attuale | Auto-refreshing detail view of the most recent charging session |

All dashboards that browse historical records (Ricariche, Viaggi, Velocità e
Temperatura, Curva di Ricarica) respect Grafana's global time-range picker.
`Stato Attuale` and `Qualità Dati` intentionally ignore it — the former always shows
the latest known state, the latter scans the whole dataset for anomalies.

## Prerequisites

- A Grafana instance (tested on 11.x) with the
  [`frser-sqlite-datasource`](https://github.com/fr-ser/grafana-sqlite-datasource)
  community plugin installed:
  ```
  GF_INSTALL_PLUGINS=frser-sqlite-datasource
  ```
- The LeapMotor Mate SQLite database file mounted **read-only** into the Grafana
  container.

## Setup

1. Mount the database into the Grafana container, e.g. in `docker-compose.yml`:
   ```yaml
   services:
     grafana:
       image: grafana/grafana:11.3.0
       environment:
         - GF_INSTALL_PLUGINS=frser-sqlite-datasource
         - GF_PLUGINS_ALLOW_LOADING_UNSIGNED_PLUGINS=frser-sqlite-datasource
       volumes:
         - /path/to/leapmotor-mate/data/leapmotor_mate.db:/mate-data/leapmotor_mate.db:ro
         - ./provisioning:/etc/grafana/provisioning
         - ./dashboards:/etc/grafana/provisioning/dashboards/leapmotor-mate
   ```
2. Copy `provisioning/` and `dashboards/` from this repo next to your
   `docker-compose.yml` as shown above (the datasource provisioning file expects
   the dashboards to be mounted at
   `/etc/grafana/provisioning/dashboards/leapmotor-mate`).
3. Start Grafana. The datasource and all dashboards are provisioned automatically
   into a "LeapMotor Mate" folder.

### Manual import (no provisioning)

If you'd rather not use file provisioning, add the `frser-sqlite-datasource`
datasource manually (any name, any UID) pointing at the mounted `.db` file, then
import each `dashboards/*.json` file via **Dashboards → New → Import**, selecting
your datasource when prompted.

## Template variables

- `charge_id` / `trip_id` — session/trip picker (Curva di Ricarica, Mappe)
- `period` — `Mensile` / `Annuale` bucket size (Ammortamento)
- `extra_cost`, `fuel_price_km` — assumptions you set yourself for the
  amortization comparison (purchase price delta vs. a combustion car, equivalent
  fuel cost per km); these are not derived from any real data, matching how the
  reference TeslaMate dashboard also relies on user-supplied cost assumptions.

## Notes

- No personal data (VIN, GPS history, credentials, real hostnames) is stored in
  these dashboard definitions — they are query/layout definitions only. Actual
  data lives in your own `leapmotor_mate.db`, which is never committed here.
- `RangeDegradation` (battery state-of-health) from the reference repo is
  intentionally not included — TeslaMate computes it with its own tested Python
  algorithm; reimplementing that in raw SQL was judged too risky to ship.

## License

MIT — see [LICENSE](LICENSE).
