---
name: skyspark-pub-ui
description: Build custom JavaScript views for SkySpark pub UI. Use this skill when working with SkySpark custom views, Chart.js visualizations in SkySpark, Haystack/Zinc API calls, or Axon query integration. Provides patterns for authentication, data fetching, and modular JavaScript without a bundler.
---
# SkySpark Pub UI Development Skill
This skill provides guidance for building custom JavaScript views that run in SkySpark's pub UI framework. These views render in the browser, fetch data via SkySpark's Haystack API, and display interactive visualizations.
## Architecture Overview

# SkySpark Pub UI Development Skill

This skill provides guidance for building custom JavaScript views that run in SkySpark's pub UI framework. These views render in the browser, fetch data via SkySpark's Haystack API, and display interactive visualizations.

## Architecture Overview

### File Locations
- **Pub UI files**: `{skyspark-var}/pub/ui/` — served at URL path `/pub/ui/`
- **View records**: Define views in SkySpark with `jsHandler` pointing to your global function
- **No restart required**: Files added to `pub/ui/` are available immediately — SkySpark serves them dynamically, no restart needed

### Root-Only Auto-Discovery (CRITICAL)
**CRITICAL**: SkySpark only auto-discovers and loads JS files from the **root** of `{var}/pub/ui/`. Files in subdirectories are completely invisible to SkySpark's scanner.

- **Entry files MUST be at the root**: `{var}/pub/ui/<appName>Entry.js` — always, on every server
- **Handler/module files CAN live in subdirectories**: e.g., `{var}/pub/ui/<appName>/<appName>Handler.js` — because they are loaded dynamically by the entry file via `<script>` injection, not by SkySpark's auto-discovery

**Symptom**: View works when entry file is at root, breaks when moved into a subdirectory
**Cause**: SkySpark cannot discover JS files below the pub/ui/ root
**Fix**: Always keep entry files at `{var}/pub/ui/` root; subdirectories are fine for handler/module files

### File Mount Setup (When Direct Server Access Is Unavailable)
To manage pub/ui files from within SkySpark's file browser, configure a file mount:
### File Mount Setup (When Direct Server Access Is Unavailable)
To manage pub/ui files from within SkySpark's file browser, configure a file mount:

1. Open the **Debug** app in SkySpark and find the **Var Dir** value (e.g., `C:\skyspark-3.1.10\var`)
2. Convert the path to a Unix-style localPath:
   - Remove the drive letter (`C:`)
   - Flip backslashes to forward slashes
   - Append `/pub/ui/` at the end
   - Result: `/skyspark-3.1.10/var/pub/ui/`
3. Create a file mount record with:
   - `path`: `/ui/` (or your preferred virtual path)
   - `localPath`: the converted path above
> **Note**: Only set up a file mount if the server does not already have a pub/ui entry file. If it does, the file mount process is not needed — manage files directly.
### Standard Folder Structure
All pub UI apps should use this layout in `{skyspark-var}/pub/ui/`:
```
/pub/ui/
├── <appName>Entry.js          # Entry point — thin loader, MUST be at root on every server
├── <appName>/
│   ├── <appName>Handler.js    # Main handler — loaded dynamically, can live in subdir

> **Note**: Only set up a file mount if the server does not already have a pub/ui entry file. If it does, the file mount process is not needed — manage files directly.

### Standard Folder Structure
All pub UI apps should use this layout in `{skyspark-var}/pub/ui/`:

```
/pub/ui/
├── <appName>Entry.js          # Entry point — thin loader, deployed per server
├── <appName>/
│   ├── <appName>UI.js         # UI module — loads all deps, initializes React
│   ├── <appName>Styles.css    # All CSS styles (single file, scoped with #<appId>)
│   ├── App.js                 # Root React component
│   ├── components/            # React components (one per file)
│   │   ├── Header.js
│   │   ├── Toast.js
│   │   └── ...
│   ├── constants/             # Static config (field definitions, enums)
│   │   └── fields.js
│   ├── evals/                 # Axon eval wrappers (one per server operation)
│   │   ├── loadParts.js
│   │   ├── savePart.js
│   │   └── ...
│   ├── hooks/                 # React custom hooks
│   │   ├── useToast.js
│   │   └── ...
│   └── utils/                 # Pure utility functions
│       ├── api.js
│       ├── validation.js
│       └── ...
```
**Key principle**: The full folder structure and view live on the cloud server. Each individual SkySpark server only needs the thin `<appName>Entry.js` file at the pub/ui/ root — check if one already exists before setting up a file mount.

### Module Pattern (No Bundler)
Since SkySpark pub UI runs vanilla JavaScript without build tools, use a global namespace pattern:

**Key principle**: The full folder structure and view live on the cloud server. Each individual SkySpark server only needs the thin `<appName>Entry.js` file — check if one already exists before setting up a file mount.

### Module Pattern (No Bundler)
Since SkySpark pub UI runs vanilla JavaScript without build tools, use a global namespace pattern:

```javascript
window.MyView = window.MyView || {};
window.MyView.state = { /* shared state */ };
window.MyView.api = { /* API functions */ };
window.MyView.chart = { /* chart functions */ };

// Entry point called by SkySpark
window.MyViewHandler = {
  onUpdate: function(arg) {
    var view = arg.view;
    var elem = arg.elem;
    // Build your view here
  }
};
```

### Handler Declaration Pattern (CRITICAL)
**CRITICAL**: SkySpark expects a global handler object. The handler must NOT be declared inside an IIFE — it must be at global scope:
```javascript
// CORRECT: Global var at top level, before IIFE

**CRITICAL**: SkySpark expects a global handler object. When using an IIFE, the handler MUST be declared as a global `var` BEFORE the IIFE:

```javascript
// CORRECT: Global var declared before IIFE
var myHandlerName = {};
(function() {
  myHandlerName.onUpdate = function(arg) {
    // ... implementation
  };
})();

// ALSO CORRECT: Global var declared after a separate IIFE (still at global scope)
(function() {
  // load scripts, setup, etc. — handler not declared here
})();
var myHandlerName = {};
myHandlerName.onUpdate = function(arg) { ... };

// WRONG: Handler defined inside IIFE is not globally accessible
(function() {
  var myHandlerName = {};  // NOT accessible to SkySpark!
  myHandlerName.onUpdate = function(arg) { ... };
})();
```
**Symptom**: `jsView handler not found: myHandlerName` error in console
**Cause**: Handler object is scoped inside IIFE, not globally accessible
**Fix**: Declare handler as `var myHandlerName = {};` at global scope — before OR after a separate IIFE, just never inside one

### Entry File and Module Global Naming (CRITICAL)
**CRITICAL**: The entry file's handler global and the cloud module's exposed global must use **different names** to avoid collision.

- **Entry file** defines the `jsHandler` global that SkySpark looks for (e.g., `var myAppHandler = {}`)
- **Cloud module** exposes itself under a separate app global (e.g., `window.myApp = myAppHandler`)
- **Entry file's `onUpdate`** delegates to the module global

```javascript
// In the cloud handler file — at the very end:
window.myApp = myAppHandler;  // expose under a different name than the jsHandler

// In the entry file:
var myAppHandler = {};  // this is what jsHandler in the trio points to
myAppHandler.onUpdate = function(arg) {
  var view = arg.view;
  var elem = arg.elem;
  view.removeAll();
  if (window.myApp && typeof window.myApp.onUpdate === 'function') {
    window.myApp.onUpdate(arg);
  }
};
```

**Symptom**: Handler overwrites entry file stub when module loads, causing unpredictable behavior
**Cause**: Entry file and cloud module both define the same global name
**Fix**: Module exposes itself under `window.<appName>App`; entry file delegates to that

**Symptom**: `jsView handler not found: myHandlerName` error in console
**Cause**: Handler object is scoped inside IIFE, not globally accessible
**Fix**: Declare handler as `var myHandlerName = {};` BEFORE the IIFE opening

### View Record (Trio Format)
```trio
dis: "My Custom View"
view
jsHandler: { var defVal:"MyViewHandler" }
```

## Deployment Architecture
### Two-Server Pattern
When hosting modular JS outside SkySpark (e.g., on a cloud server), there are two servers involved:
1. **Every SkySpark server** (`{var}/pub/ui/`): Entry file only — thin loader at the root, one per app per server
2. **Cloud server** (e.g., `http://34.212.16.148/pub/ui/`): Full module directory with all handler/module files

**Deployment layout**:
```
Local SkySpark server:
  {var}/pub/ui/
  └── myAppEntry.js            ← MUST be at root

Cloud SkySpark server:
  {var}/pub/ui/
  ├── myAppEntry.js            ← MUST be at root (same file as local)
  └── myApp/
      └── myAppHandler.js     ← loaded dynamically, safe in subdir
```

**Key rules**:
- Entry file goes to root `{var}/pub/ui/` on **all** servers — local and cloud
- The entry file is identical on all servers (use a relative URL so it works everywhere)
- Handler/modules only need to exist on the cloud server
- Relative URL in entry file resolves to whichever server is serving the page — works on cloud, silently fails on local (acceptable if users access via cloud)
- No restart needed after deploying files to pub/ui/

**Deployment checklist**:
- Before adding an entry file to a SkySpark server, check if one already exists — each server only needs one entry file per app
- If no entry file exists and you lack direct server access, use the file mount process to deploy it
- Entry file must always be at the pub/ui/ ROOT — never in a subdirectory

### Confirmed Working Entry File Pattern
This pattern is confirmed working and matches real-world deployments:
```javascript
// myAppEntry.js
// Deploy to: {var}/pub/ui/ ROOT on every server (local and cloud)
// SkySpark only auto-discovers JS at pub/ui/ root — subdirs are ignored.
// The handler is loaded dynamically so it can live in a subdirectory on the cloud.
//
// View record (trio) jsHandler should point to: myAppHandler

(function () {
  var src = '/pub/ui/myApp/myAppHandler.js';
  var script = document.createElement('script');
  script.src = src;
  script.async = false;
  script.onload = function () {
    console.log('[myApp] Handler loaded. window.myApp:', typeof window.myApp);
  };
  script.onerror = function () {
    console.error('[myApp] Failed to load handler from:', src);
  };
  document.head.appendChild(script);
})();

var myAppHandler = {};

myAppHandler.onUpdate = function (arg) {
  var view = arg.view;
  var elem = arg.elem;
  view.removeAll();
  if (window.myApp && typeof window.myApp.onUpdate === 'function') {
    window.myApp.onUpdate(arg);
  } else {
    console.warn('[myApp] onUpdate called but window.myApp not ready yet.');
  }
};
```

And in the cloud handler file, at the very end:
```javascript
// Expose under the app global that the entry file delegates to
window.myApp = myAppHandler;
console.log('[myApp] Handler ready. window.myApp exposed.');
```

### Modular Loader Entry File Pattern (Complex Apps)
For apps with multiple JS modules to load sequentially:
```javascript
var myHandler = {};
(function() {
  var BASE_URL = 'http://my-server/pub/ui/myProject_v3/';
  var modules = ['core/config.js', 'data/api.js', 'index.js'];
  var loaded = false, loading = false, pendingCalls = [];
  function loadModules(cb) {
    var i = 0;
    function next() {
      if (i >= modules.length) { cb(); return; }
      var s = document.createElement('script');
      s.src = BASE_URL + modules[i];
      s.onload = function() { i++; next(); };
      s.onerror = function() { i++; next(); };
      document.head.appendChild(s);
    }
    next();
  }
  myHandler.onUpdate = function(arg) {
    if (loaded) { window.MyProject.onUpdate(arg); return; }
    pendingCalls.push(arg);
    if (!loading) {
      loading = true;
      loadModules(function() {
        loaded = true; loading = false;
        pendingCalls.forEach(function(a) { window.MyProject.onUpdate(a); });
        pendingCalls = [];
      });
    }
  };
})();
```

### Versioning with Directory Copies
When creating a new version (e.g., v2 → v3), copy the entire directory and update ALL internal references:
**Files to update when versioning:**
1. `core/loader.js` — path marker used to resolve base URL (e.g., `eventAnnotationsPlot_v2/` → `eventAnnotationsPlot_v3/`)
2. Entry files — BASE_URL, handler name, module list, bundle filename
3. `build.ps1` — output filename, header comments
4. `index.js` — canvas IDs, version labels
5. Any hardcoded path references in other modules
**CRITICAL**: The loader's `_resolveBasePath()` searches `<script>` src attributes for a directory marker. If this still says `v2/` in a v3 build, Chart.js and vendor files will fail to load silently.

## SkySpark Session & Authentication
**CRITICAL**: Use `view.session()` to get session credentials, NOT `fan.haystack.ui.HSession.cur()`.
## SkySpark Session & Authentication

**CRITICAL**: Use `view.session()` to get session credentials, NOT `fan.haystack.ui.HSession.cur()`.

```javascript
var session = view.session();
var attestKey = session.attestKey();
var projectName = session.proj().name();
```
## Haystack API Calls
### Eval Endpoint
POST to `/api/{projectName}/eval` with Zinc-formatted body:
```javascript
function evalAxon(axonExpr) {
  var body = 'ver: "3.0"\nexpr\n"' + axonExpr.replace(/"/g, '\\"') + '"';

## Haystack API Calls

### Eval Endpoint
POST to `/api/{projectName}/eval` with Zinc-formatted body:

```javascript
function evalAxon(axonExpr) {
  var body = 'ver: "3.0"\nexpr\n"' + axonExpr.replace(/"/g, '\\"') + '"';

  return fetch('/api/' + projectName + '/eval', {
    method: 'POST',
    headers: {
      'Content-Type': 'text/zinc',
      'Accept': 'application/json',
      'Attest-Key': attestKey
    },
    body: body
  }).then(function(r) { return r.json(); });
}
```
### Unwrapping Nested Grids
The eval endpoint often wraps results in a single-row grid with a `val` column:

### Unwrapping Nested Grids
The eval endpoint often wraps results in a single-row grid with a `val` column:

```javascript
.then(function(data) {
  // Unwrap nested grid if present
  if (data.rows && data.rows.length === 1 &&
      data.cols && data.cols.length === 1 &&
      data.cols[0].name === 'val') {
    var inner = data.rows[0].val;
    if (inner && inner.rows && inner.cols) {
      data = inner;
    }
  }
  return data;
});
```
### Haystack Value Extraction
Haystack JSON wraps values in objects with `_kind` indicating the type:
```javascript
// Haystack Number with unit
{ "_kind": "number", "val": 123.45, "unit": "kW" }
// Haystack Ref
{ "_kind": "ref", "val": "p:project:r:abc123", "dis": "Display Name" }
// Haystack DateTime
{ "_kind": "dateTime", "val": "2026-02-03T06:00:00-08:00", "tz": "Los_Angeles" }

### Haystack Value Extraction
Haystack JSON wraps values in objects with `_kind` indicating the type:

```javascript
// Haystack Number with unit
{ "_kind": "number", "val": 123.45, "unit": "kW" }

// Haystack Ref
{ "_kind": "ref", "val": "p:project:r:abc123", "dis": "Display Name" }

// Haystack DateTime
{ "_kind": "dateTime", "val": "2026-02-03T06:00:00-08:00", "tz": "Los_Angeles" }

// Universal value extractor
function extractValue(val) {
  if (!val) return null;
  if (val._kind === 'number') return val.val;
  if (val._kind === 'dateTime') return val.val;  // Returns ISO string
  if (val._kind === 'ref') return val.val;
  if (val.val !== undefined) return val.val;
  return val;
}
```
### Fantom List Iteration (CRITICAL)
**CRITICAL**: SkySpark returns Fantom objects, not JavaScript arrays. Use Fantom methods:

### Fantom List Iteration (CRITICAL)

**CRITICAL**: SkySpark returns Fantom objects, not JavaScript arrays. Use Fantom methods:

```javascript
// WRONG: JavaScript array methods
var rows = data.rows();
rows.length;     // undefined!
rows[0];         // undefined!

// CORRECT: Fantom List methods
var rows = data.rows();
var count = rows.size();      // Use size() not length
var firstRow = rows.get(0);   // Use get(i) not [i]
```
**Symptom**: `undefined` when accessing rows or checking length
**Cause**: Using JavaScript array syntax on Fantom List objects
**Fix**: Use `.size()` instead of `.length`, and `.get(i)` instead of `[i]`
## Reading SkySpark Variables
Access view variables set by the SkySpark UI:

**Symptom**: `undefined` when accessing rows or checking length
**Cause**: Using JavaScript array syntax on Fantom List objects
**Fix**: Use `.size()` instead of `.length`, and `.get(i)` instead of `[i]`

## Reading SkySpark Variables

Access view variables set by the SkySpark UI:

```javascript
function tryReadVar(view, varName) {
  try {
    var val = view.get(varName);
    if (val) return val.toStr ? val.toStr() : String(val);
  } catch (e) {}
  return null;
}

// Common variables
var siteRef = tryReadVar(view, 'site') || tryReadVar(view.parent(), 'site');
var dateRange = tryReadVar(view, 'dateRange') || tryReadVar(view.parent(), 'dateRange');
```
### Parsing Date Ranges
SkySpark date ranges come as `"2025-01-01..2025-01-31"`:

### Parsing Date Ranges
SkySpark date ranges come as `"2025-01-01..2025-01-31"`:

```javascript
if (dateRange && dateRange.indexOf('..') !== -1) {
  var parts = dateRange.split('..');
  var startDate = parts[0];
  var endDate = parts[1];
}
```
## Chart.js Integration
### Loading Chart.js
Load from local vendor files or CDN:
```javascript
function loadChartJs(callback) {
  if (window.Chart) { callback(); return; }

## Chart.js Integration

### Loading Chart.js
Load from local vendor files or CDN:

```javascript
function loadChartJs(callback) {
  if (window.Chart) { callback(); return; }

  var scripts = [
    'chart.umd.min.js',
    'chartjs-adapter-date-fns.bundle.min.js',
    'chartjs-plugin-annotation.min.js'
  ];

  function loadNext(index) {
    if (index >= scripts.length) { callback(); return; }
    var script = document.createElement('script');
    script.src = '/pub/ui/vendor/' + scripts[index];
    script.onload = function() { loadNext(index + 1); };
    document.head.appendChild(script);
  }
  loadNext(0);
}
```

### Time-Series Chart Config
```javascript
new Chart(ctx, {
  type: 'line',
  data: { datasets: [{ label: 'Power (kW)', data: points }] },
  options: {
    responsive: true,
    maintainAspectRatio: false,
    scales: {
      x: {
        type: 'time',
        time: { unit: 'day', displayFormats: { day: 'MMM dd' } }
      },
      y: { title: { display: true, text: 'Power (kW)' } }
    }
  }
});
```
## Performance Best Practices

## Performance Best Practices

### Minimize API Calls
- Batch multiple queries into a single Axon expression returning a dict
- Cache data in `state` to avoid refetching on UI toggle changes
- Filter data server-side in Axon, not client-side in JavaScript

**BAD** — 5 separate API calls:
```javascript
Promise.all([
  evalAxon('getSiteName(site)'),
  evalAxon('getPowerData(site, dates)'),
  evalAxon('getEvents(site, dates)'),
  evalAxon('getCost1(site, dates)'),
  evalAxon('getCost2(site, dates)')
])
```

**GOOD** — 1 API call returning all data:
```javascript
evalAxon('{' +
  'siteName: getSiteName(site), ' +
  'power: getPowerData(site, dates), ' +
  'events: getEvents(site, dates), ' +
  'cost1: getCost1(site, dates), ' +
  'cost2: getCost2(site, dates)' +
'}')
```

### Avoid `readAll` Without Filters
**BAD** — fetches entire database:
```javascript
'readAll(event)'
```

**GOOD** — filter server-side:
```javascript
'readAll(event).findAll(x => x->siteRef == site and x->startDate >= start and x->endDate <= end)'
```
### Bundle Modules for Production
Instead of loading 15 separate JS files sequentially (adding 1-3s of network latency), concatenate into a single bundle:
```bash
cat config.js loader.js api.js chart.js index.js > myview.bundle.js
```
### Throttle Mouse Handlers
Canvas mousemove events fire 60+ times/sec. Use `requestAnimationFrame`:

### Bundle Modules for Production
Instead of loading 15 separate JS files sequentially (adding 1-3s of network latency), concatenate into a single bundle:

```bash
cat config.js loader.js api.js chart.js index.js > myview.bundle.js
```

### Throttle Mouse Handlers
Canvas mousemove events fire 60+ times/sec. Use `requestAnimationFrame`:

```javascript
var pending = null;
canvas.addEventListener('mousemove', function(e) {
  if (pending) return;
  pending = requestAnimationFrame(function() {
    pending = null;
    handleHover(e);
  });
});
```
### Avoid Redundant Chart Recreation
Don't create an empty chart that's immediately destroyed when data arrives. Create the chart once with real data:

### Avoid Redundant Chart Recreation
Don't create an empty chart that's immediately destroyed when data arrives. Create the chart once with real data:

```javascript
// BAD - creates empty chart, then destroys and recreates
chart.createChart(canvas, [], []);  // empty
// ...later...
chart.createChart(canvas, data, events);  // destroys empty, creates new

// GOOD - show placeholder, create chart once with data
showPlaceholder();
loadData().then(function(data) {
  hidePlaceholder();
  chart.createChart(canvas, data, events);  // only creation
});
```
## Canvas Overlay Alignment
When drawing on an overlay canvas that sits on top of a Chart.js chart, sync dimensions:

## Canvas Overlay Alignment

When drawing on an overlay canvas that sits on top of a Chart.js chart, sync dimensions:

```javascript
function syncOverlaySize() {
  // Sync pixel buffer dimensions
  overlay.width = chartCanvas.width;
  overlay.height = chartCanvas.height;
  // Sync CSS display dimensions
  var rect = chartCanvas.getBoundingClientRect();
  overlay.style.width = rect.width + 'px';
  overlay.style.height = rect.height + 'px';
}
// Call after chart creation and on window resize
window.addEventListener('resize', syncOverlaySize);
```
### DevicePixelRatio Scaling (CRITICAL)
**CRITICAL**: Chart.js reports `chartArea` coordinates in CSS space, but the canvas pixel buffer is scaled by `devicePixelRatio`. When drawing on overlay canvases, you MUST scale coordinates:
```javascript
var dpr = window.devicePixelRatio || 1;
var rawChartArea = chartInstance.chartArea;

// Call after chart creation and on window resize
window.addEventListener('resize', syncOverlaySize);
```

### DevicePixelRatio Scaling (CRITICAL)

**CRITICAL**: Chart.js reports `chartArea` coordinates in CSS space, but the canvas pixel buffer is scaled by `devicePixelRatio`. When drawing on overlay canvases, you MUST scale coordinates:

```javascript
var dpr = window.devicePixelRatio || 1;
var rawChartArea = chartInstance.chartArea;

// Scale chartArea for pixel buffer drawing
var chartArea = {
  top: rawChartArea.top * dpr,
  bottom: rawChartArea.bottom * dpr,
  left: rawChartArea.left * dpr,
  right: rawChartArea.right * dpr
};
// Scale x-coordinates from xScale
var xPixel = xScale.getPixelForValue(dateValue) * dpr;

// Scale x-coordinates from xScale
var xPixel = xScale.getPixelForValue(dateValue) * dpr;

// Scale mouse coordinates for tooltips
var scaledMouseX = mouseX * dpr;
var scaledMouseY = mouseY * dpr;
```
**Symptom**: Annotations stop partway down the chart (e.g., at 60% height on a 1.5x DPR display)
**Cause**: Drawing at CSS coordinates (e.g., y=416) on a scaled pixel buffer (e.g., height=705)
**Fix**: Multiply all Chart.js coordinates by `devicePixelRatio`
### Explicit X-Axis Date Range Bounds
To ensure the chart shows a specific date range (not auto-fit to data), set explicit min/max:

**Symptom**: Annotations stop partway down the chart (e.g., at 60% height on a 1.5x DPR display)
**Cause**: Drawing at CSS coordinates (e.g., y=416) on a scaled pixel buffer (e.g., height=705)
**Fix**: Multiply all Chart.js coordinates by `devicePixelRatio`

### Explicit X-Axis Date Range Bounds

To ensure the chart shows a specific date range (not auto-fit to data), set explicit min/max:

```javascript
scales: {
  x: {
    type: 'time',
    min: new Date(startDate).getTime(),
    max: new Date(endDate + 'T23:59:59').getTime()
  }
}
```
**Symptom**: Data line doesn't start at the left edge of the chart; events appear before data starts
**Cause**: Chart.js auto-fits x-axis to the data range, not the requested date range
**Fix**: Pass `dateRange` to chart creation and set explicit `min`/`max` on x-axis scale
## Haystack Currency Values
Axon functions may return currency values. These come through the JSON API in multiple formats:
```javascript
// Raw number (most common for cost columns in grids)
{ "val": 215663 }
// Haystack Number with unit
{ "_kind": "number", "val": 215663, "unit": "$" }
// String-formatted currency (from display functions)
"$215,663"

**Symptom**: Data line doesn't start at the left edge of the chart; events appear before data starts
**Cause**: Chart.js auto-fits x-axis to the data range, not the requested date range
**Fix**: Pass `dateRange` to chart creation and set explicit `min`/`max` on x-axis scale

## Deployment Architecture

### Two-Server Pattern
When hosting modular JS outside SkySpark (e.g., on a cloud server), there are two servers involved:

1. **SkySpark server** (`{var}/pub/ui/`): Entry files only (thin loaders, one per app per server)
2. **Cloud/CDN server** (e.g., `http://34.212.16.148/pub/ui/`): Full module directories with the complete folder structure

**Deployment checklist**:
- Before adding an entry file to a SkySpark server, check if one already exists — each server only needs one entry file per app
- If no entry file exists and you lack direct server access, use the file mount process to deploy it
- The entry file on the SkySpark server points to module files hosted on the cloud server

### Entry File Pattern (Modular Loader)
```javascript
var myHandler = {};
(function() {
  var BASE_URL = 'http://my-server/pub/ui/myProject_v3/';
  var modules = ['core/config.js', 'data/api.js', 'index.js'];
  var loaded = false, loading = false, pendingCalls = [];

  function loadModules(cb) {
    var i = 0;
    function next() {
      if (i >= modules.length) { cb(); return; }
      var s = document.createElement('script');
      s.src = BASE_URL + modules[i];
      s.onload = function() { i++; next(); };
      s.onerror = function() { i++; next(); };
      document.head.appendChild(s);
    }
    next();
  }

  myHandler.onUpdate = function(arg) {
    if (loaded) { window.MyProject.onUpdate(arg); return; }
    pendingCalls.push(arg);
    if (!loading) {
      loading = true;
      loadModules(function() {
        loaded = true; loading = false;
        pendingCalls.forEach(function(a) { window.MyProject.onUpdate(a); });
        pendingCalls = [];
      });
    }
  };
})();
```

### Versioning with Directory Copies
When creating a new version (e.g., v2 → v3), copy the entire directory and update ALL internal references:

**Files to update when versioning:**
1. `core/loader.js` — path marker used to resolve base URL (e.g., `eventAnnotationsPlot_v2/` → `eventAnnotationsPlot_v3/`)
2. Entry files — BASE_URL, handler name, module list, bundle filename
3. `build.ps1` — output filename, header comments
4. `index.js` — canvas IDs, version labels
5. Any hardcoded path references in other modules

**CRITICAL**: The loader's `_resolveBasePath()` searches `<script>` src attributes for a directory marker. If this still says `v2/` in a v3 build, Chart.js and vendor files will fail to load silently.

## Haystack Currency Values

Axon functions may return currency values. These come through the JSON API in multiple formats:

```javascript
// Raw number (most common for cost columns in grids)
{ "val": 215663 }

// Haystack Number with unit
{ "_kind": "number", "val": 215663, "unit": "$" }

// String-formatted currency (from display functions)
"$215,663"

// Universal currency parser
function parseCurrencyValue(raw) {
  var val = extractValue(raw);  // unwrap _kind wrapper
  if (typeof val === 'number') return val;
  if (typeof val === 'string') {
    return parseFloat(val.replace(/[\$,\s]/g, '')) || 0;
  }
  return 0;
}
```
**Symptom**: Cost widget shows `$0` or `$NaN` despite API returning valid data
**Cause**: Value is a currency string like `"$215,663"` — `parseFloat` fails on `$` and commas
**Fix**: Strip `$`, commas, and spaces before parsing
## Common Pitfalls
1. **Handler not found**: Declare handler as global `var` at top level — before or after a separate IIFE, never inside one

**Symptom**: Cost widget shows `$0` or `$NaN` despite API returning valid data
**Cause**: Value is a currency string like `"$215,663"` — `parseFloat` fails on `$` and commas
**Fix**: Strip `$`, commas, and spaces before parsing

## Common Pitfalls

1. **Handler not found**: Declare handler as global `var` BEFORE IIFE: `var myHandler = {};`
2. **Wrong session API**: Use `view.session()`, not `fan.haystack.ui.HSession.cur()`
3. **Missing `/pub/` in URLs**: Files at `{var}/pub/ui/` are served at `/pub/ui/`, not `/ui/`
4. **Nested grid responses**: Always check for and unwrap the `val` wrapper
5. **Date range spans**: Axon date ranges use `..` syntax: `2025-01-01..2025-01-31`
6. **Polling accumulation**: Clear previous `setInterval` before starting a new one
7. **Canvas cloning**: `cloneNode` on canvas elements loses pixel content — use `removeEventListener` instead
8. **Annotations stop mid-chart**: Scale `chartArea` coordinates by `devicePixelRatio` when drawing on overlay canvas
9. **Data doesn't fill x-axis**: Set explicit `min`/`max` on x-axis scale from requested date range
10. **Fantom List iteration fails**: Use `.size()` and `.get(i)`, not `.length` and `[i]`
11. **DateTime parsing fails**: Extract `.val` from Haystack `{_kind: "dateTime", val: "..."}` objects
12. **Path resolution fails after versioning**: When copying a project directory (v2 → v3), update the folder marker in `loader.js` — it scans `<script>` tags to find the base path
13. **Duplicate entry files**: Each SkySpark server only needs one `<appName>Entry.js` — check before deploying a new one; no restart is needed after adding it
14. **Wrong file mount localPath**: Get the Var Dir from the Debug app, strip the drive letter, flip backslashes, and append `/pub/ui/` — a wrong path means files are not written where SkySpark serves them from
15. **Currency values parse as $0**: Haystack may return `"$215,663"` strings — strip `$` and commas before `parseFloat`
16. **Tooltips show for hidden events**: When implementing event visibility toggles, also filter hidden events from hover detection, not just from rendering
17. **Global function name conflicts**: In shared namespace, prefix helpers with your module path (e.g., `window.MyProject.myModule.myHelper`) to avoid collisions with other pub/ui scripts
18. **Entry file in subdirectory not discovered**: SkySpark only scans the pub/ui/ root — moving the entry file into a subdirectory breaks the view silently
19. **Entry file and module share the same global name**: Entry file defines the jsHandler global; the cloud module must expose itself under a different name (e.g., `window.myApp`) to avoid overwriting the stub before delegation is set up
20. **Relative URL resolves to wrong server**: A relative `/pub/ui/...` path in the entry file resolves to whichever server is currently serving the page. On the cloud server this is correct; on a local server it will 404 if the handler isn't deployed locally (acceptable if users primarily access via cloud)

## Example: Minimal Working View
```javascript
window.MinimalView = window.MinimalView || {};
window.MinimalView.state = {};

## Example: Minimal Working View

```javascript
window.MinimalView = window.MinimalView || {};
window.MinimalView.state = {};

window.MinimalViewHandler = {
  onUpdate: function(arg) {
    var view = arg.view;
    var elem = arg.elem;
    var state = window.MinimalView.state;
    view.removeAll();

    view.removeAll();

    // Get session
    var session = view.session();
    state.attestKey = session.attestKey();
    state.projectName = session.proj().name();

    // Build UI
    var container = document.createElement('div');
    container.style.padding = '20px';
    elem.appendChild(container);
    var title = document.createElement('h2');
    title.textContent = 'Loading...';
    container.appendChild(title);
    // Fetch data
    var axon = 'readById(@mySite).dis';
    var body = 'ver: "3.0"\nexpr\n"' + axon.replace(/"/g, '\\"') + '"';

    var title = document.createElement('h2');
    title.textContent = 'Loading...';
    container.appendChild(title);

    // Fetch data
    var axon = 'readById(@mySite).dis';
    var body = 'ver: "3.0"\nexpr\n"' + axon.replace(/"/g, '\\"') + '"';

    fetch('/api/' + state.projectName + '/eval', {
      method: 'POST',
      headers: {
        'Content-Type': 'text/zinc',
        'Accept': 'application/json',
        'Attest-Key': state.attestKey
      },
      body: body
    })
    .then(function(r) { return r.json(); })
    .then(function(data) {
      if (data.rows && data.rows[0]) {
        title.textContent = data.rows[0].val || 'Site';
      }
    })
    .catch(function(err) {
      title.textContent = 'Error: ' + err.message;
    });
  }
};
```
