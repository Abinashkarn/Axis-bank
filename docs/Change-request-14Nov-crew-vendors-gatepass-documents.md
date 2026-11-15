---
title: "Axis CIT — Change request after customer demo (14 Nov 2026)"
subtitle: "For Deepak · Track0tic™ Axis Bank tenant · crew members, KYC for all, printable gate pass, document expiry alerts"
---

Six changes from the bank's feedback in today's demo. All are phase 1 and should go to UAT before the next demo.

| # | Feedback | Change in one line |
|---|---|---|
| 1 | A cash van carries a crew of 2 or more — driver, custodian 1–2, gunman 1–2 | A run / duty has a **crew list**, not a single driver |
| 2 | "Driver" should be "Crew member"; every crew member needs full KYC (bank audit) | Rename the module, add a **role** field, same KYC and two-person approval for every role |
| 3 | Every duty / work order must be printable as a **gate pass**, with boxes to write opening and closing odometer by hand | "Print gate pass" PDF on every run, plus a portal field to key in the hand-written readings |
| 4 | Vendor agreements, van papers (RC, PUC, fitness, insurance, year of manufacture) and crew licences must be tracked with **expiry alerts** | One documents module with valid-till dates, 60/30/7-day alerts, blocking on expiry, Compliance page |
| 5 | Approvers / admins must see and print one **collated PDF** per vendor, van and crew member — form + all documents + validity + approval trail — for internal bank audit | Approval dossier: approve from the dossier, Dossier tab everywhere, audit pack ZIP |
| 6 | Too much data entry for the bank → the bank sends the vendor a **link** and the vendor enters its own masters; **three-person rule**: vendor maker → bank operator checker → bank admin approver | Vendor invite + self-onboarding wizard + approval queue with three stages |

# 0. Sidebar order and Locations master

Sidebar (team build, with the new items slotted in): **Dashboard · Approvals · Locations · CIT Vendors · Vans · Crew members (Drivers) · ~~Inventory~~ (hide for CIT tenant) · Work Orders · Duties · Live Tracking · Van Health · Alerts · Compliance · Reports · Users.**

**Locations** is the bank's own master — currency chests, hubs, branches, ATMs — and is where work orders and duties are anchored: a work order's origin and return is a location of type *chest* or *hub*; its planned stops are locations of type *branch* or *ATM*; each location is a geofence (radius per type) that logs arrival, departure and dwell. Fields: name, bank code (SOL / branch code), type, address, lat/long (pin on map), geofence radius, operating hours, contact. Added by Axis ops (or bulk Excel), approved by a second Axis admin (two bank users; no vendor involved). Only approved locations appear in Create work order. Rename "CIT Runs" → **Work Orders** everywhere to match the build.

# 1. Crew members (rename of Drivers)

**Sidebar**: *Drivers* → **Crew members**. Page title "Crew members", KPIs: Total · KYC verified · KYC pending · Licence expiring < 60 d · By role (Driver / Custodian / Gunman).

**Add crew member** = the existing Add driver form plus:

| Field | Rule |
|---|---|
| Role | required: Driver · Custodian · Gunman (tenant can add more, e.g. Supervisor) |
| Full name as on Aadhaar, mobile, DOB, Aadhaar (checksum, stored masked), **passport-type photo**, Aadhaar front/back upload | required for **every** role (driver, custodian, gunman) |
| Passport-type photo rules | 35 × 45 mm ratio (7:9), plain light background, face straight, no cap / dark glasses, face 70–80 % of frame, taken within 6 months; min 600 × 771 px, JPG/PNG ≤ 2 MB. Capture from camera on the portal/phone or upload; the system crops to 7:9, checks a single face is present and size/brightness, and rejects otherwise ("Retake — face too small / background not plain"). Re-photo required every 3 years or on role change. Printed on gate pass, dossier cover and shown on the driver's crew-on-board screen |
| When the photo is taken | (a) **At vendor registration / crew onboarding** — the vendor user (or Axis ops on its behalf) adds each crew member with the passport photo and KYC documents; nothing is active until a second Axis admin approves from the dossier. (b) **At every KYC update** — renewal of DL / arms licence / guard card / police verification, role change, vendor change, or any edit of name / Aadhaar: the form asks for a fresh photo if the current one is older than 3 years or the face check fails; the update goes back to Pending and needs approval again. (c) Vendor registration itself: passport photo + ID of the **authorised signatory** and the **ops contact** (the people who will log in as vendor users), stored in the vendor dossier |
| Driving licence number / class / valid till + upload | required for **Driver**; hidden for others |
| Arms licence number / valid till + upload | **required for Gunman**; hidden for others |
| Police verification certificate (upload + date) | optional for all; bank may ask — keep the field |
| Vendor | required; one vendor per crew member |

Rules unchanged: Submit disabled until required fields and uploads are present; expired licence blocked; Aadhaar checksum; three-person approval (vendor maker → bank checker → bank approver, see section 6); Reject / send back with reason; Aadhaar masked in lists. Only a crew member with role **Driver**, KYC verified and licence valid can start a duty.

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
| Role        Name          Vendor        Mobile      ID (masked)  Photo 35×45 |
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

# 4A. Approval dossier — one collated PDF per master, for approvers and bank audit

Whoever approves (Approver, Admin, Super admin) must see **everything about the record on one screen and one printable PDF** before clicking Approve, and auditors must be able to pull the same pack later. One generator, three entity types.

## 4A.1 What the dossier contains

| Section | Vendor / agency | Van | Crew member (driver / custodian / gunman) |
|---|---|---|---|
| 1. Cover | Legal name, vendor type / services, GSTIN, PAN, contract no., regions, status, dossier no. + QR, generated by / at | Registration, vendor, make / model / year, home chest, device, status | Name as on Aadhaar, **passport-type photo (35 × 45 mm)**, role, vendor, crew ID, mobile, DOB, status |
| 2. Form as submitted | Every field of Register vendor with the value entered, who entered it and when; passport photos + ID of authorised signatory and ops contact | Every field of Add van | Every field of Add crew member (Aadhaar masked, full number visible only to Admin with reason) |
| 3. Validity table | Agreement / MOU from–to, insurance, PSARA, empanelment — each with number, valid till, days left, status chip | RC, insurance, PUC, fitness, permit, road tax; year of manufacture vs max age | DL (driver) / arms licence + guard card (gunman) / police verification (all), each with number, valid till, days left, status |
| 4. Documents | Each uploaded file rendered as a full page in the PDF (images scaled, PDFs merged), with a header "Document 3 of 7 · Agreement · uploaded by … on …" | same | same |
| 5. Linked records | Vans and crew under this vendor with their own status | Crew currently assigned; last 10 duties | Runs in the last 90 days; duties; KM; alerts involving this person |
| 6. Approval trail | Registered by (user, time, IP) → changes → approved / rejected by (user, time, reason) → renewals → overrides | same | same |
| 7. Sign-off page | "Verified by ______ (Approver) · Date ______ · Remarks" + checkbox list of what was checked | same | same |

Every page carries: tenant, entity id, dossier no., page x / y, "Confidential — Axis Bank internal audit", Track0tic™ footer. Dossier no. = `DOS-<entity>-<id>-<version>`; a new version is generated on every approval, renewal or edit, older versions kept 7 years and listed.

## 4A.2 Where it appears

- **Pending rows** (vendors, vans, crew) and the **Approvals queue**: button **Review dossier** opens it full-screen. The bottom bar depends on the viewer's stage: checker sees *Send back* / *Mark checked*; approver sees *Reject* / *Approve* (only when already Checked). Each stage must tick its checklist / confirm it has seen all N documents.
- **Entity detail pages**: *Dossier* tab — View · Print · Download PDF · version list.
- **Compliance page**: Download dossier per row.
- **Audit pack** (Admin / Super admin, Reports): pick vendor(s) and date range → one ZIP with every dossier (vendor + its vans + its crew) plus an index CSV (entity, dossier no., status, expiries). Generated in the background; link emailed; download logged.
- **Vendor**: no login; a *Download my dossier* task link can be sent on request (read-only, 7 days).

## 4A.3 Generation and storage

- Server-side PDF (e.g. wkhtmltopdf / Puppeteer on an HTML template, or a PDF library) merging uploaded PDFs and images; A4; images downscaled to ≤ 150 dpi; expect 2–10 MB per dossier.
- Generated on demand, then cached in S3 under the same bucket / retention as KYC documents (SSE-KMS, Object Lock, 7 years). `dossiers` table: id, entity_type, entity_id, version, generated_by, generated_at, trigger (approval | renewal | manual | audit_pack), file_url, sha256, page_count.
- Every view, print and download is written to the audit log (who, when, dossier no., IP) — this is what the bank's internal audit asks for first.
- Watermark "Printed by <user> on <date>" on every printed page.

## 4A.4 Tests (section K, continued)

| # | Steps | Expected |
|---|---|---|
| K20 | Vendor pending → Review dossier | Full dossier with 7 sections; Approve disabled until documents section viewed / confirmed |
| K21 | Approve from dossier with reason | Status Active; dossier version 2 generated with the approval in the trail; audit row |
| K22 | Crew member dossier as Approver | Aadhaar masked everywhere; as Admin with reason → unmasked, logged |
| K23 | Print dossier | A4 PDF, every uploaded document as a page, watermark "Printed by …", page x / y |
| K24 | Renew van PUC → open dossier | New version listed; validity table shows the new date; old version still downloadable |
| K25 | Reports → Audit pack for Vendor A, last 12 months | ZIP with vendor + all vans + all crew dossiers + index CSV; email link; download logged |
| K26 | Vendor user login → Dossier of another vendor's van | Not visible / 403 |

# 6. Vendor self-onboarding and the three-person rule

The bank should only **invite** and **approve**; the vendor types and uploads its own masters. Every master (vendor, van, crew member) and every later change passes through three people:

| Stage | Who | What they do | Status after |
|---|---|---|---|
| 1 · Maker | **Vendor user** (or Axis ops on the vendor's behalf, flagged "entered by bank") | Fills the form, uploads documents and passport photos, submits | *Submitted · awaiting checker* |
| 2 · Checker | **Bank operator** (Operator / Dispatcher role) | Opens the dossier, verifies each document against the form (checklist), can send back to the vendor with remarks, ticks "Checked" | *Checked · awaiting approver* |
| 3 · Approver | **Bank admin** (Admin / Super admin, or a user with Approver role) | Opens the same dossier with the checker's ticks and remarks visible, approves or rejects with reason | *Active / Verified* or *Rejected* |

Rules: the three must be three different users; the checker cannot be the maker, the approver cannot be the checker or the maker (enforced by user id, also when Axis ops entered on behalf of the vendor). Any later edit, renewal or role change restarts at stage 1. All three names, times and IPs are printed in the dossier's approval trail. Tenant setting `approval_stages` = 3 (default for Axis) or 2 (maker + approver) for smaller banks.

## 6.1 Task links instead of a vendor login (decision 15 Nov)

No vendor portal and no vendor accounts. Every piece of work the bank needs from a vendor is a **task link**: single-purpose, single-use, sent by SMS + email, opens only after an OTP to the vendor's registered mobile, valid 7 days, dead after submit. Task types: *Onboarding* (V1–V4), *Add crew member*, *Add / replace van*, *Renew document* (pre-filled to that document), *Update contact*, *Fix sent-back item*.

1. Axis admin creates a **vendor shell** — company name, GSTIN, services provided, **authorised signatory name + mobile + email, ops contact name + mobile + email** (all required; optional escalation and billing email) — and clicks *Send task link → Onboarding*. Every task link goes to **both SMS and email** of the signatory, with the ops contact in copy; the signatory mobile is the OTP number. Email and mobile changes are themselves a task (*Update contact*) that goes through the three stages. Or Axis ops enters everything directly (flagged "entered by bank on behalf of vendor") — both paths stay available.
2. **Ongoing changes**: (a) renewal links are generated **automatically** 60 / 30 / 7 days before any document expires and sent to the vendor; (b) for anything else the vendor opens a public **"Request a link"** page — enters GSTIN + registered mobile → OTP → chooses the task → receives a fresh link; (c) or the vendor phones / emails Axis ops, who click *Send task link* or enter it themselves.
3. Status goes back to the vendor by SMS / email at each stage (checked, approved, sent back — a sent-back item comes with a new link to fix it). Every link sent, opened and used is logged with time, mobile and IP.
4. Why this is easier: nothing for the bank to administer (no vendor users, passwords, resets, deactivations); nothing for the vendor to remember; each link does one thing and expires; the audit log shows exactly who did what through which link. A read-only vendor view (P11) stays optional for later.

## 6.2 Vendor onboarding wizard (vendor portal, save-and-resume)

| Step | Content | Completeness |
|---|---|---|
| V1 Company | Legal name, GST, PAN, regions, billing type; agreement / MOU + dates; CIT insurance; PSARA (if gunmen); signatory + ops-contact passport photo and ID | all required docs present |
| V2 Vans | Add one by one, or **bulk Excel template** (registration, make, model, year, RC / insurance / PUC / fitness / permit numbers and dates) then upload the papers per van; year of manufacture checked against max age | every van has all 5 documents |
| V3 Crew | Add one by one, or bulk Excel for text fields; per person: role, passport photo, Aadhaar front / back, DL (driver) / arms licence + guard card (gunman) / police verification (all) | every crew member complete |
| V4 Review & submit | Summary with counts and missing items; "I confirm the information is true" with signatory name; *Submit for bank verification* | — |
| Later | No dashboard: status by SMS / email; renewals through automatic links; other changes through "Request a link" or Axis ops; each change goes through the three stages | — |

## 6.3 Bank side

- **Approval queue** (sidebar: *Approvals*, badge with count): tabs *Awaiting checker* / *Awaiting approver* / *Sent back* / *Done*; each row opens the dossier; filters by vendor, entity type, age in queue; SLA colour after 2 working days.
- **Checker view of the dossier**: checklist per document ("number matches scan", "validity read correctly", "photo matches ID"), remarks, buttons *Send back to vendor* / *Mark checked*.
- **Approver view**: checker's ticks and remarks shown read-only, own checklist, *Reject* / *Approve*; approval only possible when status is *Checked*.
- **Renewals**: 60 / 30 / 7-day alerts go to the vendor with a direct renewal link; the renewed document re-enters the queue; the old one stays valid until approved or expiry, whichever first.
- **Bulk approval** is not allowed — each dossier is approved individually (bank audit requirement); the checker may check several of one vendor in a session.

## 6.4 Data

`approval_requests`: id, entity_type, entity_id, version, maker_user_id, maker_type (vendor | bank_on_behalf), submitted_at, checker_user_id, checked_at, checker_remarks, approver_user_id, decided_at, decision (approved | rejected), reason, sla_due_at. `vendors` add: `signatory_name`, `signatory_mobile`, `signatory_email`, `ops_name`, `ops_mobile`, `ops_email`, `escalation_contact`, `billing_email` (mobile E.164 + email format validated; mobile unique across vendors). `task_links`: vendor_id, task_type (onboarding | add_crew | add_van | renew_document | update_contact | fix_sent_back), target_id (document / entity), token_hash, sent_to_mobile, sent_to_email, sent_by (user | system | vendor_request), sent_at, expires_at, opened_at, used_at, ip. `tenant_settings.approval_stages`.

## 6.5 Tests (section K, continued)

| # | Steps | Expected |
|---|---|---|
| K27 | Axis admin creates vendor shell → Send task link → Onboarding | Link arrives by **SMS and email** (signatory; ops contact in copy); logged; opens only after OTP to the signatory mobile; second use refused; expires after 7 days |
| K27d | Register vendor without signatory email or mobile | Submit disabled; invalid email / non-E.164 mobile rejected |
| K27a | Vendor opens "Request a link" with GSTIN + registered mobile → OTP → Add crew | Fresh task link received; wrong mobile refused |
| K27b | Document 60 days from expiry | Renewal task link sent automatically to the vendor, pre-filled to that document; logged |
| K27c | Bank ops enters a van on behalf of the vendor | Flagged "entered by bank"; that user cannot check or approve it |
| K28 | Vendor completes V1–V4 and submits | Status *Submitted · awaiting checker* on vendor, each van, each crew member; bank queue count updates |
| K29 | Vendor task link has no check / approve controls; API from that token refused | 403 |
| K30 | Bank operator opens dossier, ticks checklist, *Mark checked* | Status *Checked · awaiting approver*; operator cannot approve |
| K31 | Same operator logs in as admin and tries to approve | Refused: "Checker and approver must be different users" |
| K32 | Different admin approves | Active; dossier trail shows maker, checker, approver with times |
| K33 | Checker sends a van back with remark "PUC scan unreadable" | Vendor sees *Sent back* with the remark; re-upload restarts at stage 1 |
| K34 | Vendor edits an approved crew member's mobile | Record goes back to *Submitted*; old value still used until approved |
| K35 | Bulk Excel with one invalid year of manufacture | Row-level error shown; other rows imported as drafts |
| K36 | Set `approval_stages` = 2 on a test tenant | Checker stage skipped; maker → approver only |

# 5. Build order

1. Rename Drivers → Crew members, add role and the role-specific fields; migrate existing drivers as role = driver. Vendor "Services provided" + tenant `crew_sourcing` setting. (1 day UI + migration)
2. Run crew: `run_crew`, Create run crew section, run detail crew table, `min_crew` setting. (1 day)
3. Driver page: Crew on board step, block below minimum, crew shown on duty / complete, Hindi. (1 day)
4. Gate pass PDF + Print buttons + print log + QR. (1 day)
5. Gate pass readings card, Duties columns, flags, reports. (1 day)
6. Crew attendance export. (½ day)
7. Documents table + uploads on vendor / van / crew forms, nightly expiry job, alerts, Compliance page, run-planner blocking, gate-pass footer. (3 days)
8. Approval dossier PDF generator + Review-dossier approval flow + Dossier tab + audit pack ZIP + view/print logging. (3 days)
9. Vendor invite link + vendor onboarding wizard (V1–V4, bulk Excel) + three-stage approval queue (maker / checker / approver) + renewal links. (4 days)

# 6. Tests to add to the handbook (section K)

| # | Steps | Expected |
|---|---|---|
| K1 | Sidebar | "Crew members", no "Drivers" anywhere (portal, driver page, reports) |
| K2 | Add crew member role Gunman without DL fields | Accepted; DL fields hidden; arms licence + Aadhaar + uploads required |
| K3 | Add crew member role Driver without DL | Submit disabled |
| K3a | Upload a landscape group photo as the crew photo | Rejected: "Passport-type photo required — one face, plain background"; camera capture offered |
| K3b | Upload a valid passport-type photo | Accepted, auto-cropped to 7:9; appears on dossier cover, gate pass and crew-on-board screen |
| K3c | Log in as vendor user → Add crew member with photo + KYC | Saved as Pending for Axis approval; vendor cannot approve; dossier generated |
| K3d | Renew a driver's DL on a record whose photo is 3+ years old | Form demands a new photo before Submit; record returns to Pending |
| K3e | Register vendor without authorised-signatory photo | Submit disabled |
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
