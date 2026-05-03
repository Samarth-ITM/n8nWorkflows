Automated Daily Customer Win-Back Campaign with AI Offers

https://n8nworkflows.xyz/workflows/automated-daily-customer-win-back-campaign-with-ai-offers-4001


# Automated Daily Customer Win-Back Campaign with AI Offers

### 1. Workflow Overview

This workflow automates a **daily customer win-back campaign** to reduce churn by targeting customers predicted to be at high risk of leaving. It runs on a schedule, fetches customer data from Google Sheets, filters for high-risk customers without recent campaigns, and then uses **Google Gemini AI** to generate personalized offers. These offers are emailed via Gmail, and all actions are logged for monitoring.

The workflow logically divides into four main blocks:

- **1.1 Daily Trigger & Data Fetch:** Automatically starts daily and retrieves customer data from Google Sheets.
- **1.2 Filtering & Eligibility Check:** Filters customers with churn risk above 0.7 and no prior campaign date; branches based on whether eligible customers exist.
- **1.3 Win-Back Offer Generation & Delivery Loop:** For each eligible customer, generates an AI personalized offer, logs the action, and sends the offer email.
- **1.4 No Eligible Customers Logging:** If no customers qualify, logs a “NOT_FOUND” status in the system log.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Trigger & Data Fetch

- **Overview:** Initiates the workflow automatically once every day and fetches all customer data from the designated Google Sheet.

- **Nodes Involved:**  
  - Scheduled Start: Daily Churn Check  
  - Fetch Customer Data from Sheet  

- **Node Details:**

  - **Scheduled Start: Daily Churn Check**  
    - Type: Schedule Trigger  
    - Role: Triggers workflow at a defined interval (daily).  
    - Configuration: Runs every day (default interval with no specific hour set; user should define the exact hour).  
    - Inputs: None (trigger node).  
    - Outputs: Triggers the next node to start data fetch.  
    - Edge Cases: Misconfigured schedule may cause unintended runs or no runs.  
    - Version: 1.2  

  - **Fetch Customer Data from Sheet**  
    - Type: Google Sheets (Read)  
    - Role: Retrieves all customer rows from the ‘Customer Data’ sheet.  
    - Configuration: Uses OAuth2 credentials linked to Google Sheets; specifies Spreadsheet ID and Sheet Name ('Customer Data').  
    - Key Expressions: None specifically; reads all rows.  
    - Input: Trigger from Scheduled Start node.  
    - Output: Outputs all customer records as JSON for filtering.  
    - Edge Cases: API quota limits, invalid credentials, missing or malformed sheet data.  
    - Version: 4.5  

---

#### 1.2 Filtering & Eligibility Check

- **Overview:** Filters customers with a predicted churn score > 0.7 and no prior campaign date, then checks if any customers passed the filter.

- **Nodes Involved:**  
  - Filter High Churn Risk & No Campaign Customers  
  - Check if Eligible Customers Found  

- **Node Details:**

  - **Filter High Churn Risk & No Campaign Customers**  
    - Type: Filter  
    - Role: Selects customers with churn risk above 0.7.  
    - Configuration:  
      - Condition: `$json.predicted_churn_score.toNumber() > 0.7`  
      - Note: The originally described filter also includes checking if `created_campaign_date` is empty, but this condition is not present in JSON; user should verify or adjust logic.  
    - Inputs: All customer records.  
    - Outputs: Pass (eligible customers) or Fail (non-eligible).  
    - Edge Cases: Malformed score data, missing fields, or incorrect data types causing evaluation issues.  
    - Version: 2.2  

  - **Check if Eligible Customers Found**  
    - Type: If  
    - Role: Checks if any customers were filtered as eligible.  
    - Configuration: Condition checks if the filtered list is **not empty** (`{{ $json.isEmpty() }}` is false).  
    - Inputs: Output from Filter node.  
    - Outputs:  
      - True branch: Eligible customers found.  
      - False branch: No eligible customers.  
    - Edge Cases: Incorrect expression evaluation, empty datasets.  
    - Version: 2.2  

---

#### 1.3 Win-Back Offer Generation & Delivery Loop

- **Overview:** Iterates over each eligible customer, generates a personalized AI offer using Google Gemini, logs the offer sent, and emails the customer.

- **Nodes Involved:**  
  - Process Each Eligible Customer (SplitInBatches)  
  - Generate Win-Back Offer (Langchain Chain LLM)  
  - (LLM Model for Offer Generation) (Langchain Chat Model - Google Gemini)  
  - (Parse Offer JSON) (Langchain Output Parser - Structured)  
  - Log Sent Offer in System Log (Google Sheets Append/Update)  
  - Send Win-Back Offer via Email (Gmail)  

- **Node Details:**

  - **Process Each Eligible Customer**  
    - Type: SplitInBatches  
    - Role: Loops through eligible customers one by one to process individually.  
    - Configuration: Default batch size 1 to process sequentially.  
    - Inputs: Array of eligible customers.  
    - Outputs: Single customer record per batch.  
    - Edge Cases: Large datasets may slow processing; batch size can be optimized.  
    - Version: 3  

  - **Generate Win-Back Offer**  
    - Type: Langchain Chain LLM  
    - Role: Composes prompt with customer churn score and preferred categories, sends to LLM for offer generation.  
    - Configuration:  
      - Prompt includes detailed rules for offer types based on churn score intervals: INFORMATIONAL (0.7-0.8), BONUS_POINTS (0.8-0.9), DISCOUNT_PERCENTAGE (0.9-1.0).  
      - Output expected strictly as structured JSON with fields: customer_id, action_taken, offer_type, offer_value, offer_title, offer_details, communication_channel, timestamp.  
    - Inputs: Single customer data from SplitInBatches.  
    - Outputs: Raw LLM response JSON.  
    - Edge Cases: LLM API failures, prompt misinterpretation, malformed JSON output.  
    - Version: 1.5  

  - **(LLM Model for Offer Generation)**  
    - Type: Langchain Chat Model - Google Gemini  
    - Role: Sends the prompt from Chain LLM node to Google Gemini AI for processing.  
    - Configuration:  
      - Uses Google Vertex AI credentials with Gemini 2.0 pro experimental model.  
      - No additional parameter customization.  
    - Inputs: Prompt from Chain LLM node.  
    - Outputs: LLM raw response forwarded to output parser.  
    - Edge Cases: API rate limits, auth failures, latency issues.  
    - Version: 1  

  - **(Parse Offer JSON)**  
    - Type: Langchain Output Parser - Structured  
    - Role: Parses raw LLM JSON output into structured JSON for downstream use.  
    - Configuration: JSON schema example provided to validate and parse output.  
    - Inputs: Raw LLM output.  
    - Outputs: Parsed JSON with offer details.  
    - Edge Cases: Parsing errors if LLM output deviates from schema, incomplete fields.  
    - Version: 1.2  

  - **Log Sent Offer in System Log**  
    - Type: Google Sheets (Append or Update)  
    - Role: Logs action "SENT_WINBACK_OFFER" with timestamp and customer ID in the 'SYSTEM_LOG' sheet.  
    - Configuration:  
      - Maps fields from parsed JSON: system_log = action_taken, date = timestamp, customer_id.  
      - Spreadsheet and sheet configured for system log.  
    - Inputs: Parsed offer JSON.  
    - Outputs: Confirmation of log append.  
    - Edge Cases: Sheet access issues, API quota, malformed data.  
    - Version: 4.5  

  - **Send Win-Back Offer via Email**  
    - Type: Gmail  
    - Role: Sends an email to the customer’s email with the offer details.  
    - Configuration:  
      - Recipient: `user_mail` from the original customer data.  
      - Subject: `offer_title` from parsed JSON.  
      - Body: `offer_details` from parsed JSON as plain text.  
      - Uses OAuth2 Gmail credentials.  
    - Inputs: Parsed JSON for content, plus original customer data for email address.  
    - Outputs: Email sent confirmation.  
    - Edge Cases: Auth failure, quota limits, invalid email address.  
    - Version: 2.1  

---

#### 1.4 No Eligible Customers Logging

- **Overview:** Handles the case where no customers meet the filtering criteria by logging a "NOT_FOUND" status with timestamp into the system log.

- **Nodes Involved:**  
  - Set 'Not Found' Status  
  - Log 'Not Found' in System Log  

- **Node Details:**

  - **Set 'Not Found' Status**  
    - Type: Set  
    - Role: Creates a JSON object with `system_log = NOT_FOUND` and current timestamp.  
    - Configuration: Assigns two fields: `system_log` (string "NOT_FOUND") and `date` (current ISO timestamp `$now`).  
    - Inputs: None (triggered from If node false branch).  
    - Outputs: JSON for logging.  
    - Edge Cases: None significant.  
    - Version: 3.4  

  - **Log 'Not Found' in System Log**  
    - Type: Google Sheets (Append or Update)  
    - Role: Logs the NOT_FOUND status and timestamp in the 'SYSTEM_LOG' sheet.  
    - Configuration: Maps `system_log` and `date` fields accordingly.  
    - Inputs: JSON from Set node.  
    - Outputs: Confirmation of log append.  
    - Edge Cases: Sheet/API issues.  
    - Version: 4.5  

---

### 3. Summary Table

| Node Name                         | Node Type                      | Functional Role                              | Input Node(s)                         | Output Node(s)                             | Sticky Note                                                                                                                                                                                                                                                                        |
|----------------------------------|--------------------------------|----------------------------------------------|-------------------------------------|--------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Scheduled Start: Daily Churn Check | Schedule Trigger               | Starts workflow daily                        | None                                | Fetch Customer Data from Sheet              | # 00. Daily Start & Fetch Customer Data: Automatically triggers workflow daily and fetches customers.                                                                                                                                                                               |
| Fetch Customer Data from Sheet    | Google Sheets (Read)           | Reads all customer data                      | Scheduled Start                     | Filter High Churn Risk & No Campaign Customers | # 00. Daily Start & Fetch Customer Data                                                                                                                                                                                                                                           |
| Filter High Churn Risk & No Campaign Customers | Filter                       | Filters high churn risk customers            | Fetch Customer Data from Sheet      | Check if Eligible Customers Found           | # 01. Filter & Branch: Filters customers with churn > 0.7 and no campaign date; branches accordingly.                                                                                                                                                                              |
| Check if Eligible Customers Found | If                            | Checks if any customers passed filter        | Filter High Churn Risk & No Campaign Customers | Process Each Eligible Customer (true), Set 'Not Found' Status (false) | # 01. Filter & Branch                                                                                                                                                                                                                                                             |
| Process Each Eligible Customer    | SplitInBatches                | Loops through eligible customers             | Check if Eligible Customers Found   | Generate Win-Back Offer                      | # 02. Generate & Send Win-Back Offer (Loop): Processes customers one by one generating and sending offers.                                                                                                                                                                          |
| Generate Win-Back Offer           | Langchain Chain LLM           | Prepares prompt for AI offer generation      | Process Each Eligible Customer      | (LLM Model for Offer Generation)             | # 02. Generate & Send Win-Back Offer (Loop)                                                                                                                                                                                                                                       |
| (LLM Model for Offer Generation) | Langchain Chat Model Google Gemini | Sends prompt to Google Gemini AI              | Generate Win-Back Offer             | (Parse Offer JSON)                           | # 02. Generate & Send Win-Back Offer (Loop)                                                                                                                                                                                                                                       |
| (Parse Offer JSON)                | Langchain Output Parser Structured | Parses AI JSON response                        | (LLM Model for Offer Generation)   | Log Sent Offer in System Log                  | # 02. Generate & Send Win-Back Offer (Loop)                                                                                                                                                                                                                                       |
| Log Sent Offer in System Log      | Google Sheets (Append/Update) | Logs sent offer details                       | (Parse Offer JSON)                  | Send Win-Back Offer via Email                 | # 02. Generate & Send Win-Back Offer (Loop)                                                                                                                                                                                                                                       |
| Send Win-Back Offer via Email     | Gmail                        | Sends personalized offer email to customer  | Log Sent Offer in System Log        | Process Each Eligible Customer (continue loop) | # 02. Generate & Send Win-Back Offer (Loop)                                                                                                                                                                                                                                       |
| Set 'Not Found' Status            | Set                          | Sets log record when no eligible customers   | Check if Eligible Customers Found   | Log 'Not Found' in System Log                 | # 03. Handle No Eligible Customers: Sets status when no customers found.                                                                                                                                                                                                           |
| Log 'Not Found' in System Log     | Google Sheets (Append/Update) | Logs NOT_FOUND status in system log          | Set 'Not Found' Status              | None                                         | # 03. Handle No Eligible Customers: Logs NOT_FOUND status.                                                                                                                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to run daily at desired hour (e.g., 9 AM).  
   - No inputs.  

2. **Add Google Sheets Node to Fetch Customer Data**  
   - Type: Google Sheets (Read)  
   - Configure credentials with Google Sheets OAuth2.  
   - Enter Spreadsheet ID and Sheet Name for ‘Customer Data’.  
   - Set operation to read all rows.  
   - Connect output from Schedule Trigger node.  

3. **Add Filter Node to Select High Churn Risk Customers**  
   - Type: Filter  
   - Add condition: `{{ $json.predicted_churn_score.toNumber() > 0.7 }}`  
   - Optionally add condition for `created_campaign_date` being empty (adjust logic as needed).  
   - Connect input from Fetch Customer Data node.  

4. **Add If Node to Check Eligibility**  
   - Type: If  
   - Condition: `{{ $json.isEmpty() }} == false` — true if customers found.  
   - Connect input from Filter node.  

5. **Add SplitInBatches Node to Loop Over Customers**  
   - Type: SplitInBatches  
   - Default batch size 1.  
   - Connect “true” output from If node.  

6. **Add Langchain Chain LLM Node for Offer Generation**  
   - Type: Langchain Chain LLM  
   - Enter prompt that includes rules for churn score intervals and output JSON structure.  
   - Use expressions to insert `predicted_churn_score` and `preferred_categories` from batch customer data.  
   - Connect input from SplitInBatches node.  

7. **Add Langchain Chat Model Node (Google Gemini)**  
   - Type: Langchain Chat Model  
   - Select Google Gemini credentials (Google Vertex AI API OAuth2).  
   - Model: models/gemini-2.0-pro-exp.  
   - Connect input from Chain LLM node.  

8. **Add Langchain Output Parser (Structured)**  
   - Type: Output Parser Structured  
   - Paste JSON schema example for expected output.  
   - Connect input from Gemini Chat Model node.  

9. **Add Google Sheets Node to Log Sent Offer**  
   - Type: Google Sheets (Append/Update)  
   - Configure with same Google Sheets OAuth2 credentials.  
   - Spreadsheet ID and Sheet Name for ‘SYSTEM_LOG’.  
   - Map columns: `system_log` = `action_taken`, `date` = `timestamp`, `customer_id` = `customer_id` from parsed JSON.  
   - Connect input from Output Parser node.  

10. **Add Gmail Node to Send Email**  
    - Type: Gmail  
    - Configure with Gmail OAuth2 credentials.  
    - Set ‘Send To’ as customer email: `{{ $json.user_mail }}` from SplitInBatches input (ensure original data is accessible).  
    - Subject: `{{ $json.output.offer_title }}` from parsed JSON.  
    - Message body: `{{ $json.output.offer_details }}` as plain text.  
    - Connect input from Log Sent Offer node.  

11. **Connect Gmail Node Output Back to SplitInBatches Node**  
    - To iterate next customer until done.  

12. **Add Set Node to Handle No Eligible Customers**  
    - Type: Set  
    - Assign: `system_log` = “NOT_FOUND”  
    - Assign: `date` = `{{$now}}` (current ISO timestamp).  
    - Connect input from “false” branch of If node.  

13. **Add Google Sheets Node to Log ‘Not Found’ Status**  
    - Type: Google Sheets (Append/Update)  
    - Configure with SYSTEM_LOG sheet same as before.  
    - Map columns: `system_log` and `date` from Set node.  
    - Connect input from Set node.  

14. **Activate Workflow**  
    - Confirm all credentials and parameters are correctly set.  
    - Test run manually or wait for scheduled trigger.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                               | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Verify the `created_campaign_date` logic in the Filter node carefully; adjust as needed to suit your campaign strategy (e.g., avoid sending to recently contacted customers). | Important for correct customer targeting.                                                                       |
| The personalized offer messages are generated in Turkish based on the prompt; adapt the prompt language and offer logic if needed.                                        | Customization of AI behavior via prompt.                                                                        |
| Google Gemini AI requires Google Cloud Vertex AI API enabled and correct OAuth2 credentials setup in n8n.                                                                  | Ensure Google Cloud project is properly configured.                                                             |
| Gmail node requires OAuth2 credentials with send email permission; ensure correct Gmail account and scopes are authorized.                                               | Gmail API integration.                                                                                           |
| The workflow logs all actions in a ‘SYSTEM_LOG’ Google Sheet to allow tracking of campaign effectiveness and troubleshooting.                                            | Enables audit trail for marketing campaigns.                                                                    |
| Example customer data format provided in Sticky Note for reference to expected fields and types.                                                                           | Data format for ‘Customer Data’ sheet, critical for correct workflow function.                                  |
| For large customer datasets, consider tuning batch sizes or scaling workflow execution to avoid API rate limits or timeouts.                                             | Performance optimization and scaling considerations.                                                            |

---

This documentation provides a detailed and structured reference enabling users and AI agents to fully understand, reproduce, and maintain the Automated Daily Customer Win-Back Campaign workflow with AI offers in n8n.