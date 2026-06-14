# HVAC AI Inbound Call Handler — Monitoring Guide

**Scenario:** HVAC AI Inbound Call Handler — Alex Voice Agent  
**Last Audited:** 2026-06-14  
**Bugs Found:** 24  
**Zone:** us2.make.com  

---

## Section 1: Module Health Checklist

| Module | Name | Healthy Run | Failure Signs |
|--------|------|-------------|---------------|
| 1 | twilio:WatchCalls | Triggers once per completed call; From/To show E.164 numbers (e.g. +14155551234) | Triggers 4-5 times per call (status filter was empty — original bug); never triggers (wrong To number — missing + prefix); 1.From shows raw number without + |
| 2 | Set Call Timestamp | call_timestamp set to current date/time string like "06/14/2026 14:32:10" | Empty call_timestamp seen downstream; was ParseJSON in original (bug) |
| 3 | Set caller_phone | caller_phone = caller's E.164 number from {{1.From}} | caller_phone contains instruction text (original bug: value was a UI walkthrough string) |
| 4 | Vapi — Launch Alex | Returns 200 with call id (UUID), status="queued"; parseResponse=true gives structured fields | 401 Unauthorized (bad/empty API key — original bug used {{connection.vapiApiKey}} which doesn't exist); 422 Unprocessable Entity (phoneNumberId was Twilio number string instead of Vapi UUID — original bug) |
| 5 | Parse Vapi Call Response | id field populated (UUID string), status = "queued" or "ringing" | Empty/null id — means Vapi call failed; all downstream routes go to Route 2 or 3 |
| 6 | Set gpt_prompt | gpt_prompt variable populated with caller reference string | Original bug: leading space in variable name made it " gpt_prompt" — unreferenceable |
| 7 | Router | Exactly ONE route executes per call (Route 1 if Vapi succeeded, Route 2 if Vapi failed, Route 3 as fallback) | ALL routes execute simultaneously — catastrophic original bug when no filter conditions were set |
| 22 | Filter: Vapi Callback Succeeded | Passes through when {{5.id}} is not empty (Vapi call was initiated) | Blocks correctly when Vapi failed — this is expected behavior, not an error |
| 8 | Set call_status (qualified) | call_status = "qualified" for calls entering Route 1 | — |
| 9 | OpenAI Summarize Call | Returns valid JSON with caller_name, issue_type, address, urgency, preferred_time, summary fields | 404 model not found (original bug: gpt-5.5 doesn't exist, fixed to gpt-4o); {{transcript}} undefined (original bug, fixed to {{5.artifact.transcript}}); JSON parse error in module 24 if OpenAI wraps output in markdown code fences |
| 24 | Parse AI Call Summary | caller_name, issue_type, address, urgency, summary all populated as discrete fields | ParseJSON fails if OpenAI returns markdown-wrapped JSON (```json ... ```) — add "Do not use markdown" to prompt if this occurs |
| 10 | Set call_status (booked) | call_status = "booked" | Original bug: was set to "missed" inside the qualified route; also had leading space in name |
| 11 | Google Calendar | Creates event for next day with caller phone, issue type, address, urgency in description; 2-hour time block | Original bug: "quick" mode created all-day event with no time; fixed to "detail" mode with startDate/endDate |
| 12 | SMS to Caller (confirm) | Caller receives confirmation SMS with their name (or "there" as fallback) | Original bug: {{caller_name}} was undefined — SMS went out as "Hi , your HVAC..."; fixed to {{ifempty(24.caller_name; "there")}} |
| 13 | SMS to Owner (alert) | Owner's phone (+17148775606) receives new lead alert | Original bug: to field was {{1.From}} — alert went to the CALLER, not the owner; fixed to hardcoded +17148775606 |
| 14 | Sheets — Log Booked | Row added with date, phone, name, issue, address, urgency all populated | Original bugs: empty date ({{3.call_timestamp}} → fixed to {{2.call_timestamp}}); empty data columns ({{6.xxx}} → fixed to {{24.xxx}}); Caller Phone column missing (fixed) |
| 23 | Filter: Vapi Call Failed | Passes through when {{5.id}} is empty (Vapi did not return a call ID) | — |
| 15 | Set call_status (unqualified) | call_status = "unqualified" for Route 2 | — |
| 16 | SMS to Caller (missed) | Caller receives "sorry we missed you" SMS from (949) 379-2748 | — |
| 17 | SMS to Owner (missed) | Owner (+17148775606) receives missed call alert with caller number | — |
| 18 | Sheets — Log Missed | Row added with date and phone; Caller Name/Issue/Address/Urgency = N/A | Original bugs: empty date (fixed); Caller Phone column missing (fixed) |
| 19 | Set call_status (fallback) | call_status = "missed-fallback" | Original bug: was set to "qualified" in a route that sends "sorry we missed you" SMS — mislabeled |
| 20 | SMS to Caller (fallback) | Caller receives fallback "sorry we missed you" SMS | — |
| 21 | Sheets — Log Fallback | Row added with date and phone; all data fields hardcoded to "Unknown"/"Not captured" | Original bugs: empty date (fixed); broken variable references to {{6.xxx}} (fixed to hardcoded defaults); Caller Phone missing (fixed) |

---

## Section 2: Common Failure Points and How to Diagnose in Make

### Scenario never triggers (phone number format bug — FIXED)

**Original symptom:** Scenario runs but never matches any inbound calls.  
**Root cause:** Module 1 had `"to": "19493792748"` without the `+` prefix. Twilio delivers phone numbers in E.164 format (+19493792748). The filter never matched.  
**In Make execution log:** Scenario shows 0 records processed, or executes with 0 bundles output from Module 1.  
**Fixed value:** `"to": "+19493792748"`  
**How to verify:** In Make, go to the scenario and inspect Module 1's output bundle. The `To` field should show `+19493792748`.

---

### Duplicate runs per call (status filter — FIXED)

**Original symptom:** Every incoming call produces 4-5 scenario executions. Callers receive multiple duplicate SMS messages. Multiple Google Calendar events created per call. Google Sheets has duplicate rows.  
**Root cause:** Module 1 had `"status": ""` (empty). Twilio fires status callbacks for queued, ringing, in-progress, and completed states. With no filter, all fire the scenario.  
**In Make execution log:** You see 4-5 runs within 30-60 seconds with the same `1.CallSid` value across all of them.  
**Fixed value:** `"status": "completed"` — only processes after the call finishes.  
**How to verify:** Check execution history. Each unique CallSid should appear exactly once.

---

### Module 2 ParseJSON failure (wrong data source — FIXED)

**Original symptom:** Module 2 turns orange/red or passes through empty data. All downstream modules receive null for any field from Module 2.  
**Root cause:** Original Module 2 was `json:ParseJSON` parsing `{{1.body}}`. The `twilio:WatchCalls` trigger returns structured fields directly (1.From, 1.To, 1.Status, etc.) — there is no `1.body` field. The ParseJSON received null/empty string.  
**Fixed:** Module 2 is now `util:SetVariable2` that sets `call_timestamp` using `formatDate(now; "MM/DD/YYYY HH:mm:ss")`.

---

### Vapi 401 Unauthorized

**Symptom:** Module 4 turns red. Output shows HTTP 401 response.  
**Root cause:** Original bug used `{{connection.vapiApiKey}}` which is not a valid Make variable. The Authorization header sent an empty bearer token.  
**In Make execution log:** Module 4 output bundle shows `statusCode: 401`, `data: {"message": "Unauthorized"}`.  
**Fix:** Replace `YOUR_VAPI_API_KEY_HERE` in Module 4's Authorization header with your actual Vapi API key from app.vapi.ai > Account > API Keys.  
**Note:** After fixing, Module 4 will return a 200 with a call object containing `id`, `status`, `phoneNumberId`.

---

### Vapi 422 Unprocessable Entity (bad phoneNumberId)

**Symptom:** Module 4 turns red. HTTP 422 response. Call never initiates.  
**Root cause:** Original bug passed `{{1.To}}` (the Twilio phone number string "+19493792748") as the `phoneNumberId`. The Vapi API expects a UUID of the phone number object configured in your Vapi dashboard, not the raw Twilio number.  
**In Make execution log:** Module 4 output shows `statusCode: 422`, body contains error about invalid phoneNumberId format.  
**Fix:** Replace `YOUR_VAPI_PHONE_NUMBER_UUID_HERE` in Module 4's request body with the UUID from Vapi Dashboard > Phone Numbers > (your number) > copy the ID field (looks like: abc12345-1234-1234-1234-abcdef123456).  
**Also fixed:** Added required `customer.number` field pointing to `{{1.From}}` so Vapi knows which number to call back.

---

### OpenAI 404 model not found (gpt-5.5 — FIXED)

**Symptom:** Module 9 turns red. OpenAI returns 404 error on every call.  
**Root cause:** Original blueprint specified `"model": "gpt-5.5"` which does not exist in OpenAI's API.  
**In Make execution log:** Module 9 shows `statusCode: 404`, body: `{"error": {"code": "model_not_found", "message": "..."}}`.  
**Fixed value:** `"model": "gpt-4o"` — a valid, production-ready model.  
**Alternative valid models:** `gpt-4o-mini` (cheaper, faster), `gpt-4-turbo`.

---

### OpenAI returns markdown-wrapped JSON (breaks Module 24 ParseJSON)

**Symptom:** Module 24 turns red with a JSON parse error. Module 9 shows a successful response but the content starts with ` ```json `.  
**Root cause:** OpenAI sometimes wraps JSON output in markdown code fences despite being instructed not to. The prompt in the fixed blueprint says "no markdown, no code fences" but this is not 100% reliable.  
**In Make execution log:** Module 9 output — `choices[1].message.content` starts with ` ``` ` instead of `{`.  
**Fix options:**  
1. Change Module 9's `response_format` from `"text"` to `"json_object"` — this enforces raw JSON output (requires the word "json" in the prompt, which is already present).  
2. Add a Module between 9 and 24: `text:Replace` to strip ` ```json ` prefix and ` ``` ` suffix.  
3. Accept occasional failures and check weekly.

---

### Google Calendar creates wrong event type (FIXED)

**Original symptom:** Google Calendar event is created as an all-day event with no specific time.  
**Root cause:** Original Module 11 used `"select": "quick"` mode with only an event title. Google's natural language parser can't extract a time from "HVAC Service Call — +19493792748".  
**Fixed:** Changed to `"select": "detail"` mode with explicit `startDate` (tomorrow), `endDate` (tomorrow +2 hours), `summary`, and `description` fields.  
**In Make execution log:** Module 11 output bundle will include `start.dateTime` and `end.dateTime` fields when working correctly.

---

### Twilio SMS delivers to wrong recipient (Module 13 — FIXED)

**Original symptom:** The HVAC owner receives a confirmation SMS meant for the caller. The caller receives the owner's "New HVAC Lead" alert. No one gets the right message.  
**Root cause:** Module 13's `to` field was `{{1.From}}` (the caller's number) instead of the owner's number. Module 17 in Route 2 correctly hardcoded `+17148775606`.  
**Fixed:** Module 13 `to` is now `+17148775606` to match Module 17.

---

### Google Sheets rows have empty columns (FIXED)

**Original symptom:** Rows appear in Google Sheets but Date (A) is empty and most data columns (C-F) are empty.  
**Root cause (Date):** All three sheet modules referenced `{{3.call_timestamp}}`. Module 3 is the `caller_phone` SetVariable2 — it outputs `caller_phone`, NOT `call_timestamp`. Fixed to `{{2.call_timestamp}}` (Module 2 in the fixed blueprint sets the timestamp).  
**Root cause (Data columns):** Modules 14 and 21 referenced `{{6.caller_name_final}}`, `{{6.issue_type}}`, etc. Module 6 is a single-variable SetVariable2 that only outputs `gpt_prompt`. None of those field names exist on its output. Fixed to reference `{{24.xxx}}` (Module 24 is the ParseJSON of OpenAI output).  
**Root cause (Caller Phone):** All three sheet modules omitted the `Caller Phone` (column B) field entirely. Fixed to include `"Caller Phone": "{{1.From}}"` in all three mappers.

---

### Router sends to all routes simultaneously (FIXED)

**Original symptom:** Every call receives 3 different SMS messages simultaneously (one confirmation + two "sorry we missed you" variants). Google Sheets gets 3 rows per call. Calendar gets an event AND the missed-call flow also logs.  
**Root cause:** The `builtin:BasicRouter` in Make sends data to ALL routes when no filter conditions are configured. None of the 3 routes had any filters.  
**Fixed:** Added `flow:Filter` modules:  
- Module 22 (Route 1 entry): passes through when `{{5.id}}` is not empty (Vapi succeeded)  
- Module 23 (Route 2 entry): passes through when `{{5.id}}` is empty (Vapi failed)  
- Route 3: no filter — acts as catch-all fallback for any edge cases  
**In Make execution log after fix:** Only one route's modules will show output bundles. The other routes will show the filter module with 0 bundles output (filtered out).

---

## Section 3: Per-Service Diagnostics

### Twilio

**Checking call logs:**  
1. Go to console.twilio.com > Monitor > Logs > Calls  
2. Find calls to +19493792748 — each entry shows From, To, Status, Duration, SID  
3. Compare the call count in Twilio to the scenario run count in Make — they should match 1:1 after the status filter fix  

**Verifying WatchCalls trigger number:**  
1. In Make, open the scenario and click Module 1  
2. The "To phone number" field should show `+19493792748` (with + prefix)  
3. If it shows `19493792748` (no +), the trigger will never match completed calls  

**Checking SMS delivery:**  
1. In Make execution log, click a successful run and expand Module 12 (or 16 or 20)  
2. The output bundle will show `sid` (a Message SID like `SM...`), `status`, `to`, `from`, `body`  
3. Cross-reference the Message SID in Twilio console > Monitor > Logs > Messages  
4. Verify the `to` field for Module 12 is the caller's number, and the `to` field for Module 13 is `+17148775606`  

**Common Twilio errors:**  
- Error 21211 (Invalid 'To' phone number): the `to` field in an SMS module has an invalid format — ensure E.164 format (+XXXXXXXXXXX)  
- Error 21614 (Not a mobile number): the caller's number is a landline — SMS will fail but this is expected behavior  
- Error 20003 (Authentication failure): Twilio connection credentials expired — re-authenticate in Make > Connections  

---

### Vapi

**Checking call logs:**  
1. Go to app.vapi.ai > Calls  
2. Filter by date or assistant ID (`f50d49cd-453c-47ac-9d72-7db9412006fb`)  
3. Each call shows status, duration, cost, and a transcript once the call completes  

**Verifying the assistantId:**  
1. In Vapi dashboard, go to Assistants  
2. Find "Alex" assistant and confirm its ID matches `f50d49cd-453c-47ac-9d72-7db9412006fb`  
3. If the assistant was re-created or the ID changed, update Module 4's request body  

**Finding your phoneNumberId UUID:**  
1. In Vapi dashboard, go to Phone Numbers  
2. Click on your HVAC phone number  
3. Copy the "ID" field — it will be a UUID like `abc12345-1234-1234-1234-abcdef123456`  
4. Replace `YOUR_VAPI_PHONE_NUMBER_UUID_HERE` in Module 4's request body with this UUID  

**Getting your Vapi API key:**  
1. In Vapi dashboard, go to Account (top right) > API Keys  
2. Create or copy an existing API key  
3. Replace `YOUR_VAPI_API_KEY_HERE` in Module 4's Authorization header value  

**Confirming a Vapi call was placed:**  
1. In Make execution log, expand Module 4 output  
2. With `parseResponse: true`, you'll see `id` (UUID), `status` ("queued"), `phoneNumberId`, `assistantId`  
3. Cross-reference the `id` in Vapi dashboard > Calls  

**IMPORTANT — Architectural gap (transcript flow):**  
This blueprint's Module 4 places an OUTBOUND call from Vapi to the customer. The Make scenario receives the Twilio inbound trigger, then immediately fires Vapi to call the customer back. This is call initiation, not call completion.

The post-call transcript (referenced as `{{5.artifact.transcript}}` in Module 9) is NOT available at the time Module 4 runs. Transcripts are delivered by Vapi via a POST to a webhook URL AFTER the call ends.

To make Module 9's transcript analysis work correctly, you need a SECOND Make scenario:
1. In Make, create a new scenario with a Webhook trigger  
2. In Vapi dashboard > Assistants > Alex > End of Call Report — set the webhook URL to your new Make webhook  
3. The second scenario receives `artifact.transcript`, `artifact.recordingUrl`, `summary`, etc.  
4. That second scenario can then run the OpenAI analysis (Module 9's logic) and the downstream logging  

Until the second scenario is built, Module 9 will receive an empty transcript and OpenAI will return default/empty values for all fields.

---

### OpenAI

**Checking API usage:**  
1. Go to platform.openai.com > Usage  
2. Filter by date — you should see API calls from your Make connection  
3. Each call should consume approximately 500-800 tokens (prompt + completion) at gpt-4o pricing  
4. If you see 0 calls but the scenario runs, check the API key in Make > Connections  

**Verifying the model name:**  
1. Valid models as of 2026: `gpt-4o`, `gpt-4o-mini`, `gpt-4-turbo`, `gpt-4`  
2. The original blueprint used `gpt-5.5` which does not exist — fixed to `gpt-4o`  
3. In Make execution log, Module 9 output bundle shows the `model` field confirming which model responded  

**Handling inconsistent JSON output:**  
1. Occasionally OpenAI may return extra text before or after the JSON object  
2. To force clean JSON: change Module 9's `response_format` to `json_object` in Make  
3. This requires the word "json" to appear in the prompt — it does ("return ONLY a valid JSON object")  
4. To test: in OpenAI Playground (platform.openai.com/playground), paste the prompt with a sample transcript and verify the output is parseable JSON  

**Testing the prompt manually:**  
1. Go to platform.openai.com/playground > Chat  
2. Select model: gpt-4o  
3. Paste a mock transcript (e.g. "Hi this is John Smith, I'm at 123 Main St in Irvine, my AC isn't cooling, it's been out for 2 days, I need someone urgently")  
4. Verify the output is a JSON object with caller_name, issue_type, address, preferred_time, urgency, summary fields  

---

### Google Calendar

**Verifying the calendar connection:**  
1. The calendar ID in Module 11 is `joshuaparadajoker@gmail.com`  
2. In Make > Connections, verify the Google connection is authenticated with the same Gmail account  
3. In Google Calendar (calendar.google.com), check that events appear in the primary calendar  

**Quick mode vs Detail mode:**  
1. The original blueprint used "quick" mode which creates events via natural language parsing — this caused all-day events with no time  
2. The fixed blueprint uses "detail" mode with explicit `startDate` and `endDate` fields  
3. In Make execution log, Module 11 output will include `start.dateTime` for detail mode events  

**Finding created events:**  
1. Go to calendar.google.com  
2. Search for "HVAC Service Call" — you should find events with the caller's phone number in the title  
3. Events should be scheduled for tomorrow (relative to when the call came in) with a 2-hour block  
4. The event description should contain the caller's phone, issue type, address, urgency, and summary  

---

### Google Sheets

**Verifying the spreadsheet:**  
1. The spreadsheet ID path is `/1Xw0sQzbdEEKXMitNE9tqfmjVwwNuWPE4flR6M75dieE`  
2. Access it at: https://docs.google.com/spreadsheets/d/1Xw0sQzbdEEKXMitNE9tqfmjVwwNuWPE4flR6M75dieE  
3. Verify the spreadsheet exists and is accessible by joshuaparadajoker@gmail.com  

**Required column headers (Row 1 of Sheet1):**  
The sheet must have these exact headers in this order for the mapper to work:  
- A: `Date`  
- B: `Caller Phone`  
- C: `Caller Name`  
- D: `Issue Type`  
- E: `Address`  
- F: `Urgency`  
- G: `Preferred`  
- H: `Time`  
- I: `Call Duration`  

**Diagnosing empty columns:**  
- Empty column A (Date): Check that Module 2 is running and `2.call_timestamp` is set. If Module 2 turns orange, it wasn't configured correctly.  
- Empty column B (Caller Phone): Was missing in original. Fixed to `{{1.From}}`. If still empty, check that Module 1 is outputting a `From` field.  
- Empty columns C-F (Name, Issue, Address, Urgency): These come from Module 24 (ParseJSON of OpenAI output). If Module 9 fails (bad model, bad API key) or Module 24 fails (invalid JSON), these will be empty. Check Module 9 and 24 output in execution log.  

**Diagnosing wrong columns vs empty columns:**  
- If data appears in wrong columns, the column headers in the sheet don't exactly match the mapper keys (case-sensitive). Compare sheet headers with the `values` keys in the module mapper.  
- The mapper uses `useColumnHeaders: true` which means it matches by header name, not position.  

---

## Section 4: Weekly Review Checklist

Run this check every Monday morning (or after any high-volume week):

- [ ] **Execution count vs call volume:** In Make > Scenario History, count runs for the past week. In Twilio console > Monitor > Calls, count completed calls to +19493792748. These should match 1:1. More Make runs than Twilio calls = status filter bug returning. Fewer = scenario is off or number mismatch.

- [ ] **No duplicate rows in Sheets:** Open the Google Sheet. Sort by Date column. Check for multiple rows with the same timestamp (within a few seconds). Duplicates = router filter not working or status filter missing.

- [ ] **Spot-check 3 Google Sheets rows:** Pick 3 random rows and verify:
  - Column A (Date): populated with a date/time string
  - Column B (Caller Phone): shows a real phone number in E.164 format (+XXXXXXXXXX)
  - Column D (Issue Type): shows a real issue type or "Unknown" (not empty, not "N/A" for Route 1 rows)

- [ ] **SMS recipient check:** In Make execution log, open 2-3 recent Route 1 runs. Expand Module 13 (SMS to Owner). Verify the `to` output field shows `+17148775606`, NOT the caller's number. If it shows a caller number, the original Bug 18 has re-appeared.

- [ ] **No red modules in execution history:** In Make > Scenario History, look for any runs with a red X (error) icon. Click into them and identify which module failed. Common failures: Module 4 (Vapi API key expired), Module 9 (OpenAI quota exceeded), Module 11 (Google Calendar auth expired).

- [ ] **Vapi callback calls appearing:** In Vapi dashboard > Calls, verify outbound calls are being initiated to caller phone numbers. Each call should show status "ended" with a duration > 0. If you see status "failed" repeatedly, the phoneNumberId or assistantId in Module 4 may be wrong.

- [ ] **OpenAI token consumption:** In platform.openai.com > Usage, verify the weekly token count is reasonable. For a business handling 20 calls/week, expect 10,000-16,000 tokens (500-800 per call). A spike could indicate the scenario triggered more than expected.

- [ ] **Google Calendar events have time slots:** In Google Calendar, switch to Week view. HVAC Service Call events should appear as time-blocked appointments (not all-day events spanning the top of the calendar). All-day events = calendar module reverted to "quick" mode.

- [ ] **Scenario is ON:** In Make, verify the scenario toggle is green (active). Check the scheduled trigger or webhook trigger is configured and active. A turned-off scenario won't process any calls.

- [ ] **Make operations quota:** In Make > Organization > Usage, check operations used vs your plan limit. Each scenario run uses approximately 15-20 operations (one per module execution). At 20 calls/week, that's 300-400 operations/week. Ensure you're not approaching your monthly cap.

---

## Bug Fix Summary Reference

| Bug | Module | Original Value | Fixed Value |
|-----|--------|----------------|-------------|
| 1 | 1 — WatchCalls | `"to": "19493792748"` | `"to": "+19493792748"` |
| 2 | 1 — WatchCalls | `"status": ""` | `"status": "completed"` |
| 3 | 2 — ParseJSON | Parsed `{{1.body}}` (doesn't exist) | Replaced with SetVariable2 setting call_timestamp |
| 4 | 3 — SetVariable2 | `"value": "click into the value field..."` | `"value": "{{1.From}}"` |
| 5 | 4 — Vapi HTTP | `"Bearer {{connection.vapiApiKey}}"` | `"Bearer YOUR_VAPI_API_KEY_HERE"` |
| 6 | 4 — Vapi HTTP | `"phoneNumberId": "{{1.To}}"` | `"phoneNumberId": "YOUR_VAPI_PHONE_NUMBER_UUID_HERE"` + customer.number field |
| 7 | 4 — Vapi HTTP | `"parseResponse": false` | `"parseResponse": true` |
| 8 | 6 — SetVariable2 | `"name": " gpt_prompt"` (leading space) | `"name": "gpt_prompt"` |
| 9 | 6 — SetVariable2 | Dead code, no downstream use | Repurposed to dynamic caller reference string |
| 10 | 7 — Router | No filter conditions on any route | Added flow:Filter modules 22 and 23 |
| 11 | 9 — OpenAI | `"model": "gpt-5.5"` | `"model": "gpt-4o"` |
| 12 | 9 — OpenAI | `{{transcript}}` (undefined variable) | `{{5.artifact.transcript}}` |
| 13 | 9 → 14 | No ParseJSON after OpenAI | Added Module 24 (json:ParseJSON) |
| 14 | 10 — SetVariables | `"value": "missed"` (wrong route) | `"value": "booked"` |
| 15 | 10 — SetVariables | `"name": " call_status"` (leading space) | `"name": "call_status"` |
| 16 | 11 — Google Calendar | `"select": "quick"` (all-day event) | `"select": "detail"` with startDate/endDate/summary/description |
| 17 | 12 — Twilio SMS | `{{caller_name}}` (undefined) | `{{ifempty(24.caller_name; "there")}}` |
| 18 | 13 — Twilio SMS | `"to": "{{1.From}}"` (sends to caller) | `"to": "+17148775606"` (owner's number) |
| 19 | 14 — Sheets | `"Date": "{{3.call_timestamp}}"` | `"Date": "{{2.call_timestamp}}"` |
| 20 | 14 — Sheets | `{{6.caller_name_final}}` etc. (nonexistent) | `{{24.caller_name}}` etc. |
| 21 | 14 — Sheets | Caller Phone column missing | Added `"Caller Phone": "{{1.From}}"` |
| 22 | 18 — Sheets | Same date bug + missing Caller Phone | Fixed date ref + added Caller Phone |
| 23 | 19 — SetVariables | `"value": "qualified"` (wrong label) | `"value": "missed-fallback"` |
| 24 | 21 — Sheets | Same date bug + broken refs + missing phone | Fixed all three issues |
