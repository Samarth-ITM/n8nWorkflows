Smart Gmail Auto-Labeler with Gemini AI & Sender History

https://n8nworkflows.xyz/workflows/smart-gmail-auto-labeler-with-gemini-ai---sender-history-11518


# Smart Gmail Auto-Labeler with Gemini AI & Sender History

### 1. Workflow Overview

This workflow automates Gmail email classification and labeling by leveraging Google Gemini AI and sender email history. It targets users aiming to maintain a zero-inbox by automatically categorizing incoming unread emails into predefined Gmail labels based on content and sender behavior. The workflow logically divides into these blocks:

- **1.1 Input Reception:** Watches Gmail inbox for new unread emails, batching them for processing.
- **1.2 Label Existence Check:** Skips emails that already have AI-assigned labels to avoid duplicate work.
- **1.3 AI-Driven Classification:** Uses sender history and AI tools to determine the most appropriate label.
- **1.4 Label Application & Archiving:** Applies the selected label and archives the email by removing it from the inbox.
- **1.5 Setup & Utility:** Provides manual triggers, label ID retrieval, and instructions for initial setup and maintenance.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block continuously polls Gmail every minute to detect unread emails and prepares them for processing in manageable batches.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Loop Over Items  

- **Node Details:**

  - **Gmail Trigger**  
    - *Type:* Gmail Trigger node (polling trigger)  
    - *Configuration:* Polls Gmail every minute for unread emails; does not use simple mode to access detailed email data.  
    - *Key Expressions:* None besides filter for unread (`readStatus: "unread"`).  
    - *Connections:* Outputs to Loop Over Items node.  
    - *Edge Cases:* Potential Gmail API rate limits, OAuth token expiration, large volumes of unread emails causing delays.

  - **Loop Over Items**  
    - *Type:* Split In Batches node  
    - *Configuration:* Splits incoming emails into individual items for sequential processing.  
    - *Connections:* Primary output triggers the label existence check block; secondary output triggers classification path.  
    - *Edge Cases:* Extremely large batches may slow down processing; no batch size explicitly set (defaults apply).

#### 1.2 Label Existence Check

- **Overview:** Determines if an email already carries an AI-assigned label to avoid redundant classification.

- **Nodes Involved:**  
  - Check Label Existence (Code node)  
  - If Not Assigned (If condition node)  

- **Node Details:**

  - **Check Label Existence**  
    - *Type:* Code node (JavaScript)  
    - *Function:* Checks current email labels against a predefined map of AI labels to see if labeling is already done.  
    - *Config Highlights:* Contains a hardcoded mapping between Gmail label IDs and label names; returns a boolean flag and matched label if present.  
    - *Key Expressions:* Uses `$input.first().json.labelIds` to find existing labels.  
    - *Edge Cases:* Requires manual update of label IDs to match user setup; missing or changed label IDs cause false negatives.

  - **If Not Assigned**  
    - *Type:* If node  
    - *Function:* Passes execution only if `hasAILabelAssigned` is false.  
    - *Edge Cases:* Strict boolean comparison; misconfiguration in previous node affects flow.

#### 1.3 AI-Driven Classification

- **Overview:** Uses Google Gemini AI combined with sender history and label example fetching to decide the correct label for each email.

- **Nodes Involved:**  
  - Email Classifier Agent (LangChain Agent node)  
  - Get Emails By Sender Email (Gmail Tool node)  
  - Get Emails By Label (Gmail Tool node)  
  - Google Gemini Chat Model (LangChain LM node)  
  - Structured Output Parser (LangChain Output Parser node)  

- **Node Details:**

  - **Email Classifier Agent**  
    - *Type:* LangChain Agent node  
    - *Function:* Core AI logic for classification; runs a stepwise process calling tools to fetch sender history and label examples, and applies classification rules.  
    - *Configuration:* Uses a detailed system message describing label definitions, classification rules (including the 80% majority bias), and expected JSON output format.  
    - *Key Expressions:* Accesses email subject and body from the currently processed item; calls tools "Get Emails By Sender Email" and "Get Emails By Label" as needed.  
    - *Edge Cases:* AI model errors, lack of prior sender history, ambiguous emails, tool call failures, or unexpected output formats.

  - **Get Emails By Sender Email**  
    - *Type:* Gmail Tool node  
    - *Function:* Fetches up to 10 previous emails from the sender before the current email date to provide context.  
    - *Config Highlights:* Filters by sender address and received before current email date; excludes spam/trash; returns subject, body, and prior labels.  
    - *Edge Cases:* Gmail API limits; sender with no history; latency in fetching emails.

  - **Get Emails By Label**  
    - *Type:* Gmail Tool node  
    - *Function:* Fetches up to 10 emails from a given label category to provide example data for uncertain classifications.  
    - *Config Highlights:* Uses label name input from AI agent context; excludes spam/trash.  
    - *Edge Cases:* No emails found under label; incorrect label name passed; API errors.

  - **Google Gemini Chat Model**  
    - *Type:* LangChain Google Gemini language model node  
    - *Function:* Provides the underlying AI reasoning engine for the agent.  
    - *Credentials:* Requires Google PaLM API credentials.  
    - *Edge Cases:* API quota or authentication failures; latency.

  - **Structured Output Parser**  
    - *Type:* LangChain Output Parser node  
    - *Function:* Parses AI agent raw output into structured JSON with a single `label` field.  
    - *Config Highlights:* Expects JSON schema `{ "label": "The Selected Label Name" }`.  
    - *Edge Cases:* Parsing errors if AI output deviates from expected format.

#### 1.4 Label Application & Archiving

- **Overview:** Applies the AI-determined Gmail label to the email thread and archives it by removing the INBOX label, effectively cleaning the inbox.

- **Nodes Involved:**  
  - If not None (If node)  
  - Convert Label to Label ID (Code node)  
  - Add label to thread (Gmail node)  
  - Remove Inbox label from thread (Gmail node)  

- **Node Details:**

  - **If not None**  
    - *Type:* If node  
    - *Function:* Continues only if AI returned a label other than "None".  
    - *Edge Cases:* Case-sensitive comparison; must handle exact string.

  - **Convert Label to Label ID**  
    - *Type:* Code node (JavaScript)  
    - *Function:* Maps the AI label name to the corresponding Gmail label ID required for Gmail API calls.  
    - *Config Highlights:* Contains a hardcoded dictionary mapping label names to Gmail label IDs; returns Gmail label ID or null.  
    - *Edge Cases:* Requires manual update with user-specific label IDs; missing mappings lead to null output and failure to apply label.

  - **Add label to thread**  
    - *Type:* Gmail node  
    - *Function:* Adds the selected Gmail label to the email thread.  
    - *Config Highlights:* Uses threadId from the current item; applies labelIds from previous node.  
    - *Edge Cases:* Gmail API errors, invalid label ID, threadId missing.

  - **Remove Inbox label from thread**  
    - *Type:* Gmail node  
    - *Function:* Removes the INBOX label to archive email after labeling.  
    - *Config Highlights:* Uses threadId from the first message in the thread; removes "INBOX" label.  
    - *Edge Cases:* Email already archived; API failures.

#### 1.5 Setup & Utility

- **Overview:** Provides manual trigger for testing, retrieves Gmail label IDs needed for mappings, and includes multiple sticky notes for user instructions, setup, and workflow documentation.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get many messages (Gmail node)  
  - Get Labels Info (Gmail node)  
  - Various Sticky Notes  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger node  
    - *Function:* Allows manual execution for testing and setup.  
    - *Connections:* Triggers "Get many messages".

  - **Get many messages**  
    - *Type:* Gmail node  
    - *Function:* Fetches up to 10 messages from Gmail for testing or setup purposes.  
    - *Edge Cases:* API limits; manual operation only.

  - **Get Labels Info**  
    - *Type:* Gmail node  
    - *Function:* Retrieves all Gmail labels and their IDs, enabling users to update label mappings in code nodes.  
    - *Edge Cases:* Requires correct OAuth credentials; manual inspection needed.

  - **Sticky Notes**  
    - Provide detailed setup instructions, workflow overview, AI classification explanation, tool descriptions, label application notes, label ID retrieval steps, and trigger details.  
    - Facilitate user understanding and correct configuration.

---

### 3. Summary Table

| Node Name                 | Node Type                           | Functional Role                                     | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                                  |
|---------------------------|-----------------------------------|----------------------------------------------------|--------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------|
| Gmail Trigger             | Gmail Trigger                     | Poll unread Gmail emails every minute              |                                | Loop Over Items                 | "Polls Gmail every 1 minute\nOnly fetches UNREAD emails\n⚙️ To change frequency: Edit pollTimes → mode"    |
| Loop Over Items           | Split In Batches                  | Processes each email individually                   | Gmail Trigger                  | Check Label Existence, Email Classifier Agent (secondary) |                                                                                                              |
| Check Label Existence     | Code                             | Checks if email already has an AI-assigned label   | Loop Over Items                | If Not Assigned                | "Check Label Existence node:\n- Looks at email's current labels\n- Checks if any AI label exists\n- Returns flag\n⚠️ Update labelMap with your label IDs!" |
| If Not Assigned           | If                               | Continues only if email is not already labeled     | Check Label Existence          | Email Classifier Agent          |                                                                                                              |
| Email Classifier Agent    | LangChain Agent                  | AI classification logic using sender history/tools | If Not Assigned                | If not None                    | "AI CLASSIFICATION ENGINE\nStepwise AI logic with sender history and label examples.\nReturns JSON label only." |
| Get Emails By Sender Email| Gmail Tool                      | Fetches previous emails from sender                 | Email Classifier Agent (tool) | Email Classifier Agent          | "Fetches up to 10 previous emails from same sender before current email date."                              |
| Get Emails By Label       | Gmail Tool                      | Fetches emails by label category for comparison     | Email Classifier Agent (tool) | Email Classifier Agent          | "Fetches up to 10 emails from a specific label category.\nUsed when sender history insufficient."           |
| Google Gemini Chat Model  | LangChain LM (Google Gemini)    | AI language model used by the classification agent | Email Classifier Agent (AI LM) | Email Classifier Agent          |                                                                                                              |
| Structured Output Parser  | LangChain Output Parser          | Parses AI output to JSON                             | Email Classifier Agent (parser)| Email Classifier Agent          |                                                                                                              |
| If not None               | If                               | Continues only if AI label is not "None"            | Email Classifier Agent          | Convert Label to Label ID       |                                                                                                              |
| Convert Label to Label ID | Code                             | Maps AI label name to Gmail label ID                 | If not None                   | Add label to thread            | "Maps AI label names to Gmail label IDs.\n⚠️ Update mapping with your label IDs!"                          |
| Add label to thread       | Gmail                            | Applies label to email thread                         | Convert Label to Label ID      | Remove Inbox label from thread  | "Applies the label to the email thread."                                                                    |
| Remove Inbox label from thread | Gmail                      | Removes INBOX label to archive the email             | Add label to thread            | Loop Over Items                | "Archives email by removing INBOX label."                                                                    |
| When clicking ‘Execute workflow’ | Manual Trigger            | Manual start for testing and setup                   |                                | Get many messages              |                                                                                                              |
| Get many messages         | Gmail                            | Fetches sample messages for setup/testing            | When clicking ‘Execute workflow’ | Loop Over Items               |                                                                                                              |
| Get Labels Info           | Gmail                            | Retrieves Gmail label IDs                             |                                |                                | "Use to get label IDs for mapping in code nodes.\nSee Setup Instructions sticky note."                      |
| Workflow Overview         | Sticky Note                     | Describes workflow purpose and outcomes              |                                |                                | "Auto-classifies & labels Gmail using AI\nZero-inbox email management system."                              |
| Setup Instructions        | Sticky Note                     | Stepwise setup guide                                  |                                |                                | "Create Gmail labels, get label IDs, configure credentials, test workflow."                                 |
| Gmail Trigger Note        | Sticky Note                     | Details Gmail Trigger configuration                   |                                |                                | "Polls Gmail every 1 minute, unread emails only.\nChange poll frequency in node settings."                  |
| Label Check Note          | Sticky Note                     | Explains label existence check logic                  |                                |                                | "Skip emails already labeled.\nUpdate labelMap in code node with your label IDs."                           |
| AI Agent Note             | Sticky Note                     | Explains AI classification engine and workflow logic |                                |                                | "Describes how AI uses sender history and label examples, 80% majority rule bias, and output format."       |
| Sender Tool Note          | Sticky Note                     | Describes "Get Emails By Sender Email" tool           |                                |                                | "Fetches up to 10 previous emails from sender before current email date."                                  |
| Label Tool Note           | Sticky Note                     | Describes "Get Emails By Label" tool                   |                                |                                | "Fetches up to 10 emails from a label category for comparison."                                            |
| Label Application Note    | Sticky Note                     | Explains label application and archiving process      |                                |                                | "Maps AI label → Gmail label ID.\nAdds label and archives email.\nUpdate mappings with your label IDs."     |
| Get Label IDs             | Sticky Note                     | Instructions on retrieving Gmail label IDs            |                                |                                | "Manual trigger and steps to get label IDs.\nUpdate code nodes accordingly."                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Labels** in your Gmail account exactly as:  
   - Meetings  
   - Inquiries  
   - Notify / Verify  
   - Expenses  
   - Orders / Deliveries  
   - Trash Likely  

2. **Create Credentials:**
   - Set up Gmail OAuth2 credentials with read/write access.
   - Set up Google Gemini (PaLM) API credentials.

3. **Create `Gmail Trigger` Node:**
   - Type: Gmail Trigger  
   - Poll every minute (pollTimes mode: everyMinute)  
   - Filter for unread emails only (`readStatus: "unread"`)  
   - Use your Gmail OAuth2 credentials.

4. **Add `Loop Over Items` Node:**
   - Type: Split In Batches  
   - Connect `Gmail Trigger` output to this node input.  
   - Default batch size (1) to process emails individually.

5. **Add `Check Label Existence` Node:**
   - Type: Code node  
   - JavaScript code: Map known Gmail label IDs to label names; check if email has any of these labels assigned.  
   - Update labelMap with your exact Gmail label IDs.  
   - Connect `Loop Over Items` output to it.

6. **Add `If Not Assigned` Node:**
   - Type: If condition node  
   - Condition: Check if `hasAILabelAssigned` from previous node is false.  
   - Connect `Check Label Existence` output to this node.

7. **Add `Email Classifier Agent` Node:**
   - Type: LangChain Agent node  
   - Configure system prompt with label definitions, classification rules, and output format (see overview for exact wording).  
   - Configure prompt to include current email subject and body from `Loop Over Items`.  
   - Define tools in agent:  
     - `Get Emails By Sender Email` node (Gmail Tool)  
     - `Get Emails By Label` node (Gmail Tool)  
   - Connect `If Not Assigned` true output to this node.

8. **Add `Get Emails By Sender Email` Node:**
   - Type: Gmail Tool node  
   - Operation: getAll  
   - Filters: sender = current email sender, receivedBefore = current email date  
   - Limit: 10 emails  
   - Connect as a tool in the agent.

9. **Add `Get Emails By Label` Node:**
   - Type: Gmail Tool node  
   - Operation: getAll  
   - Filters: label query from AI context  
   - Limit: 10 emails  
   - Connect as a tool in the agent.

10. **Add `Google Gemini Chat Model` Node:**
    - Type: LangChain LM node  
    - Credentials: Google PaLM API credentials  
    - Connect as AI language model in agent.

11. **Add `Structured Output Parser` Node:**
    - Type: LangChain Output Parser node  
    - JSON schema example: `{ "label": "Category Name" }`  
    - Connect as output parser in agent.

12. **Add `If not None` Node:**
    - Type: If condition node  
    - Condition: AI output label is not "None" (string comparison)  
    - Connect agent main output to this node.

13. **Add `Convert Label to Label ID` Node:**
    - Type: Code node  
    - JavaScript code: Map AI label name (e.g. "Meetings") to Gmail label ID string.  
    - Update mapping with your Gmail label IDs.  
    - Connect `If not None` true output to this node.

14. **Add `Add label to thread` Node:**
    - Type: Gmail node  
    - Operation: addLabels  
    - Resource: thread  
    - Thread Id: from current item  
    - Label Ids: from previous node output  
    - Use Gmail OAuth2 credentials.  
    - Connect `Convert Label to Label ID` output to this node.

15. **Add `Remove Inbox label from thread` Node:**
    - Type: Gmail node  
    - Operation: removeLabels  
    - Resource: thread  
    - Thread Id: from email message data  
    - Label Ids: ["INBOX"]  
    - Use Gmail OAuth2 credentials.  
    - Connect `Add label to thread` output to this node.

16. **Connect `Remove Inbox label from thread` output back to `Loop Over Items`** to continue processing next emails.

17. **Add Manual Trigger and Setup Nodes:**
    - Manual Trigger node to start workflow manually.  
    - Add `Get many messages` Gmail node to fetch sample emails for testing.  
    - Add `Get Labels Info` Gmail node to retrieve Gmail label IDs for mapping.  
    - Use these nodes for initial setup and troubleshooting.

18. **Add Sticky Notes:**
    - Create sticky notes containing workflow overview, setup instructions, AI logic explanation, tool descriptions, label application notes, and instructions for retrieving label IDs.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow automatically classifies incoming Gmail messages using AI and sender history to maintain zero inbox.       | Workflow Overview sticky note                                                                                 |
| Setup requires creation of exact Gmail labels and updating label ID mappings in Code nodes before activation.        | Setup Instructions sticky note                                                                                 |
| AI classification uses an 80% majority bias rule on sender history to improve label accuracy and consistency.        | AI Agent Note sticky note                                                                                       |
| Use manual trigger and "Get Labels Info" node to retrieve Gmail label IDs for mapping in code nodes.                 | Get Label IDs sticky note                                                                                       |
| Gmail Trigger polls every minute and only fetches unread emails by default; frequency is editable in node settings.  | Gmail Trigger Note sticky note                                                                                  |
| “Check Label Existence” node prevents re-labeling already processed emails; labelMap must be manually updated.       | Label Check Note sticky note                                                                                    |
| “Get Emails By Sender Email” and “Get Emails By Label” nodes fetch contextual emails to assist AI classification.    | Sender Tool Note and Label Tool Note sticky notes                                                              |
| Label application involves mapping AI output to Gmail label IDs, applying the label, and archiving the email.        | Label Application Note sticky note                                                                              |
| For more information on Gmail labels and API usage, refer to official Google Gmail API documentation.                | External resource recommendation (not included as a clickable link here)                                       |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, adhering strictly to current content policies without any illegal, offensive, or protected material. All manipulated data is legal and publicly accessible.