Send personalized hotel pre-arrival messages using OpenAI, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/send-personalized-hotel-pre-arrival-messages-using-openai--google-sheets-and-slack-12948


# Send personalized hotel pre-arrival messages using OpenAI, Google Sheets and Slack

## 1. Workflow Overview

**Purpose:** Automate a premium hotel guest pre-arrival experience. Every day, the workflow pulls guest/reservation data from Google Sheets, updates/merges guest profiles, detects guests arriving within **2 days**, generates a **personalized welcome message** with OpenAI, and delivers it via **Slack** (and logs it) or via **email fallback**.

**Target use cases:** Boutique hotels, resorts, short-term rental operators, property managers who want consistent, personalized pre-arrival messaging based on preferences, allergies, and special occasions.

### 1.1 Scheduled Intake (Google Sheets)
Runs on a daily schedule and loads guest rows from the “master” sheet.

### 1.2 Smart Profile Management (merge vs. create)
Determines whether a guest already exists (based on `guest_id`). Returning guests are merged (including deduplicating allergies, incrementing visit count); first-timers get a new profile.

### 1.3 Pre-arrival Window Detection (2-day filter)
Computes `days_until_checkin` and filters only those arriving in the next 0–2 days.

### 1.4 AI Message Generation (OpenAI)
Builds a concierge-style prompt from guest/profile fields and requests a short message with strict constraints (no emojis/questions, 2–4 sentences).

### 1.5 Delivery + Logging (Slack/Email + Sheets log)
If a message is generated, it is sent to Slack and logged to Sheet3; otherwise it sends an email fallback.

### 1.6 Error Monitoring (Slack alert)
Any workflow error triggers a Slack alert to a designated channel.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Data Retrieval
**Overview:** Triggers the workflow daily and retrieves guest/profile rows from Google Sheets (Sheet1). This data is the basis for profile management and messaging.

**Nodes Involved:**  
- Daily Check Schedule  
- Fetch Guest Profiles

#### Node: Daily Check Schedule
- **Type / role:** `Schedule Trigger` — entry point that runs on a timed interval.
- **Configuration (interpreted):** Interval-based schedule (JSON shows `rule.interval` with defaults; sticky note recommends “daily at 9 AM”).
- **Inputs/Outputs:** No inputs. Output → **Fetch Guest Profiles**.
- **Failure/edge cases:** Misconfigured timezone/schedule; workflow inactive (`active: false`) means it won’t run until enabled.
- **Version notes:** `typeVersion 1.3`—schedule UI differs slightly across n8n versions.

#### Node: Fetch Guest Profiles
- **Type / role:** `Google Sheets` — reads guest profile/reservation records.
- **Configuration (interpreted):**
  - Spreadsheet: “Guest Database” (`[YOUR_SPREADSHEET_ID]`)
  - Sheet: “Sheet1” (gid=0)
  - Operation is not explicitly shown in JSON; by default this node typically performs a **Read/Get Many** depending on UI selection. Given the workflow, intent is “fetch all reservations/profiles”.
- **Key data fields expected downstream:** `guest_id`, `guest_key`, `name`, `email`, `phone`, `room_preference`, `food_allergies`, `special_occasion`, `check_in_date`.
- **Inputs/Outputs:** Input from schedule; output → **Guest Exists?**
- **Credentials:** Google Sheets OAuth2.
- **Failure/edge cases:** OAuth scope issues, spreadsheet ID invalid, sheet missing/renamed, quota limits, empty sheet output causing later expressions to fail.
- **Version notes:** `typeVersion 4.7` (modern Google Sheets node).

---

### Block 2 — Smart Profile Management
**Overview:** Decides whether a row belongs to a returning guest or a first-time guest. Returning guests get merged data and incremented `visit_count`; new guests get an initialized profile.

**Nodes Involved:**  
- Guest Exists?  
- Merge Returning Guest Data  
- Create First-Time Guest Profile  
- Save Updated Profile

#### Node: Guest Exists?
- **Type / role:** `If` — branching based on presence of `guest_id`.
- **Configuration (interpreted):** Condition: `guest_id` **is not empty** (`={{ $json.guest_id }}`).
- **Inputs/Outputs:**
  - Input from **Fetch Guest Profiles**
  - **True** → Merge Returning Guest Data
  - **False** → Create First-Time Guest Profile
- **Failure/edge cases:** If Sheet1 uses a different identifier (e.g., empty `guest_id` but still returning), logic misclassifies guests.

#### Node: Merge Returning Guest Data
- **Type / role:** `Function` — builds an updated profile by combining incoming data with an “existing profile”.
- **Configuration (interpreted):**
  - Reads the current item (`incoming = $input.first().json`)
  - Reads “existing” data as `$('Fetch Guest Profiles').first().json`
  - Merges allergies using a Set dedupe pattern
  - Increments `visit_count`
  - Adds `last_visit_date` and sets `profile_id: existing.id`
- **Key expressions/variables:**
  - `$('Fetch Guest Profiles').first().json` (critical: this does **not** actually look up a matching guest; it always takes the *first row* from Fetch Guest Profiles)
  - `mergedAllergies = [...new Set([...existingAllergies, ...newAllergies])]`
- **Inputs/Outputs:** Input from **Guest Exists? (true)** → output to **Save Updated Profile**
- **Failure/edge cases / correctness concerns:**
  - **Major data integrity risk:** “existing” profile is not selected by guest key/id; it is always the first row of Sheet1. Returning guests may be merged with the wrong profile.
  - If `food_allergies` is stored as a comma-separated string in Sheets, `.join()` and array merge behavior may break.
  - `existing.id` may not exist (depends on Google Sheets node output format), making `profile_id` undefined.
  - Function node may throw if referenced node has no items.
- **Version notes:** Function node is legacy-ish; Code node is often preferred in newer n8n, but Function still works depending on instance settings.

#### Node: Create First-Time Guest Profile
- **Type / role:** `Function` — creates a normalized profile object for new guests.
- **Configuration (interpreted):**
  - Sets `visit_count: 1`
  - Initializes timestamps `created_at`, `last_visit_date`
  - Ensures `food_allergies` defaults to `[]`
- **Inputs/Outputs:** Input from **Guest Exists? (false)** → output to **Save Updated Profile**
- **Failure/edge cases:** Same data typing issues (Sheets strings vs arrays), missing required fields.

#### Node: Save Updated Profile
- **Type / role:** `Google Sheets` — writes profile updates to Sheet2.
- **Configuration (interpreted):**
  - Operation: **Append or Update**
  - Sheet: “Sheet2” (gid=254632980)
  - Mapping: `autoMapInputData` (relies on incoming JSON keys matching column headers)
- **Inputs/Outputs:** Input from either profile function → output to **Calculate Days Until Check-In**
- **Credentials:** Google Sheets OAuth2.
- **Failure/edge cases:**
  - Append/update requires a reliable matching key; JSON shows `matchingColumns` empty, so updates may not match existing rows and can **append duplicates**.
  - Column/header mismatches lead to missing or miswritten fields.
  - Type conversion disabled; dates may be stored as strings inconsistently.

---

### Block 3 — Pre-Arrival Window Detection
**Overview:** Computes how many days remain until check-in and flags guests arriving within the next 2 days (inclusive).

**Nodes Involved:**  
- Calculate Days Until Check-In  
- Within 2-Day Window?

#### Node: Calculate Days Until Check-In
- **Type / role:** `Function` — date arithmetic and boolean flagging.
- **Configuration (interpreted):**
  - `diffDays = Math.ceil((checkInDate - now) / msPerDay)`
  - Adds:
    - `days_until_checkin: diffDays`
    - `within_window: diffDays <= 2 && diffDays >= 0`
- **Inputs/Outputs:** Input from **Save Updated Profile** → output to **Within 2-Day Window?**
- **Failure/edge cases:**
  - `new Date($json.check_in_date)` can yield **Invalid Date** if Sheets stores in unexpected format (e.g., `DD/MM/YYYY`), producing `NaN`.
  - Timezone effects: a check-in “2 days away” may evaluate as 1 or 3 depending on time-of-day and timezone; using `ceil` tends to include partial days.
  - Past check-ins are excluded by `>= 0`.

#### Node: Within 2-Day Window?
- **Type / role:** `If` — filters to only those flagged `within_window`.
- **Configuration (interpreted):** boolean condition `{{$json.within_window}} == true`
- **Inputs/Outputs:**
  - Input from **Calculate Days Until Check-In**
  - **True branch (index 1 in JSON)** → Generate Personalized Welcome Message
  - **False branch** → stops (no downstream nodes)
- **Failure/edge cases:** If `within_window` is undefined (date parse issues), condition fails and no messages are generated.

---

### Block 4 — AI Message Personalization
**Overview:** Uses OpenAI to generate a short concierge-style pre-arrival message using guest-specific details.

**Nodes Involved:**  
- Generate Personalized Welcome Message  
- Check Message Generated

#### Node: Generate Personalized Welcome Message
- **Type / role:** `OpenAI` — text generation.
- **Configuration (interpreted):**
  - Prompt injects:
    - `name`, `room_preference`, `food_allergies.join(", ")`, `special_occasion`, `visit_count`
  - Output constraints: 2–4 sentences, no emojis, no questions, do not mention systems/visit count/data sources, end with a positive welcome line.
  - Output: “ONLY the message text.”
- **Key expressions/variables:**
  - `{{$json.food_allergies.join(", ")}}` (assumes `food_allergies` is an array)
- **Inputs/Outputs:** Input from **Within 2-Day Window? (true)** → output to **Check Message Generated**
- **Credentials:** OpenAI API key.
- **Failure/edge cases:**
  - If `food_allergies` is a string/null, `.join` throws (expression error) or fails to render.
  - Model/provider configuration not shown; node may default to a specific model. Incompatibility/limits can cause failures (rate limits, 429, timeouts).
  - Output field name expected downstream is `text`; if the node returns a different structure depending on n8n/OpenAI node version, downstream IF may fail.
- **Version notes:** `typeVersion 1`—OpenAI node behavior can vary across n8n versions.

#### Node: Check Message Generated
- **Type / role:** `If` — validates OpenAI returned text.
- **Configuration (interpreted):** Condition: `$json.text` is not empty.
- **Inputs/Outputs:**
  - Input from **Generate Personalized Welcome Message**
  - **True** → Send Slack Notification AND Log Message to Sheet
  - **False** → Send Welcome Email (fallback path)
- **Failure/edge cases:** If OpenAI node output places content elsewhere (e.g., `choices[0].message.content`), `$json.text` will be empty even though a message exists.

---

### Block 5 — Multi-Channel Delivery + Logging
**Overview:** Delivers the message primarily via Slack and logs it to Google Sheets (Sheet3). If message is missing, sends an email.

**Nodes Involved:**  
- Send Slack Notification  
- Log Message to Sheet  
- Send Welcome Email

#### Node: Send Slack Notification
- **Type / role:** `Slack` — posts the generated message to a channel.
- **Configuration (interpreted):**
  - Channel: `guest-notifications` (ID `C09GNB90TED` shown)
  - Text: `={{ $json.text }}`
- **Inputs/Outputs:** Input from **Check Message Generated (true)**. No outgoing connections.
- **Credentials:** Slack token/connection.
- **Failure/edge cases:** Missing scopes (`chat:write`), invalid channel ID, token revoked, Slack API rate limiting.

#### Node: Log Message to Sheet
- **Type / role:** `Google Sheets` — logs the generated text and metadata to Sheet3.
- **Configuration (interpreted):**
  - Operation: **Append or Update**
  - Sheet: “Sheet3” (gid=369348291)
  - Auto-map enabled; schema includes columns like `text`, `index`, `logprobs`, `finish_reason` (these resemble OpenAI response metadata)
- **Inputs/Outputs:** Input from **Check Message Generated (true)**. No outgoing connections.
- **Credentials:** Google Sheets OAuth2.
- **Failure/edge cases:** Same append/update matching risk (matching columns empty → duplicates). Also logging only OpenAI fields may omit guest identifiers unless present in the incoming JSON at this step.

#### Node: Send Welcome Email
- **Type / role:** `Email Send` — fallback delivery method.
- **Configuration (interpreted):**
  - To: `={{$json.email}}`
  - From: `user@example.com` (placeholder)
  - Subject: “Your Upcoming Stay - We're Ready for You!”
  - Body text: `={{ $('Generate Personalized Welcome Message').item.json.text }}`
- **Inputs/Outputs:** Input from **Check Message Generated (false)**. No outgoing connections.
- **Credentials:** SMTP.
- **Failure/edge cases / logic concerns:**
  - **Fallback inconsistency:** This node triggers when `$json.text` is empty, but the email body explicitly references the OpenAI text anyway—so the email may be blank or error.
  - If the current item at this branch does not contain `email`, `toEmail` will be empty → send failure.
  - SMTP configuration errors, blocked port, SPF/DKIM issues, provider rejection.
- **Version notes:** `typeVersion 2`—Email node fields differ from v1.

---

### Block 6 — Error Monitoring
**Overview:** Catches any runtime workflow error and posts a Slack alert so operators can investigate.

**Nodes Involved:**  
- Error Trigger  
- Alert on Workflow Failure

#### Node: Error Trigger
- **Type / role:** `Error Trigger` — secondary entry point invoked when the workflow fails.
- **Configuration (interpreted):** Default error trigger (no extra params).
- **Inputs/Outputs:** No standard input. Output → **Alert on Workflow Failure**
- **Failure/edge cases:** If Slack node fails too, alerts won’t be delivered.

#### Node: Alert on Workflow Failure
- **Type / role:** `Slack` — posts a fixed error alert message to a channel.
- **Configuration (interpreted):**
  - Channel: `workflow-errors` (ID shown as `C09GNB90TED` in JSON; note: same ID as guest-notifications, likely a placeholder/mistake)
  - Text includes a warning header and instruction to check execution log.
- **Inputs/Outputs:** Input from **Error Trigger**. No outgoing connections.
- **Credentials:** Slack token/connection.
- **Failure/edge cases:** Wrong channel ID means alerts go to wrong place; missing `chat:write` scope; rate limits.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Documentation / annotation |  |  | ## 🏨 AI-Powered Guest Pre-Arrival Experience Automation using OpenAI, Google Sheets & Slack… (how it works + setup steps + use case) |
| Sticky Note1 | Sticky Note | Documentation / annotation |  |  | ## 📅 Scheduled Data Retrieval… |
| Sticky Note2 | Sticky Note | Documentation / annotation |  |  | ## 👤 Smart Profile Management… |
| Sticky Note3 | Sticky Note | Documentation / annotation |  |  | ## ⏰ Pre-Arrival Window Detection… |
| Sticky Note4 | Sticky Note | Documentation / annotation |  |  | ## 🤖 AI Message Personalization… |
| Sticky Note5 | Sticky Note | Documentation / annotation |  |  | ## 📬 Multi-Channel Delivery… |
| Sticky Note6 | Sticky Note | Documentation / annotation |  |  | ## ⚠️ Error Monitoring… |
| Sticky Note7 | Sticky Note | Documentation / annotation |  |  | ## 🔐 Required Credentials… |
| Daily Check Schedule | Schedule Trigger | Daily workflow start | — | Fetch Guest Profiles | ## 📅 Scheduled Data Retrieval… |
| Fetch Guest Profiles | Google Sheets | Read guest/reservation rows from Sheet1 | Daily Check Schedule | Guest Exists? | ## 📅 Scheduled Data Retrieval… |
| Guest Exists? | IF | Branch returning vs new guest (guest_id present) | Fetch Guest Profiles | Merge Returning Guest Data; Create First-Time Guest Profile | ## 👤 Smart Profile Management… |
| Merge Returning Guest Data | Function | Merge profile data + dedupe allergies + increment visits | Guest Exists? (true) | Save Updated Profile | ## 👤 Smart Profile Management… |
| Create First-Time Guest Profile | Function | Create normalized first-time profile | Guest Exists? (false) | Save Updated Profile | ## 👤 Smart Profile Management… |
| Save Updated Profile | Google Sheets | Append/update into Sheet2 | Merge Returning Guest Data; Create First-Time Guest Profile | Calculate Days Until Check-In | ## 👤 Smart Profile Management… |
| Calculate Days Until Check-In | Function | Compute days until arrival + within_window flag | Save Updated Profile | Within 2-Day Window? | ## ⏰ Pre-Arrival Window Detection… |
| Within 2-Day Window? | IF | Filter to arrivals within 0–2 days | Calculate Days Until Check-In | Generate Personalized Welcome Message (true branch) | ## ⏰ Pre-Arrival Window Detection… |
| Generate Personalized Welcome Message | OpenAI | Create concierge-style pre-arrival text | Within 2-Day Window? | Check Message Generated | ## 🤖 AI Message Personalization… |
| Check Message Generated | IF | Validate OpenAI `text` output exists | Generate Personalized Welcome Message | Send Slack Notification + Log Message to Sheet (true); Send Welcome Email (false) | ## 📬 Multi-Channel Delivery… |
| Send Slack Notification | Slack | Post message to Slack channel | Check Message Generated (true) | — | ## 📬 Multi-Channel Delivery… |
| Log Message to Sheet | Google Sheets | Log OpenAI response to Sheet3 | Check Message Generated (true) | — | ## 📬 Multi-Channel Delivery… |
| Send Welcome Email | Email Send | Email fallback delivery | Check Message Generated (false) | — | ## 📬 Multi-Channel Delivery… |
| Error Trigger | Error Trigger | Start error-handling path on failure | — | Alert on Workflow Failure | ## ⚠️ Error Monitoring… |
| Alert on Workflow Failure | Slack | Send failure alert to Slack | Error Trigger | — | ## ⚠️ Error Monitoring… |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: *AI-Powered Guest Pre-Arrival Experience Automation using OpenAI, Google Sheets & Slack* (or your desired title).
   - Keep workflow **inactive** until tested.

2. **Add credentials (required)**
   - **Google Sheets OAuth2**: authorize read/write access to the target spreadsheet.
   - **OpenAI API**: add API key credential.
   - **Slack API**: connect workspace with permission to post messages to target channels.
   - **SMTP**: configure host/port/user/password (or provider-based SMTP).

3. **Create the schedule entry node**
   - Add node: **Schedule Trigger** named **Daily Check Schedule**
   - Configure: run **daily** (recommended 9:00 AM) in your chosen timezone.

4. **Fetch guest data from Google Sheets**
   - Add node: **Google Sheets** named **Fetch Guest Profiles**
   - Select document: your “Guest Database” spreadsheet.
   - Select sheet: **Sheet1** (master guest/reservation data).
   - Operation: configure to **read all rows** (e.g., “Get Many”).
   - Connect: **Daily Check Schedule → Fetch Guest Profiles**

5. **Branch by returning vs first-time guest**
   - Add node: **IF** named **Guest Exists?**
   - Condition (String): `{{$json.guest_id}}` **is not empty**
   - Connect: **Fetch Guest Profiles → Guest Exists?**

6. **Returning guest merge logic**
   - Add node: **Function** named **Merge Returning Guest Data**
   - Paste logic equivalent to:
     - Merge incoming fields with existing fields
     - Deduplicate `food_allergies`
     - Increment `visit_count`
     - Set `last_visit_date` timestamp
   - Connect: **Guest Exists? (true) → Merge Returning Guest Data**
   - Important: implement a *real lookup* for “existing profile” (e.g., search Sheet1/Sheet2 by `guest_id` or `guest_key`) rather than using the first row.

7. **First-time guest profile creation**
   - Add node: **Function** named **Create First-Time Guest Profile**
   - Configure to output:
     - `visit_count: 1`
     - `created_at`, `last_visit_date`
     - `food_allergies: []` when missing
   - Connect: **Guest Exists? (false) → Create First-Time Guest Profile**

8. **Write profile to Sheet2**
   - Add node: **Google Sheets** named **Save Updated Profile**
   - Document: same spreadsheet
   - Sheet: **Sheet2**
   - Operation: **Append or Update**
   - Mapping: Auto-map input data
   - Configure **matching columns** (recommended): `guest_id` or `guest_key` so updates don’t create duplicates.
   - Connect:
     - **Merge Returning Guest Data → Save Updated Profile**
     - **Create First-Time Guest Profile → Save Updated Profile**

9. **Compute days until check-in**
   - Add node: **Function** named **Calculate Days Until Check-In**
   - Compute:
     - `days_until_checkin`
     - `within_window` (true if 0–2 days)
   - Connect: **Save Updated Profile → Calculate Days Until Check-In**

10. **Filter by pre-arrival window**
   - Add node: **IF** named **Within 2-Day Window?**
   - Boolean condition: `{{$json.within_window}}` equals `true`
   - Connect: **Calculate Days Until Check-In → Within 2-Day Window?**

11. **Generate message with OpenAI**
   - Add node: **OpenAI** named **Generate Personalized Welcome Message**
   - Configure prompt using fields:
     - name, room_preference, food_allergies, special_occasion, visit_count
   - Ensure `food_allergies` is an array before using `.join(", ")` (or cast safely).
   - Connect: **Within 2-Day Window? (true) → Generate Personalized Welcome Message**

12. **Validate message exists**
   - Add node: **IF** named **Check Message Generated**
   - String condition: `{{$json.text}}` is not empty (or adapt to the actual OpenAI output field).
   - Connect: **Generate Personalized Welcome Message → Check Message Generated**

13. **Slack delivery**
   - Add node: **Slack** named **Send Slack Notification**
   - Operation: post message to a channel (e.g., `guest-notifications`)
   - Text: `{{$json.text}}`
   - Connect: **Check Message Generated (true) → Send Slack Notification**

14. **Log message to Sheet3**
   - Add node: **Google Sheets** named **Log Message to Sheet**
   - Sheet: **Sheet3**
   - Operation: Append (or Append/Update with matching keys such as guest_id + check_in_date)
   - Ensure you log identifiers (guest_id, guest_key, check_in_date) in addition to the OpenAI text to make logs useful.
   - Connect: **Check Message Generated (true) → Log Message to Sheet**

15. **Email fallback**
   - Add node: **Email Send** named **Send Welcome Email**
   - To: `{{$json.email}}`
   - Subject: “Your Upcoming Stay - We're Ready for You!”
   - Body: use the generated message text (ensure it’s available on this branch; often you’d pass it along rather than referencing another node)
   - Connect: **Check Message Generated (false) → Send Welcome Email**

16. **Error handling**
   - Add node: **Error Trigger** named **Error Trigger**
   - Add node: **Slack** named **Alert on Workflow Failure**
   - Configure channel (e.g., `workflow-errors`) and a fixed alert text.
   - Connect: **Error Trigger → Alert on Workflow Failure**

17. **Test end-to-end**
   - Run with a small set of sample rows:
     - One returning guest, one new guest
     - Check-in dates inside and outside the 2-day window
     - Allergies as array vs string (normalize as needed)
   - Enable workflow only after successful tests.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided disclaimer |
| “Replace all example emails and IDs with your own before deployment.” | Credential placeholders and IDs appear in nodes like Email Send (`user@example.com`) and Spreadsheet ID (`[YOUR_SPREADSHEET_ID]`). |
| Setup guidance: connect Google Sheets (Sheet1 profiles, Sheet2 updates, Sheet3 message log); add OpenAI credentials; configure Slack + SMTP; schedule daily at 9 AM; test with sample data. | From sticky note content within the workflow |

