Real-Time Lead Enrichment in Slack with Extruct AI In-Thread Reply

https://n8nworkflows.xyz/workflows/real-time-lead-enrichment-in-slack-with-extruct-ai-in-thread-reply-5901


# Real-Time Lead Enrichment in Slack with Extruct AI In-Thread Reply

---

### 1. Workflow Overview

This workflow automates **real-time lead enrichment** within Slack channels using the Extruct AI service. Its primary purpose is to monitor Slack messages for potential lead email addresses or company domains, extract relevant company identifiers, enrich these with detailed company data via the Extruct API, and reply directly in the Slack thread with a structured company profile card.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Initialization:** Watches Slack for new messages and sets the Extruct table ID required for enrichment.
- **1.2 Lead Extraction:** Parses incoming Slack messages to extract company domains or names, filtering out generic or invalid domains.
- **1.3 Enrichment Request Flow:** Sends the extracted domain to the Extruct API to start the enrichment process and polls the API until enrichment is complete.
- **1.4 Data Retrieval & Formatting:** Fetches the enriched company data and formats it into a Slack-friendly card.
- **1.5 Slack Publishing:** Posts the formatted company profile back into the original Slack thread as a reply.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Initialization

- **Overview:**  
Monitors a specified Slack channel for new messages and initializes the workflow by setting the Extruct table ID for later API calls.

- **Nodes Involved:**  
  - New Message Catcher  
  - Set Extruct Table ID

- **Node Details:**

  - **New Message Catcher**  
    - Type: Slack Trigger  
    - Role: Listens for new messages in Slack channels to trigger the workflow.  
    - Configuration:  
      - Trigger on "message" events.  
      - Channel ID configured dynamically (empty by default, set via credential or UI).  
      - Uses Slack OAuth2 credentials with bot token and required scopes (`channels:read`, `channels:history`, `chat:write`).  
    - Inputs: Webhook from Slack event subscription.  
    - Outputs: Emits message JSON data including text, timestamps, thread IDs.  
    - Edge Cases: Could miss messages if Slack OAuth scopes or event subscription are misconfigured; potential webhook downtime or rate limits.

  - **Set Extruct Table ID**  
    - Type: Set Node  
    - Role: Assigns a static Extruct API table ID (user-provided) for enrichment requests.  
    - Configuration:  
      - Sets a single string variable `EXTRUCT_TABLE_ID` with the user’s Extruct table ID.  
      - User must replace `"YOUR_EXTRUCT_TABLE_ID"` with their actual ID.  
    - Inputs: Triggered by New Message Catcher node.  
    - Outputs: Passes forward the table ID as JSON data.  
    - Edge Cases: Missing or invalid table ID will cause API calls to fail downstream.

---

#### 2.2 Lead Extraction

- **Overview:**  
Extracts a company domain from Slack message text by searching for email addresses or domain patterns, excluding generic email providers.

- **Nodes Involved:**  
  - Extract Company Name Input  
  - Company Name Exists?

- **Node Details:**

  - **Extract Company Name Input**  
    - Type: Code Node (JavaScript)  
    - Role: Parses Slack message text to extract and return the company domain.  
    - Configuration:  
      - Ignores Slack thread replies to avoid duplicate processing.  
      - Uses regex to find an email domain or domain pattern in the message text.  
      - Filters out generic/common email domains such as gmail.com, outlook.com, yahoo.com, etc.  
      - Returns JSON with `companyDomain` if valid domain found; otherwise returns null to filter out.  
    - Inputs: Receives JSON message data from Set Extruct Table ID node.  
    - Outputs: Emits items only if a valid company domain is extracted.  
    - Edge Cases: Messages without domain or emails, or only generic domains, produce no output; malformed text or regex failures are unlikely but possible.

  - **Company Name Exists?**  
    - Type: If Node  
    - Role: Checks if the `companyDomain` field exists and is non-empty.  
    - Configuration:  
      - Condition: `companyDomain` is not empty (strict check).  
    - Inputs: From Extract Company Name Input node.  
    - Outputs:  
      - True branch proceeds to enrichment request.  
      - False branch ends workflow (no domain to enrich).  
    - Edge Cases: If code node outputs unexpected data, condition may fail and skip enrichment.

---

#### 2.3 Enrichment Request Flow

- **Overview:**  
Sends the extracted company domain to Extruct API to initiate enrichment, then waits and polls periodically until the enrichment is complete.

- **Nodes Involved:**  
  - Start Company Enrichment  
  - Hold for API Processing  
  - Check Enrichment Status  
  - Is Company Info Ready?

- **Node Details:**

  - **Start Company Enrichment**  
    - Type: HTTP Request  
    - Role: POST request to Extruct API to add a row with the extracted company domain and start enrichment.  
    - Configuration:  
      - URL constructed dynamically: `https://api.extruct.ai/v1/tables/{EXTRUCT_TABLE_ID}/rows`  
      - Body: JSON containing the domain in the "input" field under "rows".  
      - Auth: Bearer token via Generic Credential node holding the Extruct API token.  
      - Sends JSON body with "run": true to immediately start processing.  
    - Inputs: From Company Name Exists? (true branch).  
    - Outputs: API response indicating acceptance of the enrichment request.  
    - Edge Cases: API errors, authentication failure, invalid table ID, network issues.

  - **Hold for API Processing**  
    - Type: Wait Node  
    - Role: Pauses workflow execution to allow Extruct API time to process enrichment.  
    - Configuration: Default wait (no duration specified explicitly in JSON) — likely short delay to avoid rapid polling.  
    - Inputs: From Start Company Enrichment or Is Company Info Ready? node (looping).  
    - Outputs: Proceeds to check enrichment status.  
    - Edge Cases: Excessive waiting or too short wait may cause premature polling or delays.

  - **Check Enrichment Status**  
    - Type: HTTP Request  
    - Role: GET request to fetch the status of the enrichment run.  
    - Configuration:  
      - URL: `https://api.extruct.ai/v1/tables/{EXTRUCT_TABLE_ID}`  
      - Auth: Bearer token via Generic Credential.  
    - Inputs: From Hold for API Processing node.  
    - Outputs: JSON containing enrichment `status.run_status`.  
    - Edge Cases: API down, auth failure, or malformed responses.

  - **Is Company Info Ready?**  
    - Type: If Node  
    - Role: Checks if enrichment run status equals `"running"`.  
    - Configuration:  
      - Condition: `status.run_status` == `"running"`  
    - Inputs: From Check Enrichment Status.  
    - Outputs:  
      - True branch loops back to Hold for API Processing (wait and re-check).  
      - False branch means enrichment is complete, proceeds to data retrieval.  
    - Edge Cases: API might return unexpected status, causing infinite loops or premature exits.

---

#### 2.4 Data Retrieval & Formatting

- **Overview:**  
Retrieves the enriched company data from Extruct API and formats it into a structured JSON object suitable for Slack message formatting.

- **Nodes Involved:**  
  - Get Company Data  
  - Format Slack Company Card

- **Node Details:**

  - **Get Company Data**  
    - Type: HTTP Request  
    - Role: GET request to retrieve enriched company data rows.  
    - Configuration:  
      - URL: `https://api.extruct.ai/v1/tables/{EXTRUCT_TABLE_ID}/data`  
      - Auth: Bearer token via Generic Credential.  
    - Inputs: From Is Company Info Ready? (false branch).  
    - Outputs: JSON containing array of data rows with enrichment answers.  
    - Edge Cases: No rows returned or empty data, API errors.

  - **Format Slack Company Card**  
    - Type: Code Node (JavaScript)  
    - Role: Extracts relevant fields from the last data row and maps them into Slack message variables.  
    - Configuration:  
      - Extracts: company_name, company_website, linkedin_profile, company_num_employees, company_industry, recent_news, key_people.  
      - Throws error if no rows found.  
      - Outputs JSON object with these fields for downstream Slack formatting.  
    - Inputs: From Get Company Data.  
    - Outputs: Structured JSON for Slack message template.  
    - Edge Cases: Missing fields or empty responses cause errors.

---

#### 2.5 Slack Publishing

- **Overview:**  
Posts a formatted Slack message with enriched company details back into the original Slack thread where the lead was posted.

- **Nodes Involved:**  
  - Send Company Details

- **Node Details:**

  - **Send Company Details**  
    - Type: Slack Node (chat.postMessage)  
    - Role: Sends a reply message in Slack with the enriched company profile card.  
    - Configuration:  
      - Message text includes company name, website (formatted as a link), LinkedIn link, number of employees, industry, recent news, and key people.  
      - Posts as a threaded reply using `thread_ts` matching the original message timestamp.  
      - Unfurl links disabled for cleaner appearance.  
      - Uses Slack OAuth2 credentials with appropriate scopes.  
      - Channel is set to the same channel monitored by the trigger node.  
    - Inputs: From Format Slack Company Card node.  
    - Outputs: Slack post confirmation.  
    - Edge Cases: Slack API rate limits, permissions, or invalid thread timestamp cause posting failure.

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                                  | Input Node(s)            | Output Node(s)               | Sticky Note                                                                                      |
|-------------------------|---------------------------|-------------------------------------------------|--------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| New Message Catcher      | Slack Trigger             | Trigger on new Slack messages                    | —                        | Set Extruct Table ID         | ## Trigger & Initialization: Monitors Slack for new messages and assigns the Extruct table ID. |
| Set Extruct Table ID     | Set                       | Assigns Extruct table ID                         | New Message Catcher       | Extract Company Name Input   | ## Trigger & Initialization: Monitors Slack for new messages and assigns the Extruct table ID. |
| Extract Company Name Input | Code                    | Extracts company domain from Slack message text | Set Extruct Table ID      | Company Name Exists?         | ## Lead Extraction: Parses each message for a company domain or name and filters out invalid entries. |
| Company Name Exists?     | If                        | Checks if extracted domain exists                | Extract Company Name Input| Start Company Enrichment (true branch) | ## Lead Extraction: Parses each message for a company domain or name and filters out invalid entries. |
| Start Company Enrichment | HTTP Request              | Sends domain to Extruct API to start enrichment | Company Name Exists?      | Hold for API Processing      | ## Enrichment Request Flow: Sends the domain to Extruct API and polls until the enrichment run completes. |
| Hold for API Processing  | Wait                      | Waits before polling enrichment status           | Start Company Enrichment, Is Company Info Ready? (true branch) | Check Enrichment Status | ## Enrichment Request Flow: Sends the domain to Extruct API and polls until the enrichment run completes. |
| Check Enrichment Status  | HTTP Request              | Checks if enrichment is complete                  | Hold for API Processing   | Is Company Info Ready?       | ## Enrichment Request Flow: Sends the domain to Extruct API and polls until the enrichment run completes. |
| Is Company Info Ready?   | If                        | Decides whether to continue waiting or proceed  | Check Enrichment Status   | Hold for API Processing (true), Get Company Data (false) | ## Enrichment Request Flow: Sends the domain to Extruct API and polls until the enrichment run completes. |
| Get Company Data         | HTTP Request              | Retrieves enriched company data                   | Is Company Info Ready? (false branch) | Format Slack Company Card | ## Data Retrieval & Formatting: Retrieves the enriched company details and builds a structured Slack card. |
| Format Slack Company Card| Code                      | Formats company data into Slack message variables| Get Company Data          | Send Company Details         | ## Data Retrieval & Formatting: Retrieves the enriched company details and builds a structured Slack card. |
| Send Company Details     | Slack                     | Posts formatted company profile in Slack thread | Format Slack Company Card | —                           | ## Slack Publishing: Posts the formatted company profile back into the original Slack thread.  |
| Sticky Note              | Sticky Note               | Quickstart instructions and overview             | —                        | —                           | Long detailed Quickstart guide and setup instructions with links and images.                   |
| Sticky Note1             | Sticky Note               | Trigger & Initialization block summary           | —                        | —                           | ## Trigger & Initialization: Monitors Slack for new messages and assigns the Extruct table ID. |
| Sticky Note2             | Sticky Note               | Lead Extraction block summary                      | —                        | —                           | ## Lead Extraction: Parses each message for a company domain or name and filters out invalid entries. |
| Sticky Note3             | Sticky Note               | Enrichment Request Flow block summary             | —                        | —                           | ## Enrichment Request Flow: Sends the domain to Extruct API and polls until the enrichment run completes. |
| Sticky Note4             | Sticky Note               | Data Retrieval & Formatting block summary          | —                        | —                           | ## Data Retrieval & Formatting: Retrieves the enriched company details and builds a structured Slack card. |
| Sticky Note5             | Sticky Note               | Slack Publishing block summary                      | —                        | —                           | ## Slack Publishing: Posts the formatted company profile back into the original Slack thread.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger Node (New Message Catcher):**  
   - Type: Slack Trigger  
   - Trigger on "message" events in the desired Slack channel.  
   - Configure Slack OAuth2 credentials with Bot Token (`xoxb-...`) having scopes: `channels:read`, `channels:history`, `chat:write`.  
   - Save the webhook URL generated for Slack Event Subscription.

2. **Configure Slack App:**  
   - Create Slack app with required scopes and event subscription to the webhook URL from step 1.  
   - Subscribe to `message.channels` events.  
   - Install and authorize the app to your workspace and channel.

3. **Add Set Node (Set Extruct Table ID):**  
   - Create a Set node named "Set Extruct Table ID".  
   - Add a string variable `EXTRUCT_TABLE_ID` with your Extruct table ID (copied from your Extruct dashboard).  
   - Connect "New Message Catcher" output to this node.

4. **Add Code Node (Extract Company Name Input):**  
   - Create a Code node with JavaScript code to:  
     - Ignore thread replies by checking `thread_ts` vs `ts`.  
     - Extract domain from email or domain pattern in message text.  
     - Filter out generic email domains (e.g., gmail.com).  
     - Output JSON `{ companyDomain: domain }` or null if invalid.  
   - Connect "Set Extruct Table ID" output to this node.

5. **Add If Node (Company Name Exists?):**  
   - Condition: Check if `companyDomain` is not empty.  
   - Connect "Extract Company Name Input" output to this node.

6. **Add HTTP Request Node (Start Company Enrichment):**  
   - Method: POST  
   - URL: `https://api.extruct.ai/v1/tables/{{ $json.EXTRUCT_TABLE_ID }}/rows`  
   - Body (JSON):  
     ```json
     {
       "rows": [
         {
           "data": {
             "input": "{{ $json.companyDomain }}"
           }
         }
       ],
       "run": true
     }
     ```  
   - Authentication: Bearer Token with your Extruct API key (Generic Credential).  
   - Connect "Company Name Exists?" true output to this node.

7. **Add Wait Node (Hold for API Processing):**  
   - Default wait time (optionally set a few seconds).  
   - Connect "Start Company Enrichment" output to this node.

8. **Add HTTP Request Node (Check Enrichment Status):**  
   - Method: GET  
   - URL: `https://api.extruct.ai/v1/tables/{{ $json.EXTRUCT_TABLE_ID }}`  
   - Authentication: Bearer Token with Extruct API key.  
   - Connect "Hold for API Processing" output to this node.

9. **Add If Node (Is Company Info Ready?):**  
   - Condition: Check if `status.run_status` equals `"running"`.  
   - Connect "Check Enrichment Status" output to this node.  
   - True output loops back to "Hold for API Processing" node.  
   - False output proceeds to next step.

10. **Add HTTP Request Node (Get Company Data):**  
    - Method: GET  
    - URL: `https://api.extruct.ai/v1/tables/{{ $json.EXTRUCT_TABLE_ID }}/data`  
    - Authentication: Bearer Token with Extruct API key.  
    - Connect "Is Company Info Ready?" false output to this node.

11. **Add Code Node (Format Slack Company Card):**  
    - JavaScript code to:  
      - Extract the last row’s data fields: company_name, website, linkedin, employees, industry, recent news, key people.  
      - Return a JSON object with these fields for Slack formatting.  
    - Connect "Get Company Data" output to this node.

12. **Add Slack Node (Send Company Details):**  
    - Action: Post message in Slack channel.  
    - Channel: Same Slack channel monitored.  
    - Message text with Slack markdown formatting referencing fields from "Format Slack Company Card" node, e.g.:  
      ```
      *Company Name:* {{$json.name}}
      *Website:* <{{$json.website}}|{{$json.website}}>
      *LinkedIn:* <{{$json.linkedin}}|{{$json.linkedin}}>
      *Number of Employees:* {{$json.employees}}
      *Industry:* {{$json.industry}}
      *Recent News:*
      {{$json.recentNews}}
      
      *Key People:*
      {{$json.keyPeople}}
      ```  
    - Threading: Use `thread_ts` from original Slack message to reply in thread.  
    - Disable link unfurling and workflow link inclusion for cleanliness.  
    - Use Slack OAuth2 credentials with appropriate scopes.  
    - Connect "Format Slack Company Card" output to this node.

13. **Activate the Workflow:**  
    - Ensure all credentials are configured correctly.  
    - Turn on the workflow in n8n editor.  
    - Invite your bot to the monitored Slack channel with `/invite @botname`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                | Context or Link                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| Automatic lead enrichment in Slack: monitors your Slack channel for new lead emails posted there, extracts each company’s name or domain, sends it to Extruct API for data enrichment, then posts back a structured Slack card with company details. | Workflow Purpose Overview                                                                |
| Free 1,000-credit trial at Extruct AI: https://www.extruct.ai/                                                                                            | Extruct AI official site for account signup                                              |
| Open Extruct table template to create your enrichment table: https://app.extruct.ai/tables/shared/wZ6FxspNc5ctrB55                                         | Extruct table template URL                                                               |
| Slack app setup instructions including required OAuth scopes and event subscription enablement.                                                            | Slack API docs: https://api.slack.com/apps                                               |
| Slack Bot Token Scopes required: `channels:read`, `channels:history`, `chat:write`                                                                          | Slack OAuth2 Bot permission requirements                                                |
| Invite bot to channel with `/invite @botname` to enable message monitoring and replies.                                                                     | Slack channel bot usage instructions                                                     |
| Extruct API authentication uses Bearer Token via generic HTTP credentials in n8n.                                                                           | Secure API credential handling                                                           |
| The workflow is designed to ignore Slack thread replies to avoid duplicate processing.                                                                       | Implementation detail in domain extraction code node                                    |
| Slack messages are posted back as threaded replies matching the original message timestamp (`thread_ts`).                                                   | Slack message threading best practice                                                    |
| The workflow includes rich error handling by filtering invalid domains and throwing errors if data rows are missing.                                       | Error handling notes                                                                     |
| Slack message formatting uses markdown and link formatting for clickable URLs.                                                                              | Slack message formatting guide: https://api.slack.com/reference/block-kit/block-elements  |

---

**Disclaimer:**  
The provided text is derived exclusively from an n8n automated workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---