Run an AI SDR sales pipeline with OpenAI, Google Sheets, Gmail and Calendar

https://n8nworkflows.xyz/workflows/run-an-ai-sdr-sales-pipeline-with-openai--google-sheets--gmail-and-calendar-13529


# Run an AI SDR sales pipeline with OpenAI, Google Sheets, Gmail and Calendar

## 1. Workflow Overview

**Title:** Run an AI SDR sales pipeline with OpenAI, Google Sheets, Gmail and Calendar

This workflow implements an automated SDR-style outbound pipeline using Google Sheets as both a temporary lead intake list and a master CRM, Gmail for sending follow-ups, Google Calendar to detect booked calls, and OpenAI (via n8n LangChain nodes) to personalize email templates.

### 1.1 Block 1 — CRM Agent (Lead intake → CRM + Lead List reset)
Runs daily, pulls new leads from a “Lead List” sheet, upserts them into a CRM sheet, assigns/normalizes an ID based on row number, then clears the Lead List.

### 1.2 Block 2 — Follow-Up Agent (Sequenced outreach every 48h)
Runs on a schedule, selects CRM records where the sequence has started and no call is scheduled, checks timing (last touch ≥ 2 days) and follow-up count (< 3), chooses the correct follow-up template (1/2/3), personalizes it with OpenAI, sends via Gmail, and updates CRM tracking fields.

### 1.3 Block 3 — Concierge Agent (Calendar booking → stop sequence)
Triggered when a calendar event is created. If it matches organizer criteria, updates the matching CRM record to “Call Scheduled? = true” using the attendee email, pulling them out of the automated follow-up selection logic.

### 1.4 Block 4 — No-Show Agent (Reschedule email + status update)
Runs every 6 hours, finds CRM rows marked as no-show (“Showed Call?” = “No”), sends a reschedule email, then updates “Showed Call?” to “Followed up” to prevent duplicate emails.

---

## 2. Block-by-Block Analysis

### Block 1 — CRM Agent (Lead List → CRM)

**Overview:** Imports leads from a temporary Google Sheet into the CRM Google Sheet once per day. After upserting records, it sets the CRM “ID” field (based on Google Sheets row number) and deletes imported leads from the Lead List.

**Nodes involved:**
- Schedule Trigger
- Email List
- Loop Over Items
- Update CRM
- Get row(s) from CRM
- Wait
- Finalize Contact Record in Sheet
- Count
- Reset email list

#### Sticky Note coverage
- **Sticky Note:** “# Flow 1: CRM Agent — Moves leads from Lead List to CRM”

#### Node details

**1) Schedule Trigger**
- **Type / role:** `Schedule Trigger` — entry point; runs periodically.
- **Config:** Every 24 hours.
- **Output:** Triggers **Email List**.
- **Potential failures:** None typical; clock/timezone misunderstandings depending on n8n instance settings.
- **Version:** 1.2

**2) Email List**
- **Type / role:** `Google Sheets` — reads leads from Lead List spreadsheet.
- **Config (interpreted):**
  - Document: **[AFT] The Sales Agent - Lead List** (`documentId: 1msNL...`)
  - Sheet/tab: `gid=0` (“Sheet1”)
  - Operation is not explicitly shown, but the node is used as a “read rows” source into batching.
- **Credentials:** Google Sheets OAuth2.
- **Output:** Items (rows) → **Loop Over Items**.
- **Edge cases:**
  - Empty sheet → downstream nodes get 0 items.
  - Missing expected columns (First Name, Contact Email, etc.) → expressions in later nodes resolve to `null`.
  - OAuth token expiration / permissions.
- **Version:** 4.7

**3) Loop Over Items**
- **Type / role:** `Split In Batches` — iterates through leads.
- **Config:** Default batching options (batch size not specified; n8n default applies).
- **Connections:**
  - Output 1 → **Update CRM**
  - Output 2 → **Count** (this is unusual wiring; it implies Count is fed in parallel while processing batches, but in n8n it will run per batch execution path depending on how the node is triggered).
- **Edge cases:**
  - If batch size default is large, may hit Sheets rate limits.
- **Version:** 3

**4) Update CRM**
- **Type / role:** `Google Sheets` — upsert (append or update) lead into CRM.
- **Config:**
  - Document: **[AFT] The Sales Agent - CRM** (`documentId: 1sEx...`)
  - Operation: **appendOrUpdate**
  - Matching column: **ID**
  - Mapped columns: Role, First Name, Last Name, Company info, Contact Email, etc.
  - Sets `ID = null` intentionally, so records will be appended (or updated only if a row with empty ID matches, which typically won’t happen). Practically: this behaves like **append**.
- **Output:** → **Get row(s) from CRM**
- **Edge cases / risks:**
  - Using **ID** as the match key while setting ID to `null` means deduplication by email does *not* occur here. Duplicates can be created if the same lead appears again.
  - Column schema drift in Sheets breaks mapping.
- **Version:** 4.7

**5) Get row(s) from CRM**
- **Type / role:** `Google Sheets` — fetches CRM row by email to get metadata (notably `row_number`).
- **Config:**
  - Filters: `Contact Email` equals `{{$json['Contact Email']}}`
- **Output:** → **Wait**
- **Edge cases:**
  - If duplicates exist for the same email, this may return multiple rows; downstream “Finalize” may update multiple.
  - If no row found, `row_number` missing → Finalize step fails or writes blanks.
- **Version:** 4.7

**6) Wait**
- **Type / role:** `Wait` — delays continuation (15 units; node uses “amount: 15”).
- **Config:** Amount = 15 (in the Wait node, unit depends on node settings; here it’s not explicit in JSON snippet—commonly minutes in UI, but confirm in n8n).
- **Purpose:** Likely gives Google Sheets time to commit changes so row_number is reliable.
- **Output:** → **Finalize Contact Record in Sheet**
- **Edge cases:** Adds latency; can slow large imports significantly.
- **Version:** 1.1

**7) Finalize Contact Record in Sheet**
- **Type / role:** `Google Sheets` — upserts CRM row to set ID = row_number.
- **Config:**
  - Operation: **appendOrUpdate**
  - Matching column: **Contact Email**
  - Writes:
    - `ID = {{$json.row_number}}`
    - `Contact Email = {{$json['Contact Email']}}`
- **Output:** → **Loop Over Items** (to continue batching loop)
- **Edge cases:**
  - If `row_number` missing, ID becomes empty/incorrect.
  - If duplicates exist, multiple rows may get same ID logic inconsistently.
- **Version:** 4.7

**8) Count**
- **Type / role:** `Summarize` — counts number of processed leads by summarizing a field.
- **Config:** Summarize field “Contact Email”; output includes `count_Contact_Email`.
- **Output:** → **Reset email list**
- **Edge cases:** If no items reach Count, delete amount may be `0` or undefined.
- **Version:** 1.1

**9) Reset email list**
- **Type / role:** `Google Sheets` — deletes rows from Lead List after import.
- **Config:**
  - Operation: **delete**
  - Number to delete: `{{$json.count_Contact_Email}}`
  - Document: Lead List sheet (`1msNL...`)
- **Edge cases / risks:**
  - Deleting “N rows” typically deletes from the top; if the sheet includes headers or sorting, may delete wrong data.
  - If count is wrong due to batching wiring, may delete too many/too few.
- **Version:** 4.7

---

### Block 2 — Follow-Up Agent (AI-personalized 3-step sequence)

**Overview:** On a schedule, selects active leads who are in sequence but haven’t booked a call, ensures 48 hours have passed since last send and that fewer than 3 follow-ups were sent, then generates the appropriate follow-up email (1–3) via OpenAI-driven placeholder replacement, sends through Gmail, and updates the CRM.

**Nodes involved:**
- Schedule Trigger (Follow Ups)
- Get rows from CRM
- Filter
- Which follow-up phase?
- OpenAI Follow Up
- Follow Up 1
- Send First Follow Up
- Update CRM (Follow Up 1)
- Follow Up 2
- Send Second Follow Up
- Update CRM (Follow Up 2)
- Follow Up 3
- Send Third Follow Up
- Update CRM (Follow Up 3)

#### Sticky Note coverage
- **Sticky Note1:** “# Flow 2: Follow-Up Agent — Sends 3 personalized follow-up emails every 2 days”

#### Node details

**1) Schedule Trigger (Follow Ups)**
- **Type / role:** `Schedule Trigger` — entry point for follow-up cycle.
- **Config:** `rule.interval:[{}]` (effectively unspecified in the exported JSON; in UI it must be configured).
- **Output:** → **Get rows from CRM**
- **Edge cases:** If misconfigured, it may never run or run too frequently.
- **Version:** 1.2

**2) Get rows from CRM**
- **Type / role:** `Google Sheets` — queries CRM for targets.
- **Config:**
  - Filters:
    - `Started Sequence? = "true"`
    - `Call Scheduled? = "false"`
- **Output:** → **Filter**
- **Edge cases:**
  - Sheet uses string values `"true"/"false"` (not booleans). If CRM contains TRUE/FALSE boolean types or other casing, filter misses rows.
- **Version:** 4.7

**3) Filter**
- **Type / role:** `Filter` — enforces timing and maximum attempts.
- **Config (interpreted):**
  - Condition A (date): `Sequence Last Date` **beforeOrEquals** `($now - 2 days).format('MM-dd-yy')`
  - Condition B (count): `Follow Ups` **lt** `3`
- **Key expressions:**
  - `{{ $('Get rows from CRM').item.json['Sequence Last Date'] }}`
  - `{{ $now.minus({days: 2}).format('MM-dd-yy') }}`
  - `{{ $('Get rows from CRM').item.json['Follow Ups'] }}`
- **Output:** → **Which follow-up phase?**
- **Edge cases / risks:**
  - Date stored as string `MM-dd-yy` compared as dateTime: parsing can fail if empty, different locale, or actual Sheets date type.
  - `Follow Ups` stored as string; “loose” validation may coerce, but blanks can behave unexpectedly.
- **Version:** 2.2

**4) Which follow-up phase?**
- **Type / role:** `Switch` — routes to follow-up template 1/2/3 based on current count.
- **Rules:**
  - Output “follow up 1” if `Follow Ups < 1`
  - Output “follow up 2” if `Follow Ups < 2`
  - Output “follow up 3” if `Follow Ups < 3`
- **Connections:**
  - Output 1 → **Follow Up 1**
  - Output 2 → **Follow Up 2**
  - Output 3 → **Follow Up 3**
- **Edge cases:** If Follow Ups is missing/null, it may satisfy multiple rules; switch evaluation order matters (first match wins).
- **Version:** 3.3

**5) OpenAI Follow Up**
- **Type / role:** `lmChatOpenAi` — provides the language model used by the LLM chain nodes.
- **Config:** Model `gpt-4.1-nano`.
- **Connections:** Linked via `ai_languageModel` to **Follow Up 1/2/3**.
- **Credentials:** OpenAI credentials are implied (not shown in this node export, but required in n8n).
- **Edge cases:**
  - Model availability changes; ensure your OpenAI account has access.
  - Rate limits; email bursts could throttle.
- **Version:** 1.2

**6) Follow Up 1 / Follow Up 2 / Follow Up 3**
- **Type / role:** `@n8n/n8n-nodes-langchain.chainLlm` — generates final email text by replacing placeholders in a fixed template.
- **Config:**
  - Each node includes a base email with placeholders like `[Name]` and `[Calendar Link]`.
  - A system-style instruction message provides available variables from the CRM row:
    - First Name, Role, Company Name, Company Industry
    - Calendar link: `https://calendar.app.google/iSiNxcUzP2ajSMtT8`
  - Instruction: “Only output the final email… Do not include comments…”
- **Inputs:** From **Which follow-up phase?** plus the linked OpenAI model.
- **Outputs:** `text` (used by Gmail nodes).
- **Edge cases:**
  - If First Name is missing, `[Name]` may remain unchanged depending on the model’s adherence.
  - Prompt injection risk if CRM fields contain adversarial text (low likelihood but possible).
- **Version:** 1.7

**7) Send First/Second/Third Follow Up**
- **Type / role:** `Gmail` — sends follow-up email.
- **Config:**
  - To: `{{ $('Get rows from CRM').item.json['Contact Email'] }}`
  - Subject: “Follow Up 1/2/3”
  - Body: `{{ $json.text }}`
  - appendAttribution: false
  - emailType: text
- **Credentials:** Gmail OAuth2.
- **Edge cases:**
  - Gmail API quota / sending limits.
  - Invalid emails cause hard failures unless “Continue On Fail” is enabled (not shown).
- **Version:** 2.1

**8) Update CRM (Follow Up 1/2/3)**
- **Type / role:** `Google Sheets` — updates follow-up counters and last date.
- **Config:**
  - Operation: `update`
  - Match: `Contact Email`
  - Sets:
    - `Follow Ups` to `1/2/3`
    - `Sequence Last Date` to `{{$now.toFormat('MM-dd-yy')}}`
- **Edge cases:**
  - If multiple CRM rows share same email, multiple updates may occur.
  - Date format must match the Filter node’s expected parsing.
- **Version:** 4.7

---

### Block 3 — Concierge Agent (Calendar booking → CRM update)

**Overview:** Listens for newly created Google Calendar events. If the organizer matches a specific display name, it updates the CRM record of the first attendee to “Call Scheduled? = true”; otherwise does nothing.

**Nodes involved:**
- Google Calendar Trigger
- If
- Update CRM (Call Scheduled)
- No Operation, do nothing

#### Sticky Note coverage
- **Sticky Note2:** “# Flow 3: Concierge Agent — Updates CRM when someone books a call”

#### Node details

**1) Google Calendar Trigger**
- **Type / role:** `Google Calendar Trigger` — entry point; detects event creation.
- **Config:**
  - triggerOn: `eventCreated`
  - pollTimes: every minute
  - calendarId: `user@example.com` (label “Test”)
- **Output:** → **If**
- **Edge cases:**
  - Polling every minute can hit API quotas in large calendars.
  - Some event types may not include attendees; downstream assumes `attendees[0].email`.
- **Version:** 1

**2) If**
- **Type / role:** `If` — gatekeeper condition.
- **Condition:** `{{$json.organizer.displayName}} == "Test"`
- **Outputs:**
  - True → **Update CRM (Call Scheduled)**
  - False → **No Operation, do nothing**
- **Edge cases:** Organizer displayName may differ (user settings, delegated calendars), causing missed updates.
- **Version:** 2.2

**3) Update CRM (Call Scheduled)**
- **Type / role:** `Google Sheets` — updates CRM to stop follow-ups.
- **Config:**
  - Operation: `update`
  - Match: `Contact Email`
  - Sets:
    - `Contact Email = {{$json.attendees[0].email}}`
    - `Call Scheduled? = "true"`
- **Edge cases / risks:**
  - If attendees array is empty or first attendee is not the lead, wrong CRM record may be updated or node will error.
  - String `"true"` must align with Follow-Up Agent filter expecting `"false"`/`"true"`.
- **Version:** 4.7

**4) No Operation, do nothing**
- **Type / role:** `NoOp` — explicit “do nothing”.
- **Config:** None.
- **Version:** 1

---

### Block 4 — No-Show Agent (No-show email + status update)

**Overview:** Every 6 hours, finds CRM entries marked as not having shown up (“No”), sends a rescheduling email, and updates the CRM field to “Followed up” to avoid repeat sends.

**Nodes involved:**
- Schedule Trigger (No shows)
- Check for "No Shows"
- Send No-Show Follow-Up
- Update No-Show Status

#### Sticky Note coverage
- **Sticky Note3:** “# Flow 4: No-Show Agent — Emails leads who miss calls”

#### Node details

**1) Schedule Trigger (No shows)**
- **Type / role:** `Schedule Trigger` — entry point.
- **Config:** Every 6 hours.
- **Output:** → **Check for "No Shows"**
- **Version:** 1.2

**2) Check for "No Shows"**
- **Type / role:** `Google Sheets` — selects no-show records.
- **Config:**
  - Filter: `Showed Call? = "No"`
- **Output:** → **Send No-Show Follow-Up**
- **Edge cases:** If “Showed Call?” uses other values (e.g., FALSE, “no-show”), these are missed.
- **Version:** 4.7

**3) Send No-Show Follow-Up**
- **Type / role:** `Gmail` — sends reschedule email.
- **Config:**
  - To: `{{$json['Contact Email']}}`
  - Subject: “Quick follow-up on our call”
  - Body includes inline expression `Hi {{ $json['First Name'] }}, ...` and a calendar link.
  - appendAttribution: false
- **Output:** → **Update No-Show Status**
- **Version:** 2.1

**4) Update No-Show Status**
- **Type / role:** `Google Sheets` — marks record as already followed up.
- **Config:**
  - Operation: `appendOrUpdate`
  - Match: `Contact Email`
  - Sets:
    - `Showed Call? = "Followed up"`
    - `Contact Email` from the “Check for "No Shows"” item
- **Edge cases:** appendOrUpdate could append new rows if matching fails (e.g., whitespace/case differences).
- **Version:** 4.7

---

## 3. Summary Table (All Nodes)

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Block label/comment |  |  | # Flow 1: CRM Agent\nMoves leads from Lead List to CRM |
| Schedule Trigger | Schedule Trigger | Daily trigger for CRM import |  | Email List | # Flow 1: CRM Agent\nMoves leads from Lead List to CRM |
| Email List | Google Sheets | Read Lead List rows | Schedule Trigger | Loop Over Items | # Flow 1: CRM Agent\nMoves leads from Lead List to CRM |
| Loop Over Items | Split In Batches | Iterate lead rows | Email List; Finalize Contact Record in Sheet | Update CRM; Count | # Flow 1: CRM Agent\nMoves leads from Lead List to CRM |
| Update CRM | Google Sheets | Upsert lead into CRM | Loop Over Items | Get row(s) from CRM | # Flow 1: CRM Agent\nMoves leads from Lead List to CRM |
| Get row(s) from CRM | Google Sheets | Fetch CRM row by email (row_number) | Update CRM | Wait | # Flow 1: CRM Agent\nMoves leads from Lead List to CRM |
| Wait | Wait | Delay before final upsert | Get row(s) from CRM | Finalize Contact Record in Sheet | # Flow 1: CRM Agent\nMoves leads from Lead List to CRM |
| Finalize Contact Record in Sheet | Google Sheets | Set ID=row_number (match by email) | Wait | Loop Over Items | # Flow 1: CRM Agent\nMoves leads from Lead List to CRM |
| Count | Summarize | Count imported leads for deletion | Loop Over Items | Reset email list | # Flow 1: CRM Agent\nMoves leads from Lead List to CRM |
| Reset email list | Google Sheets | Delete imported rows from Lead List | Count |  | # Flow 1: CRM Agent\nMoves leads from Lead List to CRM |
| Sticky Note1 | Sticky Note | Block label/comment |  |  | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| Schedule Trigger (Follow Ups) | Schedule Trigger | Trigger follow-up cycle |  | Get rows from CRM | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| Get rows from CRM | Google Sheets | Select active sequence leads | Schedule Trigger (Follow Ups) | Filter | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| Filter | Filter | Enforce 48h + max 3 follow-ups | Get rows from CRM | Which follow-up phase? | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| Which follow-up phase? | Switch | Route to FU1/FU2/FU3 | Filter | Follow Up 1; Follow Up 2; Follow Up 3 | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| OpenAI Follow Up | OpenAI Chat Model (LangChain) | LLM provider for chains |  | Follow Up 1/2/3 (ai_languageModel) | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| Follow Up 1 | LangChain LLM Chain | Personalize template #1 | Which follow-up phase? | Send First Follow Up | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| Send First Follow Up | Gmail | Send FU1 email | Follow Up 1 | Update CRM (Follow Up 1) | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| Update CRM (Follow Up 1) | Google Sheets | Set Follow Ups=1 + last date | Send First Follow Up |  | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| Follow Up 2 | LangChain LLM Chain | Personalize template #2 | Which follow-up phase? | Send Second Follow Up | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| Send Second Follow Up | Gmail | Send FU2 email | Follow Up 2 | Update CRM (Follow Up 2) | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| Update CRM (Follow Up 2) | Google Sheets | Set Follow Ups=2 + last date | Send Second Follow Up |  | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| Follow Up 3 | LangChain LLM Chain | Personalize template #3 | Which follow-up phase? | Send Third Follow Up | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| Send Third Follow Up | Gmail | Send FU3 email | Follow Up 3 | Update CRM (Follow Up 3) | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| Update CRM (Follow Up 3) | Google Sheets | Set Follow Ups=3 + last date | Send Third Follow Up |  | # Flow 2: Follow-Up Agent\nSends 3 personalized follow-up emails every 2 days |
| Sticky Note2 | Sticky Note | Block label/comment |  |  | # Flow 3: Concierge Agent\nUpdates CRM when someone books a call |
| Google Calendar Trigger | Google Calendar Trigger | Detect newly created events |  | If | # Flow 3: Concierge Agent\nUpdates CRM when someone books a call |
| If | If | Check organizer criteria | Google Calendar Trigger | Update CRM (Call Scheduled); No Operation, do nothing | # Flow 3: Concierge Agent\nUpdates CRM when someone books a call |
| Update CRM (Call Scheduled) | Google Sheets | Mark call scheduled in CRM | If (true) |  | # Flow 3: Concierge Agent\nUpdates CRM when someone books a call |
| No Operation, do nothing | NoOp | Do nothing branch | If (false) |  | # Flow 3: Concierge Agent\nUpdates CRM when someone books a call |
| Sticky Note3 | Sticky Note | Block label/comment |  |  | # Flow 4: No-Show Agent\nEmails leads who miss calls |
| Schedule Trigger (No shows) | Schedule Trigger | Periodic no-show check |  | Check for "No Shows" | # Flow 4: No-Show Agent\nEmails leads who miss calls |
| Check for "No Shows" | Google Sheets | Find no-show leads | Schedule Trigger (No shows) | Send No-Show Follow-Up | # Flow 4: No-Show Agent\nEmails leads who miss calls |
| Send No-Show Follow-Up | Gmail | Email reschedule prompt | Check for "No Shows" | Update No-Show Status | # Flow 4: No-Show Agent\nEmails leads who miss calls |
| Update No-Show Status | Google Sheets | Mark no-show as followed up | Send No-Show Follow-Up |  | # Flow 4: No-Show Agent\nEmails leads who miss calls |
| Sticky Note4 | Sticky Note | Global description & links |  |  | # Sales Agent\nThis n8n template is a complete, automated sales operating system... (includes links) |

---

## 4. Reproducing the Workflow from Scratch (Manual Build Steps)

### Prerequisites (before building)
1. **Create/duplicate two Google Sheets:**
   - **Lead List** spreadsheet with columns at minimum:
     - First Name, Last Name, Role, Company Name, Company Address, Company Industry, Website (optional), Contact Phone(optional), Contact Email
   - **CRM** spreadsheet with columns at minimum:
     - ID, First Name, Last Name, Role, Company Name, Company Address, Company Industry, Website (optional), Contact Phone(optional), Contact Email,
     - Started Sequence?, Sequence Last Date, Follow Ups, Call Scheduled?, Showed Call?
2. Ensure CRM columns store booleans consistently as **strings** `"true"` / `"false"` if you keep the workflow as-is.

### Credentials to set up in n8n
3. Add credentials:
   - **Google Sheets OAuth2** (access to both spreadsheets)
   - **Gmail OAuth2**
   - **Google Calendar OAuth2**
   - **OpenAI** credential (for LangChain OpenAI Chat Model node)

---

### Flow 1: CRM Agent
4. Add **Schedule Trigger**:
   - Interval: every **24 hours**
5. Add **Google Sheets** node “Email List”:
   - Operation: **Read/Get Many** rows (use the UI equivalent)
   - Select Lead List document + Sheet1
6. Connect: Schedule Trigger → Email List
7. Add **Split In Batches** node “Loop Over Items”:
   - Keep defaults (or set batch size, e.g., 10–50 to avoid rate limits)
8. Connect: Email List → Loop Over Items
9. Add **Google Sheets** node “Update CRM”:
   - Document: CRM
   - Operation: **Append or Update**
   - Matching column: **ID**
   - Map columns from Lead List fields to CRM fields
   - Set `ID` to an expression that evaluates to `null` (so it appends): `={{ null }}`
10. Connect: Loop Over Items → Update CRM
11. Add **Google Sheets** node “Get row(s) from CRM”:
   - Document: CRM
   - Operation: **Get Many / Read with filter**
   - Filter: Contact Email equals `={{ $json['Contact Email'] }}`
12. Connect: Update CRM → Get row(s) from CRM
13. Add **Wait** node:
   - Amount: 15 (confirm unit in UI; set to minutes if that was intended)
14. Connect: Get row(s) from CRM → Wait
15. Add **Google Sheets** node “Finalize Contact Record in Sheet”:
   - Operation: **Append or Update**
   - Matching column: **Contact Email**
   - Set:
     - ID = `={{ $json.row_number }}`
     - Contact Email = `={{ $json['Contact Email'] }}`
16. Connect: Wait → Finalize Contact Record in Sheet
17. Connect: Finalize Contact Record in Sheet → Loop Over Items (to continue batching)
18. Add **Summarize** node “Count”:
   - Summarize field: Contact Email (to produce `count_Contact_Email`)
19. Connect: Loop Over Items (second output path as per your design) → Count
20. Add **Google Sheets** node “Reset email list”:
   - Document: Lead List
   - Operation: **Delete**
   - Number to delete = `={{ $json.count_Contact_Email }}`
21. Connect: Count → Reset email list

---

### Flow 2: Follow-Up Agent
22. Add **Schedule Trigger** “Schedule Trigger (Follow Ups)”:
   - Configure the desired frequency (intended: check daily; logic sends only if 48h elapsed)
23. Add **Google Sheets** “Get rows from CRM”:
   - Filter: Started Sequence? = `true`
   - Filter: Call Scheduled? = `false`
24. Connect: Trigger (Follow Ups) → Get rows from CRM
25. Add **Filter** node:
   - Condition 1: Sequence Last Date <= `$now.minus({days:2}).format('MM-dd-yy')`
   - Condition 2: Follow Ups < 3
26. Connect: Get rows from CRM → Filter
27. Add **Switch** node “Which follow-up phase?”:
   - Rule to FU1: Follow Ups < 1
   - Rule to FU2: Follow Ups < 2
   - Rule to FU3: Follow Ups < 3
28. Connect: Filter → Switch
29. Add **OpenAI Chat Model** node “OpenAI Follow Up”:
   - Model: `gpt-4.1-nano` (or an available equivalent)
30. Add three **LangChain LLM Chain** nodes “Follow Up 1/2/3”:
   - Paste each email template text
   - Add the instruction message listing variables (First Name, Role, Company Name, Industry, Calendar Link)
   - Ensure they are configured to use the **OpenAI Follow Up** model (AI connection)
31. Connect: OpenAI Follow Up → Follow Up 1/2/3 via AI model connection.
32. Connect: Switch outputs → respective Follow Up nodes.
33. Add three **Gmail** nodes “Send First/Second/Third Follow Up”:
   - To = CRM Contact Email expression
   - Subject = Follow Up 1/2/3
   - Body = `={{ $json.text }}`
34. Connect each Follow Up node → corresponding Gmail node.
35. Add three **Google Sheets** update nodes “Update CRM (Follow Up 1/2/3)”:
   - Operation: **Update**
   - Match by Contact Email
   - Set Follow Ups to 1/2/3 (string is acceptable if your CRM stores strings)
   - Set Sequence Last Date to `={{ $now.toFormat('MM-dd-yy') }}`
36. Connect each Gmail node → corresponding CRM update node.

---

### Flow 3: Concierge Agent
37. Add **Google Calendar Trigger**:
   - Trigger: event created
   - Poll: every minute (or a less frequent interval)
   - Calendar: select your booking calendar
38. Add **If** node:
   - Condition: organizer.displayName equals your expected value (replace “Test”)
39. Connect: Calendar Trigger → If
40. Add **Google Sheets** node “Update CRM (Call Scheduled)”:
   - Operation: Update
   - Match: Contact Email
   - Contact Email = `={{ $json.attendees[0].email }}`
   - Call Scheduled? = `"true"`
41. Connect: If (true) → Update CRM (Call Scheduled)
42. Add **NoOp** node and connect: If (false) → NoOp

---

### Flow 4: No-Show Agent
43. Add **Schedule Trigger (No shows)**:
   - Every 6 hours
44. Add **Google Sheets** node “Check for ‘No Shows’”:
   - Filter: Showed Call? = “No”
45. Connect: Trigger → Check for “No Shows”
46. Add **Gmail** node “Send No-Show Follow-Up”:
   - To = `={{ $json['Contact Email'] }}`
   - Subject = “Quick follow-up on our call”
   - Body includes name + calendar link (as in the workflow)
47. Connect: Check for “No Shows” → Send No-Show Follow-Up
48. Add **Google Sheets** node “Update No-Show Status”:
   - Operation: Append or Update
   - Match: Contact Email
   - Set Showed Call? = “Followed up”
49. Connect: Send No-Show Follow-Up → Update No-Show Status

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “# Sales Agent … complete, automated sales operating system … divided into four autonomous processes …” | Sticky Note content describing intent and usage (covers the whole workflow) |
| “Ask in the n8n Forum” | https://community.n8n.io/ |
| “shoot me a DM on LinkedIn” | http://linkedin.com/in/vincentthenguyen/ |
| Calendar link used in follow-ups | https://calendar.app.google/iSiNxcUzP2ajSMtT8 |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.