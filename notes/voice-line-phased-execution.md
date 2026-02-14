# Voice Line: Phased Execution (Transfer First → Standardized Import Last)

**Status:** Agent works with `check_reservation` against existing Supabase `reservations` table. Missing: driver phone for transfer, `import_meta` table, transfer flow, optional upload UI, and CSV/Excel import in a standardized format.

**Order:** We build transfer and operator UX first; data is assumed to exist (manual entry, one-off scripts, or existing tools). The **creation of the import of data in a standardized format** (parser, column mapping, upsert) is **Phase 5** and happens last before deferred work.

---

## Current POC DB (ReThread / Supabase)

| Table | Purpose | Relevant for transfer? |
|-------|---------|------------------------|
| **reservations** | Trip data; used by `check_reservation`. Columns: id (uuid), confirmation_number, customer_name, telephone, pickup_time, dropoff_time, flight_time, pickup_address, dropoff_info, terminal, direction, flight_info, special_instructions, vehicle_assigned, **driver_assigned** (text), num_passengers, status, created_at. RLS off. | Yes. Has `driver_assigned` (driver name); **no driver_phone**. |
| **drivers** | Driver roster. Columns: id (text PK), name, available_days, created_at. RLS off. | Yes. **No `phone` column** — must add for transfer. |
| **import_meta** | — | **Does not exist.** Phase 1.1 adds it. |
| vehicles | Fleet. id, make, model, year, etc. RLS off. | No (reference only). |
| pending_after_hours_calls | After-hours call queue (call_sid, phone, place_id). RLS off. | No (separate flow). |
| transport_leads | Lead/call tracking (phone, place_id, call_sid, recording_url, etc.). RLS off. | No (separate flow). |
| leads | Form leads (source, form_data, email, metadata). RLS on. | No (separate flow). |

**Gaps for transfer:** (1) **import_meta** missing — add table. (2) **Driver phone** missing — `drivers` has no `phone`; `reservations` has `driver_assigned` (name). Add `phone` to `drivers` and resolve driver phone via join: `reservations.driver_assigned` → `drivers.name` (or `drivers.id` if you store id in driver_assigned).

**Driver data split:** Driver *name* is per-reservation (from daily CSV → `reservations.driver_assigned`). Driver *phone* is in `drivers` and must be maintained separately (driver list). Front end needs both: reservation upload and a way to manage the driver list so the agent can resolve transfer target. Working hours / available_days are not used for transfer.

### Execution Protocol

1. **One phase at a time**: Implement a single phase (e.g. 1.1, 1.2), then:
   - Run any project lint/tests (e.g. `npm run lint` or project-specific checks).
   - Use **ReadLints** on the file(s) you edited; re-run after a short wait so TypeScript/IDE errors are reported.
   - If errors exist: fix before proceeding. If clean: mark the substep `[x]` complete in this file.

---

## Phase 1: Data Model for Transfer (No Import Yet)

Schema and metadata so transfer flow can run. Data is populated by other means (manual, one-off script, or existing `migrate_csvs.py`) until Phase 5.

### 1.1 Import Metadata Table

- [x] Add migration: `import_meta` table with at least:
  - `id`, `source` (e.g. filename or "cli"), `last_import_at` (timestamptz), `row_count`, `created_at`
- [x] Optional: `checksum` or `file_name` for idempotency / debugging
- [x] Document purpose: gate "import fresh enough?" in agent/transfer logic

**Acceptance Criteria:**

- [x] Table exists in Supabase; can insert/select `last_import_at`
- [x] Migration is reversible or documented in repo
- [x] For now: one row can be inserted manually or by a one-off script to represent "data is fresh" for testing transfer

### 1.2 Driver Phone in Data Model

- [x] **Add `phone` column to `drivers`** (POC has `drivers.id`, `drivers.name`, `drivers.available_days` only — no phone today).
- [x] Use existing link: `reservations.driver_assigned` (text) holds driver name → join to `drivers.name` (or store driver id in `driver_assigned` and join to `drivers.id`) to get `drivers.phone`.
- [x] Document in integration plan: driver phone comes from `drivers.phone` via join on `driver_assigned`.
- [x] Data for driver phone is entered manually or via existing tools until Phase 5.

**Acceptance Criteria:**

- [x] `drivers.phone` exists; each reservation can resolve to a driver phone via `driver_assigned` → drivers.
- [x] Edge function or agent can read driver phone for transfer (implementation in Phase 2).
- [x] No parser/import yet; population is out-of-band.

---

## Phase 2: Transfer Flow (Must Ship)

Agent confirms reservation, then offers to transfer caller to driver. Only if import is fresh enough.

### 2.1 "Import Fresh Enough" Guard

- [x] In `check_reservation` or a small helper: read `import_meta.last_import_at` (latest row)
- [x] Define policy (e.g. consider fresh if within last 24 hours); expose as `import_fresh: boolean` or equivalent in response
- [x] Agent prompt: if reservation found but import not fresh, do not offer transfer; take message instead

**Acceptance Criteria:**

- [x] API returns whether data is considered fresh
- [x] Agent does not offer driver transfer when data is stale
- [x] Agent still answers reservation lookup and takes message when stale

### 2.2 Retell Transfer to Driver

- [x] Add Retell *Transfer call* (or Call Transfer Node) using driver phone from DB
- [x] Ensure driver phone is returned from `check_reservation` or from a follow-up read; pass to Retell transfer
- [x] Test: call in, give confirmation/phone, ask for driver → call transfers to driver number

**Acceptance Criteria:**

- [x] After confirming reservation, agent offers to connect to driver
- [x] When caller accepts, Retell transfers to the driver phone from Supabase
- [x] Caller can speak to driver; call ends normally
- [x] No transfer offered when driver phone missing or data stale

---

## Phase 3: Operator-Facing Import UX (Should Ship)

Reduce friction so operator doesn’t need to run CLI. Upload accepts file and will process it once Phase 5 parser exists (or stub with "coming in Phase 5" and wire in Phase 5).

**Implementation:** Edge Function `supabase/functions/import-upload` (GET = latest `import_meta`, POST = multipart: `type` reservations|drivers, `file` CSV). Auth via `UPLOAD_API_KEY` (header `X-Import-API-Key` or `Authorization: Bearer`). Operator page: `all-an-ai/landing/operator-import.html`. Doc: `docs/OPERATOR_IMPORT.md`. A React (or other SPA) operator UI for a more intuitive upload is optional polish (e.g. Phase 4 or later); the current static page is sufficient to ship.

**Env vars (server-side only):** Set `UPLOAD_API_KEY` in **Supabase Dashboard → Edge Functions → import-upload → Secrets**. That value is what the operator (or your React app) sends in the request header so the function can reject unauthenticated uploads. You do not add env vars to the static HTML page; the operator pastes the key into the form. No client-side env is required.

### 3.1 Upload Endpoint or Page

- [x] Option A: Supabase Edge Function that accepts multipart file upload; when Phase 5 exists, parses CSV/Excel, runs upsert, updates `import_meta`
- [x] Option B: Simple web page with file input, POST to Edge Function or backend, then redirect with success/error
- [x] Auth: protect upload (e.g. Supabase Auth or API key) so only operator/admin can upload
- [x] Show last import time and row count after upload (from `import_meta`)
- [x] **Driver list:** Provide a way to maintain driver name + phone (e.g. upload a driver roster CSV, or a simple form/table) so `drivers` is populated/updated independently of the reservation CSV. Required for transfer; can be one-time or ongoing. (Reservation CSV supplies driver *name* only.)

**Acceptance Criteria:**

- [x] Operator can upload a CSV (or Excel) without running a script locally (once Phase 5 is done; until then endpoint can return "import not yet available" or wire when Phase 5 lands)
- [x] Reservations and driver phone update in Supabase after upload (when Phase 5 parser is wired)
- [x] `last_import_at` updates and is visible to operator
- [x] Upload is not publicly writable without auth

### 3.2 Error Handling and Feedback

- [x] On parse failure: return clear error (e.g. "Missing column: telephone") and do not update `import_meta`
- [x] Optional: store last error message or log for debugging
- [x] Document supported column names / template for operator (aligned with Phase 5 format)

**Acceptance Criteria:**

- [x] Invalid or malformed file returns a clear error message
- [x] Successful upload shows row count and timestamp
- [x] One documented "expected columns" or sample template for the supported export format

---

## Phase 4: Polish and Safety (Before First Operator)

### 4.1 Retell Request Verification

- [x] Verify `X-Retell-Signature` (or Retell’s recommended method) in `check_reservation` Edge Function so only Retell can call it with real traffic
- [x] Document required env (e.g. Retell webhook secret) in README or setup doc

**Acceptance Criteria:**

- [x] Edge Function rejects requests without valid signature when secret is set
- [x] Retell dashboard webhook still works for your agent

**Implementation:** When `RETELL_WEBHOOK_SECRET` is set on `check-reservation`, the function verifies the `x-retell-signature` header (HMAC-SHA256 of raw body with the secret). If unset, verification is skipped so the dashboard and testing still work. Use your Retell API key (with webhook badge) as the secret. See Retell [Secure the webhook](https://docs.retellai.com/features/secure-webhook). Documented in runbook and §7 of integration plan.

### 4.2 Confirmation Number from Export

- [x] Prefer operator’s confirmation/trip id over generated ids so caller and dispatch use same reference
- [x] If export has no id: document "we generate RES-YYYYMMDD-NNNN" and keep behavior consistent (used when Phase 5 parser runs)

**Acceptance Criteria:**

- [x] When export contains Trip# or confirmation, that value is stored and returned by `check_reservation`
- [x] Caller can use the number from their booking email/screen and agent finds the reservation

**Implementation:** Parser (Phase 5) will use Trip# from export when present; otherwise generate `RES-YYYYMMDD-NNNN`. Documented in `docs/OPERATOR_IMPORT.md` and `docs/OPERATOR_DAILY_RUNBOOK.md`. `check_reservation` already returns whatever `confirmation_number` is in the DB.

### 4.3 Docs and Runbook

- [x] Update `voice-agent-integration-plan.md` or README with: how to run CSV import (CLI + upload URL once Phase 5 exists), how often to import, what "import fresh" means
- [x] One-page runbook: "Operator daily flow" (export from Hudson → upload here → agent uses data until next import)

**Acceptance Criteria:**

- [x] New teammate or operator can follow docs to perform a daily import (once Phase 5 is live)
- [x] Runbook lists any env vars, URLs, and expected export format

**Implementation:** §7 added to `voice-agent-integration-plan.md` (import how-to, how often, “import fresh”). Runbook: `docs/OPERATOR_DAILY_RUNBOOK.md` (daily flow, env vars, URLs, expected CSV formats).

---

## Phase 5: Standardized Import (Last – CSV/Excel in a Standard Format)

**This phase is done last.** It creates the import of data in a standardized format: one parser, one column mapping, upsert into `reservations` and driver data, and `import_meta` update.

### 5.1 Parser for One Operator Export Format

- [x] Pick one real export format (e.g. Ahmad’s cousin’s Hudson export or existing `Oct 26 Data (3).csv` column layout)
- [x] Add script or module that: reads CSV/Excel, maps columns to `reservations` schema (confirmation_number, customer_name, telephone, pickup_time, dropoff_time, flight_time, pickup_address, dropoff_info, terminal, direction, flight_info, special_instructions, vehicle_assigned, driver_assigned, num_passengers, status)
- [x] Confirmation number: use from export when present (e.g. Trip#), otherwise generate deterministic id
- [x] Upsert into `reservations` (match by confirmation_number or business key), then write one row to `import_meta` with `last_import_at = now()`, row_count
- [x] Driver phone: reservation CSV has driver *name* only. Populate `drivers.phone` from a separate driver list (upload or UI from Phase 3), not from the reservation export. Parser writes `driver_assigned` to reservations; driver phone is resolved at call time via join to `drivers`.
- [x] Handle encoding (UTF-8, latin-1) and missing columns gracefully

**Acceptance Criteria:**

- [x] Run script with sample CSV → `reservations` rows appear/update in Supabase
- [x] `import_meta.last_import_at` updates after each run
- [x] `check_reservation` returns correct data for a reservation from the imported file
- [x] No duplicate reservations for same confirmation/trip id when re-running import
- [x] Driver phone available via `drivers` (from driver list); `driver_assigned` on reservation links to driver for transfer

**Implementation:** Hudson-style parser in `import-upload` Edge Function (by column index). Unique index on `reservations(confirmation_number)` (migration `20250211000003_reservations_confirmation_unique.sql`) for upsert. CLI: `migrate_csvs.py` uses Trip# when present, upserts reservations, writes `import_meta`. Upload: same parser in Edge Function; POST `type=reservations` + file → parse, upsert, insert `import_meta`.

### 5.2 Wire Parser to Upload (if Phase 3 built first)

- [x] Upload endpoint or page from Phase 3 calls this parser (or CLI logic) so operator upload triggers same upsert and `import_meta` update
- [x] Document which format is supported (single CSV vs. two files) in integration plan

**Acceptance Criteria:**

- [x] Operator upload uses the same code path as CLI import
- [x] One documented "expected columns" or sample template

---

## Phase 6: Deferred (Post–First Operator)

### 6.1 Second Operator Export Format

- [ ] Add second parser or column mapping for a different export (e.g. Limo Anywhere vs. Hudson)
- [ ] Optional: upload page lets operator pick "Format: Hudson / Limo Anywhere" or auto-detect from headers

**Acceptance Criteria:**

- [ ] Both formats can be imported without code deploy (config or dropdown)
- [ ] `reservations` schema remains single; mapping is the only difference

### 6.2 Optional API Sync (Hudson / Fleetio)

- [ ] Only if an operator or partnership requires it: replace or supplement CSV import with scheduled sync from Hudson/Fleetio APIs
- [ ] Reuse same `reservations` (and driver) schema; only ingestion path changes
- [ ] Keep "import fresh" concept (e.g. last_sync_at) for transfer guard

**Acceptance Criteria:**

- [ ] Sync job writes to same tables as CSV import
- [ ] Agent and transfer logic unchanged
- [ ] Documentation updated to describe both "CSV upload" and "API sync" options

---

## Launch Checklist (First Operator Demo)

- [ ] Agent answers on Retell number and looks up reservation by confirmation # or phone
- [ ] CSV/Excel import (CLI or upload) populates `reservations` and driver phone (Phase 5 complete)
- [ ] "Import fresh" guard prevents transfer when data is stale
- [ ] When data is fresh, agent offers and completes transfer to driver
- [ ] Retell webhook/request verification enabled for production
- [ ] Operator runbook and export format documented
- [ ] One successful end-to-end test: import → call in → lookup → transfer to driver
