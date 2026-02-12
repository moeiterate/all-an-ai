# Voice Line Plan — Plain English + Supabase Reality Check

## What we're trying to do

You have a **Retell voice agent** that answers after-hours calls. Callers give a confirmation number or phone number; the agent looks up their reservation in Supabase and answers questions (pickup time, address, etc.).

**Goal:** Add the ability to **transfer the call to the driver** when the caller asks. So: lookup reservation → confirm it’s them → “I can connect you to your driver” → Retell transfers the call to the driver’s phone number.

That driver phone number has to come from **your data in Supabase**. So we need:

1. **Data model** — Reservations (you already use this) plus a way to get a **driver phone** for each reservation (column on `reservations` or a `drivers` table with phone).
2. **“Import fresh”** — A way to know “this data was updated recently” so we only offer transfer when it’s current (e.g. last import &lt; 24 hours). That’s the `import_meta` table.
3. **Transfer in Retell** — Use Retell’s transfer feature to dial the driver phone we get from Supabase.
4. **Later** — Operator upload/CLI to import CSV/Excel into the same tables (that’s the “standardized import” we pushed to the last phase).

So: **transfer first** (using whatever data is already in Supabase or you add by hand), **standardized CSV/Excel import last**.

---

## What Supabase has vs doesn’t have (and MCP)

- Your app talks to **one Supabase project**. From `.env.local` that project is:
  - **URL:** `https://zmilcwstkgwxqxqcuiij.supabase.co`
  - **Project ref:** `zmilcwstkgwxqxqcuiij`

- **MCP:** The Supabase MCP was run; it only listed projects in *your* linked account (e.g. BudgetApp, QuoteToPDFMVP). The project `zmilcwstkgwxqxqcuiij` **did not appear**. So either:
  - That project is in a different Supabase org (e.g. a client’s or teammate’s), or
  - You’re not logged into the account that owns it in the MCP.

So we **cannot** use the MCP to list tables for the Voice Line project until that project is in an account you’ve linked to Cursor.

### What you need to do to see what’s in Supabase

**Option A — You have dashboard access to that project**

1. Go to [Supabase Dashboard](https://supabase.com/dashboard) and open the project whose URL is `zmilcwstkgwxqxqcuiij` (or the one that matches your `SUPABASE_URL` in `.env.local`).
2. **Table Editor:** See which tables exist: at minimum we care about `reservations`; the plan also expects `import_meta` (we’ll add) and some form of driver phone (see below).
3. **SQL Editor:** Run:
   - `SELECT column_name, data_type FROM information_schema.columns WHERE table_schema = 'public' AND table_name = 'reservations' ORDER BY ordinal_position;`
   - If you have a `drivers` table: same for `table_name = 'drivers'`.

That tells you exactly what columns exist.

**Option B — You want MCP to work**

1. Log in to Supabase (in the account that **owns** the Voice Line project).
2. In Cursor, ensure the Supabase MCP is configured to use that account (e.g. same Supabase login / access token that can see project `zmilcwstkgwxqxqcuiij`).
3. Once that project shows up in `list_projects`, you can use `list_tables` for that project to see tables and columns without using the dashboard.

**If you don’t own the project**

- You need the project owner (or someone with SQL/Table Editor access) to run the same SQL above and send you:
  - List of tables in `public`.
  - For `reservations`: column names and types.
  - If present: `drivers` (and any other driver-related table) column names and types.
- Then we can align the plan and code (e.g. add `driver_phone` or use `drivers.phone` and a join).

---

## What the code assumes today

From this repo:

| Thing | Where it’s used | What’s assumed |
|-------|------------------|----------------|
| **reservations** | `check-reservation` Edge Function, `migrate_csvs.py` | Table exists. Columns: `confirmation_number`, `customer_name`, `telephone`, `pickup_time`, `dropoff_time`, `flight_time`, `pickup_address`, `dropoff_info`, `terminal`, `direction`, `flight_info`, `special_instructions`, `vehicle_assigned`, `driver_assigned`, `num_passengers`, `status`. |
| **Driver phone** | Not used yet | Needed for transfer. Either: `reservations.driver_phone`, or a `drivers` table with e.g. `id`, `name`, `phone` and a link from reservation to driver (e.g. `reservations.driver_id` or match on `driver_assigned` → driver name). |
| **drivers** | `migrate_csvs.py` | Script inserts into `drivers` with `id`, `name`, `available_days`. **No phone column** in the script — so either the table already has a `phone` column we’re not filling, or we add it and backfill. |
| **import_meta** | Plan only | Doesn’t exist yet. Phase 1 adds it: `id`, `source`, `last_import_at`, `row_count`, `created_at` (and optional `file_name` / `checksum`). |

So: **Supabase currently has (or is assumed to have):** `reservations` and possibly `drivers` / `vehicles`. It **does not** have `import_meta`, and we still need a **driver phone** source for transfer.

---

## Summary

- **Plan in one sentence:** Add call transfer to driver using Supabase data; add schema for “import fresh” and driver phone; implement transfer in Retell; do standardized CSV/Excel import last.
- **Supabase:** The app’s project (`zmilcwstkgwxqxqcuiij`) wasn’t visible to the MCP, so we can’t list its tables from here. Use the dashboard (or owner-run SQL) to confirm `reservations` (and optionally `drivers`) and whether driver phone exists; then we add whatever’s missing (`import_meta`, `driver_phone` or `drivers.phone`).
- **If you don’t own the project:** Get table/column list from the owner and we can still design the migrations and code against that.
