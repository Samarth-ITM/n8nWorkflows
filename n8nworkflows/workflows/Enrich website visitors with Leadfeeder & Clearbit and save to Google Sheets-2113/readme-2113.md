Enrich website visitors with Leadfeeder & Clearbit and save to Google Sheets

https://n8nworkflows.xyz/workflows/enrich-website-visitors-with-leadfeeder---clearbit-and-save-to-google-sheets-2113


# Enrich website visitors with Leadfeeder & Clearbit and save to Google Sheets

---

# 1. Workflow Overview

This n8n workflow automates the enrichment and tracking of website visitors as potential leads by integrating Leadfeeder, Clearbit, and Google Sheets. Targeted for sales and marketing teams aiming to maximize outreach, it runs once daily to extract leads from Leadfeeder, apply engagement criteria, enrich company data with Clearbit, filter by company criteria, and finally save qualified leads to a Google Sheet for further action.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Setup:** Initiates the workflow daily and configures essential parameters like Leadfeeder accounts and target Google Sheets URL.
- **1.2 Leadfeeder Account Retrieval & Filtering:** Retrieves all Leadfeeder accounts and filters only those specified in the setup.
- **1.3 Lead Retrieval & Splitting:** Fetches leads from each selected Leadfeeder account and splits the lead data for individual processing.
- **1.4 Lead Engagement Filtering:** Applies an engagement filter (e.g., minimum number of visits) to select relevant leads.
- **1.5 Company Data Enrichment:** Enriches filtered leads’ companies with Clearbit data.
- **1.6 Company Criteria Filtering:** Filters enriched companies based on criteria such as minimum employee count.
- **1.7 Save to Google Sheets:** Appends or updates the filtered, enriched lead data to a specified Google Sheet.

---

# 2. Block-by-Block Analysis

### 2.1 Scheduled Trigger & Setup

- **Overview:**  
  This block starts the workflow at a configured hour daily and sets up key variables such as Leadfeeder account names and the Google Sheets URL for saving results.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Setup  
  - Sticky Note (Setup instructions)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow once per day at 7 AM.  
    - Configuration: Trigger at hour 7 (local time).  
    - Inputs: None (trigger node).  
    - Outputs: Connected to Setup node.  
    - Edge Cases: If n8n server time zone mismatches expected time zone, timing may differ.

  - **Setup**  
    - Type: Set  
    - Role: Defines static parameters used downstream.  
    - Configuration:  
      - `Leadfeeder Accounts`: Array of strings specifying monitored Leadfeeder account names (e.g., ["n8n", "someOtherAccount"]).  
      - `Google Sheets URL`: String with URL to the target Google Sheets document.  
    - Inputs: From Schedule Trigger.  
    - Outputs: To "Get all Leedfeeder accounts" node.  
    - Edge Cases: Invalid or missing URLs or account names will cause filtering downstream to fail or be empty.

  - **Sticky Note (Setup Instructions)**  
    - Content: Details credential setup and copy instructions for Google Sheet template.  
    - Role: User guidance.  
    - No inputs/outputs.

---

### 2.2 Leadfeeder Account Retrieval & Filtering

- **Overview:**  
  Retrieves all Leadfeeder accounts via API and filters to only those listed in the Setup node.

- **Nodes Involved:**  
  - Get all Leedfeeder accounts  
  - Split out accounts  
  - Only for wanted accounts

- **Node Details:**

  - **Get all Leedfeeder accounts**  
    - Type: HTTP Request  
    - Role: Calls Leadfeeder API to fetch all accounts accessible by the API token.  
    - Configuration:  
      - URL: `https://api.leadfeeder.com/accounts`  
      - Authentication: HTTP Header Auth with Leadfeeder token.  
    - Inputs: From Setup node.  
    - Outputs: To Split out accounts.  
    - Edge Cases: API token invalid or expired → authentication error; API downtime.

  - **Split out accounts**  
    - Type: Split Out  
    - Role: Splits the array of accounts into individual items for filtering.  
    - Configuration: Splits on field `data` (the accounts array).  
    - Inputs: From "Get all Leedfeeder accounts".  
    - Outputs: To Only for wanted accounts.  
    - Edge Cases: Empty accounts list → no outputs.

  - **Only for wanted accounts**  
    - Type: Filter  
    - Role: Keeps only accounts whose names are included in the Setup node’s `Leadfeeder Accounts` array.  
    - Configuration: Boolean filter checking if `Setup` node’s array includes the current account’s `attributes.name`.  
    - Inputs: From Split out accounts.  
    - Outputs: To Get Leads node.  
    - Edge Cases: If no accounts match, no leads will be fetched downstream.

---

### 2.3 Lead Retrieval & Splitting

- **Overview:**  
  For each filtered Leadfeeder account, fetches leads from the last day and splits them for individual processing.

- **Nodes Involved:**  
  - Get Leads  
  - Split Out Leads

- **Node Details:**

  - **Get Leads**  
    - Type: HTTP Request  
    - Role: Fetches leads for a Leadfeeder account filtered by date range (previous day to today).  
    - Configuration:  
      - URL: `https://api.leadfeeder.com/accounts/{{ $json.id }}/leads` (dynamic account ID)  
      - Query parameters:  
        - `start_date`: yesterday’s date (ISO format)  
        - `end_date`: today’s date (ISO format)  
      - Authentication: HTTP Header Auth with Leadfeeder token.  
    - Inputs: From Only for wanted accounts.  
    - Outputs: To Split Out Leads.  
    - Edge Cases: API rate limits, invalid account ID, no leads returned.

  - **Split Out Leads**  
    - Type: Split Out  
    - Role: Splits leads array into individual lead objects for further filtering.  
    - Configuration: Splits on field `data`.  
    - Inputs: From Get Leads.  
    - Outputs: To Filter leads by engagement.  
    - Edge Cases: Empty leads array → no outputs.

---

### 2.4 Lead Engagement Filtering

- **Overview:**  
  Filters leads by engagement criteria such as minimum number of visits.

- **Nodes Involved:**  
  - Filter leads by engagement  
  - Sticky Note (Engagement criteria)

- **Node Details:**

  - **Filter leads by engagement**  
    - Type: Filter  
    - Role: Keeps leads where `attributes.visits` > 3 (configurable threshold).  
    - Configuration: Numeric comparison, left value is `$json.attributes.visits`, right value is 3.  
    - Inputs: From Split Out Leads.  
    - Outputs: To Enrich company node.  
    - Edge Cases: Missing `visits` attribute → filter fails or excludes lead; non-numeric values.

  - **Sticky Note (Engagement criteria)**  
    - Content: "Adjust your engagement criteria here"  
    - Role: User guidance on filter threshold adjustment.

---

### 2.5 Company Data Enrichment

- **Overview:**  
  Enriches each filtered lead’s company data by querying Clearbit using the company’s domain.

- **Nodes Involved:**  
  - Enrich company

- **Node Details:**

  - **Enrich company**  
    - Type: Clearbit node (API integration)  
    - Role: Retrieves enriched company info from Clearbit based on domain.  
    - Configuration:  
      - Domain: dynamic expression `{{$json.attributes.website_url}}` (lead’s website URL)  
      - No additional fields configured.  
      - Credentials: Clearbit API key configured.  
    - Inputs: From Filter leads by engagement.  
    - Outputs: To Filter Leads by company criteria.  
    - Edge Cases: Clearbit API limits, domain missing or invalid, API errors.

---

### 2.6 Company Criteria Filtering

- **Overview:**  
  Filters enriched companies by company-specific criteria, such as minimum employee count.

- **Nodes Involved:**  
  - Filter Leads by company criteria  
  - Sticky Note (Company criteria)

- **Node Details:**

  - **Filter Leads by company criteria**  
    - Type: Filter  
    - Role: Keeps only companies with `metrics.employees` > 100.  
    - Configuration: Numeric comparison on `metrics.employees`.  
    - Inputs: From Enrich company.  
    - Outputs: To Save leads to Google Sheets.  
    - Edge Cases: Missing `metrics.employees` value, non-numeric, API enrichment failures.

  - **Sticky Note (Company criteria)**  
    - Content: "Adjust your company criteria here"  
    - Role: Guidance for modifying employee count or other filters.

---

### 2.7 Save to Google Sheets

- **Overview:**  
  Appends or updates filtered and enriched leads into a Google Sheet for record keeping and further processing.

- **Nodes Involved:**  
  - Save leads to Google Sheets

- **Node Details:**

  - **Save leads to Google Sheets**  
    - Type: Google Sheets node  
    - Role: Appends or updates rows in a specified Google Sheet tab based on lead domain matching.  
    - Configuration:  
      - Operation: Append or update  
      - Document ID: Taken dynamically from `Setup` node's Google Sheets URL  
      - Sheet Name: Fixed to `gid=0` (default first tab)  
      - Columns mapped include:  
        - `name`, `domain`, `visits`, `quality`, `twitter`, `linkedin`, `employees`, `description`  
      - Uses fallback logic to handle missing social handles or employee counts from either Clearbit or Leadfeeder data.  
    - Inputs: From Filter Leads by company criteria.  
    - Outputs: None (end of workflow).  
    - Credentials: Google Sheets OAuth2 API configured.  
    - Edge Cases: Google API quota limits, invalid document URL, permission issues, data type mismatches.

---

# 3. Summary Table

| Node Name                   | Node Type          | Functional Role                        | Input Node(s)                | Output Node(s)                   | Sticky Note                                                                                                      |
|-----------------------------|--------------------|-------------------------------------|-----------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger   | Initiate workflow daily at 7 AM     | None                        | Setup                           | Adjust the schedule here                                                                                        |
| Setup                      | Set                | Define static parameters             | Schedule Trigger            | Get all Leedfeeder accounts     | 1. Add Leadfeeder credentials 2. Add Google Sheet credentials 3. Save Leadfeeder accounts in Setup 4. Copy Google Sheets Template (https://docs.google.com/spreadsheets/d/1a2gfBjZZpN0jiD7apR8fPplRp2aPHVy2_5lp4Yzp778/edit?usp=sharing) and add its URL to Setup |
| Get all Leedfeeder accounts | HTTP Request       | Retrieve Leadfeeder accounts         | Setup                       | Split out accounts              |                                                                                                                  |
| Split out accounts          | Split Out          | Split accounts array                  | Get all Leedfeeder accounts | Only for wanted accounts         |                                                                                                                  |
| Only for wanted accounts    | Filter             | Filter accounts by Setup list        | Split out accounts          | Get Leads                      |                                                                                                                  |
| Get Leads                  | HTTP Request       | Fetch leads for each account          | Only for wanted accounts    | Split Out Leads                |                                                                                                                  |
| Split Out Leads            | Split Out          | Split leads array                     | Get Leads                  | Filter leads by engagement      |                                                                                                                  |
| Filter leads by engagement | Filter             | Filter leads by minimum visits (>3)  | Split Out Leads            | Enrich company                 | Adjust your engagement criteria here                                                                             |
| Enrich company             | Clearbit           | Enrich lead company data              | Filter leads by engagement  | Filter Leads by company criteria |                                                                                                                  |
| Filter Leads by company criteria | Filter        | Filter companies by employee count (>100) | Enrich company             | Save leads to Google Sheets     | Adjust your company criteria here                                                                                |
| Save leads to Google Sheets | Google Sheets      | Append/update leads in Google Sheet  | Filter Leads by company criteria | None                      |                                                                                                                  |
| Sticky Note                | Sticky Note        | Setup instructions                    | None                        | None                           | See Setup node sticky note for details                                                                           |
| Sticky Note1               | Sticky Note        | Engagement criteria guidance          | None                        | None                           | Adjust your engagement criteria here                                                                             |
| Sticky Note2               | Sticky Note        | Company criteria guidance             | None                        | None                           | Adjust your company criteria here                                                                                |
| Sticky Note3               | Sticky Note        | Schedule guidance                     | None                        | None                           | Adjust the schedule here                                                                                        |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node:**  
   - Set trigger time to 7:00 AM daily (local time).  
   - No credentials needed.

2. **Create a Set Node named "Setup":**  
   - Add two parameters:  
     - `Leadfeeder Accounts` (Array): e.g., `["n8n", "someOtherAccount"]`  
     - `Google Sheets URL` (String): Paste your Google Sheets URL (copied from the provided template or your own sheet).  
   - Connect Schedule Trigger output to Setup input.

3. **Create HTTP Request Node "Get all Leedfeeder accounts":**  
   - Method: GET  
   - URL: `https://api.leadfeeder.com/accounts`  
   - Authentication: HTTP Header Auth with Leadfeeder API token (see Setup sticky note for token location).  
   - Connect Setup output to this node.

4. **Create Split Out Node "Split out accounts":**  
   - Field to split out: `data`  
   - Connect "Get all Leedfeeder accounts" output to this node.

5. **Create Filter Node "Only for wanted accounts":**  
   - Condition: Boolean true if `{{ $('Setup').first().json["Leadfeeder Accounts"].includes($json.attributes.name) }}`  
   - Connect "Split out accounts" output to this node.

6. **Create HTTP Request Node "Get Leads":**  
   - Method: GET  
   - URL: `https://api.leadfeeder.com/accounts/{{ $json.id }}/leads`  
   - Authentication: HTTP Header Auth with Leadfeeder API token.  
   - Query parameters:  
     - `start_date`: `={{ $now.minus(1, 'day').toFormat('yyyy-MM-dd') }}`  
     - `end_date`: `={{ $now.toFormat('yyyy-MM-dd') }}`  
   - Connect "Only for wanted accounts" output to this node.

7. **Create Split Out Node "Split Out Leads":**  
   - Field to split out: `data`  
   - Connect "Get Leads" output to this node.

8. **Create Filter Node "Filter leads by engagement":**  
   - Condition: Numeric comparison, keep leads where `{{$json.attributes.visits}} > 3` (adjust threshold as needed).  
   - Connect "Split Out Leads" output to this node.

9. **Create Clearbit Node "Enrich company":**  
   - Domain parameter: `={{ $json.attributes.website_url }}`  
   - Credentials: Configure Clearbit API key.  
   - Connect "Filter leads by engagement" output to this node.

10. **Create Filter Node "Filter Leads by company criteria":**  
    - Condition: Numeric comparison, keep if `{{$json.metrics.employees}} > 100` (adjust as needed).  
    - Connect "Enrich company" output to this node.

11. **Create Google Sheets Node "Save leads to Google Sheets":**  
    - Operation: Append or update  
    - Document ID: `={{ $('Setup').first().json["Google Sheets URL"] }}`  
    - Sheet Name: Use `gid=0` (first tab) or your target tab name.  
    - Columns: Map fields such as `name`, `domain`, `visits`, `quality`, `twitter`, `linkedin`, `employees`, `description` with fallback expressions pulling data from Clearbit or Leadfeeder as per original workflow.  
    - Credentials: Configure Google Sheets OAuth2 credentials.  
    - Connect "Filter Leads by company criteria" output to this node.

12. **Optionally add Sticky Notes to document:**  
    - Setup instructions, engagement criteria reminder, company criteria reminder, and schedule info for user clarity.

Make sure all credentials are properly configured:  
- **Leadfeeder**: HTTP Header Auth with header `Authorization: Token token=yourapitoken`.  
- **Google Sheets**: OAuth2 credentials with edit access to the target spreadsheet.  
- **Clearbit**: API key configured in Clearbit node credentials.

---

# 5. General Notes & Resources

| Note Content                                                                                                                                                                    | Context or Link                                                                                                                                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Leadfeeder API token can be found under **Settings -> Personal -> API-Token** in Leadfeeder dashboard.                                                                          | Credential setup for Leadfeeder API                                                                                                                              |
| Google Sheets Template to use with this workflow: [https://docs.google.com/spreadsheets/d/1a2gfBjZZpN0jiD7apR8fPplRp2aPHVy2_5lp4Yzp778/edit?usp=sharing](https://docs.google.com/spreadsheets/d/1a2gfBjZZpN0jiD7apR8fPplRp2aPHVy2_5lp4Yzp778/edit?usp=sharing) | Predefined sheet with columns for leads                                                                                                                          |
| To enhance this workflow, consider adding auto outreach workflows or employee discovery workflows based on enriched leads.                                                     | Suggested project expansions                                                                                                                                    |
| Engagement and company criteria are fully customizable by editing the corresponding Filter nodes.                                                                               | Flexibility to adjust lead qualification                                                                                                                        |
| The workflow respects API rate limits and expects proper error handling at the n8n level (e.g., retry or failure nodes can be added if desired).                               | Operational considerations                                                                                                                                       |

---

This documentation fully describes the workflow’s logic, configuration, and implementation, enabling confident reproduction, modification, and extension in n8n.