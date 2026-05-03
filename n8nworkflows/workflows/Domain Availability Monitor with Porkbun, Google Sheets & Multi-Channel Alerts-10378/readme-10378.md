Domain Availability Monitor with Porkbun, Google Sheets & Multi-Channel Alerts

https://n8nworkflows.xyz/workflows/domain-availability-monitor-with-porkbun--google-sheets---multi-channel-alerts-10378


# Domain Availability Monitor with Porkbun, Google Sheets & Multi-Channel Alerts

### 1. Workflow Overview

This workflow automates the monitoring of domain name availability using the Porkbun API, integrates with Google Sheets for domain management, and sends multi-channel alerts when a domain becomes available. The primary use case is domain sniping—tracking desired domains and promptly notifying users when they are ready for registration. 

The workflow consists of the following logical blocks:

- **1.1 Scheduled Trigger & Input Retrieval**: Periodically triggers the workflow every 30 minutes to retrieve a filtered list of domains from a Google Sheet that are currently marked as unavailable.
- **1.2 Domain Availability Checking Loop**: Processes each domain individually by querying the Porkbun API to check availability.
- **1.3 Conditional Handling & Alerts**: If a domain is available, triggers multiple alert channels (email via Gmail, Discord notification) and updates the domain status in Google Sheets.
- **1.4 Throttling & Flow Control**: Inserts wait steps to prevent API rate limits or flooding alerts.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Input Retrieval

- **Overview:**  
Initiates the workflow every 30 minutes and retrieves the list of domains to monitor from a Google Sheet. Filters out domains already marked as available.

- **Nodes Involved:**  
  - Check every 30 Minutes  
  - Get Domains from Sheet

- **Node Details:**  

  - **Check every 30 Minutes**  
    - Type: Schedule Trigger  
    - Configuration: Interval set to trigger every 30 minutes.  
    - Inputs: None (trigger node).  
    - Outputs: Starts workflow by triggering "Get Domains from Sheet".  
    - Failure Modes: None typical; if n8n’s scheduling engine fails, the workflow might not trigger.  
    - Version: v1.

  - **Get Domains from Sheet**  
    - Type: Google Sheets  
    - Configuration: Reads from a Google Sheet (specified by document ID and Sheet1). Filters to only fetch rows where `isAvailable` is "no".  
    - Input: Trigger from the Schedule node.  
    - Output: Emits array of domain records with `Domain` and `isAvailable` fields.  
    - Credentials: Requires Google Sheets OAuth2 credentials.  
    - Failure Modes: API quota exceeded, invalid credentials, network errors.  
    - Version: 4.7.

---

#### 2.2 Domain Availability Checking Loop

- **Overview:**  
Splits the list of domains into batches (single domains here) and checks the availability of each domain individually via the Porkbun API.

- **Nodes Involved:**  
  - Process Each Domain  
  - Check Domain Availability  
  - Validate API KEY (support node, optional)  
  - Sticky Note (instructional)

- **Node Details:**

  - **Process Each Domain**  
    - Type: Split In Batches  
    - Configuration: Processes input items one by one (default batch size).  
    - Input: Domains array from "Get Domains from Sheet".  
    - Output: Individual domain items passed downstream for availability check.  
    - Failure Modes: No items input results in no output.  
    - Version: 3.

  - **Check Domain Availability**  
    - Type: HTTP Request  
    - Configuration: POST request to Porkbun API endpoint `/domain/checkDomain/{{ $json.Domain }}` with API and secret keys passed as form-urlencoded body parameters.  
    - Input: Single domain JSON item from "Process Each Domain".  
    - Output: JSON response containing availability status, pricing, premium flag, and renewal/transfer fees.  
    - Key expressions: URL dynamically constructed with domain name.  
    - Credentials: Requires Porkbun API Key and Secret API Key.  
    - Failure Modes: Authentication errors (invalid keys), network/timeout errors, API limits, incorrect domain format.  
    - Version: 4.2.

  - **Validate API KEY** (optional node)  
    - Type: HTTP Request  
    - Configuration: POST request to Porkbun API `/ping` endpoint to verify API keys’ validity.  
    - Input: N/A (could be used manually or as a test step).  
    - Output: API response confirming connectivity and key validity.  
    - Failure Modes: Invalid or expired API keys, network errors.  
    - Version: 4.2.

  - **Sticky Notes**  
    - Provide guidance on obtaining Porkbun API keys and validating them. No direct execution role.

---

#### 2.3 Conditional Handling & Alerts

- **Overview:**  
Evaluates the domain availability result. If available, triggers alert notifications via Email (Gmail) and Discord, then updates the Google Sheet status to "yes". If not available, waits before processing next domain.

- **Nodes Involved:**  
  - Domain Available? (If)  
  - Send Email Alert  
  - Send Discord Notification  
  - Update Sheet: Mark Available  
  - Wait 10 Seconds

- **Node Details:**

  - **Domain Available?**  
    - Type: If  
    - Configuration: Checks if the Porkbun API response field `response.avail` equals `"yes"`.  
    - Input: API response from "Check Domain Availability".  
    - Output: Two branches — true (available) and false (not available).  
    - Failure Modes: Missing or malformed response data can cause expression errors.  
    - Version: 2.

  - **Send Email Alert**  
    - Type: Gmail  
    - Configuration: Sends an HTML formatted email alerting that the domain is available, including price details, premium status, renewal and transfer fees, with a call-to-action button linking to Porkbun checkout. Subject includes the domain name dynamically.  
    - Input: True branch from "Domain Available?".  
    - Credentials: Gmail OAuth2 credentials configured.  
    - Failure Modes: Authentication errors, quota limits, malformed email template errors.  
    - Version: 2.1.

  - **Send Discord Notification**  
    - Type: Discord  
    - Configuration: Sends a message to a specified Discord channel using a bot. Message contains domain availability info and pricing details dynamically inserted.  
    - Input: After "Send Email Alert".  
    - Credentials: Discord Bot API credentials required.  
    - Failure Modes: Invalid webhook or bot token, channel permissions, rate limits.  
    - Version: 2.

  - **Update Sheet: Mark Available**  
    - Type: Google Sheets  
    - Configuration: Updates the corresponding domain row in Google Sheets, setting `isAvailable` to `"yes"` to mark it as available. Uses the `Domain` column as the matching key.  
    - Input: After "Send Discord Notification".  
    - Credentials: Same Google Sheets OAuth2 credentials as before.  
    - Failure Modes: API quota exceeded, permission denied, sheet modifications conflicts.  
    - Version: 4.7.

  - **Wait 10 Seconds**  
    - Type: Wait  
    - Configuration: Pauses workflow execution for 10 seconds before processing the next domain.  
    - Input: False branch from "Domain Available?" and after "Update Sheet: Mark Available".  
    - Purpose: Avoids hitting rate limits or flooding alerts.  
    - Version: 1.1.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                           | Input Node(s)            | Output Node(s)                  | Sticky Note                                                                                           |
|--------------------------|--------------------|-----------------------------------------|--------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------|
| Sticky Note1             | Sticky Note        | Instruction to obtain Porkbun API keys  | None                     | None                          | "Get Porkbun API Keys: 1. Login... 5. Save both keys immediately!"                                  |
| Validate API KEY         | HTTP Request       | Optional test to validate Porkbun API keys | None                     | None                          |                                                                                                     |
| Sticky Note              | Sticky Note        | Instruction label for API key validation| None                     | None                          | "Validate API Key"                                                                                   |
| Check every 30 Minutes   | Schedule Trigger   | Triggers workflow every 30 minutes      | None                     | Get Domains from Sheet         |                                                                                                     |
| Get Domains from Sheet   | Google Sheets      | Fetch domains where `isAvailable`=no    | Check every 30 Minutes   | Process Each Domain            |                                                                                                     |
| Process Each Domain      | Split In Batches   | Processes domains one by one             | Get Domains from Sheet   | Check Domain Availability      |                                                                                                     |
| Check Domain Availability| HTTP Request       | Queries Porkbun API for domain availability | Process Each Domain      | Domain Available?              |                                                                                                     |
| Domain Available?        | If                 | Checks if domain is available            | Check Domain Availability| Send Email Alert (true), Wait 10 Seconds (false) |                                                                                                     |
| Send Email Alert         | Gmail              | Sends email notification when domain available | Domain Available? (true) | Send Discord Notification      |                                                                                                     |
| Send Discord Notification| Discord            | Sends Discord message for available domain | Send Email Alert          | Update Sheet: Mark Available   |                                                                                                     |
| Update Sheet: Mark Available | Google Sheets  | Updates domain status to "yes" in sheet | Send Discord Notification | Wait 10 Seconds               |                                                                                                     |
| Wait 10 Seconds          | Wait               | Pauses workflow for 10 seconds           | Domain Available? (false), Update Sheet: Mark Available | Process Each Domain         |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger every 30 minutes.

2. **Add Google Sheets Node: Get Domains from Sheet**  
   - Connect from Schedule Trigger.  
   - Configure credentials for Google Sheets OAuth2.  
   - Set operation to "Read Rows".  
   - Specify the Google Sheet document ID and Sheet1 (gid=0).  
   - Add filter: `isAvailable` equals "no" to only fetch domains not yet available.

3. **Add Split In Batches Node: Process Each Domain**  
   - Connect from "Get Domains from Sheet".  
   - Use default batch size (1) to process domains one at a time.

4. **Add HTTP Request Node: Check Domain Availability**  
   - Connect from "Process Each Domain".  
   - Configure POST request to `https://api.porkbun.com/api/json/v3/domain/checkDomain/{{ $json.Domain }}`.  
   - Set content type to form-urlencoded.  
   - Add body parameters:  
     - `secretapikey`: Your Porkbun Secret API Key  
     - `apikey`: Your Porkbun API Key  
   - Set credentials accordingly or enter keys manually.  
   - Ensure dynamic URL uses expression for Domain field.

5. **Add If Node: Domain Available?**  
   - Connect from "Check Domain Availability".  
   - Condition: Check if `response.avail` equals string `"yes"`.  
   - True branch: domain available.  
   - False branch: domain not available.

6. **Add Gmail Node: Send Email Alert**  
   - Connect from "Domain Available?" true branch.  
   - Configure Gmail OAuth2 credentials.  
   - Set email recipient.  
   - Compose HTML message with dynamic insertion of domain name, prices, and premium status using expressions referencing "Process Each Domain" and API response fields.  
   - Set email subject dynamically with domain name.

7. **Add Discord Node: Send Discord Notification**  
   - Connect from "Send Email Alert".  
   - Configure Discord Bot API credentials.  
   - Set target guild ID and channel ID.  
   - Compose message with domain details dynamically inserted.

8. **Add Google Sheets Node: Update Sheet: Mark Available**  
   - Connect from "Send Discord Notification".  
   - Configure with same Google Sheets credentials.  
   - Set operation to "Update".  
   - Match rows by `Domain` column.  
   - Update `isAvailable` column to "yes".

9. **Add Wait Node: Wait 10 Seconds**  
   - Connect from "Update Sheet: Mark Available" and from "Domain Available?" false branch.  
   - Set wait duration to 10 seconds to avoid API throttling and flooding notifications.

10. **Connect Wait Node back to "Process Each Domain"** to continue processing next domain.

11. **(Optional) Add HTTP Request Node: Validate API KEY**  
    - For manual API key validation, POST to `https://api.porkbun.com/api/json/v3/ping` with API keys.  
    - Not connected in main flow but useful for setup verification.

12. **Add Sticky Notes for documentation**  
    - Add notes on how to obtain Porkbun API keys and validate them.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                              | Context or Link                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| To obtain Porkbun API keys: Login to https://porkbun.com/account/api, create API Key, copy both API Key and Secret API Key, and save securely.                             | Sticky Note1 instructions                        |
| Validate API keys by POST request to https://api.porkbun.com/api/json/v3/ping endpoint before running availability checks.                                                | Sticky Note instruction                          |
| Gmail OAuth2 and Discord Bot API credentials must be properly configured with permissions to send emails and messages respectively.                                        | Credential setup notes                           |
| This workflow relies on Google Sheets as the domain list database; ensure the sheet has columns "Domain" and "isAvailable" and that the Google API project has Sheets API enabled. | Google Sheets integration                        |
| Domain availability status updates in the sheet prevent redundant checks on available domains, optimizing API usage.                                                      | Workflow design rationale                        |
| The workflow includes a 10-second wait after each domain check to mitigate API rate limits and avoid notification flooding.                                               | Throttling design                               |
| Discord message format requires the bot to have permissions in the specified guild and channel.                                                                            | Discord bot permissions                          |
| Porkbun API documentation: https://porkbun.com/api/json/v3/documentation                                                                                                  | Official Porkbun API docs                        |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. The processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.