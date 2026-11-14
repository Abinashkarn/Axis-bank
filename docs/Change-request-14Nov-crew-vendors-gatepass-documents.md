---
title: "Axis CIT — Change request after customer demo (14 Nov 2026)"
subtitle: "For Deepak · Track0tic™ Axis Bank tenant · crew members, KYC for all, printable gate pass, document expiry alerts"
---

Four changes from the bank's feedback in today's demo. All are phase 1 and should go to UAT before the next demo.

| # | Feedback | Change in one line |
|---|---|---|
| 1 | A cash van carries a crew of 2 or more — driver, custodian 1–2, gunman 1–2 | A run / duty has a **crew list**, not a single driver |
| 2 | "Driver" should be "Crew member"; every crew member needs full KYC (bank audit) | Rename the module, add a **role** field, same KYC and two-person approval for every role |
| 3 | Every duty / work order must be printable as a **gate pass**, with boxes to write opening and closing odometer by hand | "Print gate pass" PDF on every run, plus a portal field to key in the hand-written readings |
| 4 | Vendor agreements, van papers (RC, PUC, fitness, insurance, year of manufacture) and crew licences must be tracked with **expiry alerts** | One documents module with valid-till dates, 60/30/7-day alerts, blocking on expiry, Compliance page |

# 1. Crew members (rename of Drivers)

**Sidebar**: *Drivers* → **Crew members**. Page title "Crew members", KPIs: Total · KYC verified · KYC pending · Licence expiring < 60 d · By role (Driver / Custodian / Gunman).

**Add crew member** = the existing Add driver form plus:

| Field | Rule |
|---|---|
| Role | required: Driver · Custodian · Gunman (tenant can add more, e.g. Supervisor) |
| Full name as on Aadhaar, mobile, DOB, Aadhaar (checksum, stored masked), photo, Aadhaar front/back upload | required for **every** role |
| Driving licence number / class / valid till + upload | required for **Driver**; hidden for others |
| Arms licence number / valid till + upload | **required for Gunman**; hidden for others |
| Police verification certificate (upload + date) | optional for all; bank may ask — keep the field |
| Vendor | required; one vendor per crew member |

Rules unchanged: Submit disabled until required fields and uploads are present; expired licence blocked; Aadhaar checksum; two-person approval (adding admin cannot approve); Reject with reason; Aadhaar masked in lists. Only a crew member with role **Driver**, KYC verified and licence valid can start a duty.

**API**: keep `/ops/drivers` working as an alias for one release, add `/ops/crew-members`. Driver page OTP login stays for role Driver only (any role may be allowed later for custodian confirmation — see 2.3).

# 2A. Who supplies the crew — two cases

| Case | Who | Van + driver | Gunmen | Custodians |
|---|---|---|---|---|
| 1 — Axis Bank (default) | One CIT vendor supplies everything | CIT vendor | same CIT vendor | same CIT vendor |
| 2 — other / PSU banks | Split across suppliers | CIT vendor | a **security agency** (separate vendor) | a **custodian agency** or the bank's own staff |

Both must work in the same product, so:

- **Vendor registration** gets a field **Services provided** (multi-select): *Van & driver* · *Gunmen* · *Custodians*. A CIT vendor ticks all three in case 1; a security agency ticks only Gunmen. Vendor type label follows from it (CIT vendor / Security agency / Custodian agency). "Bank staff" is a built-in pseudo-vendor for custodians employed by the bank.
- **Crew member** keeps exactly one vendor; the role must be one that vendor provides (a security agency cannot add a driver).
- **Run** has one **CIT vendor** (owns the van and the driver). Each crew row stores its own `vendor_id`.
- **Tenant setting `crew_sourcing`**: `single_vendor` (Axis — custodian and gunman dropdowns show only the run's CIT vendor crew) or `multi_vendor` (dropdowns show verified crew of that role from **any Active vendor that provides that service**, grouped by vendor).
- **Visibility**: a vendor user sees runs where any of its crew is deployed, but only its own people's rows; approvals stay with the bank.
- **Gate pass, duty record, Duties list, crew attendance export**: show the vendor beside each crew member.
- **Billing / reconciliation reports**: per vendor — the CIT vendor is billed on van km; a security agency on gunman-days; a custodian agency on custodian-days. Add a "Crew-days by vendor" export.

Data: `vendors.services` (set: van_driver | gunmen | custodians), `run_crew.vendor_id`, `duty_crew.vendor_id`, `tenant_settings.crew_sourcing`.

# 2. Crew on a run and on a duty

## 2.1 Data

- `crew_members` (rename of `drivers`): add `role` enum (driver | custodian | gunman | other), `arms_licence_no`, `arms_licence_valid_till`, `arms_licence_doc`, `police_verification_doc`, `police_verification_date`.
- `run_crew` (new): run_id, crew_member_id, role, is_lead (bool) — the planned crew.
- `duty_crew` (new): duty_id, crew_member_id, role, confirmed_at, confirmed_by (driver_select | otp) — who was actually on board.
- `tenant_settings`: `min_crew` JSON, default `{"driver":1,"custodian":1,"gunman":1}`; `max_crew` 6.

## 2.2 Create run (portal)

After Vendor → Van, the Driver dropdown becomes a **Crew** section:

- Driver (required, 1) — vendor's verified drivers with valid licence.
- Custodian 1, Custodian 2 (optional) — verified custodians.
- Gunman 1, Gunman 2 (optional) — verified gunmen.
- Validation against `min_crew`; a crew member cannot be on two runs at the same time on the same date.

Run detail shows the crew table: role · name · mobile · KYC state · photo thumbnail. "Reassign" lets ops swap any crew member before the run starts.

## 2.3 Driver page

- **Start duty** screen gets a **Crew on board** step after "Is this your van?": the planned crew is listed with photos and a tick box each; the driver unticks anyone absent and can add a replacement from the vendor's verified crew of that role. If the crew falls below `min_crew`, Start duty is blocked with the message "Minimum crew: 1 driver, 1 custodian, 1 gunman".
- Optional (tenant setting `crew_otp_confirm`): each custodian confirms presence with an OTP to their own mobile on the same phone — stronger evidence for audit. Off by default for the pilot.
- **On duty** and **Duty complete** screens show the crew names.
- Hindi strings for all of the above.

## 2.4 Portal after the duty

Duties list: new column **Crew** (e.g. "D: Suresh · C1: Ramesh · G1: Vikram"). Duty record: crew table with confirmed time. Reports: Daily KM & route audit gets crew columns; a new **Crew attendance** export (date, van, run, role, name, Aadhaar masked, confirmed at) for the bank's audit.

# 3. Printable gate pass

## 3.1 Where

- Run detail → **Print gate pass** (available from status Assigned onward; re-printable; each print is logged with user and time).
- Duties list / duty record → **Print gate pass** (shows the recorded readings, if any).
- Bulk: CIT Runs list → select date → "Print all gate passes for today" (one PDF, one page per run).

## 3.2 Layout (A4 portrait, one page)

```
+------------------------------------------------------------------+
| AXIS BANK — CASH VAN GATE PASS            Track0tic™   [QR of WO] |
| WO no: WO-2026…-27996F     Date: 14 Nov 2026   Print #2 14:05     |
+------------------------------------------------------------------+
| Vendor: CMS Info Systems          Contract: AXIS-CMS-2024-B2      |
| Van: DL02-AX-2002   Make: Tata 407   Tracker: TRK-MPN26EJ08…      |
| Home chest: Axis Chest Delhi Rohini                               |
+------------------------------------------------------------------+
| CREW ON BOARD                                                     |
| Role        Name          Vendor        Mobile      ID (masked) |
| Driver      Suresh Patil  CMS           98123 45002 XXXX-XXXX-7029 |
| Custodian 1 …                                                       |
| Gunman 1   …                                                       |
+------------------------------------------------------------------+
| ROUTE                                                             |
| 1. Axis Chest Delhi Rohini (origin)   planned dep ____:____       |
| 2. Axis ATM Nehru Place                                           |
| 3. Axis Branch Noida 62                                           |
| Return: Axis Chest Delhi Rohini                                   |
+------------------------------------------------------------------+
| ODOMETER (write by hand)                                          |
|  OPENING  [ _ _ _ _ _ _ ] km   time ____:____   gate sign ______  |
|  CLOSING  [ _ _ _ _ _ _ ] km   time ____:____   gate sign ______  |
|  App readings (printed if available): open ______  close ______   |
+------------------------------------------------------------------+
| Signatures:  Driver ______  Custodian ______  Chest in-charge ____ |
|              Security gate OUT ______   Security gate IN ______   |
| Generated by Track0tic™ · Ultimatemile India Pvt Ltd · page 1/1   |
+------------------------------------------------------------------+
```

QR encodes the WO number / run URL so the gate can scan it with the portal to open the run.

## 3.3 Manual odometer entry in the portal

- Run detail and duty record get a card **Gate pass readings**: Opening odometer (gate), Opening time, Closing odometer (gate), Closing time, Entered by, Entered at, photo upload of the signed gate pass.
- Role: Operator / Dispatcher or Admin; every edit written to the audit log.
- Duties list shows three figures when present: **GPS km · App odometer km · Gate-pass odometer km**, with the difference flags against GPS for both. A mismatch between app and gate-pass readings > 2 km is its own flag `odometer_source_mismatch`.
- Reports: Daily KM & route audit adds gate-pass opening / closing / km columns.

## 3.4 Data

`duties`: add `gate_open_km`, `gate_open_time`, `gate_close_km`, `gate_close_time`, `gate_entered_by`, `gate_entered_at`, `gate_pass_scan_url`. `gate_pass_prints` (new): run_id, printed_by, printed_at, print_no.

# 4. Documents, validity and expiry alerts (vendor · van · crew)

Every entity that goes on a run carries documents with an expiry date. The system stores the document, its number and its valid-till date, warns before expiry and **blocks** the entity from being put on a run once expired. One rule set, one alert engine, one "Expiring documents" page.

## 4.1 What is tracked

| Entity | Document | Number | Valid till | Upload | Required | On expiry |
|---|---|---|---|---|---|---|
| CIT vendor / agency | Agreement / MOU with the bank | contract no. | agreement date → expiry date | PDF | yes | **block new runs** for that vendor; existing runs finish |
| | GST certificate, PAN | GSTIN, PAN | — | PDF | yes | — |
| | Insurance (cash-in-transit / fidelity) | policy no. | expiry | PDF | yes | block new runs |
| | PSARA licence (security agency supplying gunmen) | licence no. | expiry | PDF | yes if services include Gunmen | block its gunmen from runs |
| | Bank empanelment letter | ref | expiry (optional) | PDF | optional | warn |
| Van | Registration certificate (RC) | registration no. | RC validity | PDF/JPG | yes | block van |
| | Year of manufacture | from RC | rule: van age ≤ `max_van_age_years` (default 10; bank may set 8) | — | yes | block van when older |
| | Insurance | policy no. | expiry | PDF | yes | block van |
| | Pollution (PUC) | certificate no. | expiry | PDF/JPG | yes | block van |
| | Fitness certificate | certificate no. | expiry | PDF | yes | block van |
| | Permit (commercial / national) | permit no. | expiry | PDF | yes | block van |
| | Road tax | receipt | paid till | PDF | optional | warn |
| | Van modification / armouring certificate (if applicable) | ref | expiry | PDF | optional | warn |
| Crew — all roles | Aadhaar (masked), photo, police verification certificate | — | PV date + 3-year revalidation | PDF/JPG | Aadhaar + photo yes; PV as per bank | warn / block per tenant setting |
| Crew — Driver | Driving licence (class LMV-TR / HMV) | DL no. | expiry | PDF/JPG | yes | **block from starting duty** |
| Crew — Gunman | Arms licence | licence no. | expiry | PDF/JPG | yes | block from runs |
| | PSARA guard training certificate / security guard card | card no. | expiry | PDF/JPG | yes | block from runs |
| | Weapon details (type, licence linkage) | — | — | — | optional | — |
| Crew — Custodian | ID card from employer / bank, police verification | — | PV date | PDF/JPG | yes | warn / block per setting |

## 4.2 Alert rules (tenant settings, defaults shown)

| Days before expiry | What happens |
|---|---|
| 60 | Yellow badge on the row; weekly digest email to vendor user + Axis ops |
| 30 | Orange badge; alert `document_expiring` (warning) in Alerts; email to vendor + Axis ops |
| 7 | Red badge; alert escalated to critical; daily email; shown on dashboard tile "Expiring in 7 days" |
| 0 (expired) | Entity blocked as per table above; alert `document_expired` (critical); the run planner greys the item out with the reason ("Fitness certificate expired 12 Nov") |

- Thresholds are per tenant (`expiry_warn_days` = [60, 30, 7]).
- Renewal: upload the new document + new valid-till → old one archived (kept 7 years), block lifted automatically, alert resolved.
- Every upload / renewal / override is in the audit log with user, time and reason.
- Override: only an Axis Admin can allow a blocked van or crew member on a run, with reason, for a limited number of days; shown on the gate pass as "Exception approved by …".

## 4.3 Screens

- **CIT Vendors → vendor detail**: *Documents* tab — one row per document with number, valid from / till, status chip (Valid · Expiring 42 d · Expired), Upload / Renew. Agreement date and expiry date are fields on Register vendor.
- **Vans → Add / Edit van**: Van KYC block — RC no. + validity, year of manufacture (auto-check against max age), insurance, PUC, fitness, permit, road tax, each with valid-till and upload. Van list gets a *Documents* column (✓ all valid / ⚠ expiring 2 / ✕ expired) and filter "Expiring / Expired".
- **Crew members → Add / Edit**: same pattern; Gunman form shows arms licence + guard card; Driver shows DL; all show police verification.
- **New page: Compliance / Expiring documents** (sidebar, under Reports): one table across vendors, vans and crew — Entity · Document · Number · Valid till · Days left · Status · Owner vendor · Action (Renew). Filters by entity type, vendor, days left. Export CSV. Dashboard tile "Documents expiring ≤ 30 d: 7".
- **Create run**: vans and crew with an expired or blocking document are greyed with the reason; a vendor with an expired agreement does not appear.
- **Gate pass**: footer line "Van docs valid till: RC 2029-03 · Fitness 2027-01 · PUC 2026-12 · Insurance 2027-05" and "Crew licences valid" so the gate can see it at a glance.
- **Driver page**: driver with an expired DL cannot log in to start duty ("Licence expired — contact your vendor").

## 4.4 Data

`documents` (new, generic): id, owner_type (vendor | vehicle | crew_member), owner_id, doc_type (enum per table above), number, issued_on, valid_from, valid_till, file_url, status (valid | expiring | expired | archived), uploaded_by, uploaded_at, verified_by, verified_at, replaced_by (id). `vehicles`: add `year_of_manufacture`. `tenant_settings`: `expiry_warn_days`, `max_van_age_years`, `pv_required_roles`, `block_on_expired` (per doc type). Nightly job recomputes status and raises alerts; `document_audit` append-only.

## 4.5 Tests to add (section K, continued)

| # | Steps | Expected |
|---|---|---|
| K11 | Register vendor with agreement expiry 20 days away | Vendor row shows "Agreement expiring 20 d"; alert in Alerts; email to ops |
| K12 | Set agreement expiry to yesterday | Vendor not offered in Create run; existing run still completes; `document_expired` critical alert |
| K13 | Add van with year of manufacture 2014 (max age 10) | Refused / blocked with reason "Van older than 10 years" |
| K14 | Add van with PUC expiring in 5 days | Van row red badge; still selectable; dashboard tile count +1 |
| K15 | Set fitness certificate expired | Van greyed in Create run with reason; cannot start duty from driver page |
| K16 | Gunman with arms licence expired | Not selectable as Gunman; alert raised |
| K17 | Renew (upload new PUC with new date) | Status back to Valid; alert auto-resolved; old doc visible as archived |
| K18 | Compliance page filter "≤ 30 days" + Export CSV | Correct rows across vendor / van / crew; CSV matches |
| K19 | Admin override on blocked van for 2 days with reason | Van selectable; gate pass shows "Exception approved by …"; audit row |

# 5. Build order

1. Rename Drivers → Crew members, add role and the role-specific fields; migrate existing drivers as role = driver. Vendor "Services provided" + tenant `crew_sourcing` setting. (1 day UI + migration)
2. Run crew: `run_crew`, Create run crew section, run detail crew table, `min_crew` setting. (1 day)
3. Driver page: Crew on board step, block below minimum, crew shown on duty / complete, Hindi. (1 day)
4. Gate pass PDF + Print buttons + print log + QR. (1 day)
5. Gate pass readings card, Duties columns, flags, reports. (1 day)
6. Crew attendance export. (½ day)
7. Documents table + uploads on vendor / van / crew forms, nightly expiry job, alerts, Compliance page, run-planner blocking, gate-pass footer. (3 days)

# 6. Tests to add to the handbook (section K)

| # | Steps | Expected |
|---|---|---|
| K1 | Sidebar | "Crew members", no "Drivers" anywhere (portal, driver page, reports) |
| K2 | Add crew member role Gunman without DL fields | Accepted; DL fields hidden; arms licence + Aadhaar + uploads required |
| K3 | Add crew member role Driver without DL | Submit disabled |
| K4 | Create run with driver only | Blocked: "Minimum crew: 1 driver, 1 custodian, 1 gunman" |
| K5 | Create run with driver + custodian + gunman | Run detail shows crew table with vendor per member |
| K5a | Tenant setting single_vendor (Axis): open Gunman dropdown | Only the run's CIT vendor gunmen listed |
| K5b | Switch tenant setting to multi_vendor; register a Security agency (services = Gunmen) with one verified gunman | Gunman dropdown now lists him under his agency; gate pass shows the agency beside his name |
| K5c | Security agency tries to add a crew member with role Driver | Refused: role not in the vendor's services |
| K6 | Same custodian on two runs same date/time | Refused |
| K7 | Driver page → Crew on board → untick the gunman → Start | Blocked below minimum; tick again → starts; duty record shows crew with confirmed time |
| K8 | Run detail → Print gate pass | A4 PDF with WO, QR, vendor, van, crew with masked IDs, route, empty odometer boxes, signature lines; print logged |
| K9 | Enter gate-pass readings in portal | Saved with user/time; Duties shows GPS · App · Gate-pass km; mismatch > 2 km flagged |
| K10 | Reports → Crew attendance export | One row per crew member per duty |

Related: wireframes https://abinashkarn.github.io/Axis-bank/ · GPS KM accuracy spec (device + MapMyIndia) · Tester handbook https://claude.ai/artifact/6uuBzutLpTug5qXYDHeUYU
