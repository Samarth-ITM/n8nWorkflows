Categorize Support Emails with AI and Create Tasks in Dart

https://n8nworkflows.xyz/workflows/categorize-support-emails-with-ai-and-create-tasks-in-dart-9581


# Categorize Support Emails with AI and Create Tasks in Dart

---
### 1. Workflow Overview

This workflow automates the triage and task creation process for incoming support emails using AI categorization and Dart task management. Upon receiving a new support email, it classifies the email into one of seven predefined categories, enriches the classification with metadata such as confidence, rationale, summary, and priority, cleans the email body for readability, and then creates a structured task in a Dartboard, including adding a comment with sender information.

The workflow is logically grouped into these blocks:

- **1.1 Input Reception:** Capturing incoming emails via Gmail trigger.
- **1.2 AI Processing:** Using a language model to categorize emails and assign metadata.
- **1.3 Parsing and Data Structuring:** Parsing AI JSON output and cleaning email body text.
- **1.4 Dart Integration:** Retrieving the target Dartboard, creating tasks, and adding sender comments.

This structure enables automated, consistent, and actionable support email handling tailored for support and operations teams using Dart for task management.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a new email arrives in the configured Gmail account. It serves as the entry point, capturing raw email data for downstream processing.

- **Nodes Involved:**  
  - Launch workflow on email receive

- **Node Details:**

  - **Launch workflow on email receive**  
    - *Type and Role:* Gmail Trigger node; listens for incoming emails.  
    - *Configuration:*  
      - Polls Gmail every minute for new emails.  
      - No filter applied by default but can be customized (e.g., filtering by sender if using Google Groups).  
    - *Input/Output:* No input; outputs full email data including subject, sender, body (HTML and text).  
    - *Version:* 1.3 (latest Gmail trigger version)  
    - *Potential Failures:* Authentication failure due to expired OAuth2 credentials, API rate limits, or Gmail service downtime.  
    - *Sticky Note:* Describes the launch and AI categorization overview (Sticky Note2 and Sticky Note).

---

#### 2.2 AI Processing

- **Overview:**  
  This block uses an OpenAI GPT-4 based model to classify the email into one of seven categories, assigning metadata such as confidence score, rationale, summary, and priority. The output is raw JSON in string form.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Email triage assistant logic

- **Node Details:**

  - **OpenAI Chat Model**  
    - *Type and Role:* LangChain OpenAI chat model node; runs GPT-4.1-mini for classification.  
    - *Configuration:*  
      - Model set to "gpt-4.1-mini" (a variant of GPT-4 optimized for concise chat).  
      - No additional options configured.  
      - Credentials use OpenAI API key.  
    - *Input:* Raw email text from "Launch workflow on email receive".  
    - *Output:* Raw JSON string with category classification and metadata.  
    - *Version:* 1.2  
    - *Potential Failures:* API key invalid or rate limited, timeout, or malformed prompt causing unexpected output.  
    - *Sub-workflow:* None.

  - **Email triage assistant logic**  
    - *Type and Role:* LangChain Agent node; applies a detailed system instruction prompt to classify email text.  
    - *Configuration:*  
      - Complex system prompt instructing to output a strict JSON schema with category, confidence, rationale, summary, and priority.  
      - Includes detailed definitions for each category and priority calibration rules to ensure consistent outputs.  
    - *Input:* Email text from Gmail trigger.  
    - *Output:* JSON string with classification results.  
    - *Version:* 2.2  
    - *Edge Cases:* If email text is ambiguous or contains mixed topics, results may vary; prompt explicitly handles category prioritization rules.  
    - *Sticky Note:* Workflow launch explanation and AI metadata assignment (Sticky Note2).

---

#### 2.3 Parsing and Data Structuring

- **Overview:**  
  Parses the JSON string output from the AI logic node into structured JSON for easier use downstream and cleans the full email HTML body into plain text for inclusion in task descriptions.

- **Nodes Involved:**  
  - Parsing stage for logic result  
  - Cleaning up full text of email

- **Node Details:**

  - **Parsing stage for logic result**  
    - *Type and Role:* Code node; parses the AI output string into JSON.  
    - *Configuration:*  
      - Trims and removes markdown-style ```json formatting if present.  
      - Parses JSON string safely, throwing an error if parsing fails to prevent downstream errors.  
    - *Input:* Raw AI output string from "Email triage assistant logic".  
    - *Output:* Parsed JSON object with keys: category, confidence, rationale, summary, priority.  
    - *Version:* 2  
    - *Potential Failures:* Malformed or incomplete JSON output from AI; parsing error will halt workflow with detailed error message.  
    - *Sticky Note:* Parsing explanation (Sticky Note1).

  - **Cleaning up full text of email**  
    - *Type and Role:* Code node; cleans HTML email body to readable plain text.  
    - *Configuration:*  
      - Strips HTML tags and elements (script, style, images, anchors).  
      - Converts line breaks and removes URLs and HTML entities.  
      - Removes problematic characters (quotes, braces, non-printables).  
      - Normalizes whitespace.  
    - *Input:* Gmail email body (HTML) from "Launch workflow on email receive".  
    - *Output:* Plain text clean email body for inclusion in Dart task.  
    - *Version:* 2  
    - *Edge Cases:* Emails with malformed HTML or unusual encodings may result in incomplete cleanup.  
    - *Sticky Note:* Email cleanup purpose and details (Sticky Note3).

---

#### 2.4 Dart Integration

- **Overview:**  
  Retrieves the target Dartboard, creates a new task with AI-classified metadata and cleaned email content, and adds a comment with sender details to maintain context.

- **Nodes Involved:**  
  - Retrieve an existing dartboard  
  - Create a new task  
  - Create a new comment

- **Node Details:**

  - **Retrieve an existing dartboard**  
    - *Type and Role:* Dart node; fetches the Dartboard by ID where tasks will be created.  
    - *Configuration:*  
      - Dartboard ID hardcoded ("K7jRC0JC2Wxz") - replace with your actual Dartboard ID.  
      - Operation: "Get Dartboard".  
    - *Input:* Cleaned email text from "Cleaning up full text of email".  
    - *Output:* Dartboard object confirming target board.  
    - *Version:* 1  
    - *Potential Failures:* Invalid Dartboard ID, authentication errors with Dart API credentials.  
    - *Sticky Note:* Dartboard targeting instructions (Sticky Note5).

  - **Create a new task**  
    - *Type and Role:* Dart node; creates a new task on the retrieved Dartboard using parsed email metadata.  
    - *Configuration:*  
      - Task title set to the email subject.  
      - Priority, tags (category), and description populated dynamically from parsed AI output and cleaned email body.  
      - Description includes rationale, confidence, summary, and full cleaned email text.  
    - *Input:* Dartboard info from previous node and parsed AI output.  
    - *Output:* Created task object (includes task ID).  
    - *Version:* 1  
    - *Potential Failures:* Dart API errors, invalid field values, missing required fields.  
    - *Sticky Note:* Task creation explanation (Sticky Note4).

  - **Create a new comment**  
    - *Type and Role:* Dart node; adds a comment to the newly created task with sender information.  
    - *Configuration:*  
      - Task ID dynamically referenced from "Create a new task" output.  
      - Comment text includes sender name and email address extracted from the original email.  
    - *Input:* Created task ID from previous node and original email sender info.  
    - *Output:* Confirmation of comment creation.  
    - *Version:* 1  
    - *Potential Failures:* API errors, missing task ID, or malformed sender info.  
    - *Sticky Note:* Part of the task creation and sender context preservation (Sticky Note4).

---

### 3. Summary Table

| Node Name                    | Node Type                      | Functional Role                                   | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                             |
|------------------------------|--------------------------------|-------------------------------------------------|-------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------|
| Launch workflow on email receive | Gmail Trigger                 | Entry point; triggers on new support email      | None                          | Email triage assistant logic   | Workflow launch through email capture; AI categorization overview (Sticky Note2, Sticky Note)         |
| OpenAI Chat Model             | LangChain OpenAI Chat Model    | Runs GPT-4 model to classify email               | Launch workflow on email receive | Email triage assistant logic  |                                                                                                        |
| Email triage assistant logic  | LangChain Agent                | Applies prompt to categorize email and assign metadata | Launch workflow on email receive | Parsing stage for logic result | Workflow launch and AI metadata assignment details (Sticky Note2)                                     |
| Parsing stage for logic result| Code                          | Parses AI JSON output string into JSON object    | Email triage assistant logic    | Cleaning up full text of email | Parsing AI output explanation (Sticky Note1)                                                          |
| Cleaning up full text of email| Code                          | Cleans email HTML body into plain text            | Parsing stage for logic result  | Retrieve an existing dartboard | Email cleanup details (Sticky Note3)                                                                   |
| Retrieve an existing dartboard| Dart                          | Fetches target Dartboard for task creation       | Cleaning up full text of email  | Create a new task              | Dartboard targeting instructions (Sticky Note5)                                                       |
| Create a new task             | Dart                          | Creates task with email metadata and cleaned body| Retrieve an existing dartboard  | Create a new comment           | Task creation explanation including metadata and description (Sticky Note4)                            |
| Create a new comment          | Dart                          | Adds sender info as comment to created task      | Create a new task              | None                          | Sender context as comment to keep task focused (Sticky Note4)                                         |
| Sticky Note                  | Sticky Note                   | Documentation and comments                        | None                          | None                          | Various notes as described above                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Node Type: Gmail Trigger  
   - Configure OAuth2 credentials for your Gmail account.  
   - Set polling mode to "every minute" or your preferred interval.  
   - Leave filters empty or configure sender filter if needed for Google Groups.

2. **Create LangChain OpenAI Chat Model node**  
   - Node Type: LangChain OpenAI Chat Model  
   - Connect input from Gmail Trigger node.  
   - Select model "gpt-4.1-mini" or suitable GPT-4 variant.  
   - Configure OpenAI API credentials with your API key.  
   - No special options required.

3. **Create LangChain Agent node ("Email triage assistant logic")**  
   - Node Type: LangChain Agent  
   - Connect input from OpenAI Chat Model node’s AI language model output.  
   - Paste the provided detailed system prompt that defines email categories, output JSON schema, examples, priority rules, and instructions.  
   - Use prompt type "define" (static prompt).  
   - No extra options needed.

4. **Create Code node ("Parsing stage for logic result")**  
   - Node Type: Code  
   - Connect input from "Email triage assistant logic" main output.  
   - Paste JavaScript code that trims markdown and parses AI output JSON safely, throwing error on failure.

5. **Create Code node ("Cleaning up full text of email")**  
   - Node Type: Code  
   - Connect input from "Parsing stage for logic result".  
   - Paste JavaScript code to clean HTML email body: normalize line breaks, strip tags, remove URLs, decode entities, remove problematic characters.

6. **Create Dart node ("Retrieve an existing dartboard")**  
   - Node Type: Dart  
   - Connect input from "Cleaning up full text of email".  
   - Set operation to "Get Dartboard".  
   - Enter your Dartboard ID (replace placeholder).  
   - Configure Dart API credentials (OAuth2 or API key).

7. **Create Dart node ("Create a new task")**  
   - Node Type: Dart  
   - Connect input from "Retrieve an existing dartboard".  
   - Set operation to "Create Task".  
   - Configure task item dynamically:  
     - Title: email subject from Gmail node.  
     - Priority: from parsed AI output.  
     - Tags: category from parsed AI output.  
     - Description: include rationale, confidence, summary, and cleaned email text.  
   - Use Dart API credentials.

8. **Create Dart node ("Create a new comment")**  
   - Node Type: Dart  
   - Connect input from "Create a new task".  
   - Set operation to "Add Task Comment".  
   - Configure comment text with sender name and email from Gmail node.  
   - Use Dart API credentials.

9. **Verify all connections and test**  
   - Ensure node input/output connections follow the sequence above.  
   - Test with sample emails to confirm correct AI categorization and task creation.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Support email routing template: AI classifies into 7 categories and creates Dart tasks for support teams.     | Sticky Note at workflow start; overview and setup instructions.                                |
| Dart integration guide for n8n: [https://help.dartai.com/en/articles/12313191-n8n-integration](https://help.dartai.com/en/articles/12313191-n8n-integration) | Official Dart integration help page for credentials and setup.                                 |
| AI model selection may impact classification quality; consider testing different GPT-4 variants or others.    | Mentioned in Sticky Notes and node configurations.                                            |
| Gmail trigger can be filtered by sender to handle Google Groups or specific inboxes.                          | Workflow note recommends filter usage for group addresses.                                    |
| Priority levels and category definitions are carefully designed for consistent AI output and task triage.     | Detailed in the system prompt inside "Email triage assistant logic" node configuration.       |

---

**Disclaimer:** The text and analysis provided derive exclusively from an automated workflow created with n8n, a workflow automation tool. The processing respects all applicable content policies and contains no illegal, offensive, or protected material. All data handled are lawful and publicly accessible.