Send a daily email summary of new Airtable contacts via Gmail

https://n8nworkflows.xyz/workflows/send-a-daily-email-summary-of-new-airtable-contacts-via-gmail-13456


# Send a daily email summary of new Airtable contacts via Gmail

## 1. Workflow Overview

**Purpose:**  
This workflow sends a **daily email summary (via Gmail)** of **new Airtable contacts created today**, formatted as a clean **HTML table**.

**Target use cases:**
- Daily sales/CRM monitoring of newly added contacts
- Lightweight daily digest without opening Airtable
- Team or personal inbox reporting of daily contact intake

### 1.1 Trigger & Scheduling
Runs automatically every day at a specified hour (default: **23:00 / 11 PM**).

### 1.2 Data Retrieval from Airtable
Queries the Airtable “Contacts” table for records whose **Created** date is the current day.

### 1.3 Formatting for Email
Converts the returned records into an **HTML table** suitable for email bodies.

### 1.4 Email Delivery via Gmail
Sends a Gmail email to a configured recipient with the HTML table embedded.

---

## 2. Block-by-Block Analysis

### Block 2.1 — Trigger & Scheduling

**Overview:**  
Starts the workflow daily at 11 PM (configurable). This is the only entry point.

**Nodes Involved:**
- Daily Evening Schedule

#### Node: Daily Evening Schedule
- **Type / Role:** `Schedule Trigger` — time-based workflow trigger.
- **Configuration (interpreted):**
  - Runs every day at **hour 23**.
  - (No minutes specified; n8n defaults typically to `00` unless configured otherwise.)
- **Inputs:** None (trigger node).
- **Outputs / Connections:**
  - Output → **Search Airtable Contacts**
- **Version notes:** Node typeVersion **1.3**.
- **Edge cases / failures:**
  - If n8n instance timezone differs from expected, the “23:00” run time may not match local time.
  - If the instance is down at trigger time, the run may be skipped unless your n8n setup supports catch-up behavior.

---

### Block 2.2 — Retrieve Today’s New Contacts from Airtable

**Overview:**  
Searches Airtable for contacts created “today” and returns selected fields for reporting.

**Nodes Involved:**
- Search Airtable Contacts

#### Node: Search Airtable Contacts
- **Type / Role:** `Airtable` — searches records in an Airtable table.
- **Configuration (interpreted):**
  - **Operation:** Search
  - **Base:** “Contacts” (Base ID: `app1bfDNQWWNpiwal`)
  - **Table:** “Contacts” (Table ID: `tblubuxXAzrkV59GY`)
  - **Returned fields:** `Company Name`, `Name`, `Email`
  - **Filter formula:**
    - `IS_SAME({Created}, TODAY(), 'day')`
    - Meaning: only records where `{Created}` is the same calendar day as today.
- **Credentials:**
  - Uses **Airtable Personal Access Token** credential (`airtableTokenApi`).
- **Inputs / Outputs:**
  - Input ← **Daily Evening Schedule**
  - Output → **Convert to HTML Table**
- **Version notes:** Node typeVersion **2.1**.
- **Edge cases / failures:**
  - **Formula dependency:** `{Created}` must exist and be a Date/DateTime field. If missing or not a date type, Airtable may return no results or error.
  - **Timezone mismatch:** `TODAY()` and your “Created” timestamps may behave unexpectedly depending on Airtable’s timezone assumptions and whether `{Created}` is date-only vs datetime.
  - **No results:** If there are no contacts today, downstream formatting/email may produce an empty table (or possibly an unexpected structure depending on the HTML node behavior).
  - **Auth/scopes:** Token missing required scopes or base access will cause 401/403 errors.
  - **Large volume:** Many records could increase payload size and email length.

---

### Block 2.3 — Convert Records to an HTML Table

**Overview:**  
Takes Airtable results and produces an HTML table string for embedding into the email.

**Nodes Involved:**
- Convert to HTML Table

#### Node: Convert to HTML Table
- **Type / Role:** `HTML` — transforms JSON items to an HTML table.
- **Configuration (interpreted):**
  - **Operation:** Convert to HTML Table
  - Default options (no custom table styling specified).
- **Key data behavior:**
  - Produces a field commonly exposed as `table` in the node’s output JSON (used later as `{{$json.table}}`).
- **Inputs / Outputs:**
  - Input ← **Search Airtable Contacts**
  - Output → **Send Daily Contacts Email**
- **Version notes:** Node typeVersion **1.2**.
- **Edge cases / failures:**
  - **Unexpected input shape:** If Airtable returns items with nested structures or missing fields, table output may include blanks or serialized objects.
  - **Empty input set:** May output an empty table or possibly no items; if no items are output, the Gmail node may not run depending on n8n execution behavior.

---

### Block 2.4 — Send Email Digest via Gmail

**Overview:**  
Sends an email containing the HTML table of new contacts.

**Nodes Involved:**
- Send Daily Contacts Email

#### Node: Send Daily Contacts Email
- **Type / Role:** `Gmail` — sends an email through Gmail using OAuth2.
- **Configuration (interpreted):**
  - **To:** `user@example.com` (placeholder; should be updated)
  - **Subject:** `New Contacts Today`
  - **Message body (HTML):**
    - Expression-based value:
      ```html
      <h3> New Contacts Today</h3>

      {{ $json.table }}
      ```
    - `{{$json.table}}` is expected from the **Convert to HTML Table** node output.
- **Credentials:**
  - Gmail OAuth2 credential (`gmailOAuth2`) named `vasarmilan@gmail.com`.
- **Inputs / Outputs:**
  - Input ← **Convert to HTML Table**
  - Output: none (terminal node)
- **Version notes:** Node typeVersion **2.2**.
- **Edge cases / failures:**
  - **OAuth expiration / revoked consent:** Gmail will fail until re-authenticated.
  - **Recipient not updated:** Emails will go to the placeholder unless changed.
  - **HTML rendering:** Email clients vary; some may strip styling. The table should still render in most clients.
  - **Payload size limits:** Very large tables may exceed Gmail limits or be clipped.
  - **Empty table scenario:** Email may be sent with a blank/empty table unless guarded by an IF node (not present).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Daily Evening Schedule | Schedule Trigger | Daily time-based entry point | — | Search Airtable Contacts | ## Workflow Overview<br><br>This automation sends you a daily email summary of all new contacts added to your Airtable base. Every evening, it searches for contacts created that day, formats them into a clean HTML table, and delivers them directly to your inbox—keeping you updated without any manual effort.<br><br>### First Setup<br><br>**Airtable Connection:**<br>1. Create a Personal Access Token at https://airtable.com/create/tokens<br>2. Add these scopes: `data.records:read`, `data.records:write`, and `schema.bases:read`<br>3. Grant access to your bases and paste the token into n8n credentials<br><br>**Gmail Connection:**<br>- Connect your Gmail account through n8n's OAuth2 authentication<br><br>### Configuration<br><br>- **Schedule**: The trigger is set to run daily at 11 PM. Adjust the time in the Schedule Trigger node to match your preference<br>- **Airtable Base & Table**: Update the Search records node to point to your own Airtable base and contacts table<br>- **Email Recipient**: Change the recipient email address in the Gmail node to your own<br>- **Data Fields**: Customize which contact fields appear in the report by modifying the Output Fields in the Airtable node |
| Search Airtable Contacts | Airtable | Query today’s newly created contacts | Daily Evening Schedule | Convert to HTML Table | ## Workflow Overview<br><br>This automation sends you a daily email summary of all new contacts added to your Airtable base. Every evening, it searches for contacts created that day, formats them into a clean HTML table, and delivers them directly to your inbox—keeping you updated without any manual effort.<br><br>### First Setup<br><br>**Airtable Connection:**<br>1. Create a Personal Access Token at https://airtable.com/create/tokens<br>2. Add these scopes: `data.records:read`, `data.records:write`, and `schema.bases:read`<br>3. Grant access to your bases and paste the token into n8n credentials<br><br>**Gmail Connection:**<br>- Connect your Gmail account through n8n's OAuth2 authentication<br><br>### Configuration<br><br>- **Schedule**: The trigger is set to run daily at 11 PM. Adjust the time in the Schedule Trigger node to match your preference<br>- **Airtable Base & Table**: Update the Search records node to point to your own Airtable base and contacts table<br>- **Email Recipient**: Change the recipient email address in the Gmail node to your own<br>- **Data Fields**: Customize which contact fields appear in the report by modifying the Output Fields in the Airtable node |
| Convert to HTML Table | HTML | Format Airtable results into an HTML table | Search Airtable Contacts | Send Daily Contacts Email | ## Workflow Overview<br><br>This automation sends you a daily email summary of all new contacts added to your Airtable base. Every evening, it searches for contacts created that day, formats them into a clean HTML table, and delivers them directly to your inbox—keeping you updated without any manual effort.<br><br>### First Setup<br><br>**Airtable Connection:**<br>1. Create a Personal Access Token at https://airtable.com/create/tokens<br>2. Add these scopes: `data.records:read`, `data.records:write`, and `schema.bases:read`<br>3. Grant access to your bases and paste the token into n8n credentials<br><br>**Gmail Connection:**<br>- Connect your Gmail account through n8n's OAuth2 authentication<br><br>### Configuration<br><br>- **Schedule**: The trigger is set to run daily at 11 PM. Adjust the time in the Schedule Trigger node to match your preference<br>- **Airtable Base & Table**: Update the Search records node to point to your own Airtable base and contacts table<br>- **Email Recipient**: Change the recipient email address in the Gmail node to your own<br>- **Data Fields**: Customize which contact fields appear in the report by modifying the Output Fields in the Airtable node |
| Send Daily Contacts Email | Gmail | Send the daily HTML digest email | Convert to HTML Table | — | ## Workflow Overview<br><br>This automation sends you a daily email summary of all new contacts added to your Airtable base. Every evening, it searches for contacts created that day, formats them into a clean HTML table, and delivers them directly to your inbox—keeping you updated without any manual effort.<br><br>### First Setup<br><br>**Airtable Connection:**<br>1. Create a Personal Access Token at https://airtable.com/create/tokens<br>2. Add these scopes: `data.records:read`, `data.records:write`, and `schema.bases:read`<br>3. Grant access to your bases and paste the token into n8n credentials<br><br>**Gmail Connection:**<br>- Connect your Gmail account through n8n's OAuth2 authentication<br><br>### Configuration<br><br>- **Schedule**: The trigger is set to run daily at 11 PM. Adjust the time in the Schedule Trigger node to match your preference<br>- **Airtable Base & Table**: Update the Search records node to point to your own Airtable base and contacts table<br>- **Email Recipient**: Change the recipient email address in the Gmail node to your own<br>- **Data Fields**: Customize which contact fields appear in the report by modifying the Output Fields in the Airtable node |
| Workflow Description | Sticky Note | Embedded documentation / setup notes | — | — | ## Workflow Overview<br><br>This automation sends you a daily email summary of all new contacts added to your Airtable base. Every evening, it searches for contacts created that day, formats them into a clean HTML table, and delivers them directly to your inbox—keeping you updated without any manual effort.<br><br>### First Setup<br><br>**Airtable Connection:**<br>1. Create a Personal Access Token at https://airtable.com/create/tokens<br>2. Add these scopes: `data.records:read`, `data.records:write`, and `schema.bases:read`<br>3. Grant access to your bases and paste the token into n8n credentials<br><br>**Gmail Connection:**<br>- Connect your Gmail account through n8n's OAuth2 authentication<br><br>### Configuration<br><br>- **Schedule**: The trigger is set to run daily at 11 PM. Adjust the time in the Schedule Trigger node to match your preference<br>- **Airtable Base & Table**: Update the Search records node to point to your own Airtable base and contacts table<br>- **Email Recipient**: Change the recipient email address in the Gmail node to your own<br>- **Data Fields**: Customize which contact fields appear in the report by modifying the Output Fields in the Airtable node |
| Video Walkthrough | Sticky Note | Embedded video link/thumbnail | — | — | # Video Walkthrough<br>[![image.png](https://vasarmilan-public.s3.us-east-1.amazonaws.com/blog_thumbnails/thumbnail_recidrbznaXlmqFUI.jpg)](https://youtu.be/lQh1fuIrBN8) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Node: Schedule Trigger**
   - Node type: **Schedule Trigger**
   - Name: `Daily Evening Schedule`
   - Configure rule: **Every day at 23:00**
   - Save.

2. **Create Airtable Credentials (Personal Access Token)**
   - In n8n: **Credentials → New → Airtable Personal Access Token**
   - In Airtable, generate a token: https://airtable.com/create/tokens
   - Grant scopes (as described in the workflow note):
     - `data.records:read`
     - `data.records:write` (not strictly required for this workflow since it only reads, but included in the note)
     - `schema.bases:read`
   - Grant base access to the base containing your contacts.
   - Paste token into n8n credential and save.

3. **Create Node: Airtable (Search)**
   - Node type: **Airtable**
   - Name: `Search Airtable Contacts`
   - Credentials: select your Airtable PAT credential
   - Operation: **Search**
   - Base: select your base (or paste Base ID)
   - Table: select your contacts table (or paste Table ID)
   - Options / Fields to return:
     - `Company Name`
     - `Name`
     - `Email`
   - Filter formula:
     - `IS_SAME({Created}, TODAY(), 'day')`
   - Connect: `Daily Evening Schedule` → `Search Airtable Contacts`

4. **Create Node: HTML (Convert to HTML Table)**
   - Node type: **HTML**
   - Name: `Convert to HTML Table`
   - Operation: **Convert to HTML Table**
   - Keep default options unless you want custom formatting.
   - Connect: `Search Airtable Contacts` → `Convert to HTML Table`

5. **Create Gmail Credentials (OAuth2)**
   - In n8n: **Credentials → New → Gmail OAuth2**
   - Complete OAuth2 login/consent for the sending Gmail account.
   - Save the credential.

6. **Create Node: Gmail (Send Email)**
   - Node type: **Gmail**
   - Name: `Send Daily Contacts Email`
   - Credentials: select your Gmail OAuth2 credential
   - To: set your recipient (replace `user@example.com`)
   - Subject: `New Contacts Today`
   - Message (set as expression-enabled HTML content):
     ```html
     <h3> New Contacts Today</h3>

     {{ $json.table }}
     ```
   - Connect: `Convert to HTML Table` → `Send Daily Contacts Email`

7. **(Optional but recommended) Add a “no new contacts” guard**
   - Add an **IF** node after the Airtable search to check if any items exist, and only then convert/send.
   - Without this, you may receive an email even when there are zero new contacts (depending on how the HTML node behaves with empty input).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Create an Airtable Personal Access Token and assign scopes `data.records:read`, `data.records:write`, `schema.bases:read` | https://airtable.com/create/tokens |
| Video Walkthrough | https://youtu.be/lQh1fuIrBN8 |
| Walkthrough thumbnail image | https://vasarmilan-public.s3.us-east-1.amazonaws.com/blog_thumbnails/thumbnail_recidrbznaXlmqFUI.jpg |