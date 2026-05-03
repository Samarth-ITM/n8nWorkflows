Sync leads between Google Sheets and HubSpot contacts

https://n8nworkflows.xyz/workflows/sync-leads-between-google-sheets-and-hubspot-contacts-13460


# Sync leads between Google Sheets and HubSpot contacts

## 1. Workflow Overview

**Purpose:**  
This workflow synchronizes lead/contact rows from a **Google Sheets** spreadsheet into **HubSpot Contacts**. When run, it reads rows from a selected sheet and **creates or updates** HubSpot contact records, using **email** as the unique identifier to avoid duplicates.

**Target use cases:**
- Importing leads captured in Google Sheets into HubSpot
- Keeping HubSpot contacts aligned with a spreadsheet used by Sales/Marketing
- Bulk upsert (create-or-update) based on email

### 1.1 Execution Trigger
Starts the workflow manually in n8n.

### 1.2 Lead Retrieval (Google Sheets)
Reads contact lead data (rows) from a specified Google Sheet/tab.

### 1.3 HubSpot Upsert (Create/Update Contact)
For each row, maps spreadsheet columns to HubSpot contact properties and upserts the contact.

---

## 2. Block-by-Block Analysis

### Block 1 — Execution Trigger
**Overview:**  
Provides a manual entry point to run the workflow on demand for testing or ad-hoc sync.

**Nodes Involved:**
- Manual Trigger - Start Workflow

#### Node: Manual Trigger - Start Workflow
- **Type / role:** `Manual Trigger` — starts workflow execution manually from the n8n editor.
- **Configuration choices:** No parameters required.
- **Key expressions/variables:** None.
- **Input / output connections:**
  - **Output →** Get Contact Leads from Google Sheet
- **Version-specific requirements:** Type version **1**.
- **Edge cases / failures:**
  - None typical (only requires user to click “Execute workflow”).
- **Sub-workflow reference:** None.

---

### Block 2 — Lead Retrieval (Google Sheets)
**Overview:**  
Fetches rows from a configured spreadsheet and sheet tab. Each row becomes an item passed downstream.

**Nodes Involved:**
- Get Contact Leads from Google Sheet

#### Node: Get Contact Leads from Google Sheet
- **Type / role:** `Google Sheets` — reads data from a spreadsheet.
- **Configuration choices (interpreted):**
  - Uses OAuth2 connection to Google.
  - Targets a specific **Document** and **Sheet** (placeholders in this template):
    - Document ID: `GOOGLE_SHEET_DOCUMENT_ID_PLACEHOLDER`
    - Sheet/tab ID: `SHEET_ID_PLACEHOLDER` (tab name cached as `example_contacts`)
  - Options: none set (defaults apply).
- **Key expressions/variables:** None in this node; it outputs row fields as JSON keys based on column headers.
- **Input / output connections:**
  - **Input ←** Manual Trigger - Start Workflow
  - **Output →** Create or Update HubSpot Contact
- **Version-specific requirements:** Type version **4.7**.
- **Edge cases / failures:**
  - Google OAuth not connected / expired token → authentication failures.
  - Wrong documentId or sheet selection → “not found” / permission errors.
  - Column header mismatches (e.g., missing `Email`) will break downstream expressions expecting those fields.
  - Empty sheet or filtered result returning 0 items → HubSpot node won’t run (no items).
- **Sub-workflow reference:** None.

---

### Block 3 — HubSpot Upsert (Create/Update Contact)
**Overview:**  
Upserts a HubSpot contact per input item by using the email field; maps additional spreadsheet columns to HubSpot properties.

**Nodes Involved:**
- Create or Update HubSpot Contact

#### Node: Create or Update HubSpot Contact
- **Type / role:** `HubSpot` — create or update a CRM contact.
- **Configuration choices (interpreted):**
  - Authentication method: **App Token** (private app access token).
  - Upsert identifier: **Email**
  - Field mappings:
    - Email: `{{ $json.Email }}`
    - First name: `{{ $json.Name }}`
    - Company name: `{{ $json['Company Name'] }}`
    - Phone number: `{{ $json['Phone Number'] }}`
- **Key expressions/variables used:**
  - `$json.Email`
  - `$json.Name`
  - `$json['Company Name']`
  - `$json['Phone Number']`
- **Input / output connections:**
  - **Input ←** Get Contact Leads from Google Sheet
  - **Output →** (none; end of workflow)
- **Version-specific requirements:** Type version **2.2**.
- **Edge cases / failures:**
  - Missing/blank/invalid email in a row → upsert may fail or create unexpected results depending on HubSpot API constraints.
  - HubSpot private app token missing/invalid → 401/403 errors.
  - Required scopes not granted → 403 “missing scopes”.
  - Rate limits for bulk imports → possible 429 responses on large sheets.
  - Column names containing extra spaces or different casing will cause expressions like `$json.Email` to be undefined.
- **Sub-workflow reference:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger - Start Workflow | n8n-nodes-base.manualTrigger | Manual execution entry point | — | Get Contact Leads from Google Sheet |  |
| Get Contact Leads from Google Sheet | n8n-nodes-base.googleSheets | Read lead rows from Google Sheets | Manual Trigger - Start Workflow | Create or Update HubSpot Contact |  |
| Create or Update HubSpot Contact | n8n-nodes-base.hubspot | Upsert HubSpot contacts based on email | Get Contact Leads from Google Sheet | — |  |
| Workflow Description | n8n-nodes-base.stickyNote | Documentation / setup notes | — | — | ## Workflow Overview; Imports leads from Google Sheets to HubSpot; prevents duplicates by matching email. HubSpot setup: create developer account at `developers.hubspot.com`, create private app (Legacy Apps), add scopes `crm.objects.contacts.read`, `crm.objects.contacts.write`, `crm.objects.companies.read`, `crm.objects.companies.write`, copy access token, create n8n HubSpot credential using APP Token. Google Sheets: connect Google account in Google Sheets node. Configure document+sheet and adjust field mappings (Email, Company Name, Name, Phone Number). |
| Video Walkthrough | n8n-nodes-base.stickyNote | Reference link | — | — | # Video Walkthrough: Thumbnail: https://vasarmilan-public.s3.us-east-1.amazonaws.com/blog_thumbnails/thumbnail_reczH8mtRj3GIIRiU.jpg — Video: https://youtu.be/GBKXYh2j74o |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add node:** `Manual Trigger`
   - Name it: **Manual Trigger - Start Workflow**
   - No configuration needed.

3. **Add node:** `Google Sheets`
   - Name it: **Get Contact Leads from Google Sheet**
   - **Credentials:** create/select **Google Sheets OAuth2** credential
     - Connect the Google account that has access to the spreadsheet.
   - **Operation:** configure to **read rows** from a sheet (use n8n’s default “Read”/“Get All” style operation as available in your version).
   - **Document:** select your spreadsheet (or paste the **Document ID**).
   - **Sheet:** select the tab that contains your leads.
   - Ensure your sheet has column headers matching the fields you will map (at minimum: `Email`). In this template the expected headers are:
     - `Email`
     - `Name`
     - `Company Name`
     - `Phone Number`

4. **Add node:** `HubSpot`
   - Name it: **Create or Update HubSpot Contact**
   - **Credentials:** create/select **HubSpot App Token** credential
     - In HubSpot:
       1. Create a developer account: `https://developers.hubspot.com`
       2. Go to **Legacy Apps** → create a **private app**
       3. Add scopes:
          - `crm.objects.contacts.read`
          - `crm.objects.contacts.write`
          - `crm.objects.companies.read`
          - `crm.objects.companies.write`
       4. Copy the **Access Token**
     - In n8n: choose authentication method **App Token** and paste the token.
   - **Operation:** choose the action that **creates or updates a contact** (upsert).
   - **Email field:** set to expression `{{ $json.Email }}`
   - **Additional fields** (map as expressions):
     - First Name → `{{ $json.Name }}`
     - Company Name → `{{ $json['Company Name'] }}`
     - Phone Number → `{{ $json['Phone Number'] }}`

5. **Connect the nodes in order:**
   - Manual Trigger - Start Workflow → Get Contact Leads from Google Sheet
   - Get Contact Leads from Google Sheet → Create or Update HubSpot Contact

6. **Run and validate:**
   - Execute the workflow manually.
   - Inspect a few items after the Google Sheets node to confirm the JSON keys match the expected column headers.
   - Confirm contacts are created/updated in HubSpot.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| HubSpot private app setup and required scopes: `crm.objects.contacts.read`, `crm.objects.contacts.write`, `crm.objects.companies.read`, `crm.objects.companies.write` | From sticky note “Workflow Description”; HubSpot developer portal: `https://developers.hubspot.com` |
| Video walkthrough (YouTube): https://youtu.be/GBKXYh2j74o | From sticky note “Video Walkthrough” |
| Walkthrough thumbnail image: https://vasarmilan-public.s3.us-east-1.amazonaws.com/blog_thumbnails/thumbnail_reczH8mtRj3GIIRiU.jpg | From sticky note “Video Walkthrough” |