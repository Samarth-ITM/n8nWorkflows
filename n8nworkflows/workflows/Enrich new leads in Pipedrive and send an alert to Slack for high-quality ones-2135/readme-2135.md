Enrich new leads in Pipedrive and send an alert to Slack for high-quality ones

https://n8nworkflows.xyz/workflows/enrich-new-leads-in-pipedrive-and-send-an-alert-to-slack-for-high-quality-ones-2135


# Enrich new leads in Pipedrive and send an alert to Slack for high-quality ones

### 1. Workflow Overview

This workflow automates the enrichment of new leads in Pipedrive using Clearbit data and sends alerts to Slack for leads matching high-quality criteria. It targets sales and marketing teams who want to automate lead qualification and reduce manual review effort. The workflow runs every 5 minutes, fetching new leads from Pipedrive, enriching lead company data via Clearbit, updating the lead records, and notifying Slack channels for leads that meet specified criteria.

Logical blocks:

- **1.1 Scheduled Trigger & Setup:** Periodically initiates the workflow and sets essential configuration parameters including Slack channel and custom field IDs.

- **1.2 Lead Retrieval & Organization Details Fetch:** Retrieves all new leads from Pipedrive, fetches detailed organization data for each lead, and prepares data for enrichment.

- **1.3 Company Enrichment via Clearbit:** Uses the Clearbit API to enrich company data based on the organization's domain.

- **1.4 Data Merging & Lead Marking:** Merges enriched data back with Pipedrive leads and marks leads as enriched with a timestamp.

- **1.5 Lead Filtering & Slack Notification:** Filters leads based on quality criteria (e.g., B2B tag and employee count), then sends alerts to a Slack channel for leads that qualify.

- **1.6 Setup Assistance Nodes:** Helper nodes to identify custom field IDs from Pipedrive for domain and enrichment date fields.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Setup

**Overview:**  
This block triggers the workflow every 5 minutes and sets up key parameters such as Slack channel and custom field IDs necessary for subsequent API calls.

**Nodes Involved:**  
- Trigger every 5 minutes  
- Setup  
- Sticky Note (Setup instructions)

**Node Details:**

- **Trigger every 5 minutes**  
  - Type: Schedule Trigger  
  - Configuration: Runs every 5 minutes (interval set to minutes)  
  - Inputs: None (start node)  
  - Outputs: Connects to Setup node  
  - Possible failures: Scheduling misconfiguration (unlikely)  

- **Setup**  
  - Type: Set  
  - Configuration: Defines static parameters including Slack channel name (`#yourChannel`), placeholders for custom field IDs (`domainCustomFieldId`, `enrichedAtCustomFieldId`), and filled IDs (`domainCustomFieldId2`, `enrichedAtCustomFieldId2`) used later  
  - Inputs: From Trigger node  
  - Outputs: Connects to "Get all leads" node  
  - Edge cases: Misconfigured or missing custom field IDs break enrichment and updates  

- **Sticky Note (Setup instructions)**  
  - Content: Detailed instructions on adding custom fields in Pipedrive and filling credential information  
  - Purpose: User guidance only; no data processing  

---

#### 1.2 Lead Retrieval & Organization Details Fetch

**Overview:**  
Fetches all non-archived leads from Pipedrive, retrieves detailed organization information for each lead, preparing data for enrichment.

**Nodes Involved:**  
- Get all leads  
- Get organization details  
- Add Organization ID to data  
- Merge data

**Node Details:**

- **Get all leads**  
  - Type: Pipedrive node (resource: lead, operation: getAll)  
  - Configuration: Filters for non-archived leads only  
  - Inputs: From Setup node  
  - Outputs: Connects to Get organization details and Merge data nodes  
  - Common errors: API rate limits, authentication failures  

- **Get organization details**  
  - Type: Pipedrive node (resource: organization, operation: get)  
  - Configuration: Uses dynamic organization ID from lead data (`{{$json.organization_id}}`)  
  - Inputs: From Get all leads  
  - Outputs: Connects to Enrich company node  
  - Possible failures: Missing or invalid organization ID, API errors  

- **Add Organization ID to data**  
  - Type: Set  
  - Configuration: Adds `organization_id` field with value from "Get organization details" node  
  - Inputs: From Enrich company node  
  - Outputs: Connects to Merge data node  
  - Edge cases: Missing or invalid data propagation  

- **Merge data**  
  - Type: Merge  
  - Configuration: Enrich merge mode, joining datasets by `organization_id`, resolving conflicts by preferring second input  
  - Inputs: Two inputs — enriched company data and lead data with organization ID  
  - Outputs: Connects to Mark lead as enriched node  
  - Failure modes: Misaligned keys causing merge failures  

---

#### 1.3 Company Enrichment via Clearbit

**Overview:**  
Uses Clearbit API to enrich company data based on the organization's domain field from Pipedrive.

**Nodes Involved:**  
- Enrich company

**Node Details:**

- **Enrich company**  
  - Type: Clearbit node  
  - Configuration: Uses domain dynamically fetched from `Setup` node's custom domain field ID (`{{$json[$('Setup').first().json.domainCustomFieldId2]}}`)  
  - Credentials: Clearbit API credentials required  
  - Inputs: From Get organization details  
  - Outputs: Connects to Add Organization ID to data node  
  - Edge cases: API rate limits, missing domain, invalid domain, Clearbit service downtime  

---

#### 1.4 Data Merging & Lead Marking

**Overview:**  
Merges enriched company data with Pipedrive lead data, updates the lead in Pipedrive to mark it as enriched by setting the enrichment date field.

**Nodes Involved:**  
- Merge data  
- Mark lead as enriched in Pipedrive

**Node Details:**

- **Merge data** (see above)  
  - Combines lead data and enriched company data for update  

- **Mark lead as enriched in Pipedrive**  
  - Type: HTTP Request  
  - Configuration: PATCH request to Pipedrive API updating lead with enrichment timestamp field (`enrichedAtCustomFieldId2`) set to current date in 'yyyy-MM-dd' format  
  - Authentication: Pipedrive API credentials  
  - Inputs: From Merge data  
  - Outputs: Connects to Keep leads that match the criteria node  
  - Edge cases: API errors, invalid field ID, network issues, date formatting errors  

---

#### 1.5 Lead Filtering & Slack Notification

**Overview:**  
Filters leads based on business criteria (B2B tag and employee count > 100), and sends a formatted alert message to a Slack channel for each qualifying lead.

**Nodes Involved:**  
- Keep leads that match the criteria  
- Send alert to Slack  
- Sticky Note (criteria adjustment guidance)

**Node Details:**

- **Keep leads that match the criteria**  
  - Type: Filter  
  - Configuration:  
    - Condition 1: `$json.tags.includes("B2B")` must be true  
    - Condition 2: `$json.metrics.employees > 100`  
  - Inputs: From Mark lead as enriched node  
  - Outputs: Leads matching criteria are forwarded to Slack notification  
  - Edge cases: Missing tags or metrics properties, case sensitivity issues, empty arrays  

- **Send alert to Slack**  
  - Type: Slack node  
  - Configuration:  
    - Sends message to channel defined in Setup node (`{{$json.slackChannel}}`)  
    - Message includes company name, website URL, estimated annual revenue, and employee count using template expressions  
  - Credentials: Slack OAuth2 credentials required  
  - Inputs: From filter node  
  - Outputs: None (end of workflow branch)  
  - Edge cases: Slack API rate limits, invalid channel, missing data fields, credential expiration  

- **Sticky Note (criteria adjustment guidance)**  
  - Content: Suggests modifying the filter condition to suit user-specific lead definitions (e.g., revenue, employees)  

---

#### 1.6 Setup Assistance Nodes

**Overview:**  
These nodes help users extract custom field IDs from Pipedrive by querying organization and lead field metadata.

**Nodes Involved:**  
- Get all organization keys  
- Split out organization field  
- Show only custom organization fields  
- Get all lead keys  
- Split out lead field data  
- Show only custom lead fields  
- Sticky Notes (run to find IDs)

**Node Details:**

- **Get all organization keys**  
  - Type: HTTP Request  
  - Configuration: GET request to `https://api.pipedrive.com/v1/organizationFields`  
  - Authentication: Pipedrive API credentials  
  - Outputs: Connects to Split out organization field  

- **Split out organization field**  
  - Type: Split Out  
  - Configuration: Splits the `data` array from previous response into individual items  
  - Outputs: Connects to Show only custom organization fields  

- **Show only custom organization fields**  
  - Type: Filter  
  - Configuration: Filters organization fields where `edit_flag` is true (custom fields)  
  - Outputs: None (for user inspection)  

- **Get all lead keys**  
  - Type: HTTP Request  
  - Configuration: GET request to `https://api.pipedrive.com/v1/leadFields`  
  - Authentication: Pipedrive API credentials  
  - Outputs: Connects to Split out lead field data  

- **Split out lead field data**  
  - Type: Split Out  
  - Configuration: Splits the `data` array into individual lead field items  
  - Outputs: Connects to Show only custom lead fields  

- **Show only custom lead fields**  
  - Type: Filter  
  - Configuration: Filters lead fields where `edit_flag` is true (custom fields)  
  - Outputs: None (for user inspection)  

- **Sticky Notes (Run me to find the Id of your custom fields)**  
  - Provide user instructions to run these nodes to retrieve and copy custom field keys needed in Setup node  

---

### 3. Summary Table

| Node Name                       | Node Type            | Functional Role                          | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                    |
|--------------------------------|----------------------|----------------------------------------|-------------------------------|--------------------------------|---------------------------------------------------------------|
| Trigger every 5 minutes         | Schedule Trigger     | Initiates workflow every 5 minutes      | None                          | Setup                          |                                                               |
| Setup                          | Set                  | Defines parameters and field IDs        | Trigger every 5 minutes        | Get all leads                  | See detailed setup instructions in Sticky Note                |
| Get all leads                  | Pipedrive            | Retrieves all non-archived leads        | Setup                         | Get organization details, Merge data |                                                               |
| Get organization details       | Pipedrive            | Gets detailed info for each lead's org | Get all leads                 | Enrich company                 |                                                               |
| Enrich company                 | Clearbit             | Enriches company data by domain          | Get organization details      | Add Organization ID to data    |                                                               |
| Add Organization ID to data     | Set                  | Adds org ID field for merging           | Enrich company                | Merge data                    |                                                               |
| Merge data                    | Merge                | Combines lead and enriched company data | Get all leads, Add Org ID to data | Mark lead as enriched in Pipedrive |                                                               |
| Mark lead as enriched in Pipedrive | HTTP Request         | Updates lead record with enrichment date | Merge data                   | Keep leads that match the criteria |                                                               |
| Keep leads that match the criteria | Filter               | Filters leads based on B2B tag and employees | Mark lead as enriched in Pipedrive | Send alert to Slack          | Adjust condition to filter leads by your desired condition    |
| Send alert to Slack            | Slack                | Sends notification for qualifying leads | Keep leads that match the criteria | None                         |                                                               |
| Get all organization keys      | HTTP Request         | Retrieves organization custom fields    | None                          | Split out organization field   | Run me to find the Id of your custom domain field             |
| Split out organization field   | Split Out            | Splits org fields array into items      | Get all organization keys     | Show only custom organization fields | Run me to find the Id of your custom domain field             |
| Show only custom organization fields | Filter               | Filters for editable (custom) org fields | Split out organization field  | None                         | Run me to find the Id of your custom domain field             |
| Get all lead keys              | HTTP Request         | Retrieves lead custom fields             | None                          | Split out lead field data      | Run me to find the Id of your enriched at domain field        |
| Split out lead field data      | Split Out            | Splits lead fields array into items      | Get all lead keys             | Show only custom lead fields   | Run me to find the Id of your enriched at domain field        |
| Show only custom lead fields   | Filter               | Filters for editable (custom) lead fields | Split out lead field data      | None                         | Run me to find the Id of your enriched at domain field        |
| Sticky Note (Setup instructions) | Sticky Note          | Provides setup instructions              | None                          | None                         | See note content in Block 1.1                                 |
| Sticky Note (Criteria adjustment) | Sticky Note          | Advises on modifying filter criteria     | None                          | None                         | Adjust condition to filter leads by your desired condition    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set it to trigger every 5 minutes.

2. **Create a Set node named "Setup":**  
   - Add fields:  
     - `slackChannel` (string): e.g., `#yourChannel`  
     - `domainCustomFieldId` (string): placeholder `<Run "Show only custom organization fields" and copy the key>`  
     - `enrichedAtCustomFieldId` (string): placeholder `<Run "Show only custom lead fields" and copy the key>`  
     - `domainCustomFieldId2` (string): fill with actual custom domain field ID (e.g., `ab26f671c92146268edacd244181e76579286e71`)  
     - `enrichedAtCustomFieldId2` (string): fill with actual enriched date custom field ID (e.g., `68a15ecb2e1255250617c1fd1c06385893334e3c`)  

3. **Connect Trigger node to Setup node.**

4. **Create a Pipedrive node "Get all leads":**  
   - Resource: Lead  
   - Operation: Get All  
   - Filter: `archived_status = not_archived`  
   - Connect Setup -> Get all leads  

5. **Create a Pipedrive node "Get organization details":**  
   - Resource: Organization  
   - Operation: Get  
   - Organization ID: Use expression `{{$json.organization_id}}` from the lead data  
   - Connect Get all leads -> Get organization details  

6. **Create a Clearbit node "Enrich company":**  
   - Set domain field to expression: `{{$json[$('Setup').first().json.domainCustomFieldId2]}}`  
   - Connect Get organization details -> Enrich company  
   - Add credentials for Clearbit API  

7. **Create a Set node "Add Organization ID to data":**  
   - Add an assignment: `organization_id` (number) = `{{$('Get organization details').item.json.id}}`  
   - Include other fields from input  
   - Connect Enrich company -> Add Organization ID to data  

8. **Create a Merge node "Merge data":**  
   - Mode: Enrich Input 2  
   - Join mode: Merge by fields  
   - Fields: `organization_id` on both inputs  
   - Clash handling: Prefer Input 2  
   - Connect Get all leads (as Input 1) and Add Organization ID to data (as Input 2) to Merge data  

9. **Create HTTP Request node "Mark lead as enriched in Pipedrive":**  
   - Method: PATCH  
   - URL: `https://api.pipedrive.com/v1/leads/{{ $json.id }}`  
   - Body parameters: set field with key from `enrichedAtCustomFieldId2` to current date formatted as `yyyy-MM-dd`  
   - Authentication: Pipedrive API OAuth2 or API key  
   - Connect Merge data -> Mark lead as enriched  

10. **Create Filter node "Keep leads that match the criteria":**  
    - Conditions:  
      - `$json.tags.includes("B2B")` is true  
      - `$json.metrics.employees` > 100  
    - Connect Mark lead as enriched -> Keep leads that match the criteria  

11. **Create Slack node "Send alert to Slack":**  
    - Channel: Use expression `{{$('Setup').item.json.slackChannel}}`  
    - Message text:  
      ```
      New high-quality lead 🤑
      *Company Name*: {{ $json.name }}
      *Website*: {{ "https://" + $json.domain }}
      *Revenue*: {{ $json.metrics.estimatedAnnualRevenue }}
      *Number of employees*: {{ $json.metrics.employees }}
      ```  
    - Credentials: Slack OAuth2 or token-based  
    - Connect Keep leads that match the criteria -> Send alert to Slack  

12. **Create HTTP Request node "Get all organization keys":**  
    - Method: GET  
    - URL: `https://api.pipedrive.com/v1/organizationFields`  
    - Authentication: Pipedrive API  
    - Connect none (manual run)  

13. **Create Split Out node "Split out organization field":**  
    - Field to split out: `data`  
    - Connect Get all organization keys -> Split out organization field  

14. **Create Filter node "Show only custom organization fields":**  
    - Condition: `edit_flag` is true  
    - Connect Split out organization field -> Show only custom organization fields  

15. **Create HTTP Request node "Get all lead keys":**  
    - Method: GET  
    - URL: `https://api.pipedrive.com/v1/leadFields`  
    - Authentication: Pipedrive API  
    - Connect none (manual run)  

16. **Create Split Out node "Split out lead field data":**  
    - Field to split out: `data`  
    - Connect Get all lead keys -> Split out lead field data  

17. **Create Filter node "Show only custom lead fields":**  
    - Condition: `edit_flag` is true  
    - Connect Split out lead field data -> Show only custom lead fields  

18. **Add Sticky Notes as needed with setup instructions and criteria adjustment hints.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                    | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow requires Pipedrive, Clearbit, and Slack credentials set up in n8n to function correctly.                                                          | Configuration prerequisite                                                                        |
| Custom fields `Domain` (organization) and `Enriched at` (lead) must be created in Pipedrive before running this workflow.                                      | Pipedrive setup step                                                                             |
| To discover the custom field IDs required for domain and enrichment date, run the nodes `Get all organization keys` and `Get all lead keys` with subsequent filtering. | Pipedrive API metadata exploration                                                               |
| Adjust lead quality criteria in the "Keep leads that match the criteria" filter node to match your business needs (e.g., revenue, tags, employee count).          | Customization hint                                                                               |
| Slack alert includes company name, website URL, estimated revenue, and number of employees to provide rich context for sales teams.                             | Slack message formatting                                                                         |
| Workflow built for n8n version 1.29.1, verify compatibility if running on other versions.                                                                       | Version note                                                                                    |

---

This documentation fully describes the workflow, enabling developers or AI agents to understand, reproduce, and modify the automation with confidence and precision.