Create & Send Client Session Summaries from Zoom Meetings via Gmail and Airtable

https://n8nworkflows.xyz/workflows/create---send-client-session-summaries-from-zoom-meetings-via-gmail-and-airtable-6324


# Create & Send Client Session Summaries from Zoom Meetings via Gmail and Airtable

### 1. Workflow Overview

This workflow automates the processing and communication of Zoom meeting assets received via Gmail, specifically for client sessions. It is designed to:

- Trigger on incoming Gmail messages containing Zoom meeting summary emails labeled with "Meeting assets."
- Extract detailed session information from the email content, such as client name, session type, date/time, duration, recording link, and various summary sections.
- Query an Airtable database to find the corresponding client by full name.
- Send a personalized email to the client with session details and recording link.
- Log the session information as a new record linked to the client in Airtable for record-keeping.

**Logical blocks:**

- **1.1 Input Reception & Trigger:** Gmail Trigger node that listens for emails containing "Meeting assets."
- **1.2 Data Extraction:** Function node to parse required session details from the email body.
- **1.3 Conditional Filtering:** If node to branch processing based on session type (e.g., exclude exploratory calls).
- **1.4 Client Lookup:** HTTP Request node to search for the client in Airtable “People” table.
- **1.5 Communication:** Gmail node to send the session summary email to the client.
- **1.6 Session Logging:** HTTP Request node to create a new session record in Airtable “Sessions” table.
- **1.7 Credential Setup Notes:** Sticky Notes to remind about Gmail and Airtable credential configuration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Trigger

- **Overview:**  
  Listens for new Gmail messages filtered by the subject/content containing "Meeting assets" every minute.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Note: Gmail Creds (Sticky Note)

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Gmail Trigger  
    - Role: Triggers workflow on incoming Gmail messages matching filter.  
    - Configuration: Filters emails where the content contains "Meeting assets"; polling interval every 1 minute.  
    - Inputs: None (trigger node)  
    - Outputs: JSON data of the email message  
    - Edge cases: Authentication failure if Gmail OAuth2 credentials not set; network/timeouts; no matching emails could result in no workflow runs.  
    - Sticky note reminds configuration of Gmail OAuth2 credentials before activation.

  - **Note: Gmail Creds**  
    - Type: Sticky Note  
    - Role: Instruction to configure Gmail OAuth2 credentials securely in n8n.

---

#### 2.2 Data Extraction

- **Overview:**  
  Parses the raw email text to extract session details such as client name, session type, date/time, duration, recording link, and multiple summary sections.

- **Nodes Involved:**  
  - Extract Fields

- **Node Details:**

  - **Extract Fields**  
    - Type: Function  
    - Role: Runs custom JavaScript to parse email body text using regular expressions.  
    - Configuration:  
      - Extracts client name from a pattern "Meeting assets for ... (Client Name) with"  
      - Extracts session type like "1 hour", "2 hours", or "exploratory call"  
      - Extracts UTC datetime from a GMT timestamp in the email  
      - Extracts duration in HH:MM:SS format  
      - Extracts Zoom recording link URL  
      - Extracts quick summary, next steps, and detailed summary text blocks  
    - Inputs: JSON with email text content  
    - Outputs: JSON enriched with extracted fields  
    - Edge cases:  
      - Regex mismatch if email format changes or unexpected formatting may cause empty or incorrect fields.  
      - Missing fields could cause downstream nodes to fail or send incomplete data.

---

#### 2.3 Conditional Filtering

- **Overview:**  
  Checks if the session type is NOT an "exploratory call" to continue processing; otherwise, stops workflow.

- **Nodes Involved:**  
  - If Exploratory

- **Node Details:**

  - **If Exploratory**  
    - Type: If  
    - Role: Conditional branching based on session type string.  
    - Configuration: Checks that sessionType does not contain "exploratory call" (case insensitive).  
    - Inputs: JSON with extracted sessionType from previous node.  
    - Outputs:  
      - True branch: continues if session is NOT exploratory  
      - False branch: stops workflow (not connected)  
    - Edge cases: If sessionType is undefined or empty, condition may behave unexpectedly; ensure sessionType is always set or use default.

---

#### 2.4 Client Lookup

- **Overview:**  
  Queries Airtable “People” table to find the client’s record by matching the full name extracted from the email.

- **Nodes Involved:**  
  - Airtable: Search People  
  - Note: Airtable Creds (Sticky Note)

- **Node Details:**

  - **Airtable: Search People**  
    - Type: HTTP Request  
    - Role: Performs GET request to Airtable API to search for a person by full name.  
    - Configuration:  
      - URL built using environment variable for Airtable base ID  
      - Query string uses Airtable formula filter: `{Full Name} = 'clientName'`  
      - Authorization header with bearer token from n8n credentials (airtableApi.token)  
      - Request method: GET  
    - Inputs: JSON containing `clientName`  
    - Outputs: JSON response with matching records array  
    - Edge cases:  
      - No matching client: downstream nodes expecting client data may fail or send email to empty address.  
      - Authentication errors if Airtable API token missing or invalid.  
      - Rate limiting or network errors.  

  - **Note: Airtable Creds**  
    - Type: Sticky Note  
    - Role: Instruction to securely store Airtable API token in n8n credentials.

---

#### 2.5 Communication

- **Overview:**  
  Sends a personalized email to the client containing session details, summaries, and recording link.

- **Nodes Involved:**  
  - Send Email

- **Node Details:**

  - **Send Email**  
    - Type: Gmail  
    - Role: Sends an email using configured Gmail OAuth2 credentials.  
    - Configuration:  
      - Subject: "Your Session Summary & Recording"  
      - Recipient email dynamically fetched from Airtable “People” record (`records[0].fields.Email`)  
      - HTML body includes client name, session type, date/time, duration, recording link, quick summary, next steps, detailed summary using JSON expressions.  
    - Inputs: JSON with session data and client email  
    - Outputs: Email send status metadata  
    - Edge cases:  
      - Missing or invalid client email leads to failure or undelivered email.  
      - Gmail API quota limits, auth errors, or network issues.  
      - If Airtable returns no records, accessing `records[0]` will cause error.

---

#### 2.6 Session Logging

- **Overview:**  
  Creates a new session record in Airtable “Sessions” table, linking it to the client record.

- **Nodes Involved:**  
  - Airtable: Create Session

- **Node Details:**

  - **Airtable: Create Session**  
    - Type: HTTP Request  
    - Role: Posts a new record to Airtable Sessions table with fields for date/time, duration, summaries, next steps, and client link.  
    - Configuration:  
      - URL uses Airtable base ID environment variable  
      - Request method: POST  
      - JSON body includes fields with session details and linked client record ID from previous Airtable search node  
      - Authorization header with bearer token credential  
      - Content-Type: application/json  
    - Inputs: JSON with session data and client Airtable record ID  
    - Outputs: JSON with created record info  
    - Edge cases:  
      - If client ID missing or incorrect, record creation will fail.  
      - Authentication or API errors, rate limiting.  

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                       | Input Node(s)          | Output Node(s)                  | Sticky Note                                                         |
|-------------------------|--------------------|------------------------------------|-----------------------|-------------------------------|--------------------------------------------------------------------|
| Note: Gmail Creds        | Sticky Note        | Reminder for Gmail OAuth2 setup     | —                     | Gmail Trigger                  | 📒 Configure your Gmail OAuth2 credentials in n8n before activating this workflow. |
| Gmail Trigger           | Gmail Trigger      | Trigger on emails containing "Meeting assets" | —                     | Extract Fields                 |                                                                    |
| Extract Fields          | Function           | Extract session details from email  | Gmail Trigger          | If Exploratory                 |                                                                    |
| If Exploratory           | If                 | Filter out "exploratory call" sessions | Extract Fields         | Airtable: Search People        |                                                                    |
| Airtable: Search People  | HTTP Request       | Find client record by full name     | If Exploratory         | Send Email, Airtable: Create Session | 🔐 Store your Airtable API token securely in n8n Credentials (airtableApi.token). |
| Send Email              | Gmail              | Send personalized session email     | Airtable: Search People | —                             |                                                                    |
| Airtable: Create Session | HTTP Request       | Log session in Airtable Sessions table | Airtable: Search People | —                             |                                                                    |
| Note: Airtable Creds     | Sticky Note        | Reminder for Airtable API token setup | —                     | —                             | 🔐 Store your Airtable API token securely in n8n Credentials (airtableApi.token). |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note: Gmail Credential Reminder**  
   - Type: Sticky Note  
   - Content: "📒 Configure your Gmail OAuth2 credentials in n8n before activating this workflow."  
   - Position near Gmail Trigger.

2. **Add Gmail Trigger Node**  
   - Configure to trigger every 1 minute.  
   - Under Filters, add a string filter: contains "Meeting assets"  
   - Connect to next node.

3. **Add Function Node "Extract Fields"**  
   - Paste the JavaScript code to extract: clientName, sessionType, dateTimeUTC, duration, recordingLink, quickSummary, nextSteps, detailedSummary from email text.  
   - Input: output of Gmail Trigger.  
   - Output: enriched JSON with extracted fields.

4. **Add If Node "If Exploratory"**  
   - Condition: string condition  
   - Check if `{{$json.sessionType}}` does NOT contain "exploratory call" (case insensitive).  
   - True branch continues; false branch leaves workflow.

5. **Add HTTP Request Node "Airtable: Search People"**  
   - Method: GET  
   - URL: `https://api.airtable.com/v0/{{$env.AIRTABLE_BASE_ID}}/People`  
   - Query Parameters: `filterByFormula` = `{Full Name} = '{{$json.clientName}}'`  
   - Authentication: Header Auth  
   - Header: Authorization: Bearer {{$credentials.airtableApi.token}}  
   - Connect True output of If node to this.

6. **Add Sticky Note: Airtable Credential Reminder**  
   - Content: "🔐 Store your Airtable API token securely in n8n Credentials (airtableApi.token)."  
   - Position near Airtable HTTP nodes.

7. **Add Gmail Node "Send Email"**  
   - Configure with Gmail OAuth2 credentials.  
   - To Email: `{{$node["Airtable: Search People"].json.records[0].fields.Email}}`  
   - Subject: "Your Session Summary & Recording"  
   - HTML Body: Use expressions to include clientName, sessionType, dateTimeUTC, duration, recordingLink, quickSummary, nextSteps, detailedSummary.  
   - Connect output of Airtable search node.

8. **Add HTTP Request Node "Airtable: Create Session"**  
   - Method: POST  
   - URL: `https://api.airtable.com/v0/{{$env.AIRTABLE_BASE_ID}}/Sessions`  
   - Headers: Authorization Bearer token from credentials; Content-Type: application/json  
   - Body (JSON Parameters enabled):  
     ```json
     {
       "fields": {
         "Date & Time": "{{$json.dateTimeUTC}}",
         "Duration": "{{$json.duration}}",
         "Quick Summary": "{{$json.quickSummary}}",
         "Detailed Summary": "{{$json.detailedSummary}}",
         "Next Steps": "{{$json.nextSteps}}",
         "Client": ["{{$node[\"Airtable: Search People\"].json.records[0].id}}"]
       }
     }
     ```  
   - Connect output of Airtable search node.

9. **Verify Credentials Setup**  
   - Gmail OAuth2 credentials configured in n8n for Gmail Trigger and Gmail send node.  
   - Airtable API token stored securely as credentials under `airtableApi.token`.  
   - Airtable Base ID stored as environment variable `$AIRTABLE_BASE_ID` for flexibility.

10. **Activate Workflow**  
    - Test with real emails containing "Meeting assets" and verify the full process: extraction, client lookup, email sending, session logging.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                        |
|------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| Workflow triggers on Gmail emails labeled "Meeting assets" from Zoom meetings. | Workflow description and trigger setup.                              |
| Use environment variables for Airtable Base ID for portability and security.  | Encourages best practices for environment variables in n8n.          |
| No API keys or tokens are hardcoded; all credentials must be securely stored in n8n. | Security best practice.                                               |
| Gmail OAuth2 setup instructions: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/ | Official n8n documentation.                                           |
| Airtable API and formula reference: https://airtable.com/api | For crafting filterByFormula queries and API usage.                   |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.