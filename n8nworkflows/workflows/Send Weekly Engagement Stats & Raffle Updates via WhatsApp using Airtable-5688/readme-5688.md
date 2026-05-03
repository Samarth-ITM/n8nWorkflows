Send Weekly Engagement Stats & Raffle Updates via WhatsApp using Airtable

https://n8nworkflows.xyz/workflows/send-weekly-engagement-stats---raffle-updates-via-whatsapp-using-airtable-5688


# Send Weekly Engagement Stats & Raffle Updates via WhatsApp using Airtable

### 1. Workflow Overview

This workflow automates the weekly distribution of personalized engagement statistics and raffle updates to users via WhatsApp. It targets organizations or campaigns that track user engagement points and raffle ticket allocations in Airtable and want to motivate ongoing participation with regular updates.

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Initiates the workflow every Wednesday at 1 PM.
- **1.2 Data Retrieval:** Queries Airtable to fetch all user records containing WhatsApp IDs, current points, and raffle voucher counts.
- **1.3 Message Preparation & Sending:** Formats a personalized WhatsApp message per user with their engagement stats and sends it via the Whapi Cloud API.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow execution automatically on a weekly schedule, specifically every Wednesday at 1 PM.

- **Nodes Involved:**  
  - Every wednesday at 1 pm (Schedule Trigger)

- **Node Details:**  
  - **Node:** Every wednesday at 1 pm  
  - **Type:** Schedule Trigger (n8n-nodes-base.scheduleTrigger)  
  - **Configuration:** Set to run weekly on Wednesdays (day 3 of the week) at 13:00 hours (1 PM).  
  - **Input/Output:** No input; output triggers the next node "Search records."  
  - **Edge Cases / Failures:**  
    - If n8n server time zone is misconfigured, triggers may occur at unintended times.  
    - Workflow will not run if n8n instance is offline or paused.  
  - **Version:** 1.2

---

#### 1.2 Data Retrieval

- **Overview:**  
  This block accesses the Airtable base and table containing user engagement data. It retrieves all records to prepare personalized messages.

- **Nodes Involved:**  
  - Search records (Airtable node)

- **Node Details:**  
  - **Node:** Search records  
  - **Type:** Airtable node (n8n-nodes-base.airtable)  
  - **Configuration:**  
    - Base ID: appREiqyOxTYwsigc ("WhatsApp Engagement Database")  
    - Table ID: tblIf7YbtyvUvDNm0 ("Table 1")  
    - Operation: Search (retrieves all records)  
  - **Credentials:** Uses Airtable Personal Access Token (OAuth token) for authentication.  
  - **Input/Output:**  
    - Input from Schedule Trigger node.  
    - Outputs records with fields including WhatsApp ID, points, raffle vouchers.  
  - **Edge Cases / Failures:**  
    - API rate limits from Airtable may cause failures.  
    - Missing or malformed records may cause errors downstream.  
    - Invalid or expired API token leads to authorization errors.  
  - **Version:** 2.1

---

#### 1.3 Message Preparation & Sending

- **Overview:**  
  For each user record retrieved, this block formats a personalized WhatsApp message including their points and raffle vouchers, then sends it using the Whapi Cloud HTTP API.

- **Nodes Involved:**  
  - Send WhasApp (Weekly Message) (HTTP Request)

- **Node Details:**  
  - **Node:** Send WhasApp (Weekly Message)  
  - **Type:** HTTP Request (n8n-nodes-base.httpRequest)  
  - **Configuration:**  
    - Method: POST  
    - URL: https://gate.whapi.cloud/messages/text  
    - Headers:  
      - Accept: application/json  
      - Authorization: Bearer {TOKEN} (a placeholder for the actual Whapi API token)  
    - Body (JSON):  
      ```json
      {
        "to": "{{ $json.WhatsApp_ID }}",
        "body": "MESSAGE"
      }
      ```  
      The `to` field uses the WhatsApp ID from each record. The `body` field is a placeholder "MESSAGE" — presumably to be replaced with the personalized message text including points and raffle vouchers.  
  - **Input/Output:**  
    - Receives data from the "Search records" node.  
    - Sends HTTP POST requests to Whapi API for each user.  
  - **Edge Cases / Failures:**  
    - Invalid WhatsApp IDs or missing phone numbers cause message failures.  
    - Authorization errors if the API token is invalid or expired.  
    - Network timeouts or API unavailability.  
    - Rate limiting from Whapi API.  
    - The placeholder "MESSAGE" suggests that message templating may be incomplete and could cause uninformative messages if not dynamically set.  
  - **Version:** 4.2

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                    | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                   |
|---------------------------|---------------------|----------------------------------|------------------------|--------------------------|-----------------------------------------------------------------------------------------------|
| Every wednesday at 1 pm    | Schedule Trigger    | Triggers workflow weekly          | —                      | Search records           | ## Weekly on wednesday                                                                        |
| Search records            | Airtable            | Retrieves user engagement data    | Every wednesday at 1 pm | Send WhasApp (Weekly Message) | ## Check points & riffle vouchers                                                             |
| Send WhasApp (Weekly Message) | HTTP Request      | Sends personalized WhatsApp msgs  | Search records         | —                        | ## Send message                                                                               |
| Sticky Note               | Sticky Note         | Workflow description and summary  | —                      | —                        | ## Scheduled Trigger: Every Wednesday at 1 pm, the workflow is automatically triggered... (full detailed description) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**  
   - Name: "Every wednesday at 1 pm"  
   - Type: Schedule Trigger  
   - Set to trigger every week on Wednesday (day 3) at 13:00 hours (1 PM).  

2. **Create the Airtable node:**  
   - Name: "Search records"  
   - Type: Airtable  
   - Configure credentials: Set up Airtable Personal Access Token with appropriate permissions.  
   - Select Base: "WhatsApp Engagement Database" (Base ID: appREiqyOxTYwsigc)  
   - Select Table: "Table 1" (Table ID: tblIf7YbtyvUvDNm0)  
   - Operation: Search (to retrieve all records)  
   - Connect the output of the Schedule Trigger node ("Every wednesday at 1 pm") to this node.  

3. **Create the HTTP Request node:**  
   - Name: "Send WhasApp (Weekly Message)"  
   - Type: HTTP Request  
   - Configure Method: POST  
   - URL: https://gate.whapi.cloud/messages/text  
   - Headers:  
     - Accept: application/json  
     - Authorization: Bearer {TOKEN} (replace {TOKEN} with your actual Whapi API key)  
   - Body Content Type: JSON  
   - Body Parameters (raw JSON):  
     ```json
     {
       "to": "{{ $json.WhatsApp_ID }}",
       "body": "Your message here including points and raffle vouchers"
     }
     ```  
   - Connect output of "Search records" node to this HTTP Request node.  
   - Ensure the message `body` field is dynamically composed to include user-specific data such as points and raffle vouchers. This may require adding a "Set" or "Function" node before this to compose the message string if needed (not included in original workflow but recommended).  

4. **Test the workflow:**  
   - Confirm Airtable API returns expected records with valid WhatsApp IDs.  
   - Confirm messages are correctly formatted and sent to test WhatsApp numbers.  
   - Monitor for failures such as invalid tokens, rate limits, or missing data.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow runs weekly on Wednesday at 1 PM to provide users with their current engagement points and raffle vouchers, encouraging continued interaction and participation in weekly raffles.                                                                                                                                                                                      | Sticky Note on Scheduled Trigger node                                                           |
| The WhatsApp message sending uses the Whapi Cloud API. Ensure you have a valid API token and that the WhatsApp IDs are properly formatted (including country codes).                                                                                                                                                                                                               | Whapi Cloud API documentation: https://whapi.cloud/docs                                        |
| Airtable limits API calls; ensure you handle pagination if your user base grows beyond Airtable's single request limit. Current workflow assumes all records fit in one request. Consider adding pagination for scalability.                                                                                                                                                          | Airtable API docs: https://airtable.com/api                                                    |
| The message body in the HTTP Request node is a placeholder "MESSAGE". To personalize messages, insert a node (e.g., Function or Set) to dynamically build the message string using fields from Airtable before sending.                                                                                                                                                                | n8n community forum for message templating examples: https://community.n8n.io/                   |
| The workflow lacks explicit error handling nodes (e.g., for API failures or missing data). Adding "Error Trigger" or conditional checks can improve robustness and alerting.                                                                                                                                                                                                       | Consider adding error handling for production use                                               |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.