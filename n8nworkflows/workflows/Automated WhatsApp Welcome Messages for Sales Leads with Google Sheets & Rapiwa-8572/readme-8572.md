Automated WhatsApp Welcome Messages for Sales Leads with Google Sheets & Rapiwa

https://n8nworkflows.xyz/workflows/automated-whatsapp-welcome-messages-for-sales-leads-with-google-sheets---rapiwa-8572


# Automated WhatsApp Welcome Messages for Sales Leads with Google Sheets & Rapiwa

### 1. Workflow Overview

This workflow automates sending personalized WhatsApp welcome messages to sales leads listed in a Google Sheet, using the Rapiwa API as a non-official WhatsApp messaging gateway. It runs every 5 minutes, continuously processing new leads marked for messaging. The workflow includes logical blocks for scheduled triggering, data fetching from Google Sheets, phone number cleaning, WhatsApp number validation, message sending, status updating, and controlled pacing with batching and wait periods.

**Logical blocks:**

- **1.1 Schedule Trigger**: Runs the workflow every 5 minutes to ensure timely processing of new leads.
- **1.2 Fetch Leads from Google Sheets**: Retrieves leads from a specified Google Sheet filtered by a "check" column indicating readiness for messaging.
- **1.3 Lead Limiting and Batch Processing**: Limits the number of leads to 60 per run and processes them one-by-one.
- **1.4 Phone Number Cleaning**: Sanitizes WhatsApp numbers by removing non-digit characters to ensure API compatibility.
- **1.5 WhatsApp Number Validation**: Uses Rapiwa API to verify if the cleaned WhatsApp numbers are valid and active.
- **1.6 Conditional Messaging and Status Update**: Sends personalized WhatsApp messages via Rapiwa for valid numbers; updates Google Sheet rows with success or failure statuses accordingly.
- **1.7 Controlled Delay**: Waits 5 seconds between sending messages to prevent API or WhatsApp blocking.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  Initiates the workflow automatically every 5 minutes to check for new leads and process them without manual intervention.

- **Nodes Involved:**  
  - Trigger Every 5 Minute1

- **Node Details:**  
  - Type: Schedule Trigger  
  - Configuration: Interval set to every 5 minutes  
  - Input: None (trigger node)  
  - Output: Starts workflow execution  
  - Edge Cases: Workflow might not trigger if n8n instance is offline or misconfigured  
  - Version: 1.2  

#### 2.2 Fetch Leads from Google Sheets

- **Overview:**  
  Fetches all leads from the Google Sheet where the `check` column is filled (indicating leads ready to be messaged). Uses Google Sheets OAuth2 credentials for access.

- **Nodes Involved:**  
  - Fetch All Pending Queries for Messaging1

- **Node Details:**  
  - Type: Google Sheets  
  - Configuration:  
    - Document ID set to a specific Google Sheet (`1amkVSIXrhOkf86YDYaddOcAamhUC4DlvFiSg3cpAH78`)  
    - Sheet GID: 0 (Sheet1)  
    - Filter: Rows where `check` column is filled (non-empty)  
    - Operation: Read rows  
  - Input: Trigger node  
  - Output: List of leads with their details  
  - Edge Cases:  
    - OAuth2 token expiration or permission errors  
    - Empty or malformed data in the sheet  
    - Incorrect Document ID or Sheet GID  
  - Version: 4.6  

#### 2.3 Lead Limiting and Batch Processing

- **Overview:**  
  Limits processing to a maximum of 60 leads per run to avoid API rate limits or performance issues. Then processes each lead individually in batches.

- **Nodes Involved:**  
  - Limit1  
  - Loop Over Items

- **Node Details:**  
  - Limit1  
    - Type: Limit  
    - Config: Maximum items = 60  
    - Input: Leads fetched from Google Sheets  
    - Output: Up to 60 leads  
    - Edge Cases: If more than 60 leads are ready, some will wait for next cycle  
    - Version: 1  

  - Loop Over Items  
    - Type: SplitInBatches  
    - Config: Default settings to process items one by one  
    - Input: Limited leads  
    - Output: Single lead per execution cycle  
    - Edge Cases: Empty batch if no leads; potential for stuck loops if node fails  
    - Version: 3  

#### 2.4 Phone Number Cleaning

- **Overview:**  
  Cleans the WhatsApp number field by removing all non-digit characters to ensure the number is in a valid format for API use.

- **Nodes Involved:**  
  - Clean WhatsApp Number1

- **Node Details:**  
  - Type: Code (JavaScript)  
  - Configuration: Custom JS code that:  
    - Converts the "WhatsApp No" field to a string safely  
    - Removes all non-digit characters using regex  
    - Updates the item’s "WhatsApp No" with the cleaned number  
  - Input: Single lead from Loop Over Items  
  - Output: Lead with cleaned WhatsApp number  
  - Edge Cases:  
    - Null or missing phone number field  
    - Non-string formats that could cause string conversion issues  
  - Version: 2  

#### 2.5 WhatsApp Number Validation

- **Overview:**  
  Validates the cleaned WhatsApp number via Rapiwa API to confirm it is a real and active WhatsApp account.

- **Nodes Involved:**  
  - Check valid whatsapp number Using Rapiwa1

- **Node Details:**  
  - Type: HTTP Request  
  - Configuration:  
    - POST to `https://app.rapiwa.com/api/verify-whatsapp`  
    - Body parameter: `number` set to the cleaned number as string  
    - Authentication: Bearer token (Rapiwa Bearer Auth credential)  
  - Input: Cleaned phone number node output  
  - Output: JSON response indicating if number exists and is valid  
  - Edge Cases:  
    - API errors (rate limiting, network issues, invalid token)  
    - Incorrect number format despite cleaning  
  - Version: 4.2  

#### 2.6 Conditional Messaging and Status Update

- **Overview:**  
  Depending on validation result, sends a personalized WhatsApp message or updates the sheet to mark the lead as not sent and unverified. Afterwards, updates Google Sheets rows with respective statuses.

- **Nodes Involved:**  
  - If  
  - Send Message Using Rapiwa1  
  - Update rows: sent & verified1  
  - Update rows: not sent & unverified1

- **Node Details:**  
  - If  
    - Type: If (conditional)  
    - Condition: Checks if `data.exists` in the validation response is true  
    - Input: Validation response  
    - Output:  
      - True: Proceed to message sending  
      - False: Proceed to mark as failed  
    - Edge Cases: Failure if expression referencing `data.exists` is missing or malformed  
    - Version: 2.2  

  - Send Message Using Rapiwa1  
    - Type: HTTP Request  
    - Configuration:  
      - POST to `https://app.rapiwa.com/api/send-message`  
      - Body parameters:  
        - `number` from cleaned WhatsApp number  
        - `message_type` fixed as "text"  
        - `message`: Personalized with name (from `name ` field with trailing space)  
      - Auth: Bearer token (Rapiwa Bearer Auth)  
    - Input: If node (true branch)  
    - Output: API response for message sending  
    - Edge Cases: API rejection, network failure, incorrect message formatting  
    - Version: 4.2  

  - Update rows: sent & verified1  
    - Type: Google Sheets  
    - Configuration:  
      - Updates row identified by `row_number`  
      - Sets `check` = "checked", `status` = "sent", `validity` = "verified"  
    - Input: Success response of Send Message  
    - Output: Confirmation of update  
    - Edge Cases: Google API errors, row not found, permission issues  
    - Version: 4.6  

  - Update rows: not sent & unverified1  
    - Type: Google Sheets  
    - Configuration:  
      - Updates row identified by `row_number`  
      - Sets `check` = "checked", `status` = "not sent", `validity` = "unverified"  
    - Input: If node (false branch)  
    - Output: Confirmation of update  
    - Edge Cases: Same as above  
    - Version: 4.6  

#### 2.7 Controlled Delay

- **Overview:**  
  Waits 5 seconds between processing each lead to avoid hitting API rate limits or being blocked by WhatsApp.

- **Nodes Involved:**  
  - Wait1

- **Node Details:**  
  - Type: Wait  
  - Configuration: 5-second delay (default, no custom parameters shown but implied by sticky notes)  
  - Input: After Google Sheets update nodes (both success and failure paths)  
  - Output: Loops back to process next lead in batch  
  - Edge Cases: Workflow stuck if wait node fails, long delays if many leads  
  - Version: 1.1  

---

### 3. Summary Table

| Node Name                             | Node Type          | Functional Role                                        | Input Node(s)                          | Output Node(s)                            | Sticky Note                                                                                                  |
|-------------------------------------|--------------------|-------------------------------------------------------|--------------------------------------|------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Sticky Note                         | Sticky Note        | Workflow overview and instructions                    | None                                 | None                                     | # Automate WhatsApp Welcome Messages from Google Sheets Using n8n and Rapiwa API [Detailed workflow description] |
| Trigger Every 5 Minute1              | Schedule Trigger   | Runs workflow every 5 minutes                          | None                                 | Fetch All Pending Queries for Messaging1 | ## Workflow Schedule This workflow runs automatically every 5 minutes...                                      |
| Fetch All Pending Queries for Messaging1 | Google Sheets      | Fetch leads filtered by 'check' column                 | Trigger Every 5 Minute1               | Limit1                                   | ## Google Sheets Connection Fetch leads marked for messaging from correct sheet and document                  |
| Limit1                              | Limit              | Limits lead processing to 60 leads per run             | Fetch All Pending Queries for Messaging1 | Loop Over Items                          | ## Limit and Process Leads Limits to 60 leads, processes one by one                                            |
| Loop Over Items                     | SplitInBatches     | Processes each lead one at a time                       | Limit1                               | Clean WhatsApp Number1                    | ## Limit and Process Leads Processes leads one by one                                                         |
| Clean WhatsApp Number1              | Code               | Cleans WhatsApp numbers (removes non-digits)           | Loop Over Items                      | Check valid whatsapp number Using Rapiwa1 | ## Clean and Verify WhatsApp Numbers Cleans numbers for API compatibility                                      |
| Check valid whatsapp number Using Rapiwa1 | HTTP Request       | Validates WhatsApp number via Rapiwa API               | Clean WhatsApp Number1               | If                                       | ## Clean and Verify WhatsApp Numbers Validates WhatsApp number with Rapiwa                                      |
| If                                 | If                 | Branches on validation result (valid number or not)    | Check valid whatsapp number Using Rapiwa1 | Send Message Using Rapiwa1 / Update rows: not sent & unverified1 | ## Message Sending & Status Update Conditional send or mark failure                                           |
| Send Message Using Rapiwa1          | HTTP Request       | Sends personalized WhatsApp message via Rapiwa API    | If (true branch)                    | Update rows: sent & verified1             | ## Message Sending & Status Update Sends message if valid                                                    |
| Update rows: sent & verified1       | Google Sheets      | Updates sheet marking lead as sent and verified        | Send Message Using Rapiwa1          | Wait1                                     | ## Message Sending & Status Update Marks message sent and number verified                                     |
| Update rows: not sent & unverified1 | Google Sheets      | Updates sheet marking lead as not sent and unverified  | If (false branch)                   | Wait1                                     | ## Message Sending & Status Update Marks message not sent and number unverified                               |
| Wait1                              | Wait               | Waits 5 seconds between messages to avoid blocks       | Update rows: sent & verified1 / Update rows: not sent & unverified1 | Loop Over Items                          | ## Message Sending & Status Update Waits to avoid API/WhatsApp blocking                                      |
| Sticky Note7                       | Sticky Note        | Notes on workflow schedule                              | None                               | None                                     | ## Workflow Schedule This workflow runs automatically every 5 minutes                                        |
| Sticky Note8                       | Sticky Note        | Notes on Google Sheets connection and filtering         | None                               | None                                     | ## Google Sheets Connection Fetch leads marked for messaging                                                  |
| Sticky Note9                       | Sticky Note        | Notes on limiting and batch processing                   | None                               | None                                     | ## Limit and Process Leads Limits to 60 leads, processes one by one                                           |
| Sticky Note10                      | Sticky Note        | Notes on cleaning and verifying WhatsApp numbers        | None                               | None                                     | ## Clean and Verify WhatsApp Numbers Cleans numbers and validates with API                                    |
| Sticky Note11                      | Sticky Note        | Notes on message sending, status update, and wait       | None                               | None                                     | ## Message Sending & Status Update Conditional messaging and status updates, with pacing                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to run every 5 minutes.

2. **Add Google Sheets node to fetch leads**  
   - Type: Google Sheets  
   - Operation: Read rows  
   - Document ID: Your Google Sheet ID (e.g., `1amkVSIXrhOkf86YDYaddOcAamhUC4DlvFiSg3cpAH78`)  
   - Sheet GID: 0 (Sheet1)  
   - Filter rows where `check` column is not empty  
   - Connect credentials via Google Sheets OAuth2  
   - Connect Schedule Trigger output to this node's input.

3. **Add a Limit node**  
   - Type: Limit  
   - Max Items: 60  
   - Connect Google Sheets output to Limit input.

4. **Add a SplitInBatches node**  
   - Type: SplitInBatches  
   - Default batch size (1) to process one lead at a time  
   - Connect Limit node output to this node.

5. **Add a Code node to clean WhatsApp numbers**  
   - Type: Code (JavaScript)  
   - Paste this code:  
     ```javascript
     const items = $input.all();
     const updatedItems = items.map((item) => {
       let waNo = item?.json["WhatsApp No"];
       let waNoStr = "";
       if (typeof waNo === 'string') {
         waNoStr = waNo;
       } else if (waNo !== undefined && waNo !== null) {
         waNoStr = String(waNo);
       }
       const cleanedNumber = waNoStr.replace(/\D/g, "");
       item.json["WhatsApp No"] = cleanedNumber;
       return item;
     });
     return updatedItems;
     ```  
   - Connect SplitInBatches output to Code node input.

6. **Add HTTP Request node to validate WhatsApp number**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/verify-whatsapp`  
   - Authentication: HTTP Bearer (configure Rapiwa Bearer Token)  
   - Body parameters: JSON with field `"number"` set to `{{ String($json["WhatsApp No"]) }}`  
   - Connect Code node output to this node.

7. **Add If node to check validation response**  
   - Type: If (Version 2)  
   - Condition: Boolean equals true on expression `{{ $json.data.exists }}`  
   - Connect HTTP Request output to If node input.

8. **Add HTTP Request node for sending WhatsApp message**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.rapiwa.com/api/send-message`  
   - Authentication: HTTP Bearer (same Rapiwa token)  
   - Body parameters:  
     - `number`: `{{ $json["WhatsApp No"] }}`  
     - `message_type`: `text`  
     - `message`:  
       ```
       Hi *{{ $json["name "] }}*,
       Thanks for purchasing from *Spagreen Creative*. Your product purchase has been confirmed.
       We truly appreciate your trust in our products. If you have any questions, feel free to message us here anytime!
       – Team *Spagreen Creative*
       ```  
   - Connect If node "true" output to this node.

9. **Add Google Sheets node to update rows as sent & verified**  
   - Type: Google Sheets  
   - Operation: Update row  
   - Document ID and Sheet GID same as fetch node  
   - Update columns:  
     - `check`: "checked"  
     - `status`: "sent"  
     - `validity`: "verified"  
     - Use `row_number` from item for matching row  
   - Connect Send Message node output to this node.

10. **Add Google Sheets node to update rows as not sent & unverified**  
    - Type: Google Sheets  
    - Operation: Update row  
    - Same config as above, but columns:  
      - `check`: "checked"  
      - `status`: "not sent"  
      - `validity`: "unverified"  
    - Connect If node "false" output to this node.

11. **Add Wait node**  
    - Type: Wait  
    - Duration: 5 seconds (default)  
    - Connect both Update rows nodes (sent & verified, not sent & unverified) outputs to Wait node.

12. **Connect Wait node output back to SplitInBatches node**  
    - This creates the loop to process the next lead in batch.

13. **Configure credentials**  
    - Google Sheets OAuth2: Ensure proper OAuth2 credentials with access to the Google Sheet.  
    - Rapiwa Bearer Auth: Create HTTP Bearer credentials with your Rapiwa API token.

14. **Validate and activate the workflow**  
    - Test with a sample Google Sheet matching the required column format (`WhatsApp No`, `name ` with trailing space, `row_number`, `check`, `validity`, `status`).  
    - Ensure the workflow triggers every 5 minutes and processes leads as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                           | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| The Google Sheet must have a `name ` column with a trailing space exactly as shown; this is critical for message personalization.                     | Workflow description and Google Sheet sample: https://docs.google.com/spreadsheets/d/1amkVSIXrhOkf86YDYaddOcAamhUC4DlvFiSg3cpAH78/edit?usp=sharing |
| Keep phone numbers in international format without spaces or symbols for best results after cleaning step.                                             | Workflow best practices                                                                                     |
| To avoid getting blocked by Rapiwa or WhatsApp, do not increase batch size beyond 60 and keep wait time of 5 seconds between messages.                | Sticky note instructions in workflow                                                                       |
| Workflow designed for non-official WhatsApp API usage via Rapiwa; official WhatsApp Business API workflows may require different setup.               | Workflow overview                                                                                           |
| You can customize this workflow to log errors, send email notifications, or attach files like PDFs/images by extending the HTTP Request node payload. | Suggestions in sticky notes                                                                                  |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. It strictly complies with content policies and contains no illegal or protected elements. All processed data is legal and public.