# Retell AI Agent: Reservation Time & After-Hours Check

This agent uses a **custom function** that calls your Supabase Edge Function to look up reservations. The function returns **structured data** (pickup/dropoff times, addresses, flight info, etc.) so the agent can answer follow-up questions dynamically (e.g. “What’s my drop-off location?”, “What time is pickup?”).

---

## Exactly what to do in Retell (step-by-step)

1. **Open the Retell dashboard**  
   Go to [dashboard.retellai.com](https://dashboard.retellai.com) and open your **Reservation & After-Hours Check** agent (or create a new agent and name it that).

2. **Add or edit the custom function**  
   - In the right-hand panel, find **Functions** and click **+ Add** (or click the existing `check_reservation` to edit).  
   - Choose **Custom Function**.  
   - Set **Name** to: `check_reservation`  
   - Set **Description** to:  
     `Look up a reservation by confirmation number or phone number. Returns full details (pickup/dropoff times, addresses, flight info, etc.) so you can answer any follow-up questions the caller asks.`  
   - Set **API Endpoint** to **POST** and **URL** to:  
     `https://zmilcwstkgwxqxqcuiij.supabase.co/functions/v1/check-reservation`  
   - **Timeout:** leave default or set to `120000`.  
   - Under **Parameters (Optional)** → JSON, paste this (with **Payload: args only** turned **ON**):
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
   - **Speak After Execution:** turn **ON** (so the agent says the result).  
   - Click **Update** or **Save**.

3. **Set the agent prompt**  
   - In the left panel, find the main **Prompt** / **Agent instructions** text box.  
   - Select all existing text and replace it with the prompt in **Section 4** below (the block that starts with "You are a reservation and after-hours helpline...").  
   - Change "a transportation company" to your company name (e.g. "Sun City Express") if you want.  
   - Save (e.g. **Publish** or wait for auto-save).

4. **Set the opening / first message (intro)**  
   - In Retell, find the **First message**, **Opening statement**, or **Welcome message** field (where the agent’s first words are set).  
   - Use this order: thank-you and company name → ask them to have confirmation or phone ready and say you can look up by that → then end with the open question.  
   - Paste this exactly (replace "Sun City Express" if needed):

   **Opening message to paste:**
   ```text
   Thank you for calling Sun City Express reservations and after-hours helpline. Please have your confirmation number or phone number handy—if you provide it, I can look up your reservation. How can I assist you with your reservation today?
   ```

   That way "How can I assist you?" is the last thing they hear, not the first.

5. **Test**  
   Use **Test your agent** (e.g. Test Chat or Test Audio) and say:  
   *"Check my reservation, confirmation number RES-20241026-0001."*  
   Then ask: *"What's my drop-off address?"* or *"What time is pickup?"* to confirm follow-ups work.  
   Check that the intro plays in the right order: thank-you → have confirmation/phone ready → "How can I assist you with your reservation today?" last.

That's it. The Supabase Edge Function is already deployed; you only configure the agent and custom function in Retell as above.

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
| **Description** | Look up a reservation by confirmation number or phone number. Returns full details (pickup/dropoff times, addresses, flight info, etc.) so you can answer any follow-up questions the caller asks. |
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

5. **Optional – Store Fields as Variables:** In the custom function config, under **Store Fields as Variables**, you can map the JSON response to variables so the agent can reference them (e.g. in prompts). Add key-value pairs such as:

   | Variable name   | Response path (JSON path into the tool response) |
   |-----------------|---------------------------------------------------|
   | `pickup_time`   | `pickup_time`                                     |
   | `dropoff_time`  | `dropoff_time`                                    |
   | `pickup_address`| `pickup_address`                                  |
   | `dropoff_location` | `dropoff_location`                             |
   | `flight_info`   | `flight_info`                                     |
   | `summary`       | `summary`                                         |

   Exact path syntax depends on Retell’s UI (e.g. `summary` or `response.summary`). If you don’t use variables, the agent still receives the full JSON in the function result and can answer from that.

6. **Optional**: Under **Request headers**, add:
   - `Authorization`: `Bearer <SUPABASE_ANON_OR_SERVICE_ROLE_KEY>`  
   if you want to require a key (Edge Functions can also use the project’s built-in auth).

7. Save the custom function.

---

## 4. Agent Prompt (Reservation & After-Hours)

Paste this into the agent’s **Prompt** so the agent knows when to call the tool and how to answer follow-ups (pickup/dropoff times, addresses, etc.):

```text
You are a reservation and after-hours helpline for a transportation company. Callers may ask about their reservation, pickup/dropoff times, addresses, flight info, or whether their pickup is during or after business hours.

Your capabilities:
- Look up a reservation by confirmation number (e.g. RES-20241026-0001) or by the phone number they’re calling from.
- After looking up, you receive full reservation details. Use that data to answer any follow-up the caller asks: pickup time, dropoff time, pickup address, drop-off location, flight time and flight info, terminal, direction, special instructions, vehicle or driver, number of passengers, and whether the pickup is within business hours (6 AM to 10 PM) or after hours.

Process:
1. If they give a confirmation number, call the check_reservation function with confirmation_number set to that value.
2. If they don’t have a confirmation number, call check_reservation with telephone set to their number (if you have it) or ask for their phone number and then call the function.
3. After the function returns, use the "summary" for a brief first response. For follow-ups (e.g. "What’s my drop-off?", "What time is pickup?", "Where do you pick me up?"), answer from the returned data: pickup_time, dropoff_time, pickup_address, dropoff_location, flight_info, special_instructions, terminal, etc. If a field is null or missing, say you don’t have that detail.
4. If the pickup is outside business hours, mention they can leave a message or call back during business hours.

Business hours: 6:00 AM to 10:00 PM.

Keep responses brief and clear. If no reservation is found (found: false in the response), say so and suggest they check the confirmation number or call back during business hours.
```

Adjust company name or hours in the prompt if needed.

---

## 5. Optional: Verify Requests from Retell

To ensure only Retell can call your Edge Function, you can verify the **X-Retell-Signature** header using your Retell API key. The Edge Function currently does not verify this; add verification in the function if you want it (see [Retell: Verifying request is from Retell](https://docs.retellai.com/build/single-multi-prompt/custom-function#verifying-request-is-from-retell)).

---

## 6. Response shape (Edge Function)

On success the function returns JSON like:

- `found`: true/false  
- `summary`: Short sentence for the agent to say first (pickup time, business hours, status).  
- When `found` is true: `pickup_time`, `dropoff_time`, `flight_time`, `pickup_address`, `dropoff_location`, `terminal`, `direction`, `flight_info`, `special_instructions`, `vehicle_assigned`, `driver_assigned`, `num_passengers`, `within_business_hours`, `business_hours_note`, plus `confirmation_number`, `customer_name`, `status`.

When `found` is false, the response includes `message` or `error` for the agent to speak.

## 7. Summary

| Item | Value |
|------|--------|
| Edge Function | `check-reservation` (Supabase) |
| Retell custom function name | `check_reservation` |
| Parameters | `confirmation_number` (string), `telephone` (string) |
| Response | JSON with full reservation fields + `summary` for dynamic Q&A |
| Business hours in logic | 6:00 AM – 10:00 PM (configurable in `index.ts`: `BUSINESS_HOURS_START`, `BUSINESS_HOURS_END`) |

After saving the agent and deploying the Edge Function, test by calling and saying e.g. “Check my reservation RES-20241026-0001,” then “What’s my drop-off address?” or “What time is pickup?”
