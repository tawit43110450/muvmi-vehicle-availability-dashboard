# Dashboard Design Reference — Vehicle Availability Dashboard

## Architecture

Single self-contained HTML file (~800 KB with 29-day embedded data). No external dependencies except Chart.js CDN.
Embed the JSON data directly as a `const REAL_DATA = {...}` JavaScript constant.

```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.5.1" crossorigin="anonymous"></script>
```

---

## ⚠️ Critical: Chart.js sizing — always use a wrapper div

Chart.js with `responsive: true` + `maintainAspectRatio: false` reads the **parent container's height**,
not the canvas element's height. Setting `height` on the `<canvas>` directly causes an infinite resize loop
(canvas grows → parent grows → Chart.js reads larger parent → canvas grows again).

**The correct pattern — apply to every chart:**
```html
<!-- Wrapper div carries the height; canvas is positioned absolutely inside it -->
<div id="my-chart-wrap" style="position:relative; height:200px; overflow:hidden;">
  <canvas id="my-canvas"></canvas>
</div>
```
```css
#my-chart-wrap         { position: relative; height: 200px; overflow: hidden; }
#my-chart-wrap canvas  { position: absolute; top: 0; left: 0; }
```

**For dynamic height (e.g. stock chart with variable number of rows):**
```js
// Set height on the WRAPPER via JS, never on the canvas element
document.getElementById('stock-canvas-wrap').style.height = (items.length * 22) + 'px';
// Then create the chart normally — Chart.js reads the wrapper height
stockChart = new Chart(canvas.getContext('2d'), { ... });
```

**What NOT to do:**
```js
canvas.height = 300;              // ❌ causes resize loop with responsive:true
canvas.style.height = '300px';   // ❌ same problem
canvas.style.maxHeight = '300px'; // ❌ Chart.js ignores max-height
```

## Password gate

A full-screen overlay renders before any dashboard content is visible.
Correct password stored as a JS constant — easy to change:

```js
const DASHBOARD_PASSWORD = 'muvmi2024';  // ← change this line to update the password
const PW_SESSION_KEY     = 'muvmi_dash_auth';
```

Authentication persists for the browser session via `sessionStorage`. On correct password,
the overlay receives class `.hidden` (`display:none`). Wrong password triggers a shake animation.

```html
<!-- Add immediately after <body> -->
<div id="pw-overlay">
  <div id="pw-card">
    <div class="pw-icon">🚌</div>
    <h2>MuvMi Fleet Dashboard</h2>
    <p class="pw-sub">Enter password to continue</p>
    <input id="pw-input" type="password" placeholder="Password"
           onkeydown="if(event.key==='Enter') checkPassword()">
    <button id="pw-btn" onclick="checkPassword()">Unlock</button>
    <div id="pw-err"></div>
  </div>
</div>
```

```css
#pw-overlay { position:fixed; inset:0; z-index:9999; background:var(--navy);
              display:flex; align-items:center; justify-content:center; }
#pw-overlay.hidden { display:none; }
#pw-card { background:var(--navy2); border:1px solid var(--border); border-radius:16px;
           padding:40px; width:320px; text-align:center; }
@keyframes shake {
  0%,100%{transform:translateX(0)} 25%{transform:translateX(-8px)} 75%{transform:translateX(8px)}
}
.shake { animation: shake .3s ease; }
```

```js
function checkPassword() {
    const val = document.getElementById('pw-input').value;
    if (val === DASHBOARD_PASSWORD) {
        sessionStorage.setItem(PW_SESSION_KEY, '1');
        document.getElementById('pw-overlay').classList.add('hidden');
    } else {
        const card = document.getElementById('pw-card');
        card.classList.remove('shake');
        void card.offsetWidth;  // reflow to re-trigger animation
        card.classList.add('shake');
        document.getElementById('pw-err').textContent = 'Incorrect password';
    }
}
// On load:
if (sessionStorage.getItem(PW_SESSION_KEY) === '1') {
    document.getElementById('pw-overlay').classList.add('hidden');
}
```

---

## Layout structure

```
┌─ Password gate (overlay, first load) ─────────────────────────┐
├─ Header (shared) ─────────────────────────────────────────────┤
│  Title · Area filter · Threshold input                        │
├─ Tab navigation ──────────────────────────────────────────────┤
│  [📊 Availability]  [🚗 Vehicle Locations]  [👤 Drivers]      │
├─ Tab 1: Availability ─────────────────────────────────────────┤
│  ‹ [Mar29 EST] [Apr3 REAL] [Apr12 SHIFT] [Apr26 FCST] ›      │
│  29-day trend chart                                           │
│  KPI cards (incl 🔧 Maintenance KPI)                          │
│  Data type banner ("✅ Real" / "📋 Shift" / "📊 Est" / "🔮") │
│  Availability Heatmap (3 sub-rows: planned/adjusted/actual)   │
│  Detail chart │ Shortage summary table (incl 🔧 column)       │
│  Overnight stock bar chart                                    │
├─ Tab 2: Vehicle Locations ────────────────────────────────────┤
│  Stats: InFleet · 🔧 Maintenance · Other · # Locations        │
│  Filters: search · area · status (incl Under Maintenance)     │
│  [≡ Vehicle List] sub-tab: sortable table + maint badge       │
│  [⊞ Location Summary] sub-tab: cards with maint count row     │
├─ Tab 3: Drivers ──────────────────────────────────────────────┤
│  Stats: # drivers · pickup points · areas                     │
│  Filters: search · primary pickup · emp type · shift · area   │
│  Sortable table: code·name·rank1·rank2·rank3·emp·shift·date  │
└───────────────────────────────────────────────────────────────┘
```

### Tab switching pattern
```js
function switchTab(name) {
  document.querySelectorAll('.tab-pane').forEach(p => p.classList.remove('active'));
  document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
  document.getElementById('tab-' + name).classList.add('active');
  document.querySelector(`.tab-btn[onclick*="'${name}'"]`).classList.add('active');
  if (name === 'vehicles') renderVehiclesTab();
  if (name === 'drivers')  renderDriversTab();
}
// HTML: <div id="tab-availability" class="tab-pane active"> ... </div>
//       <div id="tab-vehicles"     class="tab-pane"> ... </div>
//       <div id="tab-drivers"      class="tab-pane"> ... </div>
```

---

## JavaScript data model

```js
const REAL_DATA    = { /* embedded JSON — see data-pipeline.md §8 */ };
const LOCS         = REAL_DATA.locations;       // array of location objects
const AREA_LOCS    = REAL_DATA.areaLocations;   // area → [loc names], ordered by stock
// 🚗 Vehicles tab
const VEHICLES     = REAL_DATA.vehicles  || []; // [{plate, area, areaName, location, status}]
const STOCK_DATE   = REAL_DATA.stockDate || ''; // "YYYY-MM-DD" from CSV filename
// 👤 Drivers tab
const DRIVERS      = REAL_DATA.drivers      || []; // [{driverId, driverCode, nameTh, ...}]
const DRIVERS_WEEK = REAL_DATA.driversWeek  || ''; // "2026 Week 13"
const DATES      = REAL_DATA.dates;           // ["2026-03-29", ..., "2026-04-26"]
const SUMMARY    = REAL_DATA.dailySummary;    // {date → summary object}
const TODAY      = REAL_DATA.today;           // "YYYY-MM-DD" of pipeline run

const SLOTS      = 96;
const ACT_S      = 20, ACT_E = 88;   // operational window 05:00–22:00
const D_START    = 20;               // heatmap and chart start slot

const AREA_NAMES  = {
    A:'Chula-Chidlom', B:'Ari-Phayathai', E:'Rattanakosin',
    F:'Sukhumvit', G:'Phahol-Kaset', I:'Onnut-Bearing',
    L:'Bangsue', P:'Silom', Q:'Ratchada-Rama9', W:'Wongwian Yai'
};
const AREA_COLORS = {
    A:'#6366f1', B:'#f59e0b', E:'#10b981', F:'#ef4444',
    G:'#8b5cf6', I:'#ec4899', L:'#14b8a6', P:'#f97316',
    Q:'#3b82f6', W:'#84cc16'
};
const AREA_ORDER = ['A','B','E','F','G','I','L','P','Q','W'];

// 🔧 Maintenance vehicles
const MAINTENANCE_VEHICLES = new Set(REAL_DATA.maintenanceVehicles || []);
function isMaint(plate) { return MAINTENANCE_VEHICLES.has(plate); }

// Per-location maintenance count (O(n) build once at init)
const MAINT_BY_LOC = {};
VEHICLES.forEach(v => {
    if (isMaint(v.plate) && v.location) {
        MAINT_BY_LOC[v.location] = (MAINT_BY_LOC[v.location] || 0) + 1;
    }
});
const TOTAL_MAINT = MAINTENANCE_VEHICLES.size;
```

### State variables
```js
let selectedDate = TODAY;    // currently displayed date
let selectedLoc  = null;     // location clicked in heatmap → drives detail chart
let detailChart  = null;     // Chart.js instance for detail chart
let stockChart   = null;     // Chart.js instance for stock bar chart
let trendChart   = null;     // Chart.js instance for 29-day trend chart
```

---

## Helper functions

```js
function getLoc(name) { return LOCS.find(l => l.name === name); }

function filteredLocs() {
    const area = document.getElementById('f-area').value;
    if (area === 'ALL') return [...LOCS];
    return (AREA_LOCS[area] || []).map(n => getLoc(n)).filter(Boolean);
}

function isShared(loc) { return Object.keys(loc.areasServed).length > 1; }

function s2t(s) {
    return String(Math.floor(s/4)).padStart(2,'0') + ':' + String((s%4)*15).padStart(2,'0');
}

function cellCls(v, thresh) {
    if (thresh < 0) return v <= 0 ? 'c1' : 'c4';   // −1 mode: only flag zeros
    if (v <= 0)           return 'c1';  // red
    if (v <= thresh)      return 'c2';  // orange
    if (v <= thresh*2.5)  return 'c3';  // yellow
    return 'c4';                         // green
}

function getThresh() { return parseFloat(document.getElementById('f-thresh').value) || 0; }
function dayData(loc, date) { return loc.days[date] || {planned:null, adjusted:null, actual:null}; }
function hasActual(date)  { return SUMMARY[date] && SUMMARY[date].has_data; }
function isFuture(date)   { return SUMMARY[date] && SUMMARY[date].is_future; }
function hasShifts(date)  { return SUMMARY[date] && SUMMARY[date].has_shifts; }

// Central date type resolver — use this everywhere instead of chaining has_data/is_future
function getDateType(date) {
    const s = SUMMARY[date];
    if (!s) return 'est';
    if (s.has_data)   return 'real';   // clock-in/out recorded
    if (s.has_shifts) return 'shift';  // real assignments, no clocks yet → rolling no-show estimate
    if (s.is_future)  return 'fcst';   // no shift file, synthesised future
    return 'est';                       // no shift file, synthesised past
}
const DATE_TYPE_COLOR = { real:'#10b981', shift:'#06b6d4', fcst:'#6366f1', est:'#f59e0b' };
const DATE_TYPE_LABEL = { real:'REAL',    shift:'SHIFT',   fcst:'FCST',    est:'EST' };
```

### Date type summary

| `getDateType()` | Pill colour | Banner | `actual` data? | Source |
|---|---|---|---|---|
| `real` | Green | ✅ Real data — clock-in/out recorded | ✅ Yes | `has_data=true` |
| `shift` | Cyan | 📋 Shift data — real assignments, rolling no-show estimate | ❌ No | `has_shifts=true` |
| `fcst` | Indigo | 🔮 Forecast — synthesised, no shift file | ❌ No | `is_future=true` |
| `est` | Amber | 📊 Estimated — synthesised, past date | ❌ No | all false |

---

## Component: Date navigation bar

HTML:
```html
<div id="date-nav">
  <button id="btn-prev" onclick="navigateDate(-1)">‹</button>
  <div id="date-scroll"><!-- pills injected by JS --></div>
  <button id="btn-next" onclick="navigateDate(1)">›</button>
</div>
```

JS — build pills (use `getDateType()`, never inline `has_data`/`is_future` chains):
```js
function buildDateNav() {
    const scroll = document.getElementById('date-scroll');
    DATES.forEach((date, idx) => {
        const { m, day, wd } = formatDatePill(date);
        const type  = getDateType(date);
        const label = DATE_TYPE_LABEL[type];
        const color = DATE_TYPE_COLOR[type];
        const isToday = date === TODAY;
        const pill = document.createElement('div');
        pill.className = `date-pill ${type}${isToday?' today-marker':''}${date===selectedDate?' selected':''}`;
        pill.dataset.date = date;
        pill.innerHTML = `<div class="dp-date">${m} ${day}</div>
                          <div class="dp-label">${wd} · <span style="color:${color}">${label}</span></div>`;
        pill.onclick = () => selectDate(date);
        scroll.appendChild(pill);
    });
}

function selectDate(date) {
    selectedDate = date;
    document.querySelectorAll('.date-pill').forEach(p => {
        p.classList.toggle('selected', p.dataset.date === date);
    });
    document.getElementById('btn-prev').disabled = DATES.indexOf(date) <= 0;
    document.getElementById('btn-next').disabled = DATES.indexOf(date) >= DATES.length - 1;
    refresh();
}

function navigateDate(dir) {
    const idx = DATES.indexOf(selectedDate) + dir;
    if (idx >= 0 && idx < DATES.length) selectDate(DATES[idx]);
}
```

CSS — pill types:
```css
.date-pill          { border: 1.5px solid transparent; border-radius: 8px; padding: 4px 8px;
                      cursor: pointer; font-size: 10px; text-align: center; }
.date-pill.real     { border-color: #10b981; color: #10b981; }  /* green = real data */
.date-pill.est      { border-color: #f59e0b; color: #f59e0b; }  /* amber = estimated */
.date-pill.fcst     { border-color: #6366f1; color: #6366f1; }  /* indigo = forecast */
.date-pill.today-marker { box-shadow: 0 0 0 2px #ec4899; }      /* pink glow = today */
.date-pill.selected { background: rgba(255,255,255,0.06); }
```

---

## Component: 29-day trend chart

Clickable bar chart at the top. Clicking a bar calls `selectDate(DATES[index])`.

```js
function buildTrendChart() {
    const planData = DATES.map(d => SUMMARY[d].planned_gaps  || 0);
    const adjData  = DATES.map(d => SUMMARY[d].adjusted_gaps || 0);
    const actData  = DATES.map(d => SUMMARY[d].actual_gaps);   // null for non-real

    trendChart = new Chart(ctx, {
        data: {
            labels: DATES.map(d => /* short label */),
            datasets: [
                { type:'bar', label:'Planned gaps',  data:planData, backgroundColor:'#94a3b855', order:3 },
                { type:'bar', label:'Adjusted gaps', data:adjData,  backgroundColor:'#f59e0b44', order:2 },
                { type:'bar', label:'Actual gaps',   data:actData,  backgroundColor:'#10b98188', order:1 },
            ]
        },
        options: {
            responsive: true, maintainAspectRatio: false,
            onClick: (e, elements) => {
                if (elements.length > 0) selectDate(DATES[elements[0].index]);
            },
            plugins: { legend:{ display:false }, tooltip:{ mode:'index', intersect:false } },
            scales: { x:{ ticks:{font:{size:9}} }, y:{ beginAtZero:true } }
        }
    });
}
```

---

## Component: Data type banner

Context-aware label showing which category of data is displayed.

```js
function renderBanner(date) {
    const type = getDateType(date);   // always use getDateType(), never inline has_data chain
    const msgs = {
        real:  '✅ Real data — actual clock-in/out recorded',
        shift: '📋 Shift data — real assignments, rolling no-show estimate',
        fcst:  '🔮 Forecast — planned + adjusted no-show estimate',
        est:   '📊 Estimated — synthesised from template day',
    };
    const msg = msgs[type];
    // Apply class matching type ('real' | 'shift' | 'fcst' | 'est') for colour coding
}
```

Show-rate chips (rendered next to the banner):
```js
const rates = SUMMARY[date].show_rates || {};
AREA_ORDER.filter(a => rates[a] !== undefined).forEach(a => {
    // <div class="rate-chip"><strong>A</strong> 89%</div>
});
```

---

## Component: KPI cards

Compute from deduplicated locations (shared location counts once per metric):

```js
function renderKPI(date) {
    const shown = hasActual(date);
    filteredLocs().forEach(loc => {
        const d = dayData(loc, date);
        // planGaps  += slots 20–88 where d.planned[s]  <= thresh
        // adjGaps   += slots 20–88 where d.adjusted[s] <= thresh
        // actGaps   += slots 20–88 where d.actual[s]   <= thresh  (if shown)
    });
    // If !shown: hide actual and at-risk cards, show "(Forecast/Estimated)" note
}
```

---

## Component: Availability Heatmap

**Critical**: iterate `AREA_LOCS` (not `LOCS.area`) — this is what makes shared locations
appear under every area they serve.

```js
function renderHeatmap(thresh, filterArea) {
    const date  = selectedDate;
    const shown = hasActual(date);
    const areas = filterArea === 'ALL' ? AREA_ORDER : [filterArea];

    areas.forEach(area => {
        const locNames = AREA_LOCS[area] || [];
        // Render: area separator row (bold, coloured)
        locNames.forEach((name, idx) => {
            const loc = getLoc(name);
            const d   = dayData(loc, date);
            const rowCls = idx % 2 === 0 ? 'hm-row-even' : 'hm-row-odd';

            // Location label: name + area vehicle count + SHARED badge (if multi-area)
            // Per slot (s = D_START to 95): render 3 sub-rows + separator
            for (let s = D_START; s < SLOTS; s++) {
                const pv  = d.planned  ? d.planned[s]  : 0;
                const av  = d.adjusted ? d.adjusted[s] : 0;
                const acv = (shown && d.actual) ? d.actual[s] : null;
                // Sub-row 1: planned   → cellCls(pv, thresh)
                // Sub-row 2: adjusted  → cellCls(av, thresh)
                // Sub-row 3: separator → class 'ch' (thin grey line)
                // Sub-row 4: actual    → cellCls(acv, thresh) or 'c0' if null
            }
        });
    });
}
```

### Cell HTML structure (per slot)
```html
<td style="padding:0;vertical-align:middle;">
  <div class="hm-cell-wrap"
    onmouseenter="showTip(event,'Name','HH:MM',pv,av,acv)"
    onmouseleave="hideTip()"
    onclick="pickLoc('Name')">
    <div class="hm-sub c4"></div>  <!-- planned (top) -->
    <div class="hm-sub c2"></div>  <!-- adjusted -->
    <div class="hm-sub ch"></div>  <!-- separator line -->
    <div class="hm-sub c0"></div>  <!-- actual (bottom); c0 = grey if null -->
  </div>
</td>
```

### CSS for cells
```css
.hm-cell-wrap { display:flex; flex-direction:column; gap:1px; width:10px; cursor:pointer; }
.hm-sub       { width:10px; height:7px; border-radius:1px; }
.hm-sub.ch    { height:3px; background:#334155; border-radius:0; }  /* separator */
.c0 { background:#e5e7eb22; }  /* grey — no demand or null actual */
.c1 { background:#ef4444; }   /* red    — empty */
.c2 { background:#f97316; }   /* orange — at/below threshold */
.c3 { background:#eab308; }   /* yellow — moderate */
.c4 { background:#22c55e; }   /* green  — good */
```

### Alternating row CSS + area separator
```css
.hm-row-even    { background: #ffffff04; }
.hm-row-odd     { background: #00000015; }
.hm-row-sep     { border-bottom: 1px solid #334155; }  /* last row of each location group */
.hm-area-hdr    { background: #1e293b; font-weight: 700; letter-spacing: .05em; }
```

---

## Component: Detail chart (Chart.js)

Starts at D_START = slot 20 (05:00). Actual shown only when `hasActual(selectedDate)`.

```js
function renderDetailChart() {
    const d       = dayData(loc, selectedDate);
    const shown   = hasActual(selectedDate);

    const slicedPlan = (d.planned  || []).slice(D_START);
    const slicedAdj  = (d.adjusted || []).slice(D_START);
    const slicedAct  = (shown && d.actual) ? d.actual.slice(D_START) : null;
    const len    = slicedPlan.length;
    const labels = Array.from({length:len}, (_,i) =>
        (i + D_START) % 4 === 0 ? s2t(i + D_START) : ''
    );

    const datasets = [
        { type:'line', label:'Planned', data:slicedPlan,
          borderColor:locColor, borderDash:[5,3], pointRadius:0, borderWidth:1.5 },
        { type:'line', label:'Adjusted (no-show)', data:slicedAdj,
          borderColor:'#f59e0b', borderDash:[2,3], pointRadius:0, borderWidth:1.5 },
        { type:'line', label:`Threshold (${thresh})`,
          data: Array(len).fill(thresh > 0 ? thresh : 0.5),
          borderColor:'#ef444450', borderDash:[3,4], pointRadius:0, borderWidth:1 },
    ];

    if (slicedAct) {
        // Prepend bar chart dataset for actual; colour bars by threshold
        datasets.unshift({ type:'bar', label:'Actual vehicles', data:slicedAct,
            backgroundColor: slicedAct.map(v => cellToColor(v, thresh, '88')),
            borderRadius:1, order:2 });
    }

    // Tooltip title uses: s2t(ctx[0].dataIndex + D_START)
}
```

---

## Component: Shortage table

```js
function renderShortageTable(thresh) {
    const shown = hasActual(selectedDate);
    AREA_ORDER.forEach(area => {
        // Render area header row
        (AREA_LOCS[area] || []).forEach(name => {
            const d = dayData(loc, selectedDate);
            // Count ps  = planned slots ≤ thresh (ops window)
            // Count as_ = adjusted slots ≤ thresh
            // Count acts = actual slots ≤ thresh (if shown)
            // firstGap = first slot index where shortage occurs
            // Badge: 'OK' | 'At Risk' (1–4 slots) | 'Critical' (5+)
        });
    });
}
```

---

## Component: Stock bar chart

Horizontal bar chart showing overnight stock per location (static, not date-dependent).

```js
function renderStockChart() {
    // Deduplicate locations across areas
    // Sort by stock descending
    // Colour by primaryArea
    new Chart(canvas, {
        type: 'bar',
        options: { indexAxis: 'y', ... }
    });
}
```

---

## Refresh cycle

All components are re-rendered together on date change, area filter change, or threshold change:

```js
function refresh() {
    const thresh = getThresh();
    const area   = document.getElementById('f-area').value;
    updateSubtitle();          // update header date string
    renderBanner(selectedDate);
    renderKPI(selectedDate);
    renderHeatmap(thresh, area);
    renderShortageTable(thresh);
    renderDetailChart();       // only if selectedLoc is set
}
```

---

## Controls

```html
<!-- Area filter dropdown -->
<select id="f-area" onchange="refresh()">
  <option value="ALL">All Areas</option>
  <option value="A">A · Chula-Chidlom</option>
  <!-- ... all 10 areas ... -->
</select>

<!-- Shortage threshold — numeric input, range −1 to 5 -->
<!-- −1 = only flag zeros; 0 = default; 1–5 = flag below N vehicles -->
<input type="number" id="f-thresh" min="-1" max="5" step="1" value="0" oninput="refresh()">
```

---

## Color palette (CSS variables)

```css
:root {
  --navy:   #0f172a;   /* page background */
  --navy2:  #1e293b;   /* header / card background */
  --navy3:  #334155;   /* inputs / sub-elements */
  --text:   #f1f5f9;
  --text2:  #94a3b8;
  --text3:  #64748b;
  --border: #334155;
  --red:    #ef4444;
  --orange: #f97316;
  --yellow: #eab308;
  --green:  #22c55e;
  --real:   #10b981;   /* real data → green */
  --est:    #f59e0b;   /* estimated → amber */
  --fcst:   #6366f1;   /* forecast → indigo */
  --today:  #ec4899;   /* today marker → pink */
}
```

---

## Component: 🚗 Vehicle Locations tab

Two sub-views switchable via `.veh-sub-btn` buttons.

**Stat cards (4 cards):**
- 🚗 InFleet — vehicles with `status === 'InFleet'`
- 🔧 Maintenance — `TOTAL_MAINT`
- 📦 Other — vehicles with `status !== 'InFleet'`
- 📍 Locations — unique non-empty locations in filtered set

**Filter options for Status dropdown:**
```
All Status | InFleet | Under Maintenance | Other
```

**Key JS pattern:**
```js
const VEHICLES = REAL_DATA.vehicles || [];   // all rows, all statuses
const STOCK_DATE = REAL_DATA.stockDate || '';

// Location→area lookup (built from AREA_LOCS)
const LOC_TO_AREA = {};
Object.entries(AREA_LOCS).forEach(([area, locs]) =>
  locs.forEach(loc => { if (!LOC_TO_AREA[loc]) LOC_TO_AREA[loc] = area; })
);

// Area key helper — returns area letter if mapped, else Thai areaName for unmapped areas
function vehAreaKey(v) { return v.area || v.areaName || ''; }

// Filter dropdowns are populated dynamically from actual data at runtime
function initVehFilters() {
  const areaSet = new Set(VEHICLES.map(vehAreaKey).filter(Boolean));
  const sel = document.getElementById('veh-area-filter');
  [...areaSet].sort().forEach(k => {
    const opt = document.createElement('option');
    opt.value = k;
    opt.textContent = AREA_NAMES[k] ? `${k} · ${AREA_NAMES[k]}` : k;
    sel.appendChild(opt);
  });
}

function filteredVehicles() {
  const q      = document.getElementById('veh-search').value.toLowerCase();
  const area   = document.getElementById('veh-area-filter').value;
  const status = document.getElementById('veh-status-filter').value;
  return VEHICLES.filter(v => {
    if (area !== 'ALL' && vehAreaKey(v) !== area)               return false;
    if (status === 'InFleet'      && v.status !== 'InFleet')    return false;
    if (status === 'Maintenance'  && !isMaint(v.plate))         return false;
    if (status === 'Other' && (v.status === 'InFleet' || isMaint(v.plate))) return false;
    const hay = [v.plate, v.location, v.area, v.areaName].join(' ').toLowerCase();
    return !q || hay.includes(q);
  });
}
```

**Vehicle List view** — sortable table: Plate · Area badge · Parking Location · Status
- Maintenance plates show in orange (`veh-plate-maint` class) with a 🔧 MAINTENANCE badge
- Maintenance rows get orange row background (`loc-plate-maint` class)

**Location Summary view** — card grid grouped by location, sorted by vehicle count desc.
Each card shows: location name, vehicle count, area chip breakdown, maintenance count row (if any),
expandable plate list (click to toggle). Maintenance plates shown in orange.

```js
// Group by location for Summary view
// IMPORTANT: use vehAreaKey(v) — not v.area — so unmapped Thai-named areas are not dropped
const groups = {};
rows.forEach(v => {
  const loc = v.location || '(Unknown)';
  groups[loc] = groups[loc] || { plates: [], areas: {} };
  groups[loc].plates.push(v);
  const key = vehAreaKey(v);
  if (key) groups[loc].areas[key] = (groups[loc].areas[key] || 0) + 1;
});

// Maintenance count row per location card (shown if locMaint > 0)
// <div class="loc-maint-row">🔧 {locMaint} under maintenance</div>
```

**Maintenance CSS classes:**
```css
.status-maintenance { background:#f9731622; color:#f97316; border:1px solid #f9731655;
                      border-radius:6px; padding:2px 8px; font-size:10px; font-weight:700; }
.veh-plate-maint    { color:#f97316 !important; }
.loc-plate-maint    { background:#f9731615 !important; border-color:#f9731666 !important;
                      color:#f97316 !important; }
.loc-maint-row      { display:flex; align-items:center; gap:6px; margin-top:7px; padding-top:7px;
                      border-top:1px solid #f9731633; font-size:11px; color:#f97316; }
```

---

## Component: 👤 Drivers tab

Single sortable/filterable table. All filter dropdowns are populated dynamically from actual data.

**Filters:** search · Primary Pickup · Employment type · Shift type · Area

**Key JS pattern:**
```js
const DRIVERS      = REAL_DATA.drivers      || [];
const DRIVERS_WEEK = REAL_DATA.driversWeek  || '';

// Build location→area map (same as Vehicles tab)
function drvArea(rank1) {
  if (!rank1) return '';
  return LOC_TO_AREA[rank1] ||
    Object.keys(LOC_TO_AREA).find(k => rank1.includes(k) || k.includes(rank1)) || '';
}

// Populate ALL dropdowns dynamically from data
function initDrvFilters() {
  const empTypes   = [...new Set(DRIVERS.map(d => d.employmentType).filter(Boolean))].sort();
  const shiftTypes = [...new Set(DRIVERS.map(d => d.shiftType).filter(Boolean))].sort();
  const pickups    = [...new Set(DRIVERS.map(d => d.rank1).filter(Boolean))].sort();
  // append <option> elements to #drv-emp-filter, #drv-shift-filter, #drv-pickup-filter
  empTypes.forEach(t  => addOpt('drv-emp-filter',    t, t));
  shiftTypes.forEach(t => addOpt('drv-shift-filter', t, t));
  pickups.forEach(p   => addOpt('drv-pickup-filter', p, p));
}

function filteredDrivers() {
  const q      = document.getElementById('drv-search').value.toLowerCase();
  const emp    = document.getElementById('drv-emp-filter').value;
  const shift  = document.getElementById('drv-shift-filter').value;
  const area   = document.getElementById('drv-area-filter').value;
  const pickup = document.getElementById('drv-pickup-filter').value;
  return DRIVERS.filter(d => {
    if (emp    !== 'ALL' && d.employmentType !== emp)       return false;
    if (shift  !== 'ALL' && d.shiftType      !== shift)     return false;
    if (area   !== 'ALL' && drvArea(d.rank1) !== area)      return false;
    if (pickup !== 'ALL' && d.rank1          !== pickup)    return false;
    const hay = [d.driverId, d.driverCode, d.nameTh, d.heroId,
                 d.rank1, d.rank2, d.rank3].join(' ').toLowerCase();
    return !q || hay.includes(q);
  });
}
```

**Table columns:** Code + driverId · Thai name + heroId · Rank 1 (with area dot) · Rank 2 · Rank 3 · Employment type badge · Shift type badge · Updated date

**Area dot pattern** — use same `AREA_COLORS` palette as heatmap for visual consistency:
```js
function locCell(locName, isPrimary) {
  const area  = drvArea(locName);
  const color = area ? (AREA_COLORS[area] || '#6366f1') : '#475569';
  return `<span class="loc-dot" style="background:${color}"></span>${locName}` +
         (area && isPrimary ? ` <span style="color:${color}">(${area})</span>` : '');
}
```

**Sorting** — always use `localeCompare` with `{numeric: true}` so driver codes sort naturally (D001 < D010 < D100):
```js
rows.sort((a, b) => {
  const av = (a[sortCol] || '').toString();
  const bv = (b[sortCol] || '').toString();
  return sortAsc ? av.localeCompare(bv, undefined, {numeric: true})
                 : bv.localeCompare(av, undefined, {numeric: true});
});
```

---

## Component: 🔧 Maintenance in Availability tab

**KPI card** — placed alongside Planned/Adjusted/Actual cards:
```js
// Always visible regardless of date type
// <div class="kpi-card">🔧 {TOTAL_MAINT}<br><span>Maintenance</span></div>
```

**Shortage table** — extra column showing maintenance count per location:
```
| Location | Planned | Adjusted | Actual | 🔧 | Status |
```
```js
// Per location row:
const locMaint = MAINT_BY_LOC[loc.name] || 0;
// Render <td style="color:#f97316">{locMaint > 0 ? locMaint : '—'}</td>
```

**Detail chart header** — shows maintenance note when `locMaint > 0`:
```js
// <span style="color:#f97316; font-size:11px">🔧 {locMaint} under maintenance</span>
```
