Discover local business leads and run AI-powered cold email sequences with Google Maps and Gmail

https://n8nworkflows.xyz/workflows/discover-local-business-leads-and-run-ai-powered-cold-email-sequences-with-google-maps-and-gmail-12984


# Discover local business leads and run AI-powered cold email sequences with Google Maps and Gmail

## 1. Workflow Overview

**Workflow name:** Discover local business leads and run automated cold email sequences with Google Maps and Gmail  
**Purpose:** End-to-end local lead generation + AI-written cold outreach + automated 3-step email sequencing (intro, follow-up 1, follow-up 2), using **Google Maps Places API**, **website scraping to find emails**, **Google Sheets** as a database/status tracker, **Gemini** for copy generation, and **Gmail** for sending/reply threading.

**Target use cases**
- Agencies/consultants generating local business leads by ZIP + category and running lightweight cold email sequences.
- Teams wanting a hands-free pipeline: scrape → enrich email → generate copy → send intro → send timed follow-ups.

### 1.1 Logical blocks
1. **Lead scraping (scheduled):** Read ZIPs + categories from Sheets, query Google Places API, scrape websites, extract emails, store leads in “Results”, update ZIP status.
2. **Google Sheets rate-limit handling:** Exponential backoff + retry loops for Sheets reads/writes with hard stop on repeated failures.
3. **AI email generation (scheduled):** For Results rows where “intro mail” is empty, generate intro + 2 follow-ups (JSON) via Gemini, write back to the row.
4. **Intro email sending (scheduled):** For rows where intro exists and not yet sent, send intro via Gmail, compute follow-up dates (D+7, D+11), store message ID + dates/status flags.
5. **Follow-up 1 sending (daily 8am):** If due date is today and status is “no”, send follow-up 1 as a Gmail **reply** to intro message, then update sheet with message ID and status.
6. **Follow-up 2 sending (daily 8am):** If due date is today and status is “no”, send follow-up 2 as a Gmail **reply** to follow-up 1 message, then update sheet with message ID and status.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Lead scraping (ZIP × Category → Places → Website → Email → Results)
**Overview:** Runs every 15 minutes, selects a small batch of unprocessed ZIPs, cross-joins with categories, searches Google Maps, scrapes websites, extracts an email, and writes leads to Google Sheets.

**Nodes involved**
- Trigger: Lead Scraping (GMaps)
- Settings
- Get Zip Codes
- Zips
- Filter Zips
- Split Out
- Limit: Zips Per Run
- Map Zip Field
- Get Category
- Merge Zips & Categories
- Build Zip × Category Matrix
- Set Zip
- Loop Subcats
- Set Zip1
- GMaps API
- Check: GMaps Result Empty
- Split Places Array
- Scrape URL
- Extract Emails From Website
- Check: Email Found
- Add rows in Google Sheets

#### Node details

**Trigger: Lead Scraping (GMaps)** (Schedule Trigger)
- **Role:** Entry point for lead discovery.
- **Config:** Every **15 minutes**.
- **Outputs:** To `Settings`.
- **Failure modes:** Missed runs if n8n is down; time drift depending on server timezone.

**Settings** (Set)
- **Role:** Central configuration (Google Sheet URL, tab names).
- **Key fields set:**
  - `gs_url`: placeholder `<ypur-google-sheets-link>` (must be replaced)
  - `catSheet`: `"Google Maps Categories"`
  - `sheet`: `"Zips"`
- **Outputs:** To `Get Zip Codes`.
- **Failure modes:** If `gs_url` invalid, all Sheets nodes fail.

**Get Zip Codes** (Google Sheets, v4.2, executeOnce=true)
- **Role:** Read ZIP rows from the `Zips` tab.
- **Config:** Sheet name from `Settings.sheet`, doc from `Settings.gs_url`.
- **Outputs:** To `Zips`.
- **Failure modes:** OAuth/auth expired; wrong tab name; API quota.

**Zips** (Set)
- **Role:** Pass through and normalize ZIP field (`zip = $json.Zip`) while keeping other fields.
- **Outputs:** To `Filter Zips`.
- **Edge cases:** If column is not exactly `Zip`, expressions break.

**Filter Zips** (Filter)
- **Role:** Select only ZIP rows where `status` is empty (meaning not scraped yet).
- **Condition used:** `status` is empty.
- **Note:** There is a second condition with blank left/right values; it is effectively inert/misconfigured and should be removed to avoid confusion.
- **Outputs:** To `Split Out`.
- **Failure modes:** If `status` column missing, filter may behave unexpectedly.

**Split Out** (Split Out)
- **Role:** Splits items by `row_number` field (Google Sheets metadata) to ensure each row is handled independently.
- **Outputs:** To `Limit: Zips Per Run`.

**Limit: Zips Per Run** (Limit)
- **Role:** Throttle load by processing only **3 ZIPs per run**.
- **Outputs:** To `Map Zip Field` and `Get Category` in parallel.

**Map Zip Field** (Set)
- **Role:** Ensure field `Zip` is present for downstream merge/cross join.
- **Outputs:** To `Merge Zips & Categories` (input 0).

**Get Category** (Google Sheets, v4.2, executeOnce=true)
- **Role:** Load business categories from tab `Google Maps Categories`.
- **Outputs:** To `Merge Zips & Categories` (input 1).
- **Edge cases:** Column name expectations: the cross-join code looks for `Categories` (plural), while other nodes expect `Category`. If your sheet column is named `Category`, the cross-join will fail unless you adjust the code.

**Merge Zips & Categories** (Merge, v3.1)
- **Role:** Combine the two streams (ZIPs + categories) into a single input list for the cross-join.
- **Outputs:** To `Build Zip × Category Matrix`.

**Build Zip × Category Matrix** (Code)
- **Role:** Cross-join ZIP list and category list to produce one item per (Zip, Category).
- **Key logic:**
  - Collects `item.json.Zip` into `zips[]`
  - Collects `item.json.Categories` into `categories[]`
  - Throws if either list empty
  - Outputs `{ Zip, Category }` pairs
- **Outputs:** To `Set Zip`.
- **Failure modes:**
  - If category column is not `Categories`, `categories[]` remains empty → node throws error.
  - If upstream merge doesn’t provide both datasets, throws error.

**Set Zip** (Set)
- **Role:** Normalize the paired record to fields `Zip` and `Category`.
- **Outputs:** To `Loop Subcats`.

**Loop Subcats** (Split In Batches, v3)
- **Role:** Iterate over Zip×Category pairs.
- **Config:** `reset: false` (continues batching).
- **Connections:** Uses the **second output** (index 1) to `Set Zip1`, and loops back from later nodes.
- **Failure modes:** If batch size default is large, can overload API calls; consider setting a batch size explicitly.

**Set Zip1** (Set)
- **Role:** Keep the current Category item and attach the Zip from `Set Zip`.
- **Key expressions:**
  - `Zip = {{ $('Set Zip').item.json.Zip }}`
  - `Category = {{ $json.Category }}`
- **Outputs:** To `GMaps API`.
- **Edge cases:** If item linking is wrong due to batching contexts, the `$('Set Zip')...` reference can misresolve.

**GMaps API** (HTTP Request, v4.2)
- **Role:** Query **Google Places API (v1)** via `places:searchText`.
- **Config:**
  - POST `https://places.googleapis.com/v1/places:searchText?key=YOUR_TOKEN_HERE`
  - Body: `textQuery = "{{ Category }} {{ Zip }}"`
  - Headers: `X-Goog-FieldMask: *`, `Content-Type: application/json`
  - Full response enabled
- **Outputs:** To `Check: GMaps Result Empty`.
- **Failure modes:** Invalid API key; billing disabled; quota; 429; overly broad `FieldMask:*` increases payload/quota usage.

**Check: GMaps Result Empty** (IF)
- **Role:** If `$json.body` is empty, skip and loop; else proceed.
- **Connections:**
  - True/empty → back to `Loop Subcats` (continue)
  - False/not empty → `Split Places Array`
- **Edge cases:** Places API errors may still return a body with error structure; current check only tests “empty object”, not “contains error”.

**Split Places Array** (Code)
- **Role:** Convert `body.places[]` into one item per place.
- **Adds:** `websiteUri` safe access (`places[i]?.websiteUri || null`).
- **Outputs:** To `Scrape URL`.
- **Failure modes:** If API shape changes, `body.places` may not exist.

**Scrape URL** (HTTP Request, v4.3, onError=continueRegularOutput, retryOnFail=true)
- **Role:** Fetch business website HTML.
- **Config:**
  - URL: `{{$json.websiteUri}}`
  - `neverError: true` (won’t fail execution on HTTP errors)
  - Custom browser-like headers, allowUnauthorizedCerts=true
- **Outputs:** To `Extract Emails From Website`.
- **Edge cases:** `websiteUri=null` may cause invalid URL requests; consider guarding before calling.

**Extract Emails From Website** (Code, onError=continueRegularOutput)
- **Role:** Strip scripts/styles/tags, find emails via regex, filter blacklisted domains, return first email.
- **Key expressions/refs:** Pulls place from `$('Split Places Array').item.json.place`.
- **Outputs:** To `Check: Email Found`.
- **Failure modes:** HTML in `$json.data` may be missing depending on HTTP node response format; can return null email frequently.

**Check: Email Found** (IF)
- **Role:** Continue only if `email` is not empty.
- **Outputs:** To `Add rows in Google Sheets`.
- **Edge cases:** This discards leads without scraped emails (no storage). If you want phone-only leads, adjust logic.

**Add rows in Google Sheets** (Google Sheets, v4.2, operation=appendOrUpdate, onError=continueErrorOutput, alwaysOutputData=true)
- **Role:** Upsert lead row into `Results` keyed by `place_id`.
- **Maps fields:** title, email, phone, website, rating, reviews, address, types, category.
- **Outputs:** To `Get Status` and `Calc Retry Delay (Add Rows)` (parallel via error output pattern).
- **Failure modes:** Rate-limit/quota triggers; schema mismatch; invalid sheet name (`"=Results"` includes a leading `=` which is unusual—should likely be `"Results"`).

---

### Block 2.2 — Google Sheets rate-limit handling (backoff + hard stop)
**Overview:** Wraps key Google Sheets operations (Add rows, Get status, Update ZIP status) with exponential backoff retries and stops after a threshold.

**Nodes involved**
- Get Status
- Update Status to Success
- Calc Retry Delay (Add Rows)
- Wait: Backoff Before Retry (Add Rows)
- Check: Retry Limit (Add Rows)
- Stop and Error1
- Calc Retry Delay (Get Status)
- Wait: Backoff Before Retry (Get Status)
- Check: Retry Limit (Get Status)
- Stop and Error2
- Calc Retry Delay (Update Status)
- Wait: Backoff Before Retry (Update Status)
- Check: Retry Limit (Sheet Update)
- Stop and Error

#### Node details

**Get Status** (Google Sheets, v4.2, onError=continueErrorOutput)
- **Role:** Check ZIP row status in Zips tab for the currently processed Zip.
- **Filter:** lookup `Zip = {{ $('Set Zip1').item.json.Zip }}`
- **Outputs:** success path to `Update Status to Success`, error path to `Calc Retry Delay (Get Status)`.
- **Failure modes:** API limit; missing Zip column; item context mismatch.

**Update Status to Success** (Google Sheets, v4.2, operation=update, executeOnce=true, onError=continueErrorOutput, alwaysOutputData=true)
- **Role:** Mark ZIP as processed by setting `status="scraped"` for matching `Zip`.
- **Outputs:** to `Loop Subcats` (continue next pair) and to `Calc Retry Delay (Update Status)` on error-output branch.
- **Failure modes:** Same as above; additionally, `executeOnce=true` may prevent desired per-item behavior depending on context.

**Calc Retry Delay (Add Rows / Get Status / Update Status)** (Code, runOnceForEachItem)
- **Role:** Exponential backoff calculator.
- **Logic:**
  - `retryCount = $json.retryCount || 0`
  - `maxRetries = 5` (but downstream IF checks `> 10`, inconsistent)
  - delay seconds = `1 * 2^retryCount`
  - returns `{ retryCount+1, waitTimeInSeconds, status:'retrying' }`
- **Edge cases:** Since maxRetries=5, it will return “failed” after 5, but the IF “retry limit” checks `>10`, so the “Stop and Error” condition may never trigger as intended. Harmonize these values.

**Wait: Backoff Before Retry (...)** (Wait)
- **Role:** Sleep for computed backoff time then retry.
- **Config:** uses `waitTimeInSeconds` for Add Rows; uses `waitTime` for others (but code returns `waitTimeInSeconds` for all three—mismatch).
- **Failure modes:** Variable mismatch results in `undefined` wait time and node errors or immediate continuation.

**Check: Retry Limit (Add Rows / Get Status / Sheet Update)** (IF)
- **Role:** Decide whether to stop or retry.
- **Config:** checks if `retryCount > 10` from the corresponding “Calc Retry Delay” node.
- **Outputs:**
  - True → Stop and Error*
  - False → retry the original Sheets node
- **Edge cases:** As noted, inconsistent retryCount naming and thresholds.

**Stop and Error / Stop and Error1 / Stop and Error2** (Stop and Error)
- **Role:** Hard stop with message: “Google Sheets API Limit has been triggered and the workflow has stopped”.
- **Failure modes:** None (intended termination).

---

### Block 2.3 — AI email generation (Gemini → write back to Results)
**Overview:** Runs every minute. Pulls rows from Results, filters those without intro mail, generates 3-email JSON using Gemini, updates the row.

**Nodes involved**
- Trigger: Generate AI Emails
- Get row(s) in sheet
- Check: Intro Mail Empty
- Loop: Generate AI Emails
- Email Writer
- Update row in sheet

#### Node details

**Trigger: Generate AI Emails** (Schedule Trigger)
- **Role:** Entry point for AI copy generation.
- **Config:** Every **1 minute**.
- **Outputs:** To `Get row(s) in sheet`.

**Get row(s) in sheet** (Google Sheets, v4.7)
- **Role:** Fetch rows from Results (sheet/document IDs are placeholders in this node: `your-gsheet-id`).
- **Outputs:** To `Check: Intro Mail Empty`.
- **Failure modes:** This node as provided is not configured to the actual spreadsheet (unlike other Results updates which point to a specific spreadsheet). Must be aligned.

**Check: Intro Mail Empty** (IF)
- **Role:** Only generate emails where `intro mail` is empty.
- **Outputs:** To `Loop: Generate AI Emails` (on true).
- **Edge cases:** If column name differs (e.g., `intro_mail`), condition fails.

**Loop: Generate AI Emails** (Split In Batches)
- **Role:** Batch generation to avoid model/API overload.
- **Outputs:** (index 1) to `Email Writer`, then loops back after sheet update.

**Email Writer** (`@n8n/n8n-nodes-langchain.googleGemini`, v1.1)
- **Role:** Generate **JSON** with `intro_mail`, `follow_up_mail_1`, `follow_up_mail_2`.
- **Model:** `models/gemini-2.0-flash-lite`
- **System message:** Senior B2B sales copywriter; strict instruction following.
- **Prompt inputs:** `title`, `type`, `types` from the row.
- **Output:** `jsonOutput: true`, but the downstream node parses `content.parts[0].text` as JSON—depends on node output format/version.
- **Failure modes:** Model refusing/format drift; quota; output not valid JSON.

**Update row in sheet** (Google Sheets, v4.7, operation=update)
- **Role:** Write AI-generated email fields to the Results row.
- **Matching:** `row_number`
- **Key expressions:**
  - `intro mail = JSON.parse($json.content.parts[0].text).intro_mail`
  - `follow up mail 1 = ...follow_up_mail_1`
  - `follow up mail 2 = ...follow_up_mail_2`
  - `row_number = {{ $('Loop: Generate AI Emails').item.json.row_number }}`
- **Failure modes:** If Gemini output format changes, JSON.parse fails; if row_number missing, update fails.

---

### Block 2.4 — Intro email sending + follow-up schedule initialization
**Overview:** Runs every minute. Selects rows with intro mail present and not sent, sends intro email, computes follow-up dates, updates the sheet with message IDs and status flags.

**Nodes involved**
- Trigger: Send Intro Emails
- Get row(s) in sheet1
- Check: Intro Ready & Not Sent
- Loop: Send Intro Emails
- Send Intro Mail
- Generate Follow Up Dates
- Update Row with all Follow Up Mail Dates

#### Node details

**Trigger: Send Intro Emails** (Schedule Trigger)
- **Config:** every 1 minute.
- **Outputs:** `Get row(s) in sheet1`.

**Get row(s) in sheet1** (Google Sheets, v4.7)
- **Role:** Fetch Results rows (currently placeholder `your-gsheet-id`).
- **Outputs:** To `Check: Intro Ready & Not Sent`.
- **Failure modes:** Must be configured to the same spreadsheet/tab as the rest of Results operations.

**Check: Intro Ready & Not Sent** (IF)
- **Role:** Ensure intro exists and hasn’t been sent.
- **Conditions:**
  - `intro mail` not empty
  - `email_status` empty (compared against `"sent"` but right side isn’t used consistently)
- **Outputs:** To `Loop: Send Intro Emails`.
- **Edge cases:** This should typically check `email_status != "yes"` (since later code uses `"yes"`). As written, status semantics are inconsistent.

**Loop: Send Intro Emails** (Split In Batches)
- **Role:** Send intro emails in batches.
- **Outputs:** (index 1) to `Send Intro Mail`, loops back after sheet update.

**Send Intro Mail** (Gmail, v2.2, operation=send)
- **Role:** Send intro email to scraped business email.
- **Key fields:**
  - To: `{{$json.email}}`
  - Subject: `Reducing manual work at {{ $json.title }}`
  - HTML body: uses `{{ $json['intro mail'] }}`
  - Options: senderName set; replyToSenderOnly=true; appendAttribution=false
- **Outputs:** To `Generate Follow Up Dates`.
- **Failure modes:** Gmail daily send limits; invalid email; OAuth revoked.

**Generate Follow Up Dates** (Code)
- **Role:** Compute ISO dates for intro (today), FU1 (+7), FU2 (+11); set flags:
  - `intro_sent="yes"`, `fu1_sent="no"`, `fu2_sent="no"`
- **Outputs:** To `Update Row with all Follow Up Mail Dates`.
- **Edge cases:** Uses server timezone via `new Date()`; may differ from intended business timezone.

**Update Row with all Follow Up Mail Dates** (Google Sheets, v4.7, operation=update)
- **Role:** Persist send status and message IDs/dates to Results row.
- **Fields written:** `email_status`, `intro mail id`, `intro_email_sent_date`, follow-up dates, follow-up statuses.
- **Matching:** `row_number`.
- **Failure modes:** Column mismatch; row_number missing; inconsistent status naming across workflow.

---

### Block 2.5 — Follow-up 1 (daily)
**Overview:** Runs daily at 8 AM. Filters eligible rows, checks whether FU1 date is today, sends FU1 as a reply to intro message, updates status.

**Nodes involved**
- Trigger: Send Follow Up 1
- Get row(s) in sheet4
- Check: Follow Up 1 Due
- Loop: Send Follow Up 1
- Check Follow Up 1 Date
- Follow Up Mail 1
- Update row with follow up status 1

#### Node details

**Trigger: Send Follow Up 1** (Schedule Trigger)
- **Config:** daily at `triggerAtHour: 8` (timezone = n8n instance timezone).
- **Outputs:** To `Get row(s) in sheet4`.

**Get row(s) in sheet4** (Google Sheets, v4.7)
- **Role:** Fetch Results rows (placeholder config again).
- **Outputs:** To `Check: Follow Up 1 Due`.

**Check: Follow Up 1 Due** (IF)
- **Role:** Preliminary filter:
  - follow up mail 1 not empty
  - follow up mail 1 status equals `"no"`
- **Outputs:** To `Loop: Send Follow Up 1`.

**Loop: Send Follow Up 1** (Split In Batches)
- **Role:** Batch sending.
- **Outputs:** (index 1) to `Check Follow Up 1 Date`, loops back after update.

**Check Follow Up 1 Date** (Code)
- **Role:** Final due-date gate:
  - skips if `email_status !== "yes"`
  - sends only if `follow up email send date 1 === today` and status `"no"`
- **Outputs:** To `Follow Up Mail 1` (or null to skip).
- **Failure modes:** If dates stored in different format than `YYYY-MM-DD`, comparisons fail.

**Follow Up Mail 1** (Gmail, v2.2, operation=reply)
- **Role:** Reply to intro email thread.
- **Threading:** `messageId = {{$json['intro mail id']}}`
- **Body:** uses `$('Loop: Send Follow Up 1').item.json['follow up mail 1']` (note: mixed referencing; could be simplified to `$json['follow up mail 1']`).
- **Outputs:** To `Update row with follow up status 1`.
- **Failure modes:** Missing/invalid intro message ID prevents proper reply; Gmail API errors.

**Update row with follow up status 1** (Google Sheets, v4.7, update)
- **Role:** Store FU1 message id and set FU1 status to `yes`.
- **Matching:** `row_number`.
- **Outputs:** Loops back to `Loop: Send Follow Up 1`.

---

### Block 2.6 — Follow-up 2 (daily)
**Overview:** Runs daily at 8 AM. Filters eligible rows, checks whether FU2 date is today, sends FU2 as a reply to FU1 message, updates status.

**Nodes involved**
- Trigger: Send Follow Up 2
- Get row(s) in sheet5
- Check: Follow Up 2 Due
- Loop: Send Follow Up 2
- Check Follow Up 2 Date
- Follow Up Mail 2
- Update row in sheet3

#### Node details

**Trigger: Send Follow Up 2** (Schedule Trigger)
- **Config:** daily at 8 AM.
- **Outputs:** To `Get row(s) in sheet5`.

**Get row(s) in sheet5** (Google Sheets, v4.7)
- **Role:** Fetch Results rows (placeholder config again).
- **Outputs:** To `Check: Follow Up 2 Due`.

**Check: Follow Up 2 Due** (IF)
- **Role:** Preliminary filter:
  - follow up mail 2 not empty
  - follow up mail 2 status equals `"no"`
- **Outputs:** To `Loop: Send Follow Up 2`.

**Loop: Send Follow Up 2** (Split In Batches)
- **Role:** Batch sending.
- **Outputs:** (index 1) to `Check Follow Up 2 Date`, loops back after update.

**Check Follow Up 2 Date** (Code)
- **Role:** Final due-date gate:
  - `email_status` must be `"yes"`
  - `follow up email send date 2 === today` and status `"no"`
- **Outputs:** To `Follow Up Mail 2` or null.
- **Failure modes:** Same date-format/timezone issues.

**Follow Up Mail 2** (Gmail, v2.2, operation=reply)
- **Role:** Reply to FU1 to continue thread.
- **Threading:** `messageId = {{$json['follow up mail 1 id']}}`
- **Body:** `{{ $json['follow up mail 2'] }}`
- **Outputs:** To `Update row in sheet3`.
- **Failure modes:** Missing FU1 message ID; Gmail API limits.

**Update row in sheet3** (Google Sheets, v4.7, update)
- **Role:** Store FU2 message id and mark FU2 status “yes”.
- **Matching:** `row_number`.
- **Outputs:** Loops back to `Loop: Send Follow Up 2`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Trigger: Lead Scraping (GMaps) | Schedule Trigger | Start lead scraping periodically | — | Settings | ## Lead Generation through GMaps API |
| Settings | Set | Stores sheet URLs/tab names | Trigger: Lead Scraping (GMaps) | Get Zip Codes | ## Lead Generation through GMaps API |
| Get Zip Codes | Google Sheets | Read ZIP queue | Settings | Zips | ## Lead Generation through GMaps API |
| Zips | Set | Normalize zip field | Get Zip Codes | Filter Zips | ## Lead Generation through GMaps API |
| Filter Zips | Filter | Keep only unprocessed ZIPs | Zips | Split Out | ## Lead Generation through GMaps API |
| Split Out | Split Out | Split items by row metadata | Filter Zips | Limit: Zips Per Run | ## Lead Generation through GMaps API |
| Limit: Zips Per Run | Limit | Throttle ZIPs per run | Split Out | Map Zip Field; Get Category | ## Lead discovery\n\nReads ZIP codes and business categories from Google Sheets, then queries Google Maps to find matching local businesses. |
| Map Zip Field | Set | Ensure `Zip` field for merge | Limit: Zips Per Run | Merge Zips & Categories | ## Lead Generation through GMaps API |
| Get Category | Google Sheets | Read target categories | Limit: Zips Per Run | Merge Zips & Categories | ## Lead Generation through GMaps API |
| Merge Zips & Categories | Merge | Combine ZIPs and categories streams | Map Zip Field; Get Category | Build Zip × Category Matrix | ## Lead Generation through GMaps API |
| Build Zip × Category Matrix | Code | Cross-join ZIPs × categories | Merge Zips & Categories | Set Zip | ## Lead Generation through GMaps API |
| Set Zip | Set | Prepare Zip+Category item | Build Zip × Category Matrix | Loop Subcats | ## Lead Generation through GMaps API |
| Loop Subcats | Split In Batches | Iterate Zip×Category pairs | Set Zip; Update Status to Success; Check: GMaps Result Empty | Set Zip1 | ## Lead discovery\n\nReads ZIP codes and business categories from Google Sheets, then queries Google Maps to find matching local businesses. |
| Set Zip1 | Set | Attach Zip to current category | Loop Subcats | GMaps API | ## Lead discovery\n\nReads ZIP codes and business categories from Google Sheets, then queries Google Maps to find matching local businesses. |
| GMaps API | HTTP Request | Search places for “Category Zip” | Set Zip1 | Check: GMaps Result Empty | ## Lead discovery\n\nReads ZIP codes and business categories from Google Sheets, then queries Google Maps to find matching local businesses. |
| Check: GMaps Result Empty | IF | Skip if Places response empty | GMaps API | Loop Subcats; Split Places Array | ## Lead discovery\n\nReads ZIP codes and business categories from Google Sheets, then queries Google Maps to find matching local businesses. |
| Split Places Array | Code | One item per place | Check: GMaps Result Empty | Scrape URL | ## Lead discovery\n\nReads ZIP codes and business categories from Google Sheets, then queries Google Maps to find matching local businesses. |
| Scrape URL | HTTP Request | Fetch website HTML | Split Places Array | Extract Emails From Website | ## Website scraping\n\nVisits each business website and extracts public email addresses from the page content. |
| Extract Emails From Website | Code | Regex extract email | Scrape URL | Check: Email Found | ## Website scraping\n\nVisits each business website and extracts public email addresses from the page content. |
| Check: Email Found | IF | Continue only if email found | Extract Emails From Website | Add rows in Google Sheets | ## Website scraping\n\nVisits each business website and extracts public email addresses from the page content. |
| Add rows in Google Sheets | Google Sheets | Upsert lead into Results | Check: Email Found; Check: Retry Limit (Add Rows) | Get Status; Calc Retry Delay (Add Rows) | ## Lead storage\nSaves cleaned lead data (name, email, phone, rating, website) into the Results sheet. |
| Get Status | Google Sheets | Lookup ZIP row for status | Add rows in Google Sheets; Check: Retry Limit (Get Status) | Update Status to Success; Calc Retry Delay (Get Status) | ## Rate limit handling\nPrevents Google Sheets API limits using\nexponential backoff and retries. |
| Update Status to Success | Google Sheets | Mark ZIP as scraped | Get Status; Check: Retry Limit (Sheet Update) | Loop Subcats; Calc Retry Delay (Update Status) | ## Rate limit handling\nPrevents Google Sheets API limits using\nexponential backoff and retries. |
| Calc Retry Delay (Add Rows) | Code | Compute backoff for Add Rows | Add rows in Google Sheets | Wait: Backoff Before Retry (Add Rows) | ## Rate limit handling\nPrevents Google Sheets API limits using\nexponential backoff and retries. |
| Wait: Backoff Before Retry (Add Rows) | Wait | Sleep before retry | Calc Retry Delay (Add Rows) | Check: Retry Limit (Add Rows) | ## Rate limit handling\nPrevents Google Sheets API limits using\nexponential backoff and retries. |
| Check: Retry Limit (Add Rows) | IF | Stop or retry Add Rows | Wait: Backoff Before Retry (Add Rows) | Stop and Error1; Add rows in Google Sheets | ## Rate limit handling\nPrevents Google Sheets API limits using\nexponential backoff and retries. |
| Stop and Error1 | Stop and Error | Hard stop on repeated limit | Check: Retry Limit (Add Rows) | — | ## Rate limit handling\nPrevents Google Sheets API limits using\nexponential backoff and retries. |
| Calc Retry Delay (Get Status) | Code | Compute backoff for Get Status | Get Status | Wait: Backoff Before Retry (Get Status) | ## Rate limit handling\nPrevents Google Sheets API limits using\nexponential backoff and retries. |
| Wait: Backoff Before Retry (Get Status) | Wait | Sleep before retry | Calc Retry Delay (Get Status) | Check: Retry Limit (Get Status) | ## Rate limit handling\nPrevents Google Sheets API limits using\nexponential backoff and retries. |
| Check: Retry Limit (Get Status) | IF | Stop or retry Get Status | Wait: Backoff Before Retry (Get Status) | Stop and Error2; Get Status | ## Rate limit handling\nPrevents Google Sheets API limits using\nexponential backoff and retries. |
| Stop and Error2 | Stop and Error | Hard stop on repeated limit | Check: Retry Limit (Get Status) | — | ## Rate limit handling\nPrevents Google Sheets API limits using\nexponential backoff and retries. |
| Calc Retry Delay (Update Status) | Code | Compute backoff for Update Status | Update Status to Success | Wait: Backoff Before Retry (Update Status) | ## Rate limit handling\nPrevents Google Sheets API limits using\nexponential backoff and retries. |
| Wait: Backoff Before Retry (Update Status) | Wait | Sleep before retry | Calc Retry Delay (Update Status) | Check: Retry Limit (Sheet Update) | ## Rate limit handling\nPrevents Google Sheets API limits using\nexponential backoff and retries. |
| Check: Retry Limit (Sheet Update) | IF | Stop or retry Update Status | Wait: Backoff Before Retry (Update Status) | Stop and Error; Update Status to Success | ## Rate limit handling\nPrevents Google Sheets API limits using\nexponential backoff and retries. |
| Stop and Error | Stop and Error | Hard stop on repeated limit | Check: Retry Limit (Sheet Update) | — | ## Rate limit handling\nPrevents Google Sheets API limits using\nexponential backoff and retries. |
| Trigger: Generate AI Emails | Schedule Trigger | Start AI email generation | — | Get row(s) in sheet | ## Email Writer (Intro + Follow Up 1 + Follow Up 2) |
| Get row(s) in sheet | Google Sheets | Read Results rows for AI generation | Trigger: Generate AI Emails | Check: Intro Mail Empty | ## Email Writer (Intro + Follow Up 1 + Follow Up 2) |
| Check: Intro Mail Empty | IF | Only rows missing intro mail | Get row(s) in sheet | Loop: Generate AI Emails | ## AI email generation\nGenerates personalized intro and follow-up emails using business data and categories. |
| Loop: Generate AI Emails | Split In Batches | Batch AI calls | Check: Intro Mail Empty; Update row in sheet | Email Writer | ## Email Writer (Intro + Follow Up 1 + Follow Up 2) |
| Email Writer | Google Gemini (LangChain) | Generate 3 emails as JSON | Loop: Generate AI Emails | Update row in sheet | ## AI email generation\nGenerates personalized intro and follow-up emails using business data and categories. |
| Update row in sheet | Google Sheets | Write generated emails to Results | Email Writer | Loop: Generate AI Emails | ## Email Writer (Intro + Follow Up 1 + Follow Up 2) |
| Trigger: Send Intro Emails | Schedule Trigger | Start intro sending | — | Get row(s) in sheet1 | ## Intro Mail Send |
| Get row(s) in sheet1 | Google Sheets | Read Results rows for intro send | Trigger: Send Intro Emails | Check: Intro Ready & Not Sent | ## Intro Mail Send |
| Check: Intro Ready & Not Sent | IF | Intro exists & not sent | Get row(s) in sheet1 | Loop: Send Intro Emails | ## Intro Mail Send |
| Loop: Send Intro Emails | Split In Batches | Batch intro sends | Check: Intro Ready & Not Sent; Update Row with all Follow Up Mail Dates | Send Intro Mail | ## Intro Mail Send |
| Send Intro Mail | Gmail | Send initial outreach email | Loop: Send Intro Emails | Generate Follow Up Dates | ## Intro send\nSends first outreach email and stores message ID. |
| Generate Follow Up Dates | Code | Compute +7/+11 dates & flags | Send Intro Mail | Update Row with all Follow Up Mail Dates | ## Intro Mail Send |
| Update Row with all Follow Up Mail Dates | Google Sheets | Store IDs, dates, status flags | Generate Follow Up Dates | Loop: Send Intro Emails | ## Intro Mail Send |
| Trigger: Send Follow Up 1 | Schedule Trigger | Start FU1 daily run | — | Get row(s) in sheet4 | ## Follow Up 1 Mail Send |
| Get row(s) in sheet4 | Google Sheets | Read Results rows for FU1 | Trigger: Send Follow Up 1 | Check: Follow Up 1 Due | ## Follow Up 1 Mail Send |
| Check: Follow Up 1 Due | IF | FU1 content exists & status=no | Get row(s) in sheet4 | Loop: Send Follow Up 1 | ## Follow up 1\nReplies to intro email after 7 days. |
| Loop: Send Follow Up 1 | Split In Batches | Batch FU1 sends | Check: Follow Up 1 Due; Update row with follow up status 1 | Check Follow Up 1 Date | ## Follow Up 1 Mail Send |
| Check Follow Up 1 Date | Code | Ensure due today & intro sent | Loop: Send Follow Up 1 | Follow Up Mail 1 | ## Follow Up 1 Mail Send |
| Follow Up Mail 1 | Gmail | Reply to intro message ID | Check Follow Up 1 Date | Update row with follow up status 1 | ## Follow Up 1 Mail Send |
| Update row with follow up status 1 | Google Sheets | Save FU1 message ID + status | Follow Up Mail 1 | Loop: Send Follow Up 1 | ## Follow Up 1 Mail Send |
| Trigger: Send Follow Up 2 | Schedule Trigger | Start FU2 daily run | — | Get row(s) in sheet5 | ## Follow Up 2 Mail Send |
| Get row(s) in sheet5 | Google Sheets | Read Results rows for FU2 | Trigger: Send Follow Up 2 | Check: Follow Up 2 Due | ## Follow Up 2 Mail Send |
| Check: Follow Up 2 Due | IF | FU2 content exists & status=no | Get row(s) in sheet5 | Loop: Send Follow Up 2 | ## Follow up 2\nFinal reply after 11 days. |
| Loop: Send Follow Up 2 | Split In Batches | Batch FU2 sends | Check: Follow Up 2 Due; Update row in sheet3 | Check Follow Up 2 Date | ## Follow Up 2 Mail Send |
| Check Follow Up 2 Date | Code | Ensure due today & intro sent | Loop: Send Follow Up 2 | Follow Up Mail 2 | ## Follow Up 2 Mail Send |
| Follow Up Mail 2 | Gmail | Reply to FU1 message ID | Check Follow Up 2 Date | Update row in sheet3 | ## Follow Up 2 Mail Send |
| Update row in sheet3 | Google Sheets | Save FU2 message ID + status | Follow Up Mail 2 | Loop: Send Follow Up 2 | ## Follow Up 2 Mail Send |
| Get row(s) in sheet2 | Google Sheets | (Not present) |  |  |  |
| Stop and Error4 | Stop and Error | (Not present) |  |  |  |

> Note: Sticky notes “How it works”, “How to run”, and headers are included in relevant rows above where applicable; see Section 5 for full text resources.

---

## 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheet structure**
   1) Create (or copy) a spreadsheet with tabs:
   - **Zips**: columns at least `Zip`, `status`
   - **Google Maps Categories**: a column named **`Categories`** (to match the cross-join code) or change the code to read `Category`
   - **Results**: columns used by the workflow (examples):  
     `place_id, title, email, phone, website, rating, reviews, type, address, types, intro mail, intro mail id, email_status, intro_email_sent_date, follow up mail 1, follow up mail 1 id, follow up email send date 1, follow up mail 1 status, follow up mail 2, follow up mail 2 id, follow up email send date 2, follow up mail 2 status, row_number`

2. **Create credentials in n8n**
   - **Google Sheets OAuth2** credential with access to the spreadsheet.
   - **Gmail OAuth2** credential with permission to send email.
   - **Google Gemini / PaLM** credential (as required by `@n8n/n8n-nodes-langchain.googleGemini`).

3. **Build Block: Lead scraping**
   1) Add **Schedule Trigger** named `Trigger: Lead Scraping (GMaps)` → every 15 minutes.  
   2) Add **Set** named `Settings` with fields:
      - `gs_url` = your Google Sheet URL
      - `sheet` = `Zips`
      - `catSheet` = `Google Maps Categories`
   3) Add **Google Sheets** `Get Zip Codes`: read rows from doc `{{ $('Settings').item.json.gs_url }}` and sheet `{{ $('Settings').item.json.sheet }}`.
   4) Add **Set** `Zips` to map `zip = {{$json.Zip}}` (optional).
   5) Add **Filter** `Filter Zips`: keep items where `status` is empty.
   6) Add **Split Out** `Split Out` on field `row_number`.
   7) Add **Limit** `Limit: Zips Per Run` with maxItems=3.
   8) Add **Set** `Map Zip Field` setting `Zip = {{$json.Zip}}`.
   9) Add **Google Sheets** `Get Category`: read from sheet `{{ $('Settings').item.json.catSheet }}`.
   10) Add **Merge** `Merge Zips & Categories` (combine both inputs).
   11) Add **Code** `Build Zip × Category Matrix` implementing the cross join (ensure it reads your category column name).
   12) Add **Set** `Set Zip` to output `Zip` and `Category`.
   13) Add **SplitInBatches** `Loop Subcats`.
   14) Add **Set** `Set Zip1` to forward `Zip` and current `Category`.
   15) Add **HTTP Request** `GMaps API`:
       - POST to `https://places.googleapis.com/v1/places:searchText?key=YOUR_KEY`
       - JSON body: `{ "textQuery": "{{ $json.Category }} {{ $json.Zip }}" }`
       - Headers: `X-Goog-FieldMask: *`, `Content-Type: application/json`
   16) Add **IF** `Check: GMaps Result Empty` to branch if `$json.body` is empty.
   17) Add **Code** `Split Places Array` to emit one item per place and attach `websiteUri`.
   18) Add **HTTP Request** `Scrape URL` to fetch `{{$json.websiteUri}}` (set `neverError=true`).
   19) Add **Code** `Extract Emails From Website` to parse `$json.data` and return `{ place, email }`.
   20) Add **IF** `Check: Email Found` for `email notEmpty`.
   21) Add **Google Sheets** `Add rows in Google Sheets`:
       - operation: appendOrUpdate
       - match: `place_id`
       - sheet: `Results`
       - map the place fields + email.

4. **Add Block: Sheets rate-limit handling**
   - For each critical Sheets node (Add rows, Get Status, Update Status):
     1) Set `onError=continueErrorOutput`
     2) Add a **Code** node “Calc Retry Delay …” (exponential backoff)
     3) Add a **Wait** node using the computed seconds
     4) Add an **IF** “Check Retry Limit …”
     5) True → **Stop and Error**, False → route back to the original Sheets node  
   - Ensure variable names match (e.g., always use `waitTimeInSeconds`) and align `maxRetries` with the IF threshold.

5. **Build Block: AI email generation**
   1) Add **Schedule Trigger** `Trigger: Generate AI Emails` every 1 minute.
   2) Add **Google Sheets** `Get row(s) in sheet` reading from `Results` (configure to your real spreadsheet).
   3) Add **IF** `Check: Intro Mail Empty`.
   4) Add **SplitInBatches** `Loop: Generate AI Emails`.
   5) Add **Google Gemini** node `Email Writer`:
      - Model: `gemini-2.0-flash-lite`
      - Force JSON-only output with fields `intro_mail`, `follow_up_mail_1`, `follow_up_mail_2`
   6) Add **Google Sheets** `Update row in sheet` updating `intro mail`, `follow up mail 1`, `follow up mail 2` by `row_number`.

6. **Build Block: Intro send**
   1) Add **Schedule Trigger** `Trigger: Send Intro Emails` every 1 minute.
   2) Add **Google Sheets** `Get row(s) in sheet1` from `Results`.
   3) Add **IF** `Check: Intro Ready & Not Sent`.
   4) Add **SplitInBatches** `Loop: Send Intro Emails`.
   5) Add **Gmail** `Send Intro Mail` (operation send) with HTML template using `{{$json['intro mail']}}`.
   6) Add **Code** `Generate Follow Up Dates` to compute date strings and set status flags.
   7) Add **Google Sheets** `Update Row with all Follow Up Mail Dates` updating row by `row_number`.

7. **Build Block: Follow-up 1**
   1) Add **Schedule Trigger** daily at 8 AM.
   2) Read Results, filter status=no, check due date=today via Code node.
   3) Gmail reply using `messageId = intro mail id`.
   4) Update sheet with FU1 message id and set FU1 status yes.

8. **Build Block: Follow-up 2**
   1) Add **Schedule Trigger** daily at 8 AM.
   2) Read Results, filter status=no, check due date=today.
   3) Gmail reply using `messageId = follow up mail 1 id`.
   4) Update sheet with FU2 message id and set FU2 status yes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow automatically finds local business leads… includes retry logic, rate-limit handling, and status tracking…” + setup steps including copying a sheet | Sticky note “How it works” (includes link): https://docs.google.com/spreadsheets/d/1BRPF4IHoxtAIE5gBQMNP_qIux_XNDXTnmIEHYcl5kEk/edit?usp=sharing |
| “How to run” instructions: fill Zips tab, fill Categories tab, run scraping, AI writer runs every minute for rows where intro mail empty; intro sends when intro present + email status empty; FU1 after 7 days; FU2 after 11 days; follow-ups run daily 8 AM | Sticky note “How to run” |
| Placeholders must be replaced: Google Maps API key, company name, logo URL, CTA, sheet IDs in some nodes | Applies across workflow |
| Status semantics are inconsistent (`email_status` checked as empty vs code expecting `"yes"`). Standardize to a single convention (e.g., `email_status=yes/no`). | Applies to intro + follow-up blocks |
| Some “Get row(s) in sheet*” nodes use placeholder `your-gsheet-id` while other nodes target a specific spreadsheet. Align all Google Sheets nodes to the same document and `Results` tab. | Applies to AI + sending blocks |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.