# Multi-Tenant Retell AI: Phased Migration Plan

**Status:** Single-tenant POC works — one Retell agent ("Reservation & After-Hours Check") for Sun City Express with hard-coded prompt, one phone number, `check_reservation` queries all reservations with no tenant filter. Dashboard has reservation upload, driver list, and import status.

**Goal:** Any transportation company can sign up, configure their own voice agent (company name, welcome message, business hours, transfer preferences), attach their existing phone number, and have the agent only see their reservations/drivers. One shared Retell agent, N phone numbers, tenant isolation via dynamic variables and `user_id` filtering.

**Approach:** Retell's **inbound webhook** + **dynamic variables** + **agent_override**. When a call comes in on any imported number, our webhook looks up the tenant, injects their config as dynamic variables, and the agent behaves as if it's their dedicated agent. `check_reservation` filters by `user_id`.

---

## Current State

| Component | Status | Multi-tenant ready? |
|-----------|--------|---------------------|
| **Retell agent** | Single agent, hard-coded prompt for Sun City Express | No — prompt references one company |
| **check_reservation** | Edge function, queries `reservations` with no `user_id` filter | No — returns any company's data |
| **import_meta freshness** | Checks latest row globally | No — should check per-tenant |
| **Driver phone lookup** | Joins `drivers` by name, no `user_id` filter | No — could return wrong company's driver |
| **Dashboard** | Supabase Auth (`user_id` on rows), RLS on tables | Partially — data is isolated, but no agent config |
| **Phone numbers** | One Retell-purchased number bound to agent | No — need per-tenant number import |

### Zero-Downtime Constraint

**The existing Sun City Express setup (+1 520-680-0005, user moazgelhag) must keep working throughout every phase.** No phase should break the current single-tenant flow. Each phase is additive — the existing Retell agent, phone number, and `check_reservation` continue to function unchanged until the new multi-tenant path is explicitly switched on and verified.

- Phases 1-2: Add table + dashboard tab. No impact on Retell or edge functions.
- Phase 3: Templatize the Retell prompt with dynamic variables populated with Sun City Express defaults. Verified identical behavior before proceeding.
- Phase 4: Add tenant filtering with backward compat (no `user_id` in args = no filter = current behavior).
- Phase 5: Webhook is additive — only fires when a number is configured with the webhook URL. The existing Retell-purchased number stays as-is until explicitly migrated.

### Execution Protocol

1. **One phase at a time**: Implement a single phase (e.g. 1.1, 1.2), then:
   - Run any project lint/tests (e.g. `npm run lint` or project-specific checks).
   - Use **ReadLints** on the file(s) you edited; re-run after a short wait so TypeScript/IDE errors are reported.
   - If errors exist: fix before proceeding. If clean: mark the substep `[x]` complete in this file.
2. **Deploy edge functions** after each phase that modifies them.
3. **Test with existing Sun City Express number (+15206800005)** after every phase to confirm no regression.

---

## Phase 1: Tenant Configuration Table

Store per-company settings so the webhook and dashboard can read/write them. This is the foundation everything else builds on.

### 1.1 Create `tenant_config` Table

- [x] Migration: `tenant_config` table with columns:
  - `id` uuid PK (default `gen_random_uuid()`)
  - `user_id` uuid REFERENCES auth.users(id) UNIQUE — one config per auth user (the operator)
  - **Company / operator info:**
    - `company_name` text NOT NULL
    - `operator_name` text — name of the primary operator / dispatcher
    - `operator_phone` text — operator's direct phone number (for escalation, not for callers)
    - `operator_email` text — contact email (may duplicate auth email; editable)
    - `timezone` text DEFAULT 'America/Phoenix' — IANA timezone for business hours evaluation
  - **Agent behavior:**
    - `welcome_message` text — the greeting the agent speaks first
    - `business_hours_start` integer DEFAULT 6 — hour (0-23) in the tenant's timezone
    - `business_hours_end` integer DEFAULT 22 — hour (0-23) in the tenant's timezone
    - `after_hours_message` text — what agent says outside hours
    - `transfer_enabled` boolean DEFAULT true — allow call transfer to driver
  - **Call routing / forwarding schedule:**
    - `forwarding_mode` text DEFAULT 'always' — when calls forward to Retell AI:
      - `'always'` — all calls go to Retell (company has no live dispatchers, or uses Retell 24/7)
      - `'after_hours'` — calls forward to Retell only outside `business_hours_start`–`business_hours_end`; during business hours calls ring the office as normal (configured at the SIP trunk / carrier level)
      - `'overflow'` — calls forward to Retell only when the office doesn't answer (ring timeout, configured at carrier)
    - `office_phone` text — the office landline or dispatch number that rings during business hours (used for forwarding rules). Nullable if `forwarding_mode = 'always'`.
  - **Telephony / SIP:**
    - `phone_number` text — E.164 format (e.g. +15551234567), the company's inbound number
    - `retell_phone_number_id` text — Retell's ID after importing the number
    - `sip_trunk_uri` text — termination URI from their SIP provider
    - `sip_trunk_provider` text — 'twilio' | 'telnyx' | 'vonage' | 'other' (for setup instructions)
    - `sip_username` text — SIP auth username (nullable)
    - `sip_password` text — SIP auth password (nullable, encrypted or stored as Vault secret)
  - **Timestamps:**
    - `created_at` timestamptz DEFAULT now()
    - `updated_at` timestamptz DEFAULT now()
- [x] RLS: authenticated users see/update only their own row (`auth.uid() = user_id`)
- [x] Index on `phone_number` for webhook lookup (unique)
- [x] `updated_at` auto-updates via trigger or application code

**Acceptance Criteria:**

- [x] Table exists in Supabase with RLS enabled
- [x] Each user can have exactly one `tenant_config` row
- [x] Lookup by `phone_number` is fast (indexed)
- [x] Migration is reversible

### 1.2 Seed Sun City Express Config

- [x] Insert a `tenant_config` row for the existing Sun City Express user (`moazgelhag`, auth user `3cf84beb-9238-4a53-98b3-f4e4e527663c`) with:
  - `company_name`: "Sun City Express"
  - `phone_number`: "+15206800005"
  - `business_hours_start`: 6, `business_hours_end`: 22
  - `timezone`: "America/Phoenix"
  - `forwarding_mode`: "after_hours"
  - `welcome_message`: current Retell welcome message
  - `transfer_enabled`: true
- [x] Verify dashboard can read the row via Supabase client
- [x] Verify +15206800005 still works as before (no changes to Retell config in this phase)

**Acceptance Criteria:**

- [x] Existing Sun City Express operator has a config row with correct values
- [x] No disruption to current single-tenant flow — agent, number, and check_reservation unchanged
- [x] Dashboard can read the config (visible in Phase 2 settings tab)

---

## Phase 2: Dashboard Agent Settings Tab

Operators configure their company name, welcome message, business hours, and transfer preferences from the dashboard. No Retell API calls yet — just writing to `tenant_config`.

### 2.1 Agent Settings Component

- [x] New component: `AgentSettings.tsx`
  - Loads `tenant_config` for current user (upsert on first save if row doesn't exist)
  - **Section 1 — Company Info:**
    - Company name (text, required)
    - Operator name (text)
    - Operator phone (tel)
    - Operator email (email, pre-filled from auth)
    - Timezone (dropdown — US timezones to start)
  - **Section 2 — Agent Behavior:**
    - Welcome message (textarea, with a default template shown as placeholder)
    - After-hours message (textarea)
    - Business hours start / end (dropdowns, 12AM-11PM)
    - Transfer enabled (toggle)
  - **Section 3 — Call Routing:**
    - Forwarding mode (radio: Always / After hours only / Overflow)
    - Office phone number (shown when mode is "after_hours" or "overflow")
    - Explanation text per mode so the operator understands what each does
  - **Section 4 — Phone Number** (read-only in this phase; Phase 6 makes it editable):
    - Shows connected number or "Not configured"
    - Status badge: connected / not connected
  - Save button writes to `tenant_config` via Supabase client
  - Success/error feedback
  - `updated_at` set on each save

### 2.2 Add Tab to Dashboard

- [x] Add `"settings"` tab to `Dashboard.tsx` tab bar
- [x] Render `AgentSettings` when selected
- [x] Tab label: "Agent Settings"
- [x] Position it as the last tab (after Import Status) — operators configure once, use rarely

**Acceptance Criteria:**

- [x] Operator can edit and save: company name, operator name, operator phone, operator email, timezone, welcome message, after-hours message, business hours, transfer toggle, forwarding mode, office phone
- [x] Values persist across page reloads
- [x] New user sees empty form with sensible placeholders; first save creates the row
- [x] Forwarding mode explanation is clear (operator understands "after hours" means calls go to their office during the day)
- [x] No impact on existing reservation/driver functionality

---

## Phase 3: Retell Agent Prompt Templatization

Convert the current hard-coded Retell prompt to use dynamic variables so per-tenant config can be injected at call time.

### 3.1 Update Retell Agent Prompt

- [ ] Replace hard-coded "Sun City Express" with `{{company_name}}` in the Retell dashboard prompt
- [ ] Replace hard-coded welcome message with `{{welcome_message}}`
- [ ] Replace hard-coded business hours ("6 AM to 10 PM") with `{{business_hours}}`
- [ ] Add `{{user_id}}` variable — will be passed to `check_reservation` tool call args
- [ ] Add `{{transfer_enabled}}` variable — agent only offers transfer when "true"
- [ ] Test with manually-set dynamic variables in Retell dashboard to confirm variables render

### 3.2 Update `check_reservation` Tool Schema in Retell

- [ ] Add `user_id` as a parameter in the `check_reservation` function definition in Retell
- [ ] The agent should always pass `{{user_id}}` when calling `check_reservation`
- [ ] This doesn't change the edge function yet (Phase 4 does that)

**Acceptance Criteria:**

- [ ] Retell agent prompt uses `{{company_name}}`, `{{welcome_message}}`, `{{business_hours}}`, `{{user_id}}`, `{{transfer_enabled}}`
- [ ] Test call with Sun City Express values works identically to before
- [ ] No regression — current flow unchanged when variables are populated with existing values

---

## Phase 4: Tenant-Scoped Edge Functions

Make `check_reservation` and `import_meta` freshness checks tenant-aware.

### 4.1 `check_reservation` Tenant Filtering

- [ ] Accept `user_id` from Retell tool call args (sent via dynamic variable)
- [ ] Add `.eq("user_id", userId)` to the reservation query
- [ ] Add `.eq("user_id", userId)` to the `import_meta` freshness check
- [ ] Add `.eq("user_id", userId)` to the driver phone lookup
- [ ] **Backward compat:** If `user_id` is not provided (legacy calls), skip filter (no regression)
- [ ] Use business hours from args or fall back to env defaults

### 4.2 Dynamic Business Hours

- [ ] Accept `business_hours_start` and `business_hours_end` from tool call args (or dynamic variables passed through)
- [ ] Replace hard-coded `BUSINESS_HOURS_START = 6` / `BUSINESS_HOURS_END = 22` with values from args, falling back to constants
- [ ] This allows per-tenant business hours without edge function changes

**Acceptance Criteria:**

- [ ] `check_reservation` only returns reservations for the given `user_id`
- [ ] Import freshness is checked per-tenant (not globally)
- [ ] Driver phone lookup is scoped to the tenant
- [ ] Calls without `user_id` (legacy) still work (no filter applied)
- [ ] Business hours are per-tenant
- [ ] Deploy edge function and test

---

## Phase 5: Inbound Call Webhook

The central piece: Retell calls this webhook on every inbound call. It looks up the tenant by phone number and returns dynamic variables.

### 5.1 Create `inbound-webhook` Edge Function

- [ ] New edge function: `supabase/functions/inbound-webhook/index.ts`
- [ ] Receives POST from Retell with `{ to_number, from_number, ... }`
- [ ] Looks up `tenant_config` by `phone_number = to_number`
- [ ] Returns JSON response:
  ```json
  {
    "call_inbound": {
      "dynamic_variables": {
        "company_name": "Sun City Express",
        "welcome_message": "Thank you for calling Sun City Express...",
        "business_hours": "6 AM to 10 PM",
        "business_hours_start": "6",
        "business_hours_end": "22",
        "user_id": "abc-123-...",
        "transfer_enabled": "true"
      },
      "metadata": {
        "tenant_id": "abc-123-..."
      }
    }
  }
  ```
- [ ] If no tenant found for the number: return default/fallback config or reject
- [ ] Must respond within 10 seconds (Retell timeout)
- [ ] Deploy with `--no-verify-jwt` (Retell calls it directly, not via Supabase auth)

### 5.2 Register Webhook URL in Retell

- [ ] In Retell dashboard (or via API): set the inbound webhook URL on the agent or phone number to `https://zmilcwstkgwxqxqcuiij.supabase.co/functions/v1/inbound-webhook`
- [ ] Test: call the number → webhook fires → dynamic variables populate → agent uses tenant config

**Acceptance Criteria:**

- [ ] Inbound call on any imported number triggers the webhook
- [ ] Webhook returns correct tenant config as dynamic variables
- [ ] Agent greets caller with the tenant's welcome message and company name
- [ ] `check_reservation` receives the tenant's `user_id` and returns only their data
- [ ] Unknown numbers get a sensible fallback (not a crash)
- [ ] Response time < 2 seconds

---

## Phase 5B: Time-Based Call Routing (SIP / Carrier Level)

**Critical context:** Most operators don't want Retell answering calls during business hours — their dispatchers handle those. Retell should only receive calls after hours (or on overflow). This routing logic lives at the SIP trunk / carrier level, NOT in Retell.

### 5B.1 Routing Strategy by Forwarding Mode

The `forwarding_mode` in `tenant_config` determines how the SIP trunk is configured:

| Mode | SIP / Carrier Config | When Retell answers |
|------|---------------------|---------------------|
| `always` | All inbound calls forward to Retell SIP endpoint | Every call, 24/7 |
| `after_hours` | Time-of-day routing at the carrier: during `business_hours_start`–`business_hours_end` → ring office. Outside → forward to Retell SIP. | Only outside business hours |
| `overflow` | Ring office first; after X seconds no answer → forward to Retell SIP. | When office doesn't pick up |

**Implementation options per provider:**

- **Twilio:** Use Studio Flows or TwiML `<Dial>` with time-of-day logic. On timeout/after-hours, forward via SIP to Retell. Twilio supports time-based routing natively.
- **Telnyx:** Call Control API or SIP trunking with conditional forwarding rules.
- **Carrier-level (non-VoIP):** Some carriers support "time-of-day" call forwarding (e.g. Verizon Business, AT&T). The operator configures this with their carrier directly — forward to a Twilio/Telnyx number that SIP trunks to Retell.

### 5B.2 Setup Assistance in Dashboard

- [ ] When operator selects `forwarding_mode = 'after_hours'` or `'overflow'`, show provider-specific setup instructions:
  - If Twilio: display the Twilio Studio Flow template or TwiML needed
  - If Telnyx: display the call control forwarding rule
  - If carrier: display generic "contact your phone provider and set up conditional forwarding to [Retell number]"
- [ ] Include the operator's configured business hours and timezone in the instructions
- [ ] Optional future: auto-configure Twilio routing via API (Phase 9 scope)

### 5B.3 Edge Case: Retell Receives a Call During Business Hours

- [ ] Even with time-based routing, edge cases happen (carrier delay, timezone drift, etc.)
- [ ] The inbound webhook already passes `business_hours_start/end` and `timezone` to the agent
- [ ] Agent prompt should handle this: "If the call is within business hours, let the caller know the office should be open and offer to take a message or transfer to the office line"
- [ ] The `office_phone` from `tenant_config` can be used as a transfer target during hours (agent transfers caller back to the office)

**Acceptance Criteria:**

- [ ] Dashboard shows clear forwarding setup instructions based on provider and mode
- [ ] After-hours mode: calls during business hours ring the office, after hours go to Retell
- [ ] Overflow mode: calls ring the office first, then fall back to Retell on no answer
- [ ] Agent gracefully handles edge cases (call arrives during business hours)
- [ ] Sun City Express existing number (+15206800005) continues working as before

---

## Phase 6: Phone Number Management

Allow operators to connect their existing phone number via the dashboard.

### 6.1 Phone Number Setup in Dashboard

- [ ] New section in `AgentSettings.tsx`: "Phone Number"
  - Input for phone number (E.164 format)
  - Input for SIP trunk termination URI
  - Optional: SIP username/password
  - "Connect Number" button
- [ ] On submit: save to `tenant_config`, then call our backend to import the number into Retell

### 6.2 Retell Phone Number Import API Integration

- [ ] New edge function (or extend existing): `manage-phone-number`
  - Accepts phone number + SIP trunk details from dashboard
  - Calls Retell API `POST /import-phone-number` with:
    - `phone_number` (E.164)
    - `termination_uri` (from SIP trunk)
    - `sip_trunk_auth_username` / `sip_trunk_auth_password`
    - `inbound_agent_id` (our shared agent ID, stored as env var)
    - Inbound webhook URL
  - Stores returned `phone_number_sid` in `tenant_config.retell_phone_number_id`
  - Returns success/error to dashboard
- [ ] Store Retell API key as edge function secret (`RETELL_API_KEY`)

### 6.3 Phone Number Verification / Status

- [ ] After import: show status in dashboard (connected / pending / error)
- [ ] Optional: "Test Call" button or instructions for operator to call their number and verify

**Acceptance Criteria:**

- [ ] Operator enters their number + SIP details → number is imported into Retell
- [ ] Number is bound to the shared agent with the inbound webhook
- [ ] Calls to that number route through Retell → webhook → tenant config → agent
- [ ] Dashboard shows connection status
- [ ] SIP credentials are stored securely (not exposed in client)

---

## Phase 7: Onboarding Flow (New User Sign-Up)

Streamline what happens when a new transportation company signs up.

### 7.1 Post-Registration Setup Wizard

- [ ] After sign-up + email verification, redirect to a setup flow:
  1. Company name (required)
  2. Welcome message (with a sensible default template)
  3. Business hours (default 6 AM - 10 PM)
  4. Phone number setup (optional — can skip and do later)
- [ ] Creates `tenant_config` row on completion
- [ ] Redirects to main dashboard

### 7.2 Default Prompt Template

- [ ] Provide a well-crafted default welcome message template:
  `"Thank you for calling {{company_name}} reservations and after-hours helpline. Please have your confirmation number or the phone number associated with your reservation ready."`
- [ ] Operator can customize from Agent Settings tab

**Acceptance Criteria:**

- [ ] New user can go from sign-up to working voice agent in < 10 minutes (excluding SIP setup)
- [ ] Sensible defaults mean the agent works even with minimal configuration
- [ ] Operator can refine settings later from the dashboard

---

## Phase 8: Polish and Production Hardening

### 8.1 Webhook Security

- [ ] Validate Retell's webhook signature on the inbound webhook (or use IP allowlisting)
- [ ] Rate limiting on phone number import endpoint

### 8.2 SIP Credential Security

- [ ] Move SIP passwords to Supabase Vault (or encrypt at rest)
- [ ] Never return SIP password to the frontend after initial save

### 8.3 Retell API Error Handling

- [ ] Handle Retell API failures gracefully (number import fails, rate limits, etc.)
- [ ] Retry logic or clear error messages for the operator
- [ ] Monitor Retell API usage per tenant

### 8.4 Tenant Deletion / Offboarding

- [ ] When a tenant cancels: remove their phone number from Retell (`DELETE /delete-phone-number`)
- [ ] Clean up `tenant_config` row
- [ ] Reservation/driver data retention policy

**Acceptance Criteria:**

- [ ] Webhook rejects unsigned/malicious requests
- [ ] SIP credentials are not exposed
- [ ] Retell API errors surface clearly to operators
- [ ] Clean offboarding doesn't leave orphaned resources

---

## Phase 9: Deferred (Post-Launch)

### 9.1 Per-Tenant Agent Override (Option B Migration)

- [ ] If a tenant needs fundamentally different agent logic (not just text changes): use Retell API to create a dedicated agent for them
- [ ] Store `retell_agent_id` in `tenant_config`
- [ ] Inbound webhook returns `override_agent_id` instead of dynamic variables
- [ ] Only for edge cases — most tenants stay on shared agent

### 9.2 Multiple Phone Numbers Per Tenant

- [ ] Some companies may have multiple numbers (main line + after-hours + satellite office)
- [ ] Move phone numbers to a separate `tenant_phone_numbers` table (one-to-many)
- [ ] Each number can be imported independently

### 9.3 Call Analytics Per Tenant

- [ ] Track call volume, transfer success rate, average call duration per tenant
- [ ] Dashboard analytics tab
- [ ] Retell post-call webhook → store call metadata in `call_logs` table

### 9.4 Billing Integration

- [ ] Track per-tenant Retell minutes usage
- [ ] Integrate with Stripe or similar for subscription billing
- [ ] Usage-based pricing for voice minutes

---

## Implementation Order & Dependencies

```
Phase 1 (tenant_config table + seed Sun City Express)
  ├── Phase 2 (dashboard settings tab) — can build in parallel with 3
  └── Phase 3 (templatize Retell prompt)
        └── Phase 4 (tenant-scoped edge functions)
              └── Phase 5 (inbound webhook) — the big unlock
              └── Phase 5B (time-based call routing at SIP/carrier level)
                    └── Phase 6 (phone number management via dashboard)
                          └── Phase 7 (onboarding flow)
                                └── Phase 8 (hardening)
```

Phases 1-5 are the critical path to multi-tenant. Phase 5B handles per-tenant business-hours routing (SIP config, not Retell). Phase 6 enables self-service number onboarding. Phases 7-8 are polish for production. Phase 9 is post-launch.

---

## Launch Checklist (Second Operator Demo)

**Existing tenant (Sun City Express, +15206800005):**

- [ ] Agent still works identically to today — no regression
- [ ] `tenant_config` row exists with correct company name, hours, phone number
- [ ] Dashboard "Agent Settings" tab shows and saves all fields
- [ ] Calls after hours → Retell answers with Sun City Express greeting
- [ ] Calls during hours → ring office as before (SIP/carrier routing unchanged)

**Multi-tenant infrastructure:**

- [ ] `tenant_config` table exists with RLS
- [ ] Retell agent prompt uses dynamic variables (no hard-coded company name)
- [ ] `check_reservation` filters by `user_id` — tenants only see their own data
- [ ] Inbound webhook deployed and registered with Retell
- [ ] At least two phone numbers imported into Retell, each routing to correct tenant

**Second tenant verification:**

- [ ] Tenant B config saved via dashboard (company name, hours, welcome message, forwarding mode)
- [ ] Tenant B phone number imported into Retell and bound to shared agent
- [ ] End-to-end test: call Tenant B's number → hears Tenant B's greeting → only sees Tenant B's reservations
- [ ] End-to-end test: call Tenant A's number → hears Tenant A's greeting → only sees Tenant A's reservations
- [ ] No cross-tenant data leakage
- [ ] Operator can change welcome message → next call uses updated greeting
- [ ] Time-based routing works per tenant's forwarding mode and business hours
