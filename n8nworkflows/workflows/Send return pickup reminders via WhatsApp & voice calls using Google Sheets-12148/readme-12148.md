Send return pickup reminders via WhatsApp & voice calls using Google Sheets

https://n8nworkflows.xyz/workflows/send-return-pickup-reminders-via-whatsapp---voice-calls-using-google-sheets-12148


# Send return pickup reminders via WhatsApp & voice calls using Google Sheets

## 1. Workflow Overview

**Title:** Send return pickup reminders via WhatsApp & voice calls using Google Sheets

**Purpose / Use case:**  
This workflow automates daily return-pickup reminders for e-commerce orders. At **10:00 AM**, it queries a **Google Sheet** for pickups scheduled **today** with **Status = Pending**, builds personalized **WhatsApp** and **voice call** messages, sends both via **Twilio**, then updates the sheet row to **“Reminder Sent”**.

### 1.1 Scheduled Trigger (Entry)
Runs once daily at a fixed hour.

### 1.2 Data Retrieval & Message Preparation
Fetches rows matching today’s pickup date + pending status, then generates message text fields.

### 1.3 Per-Customer Notification Loop
Processes each pickup record one-by-one: WhatsApp first, then a Twilio voice call.

### 1.4 Status Update & Loop Continuation
Writes back “Reminder Sent” into the Google Sheet, then continues to the next item.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Trigger
**Overview:** Triggers the workflow every day at 10 AM, initiating the retrieval of pickups due today.

**Nodes involved:**
- Schedule Trigger

#### Node: Schedule Trigger
- **Type / role:** `Schedule Trigger` — time-based entry node.
- **Configuration (interpreted):** Runs on an interval rule that triggers at **hour 10** (local time of the n8n instance).
- **Key expressions / variables:** None.
- **Input / output:**
  - **Input:** None (entry node).
  - **Output →** `Get Pickup Scheduled Data`
- **Version specifics:** typeVersion **1.3**
- **Edge cases / failures:**
  - Timezone mismatch: “10 AM” is based on server/workflow timezone settings, not the Google Sheet’s locale.
  - If n8n is down at trigger time, run may be missed unless n8n scheduling catch-up is configured externally.

---

### Block 2 — Data Retrieval & Message Preparation
**Overview:** Pulls all rows where Pickup Date is today and Status is Pending, then constructs WhatsApp text and a voice-friendly message per row.

**Nodes involved:**
- Get Pickup Scheduled Data
- Create WhatsApp & Voice Call Message

#### Node: Get Pickup Scheduled Data
- **Type / role:** `Google Sheets` — reads rows from a spreadsheet.
- **Configuration (interpreted):**
  - **Document:** “Return Pickup Data” (Spreadsheet ID `1P-rwG-...`)
  - **Sheet:** “Sheet1” (gid=0)
  - **Filters:**
    - `Pickup Date` equals `{{$today.format('MM-dd-yyyy')}}`
    - `Status` equals `"Pending"`
- **Key expressions / variables:**
  - `{{$today.format('MM-dd-yyyy')}}` to match today’s date string.
- **Input / output:**
  - **Input ←** `Schedule Trigger`
  - **Output →** `Create WhatsApp & Voice Call Message`
- **Version specifics:** typeVersion **4.7**
- **Edge cases / failures:**
  - **Date format mismatch**: Sheet values must exactly match `MM-dd-yyyy` (e.g., `01-07-2026`). If the sheet stores real date types or another format (e.g., `dd/MM/yyyy`), the filter will return no rows.
  - Google auth/permissions issues (OAuth scope, revoked access, sharing).
  - Empty result set: downstream nodes should handle “no items” gracefully (SplitInBatches will simply have nothing to process).

#### Node: Create WhatsApp & Voice Call Message
- **Type / role:** `Code` — transforms each row into message payload fields.
- **Configuration (interpreted):**
  - **Mode:** “Run once for each item”
  - Reads columns from the incoming row:
    - `Customer Name`, `Phone Number`, `Product`, `Customer Address`, `Return Reason`, `Pickup Date`, `Order ID`
  - Creates:
    - `message_text`: WhatsApp-ready multi-line reminder text
    - `voice_message`: voice-call friendly sentence string
    - plus passthrough fields: `phone`, `order_id`, `customer_name`, `product`, `row_id`
- **Key expressions / variables:**
  - Uses `$json["Customer Name"]`-style column access.
  - Returns an object with:
    - `message_text`
    - `voice_message`
    - `phone`
    - etc.
- **Input / output:**
  - **Input ←** `Get Pickup Scheduled Data`
  - **Output →** `Loop Over Items`
- **Version specifics:** typeVersion **2**
- **Edge cases / failures:**
  - Missing/renamed columns in the sheet cause `undefined` values in messages.
  - Address/return reason may contain characters that read awkwardly via TTS; consider sanitizing for voice.
  - `row_id` is returned but the sheet update later uses matching by “Phone Number”, not row id—so duplicates can update the wrong row.

---

### Block 3 — Per-Customer Notification Loop (WhatsApp + Voice)
**Overview:** Iterates through each pickup record sequentially, sends a WhatsApp message via Twilio, then places a voice call via Twilio’s REST API using TwiML.

**Nodes involved:**
- Loop Over Items
- Send WhatsApp Notification
- Place Voice Call Reminder

#### Node: Loop Over Items
- **Type / role:** `Split In Batches` — controls per-item processing/looping.
- **Configuration (interpreted):**
  - Batch settings not explicitly set (defaults apply). Commonly, this defaults to a batch size of 1 unless changed in UI.
- **Input / output:**
  - **Input ←** `Create WhatsApp & Voice Call Message` and loop-back from `Update Reminder Status`
  - **Output (main index 1) →** `Send WhatsApp Notification`
  - **Output (main index 0):** unused in this workflow (left empty)
- **Version specifics:** typeVersion **3**
- **Edge cases / failures:**
  - If batch size is >1 unexpectedly, downstream “item” referencing (`$('Loop Over Items').item.json...`) can behave differently than expected.
  - With many rows, run time may be long; Twilio rate limits or execution timeouts could occur.

#### Node: Send WhatsApp Notification
- **Type / role:** `Twilio` — sends a WhatsApp message.
- **Configuration (interpreted):**
  - **To:** hard-coded as `+917984780565` (currently not using the per-row phone)
  - **From:** `+1234567890` (placeholder)
  - **Message:** `{{$json.message_text}}`
  - **toWhatsapp:** enabled (sends as WhatsApp)
  - **Credentials:** Twilio API credential
- **Key expressions / variables:**
  - Message body from current item: `{{$json.message_text}}`
- **Input / output:**
  - **Input ←** `Loop Over Items` (each batch item)
  - **Output →** `Place Voice Call Reminder`
- **Version specifics:** typeVersion **1**
- **Edge cases / failures:**
  - **Current misconfiguration:** the **recipient is hard-coded**, so every reminder goes to the same number, not each customer. Typically you want `to` = `={{ $json.phone }}` (or `={{ $json["Phone Number"] }}` if not remapped).
  - WhatsApp via Twilio requires:
    - WhatsApp-enabled sender (“From”, often `whatsapp:+...` depending on Twilio configuration and n8n node behavior)
    - approved templates or session rules (region/account dependent)
  - Auth errors (bad SID/token), sandbox restrictions, unapproved destination numbers.

#### Node: Place Voice Call Reminder
- **Type / role:** `HTTP Request` — calls Twilio REST API to initiate a voice call.
- **Configuration (interpreted):**
  - **Method:** POST
  - **URL:** `https://api.twilio.com/2010-04-01/Accounts/YOUR_AWS_SECRET_KEY_HERE.json`
    - This is not a valid Twilio Calls endpoint as written. Normally it should resemble:
      - `https://api.twilio.com/2010-04-01/Accounts/{AccountSID}/Calls.json`
  - **Auth:** uses predefined credential type `twilioApi`
  - **Body:** form-urlencoded parameters:
    - `Twiml`: inline `<Response><Say ...> ... </Say></Response>` containing `$('Loop Over Items').item.json.voice_message`
    - `To`: `+1234567890` (hard-coded placeholder)
    - `From`: `+1234567890` (hard-coded placeholder)
- **Key expressions / variables:**
  - `Twiml` value contains: `{{ $('Loop Over Items').item.json.voice_message }}`
- **Input / output:**
  - **Input ←** `Send WhatsApp Notification`
  - **Output →** `Update Reminder Status`
- **Version specifics:** typeVersion **4.3**
- **Edge cases / failures:**
  - **Endpoint is incorrect**: will 404 or fail auth/validation. Must be `/Calls.json` and use Account SID.
  - **Hard-coded To/From**: like WhatsApp, it won’t call the customer unless `To` is mapped (e.g., `={{ $json.phone }}`).
  - TwiML injection/escaping: if `voice_message` contains characters that break XML, the request may fail. Consider escaping `&`, `<`, `>`.
  - Voice `Polly.Joanna` implies a Polly voice name; Twilio `<Say voice="">` typically expects Twilio-supported voices (e.g., `alice`), unless using special integrations. This may be invalid depending on Twilio setup.

---

### Block 4 — Status Update & Loop Continuation
**Overview:** After notifications are sent, updates the Google Sheet row’s status to “Reminder Sent” and loops back to process the next pickup.

**Nodes involved:**
- Update Reminder Status

#### Node: Update Reminder Status
- **Type / role:** `Google Sheets` — updates matching rows.
- **Configuration (interpreted):**
  - **Operation:** Update
  - **Matching column:** `Phone Number`
  - **Values set:**
    - `Status` = `"Reminder Sent"`
    - `Phone Number` = `={{ $('Loop Over Items').item.json.phone }}`
  - **Document / Sheet:** same spreadsheet and Sheet1
- **Key expressions / variables:**
  - Uses the current loop item: `$('Loop Over Items').item.json.phone`
- **Input / output:**
  - **Input ←** `Place Voice Call Reminder`
  - **Output →** `Loop Over Items` (to continue batching)
- **Version specifics:** typeVersion **4.7**
- **Edge cases / failures:**
  - **Non-unique matching:** if multiple rows share the same phone number, the update can affect the wrong row(s). Prefer matching by a unique key (Order ID) or using `row_number`.
  - Updating “Phone Number” to itself is redundant; harmless but unnecessary.
  - If WhatsApp/call fails but the workflow continues, status may still be updated unless you add error handling.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Daily entry point at 10 AM | — | Get Pickup Scheduled Data | ## How it works… (Workflow Overview sticky note content applies to overall flow) |
| Get Pickup Scheduled Data | Google Sheets | Fetch today’s “Pending” pickups | Schedule Trigger | Create WhatsApp & Voice Call Message | ## Data Retrieval: Fetches today's pending pickups from Google Sheets and formats notification messages. / ## Google sheet https://docs.google.com/spreadsheets/d/1P-rwG-J_Wqj_SAVIgQC49qzp6bA42v-S5Y9vwBASwDg/edit?usp=sharing |
| Create WhatsApp & Voice Call Message | Code | Build WhatsApp + voice message fields | Get Pickup Scheduled Data | Loop Over Items | ## Data Retrieval: Fetches today's pending pickups from Google Sheets and formats notification messages. |
| Loop Over Items | Split In Batches | Sequentially process each pickup row | Create WhatsApp & Voice Call Message; Update Reminder Status | Send WhatsApp Notification | ## Customer Notifications: Processes each pickup one-by-one, sending WhatsApp message followed by automated voice call reminder. |
| Send WhatsApp Notification | Twilio | Send WhatsApp reminder | Loop Over Items | Place Voice Call Reminder | ## Customer Notifications: Processes each pickup one-by-one, sending WhatsApp message followed by automated voice call reminder. |
| Place Voice Call Reminder | HTTP Request | Initiate Twilio voice call with TwiML | Send WhatsApp Notification | Update Reminder Status | ## Customer Notifications: Processes each pickup one-by-one, sending WhatsApp message followed by automated voice call reminder. |
| Update Reminder Status | Google Sheets | Mark row as “Reminder Sent” | Place Voice Call Reminder | Loop Over Items | ## Status Update: Marks the pickup status as "Reminder Sent" in the sheet, then loops back for the next customer. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Send return pickup reminders via WhatsApp & voice calls using Google Sheets*.

2. **Add node: Schedule Trigger**
   - Type: **Schedule Trigger**
   - Configure: Trigger daily at **10:00** (set the interval rule to hour 10).
   - Ensure the workflow timezone is what you expect (n8n settings).

3. **Add node: Google Sheets → “Get Pickup Scheduled Data”**
   - Type: **Google Sheets**
   - Credentials: **Google Sheets OAuth2** (grant access to the spreadsheet).
   - Document: select the spreadsheet containing return pickup data.
   - Sheet: select **Sheet1** (or your target sheet).
   - Add filters:
     - Column **Pickup Date** equals expression: `{{$today.format('MM-dd-yyyy')}}`
     - Column **Status** equals: `Pending`
   - Connect: **Schedule Trigger → Get Pickup Scheduled Data**

4. **Add node: Code → “Create WhatsApp & Voice Call Message”**
   - Type: **Code**
   - Mode: **Run once for each item**
   - Paste logic that:
     - Reads `Customer Name`, `Phone Number`, `Product`, `Customer Address`, `Return Reason`, `Pickup Date`, `Order ID`
     - Outputs at least:
       - `message_text`
       - `voice_message`
       - `phone`
       - `order_id`
   - Connect: **Get Pickup Scheduled Data → Create WhatsApp & Voice Call Message**

5. **Add node: Split In Batches → “Loop Over Items”**
   - Type: **Split In Batches**
   - Set **Batch Size = 1** (recommended for sequential customer messaging).
   - Connect: **Create WhatsApp & Voice Call Message → Loop Over Items**

6. **Add node: Twilio → “Send WhatsApp Notification”**
   - Type: **Twilio**
   - Credentials: **Twilio API** (Account SID + Auth Token).
   - Enable WhatsApp sending (option is `toWhatsapp: true`).
   - Set:
     - **To:** should be the customer phone, e.g. `={{ $json.phone }}` (replace the hard-coded number)
     - **From:** your Twilio WhatsApp-enabled number (as required by your Twilio setup)
     - **Message:** `={{ $json.message_text }}`
   - Connect: **Loop Over Items → Send WhatsApp Notification**

7. **Add node: HTTP Request → “Place Voice Call Reminder”**
   - Type: **HTTP Request**
   - Authentication: **Predefined credential type → Twilio**
   - Method: **POST**
   - URL (Twilio Calls endpoint):  
     `https://api.twilio.com/2010-04-01/Accounts/<YourAccountSID>/Calls.json`
   - Content-Type: **form-urlencoded**
   - Body parameters:
     - `To` = `={{ $json.phone }}`
     - `From` = your Twilio voice-capable number
     - `Twiml` = an XML string containing a `<Say>` with your voice text, e.g.:  
       `<Response><Say>{{ $json.voice_message }}</Say></Response>`
   - Connect: **Send WhatsApp Notification → Place Voice Call Reminder**

8. **Add node: Google Sheets → “Update Reminder Status”**
   - Type: **Google Sheets**
   - Credentials: same as step 3
   - Operation: **Update**
   - Match on a **unique identifier** if possible:
     - Preferred: match by **Order ID** or **row_number**
     - Current workflow matches by **Phone Number** (works only if unique)
   - Set columns:
     - `Status` = `Reminder Sent`
   - Connect: **Place Voice Call Reminder → Update Reminder Status**

9. **Close the loop**
   - Connect: **Update Reminder Status → Loop Over Items**
   - This makes SplitInBatches continue with the next record.

10. **Credentials & external setup checklist**
   - Google Sheets OAuth2: spreadsheet must be shared/accessible to the authenticated user.
   - Twilio:
     - WhatsApp enabled (sandbox or approved WhatsApp sender).
     - A valid voice-enabled “From” number.
     - If using templates/session rules for WhatsApp, ensure compliance.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| ## Google sheet | https://docs.google.com/spreadsheets/d/1P-rwG-J_Wqj_SAVIgQC49qzp6bA42v-S5Y9vwBASwDg/edit?usp=sharing |
| ## How it works: This workflow automates return pickup reminders… (includes setup steps about connecting Google Sheets/Twilio, updating phone numbers, verifying columns, testing with one record) | From “Workflow Overview” sticky note |
| ## Data Retrieval: Fetches today's pending pickups from Google Sheets and formats notification messages. | Data Retrieval sticky note |
| ## Customer Notifications: Processes each pickup one-by-one, sending WhatsApp message followed by automated voice call reminder. | Customer Notifications sticky note |
| ## Status Update: Marks the pickup status as "Reminder Sent"… | Status Update sticky note |