Automatic Email Categorization and Labeling with Gmail and GPT-4o

https://n8nworkflows.xyz/workflows/automatic-email-categorization-and-labeling-with-gmail-and-gpt-4o-5820


# Automatic Email Categorization and Labeling with Gmail and GPT-4o

---
### 1. Workflow Overview

This workflow automates the categorization and labeling of unread Gmail emails using AI (GPT-4o). It targets users who want to prioritize their inbox by automatically classifying incoming emails into four categories: Action, Informational, Receipt, and Spam. The workflow fetches unread emails periodically, extracts relevant fields, sends email content to an AI agent for classification and summarization, and then applies Gmail labels accordingly. It also marks emails as read or deletes them based on classification.

Logical blocks include:

- **1.1 Trigger and Email Fetching:** Scheduled trigger initiates the workflow hourly and fetches all unread Gmail messages.
- **1.2 Data Preparation:** Extracts and formats necessary email fields for AI processing.
- **1.3 AI Classification and Summarization:** Uses GPT-4o to classify emails into predefined categories and summarize content.
- **1.4 Output Parsing:** Parses AI JSON response for structured data extraction.
- **1.5 Email Labeling and Actions:** Routes based on classification to add Gmail labels, mark emails as read, or delete them.
- **1.6 Error Handling and Fallback:** Utilizes fallback AI model GPT-3.5-turbo if the primary AI agent fails.
- **1.7 Setup and Documentation Notes:** Sticky notes provide setup instructions, customization tips, and labeling guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Email Fetching

- **Overview:**  
  This block triggers the workflow every hour and retrieves all unread emails from Gmail to process.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get All Unread Messages

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts workflow execution on a regular interval (hourly).  
    - Configuration: Interval set to every 1 hour.  
    - Input: None  
    - Output: Triggers downstream nodes.  
    - Edge Cases: Misconfiguration could cause frequent or missed runs; time zone considerations may apply.

  - **Get All Unread Messages**  
    - Type: Gmail node  
    - Role: Fetches all unread emails from the connected Gmail account.  
    - Configuration: Filters set to unread emails, retrieves all available messages.  
    - Input: Trigger from Schedule Trigger  
    - Output: Raw email data including body, sender, message ID, thread ID, read status, etc.  
    - Version-specific: Requires Gmail OAuth2 credentials connected beforehand.  
    - Edge Cases: Gmail API rate limits, quota exceeded, authentication errors, large inboxes causing delays.

---

#### 2.2 Data Preparation

- **Overview:**  
  Extracts and prepares essential fields from the raw email data to format it for AI processing.

- **Nodes Involved:**  
  - Edit Fields

- **Node Details:**

  - **Edit Fields**  
    - Type: Set node  
    - Role: Creates simplified structured data for AI consumption by extracting `id`, `threadId`, the main text (either plain text or HTML), and sender name (`from`).  
    - Configuration:  
      - `id` assigned from email JSON `id`.  
      - `threadId` assigned from email JSON `threadId`.  
      - `text` assigned from `text` if available, else from `html`.  
      - `from` assigned from the first sender name in the `from.value` array.  
    - Input: Raw email JSON from Gmail node  
    - Output: Cleaned JSON with minimal fields for AI processing.  
    - Edge Cases: Missing `text` and `html` fields, malformed sender info, null values.

---

#### 2.3 AI Classification and Summarization

- **Overview:**  
  Sends the prepared email content to an AI language model (GPT-4o) tasked with classifying the email into one of four categories and extracting a concise summary.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model (GPT-4o)  
  - OpenAI Fall Back Model (GPT-3.5-turbo)  
  - Structured Output Parser

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat node  
    - Role: Provides GPT-4o model interface for AI Agent.  
    - Configuration: Model set to `gpt-4o`.  
    - Input: Prompt passed from AI Agent node.  
    - Output: AI-generated classification and summary text.  
    - Requirements: Valid OpenAI API key with access to GPT-4o.  
    - Edge Cases: API rate limits, network timeouts, invalid API keys.

  - **OpenAI Fall Back Model**  
    - Type: Langchain OpenAI Chat node  
    - Role: Provides fallback GPT-3.5-turbo model for AI Agent in case GPT-4o fails.  
    - Configuration: Model set to `gpt-3.5-turbo`.  
    - Input: Prompt passed from AI Agent node on failure.  
    - Output: AI-generated classification and summary text fallback.  
    - Edge Cases: Same as OpenAI Chat Model.

  - **AI Agent**  
    - Type: Langchain Agent node  
    - Role: Orchestrates AI prompt generation and model calls. Enforces strict JSON output format.  
    - Configuration: Prompt defines classification rules and required JSON output with fields: `label`, `summary`, `id`, `threadId`.  
    - Important expressions: Uses expressions to inject email `id`, `threadId`, and content `text` into the prompt dynamically.  
    - Input: Cleaned email data from Edit Fields; AI model nodes connected as primary and fallback models.  
    - Output: Raw AI response text.  
    - Error Handling: Configured to continue workflow even if errors occur (retry enabled).  
    - Edge Cases: AI generating malformed JSON, prompt injection issues, API failures.

  - **Structured Output Parser**  
    - Type: Langchain Output Parser node  
    - Role: Parses AI Agent's raw string output into structured JSON according to a provided JSON schema.  
    - Configuration: Schema requires `label` (string), `summary` (string), `id` (string), `threadId` (string, optional).  
    - Input: AI Agent output.  
    - Output: Parsed JSON object with classification and summary.  
    - Edge Cases: Parsing failures if AI output deviates from expected JSON format.

---

#### 2.4 Email Labeling and Actions

- **Overview:**  
  Routes emails based on classification labels to add appropriate Gmail labels, mark emails as read, or delete threads if classified as spam.

- **Nodes Involved:**  
  - Switch  
  - Add label "Action" to email  
  - Add label "Informational" to email  
  - Add label "Receipt" to email  
  - Delete email  
  - Mark email as read

- **Node Details:**

  - **Switch**  
    - Type: Switch node  
    - Role: Routes emails based on `label` field from AI output.  
    - Configuration: Four outputs for `Action`, `Informational`, `Receipt`, and fallback (`extra` for all other labels).  
    - Input: Parsed AI output JSON.  
    - Output: Routes to label adding nodes or deletion node.  
    - Edge Cases: Unexpected labels not matching any case go to fallback.

  - **Add label "Action" to email**  
    - Type: Gmail node  
    - Role: Adds Gmail labels `Action` and `IMPORTANT` to the email.  
    - Configuration: Uses Gmail label IDs to add labels based on email `id`.  
    - Input: Switch output for `Action`.  
    - Output: Passes to Mark email as read node.  
    - Requirements: Labels `Action` and `IMPORTANT` must exist in Gmail.  
    - Edge Cases: Missing label IDs, permission issues.

  - **Add label "Informational" to email**  
    - Type: Gmail node  
    - Role: Adds Gmail label `Informational` to the email.  
    - Configuration: Uses Gmail label ID.  
    - Input: Switch output for `Informational`.  
    - Output: Passes to Mark email as read node.  
    - Edge Cases: Missing label ID.

  - **Add label "Receipt" to email**  
    - Type: Gmail node  
    - Role: Adds Gmail label `Receipt` to the email.  
    - Configuration: Uses Gmail label ID.  
    - Input: Switch output for `Receipt`.  
    - Output: Passes to Mark email as read node.  
    - Edge Cases: Missing label ID.

  - **Delete email**  
    - Type: Gmail node  
    - Role: Deletes the entire email thread for emails categorized as spam or unmatched labels (fallback).  
    - Configuration: Deletes by `threadId`.  
    - Input: Switch fallback output.  
    - Output: Terminal.  
    - Edge Cases: Deletion failures, Gmail permission errors.

  - **Mark email as read**  
    - Type: Gmail node  
    - Role: Marks the processed email as read to prevent reprocessing.  
    - Configuration: Marks by message `id`.  
    - Input: Output from all label adding nodes.  
    - Output: Terminal.  
    - Edge Cases: API errors, message not found.

---

#### 2.5 Setup and Documentation Notes

- **Overview:**  
  Provides important user instructions and customization tips via sticky notes embedded in the workflow canvas.

- **Nodes Involved:**  
  - Sticky Note (Schedule Trigger explanation)  
  - Sticky Note1 (Setup required instructions)  
  - Sticky Note2 (Prompt customization hints)  
  - Sticky Note3 (Label addition guidance)

- **Node Details:**

  - Sticky Note (Schedule Trigger)  
    - Content: Explains the schedule trigger runs hourly and can be adjusted.  
    - Role: User guidance.

  - Sticky Note1  
    - Content: Instructions to connect Gmail OAuth2, add OpenAI API key, and create required Gmail labels.  
    - Role: Critical setup instructions to ensure workflow functions.

  - Sticky Note2  
    - Content: Suggests customizing AI prompt to detect additional labels like Meeting or Newsletter.  
    - Role: Encourages workflow adaptability.

  - Sticky Note3  
    - Content: Advises adding more Switch conditions and Gmail actions for new labels.  
    - Role: Extension guidance.

---

### 3. Summary Table

| Node Name                      | Node Type                           | Functional Role                        | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                         |
|--------------------------------|-----------------------------------|-------------------------------------|------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger                  | Initiates workflow hourly           | None                         | Get All Unread Messages         | ### 💡 Schedule Trigger: This runs every hour. You can change the interval here depending on how often you want your Gmail to be processed. |
| Get All Unread Messages        | Gmail                            | Fetches unread Gmail emails         | Schedule Trigger             | Edit Fields                    | ### ⚠️ Setup Required: Connect Gmail OAuth2, add OpenAI API key, create labels: Action, Informational, Spam, Receipt |
| Edit Fields                   | Set                             | Prepares email data for AI          | Get All Unread Messages      | AI Agent                      |                                                                                                   |
| AI Agent                      | Langchain Agent                 | Classifies and summarizes email via GPT-4o | Edit Fields                 | Switch                        | ### 💡 Customize Classification Logic: Change prompt to detect other labels like Meeting, Newsletter |
| OpenAI Chat Model             | Langchain OpenAI Chat           | GPT-4o AI model for classification  | AI Agent (ai_languageModel)  | AI Agent                      |                                                                                                   |
| OpenAI Fall Back Model        | Langchain OpenAI Chat           | GPT-3.5 Turbo fallback AI model     | AI Agent (ai_languageModel)  | AI Agent                      |                                                                                                   |
| Structured Output Parser      | Langchain Output Parser         | Parses AI JSON response             | AI Agent                    | Switch                        |                                                                                                   |
| Switch                       | Switch                         | Routes emails based on classification | Structured Output Parser     | Add label "Action", "Informational", "Receipt", Delete email | ### 💡 Add More Labels: Add conditions here to detect Meeting, Newsletter, etc.                    |
| Add label "Action" to email   | Gmail                          | Adds Action and IMPORTANT labels    | Switch (Action output)       | Mark email as read            |                                                                                                   |
| Add label "Informational" to email | Gmail                          | Adds Informational label             | Switch (Informational output) | Mark email as read            |                                                                                                   |
| Add label "Receipt" to email  | Gmail                          | Adds Receipt label                  | Switch (Receipt output)      | Mark email as read            |                                                                                                   |
| Delete email                 | Gmail                          | Deletes email thread (Spam/fallback) | Switch (fallback output)     | None                         |                                                                                                   |
| Mark email as read           | Gmail                          | Marks emails as read after labeling | Add label nodes              | None                         |                                                                                                   |
| Sticky Note                  | Sticky Note                    | Schedule trigger explanation         | None                        | None                         | ### 💡 Schedule Trigger: This runs every hour...                                                  |
| Sticky Note1                 | Sticky Note                    | Setup instructions                   | None                        | None                         | ### ⚠️ Setup Required: Connect Gmail OAuth2...                                                   |
| Sticky Note2                 | Sticky Note                    | Prompt customization advice          | None                        | None                         | ### 💡 Customize Classification Logic...                                                         |
| Sticky Note3                 | Sticky Note                    | Adding new labels guidance            | None                        | None                         | ### 💡 Add More Labels...                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set to execute every 1 hour (adjustable).  
   - This will start the workflow periodically.

2. **Add Gmail node "Get All Unread Messages":**  
   - Operation: Get All  
   - Filter: Read Status = Unread  
   - Return All: true  
   - Connect Schedule Trigger → Get All Unread Messages.  
   - Connect Gmail account with OAuth2 credentials.

3. **Add Set node "Edit Fields":**  
   - Assign fields:  
     - `id` = `{{$json["id"]}}`  
     - `threadId` = `{{$json["threadId"]}}`  
     - `text` = `{{$json["text"] ? $json["text"] : $json["html"]}}`  
     - `from` = `{{$json["from"].value[0].name}}`  
   - Connect Get All Unread Messages → Edit Fields.

4. **Add Langchain OpenAI Chat nodes:**  
   - Primary: "OpenAI Chat Model" with model `gpt-4o`  
   - Fallback: "OpenAI Fall Back Model" with model `gpt-3.5-turbo`  
   - Provide OpenAI API credentials with access to these models.

5. **Add Langchain Agent node "AI Agent":**  
   - Use the following prompt (customizable) instructing strict JSON output with fields `label`, `summary`, `id`, `threadId`.  
   - Connect input from Edit Fields.  
   - Set AI language models: primary to OpenAI Chat Model, fallback to OpenAI Fall Back Model.  
   - Set error handling to continue on failure and enable retry.

6. **Add Langchain Output Parser node "Structured Output Parser":**  
   - Schema Type: Manual  
   - Input schema specifying `label` (string), `summary` (string), `id` (string), `threadId` (string, optional).  
   - Connect AI Agent output → Structured Output Parser.

7. **Add Switch node "Switch":**  
   - Add rules for label:  
     - "Action" → output 1  
     - "Informational" → output 2  
     - "Receipt" → output 3  
     - Default/fallback → output 4  
   - Connect Structured Output Parser → Switch.

8. **Add Gmail nodes to add labels:**  
   - "Add label 'Action' to email": add labels `Action`, `IMPORTANT` by message ID.  
   - "Add label 'Informational' to email": add label `Informational` by message ID.  
   - "Add label 'Receipt' to email": add label `Receipt` by message ID.  
   - Connect Switch outputs accordingly.

9. **Add Gmail node "Delete email":**  
   - Operation: Delete thread  
   - Use `threadId` from AI output.  
   - Connect Switch fallback output → Delete email.

10. **Add Gmail node "Mark email as read":**  
    - Operation: Mark as read by message ID.  
    - Connect all label-adding nodes → Mark email as read.

11. **Create all Gmail labels prior to running:**  
    - `Action`, `Informational`, `Spam`, `Receipt` labels must exist.  
    - Obtain Gmail label IDs and set in respective Gmail nodes.

12. **Add sticky notes for documentation (optional):**  
    - Add notes explaining schedule, setup, prompt customization, and label addition guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                  |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Connect Gmail using OAuth2 and add OpenAI API key before activating the workflow.                    | Setup instructions from Sticky Note1                            |
| Create Gmail labels: Action, Informational, Spam, Receipt to match classification outputs.           | Gmail label setup requirement                                   |
| Customize AI prompt to detect additional categories like Meeting or Newsletter for better sorting.   | Sticky Note2 — prompt customization advice                      |
| Add corresponding Switch conditions and Gmail actions to handle new labels for extensibility.       | Sticky Note3 — workflow extension tips                          |
| Workflow runs every hour by default; adjust Schedule Trigger interval to change frequency.            | Sticky Note regarding Schedule Trigger frequency                |

---

**Disclaimer:**  
This text is generated from an automated n8n workflow. It complies strictly with content policies and contains no illegal or offensive material. All data processed is legal and public.