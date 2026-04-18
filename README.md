---
name: vehicle-availability-dashboard
description: >
  Build an interactive vehicle availability dashboard for MuvMi's fleet operations,
  showing 15-minute vehicle counts per charging station location and flagging when
  any location will run out of vehicles. Use this skill whenever the user asks about
  vehicle availability, vehicle shortage prediction, fleet dashboard, charging station
  coverage, driver shift analysis, or wants to know "when will we run out of vehicles".
  Triggers: "vehicle availability", "vehicle shortage", "fleet dashboard", "run out of
  vehicles", "charging station", "driver shifts", "15-minute availability", "location
  inventory", "future shortage", "drivershifts", "vehicleMainLocation", "driverlocation".
  Always use this skill when the user mentions any combination of vehicles, locations,
  and shifts — even if they don't say "dashboard" explicitly.
---

# Vehicle Availability Dashboard — MuvMi Fleet

Build an interactive HTML dashboard showing 15-minute vehicle counts at each charging
station for any date range. Flags when a location reaches zero vehicles (shortage).
Supports both historical actuals and 2-week future projections.

---

## Data Sources & File Structure

| Dataset | Path pattern | Key use |
|---|---|---|
| Overnight inventory | `vehicleMainLocation/vehicleMainLocation_{YYYYMMDD}.csv` | Starting stock per location |
| Driver pickup locations | `driverlocation/driverlocation_{YYYY}_week{NN}.csv` | Where each driver picks up & returns |
| Shifts + work records | `DriverShiftRecord/drivershifts_{YYYY-MM-DD}.xlsx` (sheet: workrecord) | Planned times + actual clock-in/out |

### Key column reference

**vehicleMainLocation:**
`plate, Area, vehicleId, name, createdAt, StatusInFleet, Main_Location, Second_Location, Main_Weight`
→ Use `Main_Location` as the vehicle's overnight home station.

**driverlocation:**
`driverId, [date columns for each day of that week], Rank_1, Rank_2, Rank_3, Rank_4, driverCode, display_name_en, employment_type, shift_type`
→ Use `Rank_1` as the driver's designated pickup (and return) location.
→ If `Rank_1` is null/empty, fall back to the default charging station for their `hopAreaLetter`.

**drivershifts workrecord sheet:**
`_id, driverId, hopAreaId, hopAreaLetter, hopAreaName, startAt, endAt, clock_in_at, clock_out_at, date, is_deleted, vehicle_location_name, ...`
→ `startAt` / `endAt`: planned shift window.
→ `clock_in_at` / `clock_out_at`: actual times (null = no-show for clock_in; null = still out for clock_out).
→ Filter rows: `hopAreaLetter` must be in `{A, B, E, F, G, I, L, P, Q, W}`.
→ Filter rows: `is_deleted` must not be `True`.

### Default charging stations by hopAreaLetter
```
A → จุดจอดจันทนยิ่งยง
B → MH33
E → ตลาดสะพานคู่ - คลองสาน
F → MH Rama 9
G → MH33
I → MH Rama 9
L → MH33
P → MH Rama 3
Q → MH Rama 9
W → ตลาดสะพานคู่ - คลองสาน
```

### Driver location week lookup rule
Weeks start on Friday. For any target date:
1. Find the most recent Friday on or before the target date → that date's ISO week number = `target_week`
2. Look up driverlocation file for `target_week - 2` (two weeks prior)
3. Use the same year; handle year boundary if needed
4. Example: target date = Apr 3 (week 15 in their system) → use week 13 file

---

## Core Business Rules

1. **Same pickup and return location**: every driver returns the vehicle to the same station they picked it up from.
2. **No-show = vehicle stays**: if `clock_in_at` is null, the vehicle never left — it stays at the location all day.
3. **Threshold = zero**: flag any 15-min interval where vehicle count drops to 0 or below.
4. **Lead time**: there is a short window between shift start and actual vehicle departure (clock-in), and between shift end and vehicle return (clock-out). Use historical averages, split by `before 14:00` and `after 14:00` per location.
5. **Future shifts use estimated times**: for dates without actual clock-in/out, use `startAt - avg_checkin_lead` for departure and `endAt + avg_checkout_lead` for return; apply no-show rate to randomly withhold vehicles.

---

## Step-by-Step Instructions

### Step 1 — Identify folder paths
Ask the user (or infer from context) for:
- Path to `vehicleMainLocation/` folder
- Path to `driverlocation/` folder
- Path to `DriverShiftRecord/` folder
- Date range (default: today + 14 days)

If user says "my baseline analysis folder is at X", derive sub-paths from there:
```
X/vehiclePrototype/vehicleMainLocation/
X/vehiclePrototype/driverlocation/
X/DriverShiftRecord/
```

### Step 2 — Install dependencies
```bash
pip install pandas openpyxl plotly --break-system-packages -q
```

### Step 3 — Run the bundled processing script
The script at `scripts/process_vehicle_data.py` (relative to this SKILL.md) handles all data
loading, processing, and HTML generation. Run it like this:

```bash
python <skill_dir>/scripts/process_vehicle_data.py \
  --overnight-dir "<path_to_vehicleMainLocation>" \
  --driverlocation-dir "<path_to_driverlocation>" \
  --shifts-dir "<path_to_DriverShiftRecord>" \
  --start-date <YYYY-MM-DD> \
  --days 14 \
  --lookback 14 \
  --output "<output_path>/vehicle_availability_dashboard.html"
```

Replace `<skill_dir>` with the absolute path to this skill's directory.

The script will:
- Print a progress log as it processes each day
- Print a summary table of shortage windows found
- Save the HTML dashboard to `--output`

### Step 4 — Handle errors gracefully
Common issues and fixes:
- **Missing overnight file for a date**: skip that date and note it in output
- **Missing shifts file**: treat as all-no-show day (no vehicle movement)
- **Driver not in driverlocation**: fall back to default charging station via hopAreaLetter
- **All clock_in_at null in a historical file**: warn user, may be a data pipeline issue

### Step 5 — Present the output
Save the HTML file to the user's workspace folder and provide a `computer://` link.
Also print a short text summary of the most critical shortage windows, for example:
> "MH33 is projected to hit 0 vehicles on Apr 14 between 07:15–08:30 (5 drivers
>  scheduled with no available vehicles). MH Rama 9 has a risk window on Apr 15 09:00–09:45."

---

## Output Dashboard Features

The generated HTML includes:
- **Date selector** (tabs for each day in the range)
- **Location tabs** (one per charging station that has data)
- **15-min timeline chart** per location showing vehicle count over 00:00–23:59
  - Green fill when count > 0, red fill/background when count = 0
  - Hover tooltip: count + list of drivers currently out
- **Summary heatmap** (locations × time-of-day) color-coded by availability
  - Green = enough, amber = 1 vehicle, red = 0 vehicles
- **Historical vs. Projected toggle**: solid lines for actuals, dashed for estimates
- **Shortage summary table** at the bottom listing all flagged windows

---

## Notes on Future Projections

For dates without actual work records (future shifts):

1. **Lead times**: compute from last `--lookback` days of historical data
   - `checkin_lead_mins = avg(clock_in_at - startAt)` grouped by location + before/after 14:00
   - `checkout_lead_mins = avg(clock_out_at - endAt)` grouped by location + before/after 14:00
   - If insufficient data for a location, use the global average

2. **No-show rates**: compute from last `--lookback` days
   - `noshow_rate_weekday = count(clock_in_at is null) / count(total shifts)` for Mon–Fri
   - `noshow_rate_weekend = count(clock_in_at is null) / count(total shifts)` for Sat–Sun
   - Apply probabilistically: a driver is treated as no-show if `random() < noshow_rate`
   - For deterministic output, use expected value: reduce vehicle departures proportionally

3. **Conservative default**: if no historical data at all, assume 5 min check-in lead, 10 min check-out lead, 10% no-show rate.
