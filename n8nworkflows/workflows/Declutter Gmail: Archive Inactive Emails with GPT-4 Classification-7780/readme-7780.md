Declutter Gmail: Archive Inactive Emails with GPT-4 Classification

https://n8nworkflows.xyz/workflows/declutter-gmail--archive-inactive-emails-with-gpt-4-classification-7780


# Declutter Gmail: Archive Inactive Emails with GPT-4 Classification

### 1. Workflow Overview

This workflow automates the decluttering of a Gmail inbox by archiving emails that are deemed inactive or irrelevant based on AI classification using GPT-4. It targets read emails older than 45 days in the inbox and uses a LangChain AI agent to analyze each email’s content and metadata. The AI classifies whether the email requires action, extracts due dates if any, and determines if the email can be archived (marked as trashable). Emails identified as trashable are archived automatically.

**Target Use Cases:**  
- Automating email inbox management for users overwhelmed by old, read emails  
- Prioritizing actionable emails by AI triage  
- Reducing manual email archiving workload

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Initiates the workflow daily at 2 AM.  
- **1.2 Email Retrieval:** Lists all read emails in the inbox older than 45 days.  
- **1.3 Email Batching & Preparation:** Splits email list into manageable batches and prepares each email’s data for AI processing.  
- **1.4 AI Classification:** Uses a LangChain AI Agent powered by GPT-4 to classify each email with structured JSON output.  
- **1.5 Decision & Archival:** Checks AI output to decide whether to archive the email and performs the archival if applicable.  
- **1.6 Completion:** Marks the end of processing for each batch.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Automatically triggers the workflow at a scheduled time daily to ensure regular inbox maintenance.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Triggers at 2 AM every day (hour 2)  
    - Input/Output: No input; outputs trigger event to start workflow  
    - Edge Cases: Timezone considerations may affect actual trigger time; ensure server timezone matches expectations  
    - Notes: Can be customized for different intervals (e.g., hourly, weekly)

---

#### 1.2 Email Retrieval

- **Overview:**  
  Fetches all read emails in the Gmail inbox older than 45 days to identify candidates for archiving.

- **Nodes Involved:**  
  - List old inbox emails

- **Node Details:**  
  - **List old inbox emails**  
    - Type: Gmail node (Get All operation)  
    - Configuration:  
      - Filter: label:inbox, older_than:45d, readStatus: read  
      - Returns all matching emails (returnAll: true)  
      - Does not use simple mode to get full email data  
    - Input: Trigger from Schedule Trigger  
    - Output: List of email objects to be processed in batches  
    - Edge Cases:  
      - Gmail API quota limits or rate limits may cause failures  
      - Filter syntax must be correct to avoid empty results  
      - Emails with special labels or threads might behave unexpectedly  
    - Sticky Note: Explains how to modify search query to adjust email selection

---

#### 1.3 Email Batching & Preparation

- **Overview:**  
  Processes the large email list by splitting it into batches of 50 emails, then extracts and sets only the relevant fields needed for AI classification.

- **Nodes Involved:**  
  - Loop Over Emails  
  - Prep Email for AI

- **Node Details:**  
  - **Loop Over Emails**  
    - Type: SplitInBatches  
    - Configuration: batchSize = 50  
    - Input: List of emails from Gmail node  
    - Output: Sends one batch of emails downstream per execution  
    - Edge Cases: Batch size too large might cause timeouts; too small reduces efficiency  

  - **Prep Email for AI**  
    - Type: Set node  
    - Configuration: Assigns four fields from each email JSON:  
      - threadId (string)  
      - messageId (string)  
      - Subject (string)  
      - text (string) (email body content)  
    - Input: Each email item from batch  
    - Output: Clean, simplified JSON object for AI input  
    - Edge Cases: Missing fields in email JSON could cause incomplete data; expressions must resolve correctly

---

#### 1.4 AI Classification

- **Overview:**  
  An AI Agent node sends each email’s prepared data to GPT-4 with a prompt instructing it to classify the email based on presence of due dates, required actions, and archive eligibility. The output is parsed to ensure structured JSON.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Structured Output Parser  
  - Sticky Note (explains AI logic)

- **Node Details:**  
  - **AI Agent**  
    - Type: LangChain Agent  
    - Configuration:  
      - Text prompt instructs to extract due date, decide if action required, and set trashable flag  
      - Inputs used: threadId, messageId, subject, text  
      - Output format: JSON with fields threadId, messageId, requires_action (bool), due_date (YYYY-MM-DD or null), trashable (bool), reason (string)  
    - Input: Simplified email JSON from Prep Email for AI  
    - Output: Classified email JSON  
    - Edge Cases:  
      - AI generating invalid JSON or missing fields  
      - API rate limits or errors from OpenAI  
      - Latency/timeouts in AI response  
    - Uses OpenAI Chat Model node for GPT-4 processing  
    - Connected to Structured Output Parser for validation  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Configuration: Uses GPT-4o-mini model variant  
    - Input: Prompt from AI Agent  
    - Output: Raw AI response  
    - Edge Cases: API failures, network issues, model limits  

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Configuration: JSON schema validation for AI output, ensuring all required fields and data types are present  
    - Input: AI raw output  
    - Output: Parsed, validated JSON to AI Agent node  
    - Edge Cases: Parsing failures if AI output does not match schema  

---

#### 1.5 Decision & Archival

- **Overview:**  
  Uses an IF node to decide if an email should be archived based on the AI’s “trashable” flag. If true, archives the email by removing the INBOX label; otherwise, continues processing the next email.

- **Nodes Involved:**  
  - If - Should archived?  
  - Archive Email

- **Node Details:**  
  - **If - Should archived?**  
    - Type: IF node  
    - Configuration: Checks if AI output field `output.trashable` is true  
    - Input: AI Agent output  
    - Outputs:  
      - True branch: send email to Archive Email node  
      - False branch: continues looping for next email batch  
    - Edge Cases: Missing or malformed `trashable` field may cause incorrect routing  

  - **Archive Email**  
    - Type: Gmail node (Modify Labels operation)  
    - Configuration: Removes 'INBOX' label from the email thread identified by `output.threadId` to archive the email  
    - Input: True branch from IF node  
    - Output: Sends back to Loop Over Emails to continue processing  
    - Edge Cases:  
      - Permissions errors if OAuth token lacks label modification rights  
      - API rate limits or failures  
      - Thread ID invalid or email already archived  

---

#### 1.6 Completion

- **Overview:**  
  Marks the completion of processing for a batch of emails. Can be used for logging or workflow termination.

- **Nodes Involved:**  
  - Completed (NoOp node)

- **Node Details:**  
  - **Completed**  
    - Type: No Operation (NoOp) node  
    - Configuration: None  
    - Input: From Loop Over Emails after batch completion  
    - Output: None (end of the chain)  
    - Edge Cases: None

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                         | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                   |
|-------------------------|----------------------------------|---------------------------------------|--------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger                 | Triggers the workflow daily at 2 AM  | (Start)                  | List old inbox emails    | ### 📆 Schedule the Workflow<br>• The "Schedule Trigger" by default, it's 2am everyday.<br>• Modify it to your desired interval (e.g., daily, hourly). |
| List old inbox emails    | Gmail (Get All)                  | Retrieves read inbox emails >45 days  | Schedule Trigger          | Loop Over Emails         | ### 📥 Gmail: List Old Emails<br>• Fetches all read emails in inbox older than 45 days.<br>• Modify filter query to change targeted emails.                  |
| Loop Over Emails         | SplitInBatches                  | Processes emails in batches of 50     | List old inbox emails, If - Should archived?, Archive Email | Prep Email for AI, Completed |                                                                                                              |
| Prep Email for AI        | Set                            | Extracts and prepares email data for AI | Loop Over Emails          | AI Agent                |                                                                                                              |
| AI Agent                | LangChain Agent                 | Classifies emails with GPT-4          | Prep Email for AI, OpenAI Chat Model, Structured Output Parser | If - Should archived?    | ### 🧠 AI Email Classifier<br>• Sends email content to OpenAI with instructions to classify due date, action needed, and trashable.<br>• If trashable true, email can be archived. |
| OpenAI Chat Model        | LangChain OpenAI Chat Model    | GPT-4 model to process AI prompt      | AI Agent                  | AI Agent                |                                                                                                              |
| Structured Output Parser | LangChain Output Parser Structured | Validates and parses AI JSON output | OpenAI Chat Model         | AI Agent                |                                                                                                              |
| If - Should archived?    | IF                            | Decides if email is trashable         | AI Agent                  | Archive Email, Loop Over Emails |                                                                                                              |
| Archive Email            | Gmail (Modify Labels)           | Archives email by removing INBOX label | If - Should archived?     | Loop Over Emails         |                                                                                                              |
| Completed                | No Operation                   | Marks completion of batch processing  | Loop Over Emails          | (End)                   |                                                                                                              |
| Sticky Note              | Sticky Note                    | Explains AI classification logic      |                          |                         | ### 🧠 AI Email Classifier<br>• The "AI Agent" sends email content to OpenAI with a clear instruction set to classify the email (due date, action needed, trashable).<br>• If "trashable" is true, the email can be archived or trashed. |
| Sticky Note1             | Sticky Note                    | Explains Gmail email listing filter   |                          |                         | ### 📥 Gmail: List Old Emails<br>• This Gmail node fetches all emails in the inbox that are read and older than 45 days.<br>• You can modify the search filter to change which emails are targeted (e.g., change older_than:45d to a different duration). |
| Sticky Note2             | Sticky Note                    | Explains scheduling node               |                          |                         | ### 📆 Schedule the Workflow<br>• The "Schedule Trigger" by default, it's 2am everyday.<br>• Modify it to your desired interval (e.g., daily, hourly) to control how often the workflow runs. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 2 AM (triggerAtHour: 2)

2. **Create Gmail Node to List Emails:**  
   - Type: Gmail  
   - Operation: Get All  
   - Filters:  
     - labelIds: ["INBOX"]  
     - readStatus: "read"  
     - q: "label:inbox older_than:45d"  
   - Return All: true  
   - Connect Schedule Trigger output to this node's input

3. **Create SplitInBatches Node:**  
   - Type: SplitInBatches  
   - Batch Size: 50  
   - Connect Gmail node output to this node's input

4. **Create Set Node to Prepare Email Data:**  
   - Type: Set  
   - Assignments:  
     - threadId = {{$json.threadId}}  
     - messageId = {{$json.id}}  
     - Subject = {{$json.subject}}  
     - text = {{$json.text}}  
   - Connect SplitInBatches output to this node's input

5. **Set Up LangChain OpenAI Chat Model Node:**  
   - Type: LangChain OpenAI Chat Model  
   - Model: gpt-4o-mini (or GPT-4 equivalent)  
   - Configure credentials for OpenAI API (API key with GPT-4 access)  

6. **Create LangChain Structured Output Parser Node:**  
   - Type: LangChain Output Parser Structured  
   - Input Schema (JSON):  
     ```json
     {
       "title": "SimpleEmailTriage",
       "type": "object",
       "required": ["threadId", "messageId", "requires_action", "due_date", "trashable", "reason"],
       "properties": {
         "threadId": { "type": "string" },
         "messageId": { "type": "string" },
         "requires_action": { "type": "boolean" },
         "due_date": { "type": ["string", "null"], "pattern": "^\\d{4}-\\d{2}-\\d{2}$" },
         "trashable": { "type": "boolean" },
         "reason": { "type": "string" }
       },
       "additionalProperties": false
     }
     ```

7. **Create LangChain AI Agent Node:**  
   - Type: LangChain Agent  
   - Prompt Type: Define  
   - Prompt Text:  
     ```
     You are an assistant classifying Gmail emails. Output JSON only, no prose.

     Only use these inputs:
     threadId: {{ $json.threadId }}
     messageId: {{ $json.messageId }}
     subject: {{ $json.subject }}
     text: {{ $json.text }}

     Assume today's date is {{ $now.toFormat('yyyy-LL-dd') }}.

     Tasks:
     1) Extract a due date from the text if present, format YYYY-MM-DD. If none, use null.
     2) Decide if the email requires action. Action means at least one of:
        - It is an invoice or bill that expects payment
        - It explicitly requests a reply or a decision
        - It clearly contains a task or meeting to schedule
     3) Set trashable = true if any of these are true:
        - due_date is null
        - due_date is earlier than today
        - requires_action is false
        Otherwise trashable = false.

     Return JSON with this exact structure:
     {
       "threadId": "<string>",
       "messageId": "<string>",
       "requires_action": true | false,
       "due_date": "YYYY-MM-DD" | null,
       "trashable": true | false,
       "reason": "<short explanation>"
     }
     ```
   - Connect the Set node's output as the main input  
   - Connect OpenAI Chat Model as AI language model input  
   - Connect Structured Output Parser as AI output parser  

8. **Create IF Node to Check Archival Condition:**  
   - Type: IF  
   - Condition: Check if `{{$json.output.trashable}}` is true (boolean equals true)  
   - Connect AI Agent output to IF node input

9. **Create Gmail Node to Archive Email:**  
   - Type: Gmail  
   - Operation: Remove Labels  
   - Resource: thread  
   - threadId: `{{$json.output.threadId}}`  
   - Labels to remove: ["INBOX"]  
   - Connect IF node true output to this node input

10. **Connect IF node false output and Archive Email node output back to SplitInBatches node:**  
    - This creates a loop to process next batch or email

11. **Create NoOp Node named "Completed":**  
    - Connect SplitInBatches main output (after all batches processed) to this node  
    - Marks the end of each batch processing

12. **Credential Setup:**  
    - Gmail node: Authenticate with OAuth2 for Gmail API with read and modify label permissions  
    - LangChain OpenAI Chat Model: Set OpenAI API key with GPT-4 access

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                      |
|----------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The AI prompt and logic for email classification are designed to balance archiving old emails while preserving actionable ones. | See Sticky Note on AI Email Classifier node         |
| Gmail API quota limits and permissions may affect workflow execution; ensure sufficient API quota and correct OAuth scopes.    | Gmail API documentation                             |
| The workflow’s schedule is flexible; adjust the Schedule Trigger node to desired frequency.  | Sticky Note2 on scheduling                          |
| The search query in Gmail node can be customized to target different email subsets (e.g., unread, specific labels).             | Sticky Note1 on Gmail email listing filter          |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.