Generate AI Meeting Summaries and Tasks from Fathom Recordings to Dart

https://n8nworkflows.xyz/workflows/generate-ai-meeting-summaries-and-tasks-from-fathom-recordings-to-dart-10569


# Generate AI Meeting Summaries and Tasks from Fathom Recordings to Dart

### 1. Workflow Overview

This workflow automates the generation of meeting summaries and task creation from Fathom meeting recordings, integrating with Dart for documentation and task management. It is designed for teams or individuals who use Fathom for meeting recordings and want automated, structured meeting notes and actionable task follow-ups within Dart.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Receives meeting data via a webhook triggered by Fathom when a meeting ends.
- **1.2 AI Processing:** Uses an AI agent (LangChain with OpenAI GPT-4.1-mini) to generate a structured meeting summary from the transcript.
- **1.3 Summary Parsing:** Parses the AI-generated summary output into categorized JSON objects for further processing.
- **1.4 Dart Document Management:** Retrieves the target folder in Dart, creates a new document with the meeting summary, and retrieves the Dartboard.
- **1.5 Task Creation:** Creates a Dart task linking to the meeting summary document and the original Fathom recording for review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception  
**Overview:**  
This block listens for HTTP POST requests from Fathom's webhook upon meeting completion, receiving structured meeting data including transcript, summary, and metadata.

**Nodes Involved:**  
- Webhook

**Node Details:**  
- **Webhook**  
  - Type: n8n Webhook  
  - Role: Entry point triggered by Fathom meeting end event  
  - Configuration:  
    - HTTP Method: POST  
    - Path: `aabbafc6-0ce9-4c82-9de9-e4c55cd735c0` (custom webhook path)  
  - Inputs: External HTTP POST from Fathom  
  - Outputs: JSON payload with full meeting data  
  - Edge cases:  
    - Unauthorized requests (should be validated externally or via webhook security settings)  
    - Payload format changes by Fathom  
    - Missing or partial transcript or summary data

---

#### 2.2 AI Processing  
**Overview:**  
Processes the meeting transcript to generate a concise yet comprehensive summary, including both human-readable text and machine-readable JSON with detailed categories such as key takeaways, topics, action items, and next steps.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model (used as language model for AI Agent)

**Node Details:**  
- **AI Agent**  
  - Type: LangChain AI Agent (n8n)  
  - Role: Generates meeting summary based on a detailed prompt  
  - Configuration:  
    - Text prompt includes instructions for summary style and JSON formatting  
    - Input expression injects the transcript markdown from webhook: `{{ $json.body.default_summary.markdown_formatted }}`  
  - Connected to OpenAI Chat Model as its language model  
  - Edge cases:  
    - AI model API failures or rate limits  
    - Output formatting errors (non-JSON or invalid JSON)  
    - Insufficient or ambiguous transcript data causing poor summaries  

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model node  
  - Role: Provides GPT-4.1-mini model for AI Agent  
  - Configuration:  
    - Model: gpt-4.1-mini  
    - Credentials: OpenAI API key configured in n8n  
  - Edge cases:  
    - Authentication errors with OpenAI API  
    - Network or timeout failures  

---

#### 2.3 Summary Parsing  
**Overview:**  
Transforms the AI-generated summary message into a structured JSON object with defined categories for later document creation.

**Nodes Involved:**  
- Code in JavaScript

**Node Details:**  
- **Code in JavaScript**  
  - Type: Code node  
  - Role: Parses AI output string to extract and organize key summary elements into JSON arrays  
  - Configuration:  
    - Uses hardcoded example entries (static placeholders) for key_takeaways, topics, action_items, next_items  
    - Input from AI Agent output: `$input.first().json.output`  
    - Returns a JSON object with arrays for each category  
  - Edge cases:  
    - Static parsing limits real dynamic data use (suggested for future NLP improvements)  
    - Errors if AI output is missing or malformed  
    - Sync issues if AI output structure changes  

---

#### 2.4 Dart Document Management  
**Overview:**  
Retrieves the destination Dart folder, creates a new document with the formatted meeting summary, then retrieves the Dartboard for task creation.

**Nodes Involved:**  
- Retrieve an existing folder  
- Create a new doc  
- Retrieve an existing dartboard  
- Retrieve an existing doc (retrieves newly created doc by ID)

**Node Details:**  
- **Retrieve an existing folder**  
  - Type: Dart API node  
  - Role: Gets the Dart folder where meeting docs will be stored  
  - Configuration:  
    - Folder ID hardcoded as `u4FhgTk3stn7` (user must replace with their target folder ID)  
  - Credentials: Dart API OAuth2  
  - Edge cases:  
    - Folder ID invalid or inaccessible  
    - API permission errors  

- **Create a new doc**  
  - Type: Dart API node  
  - Role: Creates a new document in the retrieved folder with meeting summary contents  
  - Configuration:  
    - Title: Uses meeting title from webhook input + "(Meeting Summary)" suffix  
    - Text body: Markdown formatted summary sections (General summary, key takeaways, topics, next items, action items)  
    - Uses data from JavaScript code node outputs for summary details  
  - Credentials: Dart API OAuth2  
  - Edge cases:  
    - Markdown formatting errors  
    - API creation failures  
    - Missing input data  

- **Retrieve an existing dartboard**  
  - Type: Dart API node  
  - Role: Gets the Dartboard where review tasks will be created  
  - Configuration:  
    - Dartboard ID hardcoded as `K7jRC0JC2Wxz` (user must replace with their Dartboard)  
  - Credentials: Dart API OAuth2  
  - Edge cases:  
    - Invalid Dartboard ID  
    - Access denied  

- **Retrieve an existing doc**  
  - Type: Dart API node  
  - Role: Fetches the document just created to get properties like URL for task creation  
  - Configuration:  
    - Uses the ID of the new doc from previous node's output  
  - Credentials: Dart API OAuth2  
  - Edge cases:  
    - Timing issues if doc creation is delayed  
    - Retrieval failure  

---

#### 2.5 Task Creation  
**Overview:**  
Creates a task on the retrieved Dartboard prompting review of the meeting summary document and linking to the Fathom recording.

**Nodes Involved:**  
- Create a new task

**Node Details:**  
- **Create a new task**  
  - Type: Dart API node  
  - Role: Adds a task with a title referencing the meeting and description linking the Dart doc and Fathom recording URL  
  - Configuration:  
    - Title: "Review recent meeting: [Meeting Title]"  
    - Description includes:  
      - Link to Dart meeting summary document  
      - Link to original Fathom meeting recording  
  - Credentials: Dart API OAuth2  
  - Edge cases:  
    - API errors creating tasks  
    - Missing URLs if prior doc retrieval failed  
    - Task duplication if webhook retriggered  

---

### 3. Summary Table

| Node Name                   | Node Type                      | Functional Role                                         | Input Node(s)             | Output Node(s)                  | Sticky Note                                                                                                                      |
|-----------------------------|--------------------------------|---------------------------------------------------------|---------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Webhook                     | n8n-nodes-base.webhook          | Entry point receiving meeting data from Fathom webhook | -                         | AI Agent                       | ## 1. Workflow launch through Fathom webhook trigger Triggered by an online meeting completion connected to Fathom ...           |
| AI Agent                    | @n8n/n8n-nodes-langchain.agent | Generates AI meeting summary from transcript            | Webhook                   | Code in JavaScript             | ## 1. Workflow launch through Fathom webhook trigger Triggered by an online meeting completion connected to Fathom ...           |
| OpenAI Chat Model            | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4.1-mini model for AI Agent                 | AI Agent (ai_languageModel) | AI Agent                      | ## 1. Workflow launch through Fathom webhook trigger Triggered by an online meeting completion connected to Fathom ...           |
| Code in JavaScript           | n8n-nodes-base.code             | Parses AI output into structured JSON                     | AI Agent                  | Retrieve an existing folder    | ## 2. Parsing the AI output Parse output into usable JSON with Key takeaways, Topics, Action items, Next items                    |
| Retrieve an existing folder  | n8n-nodes-dart.dart             | Gets Dart folder for storing meeting docs                | Code in JavaScript        | Create a new doc              | ## 3. Retrieve target folder Select the Folder in your workspace where new meeting documents will be created                      |
| Create a new doc             | n8n-nodes-dart.dart             | Creates meeting summary document in Dart                  | Retrieve an existing folder | Retrieve an existing dartboard | ## 4. Create new document Creates a new meeting summary document with key sections                                               |
| Retrieve an existing dartboard | n8n-nodes-dart.dart           | Gets Dartboard for task creation                          | Create a new doc          | Retrieve an existing doc       | ## 5. Retrieve target dartboard Select the Dartboard in your workspace where new tasks will be created                            |
| Retrieve an existing doc     | n8n-nodes-dart.dart             | Retrieves newly created doc for URL details               | Retrieve an existing dartboard | Create a new task             | ## 6. Create a review task in Dart Creates a review task linking the summary doc and Fathom recording                             |
| Create a new task            | n8n-nodes-dart.dart             | Creates a review task on Dartboard                        | Retrieve an existing doc   | -                             | ## 6. Create a review task in Dart Creates a review task linking the summary doc and Fathom recording                             |
| Sticky Note                 | n8n-nodes-base.stickyNote       | Documentation note on workflow purpose                    | -                         | -                            | ## Meeting Summary Generator (Fathom → Dart) Automatic meeting summary generation and task creation workflow                      |
| Sticky Note1                | n8n-nodes-base.stickyNote       | Documentation note on webhook trigger & AI model usage   | -                         | -                            | ## 1. Workflow launch through Fathom webhook trigger Triggered by an online meeting completion connected to Fathom               |
| Sticky Note2                | n8n-nodes-base.stickyNote       | Documentation note on parsing AI output                   | -                         | -                            | ## 2. Parsing the AI output Parse output into usable JSON with Key takeaways, Topics, Action items, Next items                    |
| Sticky Note5                | n8n-nodes-base.stickyNote       | Documentation note on Dart folder retrieval               | -                         | -                            | ## 3. Retrieve target folder Select the Folder in your workspace where new meeting documents will be created                      |
| Sticky Note6                | n8n-nodes-base.stickyNote       | Documentation note on Dart document creation              | -                         | -                            | ## 4. Create new document Creates a new meeting summary document with key sections                                               |
| Sticky Note7                | n8n-nodes-base.stickyNote       | Documentation note on Dartboard retrieval                 | -                         | -                            | ## 5. Retrieve target dartboard Select the Dartboard in your workspace where new tasks will be created                            |
| Sticky Note4                | n8n-nodes-base.stickyNote       | Documentation note on Dart task creation                   | -                         | -                            | ## 6. Create a review task in Dart Creates a review task linking the summary doc and Fathom recording                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `aabbafc6-0ce9-4c82-9de9-e4c55cd735c0` (or your custom path)  
   - Purpose: Receive meeting data from Fathom webhook

2. **Add LangChain AI Agent Node**  
   - Type: LangChain Agent  
   - Configure prompt with meeting summarization instructions (include required JSON format and example)  
   - Use expression to pass transcript markdown: `{{ $json.body.default_summary.markdown_formatted }}`  
   - Connect Webhook output to this node input

3. **Add OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4.1-mini` or preferred GPT-4 variant  
   - Provide OpenAI API credentials  
   - Connect as AI language model input to AI Agent node

4. **Add Code Node (JavaScript)**  
   - Purpose: Parse AI output string into structured JSON categories  
   - Use code to initialize JSON with empty arrays, currently hardcoded placeholders for summary categories  
   - Input: AI Agent output property (e.g., `$input.first().json.output`)  
   - Output: JSON object with keys: `key_takeaways`, `topics`, `action_items`, `next_items`  
   - Connect AI Agent output to this node

5. **Add Dart Node: Retrieve Folder**  
   - Resource: Folder  
   - Operation: Get Folder  
   - Configure Folder ID (replace placeholder with your Dart folder ID)  
   - Provide Dart API OAuth2 credentials  
   - Connect Code node output to this node

6. **Add Dart Node: Create Document**  
   - Resource: Doc  
   - Operation: Create Doc  
   - Configure:  
     - Title: Use expression to set `"{{ $('Webhook').item.json.body.meeting_title }} (Meeting Summary)"`  
     - Text: Markdown including summary sections with data from the Code node (e.g., key_takeaways, topics, next_items, action_items)  
   - Provide Dart API credentials  
   - Connect Retrieve Folder output to this node

7. **Add Dart Node: Retrieve Dartboard**  
   - Resource: Dartboard  
   - Operation: Get Dartboard  
   - Configure Dartboard ID (replace placeholder with your Dartboard ID)  
   - Connect Create Document output to this node

8. **Add Dart Node: Retrieve Document**  
   - Resource: Doc  
   - Operation: Get Doc  
   - Use expression for ID: `{{ $('Create a new doc').item.json.item.id }}`  
   - Connect Retrieve Dartboard output to this node

9. **Add Dart Node: Create Task**  
   - Resource: Task  
   - Operation: Create Task  
   - Configure:  
     - Title: `"Review recent meeting: {{ $('Webhook').item.json.body.meeting_title }}"`  
     - Description: Includes link to Dart doc (`{{ $json.item.htmlUrl }}`) and Fathom share URL (`{{ $('Webhook').item.json.body.share_url }}`)  
   - Connect Retrieve Document output to this node

10. **Connect nodes in order:**  
    Webhook → AI Agent → Code in JavaScript → Retrieve Folder → Create a new doc → Retrieve Dartboard → Retrieve Document → Create a new task  

11. **Credentials Setup:**  
    - OpenAI: Configure OpenAI API key with proper scopes  
    - Dart: Configure Dart API OAuth2 credentials with access to workspaces, folders, dartboards, docs, and tasks

12. **Replace Placeholder IDs:**  
    - Folder ID in "Retrieve an existing folder" node  
    - Dartboard ID in "Retrieve an existing dartboard" node

13. **Testing:**  
    - Deploy the workflow  
    - Set Fathom webhook to point to your deployed Webhook URL  
    - Trigger a test meeting end event via Fathom

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                   | Context or Link                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Connect your Dart account (workspace + folder access) before running the workflow to ensure access rights and proper API authentication.                                                                                                                     | Workflow Setup                                            |
| Add your production webhook URL from this workflow's Webhook node into your Fathom API webhook settings.                                                                                                                                                       | [Fathom Webhook Docs](https://developers.fathom.ai/webhooks) |
| Customize the AI prompt in the AI Agent node to adjust tone, style, or add/remove summary sections such as Key Takeaways, Action Items, etc.                                                                                                                  | AI Prompt Customization                                   |
| Dart Folder and Dartboard IDs must be replaced with your actual target IDs for documents and task management.                                                                                                                                                   | Dart Workspace Configuration                               |
| This workflow relies on LangChain integration within n8n and a compatible OpenAI API key; ensure your API usage complies with your rate limits and billing.                                                                                                   | OpenAI API Usage                                           |
| The JavaScript parsing node currently uses static placeholders; consider enhancing it with NLP parsing or JSON validation for dynamic AI output in production environments.                                                                                     | Future Improvement Suggestion                              |
| The task creation node links both the Dart document and the original Fathom meeting recording, enabling seamless follow-up from within Dart.                                                                                                                 | Task Automation                                           |

---

**Disclaimer:** The provided text is derived solely from an automated n8n workflow. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.