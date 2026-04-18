---
name: vehicle-availability
description: >
  Build a Vehicle Availability Dashboard for EV fleets. Use this skill whenever the user needs to:
  visualize vehicle availability by location and time slot, process driver shift data (planned vs adjusted vs actual),
  identify shortage windows across charging stations, generate a 15-minute interval heatmap dashboard,
  synthesise multi-day estimates, forecast vehicle availability for future dates, explore no-show patterns,
  display vehicle parking locations per plate, show driver pickup point assignments,
  track maintenance vehicles excluded from service, or publish the dashboard to GitHub Pages.
  Triggers include: "vehicle availability", "shortage forecast", "charging station inventory",
  "driver shift dashboard", "fleet availability heatmap", "multi-day dashboard", "vehicle locations tab",
  "driver pickup points", "maintenance vehicles", "publish dashboard", or when the user mentions
  hopAreaLetter, drivershifts, vehicleMainLocation, driverLocation, or cannoyUsedVehicles datasets.
---

# Vehicle Availability Dashboard Skill

Build a self-contained interactive HTML dashboard with **three tabs**:
1. **📊 Availability** — 15-minute interval vehicle availability heatmap, shortage KPIs (including 🔧 maintenance count), trend chart, multi-day navigation
2. **🚗 Vehicle Locations** — per-plate parking locations from `vehicleMainLocation_*.csv` (list + location summary, with maintenance badges)
3. **👤 Drivers** — driver pickup point assignments from `driverLocation_*.xlsx` (sortable, filterable table)

## Quick reference — dataset roles

| Dataset | Naming pattern | Key columns | Dashboard tab |
|---|---|---|---|
| `vehicleMainLocation_{YYYY-MM-DD}.csv` | Date in filename | `plate`, `Area`, `Main_Location`, `StatusInFleet` | Availability + 🚗 Vehicles |
| `driverLocation_{YYYY}_week{NN}.xlsx` | Year + week | `driverId`, `driverCode`, `display_name_th`, `Rank_1/2/3`, `employment_type`, `shift_type`, `updated_at` (sheet: DriverLocation) | Availability + 👤 Drivers |
| `drivershifts_{YYYY-MM-DD}.xlsx` | Date in filename | `driverId`, `hopAreaLetter`, `startAt`, `endAt`, `clock_in_at`, `clock_out_at`, `is_deleted` (sheet: Sheet1) | Availability only |
| `cannoyUsedVehicles.xlsx` | Fixed path | `ทะเบียนรถ`, `ซ่อมแล้ว`, `รับลูกค้าได้ไหม` (sheet: Maintenance Queue) | All tabs (exclusion) |

## Running the pipeline

**Folder mode — recommended for daily use** (auto-discovers newest files in each folder):
```bash
python3 process_vehicle_data.py \
    --stock-dir   "G:/My Drive/MuvMi/BaselineAnalysis/vehiclePrototype/vehicleMainLocation" \
    --drivers-dir "G:/My Drive/MuvMi/BaselineAnalysis/vehiclePrototype/driverlocation" \
    --shifts-dir  "G:/My Drive/MuvMi/BaselineAnalysis/DriverShiftRecord"
```

The script **automatically splits** shift files by today's date:
- Files with dates **before today** → treated as real data (have clock-in/out records)
- Files with dates **on or after today** → treated as shifts-only (assignments, no clocks)

If today's and future shift files live in a different folder, pass `--future-shifts-dir`:
```bash
python3 process_vehicle_data.py \
    --stock-dir         "G:/My Drive/.../vehicleMainLocation" \
    --drivers-dir       "G:/My Drive/.../driverlocation" \
    --shifts-dir        "G:/My Drive/.../DriverShiftRecord" \
    --future-shifts-dir "G:/My Drive/.../FutureShifts"
```

**Explicit file mode** (specify each file directly):
```bash
python3 process_vehicle_data.py \
    --stock          vehicleMainLocation_2026-04-04.csv \
    --drivers        driverLocation_week13.xlsx \
    --shifts         drivershifts_2026-04-03.xlsx drivershifts_2026-04-04.xlsx \
    --future-shifts  drivershifts_2026-04-12.xlsx drivershifts_2026-04-13.xlsx \
    --today   2026-04-12 --days 14 --out vehicle_data_multiday.json
```

| Folder argument | Auto-picks |
|---|---|
| `--stock-dir` | `vehicleMainLocation_*.csv` with the **newest date** in the filename |
| `--drivers-dir` | `driverLocation_*.xlsx` with the **highest week number** |
| `--shifts-dir` | All `drivershifts_YYYY-MM-DD.xlsx` in window; **auto-split by today** |
| `--future-shifts-dir` | Same scan, merged into the future half of the split (optional) |

Both modes can be mixed. New shift files added to the folder are picked up automatically on the next run.

**Running from a Jupyter notebook** — import `runVehicleDashboard()` directly instead of using the CLI.
Jupyter injects its own values into `sys.argv`, which causes `argparse` to error with `SystemExit: 2`.

```python
from process_vehicle_data import runVehicleDashboard

# Minimal call — html_out, github_*, maintenance_file all come from CONFIG automatically
runVehicleDashboard(
    stock_dir   = r"G:\My Drive\MuvMi\BaselineAnalysis\vehiclePrototype\vehicleMainLocation",
    drivers_dir = r"G:\My Drive\MuvMi\BaselineAnalysis\vehiclePrototype\driverlocation",
    shifts_dir  = r"G:\My Drive\MuvMi\BaselineAnalysis\DriverShiftRecord",
)
# Builds the HTML, pushes index.html + README.md + skill/ to GitHub — all from CONFIG

# Full explicit call (any parameter overrides the corresponding CONFIG value)
runVehicleDashboard(
    stock_dir        = r"G:\My Drive\MuvMi\BaselineAnalysis\vehiclePrototype\vehicleMainLocation",
    drivers_dir      = r"G:\My Drive\MuvMi\BaselineAnalysis\vehiclePrototype\driverlocation",
    shifts_dir       = r"G:\My Drive\MuvMi\BaselineAnalysis\DriverShiftRecord",
    maintenance_file = r"G:\My Drive\MuvMi\BaselineAnalysis\cannoyUsedVehicles.xlsx",
    github_repo      = r"C:\Users\Tawit\OneDrive\Documents\Claude\Projects\muvmi-vehicle-availability-dashboard",
    github_filename  = "index.html",
    github_readme    = r"C:\Users\...\vehicle-availability-dashboard_SKILL.md",
    github_skills_dir= r"C:\Users\...\Dashboard for Vehicle Management\vehicle-availability-skill",
)
# Returns a compact summary dict (counts only) — does NOT dump driver/vehicle lists to Jupyter output
```

### CONFIG block — key settings

```python
CONFIG = {
    'today':            None,      # None = use actual today; d < today → REAL dates, d >= today → SHIFT/FCST
                                   # IMPORTANT: do NOT set to yesterday — None already makes yesterday the last REAL date
    'days':             29,        # total window (14 past + today + 14 future)
    'out':              'vehicle_availability_multiday.html',
    'github_repo':       r"C:\Users\Tawit\OneDrive\Documents\Claude\Projects\muvmi-vehicle-availability-dashboard",
    'github_filename':   'index.html',
    'github_readme':     r"C:\Users\...\vehicle-availability-dashboard_SKILL.md",  # auto-pushed as README.md
    'github_skills_dir': r"C:\Users\...\Dashboard for Vehicle Management\vehicle-availability-skill",
    # ^ entire vehicle-availability-skill/ folder is mirrored to skill/ in the repo on every push
    'maintenance_file':  r"G:\My Drive\MuvMi\BaselineAnalysis\cannoyUsedVehicles.xlsx",
    'password':          'muvmi2024',   # dashboard password (set in HTML constant DASHBOARD_PASSWORD)
}
```

### Reference date logic

`discover_shifts_files` uses a **strict less-than** comparison: `d < today`.

- Dates **before today** → `real_shifts` (have clock-in/out records)
- Dates **on or after today** → `future_shifts` (assignments only, no clocks)

If today is Apr 13, then Apr 12 is the last REAL date. Setting `today=None` (actual system date)
automatically makes yesterday the last REAL date. **Do NOT set `today` to yesterday** — that would
make yesterday appear as a SHIFT date instead of REAL.

Read `references/data-pipeline.md` for the full step-by-step data processing guide, including edge cases.

---

## Core concepts

### 15-minute slots
- 96 slots per day (slot 0 = 00:00, slot 95 = 23:45)
- `slot = hour × 4 + minute // 15`
- Reverse: `HH:MM = slot//4 : (slot%4)*15`
- Operational window for KPI calculation: slots 20–88 (05:00–22:00)
- Dashboard display starts at slot 20 (05:00); slots 0–19 are hidden

### Three inventory modes
| Mode | Departure column | Return column | Purpose |
|---|---|---|---|
| `planned` | `startAt` | `endAt` | Scheduled plan; weight = 1.0 |
| `adjusted` | `startAt` | `endAt` | No-show corrected; weight = per-area show_rate |
| `actual` | `clock_in_at` | `clock_out_at` | Real observed; NaT clock_in = no-show skipped |

### Areas to include
Filter shifts to `hopAreaLetter` in: `A, B, E, F, G, I, L, P, Q, W`

| Letter | Area name |
|---|---|
| A | Chula-Chidlom |
| B | Ari-Phayathai |
| E | Rattanakosin |
| F | Sukhumvit |
| G | Phahol-Kaset |
| I | Onnut-Bearing |
| L | Bangsue |
| P | Silom |
| Q | Ratchada-Rama9 |
| W | Wongwian Yai |

### Default charging stations (fallback when Rank_1 is NaN or unmapped)
| Area | Default station |
|---|---|
| A | จุดจอดจันทนยิ่งยง |
| B | MH33 |
| E | ตลาดสะพานคู่ - คลองสาน |
| F | MH Rama 9 |
| G | MH33 |
| I | MH Rama 9 |
| L | MH33 |
| P | MH Rama 3 |
| Q | MH Rama 9 |
| W | ตลาดสะพานคู่ - คลองสาน |

---

## Data pipeline summary

### Step 1 — Load maintenance vehicles (new)
`load_maintenance_vehicles(path)` returns a **set of plate strings** that are currently under maintenance
and cannot serve customers (`ซ่อมแล้ว=FALSE AND รับลูกค้าได้ไหม='ไม่ได้'`):

```python
maintenance_plates = load_maintenance_vehicles(CONFIG['maintenance_file'])
# Returns set() if path is None or file not found (graceful degradation)
```

The function tries sheet `'Maintenance Queue'` first, then falls back to sheet index 0.
Missing file prints a WARNING but does not crash the pipeline.

### Step 1b — Load overnight stock
`load_stock()` returns **five** values — the extra two feed the Vehicles tab:
```python
loc_stock, areas_served, area_loc_map, vehicles, stock_date = load_stock(path)
# vehicles   : [{plate, area, areaName, location, status}, ...] — ALL rows (not just InFleet)
# stock_date : "YYYY-MM-DD" extracted from filename
```

```python
import pandas as pd

AREA_NAME_TO_LETTER = {
    'Chula-Chidlom': 'A', 'Ari-Phayathai': 'B', 'Rattanakosin': 'E',
    'Sukhumvit': 'F',      'Phahol-Kaset': 'G', 'Onnut-Bearing': 'I',
    'Bangsue': 'L',        'Silom': 'P',         'Ratchada': 'Q',
    'Wongwianyai': 'W',
}
AREA_KEEP = {'A','B','E','F','G','I','L','P','Q','W'}

stock_df = pd.read_csv('vehicleMainLocation_{date}.csv')
stock_df = stock_df[stock_df['StatusInFleet'] == 'InFleet'].copy()
stock_df = stock_df.dropna(subset=['Main_Location'])         # drop NaN locations
stock_df['area_letter'] = stock_df['Area'].map(AREA_NAME_TO_LETTER)
stock_df = stock_df[stock_df['area_letter'].isin(AREA_KEEP)]

loc_stock    = stock_df.groupby('Main_Location')['plate'].count().to_dict()
areas_served = {}   # loc → {area_letter: count}
for _, row in stock_df.iterrows():
    loc = row['Main_Location']
    a   = row['area_letter']
    areas_served.setdefault(loc, {}).setdefault(a, 0)
    areas_served[loc][a] += 1

area_loc_map = {}
for loc, ac in areas_served.items():
    for a in ac:
        area_loc_map.setdefault(a, []).append(loc)
for a in area_loc_map:
    area_loc_map[a].sort(key=lambda l: -loc_stock[l])
```

### Step 2 — Resolve pickup locations
`load_driver_locations()` returns **three** values — the extra two feed the Drivers tab:
```python
drv_loc, drivers_list, drivers_week = load_driver_locations(path)
# drv_loc      : {driverId: Rank_1}  — unchanged dict used by pipeline
# drivers_list : [{driverId, driverCode, nameTh, heroId, rank1, rank2, rank3,
#                  employmentType, shiftType, updatedAt}, ...]
# drivers_week : "2026 Week 13"
```

```python
DEFAULT_STATIONS = {
    'A': 'จุดจอดจันทนยิ่งยง', 'B': 'MH33',
    'E': 'ตลาดสะพานคู่ - คลองสาน', 'F': 'MH Rama 9',
    'G': 'MH33', 'I': 'MH Rama 9', 'L': 'MH33',
    'P': 'MH Rama 3', 'Q': 'MH Rama 9',
    'W': 'ตลาดสะพานคู่ - คลองสาน',
}

drv_df = pd.read_excel('driverLocation_week{N}.xlsx', sheet_name='DriverLocation')
drv_loc = {}
for _, row in drv_df.iterrows():       # explicit loop — avoids KeyError on NaN driverId
    did = row['driverId']
    if pd.notna(did):
        drv_loc[did] = row['Rank_1']

def resolve_pickup(driver_id, area_letter):
    loc = drv_loc.get(driver_id)
    if pd.isna(loc) or not loc or loc not in loc_stock:
        return DEFAULT_STATIONS.get(area_letter, '')
    return loc
```

### Step 3 — Load and filter shifts
```python
shifts_df = pd.read_excel('drivershifts_{date}.xlsx', sheet_name='Sheet1')
shifts_df = shifts_df[
    shifts_df['hopAreaLetter'].isin(AREA_KEEP) &
    (shifts_df['is_deleted'] == False)
].copy()

for col in ['startAt', 'endAt', 'clock_in_at', 'clock_out_at']:
    shifts_df[col] = pd.to_datetime(shifts_df[col], errors='coerce', utc=True)
    shifts_df[col] = shifts_df[col].dt.tz_localize(None)

shifts_df['pickup_location'] = shifts_df.apply(
    lambda r: resolve_pickup(r['driverId'], r['hopAreaLetter']), axis=1
)
```

### Step 4 — Compute per-area show rates
```python
def compute_show_rates(shifts_df):
    """Fraction of shifts where clock_in_at is NOT NaT, per hopAreaLetter."""
    rates = {}
    for area, grp in shifts_df.groupby('hopAreaLetter'):
        total  = len(grp)
        showed = grp['clock_in_at'].notna().sum()
        rates[area] = round(showed / total, 4) if total > 0 else 1.0
    return rates

show_rates = compute_show_rates(shifts_df)
```

### Step 5 — Build inventory arrays (cumulative delta, three modes)

`build_inventory()` now accepts an optional `maint_stock` parameter — a per-location count of
maintenance vehicles. These are subtracted from `loc_stock` before computing the arrays so that
maintenance vehicles never appear as available to drivers.

```python
SLOTS = 96

def dt_to_slot(dt):
    if pd.isna(dt): return None
    return int(dt.hour) * 4 + int(dt.minute) // 15

def build_inventory(shifts_df, loc_stock, show_rates, mode='planned', maint_stock=None):
    """
    mode='planned'  → startAt/endAt,         weight=1.0   (full planned schedule)
    mode='adjusted' → startAt/endAt,         weight=show_rate (fractional subtraction)
    mode='actual'   → clock_in_at/clock_out, weight=1.0   (NaT = no-show, skip)
    maint_stock     → {loc: int} maintenance count subtracted from effective starting stock
    """
    maint_stock = maint_stock or {}
    # Effective stock = raw stock minus maintenance vehicles (floor 0)
    eff_stock = {loc: max(0.0, float(s) - float(maint_stock.get(loc, 0)))
                 for loc, s in loc_stock.items()}

    inv   = {loc: [eff_stock[loc]] * SLOTS for loc in loc_stock}
    delta = {loc: [0.0]            * SLOTS for loc in loc_stock}

    dep_col = 'clock_in_at'  if mode == 'actual' else 'startAt'
    ret_col = 'clock_out_at' if mode == 'actual' else 'endAt'

    for _, row in shifts_df.iterrows():
        loc  = row['pickup_location']
        area = row['hopAreaLetter']
        if loc not in loc_stock: continue
        dep = dt_to_slot(row[dep_col])
        ret = dt_to_slot(row[ret_col])
        if dep is None: continue           # no-show or NaT → skip
        if ret is None: ret = 95
        w = show_rates.get(area, 1.0) if mode == 'adjusted' else 1.0
        delta[loc][dep] -= w
        if ret < SLOTS: delta[loc][ret] += w

    for loc in loc_stock:
        s = eff_stock[loc]   # start from effective stock, not raw stock
        for i in range(SLOTS):
            s = max(0.0, s + delta[loc][i])
            inv[loc][i] = round(s, 1)
    return inv

# maint_stock is computed inside build_multiday_json():
#   maint_stock = {v['location']: count for v in vehicles if v['plate'] in maintenance_plates}
planned_inv  = build_inventory(shifts_df, loc_stock, show_rates, mode='planned',  maint_stock=maint_stock)
adjusted_inv = build_inventory(shifts_df, loc_stock, show_rates, mode='adjusted', maint_stock=maint_stock)
actual_inv   = build_inventory(shifts_df, loc_stock, show_rates, mode='actual',   maint_stock=maint_stock)
```

### Step 6 — Multi-day synthesis (days without real shift files)
```python
WEEKEND_SCALE = 0.75   # scale down Saturday/Sunday

def synthesise_day(template_inv, date_str, scale=1.0):
    """Return {loc: {planned, adjusted, actual:None}} scaled from template."""
    from datetime import datetime
    weekday = datetime.strptime(date_str, '%Y-%m-%d').weekday()
    sc = WEEKEND_SCALE if weekday >= 5 else scale
    out = {}
    for loc in template_inv['planned']:
        out[loc] = {
            'planned':  [round(v * sc, 1) for v in template_inv['planned'][loc]],
            'adjusted': [round(v * sc, 1) for v in template_inv['adjusted'][loc]],
            'actual':   None,
        }
    return out

# Rolling show rates = average across all real-data days
def rolling_show_rates(all_real_rates):
    """Average show rates across all real-data days, per area."""
    combined = {}
    for day_rates in all_real_rates:
        for area, rate in day_rates.items():
            combined.setdefault(area, []).append(rate)
    return {a: round(sum(v)/len(v), 4) for a, v in combined.items()}
```

### Step 7 — Assemble multi-day JSON output
```python
output = {
    "today":   today_str,          # "YYYY-MM-DD" of the report run date
    "dates":   all_dates,          # list of date strings, oldest → newest
    "dailySummary": {
        date: {
            "has_data":      bool,   # True for past dates with real clock-in/out records
            "has_shifts":    bool,   # True for today+future with real assignments, no clocks yet
            "is_future":     bool,   # True for dates strictly after today
            "planned_gaps":  int,    # slots 20–88 where planned <= 0
            "adjusted_gaps": int,
            "actual_gaps":   int | None,   # None when has_data=False
            "show_rates":    {area: float, ...}
        }
        for date in all_dates
    },
    "locations": [
        {
            "name":        str,
            "primaryArea": str,      # area with most vehicles at this location
            "color":       str,      # hex colour for primary area
            "stock":       int,      # overnight vehicle count (raw, before maintenance subtraction)
            "areasServed": {area: count, ...},
            "days": {
                date: {
                    "planned":  [float] * 96,   # arrays already reflect maint_stock subtraction
                    "adjusted": [float] * 96,
                    "actual":   [float] * 96 | None
                }
                for date in all_dates
            }
        }
    ],
    "areaLocations": {area: [loc_name, ...], ...},  # sorted by stock desc
    # ── 🚗 Vehicles tab ──────────────────────────────────────────
    "vehicles":  [{"plate": str, "area": str, "areaName": str,
                   "location": str, "status": str}, ...],  # ALL rows from CSV
    "stockDate": "YYYY-MM-DD",    # from vehicleMainLocation filename
    # ── 👤 Drivers tab ───────────────────────────────────────────
    "drivers":  [{"driverId": str, "driverCode": str, "nameTh": str,
                  "heroId": str, "rank1": str, "rank2": str, "rank3": str,
                  "employmentType": str, "shiftType": str, "updatedAt": str}, ...],
    "driversWeek": "2026 Week 13",  # from driverLocation filename
    # ── 🔧 Maintenance ───────────────────────────────────────────
    "maintenanceVehicles": ["plate1", "plate2", ...],  # sorted list of maintenance plates
}
```

---

## Dashboard design

Read `references/dashboard-design.md` for the complete HTML/JS template and component details.

### Password gate

The dashboard is protected by a client-side password gate that appears on every page load.
Correct password is stored as a JS constant — change it directly in the HTML:

```js
const DASHBOARD_PASSWORD = 'muvmi2024';  // ← change this to update the password
const PW_SESSION_KEY     = 'muvmi_dash_auth';
```

Authentication state is stored in `sessionStorage` — users must re-enter the password each
browser session. The overlay has a shake animation on wrong password.

### GitHub Pages auto-publish

`git_push_dashboard()` copies the HTML, README, and skill docs to a local git repo clone,
stages everything, commits, and pushes. The `github_readme` file is copied as `README.md`;
the `github_skills_dir` folder is mirrored into `skill/` inside the repo:

```python
git_push_dashboard(
    html_src    = output_html_path,
    repo_dir    = CONFIG['github_repo'],      # local clone of GitHub Pages repo
    filename    = CONFIG['github_filename'],  # 'index.html' by default
    readme_src  = CONFIG['github_readme'],    # pushed as README.md
    skills_src  = CONFIG['github_skills_dir'],# mirrored to skill/ in the repo
)
```

Resulting repo structure after push:
```
index.html                  ← dashboard
README.md                   ← from github_readme
skill/
  SKILL.md
  references/
    data-pipeline.md
    dashboard-design.md
```

To set up GitHub Pages: go to repo Settings → Pages → Branch: `main`, folder: `/ (root)`.
The dashboard is then live at `https://{username}.github.io/{repo-name}/`.
Skill docs are browsable at `https://{username}.github.io/{repo-name}/skill/SKILL.md`.

### Key data structures in JS
```js
const REAL_DATA    = { /* embedded JSON */ };
const LOCS         = REAL_DATA.locations;
const AREA_LOCS    = REAL_DATA.areaLocations;
const DATES        = REAL_DATA.dates;
const SUMMARY      = REAL_DATA.dailySummary;
const TODAY        = REAL_DATA.today;
const VEHICLES     = REAL_DATA.vehicles     || [];  // 🚗 Vehicles tab
const STOCK_DATE   = REAL_DATA.stockDate    || '';  // "YYYY-MM-DD" from CSV filename
const DRIVERS      = REAL_DATA.drivers      || [];  // 👤 Drivers tab
const DRIVERS_WEEK = REAL_DATA.driversWeek  || '';  // "YYYY Week NN"
const AREA_ORDER   = ['A','B','E','F','G','I','L','P','Q','W'];
const D_START      = 20;    // slot index for 05:00 — heatmap & chart start

// 🔧 Maintenance vehicles
const MAINTENANCE_VEHICLES = new Set(REAL_DATA.maintenanceVehicles || []);
function isMaint(plate) { return MAINTENANCE_VEHICLES.has(plate); }

// Per-location maintenance count (built from VEHICLES list + isMaint())
const MAINT_BY_LOC = {};
VEHICLES.forEach(v => {
    if (isMaint(v.plate) && v.location) {
        MAINT_BY_LOC[v.location] = (MAINT_BY_LOC[v.location] || 0) + 1;
    }
});
const TOTAL_MAINT = MAINTENANCE_VEHICLES.size;
```

### Day type classification
```js
function hasActual(date)  { return SUMMARY[date] && SUMMARY[date].has_data; }
function isFuture(date)   { return SUMMARY[date] && SUMMARY[date].is_future; }
function hasShifts(date)  { return SUMMARY[date] && SUMMARY[date].has_shifts; }
// Always use getDateType() — never inline the chain
function getDateType(date) {
    const s = SUMMARY[date];
    if (!s) return 'est';
    if (s.has_data)   return 'real';
    if (s.has_shifts) return 'shift';
    if (s.is_future)  return 'fcst';
    return 'est';
}
const DATE_TYPE_COLOR = { real:'#10b981', shift:'#06b6d4', fcst:'#6366f1', est:'#f59e0b' };
const DATE_TYPE_LABEL = { real:'REAL', shift:'SHIFT', fcst:'FCST', est:'EST' };
```

### Three-tab layout
```
Header (shared filters: area, threshold)
Tab nav: [📊 Availability] [🚗 Vehicle Locations] [👤 Drivers]
─────────────────────────────────────────────────
Tab 1 — Availability  (date nav, trend, KPI [incl 🔧 maintenance card], heatmap, detail chart, shortage table [incl 🔧 col], stock chart)
Tab 2 — Vehicle Locations  (4 stat cards: InFleet/🔧 Maintenance/Other/Locations, search+filters [incl Maintenance status filter], Vehicle List | Location Summary sub-tabs)
Tab 3 — Drivers  (stats, search+filters [incl Primary Pickup], sortable table: code, name, rank1/2/3, emp type, shift, updated)
```

### Heatmap rendering rule
Iterate `AREA_LOCS` (not `LOCS.area`). This makes shared locations appear under every area they serve:
```js
AREA_ORDER.forEach(area => {
    const locNames = AREA_LOCS[area] || [];
    locNames.forEach(name => {
        const d = loc.days[selectedDate];
        // 3 sub-rows per slot: d.planned[s], d.adjusted[s], d.actual[s]
        // d.actual may be null — render grey placeholder if so
        // Show SHARED badge if Object.keys(loc.areasServed).length > 1
    });
});
```

### Colour thresholds
```js
function cellCls(v, thresh) {
    if (thresh < 0) return v <= 0 ? 'c1' : 'c4';   // −1 = only flag zeros
    if (v <= 0)       return 'c1';   // red
    if (v <= thresh)  return 'c2';   // orange
    if (v <= thresh * 2.5) return 'c3';  // yellow
    return 'c4';                          // green
}
```

---

## No-show model notes

The `adjusted` inventory uses fractional departure weights (`show_rate` per area).
This is the expected-value estimate, but over-corrects at shared hub locations
(e.g. MH33, MH Rama 9) because all areas' no-show discounts apply to the same pool.
Present `adjusted` clearly as "optimistic lower bound" — actual typically falls
between `planned` and `adjusted` at hub locations.

---

## Key assumptions
- Drivers pick up AND return vehicles to the same `pickup_location`
- No-shows (NaT `clock_in_at`) → vehicle stays in overnight stock (no subtraction)
- `is_deleted == False` filter required before all calculations
- `driverLocation` week N-2 is used; if exact date column is absent, fall back to `Rank_1`
- Same `vehicleMainLocation` file is reused for all synthesised dates
- Stock underflow clamped at 0: `max(0.0, stock + delta)`
- Timezone-aware datetimes stripped before `dt_to_slot()` for consistency

---

## Reference files
- `references/data-pipeline.md` — Full pipeline with edge-case table, show-rate math, synthesis, folder auto-discovery, maintenance exclusion
- `references/dashboard-design.md` — HTML/JS template: password gate, date nav, trend chart, data banner, heatmap, detail chart, Chart.js sizing rules, maintenance UI components
