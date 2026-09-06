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

## Connecting Grafana to the LeapMotor Mate database

Grafana needs **read access to the same `leapmotor_mate.db` file** LeapMotor
Mate itself writes to. There's no API layer involved — the `frser-sqlite-datasource`
plugin opens the SQLite file directly. Two setups, depending on where your
Grafana lives:

### Option A — Grafana in the same `docker-compose.yml` as LeapMotor Mate

Add a `grafana` service to LeapMotor Mate's own compose file and mount its data
volume (or the specific `.db` file) into it, **read-only**:

```yaml
services:
  leapmotor-mate:
    # ... your existing service ...
    volumes:
      - ./data:/data

  grafana:
    image: grafana/grafana:11.3.0
    restart: unless-stopped
    ports:
      - "3030:3000"
    environment:
      - GF_INSTALL_PLUGINS=frser-sqlite-datasource
      - GF_PLUGINS_ALLOW_LOADING_UNSIGNED_PLUGINS=frser-sqlite-datasource
    volumes:
      - ./data/leapmotor_mate.db:/mate-data/leapmotor_mate.db:ro
      - ./grafana-dashboards/provisioning:/etc/grafana/provisioning
      - ./grafana-dashboards/dashboards:/etc/grafana/provisioning/dashboards/leapmotor-mate
      - grafana-data:/var/lib/grafana

volumes:
  grafana-data:
```

Clone this repo next to your compose file (e.g. as `grafana-dashboards/`) so the
two bind mounts above resolve, then `docker compose up -d grafana`.

### Option B — Grafana already running elsewhere (e.g. reused from another project)

If you already have a Grafana container running for something else, you only
need to add the bind mount for the `.db` file and the two provisioning folders
to its existing `docker-compose.yml`/`docker run` command — same three lines as
in Option A's `grafana` service — then restart just that container. The two
containers don't need to be on the same Docker network or share anything beyond
that one file.

Either way:

1. **Mount read-only (`:ro`)**. Grafana only ever runs `SELECT` queries, and the
   `:ro` flag guarantees it, so there's no risk of it corrupting LeapMotor
   Mate's live database.
2. **Confirm LeapMotor Mate uses WAL journal mode** (it does by default in
   recent SQLite/Node setups) so a concurrent reader never blocks or gets a
   `database is locked` error while Mate is writing:
   ```bash
   sqlite3 /path/to/leapmotor_mate.db "PRAGMA journal_mode;"
   # should print: wal
   ```
3. **Install the plugin.** `frser-sqlite-datasource` is a community (unsigned)
   plugin, hence the `GF_PLUGINS_ALLOW_LOADING_UNSIGNED_PLUGINS` env var above —
   without it Grafana refuses to load it.
4. **Copy `provisioning/` and `dashboards/`** from this repo as shown in the
   compose snippet. On startup Grafana auto-creates the datasource (fixed
   `uid: P0941905C0D8205B6`, matching the `datasource.uid` referenced inside
   every dashboard JSON here) and imports all 9 dashboards into a "LeapMotor
   Mate" folder — nothing to click through in the UI.
5. **Verify.** Open Grafana → *Connections → Data sources → LeapMotor Mate
   SQLite* → **Save & test**; it should report success. Then open any dashboard
   in the "LeapMotor Mate" folder — if you see real numbers, the mount and
   plugin are both working.

### Manual import (no provisioning)

If you'd rather not use file provisioning: add the `frser-sqlite-datasource`
datasource manually through the UI (any name, any UID) pointing its `path`
setting at the mounted `.db` file, then import each `dashboards/*.json` file via
**Dashboards → New → Import**, selecting your datasource when prompted for each.

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
