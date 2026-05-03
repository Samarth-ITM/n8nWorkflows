Capture LinkedIn leads and sync them with Google Sheets, HubSpot or Salesforce

https://n8nworkflows.xyz/workflows/capture-linkedin-leads-and-sync-them-with-google-sheets--hubspot-or-salesforce-12580


# Capture LinkedIn leads and sync them with Google Sheets, HubSpot or Salesforce

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Automatically capture potential leads from LinkedIn (new connections + post engagement), enrich and score them, prevent duplicates for 90 days, sync them to **Google Sheets** and optionally a CRM (**HubSpot or Salesforce**), then notify the sales team (Slack/email) for **Hot/Warm** leads only.

**Target use cases:**
- Sales/BD teams building an inbound-ish pipeline from LinkedIn activity
- Lightweight “lead intelligence” enrichment + prioritization (score + temperature)
- Centralizing leads to Sheets plus one CRM

### 1.1 Logical Blocks (by dependency)
1. **Capture leads (LinkedIn polling)**: Schedule trigger → fetch connections + fetch engagement → merge  
2. **Process, enrich, score, deduplicate**: Extract/score (Code) → duplicate filter (Code with static data) → route create/update (Switch)  
3. **Sync to storage & CRM**: Google Sheets append/update; HubSpot create/update; Salesforce create (all fed by Switch paths)  
4. **Notify sales team**: Build message (Code) → Filter Hot/Warm → Slack webhook + Email  
5. **Logging & stats**: Log and update workflow-wide counters (static data)

---

## 2. Block-by-Block Analysis

### Block 1 — Capture leads (LinkedIn polling)
**Overview:** Runs every 15 minutes, queries LinkedIn for two sources (connections and engagement), and merges them into a single stream for unified processing.

**Nodes involved:**
- Check for new leads every 15 minutes
- Fetch new LinkedIn connections
- Fetch post engagement data
- Merge connections and engagements

#### Node: “Check for new leads every 15 minutes”
- **Type / role:** Schedule Trigger; starts workflow on a fixed interval.
- **Config choices:** Every **15 minutes**.
- **Outputs:** Sends one execution pulse to both LinkedIn nodes in parallel.
- **Edge cases / failures:** None typical; timezone/instance scheduling drift can occur depending on hosting.

#### Node: “Fetch new LinkedIn connections”
- **Type / role:** LinkedIn node; fetches profile-related data.
- **Config choices:** `operation: getProfile`, `authentication: OAuth2`.
- **Inputs:** Schedule Trigger.
- **Outputs:** To Merge (input index 0).
- **Edge cases / failures:**
  - OAuth2 token expiry / revoked permissions
  - LinkedIn API limitations/availability
  - **Potential semantic mismatch:** “getProfile” usually returns a profile (often current user) rather than “new connections”. If the intention is “new connections,” verify the LinkedIn node supports that endpoint/operation in your n8n version.

#### Node: “Fetch post engagement data”
- **Type / role:** LinkedIn node; fetches engagement objects.
- **Config choices:** `operation: getAll`, `authentication: OAuth2`.
- **Inputs:** Schedule Trigger.
- **Outputs:** To Merge (input index 1).
- **Edge cases / failures:**
  - Same OAuth2 risks as above
  - “getAll” is generic; ensure it is scoped to a resource that actually returns engagement data in your LinkedIn node configuration/version.

#### Node: “Merge connections and engagements”
- **Type / role:** Merge node; combines two incoming streams.
- **Config choices:** `mode: combine`.
- **Inputs:** LinkedIn connections (index 0) + engagement (index 1).
- **Outputs:** To Extract and score lead data.
- **Edge cases / failures:**
  - If one branch returns **0 items**, “combine” behavior can produce **0 merged items** or unexpected pairing. If you intend “union/append,” consider Merge mode “Append” instead.
  - Item alignment: combine usually pairs items by position; mismatched lengths can cause partial data or drop-offs.

---

### Block 2 — Process, enrich, score, deduplicate, route
**Overview:** Normalizes raw LinkedIn items into a consistent “lead” schema, assigns a score and temperature, prevents duplicates using global static data (90-day window), and routes to “create” vs “update” paths.

**Nodes involved:**
- Extract and score lead data
- Filter out duplicate leads
- Route to create or update path

#### Node: “Extract and score lead data”
- **Type / role:** Code node; data transformation + scoring.
- **Config choices:** Runs **once per item**.
- **Key logic / variables:**
  - Pulls fields from multiple possible shapes: `leadData.firstName` or `leadData.author?.firstName`, etc.
  - Builds `profileUrl` from `vanityName` when present.
  - Infers `company` from headline via regex patterns (`at X`, `@ X`, `| X`).
  - Computes:
    - `engagementScore` (connection + comment + like)
    - `qualityScore` (engagementScore + completeness bonuses)
    - `leadTemperature`: Cold (<50), Warm (50–69), Hot (70+)
  - Outputs a normalized lead object including `originalData`.
- **Inputs:** Merged LinkedIn items.
- **Outputs:** To Filter out duplicate leads.
- **Edge cases / failures:**
  - Inconsistent LinkedIn payload shapes can lead to “Unknown”/empty fields (handled by defaults).
  - `profileUrl` expression precedence: `leadData.profileUrl || leadData.vanityName ? ... : 'N/A'` can be ambiguous; safer would be parentheses.
  - Locale differences: `toLocaleDateString('en-US')` may produce unexpected formatting if you need ISO dates.

#### Node: “Filter out duplicate leads”
- **Type / role:** Code node; dedup + update detection using workflow static data.
- **Config choices:** Runs **once per item**; uses `$getWorkflowStaticData('global')`.
- **Key logic / variables:**
  - Unique key: prefers `profileUrl` unless `'N/A'`, else falls back to `leadId`.
  - If seen before: sets `isDuplicate: true`, `action: 'update'`, and attaches `existingData`.
  - If new: stores minimal record, cleans entries older than **90 days**, sets `action: 'create'`.
- **Inputs:** Enriched lead items.
- **Outputs:** To Switch (“Route to create or update path”).
- **Edge cases / failures:**
  - Static data is per-workflow and persists across executions, but behavior differs depending on n8n runtime (cloud vs self-hosted, scaling/queue mode). In multi-worker setups, duplicates may slip through if static data isn’t shared.
  - Memory growth: large `processedLeads` map if high volume; cleanup helps but relies on timestamps being present and parseable.
  - This node does **not** store CRM IDs (e.g., HubSpot contact ID). Downstream “Update existing contact in HubSpot” expects `existingData.crmContactId`, which is never set in this workflow (see Block 3 risks).

#### Node: “Route to create or update path”
- **Type / role:** Switch node; branching logic.
- **Config choices:** Routes by `{{$json.action}}`:
  - Output “New Lead” when equals `create`
  - Output “Update Existing” when equals `update`
- **Inputs:** Deduped lead items.
- **Outputs:**
  - **New Lead path:** Append new lead to Google Sheets, Create new contact in HubSpot, Create new lead in Salesforce (three parallel outputs).
  - **Update Existing path:** Update existing lead in Google Sheets.
- **Edge cases / failures:**
  - If `action` is missing or unexpected, item will be dropped (no default route configured).

---

### Block 3 — Sync to platforms (Google Sheets + CRM)
**Overview:** Persists leads in Google Sheets and optionally syncs to HubSpot or Salesforce. The workflow is designed so you enable **either** HubSpot or Salesforce nodes (and disable the other).

**Nodes involved:**
- Append new lead to Google Sheets
- Update existing lead in Google Sheets
- Create new contact in HubSpot
- Update existing contact in HubSpot
- Create new lead in Salesforce

#### Node: “Append new lead to Google Sheets”
- **Type / role:** Google Sheets node; append new row.
- **Config choices:**
  - Operation: **Append**
  - Spreadsheet: placeholder `YOUR_GOOGLE_SHEET_ID`
  - Sheet: `Sheet1`
  - Column mapping: explicit mapping (leadId, fullName, firstName, lastName, email, phone, company, position, headline, location, profileUrl, leadSource, leadTemperature, qualityScore, engagementType, commentText, dateAdded, status, assignedTo, notes)
- **Inputs:** Switch “New Lead” path.
- **Outputs:** To Build notification message.
- **Version requirements:** Google Sheets node v4.5 (as in workflow).
- **Edge cases / failures:**
  - Missing spreadsheet permissions (OAuth2 scope / sharing)
  - Column header mismatch → append may fail or write into wrong columns
  - Rate limits / quota errors on Google API

#### Node: “Update existing lead in Google Sheets”
- **Type / role:** Google Sheets node; update rows.
- **Config choices:**
  - Operation: **Update**
  - Mapping mode: **autoMapInputData**
  - **No matchingColumns configured** (critical)
- **Inputs:** Switch “Update Existing” path.
- **Outputs:** To Build notification message and to Update existing contact in HubSpot (parallel).
- **Edge cases / failures:**
  - Without a configured key (e.g., matching on `leadId` or `profileUrl`), the node may be unable to locate the row to update, or may update the wrong row depending on defaults.
  - Consider configuring `matchingColumns` to `leadId` (and ensuring the sheet contains it uniquely).

#### Node: “Create new contact in HubSpot”
- **Type / role:** HTTP Request node using HubSpot predefined credential; creates contact.
- **Config choices:**
  - Method: POST
  - URL: `https://api.hubspot.com/crm/v3/objects/contacts`
  - Body: a `properties` object mapping lead fields (firstname, lastname, email, phone, company, jobtitle, linkedin_url, lead_source, lead_temperature, quality_score, city, hs_lead_status="NEW")
  - Auth: `predefinedCredentialType` → `hubspotApi`
- **Inputs:** Switch “New Lead” path.
- **Outputs:** To Build notification message.
- **Edge cases / failures:**
  - HubSpot validation: email format, required fields, property names must exist in your portal (custom properties like `linkedin_url`, `lead_temperature`, `quality_score` may need creation).
  - Duplicate email behavior: HubSpot may upsert or error depending on portal settings and endpoint rules.

#### Node: “Update existing contact in HubSpot”
- **Type / role:** HTTP Request node; updates existing contact.
- **Config choices:**
  - Method: PATCH
  - URL template: `https://api.hubspot.com/crm/v3/objects/contacts/{{ $json.existingData.crmContactId }}`
  - Body: updates engagement_type, last_engagement_date, quality_score, notes
- **Inputs:** From “Update existing lead in Google Sheets”.
- **Outputs:** To Build notification message.
- **Edge cases / failures (critical):**
  - `existingData.crmContactId` is **never set** by the provided dedup logic. This will produce an invalid URL and fail (404 / malformed).
  - To make this work, you must store HubSpot contact ID into static data when creating contacts, or search HubSpot by email/profileUrl before updating.

#### Node: “Create new lead in Salesforce”
- **Type / role:** HTTP Request node using Salesforce OAuth2 credential; creates Lead.
- **Config choices:**
  - Method: POST
  - URL: `https://YOUR_SALESFORCE_INSTANCE.salesforce.com/services/data/v58.0/sobjects/Lead/`
  - Fields: FirstName, LastName, Email, Company, Title, LeadSource="LinkedIn", Status based on temperature, Rating=temperature, Description=commentText
  - Auth: predefined credential type `salesforceOAuth2Api`
- **Inputs:** Switch “New Lead” path.
- **Outputs:** To Build notification message.
- **Edge cases / failures:**
  - Placeholder instance URL must be replaced
  - Required Salesforce fields (e.g., Company, LastName) must be present; if missing you’ll get 400 errors
  - Field-level security / validation rules may reject records

---

### Block 4 — Notify sales team (Slack + Email)
**Overview:** Builds a human-readable message, filters to only Hot/Warm leads, then sends Slack and email notifications.

**Nodes involved:**
- Build notification message
- Filter for hot and warm leads only
- Send Slack notification to sales team
- Send email notification to sales team

#### Node: “Build notification message”
- **Type / role:** Code node; formats notification text and sets a notification flag.
- **Config choices:** Runs once per item.
- **Key outputs:**
  - `notificationMessage` (includes emoji/icons and lead details)
  - `notificationTitle` (`"${leadTemperature} Lead: ${fullName}"`)
  - `shouldNotify` = true if temperature is Hot or Warm
- **Inputs:** From Sheets append, HubSpot create, Salesforce create, HubSpot update, and Sheets update.
- **Outputs:** To Filter for hot and warm leads only.
- **Edge cases / failures:**
  - Message includes emoji; some email systems/logging may not like it (usually fine).
  - If upstream CRM node returns a different JSON shape (overwriting lead fields), message generation may degrade. Ensure upstream nodes pass through lead data (or merge responses) if needed.

#### Node: “Filter for hot and warm leads only”
- **Type / role:** Filter node; gate notifications.
- **Config choices:** Condition `{{$json.shouldNotify}}` is true.
- **Inputs:** Build notification message.
- **Outputs:** To Slack + Email nodes (parallel).
- **Edge cases / failures:** If `shouldNotify` missing/false, nothing is sent (intended).

#### Node: “Send Slack notification to sales team”
- **Type / role:** HTTP Request; Slack Incoming Webhook.
- **Config choices:**
  - Method: POST
  - URL: `YOUR_SLACK_WEBHOOK_URL` (placeholder)
  - Body: `text` + `blocks` containing the formatted message
- **Inputs:** Filter pass-through.
- **Outputs:** To Log success and track statistics.
- **Edge cases / failures:**
  - Invalid webhook URL → 404
  - Slack payload formatting errors (blocks must be valid JSON structure)

#### Node: “Send email notification to sales team”
- **Type / role:** Email Send (SMTP).
- **Config choices:**
  - To: `user@example.com` (placeholder)
  - From: `user@example.com` (placeholder)
  - Subject: `{{$json.notificationTitle}}`
  - SMTP credentials required
- **Inputs:** Filter pass-through.
- **Outputs:** To Log success and track statistics.
- **Edge cases / failures:**
  - SMTP auth/relay restrictions, SPF/DKIM issues
  - Missing email body: this node as configured sets subject/to/from; if no body is configured in your n8n UI defaults, emails may be empty.

---

### Block 5 — Logging & statistics
**Overview:** Logs a success line and keeps cumulative stats (total/hot/warm/cold, lastProcessed) in workflow static data.

**Nodes involved:**
- Log success and track statistics

#### Node: “Log success and track statistics”
- **Type / role:** Code node; logging + persistent counters.
- **Config choices:** Runs once per item; uses `$getWorkflowStaticData('global').stats`.
- **Inputs:** From Slack and Email nodes.
- **Outputs:** Final workflow output item with success + cumulativeStats.
- **Edge cases / failures:**
  - Because both Slack and Email converge here, you may log/track twice per lead (once per notification path). However, the code increments totals only when `action === 'create'`, so duplicates are limited—but you may still produce two output items per notified lead.
  - Static data persistence caveats as noted earlier.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Check for new leads every 15 minutes | Schedule Trigger | Polling trigger (15 min) | — | Fetch new LinkedIn connections; Fetch post engagement data | ## Capture LinkedIn leads, sync to Google Sheets and CRM\n\nAutomatically captures and syncs LinkedIn leads with intelligent scoring, duplicate prevention, and multi-platform integration.\n\n### How it works\n\n1. **Monitor LinkedIn** - Checks every 15 minutes for new connections and post engagements\n2. **Extract data** - Pulls profile information, company, position from multiple sources\n3. **Score leads** - Calculates quality score (0-100) and assigns temperature (Hot/Warm/Cold)\n4. **Prevent duplicates** - Tracks leads for 90 days to avoid duplicate entries\n5. **Sync to platforms** - Saves to Google Sheets and your CRM (HubSpot or Salesforce)\n6. **Smart notifications** - Alerts sales team via Slack/Email for Hot and Warm leads only\n\n### Setup steps\n\n1. **Connect LinkedIn** - Add OAuth2 credentials to both LinkedIn nodes\n2. **Setup Google Sheets** - Create spreadsheet with these columns: leadId, fullName, firstName, lastName, email, phone, company, position, headline, location, profileUrl, leadSource, leadTemperature, qualityScore, engagementType, commentText, dateAdded, status, assignedTo, notes, lastUpdated\n3. **Choose CRM** - Enable either HubSpot OR Salesforce nodes (disable the other)\n4. **Configure alerts** - Add Slack webhook URL and/or email settings\n5. **Test workflow** - Run manually first to verify all connections\n6. **Activate** - Turn on for automatic lead capture\n\n### Key features\n\n- **Smart scoring**: Quality score based on engagement level and profile completeness\n- **Temperature labels**: Hot (70+), Warm (50-69), Cold (<50) for prioritization\n- **No duplicates**: 90-day tracking prevents duplicate entries\n- **Multi-CRM support**: Works with HubSpot or Salesforce\n- **Analytics built-in**: Tracks total leads and temperature distribution |
| Fetch new LinkedIn connections | LinkedIn | Pull “connections/profile” stream | Check for new leads every 15 minutes | Merge connections and engagements | ## 1. Capture leads\n\nMonitors LinkedIn connections and post engagements |
| Fetch post engagement data | LinkedIn | Pull engagement stream | Check for new leads every 15 minutes | Merge connections and engagements | ## 1. Capture leads\n\nMonitors LinkedIn connections and post engagements |
| Merge connections and engagements | Merge | Combine the two LinkedIn streams | Fetch new LinkedIn connections; Fetch post engagement data | Extract and score lead data | ## 1. Capture leads\n\nMonitors LinkedIn connections and post engagements |
| Extract and score lead data | Code | Normalize fields + compute score/temperature | Merge connections and engagements | Filter out duplicate leads | ## 2. Process and score\n\nEnriches data, assigns quality scores, detects duplicates |
| Filter out duplicate leads | Code | Dedup using global static data (90 days) | Extract and score lead data | Route to create or update path | ## 2. Process and score\n\nEnriches data, assigns quality scores, detects duplicates |
| Route to create or update path | Switch | Branch to create vs update | Filter out duplicate leads | Append new lead to Google Sheets; Create new contact in HubSpot; Create new lead in Salesforce; Update existing lead in Google Sheets | ## 3. Sync to platforms\n\nSaves to Google Sheets and CRM with create/update logic |
| Append new lead to Google Sheets | Google Sheets | Append lead row | Route to create or update path (New Lead) | Build notification message | ## 3. Sync to platforms\n\nSaves to Google Sheets and CRM with create/update logic |
| Update existing lead in Google Sheets | Google Sheets | Update lead row | Route to create or update path (Update Existing) | Build notification message; Update existing contact in HubSpot | ## 3. Sync to platforms\n\nSaves to Google Sheets and CRM with create/update logic |
| Create new contact in HubSpot | HTTP Request | HubSpot contact create | Route to create or update path (New Lead) | Build notification message | ## 3. Sync to platforms\n\nSaves to Google Sheets and CRM with create/update logic |
| Update existing contact in HubSpot | HTTP Request | HubSpot contact update | Update existing lead in Google Sheets | Build notification message | ## 3. Sync to platforms\n\nSaves to Google Sheets and CRM with create/update logic |
| Create new lead in Salesforce | HTTP Request | Salesforce Lead create | Route to create or update path (New Lead) | Build notification message | ## 3. Sync to platforms\n\nSaves to Google Sheets and CRM with create/update logic |
| Build notification message | Code | Format notification + set shouldNotify | Append new lead to Google Sheets; Create new contact in HubSpot; Create new lead in Salesforce; Update existing lead in Google Sheets; Update existing contact in HubSpot | Filter for hot and warm leads only | ## 4. Notify sales team\n\nAlerts for hot and warm leads only with complete context |
| Filter for hot and warm leads only | Filter | Only allow Hot/Warm | Build notification message | Send Slack notification to sales team; Send email notification to sales team | ## 4. Notify sales team\n\nAlerts for hot and warm leads only with complete context |
| Send Slack notification to sales team | HTTP Request | Slack webhook alert | Filter for hot and warm leads only | Log success and track statistics | ## 4. Notify sales team\n\nAlerts for hot and warm leads only with complete context |
| Send email notification to sales team | Email Send | SMTP email alert | Filter for hot and warm leads only | Log success and track statistics | ## 4. Notify sales team\n\nAlerts for hot and warm leads only with complete context |
| Log success and track statistics | Code | Console log + global stats counters | Send Slack notification to sales team; Send email notification to sales team | — | ## Capture LinkedIn leads, sync to Google Sheets and CRM\n\nAutomatically captures and syncs LinkedIn leads with intelligent scoring, duplicate prevention, and multi-platform integration.\n\n### How it works\n\n1. **Monitor LinkedIn** - Checks every 15 minutes for new connections and post engagements\n2. **Extract data** - Pulls profile information, company, position from multiple sources\n3. **Score leads** - Calculates quality score (0-100) and assigns temperature (Hot/Warm/Cold)\n4. **Prevent duplicates** - Tracks leads for 90 days to avoid duplicate entries\n5. **Sync to platforms** - Saves to Google Sheets and your CRM (HubSpot or Salesforce)\n6. **Smart notifications** - Alerts sales team via Slack/Email for Hot and Warm leads only\n\n### Setup steps\n\n1. **Connect LinkedIn** - Add OAuth2 credentials to both LinkedIn nodes\n2. **Setup Google Sheets** - Create spreadsheet with these columns: leadId, fullName, firstName, lastName, email, phone, company, position, headline, location, profileUrl, leadSource, leadTemperature, qualityScore, engagementType, commentText, dateAdded, status, assignedTo, notes, lastUpdated\n3. **Choose CRM** - Enable either HubSpot OR Salesforce nodes (disable the other)\n4. **Configure alerts** - Add Slack webhook URL and/or email settings\n5. **Test workflow** - Run manually first to verify all connections\n6. **Activate** - Turn on for automatic lead capture\n\n### Key features\n\n- **Smart scoring**: Quality score based on engagement level and profile completeness\n- **Temperature labels**: Hot (70+), Warm (50-69), Cold (<50) for prioritization\n- **No duplicates**: 90-day tracking prevents duplicate entries\n- **Multi-CRM support**: Works with HubSpot or Salesforce\n- **Analytics built-in**: Tracks total leads and temperature distribution |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name: “Capture LinkedIn leads, sync to Google Sheets and CRM” (or your preferred name)  
   - Keep workflow **inactive** until tested.

2. **Add Schedule Trigger**
   - Node: **Schedule Trigger**
   - Set interval: **Every 15 minutes**

3. **Add LinkedIn node #1 (connections/profile source)**
   - Node: **LinkedIn**
   - Authentication: **OAuth2**
   - Operation: **getProfile**
   - Connect: Schedule Trigger → this node  
   - Credentials: create/select **LinkedIn OAuth2** credential

4. **Add LinkedIn node #2 (engagement source)**
   - Node: **LinkedIn**
   - Authentication: **OAuth2**
   - Operation: **getAll**
   - Connect: Schedule Trigger → this node (parallel to node #1)

5. **Add Merge node**
   - Node: **Merge**
   - Mode: **Combine**
   - Connect:
     - LinkedIn #1 → Merge (Input 1 / index 0)
     - LinkedIn #2 → Merge (Input 2 / index 1)

6. **Add Code node: “Extract and score lead data”**
   - Node: **Code**
   - Mode: **Run Once for Each Item**
   - Paste logic that:
     - Normalizes name/headline/company/location/profileUrl
     - Computes `qualityScore` and `leadTemperature`
     - Outputs a structured lead JSON (leadId, fullName, leadSource, etc.)
   - Connect: Merge → Code

7. **Add Code node: “Filter out duplicate leads”**
   - Node: **Code**
   - Mode: **Run Once for Each Item**
   - Implement:
     - `processedLeads` stored in `$getWorkflowStaticData('global')`
     - Unique key: `profileUrl` else `leadId`
     - If seen: set `action='update'` and attach `existingData`
     - Else: store entry + cleanup entries older than 90 days + set `action='create'`
   - Connect: Extract/score → Dedup code

8. **Add Switch node: “Route to create or update path”**
   - Node: **Switch**
   - Add 2 rules on `{{$json.action}}`:
     - Equals `create` → output “New Lead”
     - Equals `update` → output “Update Existing”
   - Connect: Dedup code → Switch

9. **Google Sheets (create path): Append**
   - Node: **Google Sheets**
   - Operation: **Append**
   - Document: select your spreadsheet (or paste the Spreadsheet ID)
   - Sheet: `Sheet1`
   - Map columns explicitly to your header names (leadId, fullName, firstName, …)
   - Credentials: **Google Sheets OAuth2**
   - Connect: Switch (“New Lead”) → Sheets Append

10. **Google Sheets (update path): Update**
    - Node: **Google Sheets**
    - Operation: **Update**
    - Document + Sheet same as above
    - Important: configure how to find the row:
      - Set **Matching Columns** to something unique (recommended: `leadId` or `profileUrl`)
      - Ensure those columns exist and are unique in the sheet
    - Credentials: **Google Sheets OAuth2**
    - Connect: Switch (“Update Existing”) → Sheets Update

11. **CRM option A (HubSpot) — Create contact**
    - Node: **HTTP Request**
    - Authentication: **Predefined Credential Type → HubSpot API**
    - Method: POST
    - URL: `https://api.hubspot.com/crm/v3/objects/contacts`
    - Body: JSON with `properties` mapping your lead fields
    - Connect: Switch (“New Lead”) → HubSpot Create
    - Note: create HubSpot properties if you use custom names like `linkedin_url`, `lead_temperature`, `quality_score`.

12. **CRM option A (HubSpot) — Update contact (fix required)**
    - Node: **HTTP Request**
    - Method: PATCH
    - URL must reference an actual HubSpot contact ID.
    - To make this functional, add one of these approaches:
      - **Store** HubSpot contact ID returned from “Create contact” into static data keyed by profileUrl/email, then use it in update, or
      - **Search HubSpot** by email (HTTP GET search endpoint) before patching.
    - Connect: Sheets Update (or a dedicated branch from Switch “Update Existing”) → HubSpot Update

13. **CRM option B (Salesforce) — Create lead**
    - Node: **HTTP Request**
    - Authentication: **Predefined Credential Type → Salesforce OAuth2**
    - Method: POST
    - URL: `https://<yourInstance>.salesforce.com/services/data/v58.0/sobjects/Lead/`
    - Body fields: FirstName, LastName, Email, Company, Title, LeadSource, Status, Rating, Description
    - Connect: Switch (“New Lead”) → Salesforce Create

14. **Choose one CRM**
    - Disable the CRM branch you do not want:
      - If using HubSpot, disable Salesforce node.
      - If using Salesforce, disable HubSpot nodes.
    - This avoids double-creating leads in two CRMs.

15. **Add Code node: “Build notification message”**
    - Node: **Code**
    - Mode: Run once per item
    - Build:
      - `notificationTitle`
      - `notificationMessage`
      - `shouldNotify = (leadTemperature is Hot or Warm)`
    - Connect to it from:
      - Sheets Append
      - Sheets Update
      - HubSpot Create (if enabled)
      - HubSpot Update (if enabled)
      - Salesforce Create (if enabled)

16. **Add Filter node**
    - Node: **Filter**
    - Condition: `{{$json.shouldNotify}}` is **true**
    - Connect: Build notification message → Filter

17. **Add Slack notification**
    - Node: **HTTP Request**
    - Method: POST
    - URL: your **Slack Incoming Webhook URL**
    - Body: includes `text` and `blocks` with `{{$json.notificationMessage}}`
    - Connect: Filter → Slack node

18. **Add Email notification**
    - Node: **Email Send**
    - Configure SMTP credentials
    - Set:
      - To
      - From
      - Subject: `{{$json.notificationTitle}}`
      - (Recommended) Body: `{{$json.notificationMessage}}`
    - Connect: Filter → Email node

19. **Add Code node: “Log success and track statistics”**
    - Node: **Code**
    - Mode: Run once per item
    - Use global static data `stats` to track totals and lastProcessed
    - Connect: Slack → Log node AND Email → Log node (both)

20. **Test**
    - Run workflow manually.
    - Validate:
      - LinkedIn nodes return items as expected
      - Merge produces expected item count
      - Google Sheets append works and columns match
      - Update path correctly finds rows
      - CRM create works (and update works if implemented properly)
      - Slack/email only for Warm/Hot

21. **Activate workflow** once stable.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow intent: monitor LinkedIn every 15 minutes, extract/score, deduplicate for 90 days, sync to Google Sheets + CRM, notify only Hot/Warm, track analytics | From the main sticky note content embedded in the workflow |
| Setup guidance: create Google Sheet columns: leadId, fullName, firstName, lastName, email, phone, company, position, headline, location, profileUrl, leadSource, leadTemperature, qualityScore, engagementType, commentText, dateAdded, status, assignedTo, notes, lastUpdated | From the main sticky note content embedded in the workflow |
| CRM choice: enable HubSpot OR Salesforce nodes and disable the other | From the main sticky note content embedded in the workflow |
| Implementation gap: HubSpot update expects `existingData.crmContactId`, but current dedup tracking does not populate it; you must add “store ID” or “search then update” logic | Derived from node dependency + parameters |

If you want, I can propose a minimal modification plan to make **Google Sheets update matching** and **HubSpot update** reliably work (with the smallest node changes).