Manage garage service reminders, pickups and feedback with Airtable, WhatsApp and Gmail

https://n8nworkflows.xyz/workflows/manage-garage-service-reminders--pickups-and-feedback-with-airtable--whatsapp-and-gmail-12861


# Manage garage service reminders, pickups and feedback with Airtable, WhatsApp and Gmail

## 1. Workflow Overview

**Title:** Manage garage service reminders, pickups and feedback with Airtable, WhatsApp and Gmail

**Purpose:**  
This workflow automates customer communications for a car service garage using **Airtable as the system of record**, **Twilio WhatsApp** for messaging, and **Gmail** for email. It covers four operational stages and writes tracking flags/timestamps back to Airtable to prevent repeats (partially implemented).

**Primary use cases**
- Reduce no-shows by reminding customers of upcoming/today/overdue service appointments.
- Notify customers when a service is completed and the vehicle is ready for pickup.
- Request feedback after the vehicle is delivered.
- Remind customers about their next planned maintenance date.

### Logical blocks
1.1 **Scheduled service reminders (daily 9 AM)**  
Schedule trigger → Airtable search for pending services in date window → per-record message generation → WhatsApp → email → mark reminder sent.

1.2 **Pickup notifications (on Airtable status = “Done”)**  
Airtable Trigger (polling) → per-record message generation → WhatsApp → email → mark pickup notification sent.

1.3 **Feedback collection (on Airtable status = “Delivered”)**  
Airtable Trigger (polling) → per-record message generation (with feedback link) → WhatsApp → email → mark feedback sent.

1.4 **Next service reminders (daily 9 AM)**  
Schedule trigger → Airtable search by Next_Service_Date window → per-record message generation → WhatsApp → email → mark next reminder sent.  
**Note:** This block contains hardcoded/incorrect addressing fields that must be fixed (details below).

---

## 2. Block-by-Block Analysis

### 2.1 Global documentation / operator notes
**Overview:** Informational sticky notes describing overall workflow, setup steps, and links (Airtable base + feedback form + Google Sheet).

**Nodes involved:**  
- Sticky Note  
- Sticky Note4  
- Sticky Note6

**Node details**
- **Sticky Note** (Sticky Note node)
  - **Role:** Explains the four stages, setup steps, and Airtable base link.
  - **Key content:**  
    - Airtable Base: https://airtable.com/appmSBBXPDGEXnseu/shrV5FLUSKxEzs473  
    - Mentions customizing garage name via `GARAGE_NAME` in Code nodes.
  - **Failure modes:** None (documentation only).

- **Sticky Note4**
  - **Role:** Stores the Google Forms feedback link to update inside “Build Feedback Messages”.
  - **Link:** https://forms.gle/LHfFRQPU9fKaH8LB8

- **Sticky Note6**
  - **Role:** Stores a Google Sheet link for “Feed-Back Collect”.
  - **Link:** https://docs.google.com/spreadsheets/d/1SIIABzc81XSeJTDqAEWVbj4G5NP7nOPObytjokqKkMM/edit?usp=sharing

---

### 2.2 Scheduled Service Reminders (daily 9 AM)
**Overview:** Runs daily at 9 AM, finds “Pending” services happening from **2 days ago through 7 days ahead** where a reminder has not yet been sent, then sends WhatsApp + email and marks the reminder as sent in Airtable.

**Nodes involved:**  
- Daily Service Reminder Trigger  
- Fetch Pending Services (±7 Days)  
- Iterate Services  
- Build Reminder Messages  
- Send WhatsApp Reminder  
- Send Email Reminder  
- Mark Reminder Sent  
- Sticky Note1

#### Node details

- **Daily Service Reminder Trigger** (Schedule Trigger, v1.3)
  - **Role:** Time-based entry point.
  - **Config:** Triggers at **09:00** (workflow timezone applies).
  - **Outputs:** To **Fetch Pending Services (±7 Days)**.
  - **Failure modes:** Missed schedules if n8n is down; timezone mismatch.

- **Fetch Pending Services (±7 Days)** (Airtable node, v2.1)
  - **Role:** Searches Airtable for due services needing reminders.
  - **Config choices:**
    - Base: “Garage / Car Service Reminder System” (`appmSBBXPDGEXnseu`)
    - Table: “Tables” (`tbl3MOWawOfMkbcuk`)
    - Operation: **Search**
    - Fields returned: `Service_ID, Customer_Name, Phone, Email, Vehicle_Model, Vehicle_Number, Service_Type, Service_Date, Status`
    - `filterByFormula`:
      - `{Status}="Pending"`
      - `{Service_Date} >= TODAY()-2 days`
      - `{Service_Date} <= TODAY()+7 days`
      - `{Reminder_Sent} = FALSE()`
    - `returnAll: false` (so it will return only the Airtable node’s default page size unless otherwise configured).
  - **Outputs:** To **Iterate Services**.
  - **Edge cases / failures:**
    - Airtable auth/token error.
    - Formula errors if fields renamed or types mismatch.
    - Pagination limit risk because `returnAll=false` (may miss records beyond the first page).

- **Iterate Services** (Split In Batches, v3)
  - **Role:** Processes records in batches to avoid sending too many messages at once.
  - **Config:** Defaults (batch size not explicitly set; n8n defaults apply).
  - **Connections:**  
    - Output 1 loops back from **Mark Reminder Sent** to continue batching.  
    - Output 2 goes to **Build Reminder Messages** (this is how SplitInBatches commonly works: “Next batch” logic).
  - **Edge cases:** If upstream output is empty, nothing happens.

- **Build Reminder Messages** (Code node, v2)
  - **Role:** Builds WhatsApp + email content and categorizes reminder type (OVERDUE/TODAY/UPCOMING).
  - **Key config/logic:**
    - Hardcoded branding: `GARAGE_NAME = "PrimeCare Auto Garage"`
    - Computes `diffDays` between `Service_Date` and today (date-only).
    - Builds different copy for:
      - `diffDays < 0` → OVERDUE
      - `diffDays === 0` → TODAY
      - `diffDays <= 7` → UPCOMING
    - Outputs fields:
      - `reminderType`, `whatsappMessage`, `emailSubject`, `emailBody`, `phone`, `email`
  - **Inputs:** Items from Airtable search (expects fields at `item.json.*`).
  - **Outputs:** To **Send WhatsApp Reminder**.
  - **Edge cases / failures:**
    - Date parsing issues if `Service_Date` is not ISO-like.
    - If `Phone`/`Email` is missing, downstream sending will fail.
    - Typos in text (“please replay this email” should be “reply”).

- **Send WhatsApp Reminder** (Twilio node, v1)
  - **Role:** Sends the WhatsApp reminder via Twilio.
  - **Config:**
    - `to`: `{{ $('Build Reminder Messages').item.json.phone }}`
    - `from`: `+1234567890` (placeholder; must be a Twilio WhatsApp-enabled number/sender)
    - `toWhatsapp: true`
    - `message`: `{{ $json.whatsappMessage }}`
  - **Inputs:** Message JSON from Code node.
  - **Outputs:** To **Send Email Reminder**.
  - **Failure modes:**
    - Twilio credential/auth errors.
    - WhatsApp sender not enabled, wrong “from”, sandbox limitations.
    - Invalid phone formatting (E.164 typically required).

- **Send Email Reminder** (Gmail node, v2.2)
  - **Role:** Sends the email reminder via Gmail OAuth2.
  - **Config:**
    - `sendTo`: `{{ $('Build Reminder Messages').item.json.email }}`
    - `subject`: `{{ $('Build Reminder Messages').item.json.emailSubject }}`
    - `message`: `{{ $('Build Reminder Messages').item.json.emailBody }}`
    - `emailType: text`
    - `appendAttribution: false`
  - **Outputs:** To **Mark Reminder Sent**.
  - **Failure modes:** Gmail OAuth expiry, quota limits, invalid recipient.

- **Mark Reminder Sent** (Airtable node, v2.1)
  - **Role:** Writes back `Reminder_Sent=true` and timestamp in Airtable.
  - **Config choices:**
    - Operation: **Update**
    - **MatchingColumns:** `Phone` (updates record(s) where Airtable Phone matches)
    - Values:
      - `Phone`: from Code output (used for matching)
      - `Reminder_Sent`: true
      - `Reminder_Sent_On`: `{{ $now.toISO() }}`
  - **Connections:** To **Iterate Services** (to continue loop).
  - **Edge cases / failures:**
    - **Non-unique Phone**: could update the wrong record(s) or multiple records.
    - If phone changes, matching fails and reminder won’t be marked.
    - Safer match would be Airtable `id` or `Service_ID`.

- **Sticky Note1**
  - **Role:** Visual label for this block (scheduled reminders).

---

### 2.3 Pickup Notifications (status changes to “Done”)
**Overview:** When Airtable records meet the trigger formula `{Status}="Done"`, the workflow sends a pickup-ready WhatsApp + email and then marks pickup notification as sent.

**Nodes involved:**  
- Run to Service Status Changed → Done  
- Iterate Completed Services  
- Build Pickup Messages  
- Send Pickup WhatsApp  
- Send Pickup Email  
- Mark Pickup Notification Sent  
- Sticky Note2

#### Node details

- **Run to Service Status Changed → Done** (Airtable Trigger, v1)
  - **Role:** Event-like trigger via polling Airtable.
  - **Config:**
    - Polling: every minute
    - Trigger field: `Last Modified`
    - Additional formula: `={Status} = "Done"`
    - Auth: Airtable token API
  - **Outputs:** To **Iterate Completed Services**.
  - **Edge cases / failures:**
    - Polling can re-fire for unrelated modifications unless deduped.
    - No explicit check of `Pickup_Notification_Sent` here; duplicates can occur.

- **Iterate Completed Services** (Split In Batches, v3)
  - **Role:** Batch processing of triggered records.
  - **Connections:** Loops from **Mark Pickup Notification Sent** back into this node.
  - **Edge cases:** Same as other SplitInBatches.

- **Build Pickup Messages** (Code node, v2)
  - **Role:** Generates pickup WhatsApp + email copy.
  - **Key config/logic:**
    - `GARAGE_NAME = "PrimeCare Auto Garage"`
    - Uses `const data = $input.first().json.fields;`
      - This assumes the Airtable Trigger outputs items with `.json.fields`.
      - **Implementation detail:** It maps over `items` but always reads `$input.first()`, so if multiple items are present it will generate identical messages repeatedly (bug).
    - Outputs: `phone`, `email`, `whatsappMessage`, `emailSubject`, `emailBody`
  - **Outputs:** To **Send Pickup WhatsApp**.
  - **Edge cases / failures:**
    - If batch contains multiple items, incorrect behavior as noted.
    - Missing Phone/Email causes downstream failures.

- **Send Pickup WhatsApp** (Twilio node, v1)
  - **Role:** WhatsApp pickup notification.
  - **Config:** `to` from Build Pickup Messages, `from` placeholder, `toWhatsapp: true`.
  - **Outputs:** To **Send Pickup Email**.

- **Send Pickup Email** (Gmail node, v2.2)
  - **Role:** Email pickup notification.
  - **Outputs:** To **Mark Pickup Notification Sent**.

- **Mark Pickup Notification Sent** (Airtable node, v2.1)
  - **Role:** Updates Airtable fields.
  - **Config:**
    - Match: `Phone`
    - Set:
      - `Pickup_Notification_Sent: true`
      - `Pickup_Notification_Sent_On: {{ $now.toISO() }}`
  - **Outputs:** Loops to **Iterate Completed Services**.
  - **Edge cases:** Same “match by phone” risk; duplicates possible because trigger formula doesn’t exclude already-notified records.

- **Sticky Note2**
  - **Role:** Visual label for pickup notification block.

---

### 2.4 Feedback Collection (status changes to “Delivered”)
**Overview:** When a record is marked Delivered, the workflow sends a feedback request (WhatsApp + email) including a Google Forms link, and marks feedback as sent to prevent duplicates.

**Nodes involved:**  
- Vehicle Delivered Trigger  
- Iterate Delivered Records  
- Build Feedback Messages  
- Send Feedback WhatsApp  
- Send Feedback Email  
- Mark Feedback Sent  
- Sticky Note3  
- Sticky Note4  
- Sticky Note6

#### Node details

- **Vehicle Delivered Trigger** (Airtable Trigger, v1)
  - **Role:** Polling trigger for delivered services.
  - **Config:**
    - Poll every minute
    - Trigger field: `Last Modified`
    - Formula: `={Status} = "Delivered"`
  - **Outputs:** To **Iterate Delivered Records**.
  - **Edge cases:** Similar to “Done” trigger; may re-fire.

- **Iterate Delivered Records** (Split In Batches, v3)
  - **Role:** Batch processing.
  - **Connections:** Receives loop-back from **Mark Feedback Sent**.

- **Build Feedback Messages** (Code node, v2)
  - **Role:** Composes feedback request messages and enforces a basic anti-duplicate check.
  - **Key config/logic:**
    - `GARAGE_NAME = "PrimeCare Auto Garage"`
    - Uses `const data = $input.first().json.fields;` (same multi-item bug risk as pickup block)
    - Duplicate prevention:
      - If `data.Feedback_Sent === true` → outputs `{ skip: true }`
      - **But:** downstream nodes do not check `skip`, so it would still attempt sends unless n8n filters items (it won’t automatically). A proper IF node is missing.
    - Feedback link hardcoded: `https://forms.gle/LHfFRQPU9fKaH8LB8`
    - Outputs: `phone`, `email`, `whatsappMessage`, `emailSubject`, `emailBody`, `markFeedbackSent: true`
  - **Outputs:** To **Send Feedback WhatsApp**.
  - **Edge cases / failures:**
    - Multi-item batching bug.
    - “skip” not acted upon.
    - Missing phone/email.

- **Send Feedback WhatsApp** (Twilio node, v1)
  - **Role:** Sends WhatsApp feedback request.
  - **Outputs:** To **Send Feedback Email**.

- **Send Feedback Email** (Gmail node, v2.2)
  - **Role:** Sends feedback email.
  - **Outputs:** To **Mark Feedback Sent**.

- **Mark Feedback Sent** (Airtable node, v2.1)
  - **Role:** Marks feedback sent in Airtable.
  - **Config:**
    - Match: `Phone`
    - Set:
      - `Feedback_Sent`: `{{ $('Build Feedback Messages').item.json.markFeedbackSent }}`
      - `Feedback_Sent_On`: `{{ $now.toISO() }}`
  - **Outputs:** Loops to **Iterate Delivered Records**.
  - **Edge cases:** Same “match by phone” risk; could mark wrong record.

- **Sticky Note3 / Sticky Note4 / Sticky Note6**
  - **Role:** Documentation and links for feedback stage.

---

### 2.5 Next Service Reminders (daily 9 AM)
**Overview:** Runs daily at 9 AM, finds records whose `Next_Service_Date` is between **2 days ago and 7 days ahead**, generates WhatsApp/email reminders, sends them, then stores the “sent on” timestamp.

**Nodes involved:**  
- Next Service Reminder Trigger  
- Fetch Next Services (±7 Days)  
- Iterate Services  
- Build Next Service Reminder Messages  
- Send Next Service WhatsApp  
- Send Next Service Email  
- Mark Next Service Reminder Sent  
- Sticky Note5

#### Node details

- **Next Service Reminder Trigger** (Schedule Trigger, v1.3)
  - **Role:** Time-based entry point at 09:00.
  - **Outputs:** To **Fetch Next Services (±7 Days)**.

- **Fetch Next Services (±7 Days)** (Airtable node, v2.1)
  - **Role:** Searches by `Next_Service_Date` window.
  - **Config:**
    - Fields returned: `Customer_Name, Phone, Email, Vehicle_Model, Vehicle_Number, Service_Type, Service_Date, Next_Service_Date`
    - `filterByFormula`:
      - `Next_Service_Date >= TODAY()-2`
      - `Next_Service_Date <= TODAY()+7`
    - `returnAll: false` (pagination risk)
  - **Edge cases:**
    - No check for already-notified records (e.g., missing `{Next_Service_Notification_Sent_On}` constraint).

- **Iterate Services ** (Split In Batches, v3)
  - **Role:** Batch processing loop for next-service reminders.
  - **Connections:** Loops from **Mark Next Service Reminder Sent** back to this node.

- **Build Next Service Reminder Messages** (Code node, v2)
  - **Role:** Creates formatted reminder (OVERDUE/TODAY/UPCOMING) based on `Next_Service_Date`.
  - **Key config:**
    - `GARAGE_NAME = "PrimeCare Auto Garage"`
    - Reads `data = item.json.fields || item.json` (more robust than other code nodes)
    - If diffDays outside (-2..7) outputs `{ skip: true }` (but again, no IF node filters this later)
    - Formats date as `YYYY-MM-DD` using `toISOString().split("T")[0]` (timezone can shift date if local timezone is behind UTC).
  - **Outputs:** To **Send Next Service WhatsApp**.
  - **Edge cases / failures:**
    - `skip` not acted upon.
    - ISO formatting timezone drift.
    - Missing phone/email.

- **Send Next Service WhatsApp** (Twilio node, v1)
  - **Role:** Sends WhatsApp for next service.
  - **Config problem (critical):**
    - `to` is set to `=Phone_number` (not an n8n expression and not mapped to data).
    - This will not resolve to the customer phone and will likely fail.
    - Should be: `{{ $('Build Next Service Reminder Messages').item.json.phone }}` (or `{{ $json.phone }}` if directly connected).
  - **Outputs:** To **Send Next Service Email**.

- **Send Next Service Email** (Gmail node, v2.2)
  - **Role:** Sends email for next service.
  - **Config problem (critical):**
    - `sendTo` is hardcoded to `=infyom@gmail.com` (not expression syntax and not customer email).
    - Should be: `{{ $('Build Next Service Reminder Messages').item.json.email }}` (or `{{ $json.email }}`).
  - **Outputs:** To **Mark Next Service Reminder Sent**.

- **Mark Next Service Reminder Sent** (Airtable node, v2.1)
  - **Role:** Stores timestamp in `Next_Service_Notification_Sent_On`.
  - **Config:**
    - Match: `Phone`
    - Set:
      - `Next_Service_Notification_Sent_On: {{ $now.toISO() }}`
  - **Outputs:** Loops to **Iterate Services **.
  - **Edge cases:** Phone match risk; also should ideally store a boolean flag and/or filter on “not already sent”.

- **Sticky Note5**
  - **Role:** Visual label for this block.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Overall explanation + setup + Airtable link |  |  | ## How it works … 📊 **Airtable Base**: https://airtable.com/appmSBBXPDGEXnseu/shrV5FLUSKxEzs473 |
| Daily Service Reminder Trigger | Schedule Trigger | Daily entry point for service reminders |  | Fetch Pending Services (±7 Days) | ## Scheduled Service Reminders Runs daily at 9 AM… |
| Fetch Pending Services (±7 Days) | Airtable | Search pending services in date window | Daily Service Reminder Trigger | Iterate Services | ## Scheduled Service Reminders Runs daily at 9 AM… |
| Iterate Services | Split In Batches | Batch/loop through pending services | Fetch Pending Services (±7 Days); Mark Reminder Sent | Build Reminder Messages | ## Scheduled Service Reminders Runs daily at 9 AM… |
| Build Reminder Messages | Code | Build reminder WhatsApp/email bodies | Iterate Services | Send WhatsApp Reminder | ## Scheduled Service Reminders Runs daily at 9 AM… |
| Send WhatsApp Reminder | Twilio | Send WhatsApp reminder | Build Reminder Messages | Send Email Reminder | ## Scheduled Service Reminders Runs daily at 9 AM… |
| Send Email Reminder | Gmail | Send reminder email | Send WhatsApp Reminder | Mark Reminder Sent | ## Scheduled Service Reminders Runs daily at 9 AM… |
| Mark Reminder Sent | Airtable | Update Reminder_Sent fields | Send Email Reminder | Iterate Services | ## Scheduled Service Reminders Runs daily at 9 AM… |
| Sticky Note1 | Sticky Note | Block label |  |  | ## Scheduled Service Reminders Runs daily at 9 AM… |
| Run to Service Status Changed → Done | Airtable Trigger | Trigger on Status=Done |  | Iterate Completed Services | ## Pickup Notifications Triggered when service status changes to "Done"… |
| Iterate Completed Services | Split In Batches | Batch/loop done services | Run to Service Status Changed → Done; Mark Pickup Notification Sent | Build Pickup Messages | ## Pickup Notifications Triggered when service status changes to "Done"… |
| Build Pickup Messages | Code | Build pickup-ready messages | Iterate Completed Services | Send Pickup WhatsApp | ## Pickup Notifications Triggered when service status changes to "Done"… |
| Send Pickup WhatsApp | Twilio | Send WhatsApp pickup message | Build Pickup Messages | Send Pickup Email | ## Pickup Notifications Triggered when service status changes to "Done"… |
| Send Pickup Email | Gmail | Send pickup email | Send Pickup WhatsApp | Mark Pickup Notification Sent | ## Pickup Notifications Triggered when service status changes to "Done"… |
| Mark Pickup Notification Sent | Airtable | Update pickup notification flags | Send Pickup Email | Iterate Completed Services | ## Pickup Notifications Triggered when service status changes to "Done"… |
| Sticky Note2 | Sticky Note | Block label |  |  | ## Pickup Notifications Triggered when service status changes to "Done"… |
| Vehicle Delivered Trigger | Airtable Trigger | Trigger on Status=Delivered |  | Iterate Delivered Records | ## Feedback Collection Triggered when status changes to "Delivered"… |
| Iterate Delivered Records | Split In Batches | Batch/loop delivered records | Vehicle Delivered Trigger; Mark Feedback Sent | Build Feedback Messages | ## Feedback Collection Triggered when status changes to "Delivered"… |
| Build Feedback Messages | Code | Build feedback request messages (with link) | Iterate Delivered Records | Send Feedback WhatsApp | ## Feedback Collection Triggered when status changes to "Delivered"… |
| Send Feedback WhatsApp | Twilio | Send WhatsApp feedback request | Build Feedback Messages | Send Feedback Email | ## Feedback Collection Triggered when status changes to "Delivered"… |
| Send Feedback Email | Gmail | Send feedback email | Send Feedback WhatsApp | Mark Feedback Sent | ## Feedback Collection Triggered when status changes to "Delivered"… |
| Mark Feedback Sent | Airtable | Mark feedback sent fields | Send Feedback Email | Iterate Delivered Records | ## Feedback Collection Triggered when status changes to "Delivered"… |
| Sticky Note3 | Sticky Note | Block label |  |  | ## Feedback Collection Triggered when status changes to "Delivered"… |
| Sticky Note4 | Sticky Note | Feedback form link note |  |  | 📋 **Feedback Form** https://forms.gle/LHfFRQPU9fKaH8LB8 Update this link in the "Build Feedback Messages" node. |
| Sticky Note6 | Sticky Note | Google Sheet link note |  |  | ### Google sheet (Feed-Back Collect) https://docs.google.com/spreadsheets/d/1SIIABzc81XSeJTDqAEWVbj4G5NP7nOPObytjokqKkMM/edit?usp=sharing |
| Next Service Reminder Trigger | Schedule Trigger | Daily entry point for next-service reminders |  | Fetch Next Services (±7 Days) | ## Next Service Reminders Runs daily at 9 AM… |
| Fetch Next Services (±7 Days) | Airtable | Search by Next_Service_Date window | Next Service Reminder Trigger | Iterate Services  | ## Next Service Reminders Runs daily at 9 AM… |
| Iterate Services  | Split In Batches | Batch/loop next-service reminders | Fetch Next Services (±7 Days); Mark Next Service Reminder Sent | Build Next Service Reminder Messages | ## Next Service Reminders Runs daily at 9 AM… |
| Build Next Service Reminder Messages | Code | Build next-service WhatsApp/email bodies | Iterate Services  | Send Next Service WhatsApp | ## Next Service Reminders Runs daily at 9 AM… |
| Send Next Service WhatsApp | Twilio | Send WhatsApp next-service reminder | Build Next Service Reminder Messages | Send Next Service Email | ## Next Service Reminders Runs daily at 9 AM… |
| Send Next Service Email | Gmail | Send next-service reminder email | Send Next Service WhatsApp | Mark Next Service Reminder Sent | ## Next Service Reminders Runs daily at 9 AM… |
| Mark Next Service Reminder Sent | Airtable | Update Next_Service_Notification_Sent_On | Send Next Service Email | Iterate Services  | ## Next Service Reminders Runs daily at 9 AM… |
| Sticky Note5 | Sticky Note | Block label |  |  | ## Next Service Reminders Runs daily at 9 AM… |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. Airtable: create an **Airtable Personal Access Token** with read/write on the base.
   2. Twilio: add Twilio credentials; ensure **WhatsApp sending** is enabled (Twilio WhatsApp sender or sandbox).
   3. Gmail: connect Gmail via **OAuth2** in n8n (grant send permissions).

2) **Prepare Airtable base/table**
   1. Base: `Garage / Car Service Reminder System` (or your own).
   2. Table includes (minimum):  
      `Customer_Name, Phone, Email, Vehicle_Model, Vehicle_Number, Service_Type, Service_Date, Next_Service_Date, Status`  
      Tracking fields:  
      `Reminder_Sent (boolean), Reminder_Sent_On (datetime), Pickup_Notification_Sent (boolean), Pickup_Notification_Sent_On (datetime), Feedback_Sent (boolean), Feedback_Sent_On (datetime), Next_Service_Notification_Sent_On (datetime)`
   3. Ensure `Phone` values are consistently formatted (prefer E.164).

3) **Block A — Scheduled Service Reminders**
   1. Add **Schedule Trigger** named “Daily Service Reminder Trigger”, set **Trigger at hour: 9**.
   2. Add **Airtable** node “Fetch Pending Services (±7 Days)”:
      - Operation: Search
      - Select your base/table
      - Fields: include service/customer/contact fields
      - Filter formula:
        - Status Pending
        - Service_Date between TODAY()-2 and TODAY()+7
        - Reminder_Sent = FALSE()
      - Consider enabling **Return All** if you have many records.
   3. Add **Split In Batches** “Iterate Services”; connect Airtable → SplitInBatches.
   4. Add **Code** “Build Reminder Messages”; paste logic and set `GARAGE_NAME`.
   5. Add **Twilio** “Send WhatsApp Reminder”:
      - to: `{{ $json.phone }}` (or from the code node by reference)
      - from: your Twilio WhatsApp-enabled sender
      - toWhatsapp: true
      - message: `{{ $json.whatsappMessage }}`
   6. Add **Gmail** “Send Email Reminder”:
      - sendTo: `{{ $json.email }}`
      - subject: `{{ $json.emailSubject }}`
      - message: `{{ $json.emailBody }}`
      - emailType: text
   7. Add **Airtable Update** “Mark Reminder Sent”:
      - Operation: Update
      - Match on a unique key (**prefer Airtable record ID**; if not, use `Service_ID`)
      - Set `Reminder_Sent=true`, `Reminder_Sent_On={{ $now.toISO() }}`.
   8. Connect: Code → Twilio → Gmail → Airtable Update → back to **Split In Batches** to continue.

4) **Block B — Pickup Notifications (Status = Done)**
   1. Add **Airtable Trigger** “Run to Service Status Changed → Done”:
      - Poll every minute
      - Trigger field: Last Modified
      - Formula: `{Status}="Done"`
   2. Add **Split In Batches** “Iterate Completed Services”.
   3. Add **Code** “Build Pickup Messages”.
      - Fix recommended: use `item.json.fields` (per item), not `$input.first()`.
   4. Add **Twilio** “Send Pickup WhatsApp” (to/from/message as above).
   5. Add **Gmail** “Send Pickup Email”.
   6. Add **Airtable Update** “Mark Pickup Notification Sent”:
      - Set `Pickup_Notification_Sent=true`, `Pickup_Notification_Sent_On={{ $now.toISO() }}`
   7. Connect sequentially and loop back to SplitInBatches.

5) **Block C — Feedback Collection (Status = Delivered)**
   1. Add **Airtable Trigger** “Vehicle Delivered Trigger” with formula `{Status}="Delivered"`.
   2. Add **Split In Batches** “Iterate Delivered Records”.
   3. Add **Code** “Build Feedback Messages”:
      - Set `GARAGE_NAME`
      - Set `feedbackLink` to your Google Form URL.
      - Add proper skip handling (recommended next step).
   4. Add an **IF node** (recommended) after the Code node:
      - Condition: `{{$json.skip}} is true` → route to a NoOp/end
      - Else → proceed to send nodes
   5. Add **Twilio** “Send Feedback WhatsApp”.
   6. Add **Gmail** “Send Feedback Email”.
   7. Add **Airtable Update** “Mark Feedback Sent”:
      - Set `Feedback_Sent=true`, `Feedback_Sent_On={{ $now.toISO() }}`
   8. Loop back to SplitInBatches.

6) **Block D — Next Service Reminders**
   1. Add **Schedule Trigger** “Next Service Reminder Trigger” at 9.
   2. Add **Airtable Search** “Fetch Next Services (±7 Days)” filtering `Next_Service_Date` between TODAY()-2 and TODAY()+7.
      - Recommended: also filter out records already notified (e.g., `Next_Service_Notification_Sent_On` empty or older than X).
   3. Add **Split In Batches** “Iterate Services”.
   4. Add **Code** “Build Next Service Reminder Messages”.
   5. Add **Twilio** “Send Next Service WhatsApp” and **fix**:
      - to: `{{ $json.phone }}`
   6. Add **Gmail** “Send Next Service Email” and **fix**:
      - sendTo: `{{ $json.email }}`
   7. Add **Airtable Update** “Mark Next Service Reminder Sent”:
      - Set `Next_Service_Notification_Sent_On={{ $now.toISO() }}`
   8. Connect and loop back to SplitInBatches.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Airtable base referenced in workflow notes | https://airtable.com/appmSBBXPDGEXnseu/shrV5FLUSKxEzs473 |
| Feedback form link (must match Code node “Build Feedback Messages”) | https://forms.gle/LHfFRQPU9fKaH8LB8 |
| Google sheet (Feed-Back Collect) | https://docs.google.com/spreadsheets/d/1SIIABzc81XSeJTDqAEWVbj4G5NP7nOPObytjokqKkMM/edit?usp=sharing |
| Branding customization | Update `GARAGE_NAME` constant in all Code nodes (search “GARAGE_NAME”). |
| Known implementation issues to fix before production | Next-service block has incorrect `to`/`sendTo` values; Code nodes for pickup/feedback use `$input.first()` which breaks multi-item batching; “skip” flags are not filtered by an IF node; “match by Phone” in Airtable updates can update wrong records. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.