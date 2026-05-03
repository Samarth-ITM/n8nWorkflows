Extract Email Tasks with Gmail, ChatGPT-4o and Supabase

https://n8nworkflows.xyz/workflows/extract-email-tasks-with-gmail--chatgpt-4o-and-supabase-5496


# Extract Email Tasks with Gmail, ChatGPT-4o and Supabase

---

### 1. Workflow Overview

This workflow automates the extraction of actionable tasks from unread Gmail emails using OpenAI's GPT-4o model, then stores the structured task information into a Supabase database. It is designed for productivity-focused users who want to process incoming emails regularly, extract tasks without duplication, and maintain a centralized task repository.

**Target Use Cases:**
- Automated email task extraction for personal or team productivity.
- Avoiding redundant processing of emails already analyzed.
- Structured task data generation from unstructured email content.
- Integration of Gmail, AI task summarization, and Supabase task database.

**Logical Blocks:**

- **1.1 Trigger and Email Retrieval:** Scheduled trigger initiates the workflow; fetches unread emails from Gmail inbox.
- **1.2 Email Deduplication Check:** Checks if each email has already been processed by querying Supabase.
- **1.3 Task Extraction via GPT:** For new emails, formats the email content into a GPT prompt and sends it to ChatGPT-4o to extract structured task details.
- **1.4 Data Insertion to Supabase:** Inserts or upserts the extracted task data and email metadata into the Supabase `emails` table.
- **1.5 Control Flow and Looping:** Manages batch processing of emails and conditional routing based on deduplication results.
- **1.6 Documentation and Instructions:** A sticky note provides detailed notes, prerequisites, and credentials required.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Email Retrieval

**Overview:**  
This block schedules the workflow to run periodically and fetches all unread emails from the Gmail inbox for processing.

**Nodes Involved:**  
- Trigger Workflow  
- Get Unread Emails from Inbox  
- Loop Over Items

**Node Details:**

- **Trigger Workflow**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow execution on a time schedule (default every X minutes).  
  - Configuration: Interval set to minutes; can be adjusted as needed.  
  - Inputs: None (trigger node).  
  - Outputs: Passes empty data to next node to start email retrieval.  
  - Edge Cases: Misconfiguration may cause missed or excessive runs; time zone considerations apply.

- **Get Unread Emails from Inbox**  
  - Type: Gmail node  
  - Role: Retrieves all unread emails labeled "INBOX" using Gmail API.  
  - Configuration: Filters with labelIds=["INBOX"], readStatus="unread", operation="getAll", returnAll=true.  
  - Inputs: Trigger output.  
  - Outputs: List of unread emails with metadata and snippet.  
  - Credentials: Requires Gmail OAuth credentials configured in n8n.  
  - Edge Cases: API rate limits, auth token expiration, no unread emails (empty output).

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Splits the array of unread emails into individual items for per-email processing.  
  - Configuration: Default batch size (1 item per batch), enabling iterative handling.  
  - Inputs: List of unread emails.  
  - Outputs: Single email item per execution cycle.  
  - Edge Cases: Large volumes may cause long runtimes; batch size can be tuned.

---

#### 1.2 Email Deduplication Check

**Overview:**  
Prevents re-processing of emails already handled by checking their presence in the Supabase database.

**Nodes Involved:**  
- Get Email from Database  
- Check if Email in Database

**Node Details:**

- **Get Email from Database**  
  - Type: Supabase node  
  - Role: Queries Supabase `emails` table for an entry matching the current email's `id` (email_id).  
  - Configuration: Table `emails`, filter condition `email_id = current email id`.  
  - Inputs: Single email item from Loop Over Items.  
  - Outputs: Supabase query result (empty if email not processed).  
  - Credentials: Supabase API credentials with read access.  
  - Edge Cases: Network errors, incorrect keys, table access issues.

- **Check if Email in Database**  
  - Type: If node  
  - Role: Checks if the Supabase query result is empty (email not found).  
  - Configuration: Condition tests if result `isEmpty() == false` to direct flow.  
  - Inputs: Output of Supabase query node.  
  - Outputs:  
    - True branch (email is in DB): loops back to Loop Over Items (skip GPT).  
    - False branch (email not in DB): proceeds to GPT prompt preparation.  
  - Edge Cases: Expression evaluation errors, false positives/negatives in condition logic.

---

#### 1.3 Task Extraction via GPT

**Overview:**  
Prepares a prompt based on the email content and sends it to ChatGPT-4o to extract structured task details in JSON format.

**Nodes Involved:**  
- Prepare ChatGPT Prompt  
- Extract Task Detail from Email

**Node Details:**

- **Prepare ChatGPT Prompt**  
  - Type: Code node (JavaScript)  
  - Role: Constructs a GPT prompt embedding the email snippet, asking for JSON-formatted actionable task extraction.  
  - Configuration: Runs once per email item; prompt instructs GPT to output specific fields or null if no task.  
  - Key Expression: Uses `$('Loop Over Items').item.json.snippet` to inject email content.  
  - Inputs: Email item from Loop Over Items after passing deduplication.  
  - Outputs: JSON with single key `prompt` containing the full prompt text.  
  - Edge Cases: Missing snippet field, script errors, improper escaping.

- **Extract Task Detail from Email**  
  - Type: Langchain OpenAI node (ChatGPT-4o)  
  - Role: Sends prompt to OpenAI GPT-4o model, receives structured JSON response with task details.  
  - Configuration: Model set to `chatgpt-4o-latest`; messages contain the prompt; JSON output enabled.  
  - Inputs: Prompt from Prepare ChatGPT Prompt node.  
  - Outputs: Parsed JSON with task structure fields like task, description, due_date, estimated_minutes, deep_work.  
  - Credentials: Requires OpenAI API key with GPT-4o access.  
  - Edge Cases: API rate limiting, prompt format errors, invalid JSON response, network issues.

---

#### 1.4 Data Insertion to Supabase

**Overview:**  
Inserts or upserts the email metadata and GPT-extracted task details into the Supabase `emails` table, preventing duplicates.

**Nodes Involved:**  
- Insert Email Detail to Supabase

**Node Details:**

- **Insert Email Detail to Supabase**  
  - Type: HTTP Request node  
  - Role: Sends a POST request with JSON body to Supabase REST API to insert or update email/task record.  
  - Configuration:  
    - URL dynamically built from variable `Supabase_TaskManagement_URI` with conflict resolution on `email_id`.  
    - JSON body contains email id, subject, sender, received_at (converted ISO date), snippet body, GPT summary, deep work flag, deleted=false.  
    - Headers include API key and authorization bearer token from `Supabase_TaskManagement_ANON_KEY`.  
    - Uses `Prefer: resolution=ignore-conflict` to upsert.  
  - Inputs: Output from GPT node (task details) and email data from original email item.  
  - Outputs: Not used further (no output connected).  
  - Credentials: Supabase anon key with insert/update rights.  
  - Edge Cases: HTTP errors, auth failures, invalid data, race conditions causing conflicts.

---

#### 1.5 Control Flow and Looping

**Overview:**  
Manages the iterative processing of each email, conditional branching to avoid duplicate processing, and looping back to next email.

**Nodes Involved:**  
- Loop Over Items  
- Check if Email in Database (branches)  
- Insert Email Detail to Supabase (loops back)

**Node Details:**

- Loop Over Items node runs continuously until all unread emails are processed.  
- Check if Email in Database node directs flow: skip GPT for known emails or proceed for new.  
- After insertion, flow loops back to Loop Over Items for next email.  
- This design ensures continuous batch handling without missing or duplicating emails.

---

#### 1.6 Documentation and Instructions

**Overview:**  
A large sticky note node contains detailed workflow documentation, prerequisites, credentials, environment variables, and intended use.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Provides comprehensive documentation inside the workflow for user reference.  
  - Content includes:  
    - Workflow purpose and summary  
    - What it does step-by-step  
    - Prerequisites and setup instructions  
    - Required credentials table  
    - Environment variables and examples  
    - Scheduling details  
    - Intended use case explanation  
    - Output data fields stored in Supabase  
  - No inputs or outputs (informational only).  
  - Edge Cases: N/A

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                                  | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                                                      |
|----------------------------|----------------------------------|-------------------------------------------------|-------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Trigger Workflow           | Schedule Trigger                  | Initiates workflow on schedule                   | None                          | Get Unread Emails from Inbox    | Runs the workflow on a schedule (every X minutes) to check for new unread emails.                                                |
| Get Unread Emails from Inbox | Gmail                           | Fetches unread emails from Gmail inbox           | Trigger Workflow              | Loop Over Items                 | Fetches all unread emails from the inbox using Gmail API. These are the raw email inputs for analysis.                          |
| Loop Over Items             | SplitInBatches                   | Processes each email individually                 | Get Unread Emails from Inbox  | Get Email from Database, Prepare ChatGPT Prompt | Loops through each unread email individually, allowing per-item processing (dedup, GPT, insert).                                |
| Get Email from Database     | Supabase                        | Checks if email already processed                  | Loop Over Items (first branch) | Check if Email in Database      | Checks Supabase to see if this email has already been processed (by matching on email_id). Prevents duplicate GPT calls.         |
| Check if Email in Database  | If                             | Routes flow based on email presence in database  | Get Email from Database       | Loop Over Items (skip), Prepare ChatGPT Prompt (process) | If the email does not exist in Supabase, continue with GPT processing. Otherwise, skip this email.                              |
| Prepare ChatGPT Prompt      | Code                           | Formats email into GPT prompt                      | Check if Email in Database    | Extract Task Detail from Email  | Formats the email content into a prompt for GPT, asking it to extract a structured task in JSON format.                         |
| Extract Task Detail from Email | Langchain OpenAI (ChatGPT-4o) | Sends prompt to GPT to extract structured task   | Prepare ChatGPT Prompt        | Insert Email Detail to Supabase | Sends the email to ChatGPT-4o to extract task details like title, description, due date, duration, and focus level.             |
| Insert Email Detail to Supabase | HTTP Request                 | Inserts or upserts email and task data into Supabase | Extract Task Detail from Email | Loop Over Items                | Inserts the processed email and GPT-extracted task into the Supabase emails table. Uses upsert to avoid duplicates.             |
| Sticky Note                 | Sticky Note                    | Documentation and instructions                     | None                         | None                          | Contains detailed workflow documentation, prerequisites, credentials, environment variables, and intended use case information. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure the trigger interval to run every X minutes as desired (default minutes).  
   - Position it as the starting node.

2. **Add Gmail Node to Fetch Unread Emails**  
   - Type: Gmail  
   - Operation: getAll  
   - Filters: labelIds = ["INBOX"], readStatus = "unread"  
   - Return all emails (returnAll = true).  
   - Connect Trigger node output to this node's input.  
   - Configure Gmail OAuth credentials in n8n.

3. **Add SplitInBatches Node**  
   - Type: SplitInBatches  
   - Default batch size (1).  
   - Connect Gmail node output to this node input.  
   - This node enables processing emails one by one.

4. **Add Supabase Node to Check Email Existence**  
   - Type: Supabase  
   - Operation: get  
   - Table: `emails`  
   - Filter condition: `email_id = {{$json["id"]}}` (use current email id)  
   - Connect SplitInBatches output to this node input.  
   - Configure Supabase credentials with read access.

5. **Add If Node to Branch Flow**  
   - Type: If  
   - Condition: Check if Supabase query result is empty (`{{$json.isEmpty()}} == false`)  
   - True branch: Email exists → connect back to SplitInBatches node to skip processing.  
   - False branch: Email not exists → continue processing.  
   - Connect Supabase node output to If node input.

6. **On False Branch, Add Code Node to Prepare GPT Prompt**  
   - Type: Code (JavaScript)  
   - Mode: Run once per item  
   - Code Sample:
     ```javascript
     return {
       prompt: `You are an AI productivity assistant. Given the email below, extract any actionable task(s). Respond in this format:
       {
         "task": "Brief title of the task",
         "description": "Expanded detail if needed",
         "due_date": "2025-07-01" or null,
         "estimated_minutes": 30,
         "deep_work": true
       }
       If no actionable task exists, respond with null.

       Email:
       ${$('Loop Over Items').item.json.snippet}`
     };
     ```
   - Connect If node False output to this node input.

7. **Add Langchain OpenAI Node for GPT-4o Call**  
   - Type: Langchain OpenAI  
   - Model: `chatgpt-4o-latest`  
   - Messages: One message with content from previous node’s `prompt` key.  
   - JSON Output enabled to parse structured response.  
   - Connect Code node output to this node input.  
   - Configure OpenAI API key credential.

8. **Add HTTP Request Node to Insert Data into Supabase**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `{{$vars.Supabase_TaskManagement_URI + '/rest/v1/emails?on_conflict=email_id'}}`  
   - Headers:
     - `apikey`: `{{$vars.Supabase_TaskManagement_ANON_KEY}}`  
     - `Authorization`: `Bearer {{$vars.Supabase_TaskManagement_ANON_KEY}}`  
     - `Content-Type`: `application/json`  
     - `Prefer`: `resolution=ignore-conflict`  
   - Body (JSON):
     ```json
     {
       "email_id": {{$json.id}},
       "subject": {{$json.Subject}},
       "sender": {{$json.From}},
       "received_at": "{{ new Date(Number($json.internalDate)).toISOString() }}",
       "body": {{$json.snippet}},
       "gpt_summary": {{$json.message.content}},
       "requires_deep_work": {{$json.message.content.deep_work}},
       "deleted": false
     }
     ```
   - Connect GPT node output to this node input.  
   - Configure Supabase API anon key for authentication.

9. **Loop Back from HTTP Request to SplitInBatches**  
   - Connect HTTP Request node output back to SplitInBatches node input to continue processing next email.

10. **Add a Sticky Note Node (Optional)**  
    - Add detailed documentation, prerequisites, credentials, and environment variables as per your project needs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow automates email task extraction from Gmail to Supabase using GPT-4o, avoiding duplicate processing and saving structured task data for productivity workflows.                                                                                                                                                                                                                                                                                                                                              | Workflow purpose summary                                                                          |
| Required credentials include Gmail OAuth, OpenAI API key with GPT-4o access, and Supabase anon key with REST API permissions.                                                                                                                                                                                                                                                                                                                                                                                              | Credential requirements                                                                          |
| Environment variables to set in n8n or externally: `Supabase_TaskManagement_URI` (Supabase project URL), `Supabase_TaskManagement_ANON_KEY` (Supabase anon key) for REST API calls.                                                                                                                                                                                                                                                                                                                                         | Environment setup                                                                                |
| For Supabase, ensure the `emails` table has a unique constraint on `email_id` to prevent duplicates: `ALTER TABLE emails ADD CONSTRAINT unique_email_id UNIQUE (email_id);`.                                                                                                                                                                                                                                                                                                                                                | Database schema note                                                                             |
| Scheduling frequency can be adjusted to balance email processing latency and API usage.                                                                                                                                                                                                                                                                                                                                                                                                                                      | Workflow scheduling advice                                                                       |
| Intended for professionals to automate task extraction from emails efficiently, integrating with task management or calendar systems as a next step.                                                                                                                                                                                                                                                                                                                                                                        | Use case explanation                                                                            |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---