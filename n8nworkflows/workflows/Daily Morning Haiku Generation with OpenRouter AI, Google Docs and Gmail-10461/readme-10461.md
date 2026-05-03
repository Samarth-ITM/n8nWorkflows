Daily Morning Haiku Generation with OpenRouter AI, Google Docs and Gmail

https://n8nworkflows.xyz/workflows/daily-morning-haiku-generation-with-openrouter-ai--google-docs-and-gmail-10461


# Daily Morning Haiku Generation with OpenRouter AI, Google Docs and Gmail

### 1. Workflow Overview

This workflow automates the daily generation and distribution of a creative Japanese haiku poem each morning at 7:00 AM. It leverages OpenRouter AI to produce thematic words, constructs a traditional 5-7-5 syllabic haiku, saves the poem to a Google Docs document, and emails the haiku to a recipient using Gmail.

The workflow is divided into four logical blocks:

- **1.1 Scheduling & AI Generation:** Triggers daily at 7:00 AM, calls an AI agent to generate four key words necessary for haiku construction using OpenRouter.
- **1.2 Haiku Construction & Formatting:** Processes the AI output to assemble the haiku lines and formats the poem text along with a document title.
- **1.3 Google Docs Automation:** Creates a new Google Docs document, appends the haiku text, and updates the document.
- **1.4 Email Delivery:** Sends the finalized haiku via Gmail to a specified recipient.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & AI Generation

**Overview:**  
This block initiates the workflow daily at 7:00 AM and generates four creative words (a seasonal word, a noun, and two verbs) by invoking OpenRouter AI through an AI agent node.

**Nodes Involved:**  
- Schedule Trigger  
- AI Agent  
- OpenRouter Chat Model

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Trigger (Schedule)  
  - *Role:* Starts workflow execution every day at 7:00 AM.  
  - *Configuration:* Interval set with `triggerAtHour: 7`.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Connects to AI Agent.  
  - *Potential Failures:* Scheduling misconfiguration; node disabled; server time zone issues.

- **AI Agent**  
  - *Type:* Langchain AI Agent  
  - *Role:* Sends a prompt to generate four haiku-related words in JSON format.  
  - *Configuration:*  
    - Prompt instructs the AI to output exactly four words in JSON with keys: kigo, noun, verb1, verb2.  
    - No extra explanations allowed; strict JSON format enforced.  
    - Uses OpenRouter Chat Model as language model backend.  
  - *Key Expressions:* Static prompt text in Japanese specifying word requirements and JSON format.  
  - *Inputs:* Triggered by Schedule Trigger.  
  - *Outputs:* JSON output with words passed to the next node.  
  - *Potential Failures:* AI response formatting errors; API rate limits; authentication errors with OpenRouter; timeout; unexpected AI output structure.

- **OpenRouter Chat Model**  
  - *Type:* Langchain language model node (OpenRouter)  
  - *Role:* Provides AI model backend for AI Agent node.  
  - *Configuration:* Connected with OpenRouter API credentials; no additional parameters.  
  - *Inputs:* Receives prompt from AI Agent.  
  - *Outputs:* Returns AI response to AI Agent.  
  - *Potential Failures:* API authentication failure; network issues; quota exceeded.

---

#### 1.2 Haiku Construction & Formatting

**Overview:**  
Transforms the AI’s JSON output into a formatted 5-7-5 haiku, generates a date-based title, and prepares the text fields for Google Docs and email usage.

**Nodes Involved:**  
- Code in JavaScript  
- Edit Fields

**Node Details:**

- **Code in JavaScript**  
  - *Type:* Function (JavaScript)  
  - *Role:* Parses the AI JSON output, extracts the four words, constructs the haiku in 5-7-5 syllable format, and creates a title string with the current date.  
  - *Configuration:*  
    - Handles cases where AI output includes extraneous markdown code blocks and extracts JSON content safely.  
    - Builds three lines:  
      1. `${kigo}の中` ("Inside the seasonal word")  
      2. `${noun}で心が${verb1}` ("At the noun, the heart [verb1]s")  
      3. `今日もまた${verb2}` ("Today again [verb2]")  
    - Title format: "今日の一句 (YYYY/MM/DD)" using Japanese locale.  
  - *Key Expressions:* Accesses AI Agent output via `$node["AI Agent"].json.output`.  
  - *Inputs:* Receives AI Agent output.  
  - *Outputs:* JSON object containing `haiku` (string) and `title` (string).  
  - *Potential Failures:* JSON parse errors if AI output malformed; date localization errors; missing AI Agent output.

- **Edit Fields**  
  - *Type:* Set Node  
  - *Role:* Structures document fields: sets `documentTitle` from haiku title, builds `documentBody` combining markdown header, haiku text, and footer signature.  
  - *Configuration:*  
    - `documentTitle` assigned from `title`.  
    - `documentBody` formatted as:  
      ```
      # [title]

      [haiku]

      $2014 n8n Haiku Generator
      ```  
  - *Inputs:* Receives output from Code in JavaScript.  
  - *Outputs:* Passes formatted text fields downstream.  
  - *Potential Failures:* Missing or malformed inputs; field assignment errors.

---

#### 1.3 Google Docs Automation

**Overview:**  
Creates a new Google Docs document with the haiku title, prepares append text, and updates the document by inserting the haiku content.

**Nodes Involved:**  
- Create a document  
- Prepare Append  
- Update a document

**Node Details:**

- **Create a document**  
  - *Type:* Google Docs node (Create operation)  
  - *Role:* Creates a new Google Docs document with the haiku title as document name, saved into a specific Google Drive folder.  
  - *Configuration:*  
    - Title: taken from `documentTitle` (fallback to '今日の一句').  
    - Folder ID: fixed folder ID `"1W45ugKiRbbwp3FIFdLL690DAgkVKCKWB"`.  
  - *Credentials:* Google Docs OAuth2 credentials for user `roumut.shukuwa@gmail.com`.  
  - *Inputs:* Receives from Edit Fields.  
  - *Outputs:* Document ID and metadata for downstream use.  
  - *Potential Failures:* Permission errors; invalid folder ID; quota limits; OAuth token expiry.

- **Prepare Append**  
  - *Type:* Set Node  
  - *Role:* Prepares parameters for updating the Google Doc: extracts document ID and the haiku text for appending.  
  - *Configuration:*  
    - `docId` assigned from newly created document's ID.  
    - `text` assigned from `documentBody`.  
  - *Inputs:* Receives from Create a document node.  
  - *Outputs:* Supplies `docId` and `text` for update operation.  
  - *Potential Failures:* Missing document ID; text formatting issues.

- **Update a document**  
  - *Type:* Google Docs node (Update operation)  
  - *Role:* Inserts the haiku text into the newly created Google Docs document.  
  - *Configuration:*  
    - Operation: Update with insert action.  
    - Text to insert: from `Prepare Append.text`.  
    - Target document URL: from `Prepare Append.docId`.  
  - *Credentials:* Google Docs OAuth2 credentials (same as Create node).  
  - *Inputs:* Receives from Prepare Append.  
  - *Outputs:* Document update result.  
  - *Potential Failures:* Document locked or deleted before update; OAuth token issues; API limits.

---

#### 1.4 Email Delivery

**Overview:**  
Sends the generated haiku as a plain text email each morning to a specified recipient with the haiku text as the message body and the haiku title as the email subject.

**Nodes Involved:**  
- Send a message

**Node Details:**

- **Send a message**  
  - *Type:* Gmail node (Send Email)  
  - *Role:* Sends an email containing the haiku text and a warm greeting to a fixed recipient.  
  - *Configuration:*  
    - Recipient email: `syukuwa@joseikincenter.com`.  
    - Subject: taken from `Edit Fields.documentTitle`.  
    - Message body: haiku text plus a friendly signature "$2014 本日の俳句をお届けします $2600$FE0F".  
    - Email type: plain text (non-HTML).  
  - *Credentials:* Gmail OAuth2 credentials for user `roumut.shukuwa@gmail.com`.  
  - *Inputs:* Receives from Update a document node.  
  - *Outputs:* Email sending result.  
  - *Potential Failures:* Authentication errors; recipient address issues; Gmail API rate limits; network failures.

---

### 3. Summary Table

| Node Name            | Node Type                 | Functional Role                        | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                      |
|----------------------|---------------------------|-------------------------------------|------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger      | Schedule Trigger           | Triggers workflow daily at 7:00 AM | -                      | AI Agent                | ### Section 1 – Scheduling & AI Generation: Schedule Trigger → AI Agent → OpenRouter Model generates haiku components daily at 07:00 AM. |
| AI Agent             | Langchain AI Agent         | Generates 4 haiku words in JSON     | Schedule Trigger        | Code in JavaScript      | ### Section 1 – Scheduling & AI Generation: Schedule Trigger → AI Agent → OpenRouter Model generates haiku components daily at 07:00 AM. |
| OpenRouter Chat Model | Langchain language model   | Provides AI model backend            | AI Agent (ai_languageModel) | AI Agent            | ### Section 1 – Scheduling & AI Generation: Schedule Trigger → AI Agent → OpenRouter Model generates haiku components daily at 07:00 AM. |
| Code in JavaScript    | Function (JavaScript)      | Parses AI output, builds haiku       | AI Agent                | Edit Fields             | ### Section 2 – Build and Format Haiku: Code in JavaScript → Edit Fields combines AI outputs into a 5-7-5 haiku and sets the title and body text. |
| Edit Fields          | Set Node                   | Sets document title and body text   | Code in JavaScript      | Create a document       | ### Section 2 – Build and Format Haiku: Code in JavaScript → Edit Fields combines AI outputs into a 5-7-5 haiku and sets the title and body text. |
| Create a document     | Google Docs (Create)       | Creates Google Docs document         | Edit Fields             | Prepare Append          | ### Section 3 – Google Docs Automation: Create Document → Prepare Append → Update Document creates a Google Doc for today’s haiku and appends the new text daily. |
| Prepare Append        | Set Node                   | Prepares doc ID and text for update | Create a document       | Update a document       | ### Section 3 – Google Docs Automation: Create Document → Prepare Append → Update Document creates a Google Doc for today’s haiku and appends the new text daily. |
| Update a document     | Google Docs (Update)       | Inserts haiku text into document     | Prepare Append          | Send a message          | ### Section 3 – Google Docs Automation: Create Document → Prepare Append → Update Document creates a Google Doc for today’s haiku and appends the new text daily. |
| Send a message        | Gmail (Send Email)         | Sends haiku email                    | Update a document       | -                       | ### Section 4 – Email Delivery: Send a message (Gmail) sends the final haiku via email each morning with a warm greeting. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 7:00 AM (`triggerAtHour: 7`).  
   - No credentials needed.  

2. **Add OpenRouter Chat Model node**  
   - Type: Langchain language model (OpenRouter)  
   - Connect your OpenRouter API credentials.  
   - Leave default options.  

3. **Add AI Agent node**  
   - Type: Langchain AI Agent  
   - Set prompt text (Japanese) instructing AI to generate four JSON keys: kigo, noun, verb1, verb2, with no extra text.  
   - Set prompt type to "define" and enable output parser.  
   - Under "Language Model," select the OpenRouter Chat Model node.  

4. **Connect Schedule Trigger → AI Agent**  

5. **Add Code in JavaScript node**  
   - Paste the provided JavaScript code that:  
     - Extracts the AI Agent’s JSON output, cleans markdown code block wrappers, parses JSON, builds three haiku lines in 5-7-5 format, and creates a date-based title string in Japanese locale.  
   - Connect AI Agent → Code in JavaScript.  

6. **Add Edit Fields node (Set node)**  
   - Assign `documentTitle` from `{{ $json.title }}`.  
   - Assign `documentBody` a markdown string combining:  
     - `# [title]` header  
     - Haiku lines from `{{ $json.haiku }}`  
     - Footer line `$2014 n8n Haiku Generator`  
   - Connect Code in JavaScript → Edit Fields.  

7. **Add Create a document node (Google Docs)**  
   - Operation: Create  
   - Title: `{{ $node["Edit Fields"].json.documentTitle ?? '今日の一句' }}`  
   - Folder ID: Set your Google Drive folder ID (example: `"1W45ugKiRbbwp3FIFdLL690DAgkVKCKWB"`).  
   - Connect Edit Fields → Create a document.  
   - Use Google Docs OAuth2 credentials.  

8. **Add Prepare Append node (Set node)**  
   - Assign `docId` from `{{ $node["Create a document"].json.id }}`  
   - Assign `text` from `{{ $node["Edit Fields"].json.documentBody }}`  
   - Connect Create a document → Prepare Append.  

9. **Add Update a document node (Google Docs)**  
   - Operation: Update  
   - Action: Insert text  
   - Text to insert: `{{ $node["Prepare Append"].json.text }}`  
   - Document URL / ID: `{{ $node["Prepare Append"].json.docId }}`  
   - Connect Prepare Append → Update a document.  
   - Use Google Docs OAuth2 credentials.  

10. **Add Send a message node (Gmail)**  
    - Send To: `syukuwa@joseikincenter.com` (replace as needed)  
    - Subject: `{{ $node["Edit Fields"].json.documentTitle }}`  
    - Message: `{{ $node["Code in JavaScript"].json.haiku + "\n\n$2014 本日の俳句をお届けします $2600\xFE0F" }}`  
    - Email Type: Plain text  
    - Connect Update a document → Send a message.  
    - Use Gmail OAuth2 credentials.  

11. **Verify all connections** and ensure credentials for OpenRouter, Google Docs, and Gmail are correctly set and authorized.  

12. **Activate the workflow** to run daily at 7:00 AM.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                                             |
|------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| This workflow automatically generates a creative haiku poem every morning and emails it. | Overview sticky note in the workflow describing purpose and setup instructions.                                              |
| Adjust the schedule trigger node if a different time is preferred.           | Sticky Note1 emphasizes scheduling and AI generation timing.                                                                 |
| Use your own Google Drive folder ID to store the haiku documents.            | Specified in the Create a document node; update to your own folder if needed.                                                |
| Email recipient can be changed in the Send a message node parameters.        | Currently set to `syukuwa@joseikincenter.com`.                                                                               |
| OpenRouter AI is used as the language model; ensure API keys remain valid.    | Credentials for OpenRouter must be configured and active.                                                                    |
| The haiku is formatted strictly in 5-7-5 syllables using four AI-generated words. | Code in JavaScript node builds the haiku lines accordingly.                                                                  |
| The workflow includes error-prone points such as AI output formatting and API authentication. | Monitor logs for JSON parsing errors or credential expiration issues.                                                        |
| The workflow uses Japanese language prompts and locale-specific formatting.   | Date formatting and haiku are in Japanese locale; adapt if needed for other languages.                                       |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a workflow automation tool. The process strictly adheres to content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.