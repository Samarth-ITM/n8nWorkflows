Create Shopify customers from a Google Sheet

https://n8nworkflows.xyz/workflows/create-shopify-customers-from-a-google-sheet-4866


# Create Shopify customers from a Google Sheet

### 1. Workflow Overview

This workflow automates the creation of Shopify customers by reading customer data from a Google Sheet and using Shopify’s Admin GraphQL API to create customers in the Shopify store. It is designed for businesses that maintain customer data in Google Sheets and want to automate syncing or onboarding customers into Shopify without manual entry.

The logical blocks are:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Data Retrieval:** Fetch customers’ data from a specified Google Sheet.
- **1.3 Customer Creation:** Create customers in Shopify via the GraphQL API using the fetched data.
- **1.4 Documentation/Instructions:** Sticky notes provide configuration guidance and data format expectations.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block provides a manual trigger to start the workflow execution.
- **Nodes Involved:**  
  - Start Workflow
- **Node Details:**  
  - **Node:** Start Workflow  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually via user interaction.  
    - Configuration: No parameters; triggers start of execution.  
    - Inputs: None  
    - Outputs: Connects to Google Sheet node  
    - Version-specific: None  
    - Edge cases: None except user forgetting to trigger manually

#### 1.2 Data Retrieval

- **Overview:** Reads customer data from a Google Sheet. This data acts as input for customer creation in Shopify.
- **Nodes Involved:**  
  - Google Sheet, Fetch Customers
- **Node Details:**  
  - **Node:** Google Sheet, Fetch Customers  
    - Type: Google Sheets node  
    - Role: Reads rows from a specific Google Sheet and sheet tab named “Customers”  
    - Configuration:  
      - Document ID: points to a specific Google Sheets document (ID: 1IxiuRiu6XKBkEa1NIUebBWn73jIKUrgE9Sqj4XtQgBk)  
      - Sheet name: "Customers" (sheet tab with ID 2054184606)  
      - Credentials: Google Sheets OAuth2 account configured  
      - Mode: Reads all rows as JSON objects using header row for column names  
    - Key expressions: None beyond standard Google Sheets node setup  
    - Inputs: From Start Workflow  
    - Outputs: Customer data as JSON per row to Shopify node  
    - Version-specific: Uses Google Sheets node version 4.6  
    - Edge cases:  
      - Authentication errors if OAuth token expired or revoked  
      - Empty or improperly formatted sheet causing empty or invalid data  
      - Network or API timeout issues  
      - Missing columns or invalid phone/email formats (not validated here but may cause Shopify API errors)

#### 1.3 Customer Creation

- **Overview:** For each customer record fetched, this block sends a GraphQL mutation to Shopify Admin API to create the customer.
- **Nodes Involved:**  
  - Shopify, CustomerCreate
- **Node Details:**  
  - **Node:** Shopify, CustomerCreate  
    - Type: GraphQL node  
    - Role: Executes Shopify Admin API customer creation mutation  
    - Configuration:  
      - GraphQL Endpoint: https://store99563.myshopify.com/admin/api/2025-04/graphql.json  
      - Authentication: HTTP Header Auth with header key `X-Shopify-Access-Token` and Shopify access token value starting with `shpat_`  
      - Query: GraphQL mutation `customerCreate` with input variables mapped from Google Sheet JSON fields:  
        - email ← `{{ $json.email }}`  
        - phone ← `{{ $json.mobile_phone }}`  
        - firstName ← `{{ $json.first_name }}`  
        - lastName ← `{{ $json.last_name }}`  
        - smsMarketingConsent statically set to SUBSCRIBED and SINGLE_OPT_IN  
      - Variables: Dynamically constructed JSON based on current item  
    - Inputs: Data from Google Sheet node (each row)  
    - Outputs: Shopify API response including success or user errors  
    - Version-specific: Uses GraphQL node version 1.1  
    - Edge cases:  
      - Authentication errors if token invalid or expired  
      - Validation errors from Shopify if email or phone invalid or missing  
      - Rate limiting by Shopify API  
      - Unexpected API errors or network failures  
      - Partial failures for some customers if batch processing  
    - Sub-workflow: None

#### 1.4 Documentation / Instructions

- **Overview:** Two sticky notes provide essential instructions on data formatting and Shopify API authentication.
- **Nodes Involved:**  
  - Sticky Note (Google Sheet format)  
  - Sticky Note1 (Shopify Admin API Authentication)
- **Node Details:**  
  - **Sticky Note (Google Sheet Format)**  
    - Type: Sticky Note  
    - Content: Describes the required Google Sheet columns and formatting rules, including:  
      - Column names: first_name, last_name, email, mobile_phone  
      - Mobile phone format must be international and no spaces (e.g., +61414708406)  
      - Email must be valid  
    - Role: User guidance, no inputs or outputs  
  - **Sticky Note1 (Shopify Admin API)**  
    - Type: Sticky Note  
    - Content: Details how to generate and configure Shopify Admin API access token with proper scopes (read_customers, write_customers) and usage of header auth  
    - Role: User guidance, no inputs or outputs

---

### 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                      | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                                                                                                                                                                     |
|---------------------------|-------------------------------|------------------------------------|-----------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Start Workflow            | Manual Trigger                | Initiate workflow manually         | None                  | Google Sheet, Fetch Customers |                                                                                                                                                                                                                                                 |
| Google Sheet, Fetch Customers | Google Sheets                | Fetch customer rows from Google Sheet | Start Workflow        | Shopify, CustomerCreate  |                                                                                                                                                                                                                                                 |
| Shopify, CustomerCreate   | GraphQL                      | Create Shopify customers via API   | Google Sheet, Fetch Customers | None                    |                                                                                                                                                                                                                                                 |
| Sticky Note               | Sticky Note                  | Instructions on Google Sheet format | None                  | None                    | Columns can be in any order. N8N will treat the first row in the sheet as a column name, so use the column names below in row 1 of your sheet. The google sheet uses the following columns : first_name, last_name, email, mobile_phone. Mobile phone must be international format with no spaces. |
| Sticky Note1              | Sticky Note                  | Instructions on Shopify API auth   | None                  | None                    | Shopify's Admin API uses 'Header Auth' with a key of X-Shopify-Access-Token and a value of your shopify access token which starts with shpat_. Details on how to generate and use the token are provided.                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a Manual Trigger node named "Start Workflow".  
   - No parameters needed.

2. **Add Google Sheets Node**  
   - Add a Google Sheets node named "Google Sheet, Fetch Customers".  
   - Set Operation to "Read Rows".  
   - Set Document ID to your Google Sheet ID (e.g., `1IxiuRiu6XKBkEa1NIUebBWn73jIKUrgE9Sqj4XtQgBk`).  
   - Set Sheet Name to your customer data tab, e.g., "Customers".  
   - Connect credentials using your Google Sheets OAuth2 account.  
   - Ensure the first row in your sheet contains column headers: `first_name`, `last_name`, `email`, `mobile_phone`.  
   - Connect "Start Workflow" node output to this node’s input.

3. **Add GraphQL Node for Shopify Customer Creation**  
   - Add a GraphQL node named "Shopify, CustomerCreate".  
   - Set the Endpoint URL to your Shopify store’s Admin API endpoint, e.g., `https://yourstore.myshopify.com/admin/api/2025-04/graphql.json`.  
   - Set Authentication to "Header Auth".  
     - Header Name: `X-Shopify-Access-Token`  
     - Header Value: Your Shopify access token starting with `shpat_`.  
   - Enter the following GraphQL mutation query:

    ```graphql
    mutation customerCreate($input: CustomerInput!) {
      customerCreate(input: $input) {
        userErrors {
          field
          message
        }
        customer {
          id
          email
          phone
          taxExempt
          firstName
          lastName
          amountSpent {
            amount
            currencyCode
          }
          smsMarketingConsent {
            marketingState
            marketingOptInLevel
            consentUpdatedAt
          }
        }
      }
    }
    ```

   - Set Variables field to:

    ```json
    {
      "input": {
        "email": "{{ $json.email }}",
        "phone": "{{ $json.mobile_phone }}",
        "firstName": "{{ $json.first_name }}",
        "lastName": "{{ $json.last_name }}",
        "smsMarketingConsent": {
          "marketingState": "SUBSCRIBED",
          "marketingOptInLevel": "SINGLE_OPT_IN"
        }
      }
    }
    ```

   - Connect the output of the Google Sheets node to this GraphQL node.

4. **Add Sticky Notes for Documentation (Optional but Recommended)**  
   - Add a Sticky Note explaining the expected Google Sheet format: columns and mobile phone format.  
   - Add a Sticky Note explaining how to generate and configure Shopify Admin API access tokens, including required scopes and usage instructions.

5. **Finalize and Test**  
   - Save and activate the workflow.  
   - Trigger manually to test data flow and customer creation.  
   - Monitor for errors such as authentication, data validation, or API rate limits.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Shopify Admin API uses 'Header Auth' with key `X-Shopify-Access-Token` and value your access token starting with `shpat_`. To generate the token, create a private app in Shopify admin with at least `read_customers` and `write_customers` scopes.                                                                                                                                               | See sticky note in workflow for detailed step-by-step instructions                                              |
| Google Sheets: Ensure the first row has exact column headers: `first_name`, `last_name`, `email`, `mobile_phone`. Mobile phone numbers must be in international format without spaces (e.g., +61414708406). Shopify will reject improperly formatted phone numbers.                                                                                                                                 | See sticky note in workflow for data formatting guidance                                                       |
| Shopify GraphQL API documentation: https://shopify.dev/api/admin-graphql                                                                                                                                                                                                                                                                                                                      | Official Shopify API Documentation                                                                              |
| Google Sheets API documentation: https://developers.google.com/sheets/api                                                                                                                                                                                                                                                                                                                     | Official Google Sheets API Documentation                                                                         |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.