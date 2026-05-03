Smart Invoice Collection System with GPT-4.1, Gmail & Google Sheets

https://n8nworkflows.xyz/workflows/smart-invoice-collection-system-with-gpt-4-1--gmail---google-sheets-5391


# Smart Invoice Collection System with GPT-4.1, Gmail & Google Sheets

### 1. Workflow Overview

This workflow, titled **Smart Invoice Collection System with GPT-4.1, Gmail & Google Sheets**, automates the process of managing and following up on overdue invoices using a combination of data extraction, email history analysis, AI content generation, and automated email dispatch. It is designed for businesses that want to efficiently track overdue payments and send personalized, context-aware follow-ups without manual intervention.

The workflow is logically divided into four main blocks:

- **1.1 Invoice Data Collection:** Retrieves invoice data from Google Sheets and filters for invoices overdue by specific intervals (7, 14, 21, or 28 days).
- **1.2 Email History Intelligence:** Checks recent email communications with clients to avoid redundant or poorly timed follow-ups.
- **1.3 AI-Powered Follow-up Generation:** Uses GPT-4.1 to analyze conversation context and generate appropriate follow-up email content.
- **1.4 Automated Email Delivery:** Sends personalized follow-up emails only if approved by the AI analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Invoice Data Collection

- **Overview:**  
  Retrieves invoice records from a Google Sheet, calculates how many days have passed since the invoice was sent, and filters for invoices overdue by 7, 14, 21, or 28 days. This block identifies invoices requiring follow-up.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Schedule Trigger  
  - Google Sheets  
  - Edit Fields  
  - Filter  
  - Loop Over Items

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Entry point for manual execution of the workflow.  
    - *Config:* No parameters.  
    - *Connections:* Output to Google Sheets.  
    - *Edge Cases:* None relevant; manual start only.

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Automated periodic execution starter.  
    - *Config:* Runs on a set interval (interval details unspecified, defaults to every minute if not configured).  
    - *Connections:* Output to Google Sheets.  
    - *Edge Cases:* Misconfiguration may cause frequent or no triggers.

  - **Google Sheets**  
    - *Type:* Google Sheets node (v4.6)  
    - *Role:* Fetches rows from a specified Google Sheet tracking invoices.  
    - *Config:* Reads from sheet with ID `1oPx-jSljqW1YVpojZ9zPa10H7MoiQERIdrFESS6lUqo`, sheet named "Sheet1" (gid=0).  
    - *Credentials:* Uses OAuth2 credentials named "YouTube".  
    - *Outputs:* All rows representing invoices.  
    - *Edge Cases:* API quota limits, auth failures, empty sheets.

  - **Edit Fields**  
    - *Type:* Set node (v3.4)  
    - *Role:* Calculates `daysSinceSent` by computing the difference between current date and the invoice "Date Sent". Also extracts `email` and client's first name.  
    - *Config:*  
      - `daysSinceSent` = rounded difference in days from now to "Date Sent" column.  
      - `email` = value of the "Email" column.  
      - `firstName` = first word of "Client Name".  
    - *Inputs:* Google Sheets output.  
    - *Outputs:* Modified invoice data with new fields.  
    - *Edge Cases:* Missing or invalid dates, malformed client names.

  - **Filter**  
    - *Type:* Filter node (v2.2)  
    - *Role:* Passes only invoices overdue by exactly 7, 14, 21, or 28 days.  
    - *Config:* Condition is an OR combination of `daysSinceSent` equals any of the specified values.  
    - *Inputs:* Output from Edit Fields.  
    - *Outputs:* Filtered invoices for further processing.  
    - *Edge Cases:* Off-by-one date errors; missing `daysSinceSent`.

  - **Loop Over Items**  
    - *Type:* SplitInBatches (v3)  
    - *Role:* Processes each overdue invoice individually in sequence.  
    - *Config:* Default batch size (assumed 1).  
    - *Inputs:* Filter node output.  
    - *Outputs:* Single invoice item per iteration for downstream nodes.  
    - *Edge Cases:* Large number of invoices may slow processing.

---

#### 2.2 Email History Intelligence

- **Overview:**  
  For each overdue invoice, retrieves recent Gmail message history with the client to analyze recent communications. Aggregates email content and timestamps for AI review, ensuring that follow-ups are sent only when appropriate.

- **Nodes Involved:**  
  - Loop Over Items (continuation)  
  - Gmail  
  - Aggregate  
  - Edit Fields1

- **Node Details:**

  - **Gmail**  
    - *Type:* Gmail node (v2.1)  
    - *Role:* Retrieves up to 10 recent emails either from or to the client’s email address.  
    - *Config:*  
      - Query: `from:<client_email> OR to:<client_email>`.  
      - Limit: 10 messages.  
      - Retrieves full message details (not simple).  
    - *Credentials:* OAuth2 credential "Gmail account 4".  
    - *Inputs:* Single invoice from Loop Over Items (second main output).  
    - *Outputs:* Array of email messages with headers (including dates) and text.  
    - *Edge Cases:* Gmail API quota, auth expiration, empty email history.

  - **Aggregate**  
    - *Type:* Aggregate node (v1)  
    - *Role:* Combines email text and date headers into arrays to prepare for AI input.  
    - *Config:*  
      - Merges lists of `text` and `headers.date`.  
    - *Inputs:* Gmail output.  
    - *Outputs:* Single aggregated JSON object containing arrays of email texts and corresponding dates.  
    - *Edge Cases:* Empty email list yields empty arrays.

  - **Edit Fields1**  
    - *Type:* Set node (v3.4)  
    - *Role:* Creates a combined formatted string of email history with timestamps and sets up follow-up template array.  
    - *Config:*  
      - `textWithDate` field: concatenates each email's date and text with newline separators.  
      - `followUpTemplateArray`: an array of 4 escalating follow-up email templates with placeholders for client first name and varying urgency.  
    - *Inputs:* Output from Aggregate.  
    - *Outputs:* Adds fields required for AI prompt construction.  
    - *Edge Cases:* Empty email history produces empty `textWithDate`. Templates must be correctly formatted.

---

#### 2.3 AI-Powered Follow-up Generation

- **Overview:**  
  Uses GPT-4.1 via the OpenAI node to analyze the email conversation history and decide if a follow-up should be sent. If approved, it generates a personalized email template adjusted for context and urgency based on days overdue.

- **Nodes Involved:**  
  - Edit Fields1  
  - OpenAI  
  - Filter1

- **Node Details:**

  - **OpenAI**  
    - *Type:* OpenAI node (LangChain integration, v1.8)  
    - *Role:* Sends a system + user prompt to GPT-4.1 to:  
      - Determine if a follow-up email should be sent (verdict: true/false).  
      - Return a customized email template based on conversation history and days overdue.  
    - *Config:*  
      - Model: `gpt-4.1` with temperature 0.7 (balanced creativity).  
      - System prompt defines role as administrative assistant.  
      - User prompt includes:  
        - Template selection logic based on `daysSinceSent` (7, 14, 21, 28).  
        - Rules about avoiding follow-ups if there was communication in last 72 hours.  
        - Instruction to modify template only if necessary for context.  
        - Current date and full conversation history passed as variables.  
      - Output format: JSON with `verdict` and `emailTemplate`.  
    - *Credentials:* OpenAI API named "YouTube_Feb 4".  
    - *Inputs:* Output from Edit Fields1 (formatted conversation + templates).  
    - *Outputs:* JSON object with AI decision and content.  
    - *Edge Cases:* API rate limits, prompt failures, unexpected response formats.

  - **Filter1**  
    - *Type:* Filter node (v2.2)  
    - *Role:* Allows only items with AI `verdict` equal to `"true"` to proceed for email sending.  
    - *Config:* Checks `message.content.verdict === "true"`.  
    - *Inputs:* Output from OpenAI.  
    - *Outputs:* Filtered items approved for sending.  
    - *Edge Cases:* Missing or malformed verdict field.

---

#### 2.4 Automated Email Delivery

- **Overview:**  
  Sends the AI-approved personalized follow-up emails via Gmail drafts, using professional subject lines and contextual message bodies.

- **Nodes Involved:**  
  - Filter1  
  - Gmail1  
  - Loop Over Items (continuation)

- **Node Details:**

  - **Gmail1**  
    - *Type:* Gmail node (v2.1)  
    - *Role:* Creates a Gmail draft message ready to send with the personalized follow-up content.  
    - *Config:*  
      - Subject line: `"Re: Invoice for <firstName>"` using the client’s first name.  
      - Message body: AI-generated `emailTemplate`.  
      - Recipient: Client's email from Loop Over Items.  
      - Mode: Create draft (not send immediately).  
    - *Credentials:* OAuth2 credential "Gmail account 4".  
    - *Inputs:* Filter1 output.  
    - *Outputs:* Draft created in Gmail account.  
    - *Edge Cases:* Gmail API quota, auth errors, invalid email addresses.

  - **Loop Over Items** (main continuation)  
    - Connects Gmail1 back to Loop Over Items for batch processing.  
    - *Edge Cases:* Large batch sizes may cause delays.

---

### 3. Summary Table

| Node Name                  | Node Type                 | Functional Role                          | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                  |
|----------------------------|---------------------------|----------------------------------------|--------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger            | Manual start of workflow                |                          | Google Sheets            |                                                                                              |
| Schedule Trigger           | Schedule Trigger          | Periodic automated start                |                          | Google Sheets            |                                                                                              |
| Google Sheets             | Google Sheets (v4.6)       | Fetch invoice data                      | When clicking ‘Execute workflow’, Schedule Trigger | Edit Fields              | 📊 STEP 1: Invoice Data Collection - monitors overdue invoices from Google Sheets           |
| Edit Fields                | Set (v3.4)                | Calculate daysSinceSent, extract email and firstName | Google Sheets            | Filter                   | 📊 STEP 1: Invoice Data Collection                                                           |
| Filter                    | Filter (v2.2)             | Filter invoices overdue by 7,14,21,28 days | Edit Fields              | Loop Over Items           | 📊 STEP 1: Invoice Data Collection                                                           |
| Loop Over Items            | SplitInBatches (v3)       | Process each overdue invoice individually | Filter                   | Gmail (secondary output), Gmail1 (via Filter1) | 🔍 STEP 2: Email History Intelligence - processes each invoice separately                   |
| Gmail                     | Gmail (v2.1)              | Retrieve recent email history with client | Loop Over Items (secondary output) | Aggregate                | 🔍 STEP 2: Email History Intelligence                                                        |
| Aggregate                 | Aggregate (v1)            | Combine email texts and dates for AI input | Gmail                    | Edit Fields1             | 🔍 STEP 2: Email History Intelligence                                                        |
| Edit Fields1              | Set (v3.4)                | Format conversation and set follow-up templates | Aggregate                | OpenAI                   | 🔍 STEP 2: Email History Intelligence                                                        |
| OpenAI                    | OpenAI LangChain (v1.8)   | Analyze conversation, decide on follow-up and generate email content | Edit Fields1             | Filter1                  | 🧠 STEP 3: AI-Powered Follow-up Generation                                                   |
| Filter1                   | Filter (v2.2)             | Pass only AI-approved follow-ups       | OpenAI                   | Gmail1                   | 🧠 STEP 3: AI-Powered Follow-up Generation                                                   |
| Gmail1                    | Gmail (v2.1)              | Create Gmail draft with personalized follow-up email | Filter1                  | Loop Over Items           | 📧 STEP 4: Automated Email Delivery - sends personalized follow-ups automatically           |
| sticky-note-1             | Sticky Note               | Documentation Note                      |                          |                         | 📊 STEP 1: Invoice Data Collection                                                           |
| sticky-note-2             | Sticky Note               | Documentation Note                      |                          |                         | 🔍 STEP 2: Email History Intelligence                                                        |
| sticky-note-3             | Sticky Note               | Documentation Note                      |                          |                         | 🧠 STEP 3: AI-Powered Follow-up Generation                                                   |
| sticky-note-4             | Sticky Note               | Documentation Note                      |                          |                         | 📧 STEP 4: Automated Email Delivery                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Workflow and Set Credentials:**
   - Set up Google Sheets OAuth2 credentials.
   - Set up Gmail OAuth2 credentials.
   - Set up OpenAI API credentials.

2. **Add Trigger Nodes:**
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`.
   - Add a **Schedule Trigger** node named `Schedule Trigger`.
     - Configure interval according to desired frequency (e.g., daily).

3. **Add Google Sheets Node:**
   - Add Google Sheets node named `Google Sheets`.
   - Configure:
     - Document ID: Your invoice tracking Google Sheet ID.
     - Sheet Name: The specific sheet (e.g., "Sheet1").
   - Connect both triggers’ outputs to this node.

4. **Calculate Days Since Invoice Sent:**
   - Add a **Set** node named `Edit Fields`.
   - Add fields:
     - `daysSinceSent` = `={{ $now.diffTo($json['Date Sent'], 'days').round() }}`  
     - `email` = `={{ $json.Email }}`  
     - `firstName` = `={{ $json["Client Name"].split(" ")[0] }}`
   - Connect `Google Sheets` output to this node.

5. **Filter Overdue Invoices:**
   - Add a **Filter** node named `Filter`.
   - Add OR conditions where `daysSinceSent` equals 7, 14, 21, or 28.
   - Connect from `Edit Fields`.

6. **Process Each Invoice Individually:**
   - Add a **SplitInBatches** node named `Loop Over Items`.
   - Connect from `Filter`.
   - Default batch size (1) is suitable.

7. **Retrieve Email History:**
   - Add a **Gmail** node named `Gmail`.
   - Configure:  
     - Operation: Get All  
     - Limit: 10  
     - Query: `from:{{ $json.email }} OR to:{{ $json.email }}`
   - Use your Gmail OAuth2 credentials.
   - Connect the secondary output of `Loop Over Items` to this node.

8. **Aggregate Email Data:**
   - Add an **Aggregate** node named `Aggregate`.
   - Configure to merge lists of `text` and `headers.date`.
   - Connect `Gmail` output.

9. **Format Email History and Set Templates:**
   - Add a **Set** node named `Edit Fields1`.
   - Add fields:  
     - `textWithDate` = `={{ $json.text.map((item,index) => $json.date[index] + '\n' + item ) }}`  
     - `followUpTemplateArray` = An array of 4 escalating templates as strings with placeholders for firstName and line breaks (`\n`).
   - Connect from `Aggregate`.

10. **AI Follow-up Decision and Email Generation:**
    - Add an **OpenAI** node named `OpenAI`.
    - Configure:  
      - Model: `gpt-4.1`  
      - Temperature: 0.7  
      - Messages:  
        - System: `"You're a helpful, intelligent administrative assistant. You help me follow up with emails."`  
        - User: Dynamic prompt embedding conversation history, templates, days overdue, rules about recent communications, current date, and instructions to return JSON with `verdict` and `emailTemplate`.  
      - Enable JSON output.  
    - Use OpenAI API credentials.
    - Connect from `Edit Fields1`.

11. **Filter for AI-approved Follow-ups:**
    - Add a **Filter** node named `Filter1`.
    - Condition: `message.content.verdict` equals `"true"`.
    - Connect from `OpenAI`.

12. **Send Follow-up Emails:**
    - Add a **Gmail** node named `Gmail1`.
    - Configure:  
      - Resource: Draft  
      - Subject: `"Re: Invoice for {{ $('Loop Over Items').item.json.firstName }}"`  
      - Message: `={{ $json.message.content.emailTemplate }}`  
      - Send To: `{{ $('Loop Over Items').item.json.email }}`  
    - Use Gmail OAuth2 credentials.
    - Connect from `Filter1`.

13. **Complete Loop Cycle:**
    - Connect `Gmail1` output back to the main output of `Loop Over Items` to continue batch processing.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                           |
|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow uses 4 escalating follow-up email templates with increasing urgency customized by the AI.         | Step 3: AI-Powered Follow-up Generation                    |
| Avoids sending follow-ups if there was any communication with the client discussing the invoice in last 72 hours. | Rule embedded in OpenAI prompt                             |
| Google Sheet must contain columns: Date Sent, Client Name, Email, and Invoice ID for proper operation.           | Step 1: Invoice Data Collection                            |
| Gmail nodes require OAuth2 credentials with appropriate scopes for reading and drafting emails.                 | Gmail nodes configuration                                  |
| OpenAI integration uses LangChain node with GPT-4.1 model and requires API key with sufficient quota.           | OpenAI node details                                        |
| The workflow supports both manual execution and scheduled periodic runs.                                        | Trigger nodes                                              |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created in n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.