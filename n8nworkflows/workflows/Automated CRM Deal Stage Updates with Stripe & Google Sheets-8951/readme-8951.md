Automated CRM Deal Stage Updates with Stripe & Google Sheets

https://n8nworkflows.xyz/workflows/automated-crm-deal-stage-updates-with-stripe---google-sheets-8951


# Automated CRM Deal Stage Updates with Stripe & Google Sheets

### 1. Workflow Overview

This workflow automates the synchronization of paid invoices from Stripe with a CRM system tracked via Google Sheets. It is designed to run on a schedule (e.g., daily), fetch all paid invoices from Stripe, find corresponding customers in a Google Sheets CRM tracking document, and update the deal status to "Closed" for those customers. The workflow ensures data cleanliness by removing duplicates and filtering out empty records before updating the CRM sheet.

**Primary Use Case:**  
Automate deal stage updates in CRM based on payment completion, reducing manual effort and ensuring timely status updates.

**Logical Blocks:**  
- **1.1 Scheduled Trigger:** Initiates the workflow on a set interval.  
- **1.2 Stripe Data Retrieval:** Calls Stripe API to fetch all paid invoices.  
- **1.3 Invoice Splitting:** Splits the array of invoices into individual records for processing.  
- **1.4 Customer Lookup:** Searches for each invoice’s customer email in the Google Sheets CRM tracking sheet to retrieve CRM-specific deal IDs.  
- **1.5 Data Cleaning & Deal Status Update:** Removes duplicate entries, filters out empty data, and marks deals as "Closed".  
- **1.6 CRM Sheet Update:** Updates or appends the cleaned data back into Google Sheets, syncing the "Closed" status in the CRM tracking sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block starts the workflow automatically at a defined interval (default is daily), enabling ongoing synchronization without manual intervention.

- **Nodes Involved:**  
  - ⏰ Daily Trigger

- **Node Details:**  
  - **Node Name:** ⏰ Daily Trigger  
  - **Type:** Schedule Trigger  
  - **Configuration:** Triggers on a repeating schedule (default interval is daily, can be customized).  
  - **Key Expressions:** None  
  - **Connections:** Output → "💳 Get Paid Invoices from Stripe"  
  - **Potential Failures:** Workflow won’t start if scheduling is misconfigured or the n8n instance is offline.  
  - **Notes:** Adjust the interval to control sync frequency.

#### 2.2 Stripe Data Retrieval

- **Overview:**  
  Calls the Stripe API to fetch all invoices that have the status "paid". The response includes customer-related data necessary for CRM updates.

- **Nodes Involved:**  
  - 💳 Get Paid Invoices from Stripe

- **Node Details:**  
  - **Node Name:** 💳 Get Paid Invoices from Stripe  
  - **Type:** HTTP Request (Stripe API)  
  - **Configuration:**  
    - Method: GET  
    - URL: https://api.stripe.com/v1/invoices  
    - Query Parameter: status=paid  
    - Authentication: Stripe API credentials (predefined credential type)  
  - **Key Expressions:** None; static query parameter for filtering paid invoices.  
  - **Connections:** Output → "📋 Split Invoice List"  
  - **Edge Cases:**  
    - API rate limits or authentication failures.  
    - Empty result set if no paid invoices.  
    - Network timeouts.  
  - **Notes:** Ensure Stripe API key has permissions for invoice read access.

#### 2.3 Invoice Splitting

- **Overview:**  
  Stripe returns invoices as an array inside a field named `data`. This node splits that array into individual workflow items so each invoice can be processed separately.

- **Nodes Involved:**  
  - 📋 Split Invoice List

- **Node Details:**  
  - **Node Name:** 📋 Split Invoice List  
  - **Type:** Split Out (Array Splitter)  
  - **Configuration:**  
    - Field to split out: `data` (the array of invoices)  
    - Includes all other fields for each item.  
  - **Connections:** Output → "🔍 Find Customer in CRM Sheet"  
  - **Edge Cases:** Empty array results in no further processing.  
  - **Notes:** Critical for item-by-item processing downstream.

#### 2.4 Customer Lookup

- **Overview:**  
  For each paid invoice, this block looks up the customer email in a Google Sheets document that tracks CRM deals, retrieving associated CRM deal IDs (HubSpot, Pipedrive).

- **Nodes Involved:**  
  - 🔍 Find Customer in CRM Sheet

- **Node Details:**  
  - **Node Name:** 🔍 Find Customer in CRM Sheet  
  - **Type:** Google Sheets - Lookup  
  - **Configuration:**  
    - Sheet ID: `1i0_xNab1dLKkIJ72DLfa1PXvR9VH4d8IssbsphhAAQk`  
    - Range: Columns `A:C`  
    - Lookup Value: Customer email from Stripe invoice JSON (`{{$json.data.customer_email}}`)  
    - Lookup Column: "Stripe Email" (Column A)  
    - Authentication: OAuth2 (Google Sheets account)  
  - **Connections:** Output → "🧹 Clean Data & Mark as Closed"  
  - **Edge Cases:**  
    - No matching email found returns empty result — handle downstream.  
    - Google Sheets API quota limits or permission errors.  
    - Lookup failures due to sheet structure mismatch.  
  - **Notes:** Column headers must be exactly "Stripe Email", "HubSpot Deal ID", and "Pipedrive Deal ID".

#### 2.5 Data Cleaning & Deal Status Update

- **Overview:**  
  Cleans the data by removing duplicate entries based on email, filters out empty records, and adds a new field to mark the deal as "Closed".

- **Nodes Involved:**  
  - 🧹 Clean Data & Mark as Closed

- **Node Details:**  
  - **Node Name:** 🧹 Clean Data & Mark as Closed  
  - **Type:** Code (JavaScript)  
  - **Configuration:**  
    - Custom JS code filters items to keep unique emails only  
    - Adds the field `Deal` with value `"Closed"`  
  - **Key Expressions:**  
    ```js
    return items
      .filter((item, index, self) => {
        return (
          Object.keys(item.json).length > 0 &&
          index === self.findIndex(t => t.json["Stripe Email"] === item.json["Stripe Email"])
        );
      })
      .map(item => ({
        json: {
          ...item.json,
          Deal: "Closed"
        }
      }));
    ```  
  - **Connections:** Output → "✅ Update CRM Sheet with Closed Deals"  
  - **Edge Cases:**  
    - Items without "Stripe Email" may be filtered out or cause incorrect matching.  
    - If data structure changes upstream, code may break.  
  - **Notes:** Customize the `"Closed"` string to match CRM deal stage conventions.

#### 2.6 CRM Sheet Update

- **Overview:**  
  Takes the cleaned and processed deal data and updates or appends it into the CRM tracking Google Sheet, matching rows by "Stripe Email" and updating the deal status.

- **Nodes Involved:**  
  - ✅ Update CRM Sheet with Closed Deals

- **Node Details:**  
  - **Node Name:** ✅ Update CRM Sheet with Closed Deals  
  - **Type:** Google Sheets - Append or Update  
  - **Configuration:**  
    - Document ID: Same as Customer Lookup step  
    - Sheet Name: "CRM" (sheet at `gid=1552459854`)  
    - Matching Columns: "Stripe Email" (used to find existing rows)  
    - Columns to update: "Deal", "Stripe Email", "HubSpot Deal ID", "Pipedrive Deal ID"  
    - Mapping Mode: Define columns below with expressions mapping from JSON  
  - **Connections:** None (final node)  
  - **Edge Cases:**  
    - Google Sheets API quota or permission issues.  
    - Mismatched column headers cause update failures.  
    - Concurrent updates may cause conflicts or race conditions.  
  - **Notes:** Preserves existing CRM deal IDs while updating the deal status to "Closed".

---

### 3. Summary Table

| Node Name                        | Node Type             | Functional Role                     | Input Node(s)                     | Output Node(s)                      | Sticky Note                                                                                          |
|---------------------------------|-----------------------|-----------------------------------|----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------|
| ⏰ Daily Trigger                 | Schedule Trigger      | Starts workflow on schedule       |                                  | 💳 Get Paid Invoices from Stripe   | ## 🚀 Automation Start - Purpose and high-level workflow description                                |
| 💳 Get Paid Invoices from Stripe | HTTP Request (Stripe) | Fetches paid invoices from Stripe | ⏰ Daily Trigger                 | 📋 Split Invoice List              | ## 💳 Stripe API Call - Details about Stripe API call and setup                                    |
| 📋 Split Invoice List           | Split Out             | Splits array of invoices into items| 💳 Get Paid Invoices from Stripe | 🔍 Find Customer in CRM Sheet      | ## 📋 Split Array Data - Explains array splitting necessity                                       |
| 🔍 Find Customer in CRM Sheet    | Google Sheets Lookup  | Finds customer data in Google Sheets| 📋 Split Invoice List            | 🧹 Clean Data & Mark as Closed     | ## 🔍 Customer Lookup - CRM sheet structure and lookup explanation                                |
| 🧹 Clean Data & Mark as Closed   | Code                  | Removes duplicates and marks deals| 🔍 Find Customer in CRM Sheet    | ✅ Update CRM Sheet with Closed Deals | ## 🧹 Data Processing Logic - Code explanation and customization tips                              |
| ✅ Update CRM Sheet with Closed Deals | Google Sheets Append or Update | Updates CRM sheet with closed deals| 🧹 Clean Data & Mark as Closed   |                                   | ## ✅ Final Update Step - Details on update logic and column mapping                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**
   - Name: `⏰ Daily Trigger`
   - Type: Schedule Trigger
   - Set interval (e.g., daily at a preferred time)

3. **Add an HTTP Request node to fetch paid invoices from Stripe:**
   - Name: `💳 Get Paid Invoices from Stripe`
   - Type: HTTP Request
   - HTTP Method: GET
   - URL: `https://api.stripe.com/v1/invoices`
   - Query Parameters: `status=paid`
   - Authentication: Use Stripe API credentials (OAuth or API key via n8n credentials)
   - Connect `⏰ Daily Trigger` output to this node's input.

4. **Add a Split Out node to split the Stripe invoices array:**
   - Name: `📋 Split Invoice List`
   - Type: Split Out
   - Field to split out: `data`
   - Connect the output of `💳 Get Paid Invoices from Stripe` to this node.

5. **Add a Google Sheets node to look up customers:**
   - Name: `🔍 Find Customer in CRM Sheet`
   - Type: Google Sheets (Lookup operation)
   - Operation: Lookup
   - Sheet ID: Use your Google Sheets CRM tracking document ID
   - Range: `A:C`
   - Lookup Column: "Stripe Email"
   - Lookup Value: Expression: `{{$json.data.customer_email}}`
   - Authentication: Google Sheets OAuth2 credentials
   - Connect output of `📋 Split Invoice List` to this node.

6. **Add a Code node to clean data and mark deals as closed:**
   - Name: `🧹 Clean Data & Mark as Closed`
   - Type: Code (JavaScript)
   - Paste the following code:
     ```js
     return items
       .filter((item, index, self) => {
         return (
           Object.keys(item.json).length > 0 &&
           index === self.findIndex(t => t.json["Stripe Email"] === item.json["Stripe Email"])
         );
       })
       .map(item => ({
         json: {
           ...item.json,
           Deal: "Closed"
         }
       }));
     ```
   - Connect output of `🔍 Find Customer in CRM Sheet` to this node.

7. **Add a Google Sheets node to update or append closed deals:**
   - Name: `✅ Update CRM Sheet with Closed Deals`
   - Type: Google Sheets (Append or Update operation)
   - Document ID: Same as customer lookup node
   - Sheet Name: The sheet/tab name or gid where CRM data is stored
   - Matching Columns: `Stripe Email`
   - Columns to map/update:  
     - `Stripe Email` → `{{$json["Stripe Email"]}}`  
     - `HubSpot Deal ID` → `{{$json["HubSpot Deal ID"]}}`  
     - `Pipedrive Deal ID` → `{{$json["Pipedrive Deal ID"]}}`  
     - `Deal` → `{{$json.Deal}}` (set to "Closed")  
   - Authentication: Google Sheets OAuth2 credentials
   - Connect output of `🧹 Clean Data & Mark as Closed` to this node.

8. **Activate the workflow and test with sample data.**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Adjust the schedule trigger interval based on your business needs (hourly, daily, weekly).                     | Scheduling setup                                                                                           |
| Ensure that the Google Sheets document has exact columns: "Stripe Email", "HubSpot Deal ID", "Pipedrive Deal ID".| Sheet structure requirements                                                                               |
| Customize the deal stage string `"Closed"` in the code node to match your CRM's deal stage terminology.         | Code customization tip                                                                                     |
| Verify Stripe API credentials have read access to invoices before running workflow.                            | Stripe API permissions                                                                                      |
| Monitor execution logs regularly for error diagnosis (e.g., API failures, permission issues).                  | Workflow maintenance                                                                                       |
| Google Sheets OAuth2 authentication requires consent to the spreadsheet; ensure the credential has proper scopes.| Google Sheets credential setup                                                                             |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.