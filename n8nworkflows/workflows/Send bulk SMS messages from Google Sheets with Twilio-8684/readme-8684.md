Send bulk SMS messages from Google Sheets with Twilio

https://n8nworkflows.xyz/workflows/send-bulk-sms-messages-from-google-sheets-with-twilio-8684


# Send bulk SMS messages from Google Sheets with Twilio

### 1. Workflow Overview

This workflow automates sending personalized bulk SMS messages from data stored in a Google Sheet. It is designed for scenarios such as event reminders, bulk notifications, or any use case requiring customized SMS outreach to multiple recipients.

The logical blocks of the workflow are:

- **1.1 Google Sheets Monitoring and Input Reception:** Watches a Google Sheet for rows marked with a status "To send" to trigger message sending.
- **1.2 Configuration Setup:** Loads configurable parameters such as the Google Sheet URL and Twilio sending number.
- **1.3 Data Preparation and Personalization:** Combines configuration with row data and prepares personalized SMS content by replacing placeholders in message templates.
- **1.4 SMS Sending via Twilio:** Sends the personalized SMS message through Twilio’s API.
- **1.5 Status Update and Error Handling:** Updates the Google Sheet row status to reflect sending progress ("Sending..."), success ("Success"), or failure ("Error").

---

### 2. Block-by-Block Analysis

#### 2.1 Google Sheets Monitoring and Input Reception

- **Overview:**  
  Continuously monitors the specified Google Sheet for any rows where the "Status" column changes, specifically looking for rows that have the status "To send" to initiate the sending process.

- **Nodes Involved:**  
  - Monitor Google Sheet for SMS Queue

- **Node Details:**

  - **Monitor Google Sheet for SMS Queue**  
    - Type: Google Sheets Trigger  
    - Role: Watches the Google Sheet for changes in the "Status" column every minute.  
    - Configuration:  
      - Sheet URL and Document ID set to the target Google Sheet (editable in Config node).  
      - Columns to watch: ["Status"]  
      - Poll interval: Every minute  
    - Inputs: None (trigger node)  
    - Outputs: Emits rows where the "Status" column has updated values  
    - Potential Failures:  
      - Google authentication errors or expired OAuth tokens  
      - API rate limits or connectivity issues with Google Sheets  
      - Missing or incorrect sheet URL/document ID configuration  
    - Version: v1  

---

#### 2.2 Configuration Setup

- **Overview:**  
  Sets workflow-wide configuration parameters such as the Google Sheet URL and the Twilio sending phone number to be used in subsequent nodes.

- **Nodes Involved:**  
  - Config  
  - Configuration Helper (Sticky Note)

- **Node Details:**

  - **Config**  
    - Type: Set  
    - Role: Defines two key configuration variables:  
      - `sheet_url`: URL of the Google Sheet to monitor and update  
      - `from_number`: Twilio phone number from which SMS messages will be sent  
    - Configuration: Values are hardcoded strings, editable by users for their own deployment  
    - Inputs: Receives data from the Google Sheets Trigger node  
    - Outputs: Passes configuration data downstream for merging with row data  
    - Version: v3.4  

  - **Configuration Helper**  
    - Type: Sticky Note  
    - Role: Provides user instructions to set the Google Sheet URL and Twilio number in the Config node  
    - Inputs/Outputs: None  
    - Position: Near Config node as a visual reminder  

---

#### 2.3 Data Preparation and Personalization

- **Overview:**  
  Combines the dynamic row data from the Google Sheet with the static configuration data, then personalizes each SMS message by replacing placeholders with actual recipient data.

- **Nodes Involved:**  
  - Check if Ready to Send  
  - Update Status to Sending  
  - Merge Config with Row Data  
  - Prepare SMS Content  

- **Node Details:**

  - **Check if Ready to Send**  
    - Type: If  
    - Role: Filters incoming rows to process only those with "Status" exactly equal to "To send"  
    - Configuration: Condition checks if `Status == "To send"`  
    - Inputs: From Config node  
    - Outputs: Passes rows matching condition forward  
    - Potential Failures: Expression failures if field missing or null  
    - Version: v2.2  

  - **Update Status to Sending**  
    - Type: Google Sheets  
    - Role: Updates the row’s "Status" column to "Sending..." to indicate processing has started  
    - Configuration:  
      - Updates row by matching on unique "ID" field  
      - Uses sheet URL from merged configuration  
    - Inputs: From "Check if Ready to Send" (true branch)  
    - Outputs: Passes updated row data downstream  
    - Potential Failures: Google Sheets API errors, sheet permissions, ID matching failures  
    - Version: v4.7  

  - **Merge Config with Row Data**  
    - Type: Merge  
    - Role: Combines configuration fields (`sheet_url`, `from_number`) with the current row’s data to prepare a full dataset for message sending  
    - Configuration: Mode "combine by position" (combines corresponding items in order)  
    - Inputs: Two inputs:  
      - From "Check if Ready to Send" (row data)  
      - From "Update Status to Sending" (config data)  
    - Outputs: Single combined output for message preparation  
    - Version: v3.2  

  - **Prepare SMS Content**  
    - Type: Set  
    - Role: Constructs the SMS message text by replacing placeholders `[First Name]` and `[Last Name]` in the "Message Template" column with actual recipient names; also sets the SMS "To" and "From" phone numbers.  
    - Configuration:  
      - Expression-based replacements using JavaScript `.replace()` on the template string  
      - Sets fields:  
        - `Message`: personalized message text  
        - `To`: recipient phone number from row data  
        - `From`: Twilio sending number from config  
    - Inputs: Merged data from previous node  
    - Outputs: Prepared message data for sending via Twilio  
    - Potential Failures: Missing placeholders or null fields may cause blank messages or runtime errors  
    - Version: v3.4  

---

#### 2.4 SMS Sending via Twilio

- **Overview:**  
  Sends the prepared SMS message using Twilio’s API and continues workflow execution even if sending errors occur.

- **Nodes Involved:**  
  - Send SMS via Twilio  

- **Node Details:**

  - **Send SMS via Twilio**  
    - Type: Twilio  
    - Role: Delivers the SMS message using Twilio credentials  
    - Configuration:  
      - `To`: recipient phone number (from previous node)  
      - `From`: Twilio phone number configured earlier  
      - `Message`: personalized SMS text  
      - On error: configured to continue regular output to allow error handling downstream  
    - Credentials: Uses Twilio API credentials (Account SID and Auth Token) configured in n8n  
    - Inputs: From Prepare SMS Content node  
    - Outputs: Response from Twilio API, including success or error information  
    - Potential Failures:  
      - Twilio API authentication errors or expired credentials  
      - Invalid phone numbers or message formatting errors  
      - Network issues or API rate limiting  
    - Version: v1  

---

#### 2.5 Status Update and Error Handling

- **Overview:**  
  Checks the result of the SMS sending operation, then updates the Google Sheet row status to "Success" if sent successfully or "Error" if any failure occurred.

- **Nodes Involved:**  
  - Check Send Result  
  - Mark as Success  
  - Mark as Error  

- **Node Details:**

  - **Check Send Result**  
    - Type: If  
    - Role: Evaluates if the Twilio node returned an error or not by checking the presence of `$json.error`  
    - Configuration: Condition passes if no error field exists (`!$json.error`)  
    - Inputs: From Send SMS via Twilio node  
    - Outputs:  
      - True branch: no error → proceed to mark success  
      - False branch: error → mark failure  
    - Version: v2.2  

  - **Mark as Success**  
    - Type: Google Sheets  
    - Role: Updates the row "Status" column to "Success" indicating the SMS was sent successfully  
    - Configuration:  
      - Matches row by "ID" from the Update Status to Sending node’s data  
      - Uses sheet URL from Config node  
      - Operation: Update  
    - Inputs: True output from Check Send Result  
    - Outputs: None  
    - Potential Failures: Google Sheets API errors, permission issues, row matching failures  
    - Version: v4.7  

  - **Mark as Error**  
    - Type: Google Sheets  
    - Role: Updates the row "Status" column to "Error" indicating a failure in sending SMS  
    - Configuration:  
      - Same as Mark as Success but sets "Status" to "Error"  
    - Inputs: False output from Check Send Result  
    - Outputs: None  
    - Potential Failures: Same as Mark as Success  
    - Version: v4.7  

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                                  | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                                      |
|--------------------------------|---------------------|-------------------------------------------------|------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Monitor Google Sheet for SMS Queue | Google Sheets Trigger | Watches Google Sheet for rows to send SMS       | None                         | Config                       |                                                                                                                                 |
| Config                         | Set                 | Defines sheet URL and Twilio sending number     | Monitor Google Sheet for SMS Queue | Check if Ready to Send        | ## Configure here 👇👇 Set your Google Sheet URL and Twilio phone number in the Config node below                                |
| Check if Ready to Send          | If                  | Filters rows with Status == "To send"            | Config                       | Update Status to Sending, Merge Config with Row Data |                                                                                                                                 |
| Update Status to Sending        | Google Sheets       | Marks row as "Sending..." in Google Sheet        | Check if Ready to Send        | Merge Config with Row Data    |                                                                                                                                 |
| Merge Config with Row Data      | Merge               | Combines config data with row data                | Check if Ready to Send, Update Status to Sending | Prepare SMS Content           |                                                                                                                                 |
| Prepare SMS Content             | Set                 | Personalizes message, sets To/From phone numbers | Merge Config with Row Data    | Send SMS via Twilio           |                                                                                                                                 |
| Send SMS via Twilio             | Twilio              | Sends SMS message via Twilio API                  | Prepare SMS Content           | Check Send Result             |                                                                                                                                 |
| Check Send Result               | If                  | Checks if SMS was sent successfully or errored   | Send SMS via Twilio           | Mark as Success, Mark as Error |                                                                                                                                 |
| Mark as Success                | Google Sheets       | Updates row status to "Success"                   | Check Send Result (true)       | None                         |                                                                                                                                 |
| Mark as Error                  | Google Sheets       | Updates row status to "Error"                     | Check Send Result (false)      | None                         |                                                                                                                                 |
| Configuration Helper           | Sticky Note         | Instruction to configure Google Sheet URL, Twilio number | None                         | None                         | ## Configure here 👇👇 Set your Google Sheet URL and Twilio phone number in the Config node below                                |
| Workflow Description           | Sticky Note         | Overview and setup instructions                   | None                         | None                         | ## Workflow Overview This workflow automates sending personalized SMS messages directly from a Google Sheet. See detailed setup instructions in this note. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a **Google Sheets Trigger** node.  
   - Set it to watch the target Google Sheet by URL and document ID.  
   - Configure to watch the "Status" column only.  
   - Set polling frequency to every minute.  
   - Connect Google Sheets OAuth2 credentials for access.

2. **Set Configuration Node:**  
   - Add a **Set** node, named "Config".  
   - Define two string fields:  
     - `sheet_url`: Your Google Sheet URL (must be your own copy).  
     - `from_number`: Your Twilio phone number (e.g., "+1234567890").  
   - Connect output of Google Sheets Trigger to this node.

3. **Add Condition to Filter Rows:**  
   - Add an **If** node named "Check if Ready to Send".  
   - Set condition: `$json.Status == "To send"`.  
   - Connect the "Config" node output to this node.

4. **Update Row Status to "Sending...":**  
   - Add a **Google Sheets** node named "Update Status to Sending".  
   - Configure to update the row matching on "ID" with "Status" = "Sending...".  
   - Use the `sheet_url` from Config node for document and sheet reference.  
   - Connect the "true" branch output of "Check if Ready to Send" to this node.

5. **Merge Config and Row Data:**  
   - Add a **Merge** node named "Merge Config with Row Data".  
   - Set mode to "combine by position".  
   - Connect output 0 of "Check if Ready to Send" (row data) to input 1 of this node.  
   - Connect output of "Update Status to Sending" to input 2 of this node.

6. **Prepare the SMS Content:**  
   - Add a **Set** node named "Prepare SMS Content".  
   - Define fields:  
     - `Message`: Use expression to replace `[First Name]` and `[Last Name]` in "Message Template" with actual values from the row.  
     - `To`: Set to recipient phone number from row data.  
     - `From`: Set to `from_number` from config.  
   - Connect output of "Merge Config with Row Data" here.

7. **Send SMS via Twilio:**  
   - Add a **Twilio** node named "Send SMS via Twilio".  
   - Configure with your Twilio credentials (Account SID, Auth Token).  
   - Set:  
     - `To` = `{{$json.To}}`  
     - `From` = `{{$json.From}}`  
     - `Message` = `{{$json.Message}}`  
   - Set error handling to "Continue on Error".  
   - Connect output of "Prepare SMS Content" here.

8. **Check Send Result:**  
   - Add an **If** node named "Check Send Result".  
   - Condition: No error in response → `$json.error` does not exist or is false.  
   - Connect output of Twilio node here.

9. **Update Row Status to "Success":**  
   - Add a **Google Sheets** node named "Mark as Success".  
   - Configure to update row by "ID" setting "Status" = "Success".  
   - Use `sheet_url` from Config node.  
   - Connect "true" branch of "Check Send Result" here.

10. **Update Row Status to "Error":**  
    - Add a **Google Sheets** node named "Mark as Error".  
    - Configure similarly to "Mark as Success" but set "Status" = "Error".  
    - Connect "false" branch of "Check Send Result" here.

11. **Add Sticky Notes for User Guidance:**  
    - Add sticky notes near Config and start nodes to instruct users on setup steps, including copying the Google Sheet template, setting credentials, and configuring parameters.

12. **Credential Setup:**  
    - Add and configure Google Sheets OAuth2 credentials for all Google Sheets nodes.  
    - Add and configure Twilio credentials (Account SID and Auth Token) for the Twilio node.

13. **Test the Workflow:**  
    - Populate the Google Sheet with test data, including "Status" column set to "To send".  
    - Ensure personalized placeholders `[First Name]`, `[Last Name]` and phone numbers are properly filled.  
    - Activate the workflow and verify messages are sent and statuses update accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow is ideal for sending personalized SMS notifications triggered from spreadsheet data changes, such as event reminders or bulk marketing messages. Ensure you use your own copy of the Google Sheet template for proper update permissions.                                                                                                                                                                                                                                                                                                                                  | https://docs.google.com/spreadsheets/d/1DGhQ2YLeQ5boLYPMK4nUF8SIOJmDDqvtyleSz5IpJEc/edit?usp=sharing            |
| To use Twilio, sign up for a free trial account at https://www.twilio.com/try-twilio and obtain your Account SID, Auth Token, and Twilio phone number. Add these credentials securely in n8n.                                                                                                                                                                                                                                                                                                                                                                                                | https://www.twilio.com/try-twilio                                                                                |
| The workflow polls Google Sheets every minute, so sending latency depends on this interval. Adjust polling frequency if needed, considering Google API quotas.                                                                                                                                                                                                                                                                                                                                                                                                                             | Google Sheets API documentation                                                                                  |
| Placeholder support in message templates: `[First Name]` and `[Last Name]` will be replaced dynamically, so ensure your spreadsheet columns are correctly named and populated. Other placeholders can be added by extending the Set node expressions accordingly.                                                                                                                                                                                                                                                                                                                            |                                                                                                                 |
| If errors occur in SMS sending, the workflow marks the row status as "Error" but continues processing other rows, ensuring robustness in bulk sending scenarios.                                                                                                                                                                                                                                                                                                                                                                                                                             |                                                                                                                 |

---

**Disclaimer:** The provided content is derived exclusively from an automated workflow created with n8n, respecting all current content policies. It contains no illegal, offensive, or protected elements. All processed data is legal and public.