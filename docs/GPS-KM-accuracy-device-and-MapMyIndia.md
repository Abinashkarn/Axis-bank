---
title: "Track0tic — GPS KM accuracy for Axis CIT: device config + MapMyIndia integration"
subtitle: "For Deepak · 12 Nov 2026"
---

**Why:** GPS km today reads 15–25 % below the odometer in city driving — one fix every 6 minutes, straight lines between fixes. Two changes fix this, no van calibration needed: **Part A** on the device (fix faster while moving), **Part B** on the server (snap the track to MapMyIndia roads).

# Part A — Device config change


**Problem:** one GPS point every 6 minutes → straight lines between points → GPS km reads 15–25 % below the odometer in city driving. The fix is on the device; no van calibration needed.

## Settings to change

| Setting | Now | Change to |
|---|---|---|
| GPS fix interval while **moving** | 6 min | **10 s, or every 50 m** (whichever first) |
| GPS fix interval while **stopped** (speed < 3 km/h for 60 s) | 6 min | 2 min |
| Upload / transmit interval | 6 min | keep 2–6 min, but send **all fixes collected since the last upload as one batch** |
| Offline buffer | 30 fixes | **500 fixes** (≈ 80 min of moving at 10 s); FIFO; send oldest first when signal returns |
| Fields per fix | lat, lon, time | lat, lon, **GPS time**, speed, heading, HDOP, satellites, motion/ignition flag (+ odometer if OBD-connected) |
| Time source | device clock | **GPS time**; drop the fix if there is no valid GPS time |
| Power | — | 10-s fixes draw more; fine on vehicle power with battery backup. Battery-only units: 30 s while moving |

## Why

- **10-s fixes** follow the road instead of cutting corners — this alone removes most of the gap.
- **Speed and HDOP** let the server drop parked jitter and bad fixes.
- **GPS time** keeps buffered points in the right order when they arrive late.
- **Bigger buffer** because 30 × 10 s is only 5 minutes; a tunnel or a no-signal stretch is longer.

## Data cost

36 fixes per 6-minute batch ≈ 2 KB — about the same as today.

## Verify after the change

1. One van, one day → about 360 fixes per moving hour.
2. Parked overnight → no distance recorded.
3. Pull the SIM for 15 minutes while driving → fixes arrive later marked *buffered*, in correct time order, none lost.
4. Dashboard / Duties GPS km for a known route within a few % of the odometer.


# Part B — MapMyIndia (Mappls) integration


**Goal:** GPS km must be the distance driven **on the road**, not straight lines between fixes. MapMyIndia already serves our map tiles; use its Snap-to-Road and Route APIs server-side to turn the cleaned GPS track into road kilometres. No van calibration involved.

## 1. Where it sits

Device fixes → server cleaning → **Mappls Snap-to-Road** → road km per duty → Duties page / reports. Runs when a duty ends and again nightly (buffered points can arrive late).

## 2. API calls (server-side only, tenant key from config, never in the browser)

| Use | Mappls API | Call | Notes |
|---|---|---|---|
| Map-match the track | **Snap to Road** | `GET https://apis.mappls.com/advancedmaps/v1/{key}/snapToRoad?pts=lat,lon;lat,lon;…&timestamps=t1;t2;…&radiuses=20;20;…` | Max 100 points per call. Pass timestamps so order is kept. Sum the length of the returned matched geometry → road km for that chunk |
| Distance across an offline gap (> 5 min between fixes) | **Route, driving** | `GET https://apis.mappls.com/advancedmaps/v1/{key}/route_adv/driving/{lon,lat};{lon,lat}?overview=false` | `routes[0].distance` in metres. One call per gap |
| Address of an unplanned halt | **Reverse geocode** | `GET https://apis.mappls.com/advancedmaps/v1/{key}/rev_geocode?lat=&lng=` | Cache by lat/lon rounded to 4 decimals |
| Map on portal pages | **Mappls Map SDK** (already on dashboard) | same key | Load on Live Tracking and the duty record page too; draw matched track (blue), raw points (grey), gaps (dashed) |

## 3. Steps per duty

1. Take all fixes between duty start and end, sorted by GPS time.
2. Drop bad fixes: HDOP > 5, satellites < 4, duplicates, speed < 3 km/h and < 15 m from previous kept point (parked jitter), implied speed > 120 km/h (jump), time drift > 60 s.
3. Points inside a chest/branch geofence while stopped → snap to the geofence centre (no km inside a chest).
4. Haversine sum of the kept points → `gps_km_raw` (keep as fallback).
5. Send kept points to Snap-to-Road in chunks of ≤ 100 → sum road geometry → `gps_km_matched`. Record `match_quality` = share of chunks that matched.
6. For each gap > 5 min between kept points → Route API distance → add to `gps_km_matched`, store `gap_count`, `gap_km`, `gap_minutes`.
7. `gps_km` = matched if match_quality ≥ 0.8, else raw. Mark `match_partial = true` if any chunk fell back.
8. Compare with odometer: `diff_km = odometer_km − gps_km`; flag when |diff| > max(3 km, 5 %). Reason: offline gap / device off (gps < 1 km, odometer > 5) / odometer under-read / unexplained.
9. Write a row to `duty_km_audit` every time this runs (audit trail).

## 4. Quota and cost

A 100-km city duty at 10-s fixes ≈ 2,000 raw points → ≈ 800 after cleaning → 8 Snap-to-Road calls + a few Route calls ≈ **10 calls per duty**. Cache results on the duty; recompute only if new buffered points arrive. Check the Mappls plan limits for Snap-to-Road and Route before go-live and set a daily alert at 80 % of quota.

## 5. Errors and fallback

- HTTP error, timeout (> 5 s) or empty geometry for a chunk → use haversine for that chunk, lower `match_quality`, retry nightly.
- Key invalid / quota exhausted → whole duty on haversine, `match_partial = true`, alert `mappls_unavailable` to ops.
- Never block End duty on the API — compute asynchronously after the duty is saved.

## 6. DB fields

**duties**: `gps_km_raw`, `gps_km_matched`, `match_quality`, `match_partial`, `gps_km`, `odometer_km`, `diff_km`, `diff_pct`, `km_flag`, `km_flag_reason`, `gap_count`, `gap_km`, `gap_minutes`, `points_total`, `points_kept`, `km_computed_at`.
**telemetry_points**: add `speed_kmh`, `heading`, `hdop`, `sats`, `motion`, `source` (live | buffered), `rejected_reason`.
**duty_km_audit** (append-only): duty_id, computed_at, raw, matched, gps_km, diff_km, flag, reason.

## 7. UI

- **Duties**: columns GPS km · Odometer km · Diff km / % · Gaps; flag icon with reason; filter "Flagged only"; hover shows raw vs matched.
- **Duty record** (wireframe P4): KPI tiles raw / matched / odometer / diff / gaps; Mappls map with matched track, raw points, gaps dashed, stops numbered, halts amber; "Recompute KM" and "Mark as explained" (reason saved).
- **Live Tracking**: fix the map key so it loads like the dashboard.
- **Alerts**: `km_mismatch` (warning) when flagged; `device_off_on_duty` (critical).

## 8. Verify

1. Five real duties: matched km should be 10–20 % above raw in city, 2–5 % on highway.
2. Track on the duty record follows roads, no corner-cutting.
3. Put the device in a metal box for 15 min while driving → one offline gap with road distance, not a straight line.
4. Switch the device off for the second half of a duty → flagged `device_off`.
5. Turn the Mappls key off on UAT → duty still completes, `match_partial = true`, ops alert raised.
