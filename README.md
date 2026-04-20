
# SafeTrax Auto-Routing Intervention Analysis

## What this is

This document explains the analysis of how much ops manually corrects SafeTrax auto-routing output before trips run. The analysis covers 7,351 auto-routed trips across 11 clients over 30 days (March 22 – April 20, 2026).

---

## How to run the query

Run the MongoDB query below on any SafeTrax production instance. It pulls from the `trips` collection only.

```javascript
var now = new Date();
var endTs = now.getTime();
var startTs = endTs - (30 * 24 * 60 * 60 * 1000);

var summary = {};
var grandTotal = 0, grandIntervened = 0, grandOps = 0, grandData = 0;

print("fabricatedTripId,siteId,scheduleDate,tripLabel,shiftTime,employees,empAdd,empDel,empDelRost,empDelProf,opsIntervened,dataIntervened");

db.trips.find(
  {
    "scheduleDate": {
      "$gte": NumberLong(startTs.toString()),
      "$lt": NumberLong(endTs.toString())
    },
    "stateOfTrip": "deployed",
    "runningStatus": "completed",
    "editLogs.message": { "$regex": "From Multi Routing Approve" }
  },
  {
    "fabricatedTripId": 1, "_siteId": 1, "scheduleDate": 1,
    "tripLabel": 1, "startTime": 1, "endTime": 1,
    "editLogs.message": 1, "employees": 1, "_id": 0
  }
).forEach(function(doc) {
  if (!doc.scheduleDate || !doc.employees) return;

  var empAdd = 0, empDel = 0, empDelRost = 0, empDelProf = 0;

  for (var edoc of doc.editLogs) {
    if (!edoc.message) continue;
    var msg = edoc.message;
    if (msg.includes("Added to Trip")) {
      empAdd++;
    } else if (msg.includes("Removed: Roster Cancellation") ||
               msg.includes("Removed: selfrostering") ||
               msg.includes("Removed: adminrostering")) {
      empDelRost++;
    } else if (msg.includes("Removed: : Removed, inactive/non-transport user") ||
               msg.includes("Removed: : Removed, Privilege changed") ||
               msg.includes("Removed: : Removed, team changed") ||
               msg.includes("Removed: : Removed, stoppage changed")) {
      empDelProf++;
    } else if (msg.includes("Removed: DEFT") ||
               msg.includes("Removed: Removed using delete")) {
      empDel++;
    }
  }

  var opsIntervened = (empAdd + empDel) > 0 ? 1 : 0;
  var dataIntervened = (empDelRost + empDelProf) > 0 ? 1 : 0;
  var anyIntervened = (empAdd + empDel + empDelRost + empDelProf) > 0 ? 1 : 0;

  var siteId = doc._siteId || "unknown";
  var scheduleDate = new Date(doc.scheduleDate.toNumber()).toLocaleDateString();
  var shiftDateTime = doc.tripLabel && doc.tripLabel.includes("login") ? doc.endTime : doc.startTime;
  var shiftTime = shiftDateTime ? new Date(shiftDateTime.toNumber()).toTimeString().substring(0, 5) : "N/A";

  print(
    doc.fabricatedTripId + "," + siteId + "," + scheduleDate + "," +
    doc.tripLabel + "," + shiftTime + "," + doc.employees.length + "," +
    empAdd + "," + empDel + "," + empDelRost + "," + empDelProf + "," +
    (opsIntervened ? "YES" : "NO") + "," + (dataIntervened ? "YES" : "NO")
  );

  var key = siteId + "|" + doc.tripLabel + "|" + shiftTime;
  if (!summary[key]) {
    summary[key] = { siteId, tripLabel: doc.tripLabel, shiftTime,
      totalTrips:0, totalEmployees:0, opsIntervened:0, dataIntervened:0,
      anyIntervened:0, empAdd:0, empDel:0, empDelRost:0, empDelProf:0 };
  }
  summary[key].totalTrips++;
  summary[key].totalEmployees += doc.employees.length;
  summary[key].opsIntervened += opsIntervened;
  summary[key].dataIntervened += dataIntervened;
  summary[key].anyIntervened += anyIntervened;
  summary[key].empAdd += empAdd;
  summary[key].empDel += empDel;
  summary[key].empDelRost += empDelRost;
  summary[key].empDelProf += empDelProf;

  grandTotal++;
  grandIntervened += anyIntervened;
  grandOps += opsIntervened;
  grandData += dataIntervened;
});

print("\n--- SITE + SHIFT SUMMARY ---");
print("siteId,tripLabel,shiftTime,totalTrips,totalEmployees,opsIntervened,opsPct,dataIntervened,dataPct,anyIntervened,anyPct,empAdd,empDel,empDelRost,empDelProf");
for (var key in summary) {
  var s = summary[key];
  print(s.siteId+","+s.tripLabel+","+s.shiftTime+","+s.totalTrips+","+s.totalEmployees+","+
    s.opsIntervened+","+(s.opsIntervened/s.totalTrips*100).toFixed(1)+"%,"+
    s.dataIntervened+","+(s.dataIntervened/s.totalTrips*100).toFixed(1)+"%,"+
    s.anyIntervened+","+(s.anyIntervened/s.totalTrips*100).toFixed(1)+"%,"+
    s.empAdd+","+s.empDel+","+s.empDelRost+","+s.empDelProf);
}

print("\n--- OVERALL SUMMARY ---");
print("totalAutoRoutedTrips,totalIntervened,anyInterventionPct,opsInterventionPct,dataInterventionPct");
print(grandTotal+","+grandIntervened+","+(grandTotal>0?(grandIntervened/grandTotal*100).toFixed(1):"N/A")+"%,"+
  (grandTotal>0?(grandOps/grandTotal*100).toFixed(1):"N/A")+"%,"+
  (grandTotal>0?(grandData/grandTotal*100).toFixed(1):"N/A")+"%");
```

**Note for demo/test instances:** Remove `"stateOfTrip": "deployed", "runningStatus": "completed"` from the filter — trips on demo accounts are never completed.

---

## Output columns explained

| Column | What it means |
|---|---|
| `fabricatedTripId` | Unique trip identifier |
| `siteId` | Client identifier (MongoDB `_siteId`) |
| `scheduleDate` | Date the trip was scheduled to run |
| `tripLabel` | `login` (going to office) or `logout` (going home) |
| `shiftTime` | Shift time — endTime for login trips, startTime for logout trips |
| `employees` | Number of employees on the trip when it ran (final count) |
| `empAdd` | Number of employees manually added by ops after routing |
| `empDel` | Number of employees manually deleted by ops after routing |
| `empDelRost` | Employees automatically removed due to roster cancellations |
| `empDelProf` | Employees automatically removed due to profile changes |
| `opsIntervened` | YES if ops manually added or deleted anyone |
| `dataIntervened` | YES if any employee was automatically dropped by the system |

---

## Metrics defined

### Intervention rate
**What:** Percentage of auto-routed trips where ops made at least one manual change.  
**Formula:** `trips where opsIntervened=YES / total auto-routed trips`  
**What it tells you:** How often routing output is trusted as-is vs corrected before running.

### Edit patterns
Every intervened trip falls into one of three categories:

| Pattern | Definition | What it means |
|---|---|---|
| Add only | `empAdd > 0` and `empDel = 0` | Routing under-allocated — ops topped up |
| Delete only | `empAdd = 0` and `empDel > 0` | Routing over-allocated — ops trimmed |
| Replace | `empAdd > 0` and `empDel > 0` | Right count, wrong people — ops swapped |

### Churn rate
**What:** On trips where employees were replaced (replace pattern only), what fraction of employees were swapped out.  
**Formula:** `min(empAdd, empDel) / employees` — applied only to replace-pattern trips.  
**Why min():** A replace event is one swap — 1 add + 1 delete = 1 employee replaced. `min` counts the number of actual swap pairs without double-counting.  
**What it tells you:** How deeply the routing output was changed on trips where swapping occurred. High churn = routing picked largely the wrong people. Low churn = routing was mostly right, small adjustments needed.

**Important:** Churn can exceed 100% on 1-person trips if the same seat was swapped multiple times (e.g. Employee A removed, B added, B removed, C added = min(2,2)/1 = 200%). This is valid but should be interpreted carefully — it means the seat changed hands more than once, not that more than one person was on a 1-person trip.

### Data intervention rate
**What:** Percentage of trips where at least one employee was automatically removed by the system (not by ops).  
**Formula:** `trips where dataIntervened=YES / total auto-routed trips`  
**What it tells you:** How much upstream data instability (roster cancellations, profile changes) is making routing output stale before trips run. This is NOT a routing quality problem — it's an HR/attendance data problem.

**Subtypes:**
- `empDelRost` — employee cancelled their own roster booking after routing ran
- `empDelProf` — employee's profile changed (team, transport privilege, stoppage) after routing ran

### Net per trip
**Formula:** `(total empAdd - total empDel) / total trips` for a client or shift group.  
**Positive value:** Routing under-allocated on average — trips ran with more people than routed.  
**Negative value:** Routing over-allocated on average — trips ran with fewer people than routed.  
**Near zero:** Routing got the count right but may have picked wrong people (replace pattern).

---

## How to identify the root cause per client

| Dominant pattern | Net per trip | Likely cause | Suggested fix |
|---|---|---|---|
| Delete only | Negative | Routing over-allocates employees | Review vehicle capacity settings, reduce grouping radius |
| Add only | Positive | Routing under-allocates employees | Check capacity constraints, review unassigned employee handling |
| Replace | Near zero | Right count, wrong grouping | Review stoppage clustering, employee proximity logic |
| High data intervention | Any | Roster instability or stale profile data | Enforce booking deadlines, improve HR data sync before routing runs |

---

## Known limitations

1. **No join between `routing.jobs` and `trips`** — There is no foreign key linking a routing job document to the trips it created. `trips` has no `routingJobId` field and `routing.jobs` has no `tripId` in its output. This means array-level comparison (routing's intended employee list vs final trip employee list) is not currently possible. The edit log approach is the only available method.

2. **Edit log dependency** — The query identifies auto-routed trips by searching for `"From Multi Routing Approve"` in edit log messages. If this message format changes, trips will be missed. `isTripRouted: true` is a cleaner flag but was not consistently populated on all instances tested.

3. **`employees` is final count** — The `employees` array on a trip document reflects who was on the trip when it ran, not who routing originally assigned. This affects churn calculations for trips with multiple sequential edits.

4. **Date field type** — `scheduleDate`, `startTime`, and `endTime` are stored as MongoDB `Long` type. Use `.toNumber()` before passing to `new Date()`. Use `NumberLong(value.toString())` for range queries.

5. **Demo instances** — On non-production instances, trips are in `undeployed` state and never reach `runningStatus: "completed"`. Remove those filters when testing on demo accounts.

---

## Files in this analysis

| File | Description |
|---|---|
| `Trip_EditLog_Query.rtf` | Original query — single day, edit log categorisation |
| `app-v3__april20.txt` | 30-day production output — 7,351 trips across 11 clients |
| `routing_dashboard_v5.html` | Interactive dashboard — 6 tabs with all analysis and plain English explanations |
| `README.md` | This file |

---

## Key findings (April 2026)

- **71.1%** of auto-routed trips were manually corrected by ops before running
- **Replace** is the dominant pattern (36% of all trips) — routing gets the count right but picks wrong employees
- **694b93** is the worst client — 77.7% ops intervention + 17.4% data intervention, 73.8% churn on replace trips
- **697af8** is the best client — 52.6% intervention, 1.6% data intervention
- **2–3 employee trips** are the highest-priority fix — largest volume (4,786 trips) with both high intervention (74%) and high churn (52%)
- **61 trips** had employees drop automatically and ops never added anyone back — those trips ran short
- **Roster cancellations** (331 cases) drive 89% of data drops — a booking discipline problem, not a routing problem
- No client shows improvement week over week — the problem is structural and won't self-correct
