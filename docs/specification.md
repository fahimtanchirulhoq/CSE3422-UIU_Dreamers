# Product specification

> **Student project, not medical advice.** All patients and hospitals in this project are made up.

*Living document.* The first draft is due before Class 3; coding starts as soon as it is merged. After the instructor's review it is tagged `spec-v1`, which the MVP is graded against. Keep it in step with the code: changes go through pull requests, and MVP-scope changes need the instructor's approval in the pull request.

---

## 1. Problem, users and roles

### Problem
During dengue season, many people in rural and everyday settings do not recognise early warning signs (abdominal pain, persistent vomiting) and lose hours reaching a hospital that cannot help.

- **How patients cope today:** They go to the nearest hospital they already know, follow advice from neighbours or relatives, or phone around.
- **Why that is not good enough:** No shared, live information exists on which hospital has dengue beds or platelet support, so patients are often sent onward from hospital to hospital. Many also arrive without a CBC/platelet report.
- **Our approach:** A Community Health Worker (CHW) enters the patient's symptoms into the portal. A transparent rule engine (no ML) gives a risk tier and, for critical cases, generates a referral ticket to one suitable hospital.

### Users and roles
Health workers enter symptoms, not patients. Patients are records, not users.

| Role | Responsibilities |
| :--- | :--- |
| **Community Health Worker (CHW)** | Registers a patient (age, area), enters symptoms, day of fever and optional labs; gets a triage result; for critical cases gets a referral ticket (QR/ID) and a route to the top 3 hospitals; views a patient's past assessments. |
| **Hospital Desk / Triage Doctor** | Sees incoming tickets for their own hospital; verifies a ticket on arrival and updates status (`Admitted`, `Under Observation`, `Discharged`); keeps dengue-bed and platelet availability up to date. |
| **System Admin** | Onboards and verifies hospitals; configures symptom weights, thresholds and tier cutoffs. |

---

## 2. User stories (MVP of 5 to 8, in priority order)

### DENGUE-1: As a CHW, I want to register a patient (age, area) and enter symptoms, day of fever and optional lab values, so that I get an immediate triage result.
- **Given** a logged-in CHW selects a patient and completes age, day of fever, symptoms, and all warning-sign questions (`Yes` / `No` / `Don't know`), **when** they submit the triage form, **then** the system computes the risk tier (`LOW`, `MODERATE`, or `CRITICAL`) using the WHO/DGHS rule engine and displays the result along with *"Student project, not medical advice."*
- **Given** a CHW submits the form with a required warning-sign question left unanswered or enters a negative platelet count, **when** validation runs, **then** the submission is blocked with a clear field-level error message *(unhappy path)*.
- **Given** the rule engine encounters a missing config or unexpected error during evaluation, **when** the form is submitted, **then** the system defaults to `CRITICAL` (*"Seek care now"*) rather than `LOW` *(unhappy path)*.

### DENGUE-2: As a CHW, I want a referral ticket (QR/ID) and a route to the top 3 hospitals when a case is critical, so that I can send the patient to a hospital that can treat them.
- **Given** a triage assessment results in `CRITICAL` (either via red-flag Rule 1 or score ≥ 70%), **when** the result screen loads, **then** the system issues one `ReferralTicket` (valid for 6 hours) to the nearest verified hospital with available dengue beds > 0 and platelet status not `UNAVAILABLE`, and shows routes to the top 3 hospitals on the map.
- **Given** no verified hospital currently has confirmed bed/platelet availability, **when** a critical ticket is generated, **then** the ticket is routed to the nearest verified hospital labelled *"capacity unknown, call ahead"* *(unhappy path)*.
- **Given** the OSRM routing API fails or is rate-limited, **when** calculating routes, **then** the system falls back to straight-line (haversine) distance labelled *"approximate"* *(unhappy path)*.

### DENGUE-3: As a CHW, I want to see a patient's past assessments, so that I can track whether they are getting worse across visits.
- **Given** a patient has prior triage records, **when** the CHW opens the patient's profile, **then** all past assessments are listed in reverse chronological order showing date, fever day, risk %, tier, and any issued referral ticket status.
- **Given** a CHW searches for a patient code that does not exist, **when** the search executes, **then** the UI shows *"No patient found"* and offers a button to register a new patient *(unhappy path)*.

### DENGUE-4: As a Hospital Triage Doctor, I want to see incoming tickets with a pre-arrival summary, so that my staff can prepare before the patient arrives.
- **Given** an active `ISSUED` referral ticket targets the doctor's hospital, **when** the hospital dashboard polls for updates (every 15 s), **then** the incoming ticket appears with patient age, fever day, red-flag reasons/symptoms, optional lab values, ETA, and expiry countdown.
- **Given** a ticket remains in `ISSUED` status past its 6-hour `expires_at` window, **when** the status is checked, **then** the ticket transitions to `EXPIRED` and is removed from active incoming arrivals *(unhappy path)*.

### DENGUE-5: As a Hospital Desk user, I want to verify a ticket on arrival and update its status, so that the referral record stays accurate.
- **Given** a patient arrives with a valid `ISSUED` ticket QR/ID assigned to that hospital, **when** the Hospital Desk staff verifies the ticket and sets status to `Admitted`, **then** the ticket updates to `ADMITTED`, available dengue beds decrement by 1, and exact patient location/`RouteEstimate` rows are deleted.
- **Given** staff enter an invalid, expired, or already-closed ticket ID, **when** they attempt verification, **then** the system rejects the action with the ticket's current status *(unhappy path)*.

### DENGUE-6: As a Hospital Desk user, I want to update dengue-bed and platelet availability, so that new referrals go to hospitals that have capacity.
- **Given** an authenticated Hospital Desk user updates available dengue beds and platelet status (`AVAILABLE`, `LOW`, `UNAVAILABLE`), **when** they save changes, **then** the hospital's capacity and `availability_updated_at` timestamp update immediately for future referrals.
- **Given** a hospital sets available dengue beds to `0` while an `ISSUED` ticket is still en route, **when** capacity saves, **then** the system flags the hospital full and reroutes unarrived tickets to the next hospital on the list *(unhappy path)*.

### DENGUE-7: As a System Admin, I want to onboard and verify hospitals, so that tickets only go to verified facilities.
- **Given** a System Admin enters a new hospital's details and marks it `VERIFIED`, **when** saved, **then** the hospital becomes eligible to receive referral tickets.
- **Given** a hospital is unverified or suspended, **when** the referral engine selects candidate hospitals, **then** unverified hospitals are excluded from referral routing *(unhappy path)*.

### DENGUE-8: As a System Admin, I want to configure symptom weights and thresholds, so that the rules can be tuned without changing code.
- **Given** a System Admin updates symptom points, lab thresholds, or tier cutoffs in `ClinicalRuleConfig`, **when** saved, **then** subsequent triage assessments evaluate against the updated configuration without code redeployment.
- **Given** an admin enters invalid tier cutoffs (e.g., `LOW` cutoff ≥ `CRITICAL` cutoff), **when** saving, **then** validation rejects the update and preserves the previous active config *(unhappy path)*.

---

## 3. Non-functional requirements (measurable)

| Quality | Requirement | How we will check it |
| :--- | :--- | :--- |
| **Performance** | Triage evaluation and referral ticket generation complete in **< 500 ms** (excluding external routing API); patient assessment history loads in **< 1 s** with 1,000 records. | Automated integration benchmark tests and API response-time logs. |
| **Polling Efficiency** | Hospital dashboard polling (`If-Modified-Since` / `ETag`) returns HTTP `304 Not Modified` in **< 100 ms** when no new tickets arrive and pauses when the browser tab is hidden. | Browser DevTools network inspection and server access logs. |
| **Security & Privacy** | Passwords hashed with bcrypt/Argon2; strict role-based access control (RBAC) isolating hospital staff to their own `hospital_id`; exact GPS/address and `RouteEstimate` purged immediately when a ticket closes. | Automated RBAC authorization tests and database inspection after ticket closure (`Discharged`, `Expired`, `Cancelled`). |
| **Reliability & Safety** | 100% deterministic rule evaluation; rule engine errors or missing config always fail safe to `CRITICAL` (*"Seek care now"*); OSRM failure falls back to haversine distance within **< 200 ms**. | Unit test suite covering all 4 worked examples, boundary values (`29/30%`, `69/70%`, platelets `49,999/50,000`), and fault-injection stubs. |
| **Usability / Accessibility** | Mobile-responsive form usable on 360 px screens; all warning signs offer explicit `Yes / No / Don't know` options; every result/ticket displays `"Student project, not medical advice."` and `"© OpenStreetMap contributors"`. | Manual UI checklist across mobile/desktop viewports and automated UI rendering assertions. |

---

## 4. Scope

### MVP
- **3-tier WHO/DGHS rule engine:**
  - **Rule 1 (Red-flag guardrails, checked first):** Severe/persistent abdominal pain or tenderness, persistent vomiting, mucosal/other bleeding, fainting/lethargy/restlessness, low BP (marked "low" or systolic < 90 mmHg in adults), fluid accumulation, enlarged tender liver, platelets < 50,000/µL, or hematocrit rising ≥ 20% with falling platelets → immediate `CRITICAL (RED-FLAG)`.
  - **Rule 2 (Missing-data aware weighted score):** Fever day 1–2 / 3 / 4+ (`5/10/15`), Age < 5 or > 60 (`10`), High-risk condition (`10`), Headache/eye pain (`5`), Muscle/joint pain (`5`), Rash (`5`), Nausea (`5`), Platelets 50,000–100,000/µL (`25`), Hematocrit above normal (`15`). Max = 55 (no labs), 80 (platelets only), 95 (both labs). Missing labs excluded from denominator.
  - **Rule 3 (Tiers):** `< 30%` = `LOW` (with return-if-worse advice; floor of `MODERATE` if fever day ≥ 3 and no platelet count), `30–69%` = `MODERATE` (CBC advised), `≥ 70%` or red flag = `CRITICAL`.
- **Single-hospital referral ticketing (Rule 4):** Issues QR/ID ticket (6-hour expiry) to nearest verified hospital with beds > 0 and platelets not `UNAVAILABLE` (fallback: nearest verified hospital with *"capacity unknown, call ahead"*), shows top 2 alternatives on CHW map without alerting them, handles full-hospital rerouting and 12-hour stale capacity warnings.
- **Role-based workflows:** CHW patient registration/triage/history, Hospital Desk/Doctor incoming queue (15 s HTTP polling) & arrival verification, System Admin hospital verification & `ClinicalRuleConfig` management.

### Out of scope (MVP)
- Area-wise analytics and referral dashboard
- Patient self-check / patient login
- SMS or push notifications
- WebSockets (hospital screens use HTTP polling)
- Machine-learning models
- Bed reservation and ambulance dispatch
- Any real patient or hospital data

---

## 5. Wireframes

Wireframes will be added under `docs/img/` before merging `spec-v1`:
- `docs/img/chw-triage-form.png` — CHW patient registration, symptom & warning-sign checklist (`DENGUE-1`)
- `docs/img/chw-referral-result.png` — Critical triage result, QR referral ticket & top-3 hospital map (`DENGUE-2`)
- `docs/img/chw-patient-history.png` — Patient longitudinal assessment timeline (`DENGUE-3`)
- `docs/img/hospital-incoming-queue.png` — Hospital incoming tickets, arrival verification & bed/platelet updater (`DENGUE-4`, `DENGUE-5`, `DENGUE-6`)
- `docs/img/admin-config-hospitals.png` — System Admin hospital verification & `ClinicalRuleConfig` editor (`DENGUE-7`, `DENGUE-8`)

---

## 6. Main entities and relationships

| Entity | Key fields | Related to |
| :--- | :--- | :--- |
| **`User`** | `id`, `name`, `phone`, `password_hash`, `role` (`CHW`, `HOSPITAL_STAFF`, `ADMIN`), `hospital_id` (nullable) | Belongs to `Hospital` (staff only); creates `Patient` and `TriageAssessment` records. |
| **`Patient`** | `id`, `patient_code` (no real name), `age`, `area` (coarse upazila/district), `created_by` | Created by `User` (CHW); has many `TriageAssessment` records. |
| **`Hospital`** | `id`, `name`, `address`, `latitude`, `longitude`, `total_dengue_beds`, `available_dengue_beds`, `platelet_status` (`AVAILABLE`, `LOW`, `UNAVAILABLE`), `verification_status`, `availability_updated_at` | Has many `User` (staff), `ReferralTicket` (target), and `RouteEstimate` records. |
| **`TriageAssessment`** | `id`, `patient_id`, `entered_by`, `fever_day`, `symptoms_json`, `warning_signs_json`, `platelet_count` (opt), `hematocrit` (opt), `blood_pressure` (opt), `temp_location` (purged on close), `risk_percent`, `tier` (`LOW`, `MODERATE`, `CRITICAL`), `red_flag_reason` | Belongs to `Patient` and `User` (CHW); generates zero or one `ReferralTicket` and up to 3 `RouteEstimate` rows. |
| **`ReferralTicket`** | `id`, `assessment_id`, `hospital_id`, `ticket_code` (QR/ID), `pre_arrival_summary`, `status` (`ISSUED`, `ADMITTED`, `UNDER_OBSERVATION`, `DISCHARGED`, `EXPIRED`, `CANCELLED`), `issued_at`, `expires_at` | Belongs to `TriageAssessment` and target `Hospital`. |
| **`RouteEstimate`** | `id`, `assessment_id`, `hospital_id`, `distance_km`, `eta_minutes`, `is_approximate` (deleted when ticket closes) | Links `TriageAssessment` to top 3 candidate `Hospital` rows. |
| **`ClinicalRuleConfig`** | `id`, `symptom_weights_json`, `platelet_threshold`, `hematocrit_threshold`, `tier_cutoffs_json`, `ticket_expiry_hours`, `updated_at` | Standalone active configuration read by the triage rule engine. |

---

## 7. Risks, KPIs and scale estimate

### Biggest risks (≤ 50 words)
A wrong "Low risk" result for a patient who is actually in danger. We prevent it: all WHO warning signs force referral, "don't know" counts as "yes", fever day 3+ without a CBC can never be Low, and any engine error returns "seek care now".

*(Secondary risk: hospital staff delaying bed updates → mitigated via 12-hour "last updated" warnings and "capacity unknown, call ahead" tickets.)*

### KPIs
1. **Zero false-LOW safety rate:** **100%** of test cases containing any WHO warning sign, platelets < 50,000/µL, or fever day ≥ 3 without CBC never receive a `LOW` tier.
2. **Referral routing speed:** **≥ 95%** of `CRITICAL` triage submissions issue a valid referral ticket and top-3 hospital routes within **2 seconds** end-to-end.
3. **Hospital capacity freshness:** **≥ 80%** of active verified hospitals update their dengue-bed and platelet availability at least once every **12 hours** during simulated dengue-season load.

### Scale estimate (working shown)
- **Assumptions:** 50,000 registered users after 1 year; 5,000 daily active users (DAU).
- **Requests per day & Peak:**
  - ~10 requests per active user per day (login, load form, submit, view result, history, map) → `5,000 × 10 = 50,000 requests/day`
  - Average = `50,000 ÷ 86,400 s ≈ 0.6 requests/s`
  - **Peak factor 5x** (dengue season; most CHW activity occurs in a ~4–5 hour window): `0.6 × 5 = ~3 requests/s` (~70% reads = **~2 reads/s**, ~30% writes = **~1 write/s**).
- **Hospital screen polling (all day):**
  - 50 hospital screens polling once every 10–15 s → `50 ÷ 15` to `50 ÷ 10 = 3.3–5 reads/s` continuously.
  - **Combined peak:** `~3 + 5 = ~8 requests/s` (**~7 reads/s, ~1 write/s**).
- **Storage after 1 year:**

| Table | Rows | Bytes/row | Size |
| :--- | :--- | :---: | :---: |
| `TriageAssessment` | `2,000/day × 365 = 730,000` | 500 | 365 MB |
| `ReferralTicket` | `10% critical = 73,000` | 800 | 58 MB |
| `RouteEstimate` | `3 per ticket = 219,000` *(deleted after close, peak kept)* | 200 | 44 MB |
| `Patient` | 100,000 | 200 | 20 MB |
| `User` | 50,000 | 300 | 15 MB |
| `Hospital`, `ClinicalRuleConfig` | ~200 | 500 | < 1 MB |
| **Total** | | | **≈ 500 MB** *(≈ 1 GB with indexes)* |

- **Architectural takeaway:** One backend server and one PostgreSQL database are sufficient (no microservices). Polling uses conditional requests (`If-Modified-Since` / `ETag` returning `304`) every 15 s only while the tab is visible.

---

## 8. Privacy and ethics

- **Synthetic data only:** Made-up patients and hospitals only. Real names, NIDs, or real phone numbers must never be entered in seeds, tests, or demos.
- **What personal data we store and why:**
  - `Patient`: synthetic `patient_code`, `age`, and coarse `area` (upazila/district) to support multi-visit tracking without personally identifiable information (PII).
  - `TriageAssessment`: clinical symptoms, fever day, optional lab values, and temporary patient GPS/address needed strictly to compute road ETA to nearby hospitals.
- **Who can see it:**
  - CHWs can register patients and view triage assessments.
  - Hospital Desk / Triage Doctors can only view incoming `ReferralTicket` pre-arrival summaries assigned to their own `hospital_id`.
  - System Admins manage hospital verification and rule weights without needing patient-level clinical browsing.
- **Retention & deletion:** Exact patient location (GPS/address) and matching `RouteEstimate` rows are **deleted immediately** when a ticket closes (`Discharged`, `Expired`, or `Cancelled`). Only the coarse `area` (upazila/district) is retained.
- **Medical disclaimer:** Every result screen and referral ticket prominently displays: *"Student project, not medical advice."*

---

## 9. External API

- **APIs used:** OpenStreetMap **Nominatim** (geocoding patient address to coordinates) and **OSRM** public demo server (driving distance and road ETA). Both are free and require no API key.
- **Free / sandbox limits & compliance rules:**
  - Maximum **1 request per second** to each service enforced via a server-side queue.
  - Custom `User-Agent` header identifying our application sent on every request.
  - `"© OpenStreetMap contributors"` attribution displayed on every map and routing result.
  - No Nominatim autocomplete; geocoding runs only when the CHW clicks **Search**, and repeated area lookups are cached.
  - Hospital coordinates are stored in PostgreSQL at onboarding; we pre-filter the top 5 hospitals by straight-line distance and make a single OSRM table/route request.
- **Abstraction & Fallback when down:**
  - Both services sit behind internal interfaces (`GeocodingService`, `RoutingService`); automated tests use stubs and never call live public servers.
  - If OSRM or Nominatim times out, fails, or hits rate limits, the system falls back to straight-line (**haversine**) distance calculation and labels the ETA/distance as *"approximate"*.
