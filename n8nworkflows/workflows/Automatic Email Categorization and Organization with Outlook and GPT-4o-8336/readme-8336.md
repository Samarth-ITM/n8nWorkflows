Automatic Email Categorization and Organization with Outlook and GPT-4o

https://n8nworkflows.xyz/workflows/automatic-email-categorization-and-organization-with-outlook-and-gpt-4o-8336


# Automatic Email Categorization and Organization with Outlook and GPT-4o

---

### 1. Workflow Overview

This workflow automates the categorization and organization of Outlook emails using AI-powered analysis with GPT-4o and OpenRouter language models. Its primary use case is for freelance developers or professionals who want to intelligently classify incoming emails based on their content, sender, and metadata, and then organize them by updating categories and moving them into appropriate Outlook folders automatically.

The workflow includes the following logical blocks:

- **1.1 Input Reception and Filtering:** Periodically fetch new, uncategorized emails from Outlook and filter those without categories.
- **1.2 Email Content Preparation:** Convert email body HTML to Markdown and clean the text for AI processing.
- **1.3 AI-based Categorization:** Use advanced AI agents to analyze the email content and metadata, producing a validated JSON categorization with detailed reasoning.
- **1.4 Folder and Category Management:** Retrieve Outlook folders, match AI categories to folder names, and assign categories to emails.
- **1.5 Email Update and Organization:** Update the email categories in Outlook and move emails to the categorized folders.
- **1.6 Error Handling and Logging:** Catch and log any errors during processing to ensure robustness.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

- **Overview:**  
  This block periodically triggers the workflow and retrieves recent uncategorized and unflagged emails from Outlook. It filters out emails that already have categories assigned to avoid duplicate processing.

- **Nodes Involved:**  
  - Schedule Trigger  
  - RecievedEmail (Microsoft Outlook node)  
  - Filter1 (Filter node)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Runs every 1 minute to ensure near real-time processing of incoming emails.  
    - Input: None (trigger node)  
    - Output: Triggers downstream email retrieval.  
    - Potential Failures: System time misconfiguration, workflow not activated.

  - **RecievedEmail**  
    - Type: Microsoft Outlook - Get All Messages  
    - Configuration:  
      - Retrieves latest email with these filters: not flagged, no categories assigned, from a specific Outlook folder (folder ID provided).  
      - Fields fetched: flag, from, importance, replyTo, sender, subject, toRecipients, body, categories, isRead.  
      - Limit: 1 email per trigger to control processing.  
    - Input: Trigger from Schedule Trigger.  
    - Output: Email object(s) sent to Filter1.  
    - Credential: Microsoft Outlook OAuth2 required.  
    - Edge cases: No new emails found, API timeouts, auth failures.

  - **Filter1**  
    - Type: Filter  
    - Configuration: Checks if the email's "categories" array is empty (i.e., uncategorized).  
    - Input: Emails from RecievedEmail.  
    - Output: Passes only uncategorized emails to next steps.  
    - Edge cases: Emails with malformed or missing categories field.

#### 2.2 Email Content Preparation

- **Overview:**  
  Converts the email body HTML content to Markdown format and cleans the text by removing HTML tags, markdown links/images, special characters, multiple spaces, and trims whitespace. Sets key email metadata variables for AI processing.

- **Nodes Involved:**  
  - Loop Over Items1 (Split in Batches)  
  - Markdown1 (HTML to Markdown converter)  
  - varEmal1 (Set node)

- **Node Details:**

  - **Loop Over Items1**  
    - Type: SplitInBatches  
    - Configuration: Processes emails one by one to handle each email individually through the AI categorization.  
    - Input: Filtered emails from Filter1.  
    - Output: Single email item passed to Markdown1 and downstream nodes.  
    - Edge cases: Large batch sizes causing slow processing (default batch size used).

  - **Markdown1**  
    - Type: Markdown converter  
    - Configuration: Converts the email body HTML content to Markdown for better AI readability.  
    - Expression: Uses the email body content from Loop Over Items1.  
    - Input: Single email from Loop Over Items1.  
    - Output: Markdown formatted content forwarded.  
    - Edge cases: Malformed HTML causing conversion issues.

  - **varEmal1**  
    - Type: Set  
    - Configuration: Sets key email fields as variables for AI input: subject, importance, sender email, from email, and a cleaned version of the email body (using JavaScript regex replacements to remove unwanted characters and formatting).  
    - Expressions: Uses JSON data from previous nodes to generate cleaned email body text.  
    - Input: Markdown1 output.  
    - Output: Prepared email object for AI processing.  
    - Edge cases: Regex processing errors or unexpected input formats.

#### 2.3 AI-based Categorization

- **Overview:**  
  Utilizes two AI agents (LangChain-based) with OpenRouter language models to categorize emails into predefined categories with detailed reasoning, ensuring JSON-only valid output for categories, subcategories, analysis, and confidence scores.

- **Nodes Involved:**  
  - AI Agent1 (LangChain AI agent)  
  - varJSON1 (Set node to parse AI output)  
  - Get many folders1 (Retrieve Outlook folders for matching)  
  - If (Condition node to find matching folder)  
  - AI Agent (Fallback or secondary AI agent)  
  - OpenRouter Chat Model / OpenRouter Chat Model1 (Language model credentials for AI agents)

- **Node Details:**

  - **AI Agent1**  
    - Type: LangChain Agent  
    - Configuration:  
      - Receives cleaned email JSON and category list as input.  
      - Prompt instructs AI to categorize email into one of the provided categories only, produce JSON output with subject, category, subCategory (sparingly), analysis, confidence.  
      - System message defines multi-phase intelligent categorization engine with detailed priority rules and specialized intelligence modules.  
      - Output parser enabled to extract JSON.  
    - Input: varEmal1 JSON data.  
    - Output: Categorization JSON passed to varJSON1.  
    - Credential: OpenRouter API.  
    - Edge cases: AI output format errors, rate limits, API downtime.

  - **varJSON1**  
    - Type: Set  
    - Configuration: Extracts valid JSON portion from AI output (using regex to isolate JSON object).  
    - On error: Continues with error output to Catch Errors1.  
    - Input: AI Agent1 output.  
    - Output: Parsed categorization JSON.  
    - Edge cases: Parsing failure if AI output malformed.

  - **Get many folders1**  
    - Type: Microsoft Outlook - Get All Folders  
    - Configuration:  
      - Retrieves all folders including child folders and their parentFolderId for category matching during organization.  
      - Fields: displayName, childFolderCount, parentFolderId.  
    - Credential: Microsoft Outlook OAuth2.  
    - Input: varJSON1 output.  
    - Output: Folder list to If node.  
    - Edge cases: API failures, missing folders.

  - **If**  
    - Type: If condition  
    - Configuration: Checks if the AI-determined category matches any folder displayName retrieved from Outlook.  
    - Input: Folder list and parsed category from varJSON1.  
    - Output: Passes matched folder for moving email in later step; else, no-match branch.  
    - Edge cases: No matching folder found, case sensitivity issues.

  - **AI Agent**  
    - Type: LangChain Agent (secondary)  
    - Configuration: Similar to AI Agent1, likely a fallback or alternative AI categorization. Uses same prompt and system message.  
    - Credential: OpenRouter (backup account).  
    - Input: varEmal1 data, triggered on error or alternate path.  
    - Output: Categorization JSON for further processing.  
    - Edge cases: Same as AI Agent1.

  - **OpenRouter Chat Model / OpenRouter Chat Model1**  
    - Type: Language Model (OpenRouter) nodes providing AI inference for both AI Agents.  
    - Credentials: OpenRouter API with primary and backup accounts.  
    - Input/Output: Connected as languageModel nodes to AI Agents.  
    - Edge cases: API limits, credential expiry.

#### 2.4 Folder and Category Management

- **Overview:**  
  Processes folder display names for clean matching, sets variables with email ID and selected category, and prepares for updating the email category and moving the email into the matching folder.

- **Nodes Involved:**  
  - Summarize  
  - Code  
  - VarID Category

- **Node Details:**

  - **Summarize**  
    - Type: Summarize (aggregation)  
    - Configuration: Appends all folder displayNames into one aggregated output.  
    - Input: Get many folders node.  
    - Output: Aggregated string of folder names to Code node.  
    - Edge cases: Very large folder lists may cause performance delays.

  - **Code**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Takes aggregated folder displayName array or string, converts to a comma-separated cleaned string for easy matching.  
      - Handles input being array or string flexibly.  
    - Input: Summarize output.  
    - Output: Cleaned string of folder names.  
    - Edge cases: Unexpected input types.

  - **VarID Category**  
    - Type: Set  
    - Configuration:  
      - Sets the email ID (message ID) and the cleaned category string from Code node.  
      - Prepares variables for use in update and move actions.  
    - Input: Code and RecievedEmail nodes.  
    - Output: Variables used to update Outlook email categories.  
    - Edge cases: Missing or invalid message ID.

#### 2.5 Email Update and Organization

- **Overview:**  
  Updates the Outlook email categories based on AI results and moves the email to the appropriate folder matching the category.

- **Nodes Involved:**  
  - Update Category  
  - Move Folder  
  - Merge1  
  - If1

- **Node Details:**

  - **Update Category**  
    - Type: Microsoft Outlook - Update Message  
    - Configuration:  
      - Uses message ID from VarID Category.  
      - Updates categories with the AI-determined category and subCategory, capitalized and filtered for non-empty values.  
      - Operation: update categories array in Outlook.  
    - Input: If node output (folder matched).  
    - Credential: Microsoft Outlook OAuth2.  
    - Output: Message updated status to Move Folder.  
    - Edge cases: API failure, invalid categories format, permission issues.

  - **Move Folder**  
    - Type: Microsoft Outlook - Move Message  
    - Configuration:  
      - Moves the email message to the folder ID matched in If node.  
      - Uses message ID from VarID Category.  
    - Input: Update Category output.  
    - Credential: Microsoft Outlook OAuth2.  
    - Edge cases: Folder not found, permission denied, message already moved.

  - **Merge1**  
    - Type: Merge  
    - Configuration: Merges multiple data streams, primarily used to combine results or branch outputs before looping over items again.  
    - Input: Merges outputs from If1 and If nodes.  
    - Output: Loops back to Loop Over Items1 for processing next email.  
    - Edge cases: Merge conflicts if data structure inconsistent.

  - **If1**  
    - Type: If condition  
    - Configuration: Checks if the email has been marked as read (isRead field) to potentially branch workflow or handle emails differently.  
    - Input: Update Category or other nodes’ output.  
    - Output: Different processing routes depending on read status.  
    - Edge cases: Missing isRead field.

#### 2.6 Error Handling and Logging

- **Overview:**  
  Captures any errors during the workflow execution and logs them for debugging or alerting purposes.

- **Nodes Involved:**  
  - Catch Errors1

- **Node Details:**

  - **Catch Errors1**  
    - Type: Set node used as error catcher.  
    - Configuration: Sets an "error" field with the JSON error message.  
    - Input: Error branches from varJSON1 or AI Agent1.  
    - Output: Sends error data back to Loop Over Items1 for possible retry or logging.  
    - Edge cases: Unexpected error formats.

---

### 3. Summary Table

| Node Name          | Node Type                     | Functional Role                          | Input Node(s)         | Output Node(s)            | Sticky Note                                                                                          |
|--------------------|-------------------------------|-----------------------------------------|-----------------------|---------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger              | Periodic workflow start trigger         | -                     | RecievedEmail             |                                                                                                    |
| RecievedEmail       | Microsoft Outlook (Get All)   | Fetches latest uncategorized emails     | Schedule Trigger      | Filter1                   |                                                                                                    |
| Filter1            | Filter                       | Filters emails without categories       | RecievedEmail         | Loop Over Items1           |                                                                                                    |
| Loop Over Items1    | SplitInBatches               | Processes each email one by one          | Filter1               | Markdown1 (success), null  |                                                                                                    |
| Markdown1           | Markdown converter           | Converts email body HTML to Markdown    | Loop Over Items1      | varEmal1                  | Converts the body of the email to markdown                                                         |
| varEmal1            | Set                         | Prepares cleaned email fields for AI    | Markdown1             | Get many folders           | Set email fields                                                                                   |
| Get many folders    | Microsoft Outlook (Get All)   | Retrieves folder list for category match| varEmal1              | Summarize                 |                                                                                                    |
| Summarize           | Summarize                    | Aggregates folder display names          | Get many folders      | Code                      |                                                                                                    |
| Code                | Code (JS)                   | Cleans and formats folder names          | Summarize             | VarID Category            |                                                                                                    |
| VarID Category      | Set                         | Sets email ID and category variables     | Code, RecievedEmail    | AI Agent1                 |                                                                                                    |
| AI Agent1           | LangChain Agent              | AI categorizes email into JSON format   | VarID Category        | varJSON1                  |                                                                                                    |
| varJSON1            | Set                         | Parses AI JSON output                     | AI Agent1             | Get many folders1, Catch Errors1 |                                                                                                    |
| Catch Errors1       | Set                         | Logs errors                              | varJSON1 (error path) | Loop Over Items1           |                                                                                                    |
| Get many folders1   | Microsoft Outlook (Get All)   | Retrieves folders with parent IDs        | varJSON1              | If                        |                                                                                                    |
| If                  | If                          | Matches AI category to folder name       | Get many folders1     | Update Category, Loop Over Items1 |                                                                                                    |
| Update Category     | Microsoft Outlook (Update)    | Updates email categories                  | If                    | Move Folder               |                                                                                                    |
| Move Folder         | Microsoft Outlook (Move)      | Moves email to matched folder             | Update Category       | Merge1                    |                                                                                                    |
| Merge1              | Merge                       | Merges outputs for looping                | Move Folder, If1      | Loop Over Items1           |                                                                                                    |
| If1                 | If                          | Checks if email is read                    | Update Category       | Merge1                    | Checks if the email has been read                                                                   |
| AI Agent            | LangChain Agent              | Secondary AI categorization agent         | AI Agent1             | varJSON1                  |                                                                                                    |
| OpenRouter Chat Model | Language Model (OpenRouter)  | Provides AI model for AI Agent1           | -                     | AI Agent1                 |                                                                                                    |
| OpenRouter Chat Model1| Language Model (OpenRouter)  | Provides AI model for AI Agent            | -                     | AI Agent                  |                                                                                                    |
| Sticky Note8        | Sticky Note                  | Workflow branding and author info         | -                     | -                         | # Fully Automatic Categorise Outlook Emails with AI Built by [Can KURT](https://github.com/ck-cankurt/) at [ubden.com](https://ubden.com) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set interval to every 1 minute.

2. **Add Microsoft Outlook node ("RecievedEmail")**  
   - Operation: Get All Messages  
   - Filters: `flag/flagStatus eq 'notFlagged' and not categories/any()`  
   - Folder: Specify folder ID to monitor (e.g., Inbox or custom folder).  
   - Limit: 1  
   - Fields: flag, from, importance, replyTo, sender, subject, toRecipients, body, categories, isRead  
   - Connect Schedule Trigger → RecievedEmail  
   - Set Microsoft Outlook OAuth2 credentials.

3. **Add Filter node ("Filter1")**  
   - Condition: `$json.categories` is empty (array empty)  
   - Connect RecievedEmail → Filter1

4. **Add SplitInBatches node ("Loop Over Items1")**  
   - Default batch size (1)  
   - Connect Filter1 → Loop Over Items1

5. **Add Markdown node ("Markdown1")**  
   - Input HTML: `={{ $('Loop Over Items1').item.json.body.content }}`  
   - Connect Loop Over Items1 (success output) → Markdown1

6. **Add Set node ("varEmal1")**  
   - Assign variables:  
     - subject: `={{ $json.subject }}`  
     - importance: `={{ $json.importance }}`  
     - sender: `={{ $json.sender.emailAddress }}`  
     - from: `={{ $json.from.emailAddress }}`  
     - body: Cleaned string from `$json.data` applying regex replacements to remove HTML tags, markdown links/images, special characters, multiple spaces, and trim whitespace (see node regex logic in overview).  
   - Connect Markdown1 → varEmal1

7. **Add Microsoft Outlook node ("Get many folders")**  
   - Operation: Get All Folders  
   - Include child folders: true  
   - Fields: displayName, childFolderCount  
   - Connect varEmal1 → Get many folders  
   - Set Microsoft Outlook OAuth2 credentials.

8. **Add Summarize node ("Summarize")**  
   - Field to summarize: displayName  
   - Aggregation: append  
   - Connect Get many folders → Summarize

9. **Add Code node ("Code")**  
   - JavaScript to convert input (array or string) into a comma-separated string of folder names as cleanedString.  
   - Connect Summarize → Code

10. **Add Set node ("VarID Category")**  
    - Assign:  
      - id: `={{ $('RecievedEmail').item.json.id }}`  
      - category: `={{ $json.cleanedString }}` (from Code node)  
    - Connect Code → VarID Category

11. **Add LangChain Agent node ("AI Agent1")**  
    - Text: prompt with instruction to categorize email JSON (from varEmal1) into one of provided categories with JSON-only output.  
    - System message: detailed AI categorization engine instructions as described in overview.  
    - Connect VarID Category → AI Agent1  
    - Use OpenRouter Chat Model node as language model (connect in ai_languageModel slot).  
    - Assign OpenRouter API credentials.

12. **Add Set node ("varJSON1")**  
    - Extract JSON from AI Agent1 output using regex: `= {{ $json.output.replace(/^.*?({.*}).*$/s, '$1') }}`  
    - On error: continue output  
    - Connect AI Agent1 → varJSON1

13. **Add Microsoft Outlook node ("Get many folders1")**  
    - Operation: Get All Folders  
    - Include child folders: true  
    - Fields: displayName, childFolderCount, parentFolderId  
    - Connect varJSON1 → Get many folders1  
    - Use Microsoft Outlook OAuth2 credentials.

14. **Add If node ("If")**  
    - Condition: Check if AI output category (`{{ $('varJSON1').item.json.output.category }}`) equals any Outlook folder displayName.  
    - Connect Get many folders1 → If

15. **Add Microsoft Outlook node ("Update Category")**  
    - Operation: Update Message  
    - Message ID: `={{ $('VarID Category').item.json.id }}`  
    - Update fields: categories set to array of AI category and subCategory (capitalized, non-empty)  
    - Connect If (true output) → Update Category  
    - Use Microsoft Outlook OAuth2 credentials.

16. **Add Microsoft Outlook node ("Move Folder")**  
    - Operation: Move Message  
    - Folder ID: matched folder ID from If node  
    - Message ID: same as Update Category  
    - Connect Update Category → Move Folder  
    - Use Microsoft Outlook OAuth2 credentials.

17. **Add Merge node ("Merge1")**  
    - Connect Move Folder and If1 outputs (to handle branching) → Merge1

18. **Connect Merge1 → Loop Over Items1**  
    - To process next email batch.

19. **Add If node ("If1")**  
    - Condition: Check if email "isRead" is true.  
    - Connect Update Category → If1

20. **Add LangChain Agent node ("AI Agent")**  
    - Similar to AI Agent1 as fallback or secondary agent.  
    - Connect AI Agent1 (error output) → AI Agent  
    - Use OpenRouter Chat Model1 with backup credentials.

21. **Add Set node ("Catch Errors1")**  
    - Assign field "error" with caught error JSON.  
    - Connect varJSON1 (error output) → Catch Errors1  
    - Connect Catch Errors1 → Loop Over Items1 (to not halt workflow).

22. **Add Sticky Note**  
    - Content: "# Fully Automatic Categorise Outlook Emails with AI Built by [Can KURT](https://github.com/ck-cankurt/) at [ubden.com](https://ubden.com)"

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                 |
|----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Workflow built by Can KURT, source and updates available on GitHub                                                               | https://github.com/ck-cankurt/                                  |
| Hosted and documented on ubden.com                                                                                               | https://ubden.com                                               |
| AI categorization leverages advanced multi-phase analysis including sender intelligence, content scanning, and spam detection   | Detailed in AI Agent systemMessage prompts                      |
| Uses Microsoft Outlook OAuth2 credentials for secure API access                                                                  | Credential setup required before running                        |
| OpenRouter API accounts needed for LangChain AI agents                                                                           | Primary and backup accounts configured in workflow             |
| Workflow respects email privacy by only processing metadata and bodies of authorized Outlook accounts                            | Assumes compliance with user permissions                        |
| Regularly scheduled trigger ensures near real-time categorization of incoming emails                                              | Configurable scheduling interval                                |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---