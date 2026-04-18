# Data Pipeline Reference — Vehicle Availability Dashboard

## File inventory

| File pattern | Format | Sheet | Key columns | Dashboard use |
|---|---|---|---|---|
| `vehicleMainLocation_{YYYY-MM-DD}.csv` | CSV | — | `plate`, `Area`, `Main_Location`, `StatusInFleet` | Availability pipeline + 🚗 Vehicles tab |
| `driverLocation_{YYYY}_week{NN}.xlsx` | Excel | `DriverLocation` | `driverId`, `driverCode`, `display_name_th`, `hero_id`, `Rank_1`, `Rank_2`, `Rank_3`, `employment_type`, `shift_type`, `updated_at` | Availability pipeline + 👤 Drivers tab |
| `drivershifts_{YYYY-MM-DD}.xlsx` | Excel | `Sheet1` | `driverId`, `hopAreaLetter`, `startAt`, `endAt`, `clock_in_at`, `clock_out_at`, `is_deleted` | Availability pipeline only |
| `cannoyUsedVehicles.xlsx` | Excel | `Maintenance Queue` | `ทะเบียนรถ` (plate), `ซ่อมแล้ว` (bool), `รับลูกค้าได้ไหม` (str) | Maintenance exclusion — all tabs |

---

## Folder structure & auto-discovery

The pipeline supports a **folder mode** where each data type lives in its own folder.
Files are auto-discovered by naming convention — no need to specify individual paths.

```
G:/My Drive/MuvMi/BaselineAnalysis/
│
├── vehiclePrototype/
│   ├── vehicleMainLocation/         ← --stock-dir
│   │   ├── vehicleMainLocation_2026-04-03.csv
│   │   └── vehicleMainLocation_2026-04-04.csv   ← newest date used
│   │
│   └── driverlocation/              ← --drivers-dir
│       ├── driverLocation_2026_week12.xlsx
│       └── driverLocation_2026_week13.xlsx       ← highest week number used
│
└── DriverShiftRecord/               ← --shifts-dir
    ├── drivershifts_2026-04-03.xlsx
    ├── drivershifts_2026-04-04.xlsx
    └── drivershifts_2026-04-07.xlsx             ← all files in date window used
```

**From a terminal — folder mode:**
```bash
python3 process_vehicle_data.py \
    --stock-dir   "G:/My Drive/MuvMi/BaselineAnalysis/vehiclePrototype/vehicleMainLocation" \
    --drivers-dir "G:/My Drive/MuvMi/BaselineAnalysis/vehiclePrototype/driverlocation" \
    --shifts-dir  "G:/My Drive/MuvMi/BaselineAnalysis/DriverShiftRecord" \
    --out         "vehicle_data_multiday.json"
```

**From a terminal — explicit file mode:**
```bash
python3 process_vehicle_data.py \
    --stock   vehicleMainLocation_2026-04-04.csv \
    --drivers driverLocation_week13.xlsx \
    --shifts  drivershifts_2026-04-03.xlsx drivershifts_2026-04-04.xlsx \
    --today   2026-04-12 --days 14 --out vehicle_data_multiday.json
```

**From a Jupyter notebook** — import and call `runVehicleDashboard()` directly.
Do NOT use `%run` or the CLI args in a notebook — Jupyter's own `sys.argv` will
confuse `argparse` and cause a `SystemExit: 2` error.
The function returns a compact summary dict (counts only) — not the full driver/vehicle lists —
so Jupyter doesn't flood the output cell.

All parameters default to `None` and fall back to the CONFIG block at the top of
`process_vehicle_data.py`. In practice you only need to pass the data folder paths
from Jupyter — `html_out`, `github_repo`, `github_skills_dir`, etc. are read from
CONFIG automatically:

```python
from process_vehicle_data import runVehicleDashboard

# Minimal call — all output/GitHub/maintenance settings come from CONFIG
runVehicleDashboard(
    stock_dir   = r"G:\My Drive\MuvMi\BaselineAnalysis\vehiclePrototype\vehicleMainLocation",
    drivers_dir = r"G:\My Drive\MuvMi\BaselineAnalysis\vehiclePrototype\driverlocation",
    shifts_dir  = r"G:\My Drive\MuvMi\BaselineAnalysis\DriverShiftRecord",
)
# Builds HTML, pushes index.html + README.md + skill/ to GitHub — all from CONFIG

# Full explicit call (overrides CONFIG values)
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
```

CONFIG fallback order (inside `runVehicleDashboard()`):
1. Explicit parameter passed in the call → wins
2. `CONFIG` dict at top of script → fallback
3. Built-in default (`'index.html'`, `14` days, etc.) → last resort

**Auto-discovery rules:**
| Argument | Picks |
|---|---|
| `--stock-dir` | Most recent date in `vehicleMainLocation_YYYY-MM-DD.csv` filename |
| `--drivers-dir` | Highest week number in `driverLocation_*week{N}*.xlsx` filename |
| `--shifts-dir` | All `drivershifts_YYYY-MM-DD.xlsx` files whose date falls in the window |

Both modes can be mixed — e.g. use `--stock-dir` with `--drivers` and `--shifts`.

---

## Reference date logic

`discover_shifts_files()` splits by today using a **strict less-than** comparison (`d < today`):

- Dates **before today** → `real_shifts` dict (have clock-in/out records)
- Dates **on or after today** → `future_shifts` dict (assignments only, no clocks yet)

Setting `today=None` (the default) uses the actual system date. This means yesterday is
automatically the last REAL date, and today is the first SHIFT/FCST date.

**Do NOT manually set `today` to yesterday.** That would make yesterday appear as a SHIFT date
(assignments without clocks), which is incorrect — yesterday already has clock records.

---

## Step-by-step pipeline

### 0. Load maintenance vehicles (optional)

```python
def load_maintenance_vehicles(path: str) -> set:
    """
    Load plates currently under maintenance and unavailable for customers.
    Filters: ซ่อมแล้ว == False  AND  รับลูกค้าได้ไหม == 'ไม่ได้'

    Returns an empty set if path is None, file not found, or any read error.
    Prints a WARNING but never raises an exception.
    """
    if not path:
        return set()
    try:
        try:
            df = pd.read_excel(path, sheet_name='Maintenance Queue')
        except Exception:
            df = pd.read_excel(path, sheet_name=0)
        filtered = df[
            (df['ซ่อมแล้ว'] == False) &
            (df['รับลูกค้าได้ไหม'] == 'ไม่ได้')
        ]
        plates = set(filtered['ทะเบียนรถ'].dropna().astype(str).unique())
        print(f"Maintenance vehicles: {len(plates)} plates loaded")
        return plates
    except FileNotFoundError:
        print(f"WARNING: Maintenance file not found: {path} — no vehicles excluded")
        return set()
    except Exception as e:
        print(f"WARNING: Could not load maintenance file ({e}) — no vehicles excluded")
        return set()

maintenance_plates = load_maintenance_vehicles(CONFIG.get('maintenance_file'))
```

Computing per-location maintenance count (inside `build_multiday_json()`):
```python
maint_stock: dict[str, int] = {}
for v in vehicles:
    if v['plate'] in maintenance_plates and v['location']:
        maint_stock[v['location']] = maint_stock.get(v['location'], 0) + 1
# maint_stock is passed to every build_inventory() call
```

### 1. Overnight parking stock

```python
import pandas as pd, json
from datetime import datetime, timedelta

AREA_NAME_TO_LETTER = {
    'Chula-Chidlom': 'A', 'Ari-Phayathai': 'B', 'Rattanakosin': 'E',
    'Sukhumvit': 'F',      'Phahol-Kaset': 'G', 'Onnut-Bearing': 'I',
    'Bangsue': 'L',        'Silom': 'P',         'Ratchada': 'Q',
    'Wongwianyai': 'W',
}
# Note: 'Ratchada' and 'Wongwianyai' are the exact spellings in the Area column;
# display names are 'Ratchada-Rama9' and 'Wongwian Yai'

AREA_KEEP = {'A','B','E','F','G','I','L','P','Q','W'}
AREA_COLORS = {
    'A': '#6366f1', 'B': '#f59e0b', 'E': '#10b981', 'F': '#ef4444',
    'G': '#8b5cf6', 'I': '#ec4899', 'L': '#14b8a6', 'P': '#f97316',
    'Q': '#3b82f6', 'W': '#84cc16',
}

stock_df = pd.read_csv('vehicleMainLocation_{date}.csv')
stock_df = stock_df[stock_df['StatusInFleet'] == 'InFleet'].copy()
stock_df = stock_df.dropna(subset=['Main_Location'])    # REQUIRED: drop NaN locations
stock_df['area_letter'] = stock_df['Area'].map(AREA_NAME_TO_LETTER)
stock_df = stock_df[stock_df['area_letter'].isin(AREA_KEEP)]  # drops NaN mappings

# Total stock per physical location
loc_stock = stock_df.groupby('Main_Location')['plate'].count().to_dict()

# Area-specific vehicle counts per location (many-to-many)
areas_served = {}   # loc → {area_letter: count}
for _, row in stock_df.iterrows():
    loc = row['Main_Location']
    a   = row['area_letter']
    areas_served.setdefault(loc, {}).setdefault(a, 0)
    areas_served[loc][a] += 1

# Area → list of locations (sorted by vehicle count desc)
area_loc_map = {}
for loc, area_counts in areas_served.items():
    for a in area_counts:
        area_loc_map.setdefault(a, []).append(loc)
for a in area_loc_map:
    area_loc_map[a].sort(key=lambda l: -loc_stock[l])
```

---

### 2. Driver pickup location resolution

```python
DEFAULT_STATIONS = {
    'A': 'จุดจอดจันทนยิ่งยง',
    'B': 'MH33',
    'E': 'ตลาดสะพานคู่ - คลองสาน',
    'F': 'MH Rama 9',
    'G': 'MH33',
    'I': 'MH Rama 9',
    'L': 'MH33',
    'P': 'MH Rama 3',
    'Q': 'MH Rama 9',
    'W': 'ตลาดสะพานคู่ - คลองสาน',
}

# load_driver_locations() returns THREE values:
drv_loc, drivers_list, drivers_week = load_driver_locations('driverLocation_2026_week13.xlsx')
#   drv_loc      → {driverId: Rank_1_str}  used by pipeline (unchanged interface)
#   drivers_list → [{driverId, driverCode, nameTh, heroId, rank1, rank2, rank3,
#                    employmentType, shiftType, updatedAt}, ...]  → 👤 Drivers tab
#   drivers_week → "2026 Week 13"  → shown in tab header

# Implementation (supports both naming patterns):
def load_driver_locations(path):
    df = pd.read_excel(path, sheet_name='DriverLocation')
    def safe(row, col):
        v = row.get(col); return str(v) if (v is not None and pd.notna(v)) else ''
    drv_loc = {}
    drivers_list = []
    for _, row in df.iterrows():
        did = row.get('driverId')
        if pd.notna(did):
            drv_loc[did] = row.get('Rank_1') if pd.notna(row.get('Rank_1', None)) else None
        drivers_list.append({
            'driverId': safe(row,'driverId'), 'driverCode': safe(row,'driverCode'),
            'nameTh':   safe(row,'display_name_th'), 'heroId': safe(row,'hero_id'),
            'rank1': safe(row,'Rank_1'), 'rank2': safe(row,'Rank_2'), 'rank3': safe(row,'Rank_3'),
            'employmentType': safe(row,'employment_type'), 'shiftType': safe(row,'shift_type'),
            'updatedAt': safe(row,'updated_at'),
        })
    m = re.search(r'(\d{4}).*?week\s*(\d+)', os.path.basename(path), re.IGNORECASE)
    drivers_week = f"{m.group(1)} Week {int(m.group(2))}" if m else ''
    return drv_loc, drivers_list, drivers_week

def resolve_pickup(driver_id, area_letter):
    loc = drv_loc.get(driver_id)
    if pd.isna(loc) or not loc or loc not in loc_stock:
        return DEFAULT_STATIONS.get(area_letter, '')
    return loc
```

---

### 3. Shift loading and filtering

```python
shifts_df = pd.read_excel('drivershifts_{date}.xlsx', sheet_name='Sheet1')

# Required filters (order matters — filter before resolving pickups)
shifts_df = shifts_df[
    shifts_df['hopAreaLetter'].isin(AREA_KEEP) &
    (shifts_df['is_deleted'] == False)
].copy()

# Parse all datetime columns (handle timezone-aware or naive consistently)
for col in ['startAt', 'endAt', 'clock_in_at', 'clock_out_at']:
    shifts_df[col] = pd.to_datetime(shifts_df[col], errors='coerce', utc=True)
    shifts_df[col] = shifts_df[col].dt.tz_localize(None)  # strip TZ

# Resolve pickup locations
shifts_df['pickup_location'] = shifts_df.apply(
    lambda r: resolve_pickup(r['driverId'], r['hopAreaLetter']), axis=1
)
```

---

### 4. Compute per-area show rates

```python
def compute_show_rates(shifts_df):
    """
    Per-area fraction of shifts where clock_in_at is NOT NaT.
    show_rate = 1.0 means all drivers showed up (no no-shows).
    show_rate = 0.85 means 15% no-show rate.
    """
    rates = {}
    for area, grp in shifts_df.groupby('hopAreaLetter'):
        total  = len(grp)
        showed = grp['clock_in_at'].notna().sum()
        rates[area] = round(showed / total, 4) if total > 0 else 1.0
    return rates

show_rates = compute_show_rates(shifts_df)
# Example output: {'A': 0.8862, 'B': 0.9124, 'E': 0.9084, ...}
```

---

### 5. Build inventory arrays — three modes

`build_inventory()` accepts an optional `maint_stock` parameter. Maintenance vehicles are
subtracted from each location's effective starting stock before computing the 96-slot arrays.
This ensures maintenance vehicles never appear as available to drivers in any mode.

```python
SLOTS = 96   # 96 × 15-minute slots per day

def dt_to_slot(dt):
    """Convert datetime to 15-min slot index (0–95). Returns None for NaT."""
    if pd.isna(dt): return None
    return int(dt.hour) * 4 + int(dt.minute) // 15

def build_inventory(shifts_df, loc_stock, show_rates, mode='planned', maint_stock=None):
    """
    Compute 96-slot vehicle availability array per location.

    Parameters
    ----------
    shifts_df   : filtered DataFrame with pickup_location resolved
    loc_stock   : dict {location_name: overnight_count}  (raw stock)
    show_rates  : dict {area_letter: show_rate_float}
    mode        : 'planned' | 'adjusted' | 'actual'
    maint_stock : dict {location_name: maintenance_count}  (subtracted from eff_stock)

    Effective stock
    ---------------
    eff_stock[loc] = max(0, loc_stock[loc] - maint_stock.get(loc, 0))
    All three modes start from eff_stock, not raw loc_stock.

    Modes
    -----
    planned   → use startAt / endAt;          weight = 1.0  (worst-case, full plan)
    adjusted  → use startAt / endAt;          weight = show_rate  (expected value)
    actual    → use clock_in_at / clock_out;  weight = 1.0  (NaT = no-show, skip)

    Note on adjusted mode
    ---------------------
    Fractional departure weight models expected no-shows. However, at shared hub
    locations (e.g. MH33, MH Rama 9) multiple areas' discounts stack, causing the
    adjusted curve to drop further than reality. Present adjusted as "optimistic bound".
    """
    maint_stock = maint_stock or {}
    eff_stock = {loc: max(0.0, float(s) - float(maint_stock.get(loc, 0)))
                 for loc, s in loc_stock.items()}

    inv   = {loc: [eff_stock[loc]] * SLOTS for loc in loc_stock}
    delta = {loc: [0.0]            * SLOTS for loc in loc_stock}

    dep_col = 'clock_in_at'  if mode == 'actual' else 'startAt'
    ret_col = 'clock_out_at' if mode == 'actual' else 'endAt'

    for _, row in shifts_df.iterrows():
        loc  = row['pickup_location']
        area = row['hopAreaLetter']
        if loc not in loc_stock:
            continue

        dep = dt_to_slot(row[dep_col])
        ret = dt_to_slot(row[ret_col])

        if dep is None:
            continue          # no-show → vehicle stays, skip shift
        if ret is None:
            ret = 95          # open-ended shift → assume returns end of day

        w = show_rates.get(area, 1.0) if mode == 'adjusted' else 1.0
        delta[loc][dep] -= w
        if ret < SLOTS:
            delta[loc][ret] += w

    # Apply cumulative delta starting from eff_stock; clamp at 0
    for loc in loc_stock:
        s = eff_stock[loc]   # IMPORTANT: use eff_stock, not loc_stock
        for i in range(SLOTS):
            s = max(0.0, s + delta[loc][i])
            inv[loc][i] = round(s, 1)

    return inv

planned_inv  = build_inventory(shifts_df, loc_stock, show_rates, mode='planned',  maint_stock=maint_stock)
adjusted_inv = build_inventory(shifts_df, loc_stock, show_rates, mode='adjusted', maint_stock=maint_stock)
actual_inv   = build_inventory(shifts_df, loc_stock, show_rates, mode='actual',   maint_stock=maint_stock)
```

Synthesised fallback (for days without any shift file) also subtracts maintenance:
```python
planned_inv = {loc: [max(0.0, float(s) - float(maint_stock.get(loc, 0)))] * SLOTS
               for loc, s in loc_stock.items()}
```

---

### 6. Shifts-only days (today + future with real assignments, no clock records)

For dates that have real driver assignments but no `clock_in_at` / `clock_out_at` yet
(i.e. today and future dates), load the shift file and build `planned` + `adjusted`
using rolling show rates. `actual` is always `None` for these days.

```python
# After computing rolling show rates from all real-data days:
for date_str, shifts_path in sorted(future_shifts_paths.items()):
    shifts_df    = load_shifts(shifts_path, loc_stock, drv_loc)
    planned_inv  = build_inventory(shifts_df, loc_stock, rolling, mode='planned')
    adjusted_inv = build_inventory(shifts_df, loc_stock, rolling, mode='adjusted')
    shifts_inventories[date_str] = {
        'planned':    planned_inv,
        'adjusted':   adjusted_inv,
        'show_rates': rolling,
    }
```

`discover_shifts_files()` auto-splits by today when using folder mode:

```python
real_shifts, future_shifts = discover_shifts_files(
    folder     = "G:/My Drive/.../DriverShiftRecord",
    date_window = all_dates,
    today_str  = "2026-04-12",
    future_folder = None,   # None = same folder, split by date
)
# real_shifts   : {date_str: path}  dates < today  (have clock-in/out)
# future_shifts : {date_str: path}  dates >= today  (assignments only)
```

The date loop now has **three branches**:

| Branch | Condition | `actual` | `has_data` | `has_shifts` |
|---|---|---|---|---|
| Real data | `date in real_inventories` | computed from clock_in_at | `True` | `False` |
| Shifts-only | `date in shifts_inventories` | `None` | `False` | `True` |
| Synthesised | neither | `None` | `False` | `False` |

The `dailySummary` entry now includes `has_shifts`:

```python
daily_summary[date_str] = {
    'has_data':      has_data,
    'has_shifts':    has_shifts,   # NEW — real assignments, rolling no-show estimate
    'is_future':     is_future,
    'planned_gaps':  planned_gaps,
    'adjusted_gaps': adjusted_gaps,
    'actual_gaps':   actual_gaps,  # None when has_shifts or synthesised
    'show_rates':    show_rates,
}
```

---

### 7. Multi-day synthesis

For days without **any** shift file, derive inventory from a template day:

```python
WEEKEND_SCALE = 0.75  # scale down Saturday (5) and Sunday (6)

def synthesise_day(template_planned, template_adjusted, date_str):
    """
    Scale template inventory for a non-data day.
    Returns dict: {loc: {planned, adjusted, actual:None}}
    """
    weekday = datetime.strptime(date_str, '%Y-%m-%d').weekday()
    scale   = WEEKEND_SCALE if weekday >= 5 else 1.0
    result  = {}
    for loc in template_planned:
        result[loc] = {
            'planned':  [round(v * scale, 1) for v in template_planned[loc]],
            'adjusted': [round(v * scale, 1) for v in template_adjusted[loc]],
            'actual':   None,
        }
    return result

def rolling_show_rates(all_real_rates_list):
    """
    Average show rates across all real-data days.
    all_real_rates_list = [{'A': 0.88, ...}, {'A': 0.85, ...}, ...]
    """
    combined = {}
    for day_rates in all_real_rates_list:
        for area, rate in day_rates.items():
            combined.setdefault(area, []).append(rate)
    return {a: round(sum(v) / len(v), 4) for a, v in combined.items()}

# Template selection:
#   weekday Mon–Thu, Sat, Sun → use nearest Thursday (or last available weekday data)
#   Friday                    → use nearest Friday data
# If only one template day is available, use it for all non-Friday days.
```

---

### 7. Count shortage slots (for dailySummary)

```python
ACT_S, ACT_E = 20, 88   # operational window: 05:00–22:00

def count_gaps(inv_dict, loc_stock, thresh=0):
    """Count (location, slot) pairs in operational window where value <= thresh."""
    total = 0
    for loc in loc_stock:
        arr = inv_dict.get(loc, [])
        total += sum(1 for s in range(ACT_S, ACT_E) if arr and arr[s] <= thresh)
    return total
```

---

### 8. Assemble multi-day JSON output

```python
def primary_area(loc):
    return max(areas_served[loc], key=areas_served[loc].get)

def build_location_entry(loc, days_dict):
    pa = primary_area(loc)
    return {
        "name":        loc,
        "primaryArea": pa,
        "color":       AREA_COLORS[pa],
        "stock":       int(loc_stock[loc]),
        "areasServed": {k: int(v) for k, v in areas_served[loc].items()},
        "days":        days_dict[loc],   # {date: {planned, adjusted, actual}}
    }

# Build complete output structure
output = {
    "today":         today_str,       # "YYYY-MM-DD"
    "dates":         all_dates,       # sorted list of 29 date strings
    "dailySummary":  daily_summary,   # {date: {has_data, has_shifts, is_future, planned_gaps, ...}}
    "locations":     [build_location_entry(loc, days_by_loc) for loc in sorted_locs],
    "areaLocations": area_loc_map,
    # 🚗 Vehicles tab
    "vehicles":      vehicles,        # ALL rows [{plate, area, areaName, location, status}]
    "stockDate":     stock_date,      # "YYYY-MM-DD" from vehicleMainLocation filename
    # 👤 Drivers tab
    "drivers":       drivers_list,    # [{driverId, driverCode, nameTh, heroId, rank1/2/3, ...}]
    "driversWeek":   drivers_week,    # "YYYY Week NN"
    # 🔧 Maintenance
    "maintenanceVehicles": sorted(maintenance_plates),  # sorted list of plates
}

with open('vehicle_data_multiday.json', 'w', encoding='utf-8') as f:
    json.dump(output, f, ensure_ascii=False)
```

---

## Edge cases & gotchas

| Situation | Handling |
|---|---|
| `Main_Location` is NaN | `dropna(subset=['Main_Location'])` before building `loc_stock` |
| `Area` value not in `AREA_NAME_TO_LETTER` (e.g. 'เช่ารายวัน MuvMi RM9') | `map()` returns NaN; `isin(AREA_KEEP)` drops it automatically |
| Driver's `Rank_1` not in known `loc_stock` | Fall back to area default station |
| `driverId` is NaN | Use explicit `for` loop with `if pd.notna(did)` — avoids `KeyError: nan` |
| `clock_out_at` is NaT (shift didn't end) | Use slot 95 (23:45) for return |
| `clock_in_at` is NaT (no-show) | Skip the shift; vehicle stays parked |
| Same driver in multiple areas | Each row resolves independently |
| Location in shifts not in `vehicleMainLocation` | Skip (no stock entry) |
| Stock underflow (cumulative delta goes negative) | Clamp: `max(0.0, eff_stock + delta)` |
| Timezone-aware datetimes | Strip TZ with `.dt.tz_localize(None)` before `dt_to_slot()` |
| `is_deleted` may be bool or int | Use `== False`, not `is False` |
| Adjusted worse than planned at hubs | Expected — fractional weight stacks across all areas sharing the location |
| Weekend synthesised days | Multiply template arrays × 0.75 (WEEKEND_SCALE) |
| Future dates with real shift file (`has_shifts=True`) | `planned` + `adjusted` from real assignments using rolling rates; `actual: null`; dashboard shows cyan "SHIFT" pill |
| Future dates without any shift file | Synthesised from template; `actual: null`; dashboard shows indigo "FCST" pill |
| Past dates without any shift file (data gap) | Synthesised from template; `actual: null`; dashboard shows amber "EST" pill |
| `maintenance_file` not found or missing | Graceful degradation — empty set, WARNING printed, pipeline continues |
| `ซ่อมแล้ว` column has unexpected types | Use `== False` (not `is False`) to handle both bool and int (0) |
| Maintenance vehicle at unknown location | Skip — only vehicles with non-empty `location` contribute to `maint_stock` |
| `maint_stock` > `loc_stock` at a location | `max(0.0, ...)` clamps effective stock to zero; never goes negative |
| `today` set to yesterday by mistake | Reverts yesterday to SHIFT date (no clocks shown). Fix: always use `today=None` |
