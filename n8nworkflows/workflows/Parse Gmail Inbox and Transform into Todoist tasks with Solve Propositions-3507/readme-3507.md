Parse Gmail Inbox and Transform into Todoist tasks with Solve Propositions

https://n8nworkflows.xyz/workflows/parse-gmail-inbox-and-transform-into-todoist-tasks-with-solve-propositions-3507


# Parse Gmail Inbox and Transform into Todoist tasks with Solve Propositions

### 1. Workflow Overview

This workflow automates the process of transforming Gmail inbox emails into actionable Todoist tasks with AI-generated summaries, proposed actions, and suggested replies. It is designed for users who receive many emails and want to streamline task management and email response by leveraging AI analysis.

**Target Use Cases:**  
- Individuals or teams managing high-volume Gmail inboxes.  
- Users who want to create Todoist tasks automatically from emails.  
- Scenarios where AI-generated task summaries and reply suggestions improve productivity.  
- Collaborative environments where tasks are assigned before replying to emails.

**Logical Blocks:**

- **1.1 Trigger and Email Retrieval:** Receives trigger events and fetches starred and unread emails from Gmail inbox.  
- **1.2 Task Retrieval and Enrichment:** Retrieves open Todoist tasks and enriches email data with existing tasks to check for duplicates.  
- **1.3 AI Analysis and Task Creation:** For emails without existing tasks, calls AI to analyze email content, generate structured output, and create new Todoist tasks.  
- **1.4 Task Closure on Email Unstar:** Detects emails that were unstarred and closes corresponding Todoist tasks automatically.  
- **1.5 Optional Email Marking:** Marks emails as read and starred to track processing status (optional and configurable).

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Email Retrieval

**Overview:**  
This block initiates the workflow based on triggers and fetches relevant emails from Gmail inbox, specifically unread and starred messages.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Get Unread From Inbox (Gmail)  
- Get Starred From Inbox (Gmail)  
- Merge Starred and Unread Messages (Merge)  
- Mark As Read (Gmail)  
- Star (Gmail)  
- No Operation, do nothing (NoOp)  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Manual start for testing the workflow.  
  - Config: No parameters.  
  - Inputs: None  
  - Outputs: Triggers No Operation node.  
  - Edge Cases: None.

- **Get Unread From Inbox**  
  - Type: Gmail node (getAll operation)  
  - Role: Fetches all unread emails from the inbox.  
  - Config: Filters by labelIds: INBOX and readStatus: unread.  
  - Inputs: Triggered by No Operation node.  
  - Outputs: Passes emails to Mark As Read and Merge nodes.  
  - Edge Cases: Gmail API rate limits; large inboxes may require pagination or limits adjustment.

- **Get Starred From Inbox**  
  - Type: Gmail node (getAll operation)  
  - Role: Fetches all starred emails from the inbox.  
  - Config: Filters by labelIds: STARRED and INBOX.  
  - Inputs: Triggered by No Operation node.  
  - Outputs: Passes emails to Merge node.  
  - Edge Cases: Same as above.

- **Merge Starred and Unread Messages**  
  - Type: Merge node (combine mode)  
  - Role: Combines starred and unread emails into a single stream for processing.  
  - Config: Default combine mode, no special join fields.  
  - Inputs: From Get Unread From Inbox and Get Starred From Inbox.  
  - Outputs: Passes combined emails to enrichment nodes.  
  - Edge Cases: Duplicate emails appearing in both lists; handled downstream.

- **Mark As Read**  
  - Type: Gmail node (markAsRead operation)  
  - Role: Marks processed unread emails as read.  
  - Config: Uses messageId from input JSON.  
  - Inputs: From Get Unread From Inbox.  
  - Outputs: Passes to Star node.  
  - Edge Cases: Gmail API errors, messageId invalid or already marked.

- **Star**  
  - Type: Gmail node (addLabels operation)  
  - Role: Adds STARRED label to emails to mark them for processing.  
  - Config: Uses messageId from input JSON, adds STARRED label.  
  - Inputs: From Mark As Read node.  
  - Outputs: None (end of this branch).  
  - Edge Cases: Label addition failure, label already present.

- **No Operation, do nothing**  
  - Type: NoOp node  
  - Role: Placeholder to initiate email fetching nodes.  
  - Config: None.  
  - Inputs: From triggers (manual, schedule, email IMAP).  
  - Outputs: Triggers Get Unread From Inbox, Get Starred From Inbox, and Get Open Tasks.  
  - Edge Cases: None.

---

#### 2.2 Task Retrieval and Enrichment

**Overview:**  
This block retrieves open Todoist tasks and enriches email data by matching email subjects with existing task content to detect duplicates or related tasks.

**Nodes Involved:**  
- Get Open Tasks (Todoist)  
- Enrich Emails With Tasks (Merge)  
- Enrich Tasks with Emails (Merge)  

**Node Details:**

- **Get Open Tasks**  
  - Type: Todoist node (getAll operation)  
  - Role: Retrieves all open tasks from a specific Todoist project.  
  - Config: Filters by projectId "2351998202".  
  - Inputs: From No Operation node.  
  - Outputs: Passes tasks to two merge nodes for enrichment.  
  - Edge Cases: API rate limits, project ID invalid or inaccessible.

- **Enrich Emails With Tasks**  
  - Type: Merge node (combine mode, enrichInput1 join)  
  - Role: Enriches email data with matching tasks based on subject/content.  
  - Config: Joins on email Subject and task content fields.  
  - Inputs: From Merge Starred and Unread Messages (emails) and Get Open Tasks (tasks).  
  - Outputs: Passes enriched emails to "If Task Not Exist" node.  
  - Edge Cases: Mismatched or missing fields; join failures.

- **Enrich Tasks with Emails**  
  - Type: Merge node (combine mode, enrichInput2 join)  
  - Role: Enriches task data with matching emails based on subject/content.  
  - Config: Joins on task Subject and email content fields.  
  - Inputs: From Get Open Tasks (tasks) and Merge Starred and Unread Messages (emails).  
  - Outputs: Passes enriched tasks to "If Email Unstarred (Not Exist)" node.  
  - Edge Cases: Same as above.

---

#### 2.3 AI Analysis and Task Creation

**Overview:**  
For emails without existing corresponding tasks, this block fetches full email content, sends it to AI for analysis, parses AI output into structured JSON, and creates new Todoist tasks with AI-generated summaries and suggestions.

**Nodes Involved:**  
- If Task Not Exist (If)  
- Get Full Message (Gmail)  
- Summarize Message (Langchain Agent)  
- OpenAI Chat Model (Langchain LM Chat OpenAI)  
- Structure Output Todoist Ready (Langchain Output Parser Structured)  
- If AI responded properly (If)  
- Create Todoist Task (Todoist)  

**Node Details:**

- **If Task Not Exist**  
  - Type: If node  
  - Role: Checks if the email does not have an existing task by verifying if the "content" field is missing.  
  - Config: Condition checks if `$json.content` does not exist.  
  - Inputs: From Enrich Emails With Tasks.  
  - Outputs: True branch triggers Get Full Message; false branch ends.  
  - Edge Cases: Missing or malformed fields causing false negatives.

- **Get Full Message**  
  - Type: Gmail node (get operation)  
  - Role: Retrieves full email message including HTML content for AI analysis.  
  - Config: Uses messageId from input JSON; simple mode false to get full content.  
  - Inputs: From If Task Not Exist (true branch).  
  - Outputs: Passes full message to Summarize Message node.  
  - Edge Cases: Message not found, API errors.

- **Summarize Message**  
  - Type: Langchain Agent node  
  - Role: Sends email content to AI to generate JSON with task content, description, actions, and answer.  
  - Config: Prompt instructs AI to output JSON with fields: content (email subject), description (summary), actions (proposed actions), answer (proposed reply).  
  - Inputs: From Get Full Message.  
  - Outputs: Passes AI raw output to If AI responded properly node.  
  - Edge Cases: AI response malformed, timeout, or rate limits.

- **OpenAI Chat Model**  
  - Type: Langchain LM Chat OpenAI node  
  - Role: Provides AI language model (GPT-4o-mini) for Summarize Message node.  
  - Config: Model set to "gpt-4o-mini".  
  - Inputs: Connected internally to Summarize Message node.  
  - Outputs: AI-generated text to Summarize Message.  
  - Edge Cases: API key invalid, quota exceeded.

- **Structure Output Todoist Ready**  
  - Type: Langchain Output Parser Structured node  
  - Role: Parses AI output into structured JSON matching Todoist task fields.  
  - Config: JSON schema example includes content, description, actions, answer fields.  
  - Inputs: Connected internally to Summarize Message node.  
  - Outputs: Parsed structured output to If AI responded properly node.  
  - Edge Cases: Parsing errors if AI output deviates from schema.

- **If AI responded properly**  
  - Type: If node  
  - Role: Validates AI output contains required fields content and description.  
  - Config: Checks existence of `$json.output.content` and `$json.output.description`.  
  - Inputs: From Summarize Message (structured output).  
  - Outputs: True branch triggers Create Todoist Task; false branch ends.  
  - Edge Cases: Missing fields causing task creation to skip.

- **Create Todoist Task**  
  - Type: Todoist node (create operation)  
  - Role: Creates a new task in Todoist with AI-generated content and detailed description including proposed actions and answers.  
  - Config:  
    - Content: AI output content field.  
    - Description: AI output description plus formatted proposed actions and answer.  
    - Project: Fixed project ID "2351998202".  
  - Inputs: From If AI responded properly (true branch).  
  - Outputs: None (end of this branch).  
  - Edge Cases: API errors, invalid project ID, rate limits.

---

#### 2.4 Task Closure on Email Unstar

**Overview:**  
This block detects emails that were previously starred but are now unstarred and closes the corresponding open Todoist tasks automatically.

**Nodes Involved:**  
- If Email Unstarred (Not Exist) (If)  
- Close Task (Todoist)  

**Node Details:**

- **If Email Unstarred (Not Exist)**  
  - Type: If node  
  - Role: Checks if the email subject does not exist in the enriched task data, indicating the email was unstarred.  
  - Config: Condition checks if `$json.Subject` does not exist.  
  - Inputs: From Enrich Tasks with Emails.  
  - Outputs: True branch triggers Close Task node.  
  - Edge Cases: False positives if subject fields mismatch.

- **Close Task**  
  - Type: Todoist node (close operation)  
  - Role: Closes the Todoist task corresponding to the unstarred email.  
  - Config: Uses taskId from input JSON.  
  - Inputs: From If Email Unstarred (Not Exist) (true branch).  
  - Outputs: None (end of this branch).  
  - Edge Cases: Task already closed, API errors.

---

#### 2.5 Optional Email Marking (Read and Star)

**Overview:**  
This optional block marks unread emails as read and stars them to track processing status. It can be removed if automatic marking is not desired.

**Nodes Involved:**  
- Mark As Read (Gmail)  
- Star (Gmail)  

**Node Details:**  
(Described in 2.1)

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                                    | Input Node(s)                         | Output Node(s)                         | Sticky Note                                                                                                  |
|-----------------------------|---------------------------------|---------------------------------------------------|-------------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger                  | Manual start for testing workflow                  | None                                | No Operation, do nothing              |                                                                                                              |
| Get Unread From Inbox        | Gmail                          | Fetch unread emails from inbox                      | No Operation, do nothing             | Mark As Read, Merge Starred and Unread Messages |                                                                                                              |
| Get Starred From Inbox       | Gmail                          | Fetch starred emails from inbox                     | No Operation, do nothing             | Merge Starred and Unread Messages     |                                                                                                              |
| Merge Starred and Unread Messages | Merge                      | Combine starred and unread emails                   | Get Unread From Inbox, Get Starred From Inbox | Enrich Emails With Tasks, Enrich Tasks with Emails |                                                                                                              |
| Mark As Read                | Gmail                          | Mark unread emails as read                          | Get Unread From Inbox                | Star                                  | ## Read and Star\n**Mark messages as Read and add Star to them**\nYou can delete this section if you don't want to make this happen automatically to your emails |
| Star                        | Gmail                          | Add STARRED label to emails                         | Mark As Read                        | None                                  | Same as above                                                                                                 |
| No Operation, do nothing    | NoOp                           | Placeholder to trigger email fetching and task retrieval | When clicking ‘Test workflow’, Schedule Trigger, Email Trigger (IMAP) | Get Unread From Inbox, Get Starred From Inbox, Get Open Tasks | ## Select Trigger\n**This workflow will work with many triggers**                                            |
| Get Open Tasks              | Todoist                        | Retrieve open tasks from Todoist project           | No Operation, do nothing             | Enrich Emails With Tasks, Enrich Tasks with Emails |                                                                                                              |
| Enrich Emails With Tasks    | Merge                          | Enrich emails with matching tasks                   | Merge Starred and Unread Messages, Get Open Tasks | If Task Not Exist                     |                                                                                                              |
| Enrich Tasks with Emails    | Merge                          | Enrich tasks with matching emails                   | Get Open Tasks, Merge Starred and Unread Messages | If Email Unstarred (Not Exist)        |                                                                                                              |
| If Task Not Exist           | If                             | Check if email has no existing task                 | Enrich Emails With Tasks             | Get Full Message                      |                                                                                                              |
| Get Full Message            | Gmail                          | Retrieve full email content                          | If Task Not Exist                   | Summarize Message                    |                                                                                                              |
| Summarize Message           | Langchain Agent                | AI analysis to generate task summary and suggestions | Get Full Message                   | If AI responded properly             |                                                                                                              |
| OpenAI Chat Model           | Langchain LM Chat OpenAI       | AI language model used by Summarize Message         | Internal to Summarize Message       | Summarize Message                    |                                                                                                              |
| Structure Output Todoist Ready | Langchain Output Parser Structured | Parse AI output into structured JSON for Todoist  | Internal to Summarize Message       | If AI responded properly             |                                                                                                              |
| If AI responded properly    | If                             | Validate AI output contains required fields         | Summarize Message                   | Create Todoist Task                  |                                                                                                              |
| Create Todoist Task         | Todoist                        | Create new task in Todoist with AI-generated content | If AI responded properly            | None                                | ## Make New Task in Todoist\nIf there is no task on Todoist with same subject as your either starred or unread message, then it's time to create one.\nBut to make it easier for you to close tasks and emails, let AI help you out.\nAI not only summarizes message. It also proposes actions you should take or answers how you should reply to this email. |
| If Email Unstarred (Not Exist) | If                          | Check if email was unstarred (task missing)         | Enrich Tasks with Emails             | Close Task                         | ## Close Task\n**If Email was Unstarred manually by you, then we can probably close task, so we are closing it** |
| Close Task                 | Todoist                        | Close Todoist task corresponding to unstarred email | If Email Unstarred (Not Exist)      | None                                | Same as above                                                                                                 |
| Email Trigger (IMAP)       | Email Read IMAP                | Alternative trigger (disabled)                       | None                                | No Operation, do nothing             | ## Select Trigger\n**This workflow will work with many triggers**                                            |
| Schedule Trigger           | Schedule Trigger              | Alternative trigger (disabled)                       | None                                | No Operation, do nothing             | Same as above                                                                                                 |
| Sticky Note                | Sticky Note                   | Instructional note                                  | None                                | None                                | ## Select Trigger\n**This workflow will work with many triggers**                                            |
| Sticky Note1               | Sticky Note                   | Instructional note                                  | None                                | None                                | ## Read and Star\n**Mark messages as Read and add Star to them**\nYou can delete this section if you don't want to make this happen automatically to your emails |
| Sticky Note2               | Sticky Note                   | Instructional note                                  | None                                | None                                | ## Close Task\n**If Email was Unstarred manually by you, then we can probably close task, so we are closing it** |
| Sticky Note3               | Sticky Note                   | Instructional note                                  | None                                | None                                | ## Make New Task in Todoist\nIf there is no task on Todoist with same subject as your either starred or unread message, then it's time to create one.\nBut to make it easier for you to close tasks and emails, let AI help you out.\nAI not only summarizes message. It also proposes actions you should take or answers how you should reply to this email. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a Manual Trigger node named "When clicking ‘Test workflow’" for testing.  
   - Optionally add Schedule Trigger or Email IMAP Trigger nodes (disabled by default).

2. **Add No Operation Node:**  
   - Add a No Operation node named "No Operation, do nothing".  
   - Connect all triggers (manual, schedule, IMAP) to this node.

3. **Fetch Emails:**  
   - Add Gmail node "Get Unread From Inbox":  
     - Operation: getAll  
     - Filters: labelIds = ["INBOX"], readStatus = "unread"  
     - Credentials: Set Gmail OAuth2 credentials.  
   - Add Gmail node "Get Starred From Inbox":  
     - Operation: getAll  
     - Filters: labelIds = ["STARRED", "INBOX"]  
     - Credentials: Same Gmail OAuth2.  
   - Connect "No Operation" node to both Gmail nodes.

4. **Merge Emails:**  
   - Add Merge node "Merge Starred and Unread Messages" in combine mode.  
   - Connect outputs of "Get Unread From Inbox" and "Get Starred From Inbox" to this merge node.

5. **Mark Emails as Read and Starred (Optional):**  
   - Add Gmail node "Mark As Read":  
     - Operation: markAsRead  
     - Message ID: `={{ $json.id }}`  
     - Credentials: Gmail OAuth2.  
   - Add Gmail node "Star":  
     - Operation: addLabels  
     - Label IDs: ["STARRED"]  
     - Message ID: `={{ $json.id }}`  
     - Credentials: Gmail OAuth2.  
   - Connect "Get Unread From Inbox" to "Mark As Read", then to "Star".

6. **Retrieve Open Todoist Tasks:**  
   - Add Todoist node "Get Open Tasks":  
     - Operation: getAll  
     - Filter: projectId = "2351998202" (replace with your project ID)  
     - Credentials: Todoist API credentials.

7. **Enrich Emails and Tasks:**  
   - Add Merge node "Enrich Emails With Tasks":  
     - Mode: combine, joinMode: enrichInput1  
     - Merge by fields: Email Subject and Task Content.  
   - Add Merge node "Enrich Tasks with Emails":  
     - Mode: combine, joinMode: enrichInput2  
     - Merge by fields: Task Subject and Email Content.  
   - Connect "Merge Starred and Unread Messages" and "Get Open Tasks" to "Enrich Emails With Tasks".  
   - Connect "Get Open Tasks" and "Merge Starred and Unread Messages" to "Enrich Tasks with Emails".

8. **Check for Non-Existing Tasks:**  
   - Add If node "If Task Not Exist":  
     - Condition: Check if `$json.content` does not exist (string notExists).  
   - Connect "Enrich Emails With Tasks" output to this node.

9. **Get Full Email Content:**  
   - Add Gmail node "Get Full Message":  
     - Operation: get  
     - Message ID: `={{ $json.id }}`  
     - Simple: false (to get full content)  
     - Credentials: Gmail OAuth2.  
   - Connect "If Task Not Exist" true branch to this node.

10. **AI Analysis:**  
    - Add Langchain Agent node "Summarize Message":  
      - Prompt: Instruct AI to analyze email and output JSON with content, description, actions, answer fields.  
      - Input text: Use `{{ $json.html }}` and subject.  
      - Enable output parser.  
    - Add Langchain LM Chat OpenAI node "OpenAI Chat Model":  
      - Model: "gpt-4o-mini" (or your preferred model)  
      - Credentials: OpenAI API credentials.  
    - Add Langchain Output Parser Structured node "Structure Output Todoist Ready":  
      - JSON schema example matching expected AI output.  
    - Connect "Get Full Message" to "Summarize Message".  
    - Connect "OpenAI Chat Model" to "Summarize Message" AI model input.  
    - Connect "Structure Output Todoist Ready" to "Summarize Message" output parser.

11. **Validate AI Output:**  
    - Add If node "If AI responded properly":  
      - Conditions: Check existence of `$json.output.content` and `$json.output.description`.  
    - Connect "Summarize Message" to this node.

12. **Create Todoist Task:**  
    - Add Todoist node "Create Todoist Task":  
      - Operation: create  
      - Content: `={{ $json.output.content }}`  
      - Description: `={{ $json.output.description }}\n\n**Proposed actions**\n\n{{ $json.output.actions }}\n\n**Proposed answer**\n\n{{ $json.output.answer }}`  
      - Project: Use your Todoist project ID (e.g., "2351998202").  
      - Credentials: Todoist API credentials.  
    - Connect "If AI responded properly" true branch to this node.

13. **Detect Unstarred Emails and Close Tasks:**  
    - Add If node "If Email Unstarred (Not Exist)":  
      - Condition: Check if `$json.Subject` does not exist (string notExists).  
    - Connect "Enrich Tasks with Emails" to this node.  
    - Add Todoist node "Close Task":  
      - Operation: close  
      - Task ID: `={{ $json.id }}`  
      - Credentials: Todoist API credentials.  
    - Connect "If Email Unstarred (Not Exist)" true branch to "Close Task".

14. **Add Sticky Notes (Optional):**  
    - Add Sticky Note nodes with instructions for triggers, read/star section, task creation, and task closure as per original workflow.

15. **Final Connections:**  
    - Connect nodes as per described flow: triggers → NoOp → email fetch → merge → enrichment → conditions → AI analysis → task creation or closure.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow supports multiple triggers including manual, schedule, and IMAP email triggers.                                                                                                                                 | Sticky Note "Select Trigger"                                                                        |
| The "Read and Star" section is optional and can be removed if you do not want emails to be automatically marked as read and starred.                                                                                        | Sticky Note1                                                                                       |
| AI does not maintain memory across emails intentionally to analyze each message independently.                                                                                                                               | Description section                                                                                 |
| Adjust limits on Gmail nodes ("Get Unread From Inbox", "Get Starred From Inbox") and Todoist node ("Get Open Tasks") if you experience issues with AI output compliance or API limits.                                          | Description section                                                                                 |
| You can replace Todoist with other task management tools like Asana or databases like NocoDB by modifying task creation and retrieval nodes accordingly.                                                                      | Description section                                                                                 |
| AI prompt instructs to produce detailed JSON output with fields: content (task name), description (summary), actions (proposed actions), and answer (proposed reply).                                                           | Summarize Message node prompt                                                                       |
| OpenAI model used is "gpt-4o-mini" by default; you may change this in the OpenAI Chat Model node to suit your needs or preferred language.                                                                                   | OpenAI Chat Model node                                                                              |
| Project ID "2351998202" is used as default for Todoist tasks; replace with your own project ID in Todoist nodes.                                                                                                              | Todoist nodes configuration                                                                        |
| The workflow does not unstar emails when closing tasks to avoid inconsistencies.                                                                                                                                              | Description section                                                                                 |
| For detailed setup instructions, refer to n8n documentation on Gmail OAuth2, Todoist API credentials, and OpenAI API integration.                                                                                            | General setup instructions                                                                          |

---

This document provides a comprehensive understanding of the workflow, enabling reproduction, modification, and troubleshooting for advanced users and automation agents alike.