Real Estate Lead Generation with BatchData Skip Tracing & CRM Integration

https://n8nworkflows.xyz/workflows/real-estate-lead-generation-with-batchdata-skip-tracing---crm-integration-3666


# Real Estate Lead Generation with BatchData Skip Tracing & CRM Integration

### 1. Workflow Overview

This workflow automates real estate lead generation by integrating BatchData’s property search and skip tracing APIs with CRM systems and reporting tools. It is designed to identify motivated sellers and high-potential properties based on configurable criteria, enrich lead data with owner contact details, and output the results for marketing or acquisition efforts.

The workflow is logically divided into the following blocks:

- **1.1 Triggers:** Supports manual execution and daily scheduled runs to initiate the process.
- **1.2 Property Search Configuration:** Sets up search parameters for querying BatchData’s Property Search API.
- **1.3 Property Search & Filtering:** Calls the API, filters results based on ownership and property criteria, and scores leads.
- **1.4 Skip Tracing Owner Contact Info:** Retrieves owner contact details for qualified properties.
- **1.5 Data Formatting:** Structures property and owner data into a clean, unified format.
- **1.6 Output Generation:** Produces an Excel spreadsheet, pushes leads to a CRM (HubSpot by default), summarizes results, and sends an email notification with the report attached.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggers

**Overview:**  
This block initiates the workflow either manually or on a daily schedule.

**Nodes Involved:**  
- When clicking "Execute Workflow" (Manual Trigger)  
- Daily Schedule (Scheduled Trigger)

**Node Details:**

- **When clicking "Execute Workflow"**  
  - Type: Manual Trigger  
  - Role: Allows on-demand execution by user interaction.  
  - Configuration: No parameters; triggers workflow immediately when executed.  
  - Inputs: None  
  - Outputs: Connects to "Configure Search Parameters" node.  
  - Edge Cases: None typical; manual trigger depends on user action.

- **Daily Schedule**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers workflow daily.  
  - Configuration: Interval set to daily (default time unspecified, configurable).  
  - Inputs: None  
  - Outputs: Connects to "Configure Search Parameters" node.  
  - Edge Cases: Timezone considerations; ensure schedule matches desired execution time.

---

#### 1.2 Property Search Configuration

**Overview:**  
Sets up the JSON search parameters for the property search API call, allowing customization of location, property type, value range, equity, and other filters.

**Nodes Involved:**  
- Configure Search Parameters

**Node Details:**

- **Configure Search Parameters**  
  - Type: Set  
  - Role: Defines the search criteria as a JSON string in a variable named `search_parameters`.  
  - Configuration:  
    - Location: Austin, TX  
    - Property Type: Single-family  
    - Value Range: $200,000 to $500,000  
    - Status: Distressed  
    - Minimum Equity: 30%  
    - Limit: 50 results  
  - Key Expression: The JSON string is set as an expression to be passed to the API.  
  - Inputs: From trigger nodes  
  - Outputs: Connects to "Search Properties API" node  
  - Edge Cases: Incorrect JSON formatting or invalid parameters could cause API errors.

---

#### 1.3 Property Search & Filtering

**Overview:**  
Calls BatchData’s Property Search API with configured parameters, filters the results based on absentee ownership, years owned, sales history, and scores leads to prioritize follow-up.

**Nodes Involved:**  
- Search Properties API  
- Filter Property Results

**Node Details:**

- **Search Properties API**  
  - Type: HTTP Request  
  - Role: Sends POST request to BatchData’s property search endpoint.  
  - Configuration:  
    - URL: `https://api.batchdata.com/api/v1/properties/search`  
    - Method: POST  
    - Authentication: HTTP Header with generic credentials (API key or token)  
    - Body: Uses `search_parameters` from previous node.  
  - Inputs: From "Configure Search Parameters"  
  - Outputs: Connects to "Filter Property Results"  
  - Edge Cases: API authentication failure, network timeout, invalid response format.

- **Filter Property Results**  
  - Type: Code (JavaScript)  
  - Role: Filters properties where owner is absentee, owned for 5+ years, no sales in last 3 years; calculates a lead score based on equity, ownership duration, and tax delinquency.  
  - Configuration: Custom JS code with filtering and scoring logic.  
  - Key Expressions: Uses property fields like `owner_occupied`, `last_sale_date`, `equity_percentage`, `tax_delinquent`.  
  - Inputs: From "Search Properties API"  
  - Outputs: Connects to "Get Owner Contact Info"  
  - Edge Cases: Missing or malformed property data, date parsing errors, empty results.

---

#### 1.4 Skip Tracing Owner Contact Info

**Overview:**  
For each filtered and scored property, calls BatchData’s skip tracing API to retrieve owner contact information such as phone numbers, emails, and mailing addresses.

**Nodes Involved:**  
- Get Owner Contact Info

**Node Details:**

- **Get Owner Contact Info**  
  - Type: HTTP Request  
  - Role: Sends POST request to BatchData’s skip trace endpoint for owner contact details.  
  - Configuration:  
    - URL: `https://api.batchdata.com/api/v1/property/skip-trace`  
    - Method: POST  
    - Authentication: HTTP Header with generic credentials  
    - Body: Property identifiers from filtered results  
  - Inputs: From "Filter Property Results"  
  - Outputs: Connects to "Format Lead Data"  
  - Edge Cases: API limits, missing contact info, authentication errors, network issues.

---

#### 1.5 Data Formatting

**Overview:**  
Combines property data and skip tracing results into a structured format suitable for CRM ingestion and reporting.

**Nodes Involved:**  
- Format Lead Data

**Node Details:**

- **Format Lead Data**  
  - Type: Code (JavaScript)  
  - Role: Maps and merges property and owner contact data into a unified JSON object with relevant fields for export and CRM.  
  - Configuration: Custom JS code extracting fields such as property address, owner name, contact info, lead score, and flags like absentee owner and tax delinquency.  
  - Inputs: From "Get Owner Contact Info"  
  - Outputs: Connects to "Create Excel Spreadsheet", "Push to CRM", and "Summarize Results" nodes  
  - Edge Cases: Missing fields, inconsistent data formats, null values.

---

#### 1.6 Output Generation

**Overview:**  
Generates an Excel report, pushes leads to a CRM system, summarizes the results, and sends an email notification with the report attached.

**Nodes Involved:**  
- Create Excel Spreadsheet  
- Push to CRM  
- Summarize Results  
- Email Notification

**Node Details:**

- **Create Excel Spreadsheet**  
  - Type: Spreadsheet File  
  - Role: Converts formatted lead data into an Excel (.xlsx) file with headers.  
  - Configuration:  
    - Filename: `Property_Leads_YYYY-MM-DD.xlsx` (dynamic date)  
    - Header row enabled  
  - Inputs: From "Format Lead Data"  
  - Outputs: Connects to "Email Notification"  
  - Edge Cases: Large data sets may impact performance or file size limits.

- **Push to CRM**  
  - Type: HubSpot (CRM Integration)  
  - Role: Adds or updates lead records in HubSpot CRM.  
  - Configuration: Uses HubSpot credentials; additional fields can be customized.  
  - Inputs: From "Format Lead Data"  
  - Outputs: None (end node for CRM push)  
  - Edge Cases: Authentication errors, API rate limits, data mapping mismatches.  
  - Notes: Can be replaced with other CRM nodes (Salesforce, Zoho, etc.) as needed.

- **Summarize Results**  
  - Type: Code (JavaScript)  
  - Role: Calculates summary metrics such as total leads found and highest lead score.  
  - Configuration: Simple aggregation code returning JSON with summary fields.  
  - Inputs: From "Format Lead Data"  
  - Outputs: Connects to "Email Notification"  
  - Edge Cases: Empty input array.

- **Email Notification**  
  - Type: Email Send  
  - Role: Sends an email with the lead report attached and summary information.  
  - Configuration:  
    - Subject: Includes current date dynamically  
    - To: Configurable recipient email  
    - From: No-reply email address  
    - Attachment: Excel file from "Create Excel Spreadsheet"  
  - Inputs: From "Create Excel Spreadsheet" and "Summarize Results" (via parallel connections)  
  - Outputs: None (terminal node)  
  - Edge Cases: SMTP configuration errors, email delivery failures.

---

### 3. Summary Table

| Node Name                   | Node Type            | Functional Role                         | Input Node(s)                | Output Node(s)                          | Sticky Note                                                                                      |
|-----------------------------|----------------------|---------------------------------------|-----------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger       | Manual workflow start                  | None                        | Configure Search Parameters            | ## Workflow Triggers: Manual trigger option                                                    |
| Daily Schedule              | Schedule Trigger     | Scheduled daily workflow start         | None                        | Configure Search Parameters            | ## Workflow Triggers: Scheduled trigger option                                                 |
| Configure Search Parameters | Set                  | Define property search criteria        | When clicking "Execute Workflow", Daily Schedule | Search Properties API                  | ## Search Configuration: Customize search parameters                                           |
| Search Properties API       | HTTP Request         | Call BatchData property search API     | Configure Search Parameters | Filter Property Results                | ## Property Data Processing: Search properties API                                             |
| Filter Property Results     | Code (JavaScript)    | Filter and score properties             | Search Properties API       | Get Owner Contact Info                 | ## Property Data Processing: Filter and score properties                                       |
| Get Owner Contact Info      | HTTP Request         | Call BatchData skip trace API           | Filter Property Results     | Format Lead Data                      | ## Property Data Processing: Skip trace owner contact info                                     |
| Format Lead Data            | Code (JavaScript)    | Format combined property and owner data | Get Owner Contact Info      | Create Excel Spreadsheet, Push to CRM, Summarize Results | ## Property Data Processing: Format lead data                                                  |
| Create Excel Spreadsheet    | Spreadsheet File     | Generate Excel report file               | Format Lead Data            | Email Notification                    | ## Lead Output Options: Create Excel Spreadsheet                                               |
| Push to CRM                | HubSpot              | Push leads to CRM system                 | Format Lead Data            | None                                  | ## Lead Output Options: Push to CRM                                                           |
| Summarize Results           | Code (JavaScript)    | Summarize lead generation results       | Format Lead Data            | Email Notification                    | ## Lead Output Options: Summarize results                                                     |
| Email Notification          | Email Send           | Send summary email with report attached | Create Excel Spreadsheet, Summarize Results | None                                  | ## Lead Output Options: Email Notification                                                    |
| Sticky Note - Workflow Overview | Sticky Note          | Overview of workflow steps               | None                        | None                                  | # Property Lead Automation Workflow overview                                                  |
| Sticky Note - Triggers      | Sticky Note          | Explains manual and scheduled triggers  | None                        | None                                  | ## Workflow Triggers: Manual and scheduled options                                            |
| Sticky Note - Property Search | Sticky Note          | Explains search parameter configuration | None                        | None                                  | ## Search Configuration: Customize search criteria                                            |
| Sticky Note - Data Processing | Sticky Note          | Describes property data processing steps | None                        | None                                  | ## Property Data Processing: Search, filter, skip trace, format                               |
| Sticky Note - Output        | Sticky Note          | Describes output options and nodes      | None                        | None                                  | ## Lead Output Options: Excel, CRM push, summary, email notification                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named `When clicking "Execute Workflow"` with default settings.  
   - Add a **Schedule Trigger** node named `Daily Schedule` configured to run daily at your preferred time.

2. **Configure Search Parameters:**  
   - Add a **Set** node named `Configure Search Parameters`.  
   - Add a string field named `search_parameters` with the following JSON value (adjust as needed):  
     ```json
     {
       "location": { "city": "Austin", "state": "TX" },
       "propertyType": "single_family",
       "value": { "min": 200000, "max": 500000 },
       "status": "distressed",
       "equity": { "min": 30 },
       "limit": 50
     }
     ```  
   - Connect both trigger nodes to this node.

3. **Search Properties API Call:**  
   - Add an **HTTP Request** node named `Search Properties API`.  
   - Set method to POST.  
   - URL: `https://api.batchdata.com/api/v1/properties/search`  
   - Authentication: Use HTTP Header Auth with your BatchData API key.  
   - Body: Use the `search_parameters` from the previous node as the JSON payload.  
   - Connect `Configure Search Parameters` to this node.

4. **Filter and Score Properties:**  
   - Add a **Code** node named `Filter Property Results`.  
   - Paste the provided JavaScript code that filters for absentee owners, ownership duration, no recent sales, and calculates a lead score.  
   - Connect `Search Properties API` to this node.

5. **Skip Trace Owner Contact Info:**  
   - Add an **HTTP Request** node named `Get Owner Contact Info`.  
   - Set method to POST.  
   - URL: `https://api.batchdata.com/api/v1/property/skip-trace`  
   - Authentication: Use HTTP Header Auth with your BatchData API key.  
   - Body: Pass property identifiers from filtered results.  
   - Connect `Filter Property Results` to this node.

6. **Format Lead Data:**  
   - Add a **Code** node named `Format Lead Data`.  
   - Paste the provided JavaScript code that merges property and skip trace data into a structured JSON object with all relevant fields.  
   - Connect `Get Owner Contact Info` to this node.

7. **Create Excel Spreadsheet:**  
   - Add a **Spreadsheet File** node named `Create Excel Spreadsheet`.  
   - Operation: `toFile`  
   - File Format: `xlsx`  
   - File Name: `Property_Leads_{{ $now.format('YYYY-MM-DD') }}.xlsx`  
   - Enable header row.  
   - Connect `Format Lead Data` to this node.

8. **Push Leads to CRM:**  
   - Add a **HubSpot** node named `Push to CRM`.  
   - Configure with your HubSpot OAuth2 credentials.  
   - Map lead fields as needed.  
   - Connect `Format Lead Data` to this node.

9. **Summarize Results:**  
   - Add a **Code** node named `Summarize Results`.  
   - Paste the JavaScript code that calculates total leads and highest lead score.  
   - Connect `Format Lead Data` to this node.

10. **Send Email Notification:**  
    - Add an **Email Send** node named `Email Notification`.  
    - Configure SMTP credentials or use your email provider.  
    - Set subject to `Property Lead Report - {{ $now.format('YYYY-MM-DD') }}`.  
    - Set recipient and sender emails accordingly.  
    - Attach the Excel file from `Create Excel Spreadsheet`.  
    - Connect both `Create Excel Spreadsheet` and `Summarize Results` nodes to this node.

11. **Connect Workflow:**  
    - Connect both trigger nodes to `Configure Search Parameters`.  
    - Connect nodes sequentially as described above.  
    - Ensure parallel outputs from `Format Lead Data` connect to `Create Excel Spreadsheet`, `Push to CRM`, and `Summarize Results`.  
    - Connect `Create Excel Spreadsheet` and `Summarize Results` to `Email Notification`.

12. **Credentials Setup:**  
    - Create credentials for BatchData API (HTTP Header Auth with API key).  
    - Create credentials for HubSpot OAuth2.  
    - Create SMTP credentials for email sending.

13. **Testing and Validation:**  
    - Test manual trigger first to verify API calls and data flow.  
    - Validate filtering and scoring logic with sample data.  
    - Confirm Excel file generation and email delivery.  
    - Test CRM integration for lead creation.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| BatchData provides nationwide property data and skip tracing services ideal for real estate lead generation. | https://batchdata.com/ (official BatchData website)                                            |
| This workflow supports multiple CRM integrations; HubSpot is used as an example but can be replaced easily.  | HubSpot API docs: https://developers.hubspot.com/docs/api/                                     |
| Customize search parameters in the Set node to target different locations, property types, or value ranges.  | See "Configure Search Parameters" node sticky note for guidance.                               |
| The workflow can run manually or be scheduled daily for continuous lead generation.                           | See "Sticky Note - Triggers" for trigger options.                                             |
| Email node requires proper SMTP configuration; consider using services like SendGrid, Gmail SMTP, or others. | n8n Email node docs: https://docs.n8n.io/nodes/n8n-nodes-base.emailSend/                       |
| Lead scoring logic is customizable in the "Filter Property Results" code node to fit different investment criteria. | Modify JavaScript code as needed for your business logic.                                      |

---

This documentation provides a complete, structured understanding of the "Real Estate Lead Generation with BatchData Skip Tracing & CRM Integration" workflow, enabling reproduction, modification, and troubleshooting by advanced users and automation agents alike.