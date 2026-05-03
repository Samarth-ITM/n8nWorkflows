Bulk Lead Email Validation with Google Sheets & Anymail Finder

https://n8nworkflows.xyz/workflows/bulk-lead-email-validation-with-google-sheets---anymail-finder-7693


# Bulk Lead Email Validation with Google Sheets & Anymail Finder

### 1. Workflow Overview

This workflow automates the bulk validation of lead email addresses stored in a Google Sheets document by leveraging the Anymail Finder API. It is designed for sales, marketing, or data enrichment teams that need to verify large lists of email addresses efficiently and update their status directly in the spreadsheet.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Data Retrieval**: Triggering the workflow manually and fetching leads from Google Sheets filtered by a "VERIFY" column.
- **1.2 Batch Processing Loop**: Iterating over each lead in batches to manage API calls efficiently.
- **1.3 Email Validation via Anymail Finder API**: Sending each email address to the Anymail Finder API to retrieve validation status.
- **1.4 Writing Back Validation Status**: Updating the original Google Sheets row with the received email validation status.
- **1.5 Workflow Control and Error Handling**: Managing process flow and ensuring resilience against API or connection errors.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Retrieval

- **Overview:**  
  This block initiates the workflow manually and extracts lead email data from a specific Google Sheets document. It filters rows based on the "VERIFY" column to select emails needing verification.

- **Nodes Involved:**  
  - Manual Trigger  
  - Get Leads (Google Sheets)

- **Node Details:**  
  - **Manual Trigger**  
    - Type: Trigger node to start workflow manually.  
    - Configuration: No parameters; simply enables manual execution.  
    - Inputs: None (start node).  
    - Outputs: Triggers "Get Leads".  
    - Edge Cases: None typical; manual initiation ensures controlled start.

  - **Get Leads**  
    - Type: Google Sheets node for reading data.  
    - Configuration:  
      - Document ID set to a specific Google Sheets file.  
      - Sheet name is "Foglio1" (gid=0).  
      - Filters applied on the "VERIFY" column to fetch leads pending verification.  
      - Uses OAuth2 credentials named "Google Sheets account".  
    - Inputs: Trigger from Manual Trigger.  
    - Outputs: Passes filtered lead rows to "Loop Over Items".  
    - Edge Cases: Google API quota limits, OAuth token expiration, empty or malformed data.

#### 2.2 Batch Processing Loop

- **Overview:**  
  This block manages processing of lead rows by splitting data into manageable batches, enabling sequential or parallel processing without overwhelming the API or system.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)

- **Node Details:**  
  - **Loop Over Items**  
    - Type: SplitInBatches node for batch processing.  
    - Configuration: Default settings; no batch size explicitly defined (implies default batch size, usually 1).  
    - Inputs: Data from "Get Leads".  
    - Outputs: Sends individual lead data to "Check email status". On completion, loops back after updating status.  
    - Edge Cases: Handling empty batches, large datasets causing timeouts.

#### 2.3 Email Validation via Anymail Finder API

- **Overview:**  
  Sends each email address to Anymail Finder’s API to check its validity. Manages authentication and error tolerance.

- **Nodes Involved:**  
  - Check email status (HTTP Request)

- **Node Details:**  
  - **Check email status**  
    - Type: HTTP Request node.  
    - Configuration:  
      - POST request to "https://api.anymailfinder.com/v5.1/verify-email".  
      - Request body includes the current email extracted from the batch item (`{{$json["EMAIL "]}}`).  
      - Authenticates using HTTP Header Auth with a credential named "Anymail Finder". The header name is "Authorization" and value is the API key.  
      - Timeout set to 180 seconds to handle slow API responses.  
      - On error, configured to continue workflow execution without stopping (error output continues).  
    - Inputs: Single lead email from "Loop Over Items".  
    - Outputs: JSON response containing email validation results sent to "Update email status".  
    - Edge Cases: API rate limits, invalid API key, network timeouts, malformed emails, unexpected API response formats.

#### 2.4 Writing Back Validation Status

- **Overview:**  
  Updates the "VERIFY" column in the Google Sheets document with the email validation status returned from Anymail Finder. Uses row numbers to ensure precise updates.

- **Nodes Involved:**  
  - Update email status (Google Sheets)

- **Node Details:**  
  - **Update email status**  
    - Type: Google Sheets node for updating rows.  
    - Configuration:  
      - Document and sheet selection identical to "Get Leads".  
      - Mapping mode: Manual column mapping with "VERIFY" set to `{{$json.email_status}}` from API response and "row_number" matched to the current batch item's row number.  
      - Matching based on "row_number" to ensure correct row update.  
      - Uses the same Google Sheets OAuth2 credentials.  
    - Inputs: Email validation response from "Check email status".  
    - Outputs: Loops back to "Loop Over Items" to process next batch.  
    - Edge Cases: Row number mismatches, update conflicts if sheet changes during execution, Google API quota or permission errors.

#### 2.5 Workflow Control and Error Handling

- **Overview:**  
  Overall flow control is maintained by connections between nodes; error handling is explicitly configured to avoid workflow termination on API errors.

- **Nodes Involved:**  
  - Implicit in node connections and error configurations.

- **Details:**  
  - The "Check email status" node has "continueErrorOutput" enabled, allowing the workflow to proceed even if individual API calls fail, thus preventing total workflow failure.  
  - The batch loop allows reprocessing and controlled pacing of requests.  
  - Manual Trigger ensures controlled execution start.  
  - The sticky note node provides documentation and usage instructions but does not affect execution.

---

### 3. Summary Table

| Node Name          | Node Type             | Functional Role                       | Input Node(s)     | Output Node(s)    | Sticky Note                                                                                                      |
|--------------------|-----------------------|------------------------------------|-------------------|-------------------|------------------------------------------------------------------------------------------------------------------|
| Manual Trigger     | Manual Trigger          | Starts workflow manually            | None              | Get Leads         | ## Verify leads email address with Anymail Finder This workflow automates the **process of validating email addresses** stored in a **Google Sheets file** by using the Anymail Finder API. - Clone this [sheet](https://docs.google.com/spreadsheets/d/108sjf89zpKRyZbGq9MjcNFQOezP4hi97SJ_uAkXc8WI/edit?usp=sharing) - Get your (Anymail Finder)[https://anymailfinder.com] API Key (FREE Trial) - In "Check email status" node Create a credential named “Anymail Finder” of type HTTP Header Auth. In the “Name” field, enter "Authorization" and in the "Value" field, enter your Anymail Finder API key (in the format YOUR_API_KEY) |
| Get Leads          | Google Sheets           | Fetch leads pending verification   | Manual Trigger    | Loop Over Items    |                                                                                                                  |
| Loop Over Items    | SplitInBatches          | Process leads in batches           | Get Leads         | Check email status, Loop Over Items |                                                                                                                  |
| Check email status | HTTP Request            | Query Anymail Finder API for email validation | Loop Over Items  | Update email status | This node sends the request to Anymailfinder. Make sure you've connected your API key in the credentials (HTTP Header Auth). |
| Update email status| Google Sheets           | Update email verification status   | Check email status | Loop Over Items    |                                                                                                                  |
| Sticky Note        | Sticky Note             | Documentation and instructions     | None              | None              | See content in Manual Trigger node row.                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node with default settings.  
   - This node will initiate the workflow manually.

2. **Add Google Sheets Node ("Get Leads")**  
   - Set operation to "Read".  
   - Configure credentials with your Google Sheets OAuth2 account.  
   - Set "Document ID" to `108sjf89zpKRyZbGq9MjcNFQOezP4hi97SJ_uAkXc8WI` (or your own copy).  
   - Set "Sheet Name" to "Foglio1" or appropriate sheet (gid=0).  
   - Apply a filter on the "VERIFY" column to select only rows requiring verification.  
   - Connect "Manual Trigger" output to "Get Leads" input.

3. **Add SplitInBatches Node ("Loop Over Items")**  
   - Set to default batch size (or specify as needed, e.g., 1 for sequential processing).  
   - Connect "Get Leads" output to "Loop Over Items" input.

4. **Add HTTP Request Node ("Check email status")**  
   - Set HTTP Method to POST.  
   - Set URL to `https://api.anymailfinder.com/v5.1/verify-email`.  
   - Under "Body Parameters," add parameter "email" with value `={{ $json["EMAIL "] }}` (ensure to match exact column name including spaces).  
   - Set authentication to "HTTP Header Auth".  
   - Create new credentials:  
     - Name: "Anymail Finder".  
     - Header Name: "Authorization".  
     - Header Value: Your Anymail Finder API key (e.g., `YOUR_API_KEY`).  
   - Set timeout to 180000 ms (3 minutes).  
   - Set "On Error" to "Continue on Fail" to prevent workflow interruption.  
   - Connect "Loop Over Items" second output (batch items) to "Check email status".

5. **Add Google Sheets Node ("Update email status")**  
   - Set operation to "Update".  
   - Use same Google Sheets credentials as "Get Leads".  
   - Document ID and Sheet Name identical to "Get Leads".  
   - In column mapping, map "VERIFY" to `={{ $json.email_status }}` (field from API response).  
   - Use "row_number" column from batch item to match the correct row for update.  
   - Connect "Check email status" output to "Update email status" input.

6. **Loop Back from "Update email status" to "Loop Over Items"**  
   - Connect "Update email status" output back to the first input of "Loop Over Items" to continue processing remaining batches.

7. **Add Sticky Note for Documentation (Optional)**  
   - Create a sticky note with the provided instructions and helpful links for users.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                         | Context or Link                                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Clone the Google Sheet used here for your own data: [Google Sheet link](https://docs.google.com/spreadsheets/d/108sjf89zpKRyZbGq9MjcNFQOezP4hi97SJ_uAkXc8WI/edit?usp=sharing)              | Base data source for leads                                                                                              |
| Obtain a free API Key from Anymail Finder: [https://anymailfinder.com](https://anymailfinder.com)                                                                                      | Required for HTTP Request node authentication                                                                           |
| Credential setup for Anymail Finder: Use HTTP Header Auth with header name "Authorization" and value set to your API key                                                             | Credential configuration for API authentication                                                                         |
| Workflow handles errors gracefully by continuing on API request errors, avoiding total workflow failure                                                                              | Important for robustness                                                                                                 |
| Ensure Google Sheets API quota and OAuth credentials have sufficient permissions and scopes for read and update operations                                                           | Prevents authentication or permission-related failures                                                                  |

---

**Disclaimer:** This documentation is based exclusively on an automated workflow created with n8n, respecting current content policies and containing no illegal or protected elements. All data handled are legal and publicly accessible.