WhatsApp Number Verification & Confirmation System with Rapiwa API and Google Sheets

https://n8nworkflows.xyz/workflows/whatsapp-number-verification---confirmation-system-with-rapiwa-api-and-google-sheets-8576


# WhatsApp Number Verification & Confirmation System with Rapiwa API and Google Sheets

### 1. Workflow Overview

This workflow automates WhatsApp number verification and confirmation messaging for form submissions using the Rapiwa API and Google Sheets. It is designed for use cases such as lead capture, event registration, or customer onboarding via WhatsApp communication.

The workflow consists of the following logical blocks:

- **1.1 Input Reception and Formatting:** Receives form submission data via a webhook and formats it for processing.
- **1.2 Batch Processing:** Handles multiple submissions by processing them one at a time.
- **1.3 Phone Number Cleaning:** Standardizes the WhatsApp number by removing non-numeric characters.
- **1.4 WhatsApp Number Verification:** Uses the Rapiwa API to check if the cleaned number is registered on WhatsApp.
- **1.5 Conditional Handling:** Routes the flow depending on whether the number is verified or not.
- **1.6 WhatsApp Confirmation Messaging:** Sends a confirmation message to verified numbers via Rapiwa.
- **1.7 Data Persistence:** Appends submission data to a Google Sheet, marking entries as verified or unverified.
- **1.8 Throttling / Rate Limiting:** Adds a delay after each write operation to avoid API rate limits or Google Sheets conflicts.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Formatting

**Overview:**  
Receives form submission data using a Webhook node, then formats the incoming JSON payload to extract relevant fields and add a submission date.

**Nodes Involved:**  
- Webhook  
- Format Webhook Response Data  
- Sticky Note1 (documentation)

**Node Details:**

- **Webhook**  
  - Type: `Webhook`  
  - Role: Entry point to receive data via HTTP POST  
  - Configuration: HTTP method POST, path set to a unique identifier  
  - Input: HTTP POST payload with fields `business_name`, `location`, `whatsapp`, `email`, `name`  
  - Output: Raw webhook data forwarded to the next node  
  - Potential failures: Invalid/malformed JSON input, unauthorized access if webhook is public  

- **Format Webhook Response Data**  
  - Type: `Code` (JavaScript)  
  - Role: Extracts submitted fields and adds `submitted_date` in `YYYY-MM-DD` format  
  - Configuration: JavaScript code maps input items, extracts `business_name`, `location`, `whatsapp`, `email`, `name`, and adds current date as `submitted_date`  
  - Input: Webhook data  
  - Output: Cleaned and formatted JSON objects for downstream processing  
  - Edge cases: Missing fields in input could cause undefined values; no explicit validation  

---

#### 1.2 Batch Processing

**Overview:**  
Processes multiple submission records individually using batching to ensure scalability and controlled execution.

**Nodes Involved:**  
- Loop Over Items  
- Sticky Note2 (documentation)

**Node Details:**

- **Loop Over Items**  
  - Type: `SplitInBatches`  
  - Role: Splits incoming items to process one at a time  
  - Configuration: Default batch size (1 item per batch), no special options set  
  - Input: Array of formatted submission items  
  - Output: Single item per execution to next node  
  - Edge cases: Large batch sizes may cause timeouts; no explicit batch failure handling  

---

#### 1.3 Phone Number Cleaning

**Overview:**  
Cleans the WhatsApp number string by removing all non-digit characters to prepare it for API compatibility.

**Nodes Involved:**  
- Cleane Number  
- Sticky Note3 (documentation)

**Node Details:**

- **Cleane Number**  
  - Type: `Code` (JavaScript)  
  - Role: Sanitizes phone number strings by removing non-numeric characters  
  - Configuration: JavaScript code converts number to string if needed, then uses regex to strip non-digits  
  - Input: Single item with `whatsapp` number  
  - Output: Updated item with cleaned `number` field  
  - Edge cases: Empty or null input results in empty string; no validation if number length is plausible for WhatsApp  
  - Potential failures: Expression errors if input data structure changes  

---

#### 1.4 WhatsApp Number Verification

**Overview:**  
Sends the cleaned number to Rapiwa’s WhatsApp verification API to confirm if the number is registered on WhatsApp.

**Nodes Involved:**  
- Check valid whatsapp number Using Rapiwa  
- Sticky Note3 (documentation)

**Node Details:**

- **Check valid whatsapp number Using Rapiwa**  
  - Type: `HTTP Request`  
  - Role: Calls Rapiwa API endpoint `/api/verify-whatsapp` with cleaned number  
  - Configuration: POST request with JSON body containing `number`, bearer token authentication via Rapiwa Bearer Auth credentials  
  - Input: Item with cleaned `number` field  
  - Output: JSON response with `data.exists` boolean indicating WhatsApp registration status  
  - Edge cases: HTTP errors (timeout, 4xx/5xx), invalid token, unexpected API response format  
  - Credential requirement: `Rapiwa Bearer Auth` with valid token  

---

#### 1.5 Conditional Handling

**Overview:**  
Branches the workflow based on the verification result, separating verified and unverified WhatsApp numbers.

**Nodes Involved:**  
- If  
- Sticky Note4 (documentation)

**Node Details:**

- **If**  
  - Type: `If` node  
  - Role: Checks if `data.exists` equals boolean `true`  
  - Configuration: Condition: `{{$json.data.exists}} === true` (strict boolean)  
  - Input: API response from verification node  
  - Output:  
    - True branch: verified number path  
    - False branch: unverified number path  
  - Edge cases: Missing or malformed `data.exists` field could cause false negatives  

---

#### 1.6 WhatsApp Confirmation Messaging

**Overview:**  
Sends a confirmation WhatsApp message to verified users via Rapiwa API.

**Nodes Involved:**  
- Send Message Using Rapiwa  
- Sticky Note5 (documentation)

**Node Details:**

- **Send Message Using Rapiwa**  
  - Type: `HTTP Request`  
  - Role: Sends WhatsApp message to verified number using Rapiwa’s `/api/send-message` endpoint  
  - Configuration: POST request with JSON body containing `number`, `message_type: text`, and a personalized message using the user’s name  
  - Input: Verified number data from `If` node true branch  
  - Output: Confirmation response forwarded to append to Google Sheet  
  - Credential requirement: `Rapiwa Bearer Auth`  
  - Edge cases: API errors, message failures, invalid number format despite verification  

---

#### 1.7 Data Persistence

**Overview:**  
Appends the form submission data into a Google Sheet, marking the entry as `verified` or `unverified` depending on WhatsApp number status.

**Nodes Involved:**  
- verified append row in sheet  
- unverified append row in sheet  
- Sticky Note5 & Sticky Note6 (documentation)

**Node Details:**

- **verified append row in sheet**  
  - Type: `Google Sheets`  
  - Role: Appends data row with `validity` set to `verified`  
  - Configuration: Append operation to a specific Google Sheet and worksheet (Sheet1), maps fields including `Business Name`, `Location`, `WhatsApp Number`, `Email ` (note trailing space), `Name`, `Date`, and `validity`  
  - Input: Data with verified status  
  - Output: Triggers `Wait1` node  
  - Credential requirement: Google Sheets OAuth2 credentials  
  - Edge cases: API quota exceeded, invalid or missing sheet permissions, mapping errors  

- **unverified append row in sheet**  
  - Type: `Google Sheets`  
  - Role: Same as above but sets `validity` to `unverified`  
  - Configuration: Identical mapping, different data content  
  - Input: Data from false branch of the `If` node  
  - Output: Triggers `Wait1` node  
  - Credential requirement: Google Sheets OAuth2 credentials  
  - Edge cases: Same as above  

---

#### 1.8 Throttling / Rate Limiting

**Overview:**  
Introduces a short delay after each append operation to prevent API throttling and potential write conflicts in Google Sheets.

**Nodes Involved:**  
- Wait1  
- Sticky Note6 (documentation)

**Node Details:**

- **Wait1**  
  - Type: `Wait`  
  - Role: Pauses workflow execution for 2 seconds after each row append  
  - Configuration: 2 seconds delay  
  - Input: Triggered from append row nodes  
  - Output: Ends the flow for that batch item  
  - Edge cases: Delay could be adjusted if rate limits are stricter or more lenient  

---

### 3. Summary Table

| Node Name                      | Node Type            | Functional Role                         | Input Node(s)                 | Output Node(s)                      | Sticky Note                                                                                          |
|--------------------------------|----------------------|---------------------------------------|------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook                        | Webhook              | Receives form submission data         | -                            | Format Webhook Response Data       | ## Trigger via Webhook - example input JSON and formatting instructions (Sticky Note1)             |
| Format Webhook Response Data   | Code                 | Extracts fields, adds submission date| Webhook                      | Loop Over Items                   | See above                                                                                        |
| Loop Over Items                | SplitInBatches       | Processes submissions one at a time   | Format Webhook Response Data | Cleane Number                    | Batch processing explanation (Sticky Note2)                                                      |
| Cleane Number                 | Code                 | Cleans WhatsApp number                 | Loop Over Items              | Check valid whatsapp number Using Rapiwa | Cleans number & prepares for API call (Sticky Note3)                                              |
| Check valid whatsapp number Using Rapiwa | HTTP Request         | Verifies if number is WhatsApp registered | Cleane Number               | If                              | See note above (Sticky Note3)                                                                    |
| If                            | If                   | Routes based on verification result   | Check valid whatsapp number Using Rapiwa | Send Message Using Rapiwa / unverified append row in sheet | Conditional logic details (Sticky Note4)                                                          |
| Send Message Using Rapiwa      | HTTP Request         | Sends WhatsApp confirmation message   | If (true branch)              | verified append row in sheet       | Sends confirmation and appends verified data (Sticky Note5)                                      |
| verified append row in sheet    | Google Sheets        | Appends verified submission           | Send Message Using Rapiwa     | Wait1                           | Appends verified data to Google Sheet (Sticky Note5)                                             |
| unverified append row in sheet  | Google Sheets        | Appends unverified submission         | If (false branch)             | Wait1                           | Appends unverified data to Google Sheet (Sticky Note6)                                           |
| Wait1                         | Wait                 | Throttles write operations             | verified append row in sheet / unverified append row in sheet | -                             | Delays 2 seconds to avoid rate limits (Sticky Note6)                                             |
| Sticky Note                   | Sticky Note          | Documentation and workflow overview   | -                            | -                               | Comprehensive workflow overview and instructions                                                |
| Sticky Note1                  | Sticky Note          | Documentation: webhook & formatting   | -                            | -                               | See node Webhook and Format Webhook Response Data                                                |
| Sticky Note2                  | Sticky Note          | Documentation: batch processing       | -                            | -                               | See node Loop Over Items                                                                         |
| Sticky Note3                  | Sticky Note          | Documentation: number cleaning & verification | -                            | -                               | See nodes Cleane Number and Check valid whatsapp number Using Rapiwa                              |
| Sticky Note4                  | Sticky Note          | Documentation: conditional logic      | -                            | -                               | See node If                                                                                    |
| Sticky Note5                  | Sticky Note          | Documentation: messaging & verified append | -                            | -                               | See nodes Send Message Using Rapiwa and verified append row in sheet                             |
| Sticky Note6                  | Sticky Note          | Documentation: unverified append & wait | -                            | -                               | See nodes unverified append row in sheet and Wait1                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., `a9b6a936-e5f2-4d4c-9cf9-182de0a970d5`)  
   - No authentication (optional: secure endpoint)  
   - Purpose: Receive form submission with fields: `business_name`, `location`, `whatsapp`, `email`, `name`.

2. **Add Code Node: Format Webhook Response Data**  
   - Type: Code (JavaScript)  
   - Code: Extract `business_name`, `location`, `whatsapp`, `email`, `name` from webhook payload. Add `submitted_date` as current date in `YYYY-MM-DD` format.  
   - Input: Connect from Webhook node.

3. **Add SplitInBatches Node: Loop Over Items**  
   - Type: SplitInBatches  
   - Batch Size: 1 (default)  
   - Purpose: Process each submission individually.  
   - Input: Connect from Format Webhook Response Data node.

4. **Add Code Node: Cleane Number**  
   - Type: Code (JavaScript)  
   - Code: Convert `whatsapp` field to string if needed, remove all non-digit characters, save result as `number`.  
   - Input: Connect from Loop Over Items node.

5. **Add HTTP Request Node: Check valid whatsapp number Using Rapiwa**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/verify-whatsapp`  
   - Authentication: HTTP Bearer Auth (configure with Rapiwa Bearer Token)  
   - Body Parameters: JSON with parameter `"number"` set to cleaned number, e.g. `{{$json.whatsapp}}` or `{{$json.number}}` depending on mapping  
   - Input: Connect from Cleane Number node.

6. **Add If Node**  
   - Type: If  
   - Condition: Check if `{{$json.data.exists}} === true` (boolean strict)  
   - Input: Connect from Check valid whatsapp number Using Rapiwa node.

7. **Add HTTP Request Node: Send Message Using Rapiwa**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/send-message`  
   - Authentication: HTTP Bearer Auth (same credential as above)  
   - Body Parameters:  
     - `number`: `{{$json.data.phone}}` or from cleaned number source  
     - `message_type`: `text`  
     - `message`: `"Hi {{name}}, Thanks! Your form has been submitted successfully."` (use expression referencing name from Cleane Number node)  
   - Input: Connect from If node’s true branch.

8. **Add Google Sheets Node: verified append row in sheet**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Google Sheet ID (configured from your Google Sheets)  
   - Sheet Name: Sheet1 (or specific sheet tab)  
   - Columns to map:  
     - Business Name  
     - Location  
     - WhatsApp Number  
     - Email (note trailing space in column header)  
     - Name  
     - Date  
     - validity = "verified" (static value)  
   - Input: Connect from Send Message Using Rapiwa node.

9. **Add Google Sheets Node: unverified append row in sheet**  
   - Configure identically to above, but set `validity` = "unverified"  
   - Input: Connect from If node’s false branch.

10. **Add Wait Node: Wait1**  
    - Type: Wait  
    - Duration: 2 seconds  
    - Input: Connect from both Google Sheets append nodes (verified and unverified).

11. **Credential Setup**  
    - Google Sheets OAuth2 credentials: create and connect with Google API Console credentials, ensure access to target spreadsheet.  
    - Rapiwa Bearer Auth: HTTP Bearer token with your Rapiwa API token, ensure access to endpoints `/api/verify-whatsapp` and `/api/send-message`.

12. **Testing**  
    - Send a POST request with sample JSON to the webhook URL.  
    - Verify processing, correct WhatsApp validation, message sending, and data appending in Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| A Google Sheet formatted as per the example with exact column names including a trailing space in `Email ` is required.                                                                                                           | [Sample Google Sheet](https://docs.google.com/spreadsheets/d/1H29z_8tnsu8AvCsI7o1SjiV-5LDTwiVVQk-BNr8SMk0/edit?usp=sharing)  |
| The webhook URL should be kept secure to avoid unauthorized submissions. Consider adding authentication or IP restrictions.                                                                                                       | Security best practice                                                                                                        |
| Adjust the delay time in the Wait node if experiencing rate limits or write conflicts with Google Sheets or Rapiwa API.                                                                                                           | Rate limiting guidance                                                                                                        |
| Modify the WhatsApp message content in "Send Message Using Rapiwa" node to reflect your brand voice and requirements.                                                                                                            | Customization tip                                                                                                             |
| Rapiwa API requires a valid Bearer token and WhatsApp number linked with their platform for both verification and messaging endpoints.                                                                                            | Rapiwa API official documentation                                                                                             |
| The workflow is ideal for WhatsApp-based lead capture, event registration, or customer onboarding workflows requiring validation and confirmation.                                                                               | Use case guidance                                                                                                             |

---

This comprehensive documentation enables advanced users and automation agents to understand, reproduce, and modify the workflow effectively, anticipating potential issues with API calls, data formatting, and integration credentials.