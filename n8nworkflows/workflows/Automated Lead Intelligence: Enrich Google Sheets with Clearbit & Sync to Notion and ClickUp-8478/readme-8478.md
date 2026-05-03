Automated Lead Intelligence: Enrich Google Sheets with Clearbit & Sync to Notion and ClickUp

https://n8nworkflows.xyz/workflows/automated-lead-intelligence--enrich-google-sheets-with-clearbit---sync-to-notion-and-clickup-8478


# Automated Lead Intelligence: Enrich Google Sheets with Clearbit & Sync to Notion and ClickUp

### 1. Workflow Overview

This workflow automates lead enrichment by extracting email domains from raw lead data, enriching company information via Clearbit API, and updating multiple systems for sales and marketing operations. It targets sales and CRM teams looking to enhance lead profiles stored in Google Sheets with detailed company data, then synchronizes enriched data to Notion databases and ClickUp task management.

The workflow is composed of the following logical blocks:

- **1.1 Input Reception:** Manual trigger initiates the workflow, fetching raw lead data rows from a Google Sheets document.
- **1.2 Lead Filtering:** Filters leads to process only those that are "unbooked" based on criteria evaluated in an IF node.
- **1.3 Domain Extraction:** Extracts the domain part from the lead email addresses for enrichment.
- **1.4 Company Enrichment:** Calls Clearbit API to enrich company data based on extracted domain.
- **1.5 Data Aggregation:** Merges Clearbit data with the original lead data and fetches the company logo via HTTP request.
- **1.6 Data Synchronization:** Creates or updates corresponding pages in Notion, creates a task in ClickUp, and appends the enriched lead data back to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Starts workflow execution manually and retrieves lead data from Google Sheets.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get row(s) in sheet1 (Google Sheets)

- **Node Details:**

  - *When clicking ‘Execute workflow’*  
    - Type: Manual Trigger  
    - Role: Initiates workflow on user demand.  
    - Configuration: Default manual trigger settings.  
    - Inputs: None  
    - Outputs: Triggers "Get row(s) in sheet1" node.  
    - Failure: None expected.  
    - Version: 1

  - *Get row(s) in sheet1*  
    - Type: Google Sheets  
    - Role: Retrieves rows from a configured Google Sheet containing raw leads.  
    - Configuration: Defined spreadsheet and sheet name, likely configured to fetch all or filtered rows.  
    - Inputs: Triggered by manual trigger.  
    - Outputs: To "Check If Unbooked" node.  
    - Failure: Could fail if credentials expire, sheet unavailable, or API limit exceeded.  
    - Version: 4

---

#### 2.2 Lead Filtering

- **Overview:**  
  Filters leads to process only those that have not been booked yet, preventing duplicate processing.

- **Nodes Involved:**  
  - Check If Unbooked (IF)

- **Node Details:**

  - *Check If Unbooked*  
    - Type: IF node  
    - Role: Applies conditional logic to evaluate if lead is unbooked (likely using a column or flag in the sheet data).  
    - Configuration: Condition based on a field or property indicating booking status.  
    - Inputs: Rows from "Get row(s) in sheet1".  
    - Outputs:  
      - True: To "Extract Domain from Email"  
      - False: No further processing (filtered out).  
    - Failure: Expression errors if data missing or format unexpected.  
    - Version: 2

---

#### 2.3 Domain Extraction

- **Overview:**  
  Extracts domain names from email addresses to prepare for company enrichment.

- **Nodes Involved:**  
  - Extract Domain from Email (Function)

- **Node Details:**

  - *Extract Domain from Email*  
    - Type: Function node  
    - Role: Parses email addresses from input data to extract domain part (e.g., example.com from user@example.com).  
    - Configuration: Custom JavaScript code performing string manipulation.  
    - Inputs: Leads filtered as unbooked from IF node.  
    - Outputs: Two parallel outputs to "Get Company Logo" and "Clearbit Company Enrichment".  
    - Failure: Could fail if email format is invalid or missing.  
    - Version: 1

---

#### 2.4 Company Enrichment

- **Overview:**  
  Calls Clearbit API to retrieve enriched company data based on extracted domain.

- **Nodes Involved:**  
  - Clearbit Company Enrichment (HTTP Request)  
  - Get Company Logo (HTTP Request)

- **Node Details:**

  - *Clearbit Company Enrichment*  
    - Type: HTTP Request  
    - Role: Sends API request to Clearbit’s Company Enrichment endpoint with extracted domain as parameter.  
    - Configuration: HTTP method GET/POST with authentication header (API key).  
    - Inputs: From "Extract Domain from Email".  
    - Outputs: To "Merge" node (2nd input).  
    - Failure: API key auth errors, rate limiting, domain not found.  
    - Version: 4.1

  - *Get Company Logo*  
    - Type: HTTP Request  
    - Role: Retrieves company logo URL or image based on domain or Clearbit data.  
    - Configuration: Likely GET request to a logo service or Clearbit logo API.  
    - Inputs: From "Extract Domain from Email" (1st input).  
    - Outputs: To "Merge" node (1st input).  
    - Failure: HTTP errors, missing logo, timeouts.  
    - Version: 4.1

---

#### 2.5 Data Aggregation

- **Overview:**  
  Merges company enrichment data, logo, and original lead data into a unified data structure for further processing.

- **Nodes Involved:**  
  - Merge (Merge node)  
  - Merge Enrichment Data (Function)

- **Node Details:**

  - *Merge*  
    - Type: Merge node  
    - Role: Combines data from "Get Company Logo" and "Clearbit Company Enrichment" nodes.  
    - Configuration: Likely configured as "Merge by Index" or "Merge by Key" to combine related records.  
    - Inputs: Two inputs — logo data and company enrichment data.  
    - Outputs: To "Merge Enrichment Data".  
    - Failure: Mismatched array sizes or keys causing merge failures.  
    - Version: 3.2

  - *Merge Enrichment Data*  
    - Type: Function node  
    - Role: Combines merged company data with original lead data, possibly formatting or cleaning data for downstream systems.  
    - Configuration: Custom JavaScript code to structure final data.  
    - Inputs: From "Merge" node.  
    - Outputs: To "Create a database page" and "Create a task".  
    - Failure: Expression or data structure errors.  
    - Version: 1

---

#### 2.6 Data Synchronization

- **Overview:**  
  Updates external systems with enriched lead data: creates/updates Notion database pages, ClickUp tasks, and appends data back to Google Sheets.

- **Nodes Involved:**  
  - Create a database page (Notion)  
  - Create a task (ClickUp)  
  - Append to Enriched Leads Sheet (Google Sheets)

- **Node Details:**

  - *Create a database page*  
    - Type: Notion node  
    - Role: Creates or updates a page in a Notion database with enriched lead/company info.  
    - Configuration: Requires Notion credentials with database ID and page property mappings.  
    - Inputs: From "Merge Enrichment Data".  
    - Outputs: To "Append to Enriched Leads Sheet".  
    - Failure: API authentication, permission errors, schema mismatch.  
    - Version: 2.2

  - *Create a task*  
    - Type: ClickUp node  
    - Role: Creates a task in ClickUp for sales follow-up or lead management.  
    - Configuration: ClickUp API token, task list ID, task fields mapped from enriched data.  
    - Inputs: From "Merge Enrichment Data".  
    - Outputs: None (end node).  
    - Failure: API token invalid, rate limits, incorrect list ID.  
    - Version: 1

  - *Append to Enriched Leads Sheet*  
    - Type: Google Sheets  
    - Role: Appends enriched lead data as new rows in a dedicated Google Sheet to track processed leads.  
    - Configuration: Spreadsheet ID, sheet name, columns mapping.  
    - Inputs: From "Create a database page".  
    - Outputs: None (workflow end).  
    - Failure: Credential expiry, API errors, quota limits.  
    - Version: 4

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                      | Input Node(s)                 | Output Node(s)                     | Sticky Note |
|----------------------------|--------------------|------------------------------------|------------------------------|----------------------------------|-------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Starts workflow                    | None                         | Get row(s) in sheet1              |             |
| Get row(s) in sheet1       | Google Sheets      | Retrieves raw lead data             | When clicking ‘Execute workflow’ | Check If Unbooked                 |             |
| Check If Unbooked           | IF                 | Filters unbooked leads              | Get row(s) in sheet1          | Extract Domain from Email         |             |
| Extract Domain from Email   | Function           | Extracts domain from email          | Check If Unbooked             | Get Company Logo, Clearbit Company Enrichment |             |
| Get Company Logo            | HTTP Request       | Retrieves company logo              | Extract Domain from Email     | Merge                            |             |
| Clearbit Company Enrichment | HTTP Request       | Fetches company data from Clearbit | Extract Domain from Email     | Merge                            |             |
| Merge                      | Merge              | Combines logo and company data     | Get Company Logo, Clearbit Company Enrichment | Merge Enrichment Data            |             |
| Merge Enrichment Data       | Function           | Merges all data into final format  | Merge                        | Create a database page, Create a task |             |
| Create a database page      | Notion             | Creates/updates Notion page         | Merge Enrichment Data         | Append to Enriched Leads Sheet   |             |
| Create a task              | ClickUp            | Creates a ClickUp task              | Merge Enrichment Data         | None                            |             |
| Append to Enriched Leads Sheet | Google Sheets      | Appends enriched lead data          | Create a database page        | None                            |             |
| Sticky Note1               | Sticky Note        |                                    |                              |                                  |             |
| Sticky Note2               | Sticky Note        |                                    |                              |                                  |             |
| Sticky Note3               | Sticky Note        |                                    |                              |                                  |             |
| Sticky Note4               | Sticky Note        |                                    |                              |                                  |             |
| Sticky Note5               | Sticky Note        |                                    |                              |                                  |             |
| Sticky Note6               | Sticky Note        |                                    |                              |                                  |             |
| Sticky Note7               | Sticky Note        |                                    |                              |                                  |             |
| Sticky Note8               | Sticky Note        |                                    |                              |                                  |             |
| Sticky Note9               | Sticky Note        |                                    |                              |                                  |             |
| Sticky Note10              | Sticky Note        |                                    |                              |                                  |             |
| Sticky Note12              | Sticky Note        |                                    |                              |                                  |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add "Manual Trigger" node named "When clicking ‘Execute workflow’".  
   - No special configuration needed.

2. **Add Google Sheets Node to Get Rows**  
   - Add "Google Sheets" node named "Get row(s) in sheet1".  
   - Configure credentials with Google Sheets OAuth2.  
   - Set spreadsheet ID and sheet name containing raw leads.  
   - Configure to read rows (e.g., "Read Rows" mode).  
   - Connect "When clicking ‘Execute workflow’" output to this node's input.

3. **Add IF Node to Filter Unbooked Leads**  
   - Add "IF" node named "Check If Unbooked".  
   - Configure condition to check "booked" status column or flag in rows (e.g., `{{ $json["status"] !== "booked" }}`).  
   - Connect "Get row(s) in sheet1" output to IF node input.

4. **Add Function Node to Extract Domain**  
   - Add "Function" node named "Extract Domain from Email".  
   - Insert JavaScript code to parse domain from email, e.g.:  
     ```javascript
     return items.map(item => {
       const email = item.json.email || '';
       const domain = email.split('@')[1] || '';
       return { json: { ...item.json, domain } };
     });
     ```  
   - Connect IF node "true" output to this node.

5. **Add HTTP Request Node to Get Company Logo**  
   - Add "HTTP Request" node named "Get Company Logo".  
   - Configure GET request to a logo service or Clearbit logo endpoint using extracted domain.  
   - Include necessary authentication or headers if required.  
   - Connect "Extract Domain from Email" output to this node.

6. **Add HTTP Request Node for Clearbit Enrichment**  
   - Add "HTTP Request" node named "Clearbit Company Enrichment".  
   - Configure HTTP request (likely GET) to Clearbit Company Enrichment API.  
   - Use extracted domain as query parameter.  
   - Add API key in headers for authentication.  
   - Connect "Extract Domain from Email" output to this node.

7. **Add Merge Node to Combine Logo and Enrichment Data**  
   - Add "Merge" node named "Merge".  
   - Set mode to "Merge by Index" or appropriate matching strategy.  
   - Connect "Get Company Logo" to input 1 and "Clearbit Company Enrichment" to input 2.

8. **Add Function Node to Merge All Data**  
   - Add "Function" node named "Merge Enrichment Data".  
   - Write code to combine merged data with original lead data, mapping fields as needed.  
   - Connect "Merge" output to this node.

9. **Add Notion Node to Create Database Page**  
   - Add "Notion" node named "Create a database page".  
   - Configure Notion credentials with OAuth2.  
   - Set target database ID and map enriched lead fields to page properties.  
   - Connect "Merge Enrichment Data" output to this node.

10. **Add ClickUp Node to Create Task**  
    - Add "ClickUp" node named "Create a task".  
    - Configure ClickUp API token and target list ID.  
    - Map fields to task properties (title, description, due date, etc.).  
    - Connect "Merge Enrichment Data" output to this node.

11. **Add Google Sheets Node to Append Enriched Leads**  
    - Add "Google Sheets" node named "Append to Enriched Leads Sheet".  
    - Configure spreadsheet ID and sheet name for enriched leads.  
    - Set operation to "Append Row".  
    - Connect "Create a database page" output to this node.

12. **Verify and Test Workflow**  
    - Ensure all credentials are valid and have appropriate scopes.  
    - Test with sample data for smooth execution.  
    - Handle error cases such as missing emails, invalid domains, API failures.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Clearbit Company Enrichment API documentation is essential for correct API request setup and error handling.     | https://clearbit.com/docs#company-api                                                               |
| Google Sheets credentials require OAuth2 setup with appropriate scopes for reading and appending data.           | https://developers.google.com/sheets/api/guides/authorizing                                            |
| Notion API requires database schema mapping awareness to correctly create pages.                                | https://developers.notion.com/reference/page                                                         |
| ClickUp API token must be generated from ClickUp app with task creation permissions.                             | https://clickup.com/api                                                                             |
| Email domain extraction should handle malformed or missing emails gracefully to prevent Function node errors.   |                                                                                                   |
| Rate limits on Clearbit and APIs should be monitored; implement retry logic or throttling if needed.            |                                                                                                   |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.