# Retell AI Agent: Reservation Time & After-Hours Check

This agent uses a **custom function** that calls your Supabase Edge Function to look up reservations and report whether the pickup time is inside or outside business hours.

---

## 1. Deploy the Edge Function (Supabase)

The function lives at `supabase/functions/check-reservation/index.ts`.

**Option A – Supabase Dashboard**

1. In [Supabase Dashboard](https://supabase.com/dashboard) → your project → **Edge Functions**.
2. Create a new function named `check-reservation`.
3. Paste in the contents of `supabase/functions/check-reservation/index.ts`.
4. Deploy. The URL will be:  
   `https://<project-ref>.supabase.co/functions/v1/check-reservation`

**Option B – Supabase CLI**

```bash
cd /path/to/voiceLineMVP
npx supabase login
npx supabase link --project-ref zmilcwstkgwxqxqcuiij
npx supabase functions deploy check-reservation
```

After deploy, copy the function URL (e.g.  
`https://zmilcwstkgwxqxqcuiij.supabase.co/functions/v1/check-reservation`).

---

## 2. Create the Agent in Retell

1. Go to [Retell Dashboard](https://dashboard.retellai.com) → **Agents** → **Create Agent**.
2. Choose **Single-prompt** (or multi-prompt if you prefer).
3. **Basic settings**: Name (e.g. "Reservation & After-Hours Check"), voice, language.

---

## 3. Add the Custom Function in Retell

1. In the agent editor, open **Step 3: Add function calling** (or **Tools**).
2. Click **+ Add** → **Custom Function**.
3. Configure:

| Field | Value |
|--------|--------|
| **Name** | `check_reservation` |
| **Description** | Look up a reservation by confirmation number or phone number and check if the pickup time is within or outside business hours (6 AM–10 PM). |
| **HTTP Method** | **POST** |
| **Endpoint URL** | Your Edge Function URL, e.g.  
  `https://zmilcwstkgwxqxqcuiij.supabase.co/functions/v1/check-reservation` |

4. **Parameters** (JSON schema for POST body):

```json
{
  "type": "object",
  "properties": {
    "confirmation_number": {
      "type": "string",
      "description": "Reservation confirmation number, e.g. RES-20241026-0001"
    },
    "telephone": {
      "type": "string",
      "description": "Customer phone number to look up the reservation"
    }
  }
}
```

(At least one of `confirmation_number` or `telephone` should be provided; the LLM will fill these from the conversation.)

5. **Optional**: Under **Request headers**, add:
   - `Authorization`: `Bearer <SUPABASE_ANON_OR_SERVICE_ROLE_KEY>`  
   if you want to require a key (Edge Functions can also use the project’s built-in auth).

6. Save the custom function.

---

## 4. Agent Prompt (Reservation & After-Hours)

Paste this into the agent’s **Prompt** so the agent knows when and how to call the tool:

```text
You are a reservation and after-hours helpline for a transportation company. Callers may ask about their reservation or whether their pickup time is during or after business hours.

Your capabilities:
- Look up a reservation by confirmation number (e.g. RES-20241026-0001) or by the phone number they’re calling from.
- Tell them whether their pickup time is within business hours (6 AM to 10 PM) or after hours.

Process:
1. If they give a confirmation number, call the check_reservation function with confirmation_number set to that value.
2. If they don’t have a confirmation number but are calling from their phone, call check_reservation with telephone set to their number (if you have it) or ask for their phone number and then call the function.
3. After calling the function, relay the result in a short, friendly way: confirm the reservation, the pickup time, and whether it’s within or outside business hours. If it’s after hours, tell them they can leave a message or call back during business hours.

Business hours: 6:00 AM to 10:00 PM.

Keep responses brief and clear. If no reservation is found, say so and suggest they check the confirmation number or call back during business hours.
```

Adjust company name or hours in the prompt if needed.

---

## 5. Optional: Verify Requests from Retell

To ensure only Retell can call your Edge Function, you can verify the **X-Retell-Signature** header using your Retell API key. The Edge Function currently does not verify this; add verification in the function if you want it (see [Retell: Verifying request is from Retell](https://docs.retellai.com/build/single-multi-prompt/custom-function#verifying-request-is-from-retell)).

---

## 6. Summary

| Item | Value |
|------|--------|
| Edge Function | `check-reservation` (Supabase) |
| Retell custom function name | `check_reservation` |
| Parameters | `confirmation_number` (string), `telephone` (string) |
| Business hours in logic | 6:00 AM – 10:00 PM (configurable in `index.ts`: `BUSINESS_HOURS_START`, `BUSINESS_HOURS_END`) |

After saving the agent and deploying the Edge Function, test by calling the Retell number and saying e.g. “I’d like to check my reservation, confirmation number RES-20241026-0001” or “Can you look up my reservation by my phone number?”
