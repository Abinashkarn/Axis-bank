---
title: "Axis CIT — Suggested route, halts and deviation without map APIs"
subtitle: "For Deepak · 14 Nov 2026 · lat/long maths only, MapMyIndia used for tiles, no routing/snap API calls"
---

**Principle:** every map API call costs money per run, so routing, halt detection and deviation are computed on our server from stored lat/long. MapMyIndia is used **only to draw tiles** on the portal. Route suggestion is **optional and never mandatory**: a run can be saved with the stops in whatever order ops entered (or with no order at all — the driver decides on the road). "Suggest a route" is a button ops may click; the result can be applied, edited or ignored. Whatever order is on the run at print time goes on the gate pass; if no order was set, the gate pass lists stops unnumbered.

# 1. Suggested route (optional, on request)

Trigger: ops clicks **Suggest a route** on Plan a run or on the run detail. Nothing runs automatically. Inputs: home chest (origin and return), N planned stops with lat/long, and per stop two optional flags set by ops: **Urgent** (needs cash now) and **Heavy cash** (high value, deliver first to reduce exposure). Optional: a time window per stop (branch opens 10:00).

Algorithm (runs in < 50 ms for N ≤ 15, no external call):

1. **Distance matrix** between chest and all stops with haversine × 1.3 (city road factor; tenant setting `road_factor`, default 1.3, highway 1.15). This gives "estimated road km" without an API.
2. **Priority tiers**: tier 1 = Urgent, tier 2 = Heavy cash, tier 3 = normal. Stops are visited tier by tier; a lower tier never comes before a higher tier unless ops reorder manually.
3. **Within a tier**: nearest-neighbour from the current position, then a **2-opt pass** to remove crossings (for ≤ 15 stops this is effectively optimal).
4. **Time windows**: if a stop's window is not yet open when the van would arrive, it is pushed later within its tier; arrival time estimated at `avg_speed_kmh` (tenant setting, default 22 city).
5. **Return** to the home chest at the end; total estimated km and time shown.
6. **Output**: ordered stop list with estimated arrival time, leg km, and the reason tag for any stop moved out of pure nearest order ("Urgent — moved to #1", "Heavy cash — moved to #2").

Rules shown to the user:

- Case 1 — **Urgent**: 4 stops, stop C marked urgent → C becomes #1 regardless of distance, then nearest-neighbour for the rest from C.
- Case 2 — **Heavy cash**: 6 stops, stop E carries the heaviest load → E is #1 (or right after any Urgent stop); cash exposure drops as early as possible. If two stops are heavy, the one with the larger declared amount goes first; ties by distance.
- Both flags on different stops: Urgent first, then Heavy, then the rest.
- Ops can drag to reorder, apply the suggestion, or ignore it. `route_mode` = none (default, no order enforced) | suggested | manual. Deviation and out-of-order alerts apply **only** when route_mode is suggested or manual; with `none`, only halts and geofence visits are tracked.
- The final order is what prints on the gate pass and what the driver page shows as the stop list.

Data: `run_stops` add `priority` (urgent | heavy | normal), `declared_amount_band` (optional: <25 L, 25–50 L, 50 L–1 Cr, >1 Cr — band only, never the exact figure), `window_from`, `window_to`, `suggested_seq`, `final_seq`, `est_leg_km`, `est_arrival`. `runs` add `route_mode` (none | suggested | manual, default none), `est_total_km`, `est_total_min`, `route_reason` (text).

# 2. Halts (from the GPS track, no API)

- A **halt** = consecutive fixes for ≥ 5 min (tenant `halt_min`, default 5) all within 50 m of each other while the duty is on.
- **Planned halt** = the halt centre is inside a geofence of a stop on this run or the home chest → logged as arrival/departure/dwell for that stop.
- **Unplanned halt** = anywhere else → `halt_unplanned` event with lat/long, start, end, duration; alert (warning) when > 10 min, critical when > 20 min (tenant settings). Address for the record comes from the **nearest known location** in our own Locations table (branch / ATM / chest) + distance ("1.2 km from Karol Bagh branch") — no reverse-geocode call. Reverse geocoding can be switched on later if the bank wants street names.
- Dwell at a planned stop longer than `dwell_max` (default 60 min) → `dwell_long` alert.

# 3. Route deviation (corridor check, no API)

- Only when `route_mode` is suggested or manual. Build the **planned corridor** as the straight-line legs between the final stop order (chest → stop 1 → … → chest), each leg buffered by `corridor_m` (default 500 m city, 1,000 m highway).
- Every GPS fix on duty is tested for distance to the nearest planned leg (point-to-segment distance, pure maths).
- **Deviation** = ≥ 3 consecutive fixes (≈ 30 s at 10-s sampling) outside the corridor → `route_deviation` event with max distance off corridor and duration; alert warning at > 500 m, critical at > 2 km or > 10 min.
- (Only when a route order was applied.) **Stop skipped / out of order** = the van entered a later stop's geofence without visiting an earlier one → `stop_out_of_order` alert; shows on the duty record.
- Because legs are straight lines, a corridor of 500 m is forgiving enough for normal roads; a tighter corridor needs real road geometry (that is the only place an API would help — keep as a paid option later).

# 4. KM without Snap-to-Road

With the device at 10-s fixes and the cleaning rules already given, haversine km is typically within 3–5 % of the odometer. Keep `gps_km_raw` as the billing figure for now; Snap-to-Road stays an **optional add-on** per tenant (`map_match_enabled = false` by default). Tolerance for the flag: max(3 km, 5 %).

# 5. Screens

- **Plan a run**: planned stops are a free list (order optional). A **Suggest a route** button opens the panel — ordered list with est. km / arrival per leg, Urgent / Heavy tags, drag handles, "Use suggested" / "Keep manual" buttons, total km and time; map (tiles only) with numbered pins and straight legs. Per-stop toggles **Urgent** and **Heavy cash** with amount band.
- **Run detail / gate pass**: final order with tags; estimated km printed.
- **Duty record**: unplanned halts listed with duration and nearest location; deviation segments in amber on the map; stops out of order flagged.
- **Alerts**: `halt_unplanned`, `dwell_long`, `route_deviation`, `stop_out_of_order`.
- **Tenant settings**: `road_factor`, `avg_speed_kmh`, `halt_min`, `dwell_max`, `corridor_m`, `map_match_enabled`.

# 6. Tests (section L of the handbook)

| # | Steps | Expected |
|---|---|---|
| L0 | Create a run with 4 stops, do not click Suggest a route → Save | Run saved; route_mode none; gate pass lists stops without numbers; no deviation alerts on the duty |
| L1 | Click Suggest a route on a run with 4 stops, none flagged | Suggested order = nearest-neighbour from chest; total km shown; nothing applied until "Use suggested" |
| L2 | Mark stop C Urgent | C becomes #1 with tag "Urgent — moved to #1"; rest re-optimised from C |
| L3 | 6 stops, mark E Heavy cash (> 1 Cr) | E becomes #1; others follow by distance |
| L4 | B Urgent and E Heavy | Order B, E, then nearest-neighbour |
| L5 | Drag stop 3 to position 1 | Route mode "Manual"; km recomputed; audit row with user |
| L6 | Gate pass | Prints the final order with Urgent / Heavy tags and est. km |
| L7 | Device stationary 6 min at a non-stop location (test with device in office) | Unplanned halt logged with "x km from <nearest location>"; alert after 10 min |
| L8 | Drive 1 km away from the straight leg | `route_deviation` alert with max off-corridor distance |
| L9 | Enter stop 3 geofence before stop 2 | `stop_out_of_order` alert |
| L10 | Confirm no MapMyIndia routing / snap / geocode requests were made during L1–L9 | Only tile requests in the server/API logs |
