Support student questions via LINE using Gemini, Google Drive, and Slack

https://n8nworkflows.xyz/workflows/support-student-questions-via-line-using-gemini--google-drive--and-slack-14427


# Support student questions via LINE using Gemini, Google Drive, and Slack

# 1. Workflow Overview

This workflow implements a LINE-based student support bot for tutoring and school contexts. A student sends a message to a LINE bot, the workflow acknowledges the webhook quickly, loads prior conversation history from Google Sheets, passes the current question plus recent history to a Gemini-powered AI Agent, and then sends the generated answer back to the student through the LINE Messaging API.

The intended design also includes searching teaching materials in Google Drive, logging interactions to Google Sheets, and escalating unresolved questions to Slack. However, in the provided workflow JSON, only the Google Drive search tool entry point is present as a tool reference; the actual logging and escalation tool nodes are not included, and the Google Drive search implementation is not fully wired as a complete sub-flow. This is important for anyone reproducing or extending the workflow.

## 1.1 Input Reception

The workflow starts from a webhook that receives incoming LINE events. It immediately returns a JSON acknowledgment to LINE and, in parallel, extracts the relevant message metadata needed later in the process.

## 1.2 Runtime Configuration and Context Preparation

A Set node centralizes all environment-specific values: LINE user ID, reply token, student message, Google Sheets ID, Google Drive folder ID, Slack webhook URL, and LINE channel access token. The workflow then reads spreadsheet rows from Google Sheets and aggregates them so the agent can access conversation history in a single structure.

## 1.3 AI Processing

A conversational AI Agent receives the current student question and the last ten historical entries. The agent is configured with Gemini as its language model and is instructed to search materials first, answer in Japanese, log the interaction, and escalate to a teacher when needed.

## 1.4 Student Reply Delivery

The final generated answer is sent back to LINE using an HTTP request to the LINE Reply API, using the reply token captured from the incoming event.

## 1.5 Referenced but Incomplete Tooling

The agent prompt references three tools:

- `search_materials`
- `save_progress`
- `escalate_to_teacher`

Only `search_materials` exists as a Tool Workflow node in the JSON, and even that points back to the same workflow ID without a visible dedicated search sub-workflow implementation. The other two tools are absent. As a result, the current workflow, as provided, cannot fully perform the logging and teacher escalation behavior described in the functional description unless additional nodes or sub-workflows are added.

---

# 2. Block-by-Block Analysis

## 2.1 Block: LINE Webhook Intake and Immediate Acknowledgment

### Overview

This block receives inbound HTTP POST requests from LINE and immediately acknowledges receipt. This pattern is useful because messaging platforms often expect a quick response, while the actual AI processing can take longer.

### Nodes Involved

- LINE Receiver
- Acknowledge LINE

### Node Details

#### LINE Receiver

- **Type and technical role:** `n8n-nodes-base.webhook`  
  Entry point that receives POST requests from LINE Messaging API.
- **Configuration choices:**  
  - HTTP method: `POST`
  - Path: `line-study-bot`
  - Response mode: `responseNode`
  This means the webhook will wait for a dedicated Respond to Webhook node to produce the HTTP response.
- **Key expressions or variables used:**  
  No expressions in the node itself, but downstream nodes rely on:
  - `$json.body.events[0].source.userId`
  - `$json.body.events[0].replyToken`
  - `$json.body.events[0].message.text`
- **Input and output connections:**  
  - No input; this is a trigger node.
  - Outputs to:
    - `Acknowledge LINE`
    - `Set Config`
- **Version-specific requirements:**  
  Uses webhook node type version `2`.
- **Edge cases or potential failure types:**  
  - LINE may send event payloads that do not contain `events[0]`
  - Non-text events may not have `message.text`
  - Signature validation is not implemented here; in production, LINE signature validation should be added
  - If the path is already used by another workflow, webhook registration conflicts may occur
- **Sub-workflow reference:**  
  None

#### Acknowledge LINE

- **Type and technical role:** `n8n-nodes-base.respondToWebhook`  
  Returns an immediate JSON response to the webhook caller.
- **Configuration choices:**  
  - Respond with: JSON
  - Body: `{"status":"ok"}`
- **Key expressions or variables used:**  
  Static JSON body.
- **Input and output connections:**  
  - Input from `LINE Receiver`
  - No downstream output used
- **Version-specific requirements:**  
  Uses node version `1.1`
- **Edge cases or potential failure types:**  
  - If the webhook is not configured for `responseNode` mode, this node will not behave as intended
  - If execution errors before reaching this node, LINE may receive no valid acknowledgment
- **Sub-workflow reference:**  
  None

---

## 2.2 Block: Configuration and Event Data Extraction

### Overview

This block normalizes incoming data and stores all environment-specific settings in one place. It is the operational hub for values that must be customized before activation.

### Nodes Involved

- Set Config

### Node Details

#### Set Config

- **Type and technical role:** `n8n-nodes-base.set`  
  Extracts fields from the LINE payload and defines hard-coded configuration placeholders.
- **Configuration choices:**  
  The node assigns:
  - `userId` from the incoming LINE event
  - `replyToken` from the incoming LINE event
  - `userMessage` from the incoming LINE event
  - `sheetsId` as a manual placeholder
  - `driveFolderId` as a manual placeholder
  - `slackWebhookUrl` as a manual placeholder
  - `lineChannelToken` as a manual placeholder
- **Key expressions or variables used:**  
  - `{{ $json.body.events[0].source.userId }}`
  - `{{ $json.body.events[0].replyToken }}`
  - `{{ $json.body.events[0].message.text }}`
- **Input and output connections:**  
  - Input from `LINE Receiver`
  - Output to `Load Chat History`
- **Version-specific requirements:**  
  Uses Set node version `3.4`
- **Edge cases or potential failure types:**  
  - Breaks if `events[0]` is missing
  - Breaks for image, sticker, location, or follow events where `message.text` is absent
  - Placeholder values must be replaced before use
  - Storing secrets such as LINE channel token directly in the node is convenient but not ideal for production security
- **Sub-workflow reference:**  
  None

---

## 2.3 Block: Chat History Retrieval and Aggregation

### Overview

This block fetches historical entries from Google Sheets and aggregates them into one data structure. The resulting history is later truncated to the most recent ten exchanges inside the AI prompt.

### Nodes Involved

- Load Chat History
- Aggregate History

### Node Details

#### Load Chat History

- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from a Google Sheet that stores prior student interactions.
- **Configuration choices:**  
  - Operation: `readRows`
  - Document ID comes from `{{ $json.sheetsId }}`
  The node reads from the spreadsheet document specified at runtime.
- **Key expressions or variables used:**  
  - `{{ $json.sheetsId }}`
- **Input and output connections:**  
  - Input from `Set Config`
  - Output to `Aggregate History`
- **Version-specific requirements:**  
  Uses Google Sheets node version `4.5`
  Requires valid Google Sheets credentials configured in n8n.
- **Edge cases or potential failure types:**  
  - Missing or invalid spreadsheet ID
  - OAuth permission issues
  - If no sheet/range is configured implicitly by defaults, behavior depends on n8n node defaults and spreadsheet structure
  - As configured, there is no visible filter by `userId`, so it may read all rows for all students rather than only the current student's history
- **Sub-workflow reference:**  
  None

#### Aggregate History

- **Type and technical role:** `n8n-nodes-base.aggregate`  
  Consolidates all incoming row items into one item with a `data` array.
- **Configuration choices:**  
  - Aggregate mode: `aggregateAllItemData`
  This is used so the agent can access all sheet rows through one JSON field.
- **Key expressions or variables used:**  
  No explicit expression here, but downstream references:
  - `$json.data.slice(-10)`
- **Input and output connections:**  
  - Input from `Load Chat History`
  - Output to `Study Agent`
- **Version-specific requirements:**  
  Uses Aggregate node version `1`
- **Edge cases or potential failure types:**  
  - If the sheet is empty, `data` may be an empty array, which is acceptable but should be anticipated
  - If unexpected row structures are returned, prompt quality may degrade
- **Sub-workflow reference:**  
  None

---

## 2.4 Block: AI Agent Reasoning and Tool Access

### Overview

This block is the core decision layer. It feeds the current student message and conversation history to a Gemini-backed conversational agent, and allows the agent to call a workflow tool intended for material search.

### Nodes Involved

- Study Agent
- Gemini Model
- Search Materials Tool

### Node Details

#### Study Agent

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  A LangChain-based conversational AI agent that generates the answer and can invoke connected tools.
- **Configuration choices:**  
  - Agent type: `conversationalAgent`
  - Prompt type: defined explicitly
  - User text includes:
    - current student question from `Set Config`
    - recent conversation history from aggregated sheet rows
  - System message instructs the agent to:
    - answer as a study support assistant
    - search materials first
    - use `save_progress` after every response
    - escalate unresolved questions
    - respond in Japanese
- **Key expressions or variables used:**  
  Main text field:
  - `{{ $('Set Config').item.json.userMessage }}`
  - `{{ JSON.stringify($json.data.slice(-10)) }}`
- **Input and output connections:**  
  - Main input from `Aggregate History`
  - AI language model input from `Gemini Model`
  - AI tool input from `Search Materials Tool`
  - Main output to `Reply via LINE`
- **Version-specific requirements:**  
  Uses node version `1.7`
  Requires compatible n8n LangChain/AI node support.
- **Edge cases or potential failure types:**  
  - If `data` is undefined, `slice(-10)` fails
  - If `userMessage` is empty or undefined, the answer may be meaningless
  - The prompt references tools that are not actually connected (`save_progress`, `escalate_to_teacher`)
  - The agent may attempt to call missing tools, causing tool-resolution errors or degraded output depending on n8n version behavior
  - Token/context growth may become an issue if spreadsheet rows are verbose, even though only the last ten are included
- **Sub-workflow reference:**  
  Consumes a tool workflow node (`Search Materials Tool`) as an AI tool

#### Gemini Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
  Supplies the chat language model used by the agent.
- **Configuration choices:**  
  - Temperature: `0.3`
  Lower temperature makes answers more stable and less random.
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - Connected to `Study Agent` via AI language model port
- **Version-specific requirements:**  
  Uses node version `1`
  Requires Google Gemini / PaLM API credentials in n8n.
- **Edge cases or potential failure types:**  
  - Invalid API key or revoked credentials
  - Model quota/rate-limit issues
  - Provider-side latency or timeout
  - Regional or account restrictions depending on Gemini API availability
- **Sub-workflow reference:**  
  None

#### Search Materials Tool

- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolWorkflow`  
  Exposes a workflow as an AI-callable tool named `search_materials`.
- **Configuration choices:**  
  - Tool name: `search_materials`
  - Description: search teaching materials in Google Drive
  - Input schema:
    - required string field `query`
  - `workflowId` is set to `{{ $workflow.id }}`
- **Key expressions or variables used:**  
  - `{{ $workflow.id }}`
- **Input and output connections:**  
  - Connected to `Study Agent` via AI tool port
- **Version-specific requirements:**  
  Uses node version `2`
  Requires AI tool workflow support in the installed n8n version.
- **Edge cases or potential failure types:**  
  - As configured, it points back to the current workflow ID, but there is no visible dedicated tool-handling branch or sub-workflow implementation in the provided JSON
  - This can lead to recursion, invalid tool invocation, or a nonfunctional tool depending on runtime behavior
  - No actual Google Drive search node exists in the workflow JSON
- **Sub-workflow reference:**  
  Intended as a workflow tool, but the referenced implementation is not present as a separate sub-workflow in the provided data

---

## 2.5 Block: Reply Delivery to LINE

### Overview

This block posts the agent’s generated answer back to LINE using the reply token from the original webhook event.

### Nodes Involved

- Reply via LINE

### Node Details

#### Reply via LINE

- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls LINE’s Reply API directly.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://api.line.me/v2/bot/message/reply`
  - Sends headers and body manually
  - Authorization uses Bearer token from `Set Config`
  - Message payload is a text message containing the AI output
- **Key expressions or variables used:**  
  - `{{ $('Set Config').item.json.replyToken }}`
  - `{{ $('Set Config').item.json.lineChannelToken }}`
  - `{{ $json.output }}`
- **Input and output connections:**  
  - Input from `Study Agent`
  - No downstream connection
- **Version-specific requirements:**  
  Uses HTTP Request node version `4.2`
- **Edge cases or potential failure types:**  
  - Reply tokens expire quickly; if AI processing takes too long, LINE may reject the request
  - Invalid channel access token returns authorization errors
  - If `$json.output` is empty, LINE may reject or send a blank response
  - LINE message length limits may apply
  - JSON string construction in the `messages` field can fail if the output contains unescaped characters
- **Sub-workflow reference:**  
  None

---

## 2.6 Block: Documentation Sticky Notes

### Overview

These nodes are not part of execution logic but provide important implementation context directly inside the n8n canvas. Their content should be preserved when reproducing or maintaining the workflow.

### Nodes Involved

- Sticky Note — Overview
- Sticky Note — Input
- Sticky Note — Config
- Sticky Note — History
- Sticky Note — Agent
- Sticky Note — Tools
- Sticky Note — Reply

### Node Details

#### Sticky Note — Overview

- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation note describing the overall purpose and setup steps.
- **Configuration choices:**  
  Contains workflow overview and setup reminders.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Version `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

#### Sticky Note — Input

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Documents the message intake area.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Version `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

#### Sticky Note — Config

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Documents where to update IDs, token, and webhook URL.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Version `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

#### Sticky Note — History

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Documents Google Sheets history loading purpose.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Version `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

#### Sticky Note — Agent

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Describes the AI Agent behavior.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Version `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

#### Sticky Note — Tools

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Describes the intended tools set.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Version `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

#### Sticky Note — Reply

- **Type and technical role:** `n8n-nodes-base.stickyNote`
- **Configuration choices:**  
  Documents the reply-delivery area.
- **Input and output connections:**  
  None
- **Version-specific requirements:**  
  Version `1`
- **Edge cases or potential failure types:**  
  None
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| LINE Receiver | n8n-nodes-base.webhook | Receives POST events from LINE Messaging API |  | Acknowledge LINE; Set Config | ## Receive student message\nCaptures incoming LINE messages and extracts the student's user ID and text. |
| Acknowledge LINE | n8n-nodes-base.respondToWebhook | Returns immediate JSON acknowledgment to LINE | LINE Receiver |  |  |
| Set Config | n8n-nodes-base.set | Extracts LINE event fields and stores runtime configuration values | LINE Receiver | Load Chat History | ## Set all config here\nUpdate Sheet ID, Drive folder ID, Slack webhook URL, and LINE token in this single node before activating. |
| Load Chat History | n8n-nodes-base.googleSheets | Reads prior conversation rows from Google Sheets | Set Config | Aggregate History | ## Load conversation history\nFetches the student's past Q&A from Google Sheets to give the agent context. |
| Aggregate History | n8n-nodes-base.aggregate | Aggregates all sheet rows into one array for prompt context | Load Chat History | Study Agent | ## Load conversation history\nFetches the student's past Q&A from Google Sheets to give the agent context. |
| Study Agent | @n8n/n8n-nodes-langchain.agent | Generates the answer and optionally uses tools | Aggregate History; Gemini Model; Search Materials Tool | Reply via LINE | ## AI Agent\nJudges intent, searches Drive materials, generates a reply, and decides whether to escalate to a teacher via Slack. |
| Gemini Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides Gemini chat model to the AI Agent |  | Study Agent | ## Agent tools\nDrive: search materials. Sheets: log the result. Slack: alert teacher if unresolved. |
| Search Materials Tool | @n8n/n8n-nodes-langchain.toolWorkflow | Exposes a workflow tool for searching materials |  | Study Agent | ## Agent tools\nDrive: search materials. Sheets: log the result. Slack: alert teacher if unresolved. |
| Reply via LINE | n8n-nodes-base.httpRequest | Sends the AI answer back through LINE Reply API | Study Agent |  | ## Reply to student\nSends the agent's answer back via the LINE Reply API. |
| Sticky Note — Overview | n8n-nodes-base.stickyNote | Canvas documentation for workflow purpose and setup |  |  | ### How it works\n\nStudents send any study question to your LINE bot. The AI Agent reads their conversation history from Google Sheets, searches your uploaded teaching materials in Google Drive, and replies with a clear explanation directly in LINE.\n\nIf the agent can't find a solid answer in the materials, it flags the question and pings the designated Slack channel so a teacher can step in.\n\nEvery question is logged to Google Sheets with the student's ID, subject, and whether it was resolved — giving you a clear picture of what topics need more attention.\n\n### Setup steps\n\n1. Add your LINE Messaging API credentials.\n2. Set the Google Sheets ID (conversation log) in the **Set Config** node.\n3. Set the Google Drive folder ID (teaching materials) in the **Set Config** node.\n4. Add your Slack webhook URL in the **Set Config** node.\n5. Activate and share the LINE bot link with students. |
| Sticky Note — Input | n8n-nodes-base.stickyNote | Canvas documentation for inbound message handling |  |  | ## Receive student message\nCaptures incoming LINE messages and extracts the student's user ID and text. |
| Sticky Note — Config | n8n-nodes-base.stickyNote | Canvas documentation for environment configuration |  |  | ## Set all config here\nUpdate Sheet ID, Drive folder ID, Slack webhook URL, and LINE token in this single node before activating. |
| Sticky Note — History | n8n-nodes-base.stickyNote | Canvas documentation for history retrieval |  |  | ## Load conversation history\nFetches the student's past Q&A from Google Sheets to give the agent context. |
| Sticky Note — Agent | n8n-nodes-base.stickyNote | Canvas documentation for AI behavior |  |  | ## AI Agent\nJudges intent, searches Drive materials, generates a reply, and decides whether to escalate to a teacher via Slack. |
| Sticky Note — Tools | n8n-nodes-base.stickyNote | Canvas documentation for intended tool architecture |  |  | ## Agent tools\nDrive: search materials. Sheets: log the result. Slack: alert teacher if unresolved. |
| Sticky Note — Reply | n8n-nodes-base.stickyNote | Canvas documentation for outbound message delivery |  |  | ## Reply to student\nSends the agent's answer back via the LINE Reply API. |

---

# 4. Reproducing the Workflow from Scratch

Below is a practical rebuild sequence for the workflow as provided, followed by the additional steps needed to make the intended design fully functional.

## A. Rebuild the workflow exactly as provided

1. **Create a new workflow**
   - Name it something like: `Support student questions via LINE with an AI Agent, Google Drive, and Slack`.

2. **Add a Webhook node**
   - Node type: `Webhook`
   - Name: `LINE Receiver`
   - HTTP Method: `POST`
   - Path: `line-study-bot`
   - Response Mode: `Using Respond to Webhook Node`
   - Save the node.

3. **Add a Respond to Webhook node**
   - Node type: `Respond to Webhook`
   - Name: `Acknowledge LINE`
   - Respond With: `JSON`
   - Response Body:
     ```json
     {"status":"ok"}
     ```
   - Connect `LINE Receiver` → `Acknowledge LINE`.

4. **Add a Set node**
   - Node type: `Set`
   - Name: `Set Config`
   - Add these fields:
     1. `userId` = `{{ $json.body.events[0].source.userId }}`
     2. `replyToken` = `{{ $json.body.events[0].replyToken }}`
     3. `userMessage` = `{{ $json.body.events[0].message.text }}`
     4. `sheetsId` = `YOUR_GOOGLE_SHEETS_ID`
     5. `driveFolderId` = `YOUR_GOOGLE_DRIVE_FOLDER_ID`
     6. `slackWebhookUrl` = `YOUR_SLACK_WEBHOOK_URL`
     7. `lineChannelToken` = `YOUR_LINE_CHANNEL_ACCESS_TOKEN`
   - Connect `LINE Receiver` → `Set Config`.

5. **Add a Google Sheets node**
   - Node type: `Google Sheets`
   - Name: `Load Chat History`
   - Operation: `Read Rows`
   - Document ID: `{{ $json.sheetsId }}`
   - Configure Google Sheets credentials.
   - Connect `Set Config` → `Load Chat History`.

6. **Configure Google Sheets credentials**
   - Use OAuth2 or service account depending on your n8n setup.
   - Ensure the target spreadsheet is shared with the service account if using service-account auth.
   - The spreadsheet should contain a history table. The description suggests columns:
     - student ID
     - timestamp
     - question
     - answer
     - resolved

7. **Add an Aggregate node**
   - Node type: `Aggregate`
   - Name: `Aggregate History`
   - Aggregate mode: `Aggregate All Item Data`
   - Connect `Load Chat History` → `Aggregate History`.

8. **Add an AI Agent node**
   - Node type: `AI Agent`
   - Name: `Study Agent`
   - Agent type: `Conversational Agent`
   - Prompt mode: define manually
   - Text:
     ```text
     Student question: {{ $('Set Config').item.json.userMessage }}

     Past conversation history (recent 10 exchanges):
     {{ JSON.stringify($json.data.slice(-10)) }}
     ```
   - System message:
     ```text
     You are a helpful study support assistant for students (elementary to high school level). Your job is to answer academic questions clearly and encouragingly.

     You have access to three tools:
     1. search_materials — Search the school's teaching materials in Google Drive. Always try this first.
     2. save_progress — Log the question and your answer to Google Sheets after you respond.
     3. escalate_to_teacher — Use this ONLY when you cannot find a confident answer in the materials. Send the student's question to the teacher via Slack.

     Rules:
     - Always search materials before answering.
     - Keep explanations simple and age-appropriate.
     - If you escalate, tell the student: 'I've asked your teacher. They'll follow up with you soon!'
     - After every response (escalated or not), call save_progress to log the interaction.
     - Respond in Japanese.
     ```
   - Connect `Aggregate History` → `Study Agent`.

9. **Add a Gemini chat model node**
   - Node type: `Google Gemini Chat Model`
   - Name: `Gemini Model`
   - Temperature: `0.3`
   - Configure Gemini API credentials.
   - Connect `Gemini Model` → `Study Agent` using the AI language model port.

10. **Configure Gemini credentials**
    - Create or use an existing Gemini / Google AI API key credential in n8n.
    - Make sure the account has access to the selected model family.

11. **Add a Tool Workflow node**
    - Node type: `Tool Workflow`
    - Name: `Search Materials Tool`
    - Tool name: `search_materials`
    - Description:
      `Search the teaching materials stored in Google Drive. Use this to find explanations, formulas, or examples related to the student's question. Input: a search query string.`
    - Workflow input schema:
      - `query` (string, required)
    - Workflow ID: `{{ $workflow.id }}`
    - Connect `Search Materials Tool` → `Study Agent` using the AI tool port.

12. **Important note about this tool**
    - This reproduces the JSON exactly, but it is not sufficient for a working search implementation.
    - The node points to the current workflow, and there is no actual search branch in the JSON.
    - To make it functional, follow section **B** below.

13. **Add an HTTP Request node**
    - Node type: `HTTP Request`
    - Name: `Reply via LINE`
    - Method: `POST`
    - URL: `https://api.line.me/v2/bot/message/reply`
    - Send Headers: enabled
    - Send Body: enabled
    - Headers:
      - `Authorization` = `Bearer {{ $('Set Config').item.json.lineChannelToken }}`
      - `Content-Type` = `application/json`
    - Body parameters:
      - `replyToken` = `{{ $('Set Config').item.json.replyToken }}`
      - `messages` = `[{\"type\":\"text\",\"text\":\"{{ $json.output }}\"}]`
    - Connect `Study Agent` → `Reply via LINE`.

14. **Configure LINE Messaging API**
    - In LINE Developers Console:
      - Create a Messaging API channel
      - Copy the Channel Access Token
      - Set the webhook URL to the production URL exposed by `LINE Receiver`
    - Replace `YOUR_LINE_CHANNEL_ACCESS_TOKEN` in `Set Config`
    - Ensure webhook delivery is enabled

15. **Activate the workflow after testing**
    - Test with the webhook URL first
    - Then switch to production URL and activate

---

## B. Additional setup required to match the intended design

The provided workflow description promises material search, progress logging, and Slack escalation. Those parts are incomplete in the JSON. To make the design actually work, add the following.

### 16. Create a dedicated sub-workflow for `search_materials`

1. Create a new workflow named `search_materials`.
2. Add the proper entry node for workflow tool inputs, depending on your n8n version:
   - Usually an input schema or workflow trigger suitable for tool workflows.
3. Expect one input:
   - `query` as string
4. Add a Google Drive node to search files inside the configured folder.
   - Credential: Google Drive OAuth2
   - Search by file name, content metadata, or list files in the folder
5. If documents are PDFs or text files:
   - Add nodes to download file contents
   - Extract text where needed
   - Compare/query content against the incoming search term
6. Return a concise result object to the calling agent, such as:
   - matched file name
   - relevant excerpt
   - confidence or relevance note
7. Update the `Search Materials Tool` node in the main workflow:
   - Replace `{{ $workflow.id }}` with the ID of this dedicated `search_materials` workflow

### 17. Create a dedicated sub-workflow for `save_progress`

1. Create a new workflow named `save_progress`.
2. Define required inputs such as:
   - `studentId` string
   - `question` string
   - `answer` string
   - `resolved` boolean or yes/no string
   - optionally `subject`
   - optionally `timestamp`
3. Add a Google Sheets node with operation `Append Row`.
4. Map the columns to your sheet schema:
   - student ID
   - timestamp
   - question
   - answer
   - resolved
5. Return a success object.
6. Add a `Tool Workflow` node in the main workflow:
   - Tool name: `save_progress`
   - Description: log the interaction to Google Sheets
   - Connect it to `Study Agent` via AI tool port

### 18. Create a dedicated sub-workflow for `escalate_to_teacher`

1. Create a new workflow named `escalate_to_teacher`.
2. Define required inputs such as:
   - `studentId`
   - `question`
   - `historySummary` or context
3. Add an HTTP Request node or Slack node.
4. If using webhook:
   - URL = your Slack incoming webhook URL
   - Method = `POST`
   - Body example:
     ```json
     {
       "text": "Student question needs teacher follow-up.\nStudent: ...\nQuestion: ..."
     }
     ```
5. Return a success object.
6. Add a `Tool Workflow` node in the main workflow:
   - Tool name: `escalate_to_teacher`
   - Connect it to `Study Agent` via AI tool port

### 19. Improve Google Sheets history loading

As currently configured, `Load Chat History` likely reads all rows. To align with the intended logic:

1. Add filtering so only the current student’s rows are loaded.
2. If your Google Sheets node version supports filtering, filter by:
   - `student ID = {{ $('Set Config').item.json.userId }}`
3. Otherwise:
   - Read rows
   - Add an `IF` or `Filter`/`Code` node to keep only matching student rows
4. Sort by timestamp if necessary before aggregation.

### 20. Add LINE event validation

For production safety:

1. Add an `IF` node after `LINE Receiver`
2. Validate that:
   - `body.events[0]` exists
   - event type is a message
   - message type is text
3. Only proceed for text messages
4. For unsupported event types, still acknowledge the webhook but do not continue processing

### 21. Protect secrets properly

Instead of hardcoding secrets in `Set Config`:

1. Move the LINE token to credentials or environment variables
2. Move the Slack webhook URL to credentials or environment variables
3. Consider using n8n variables for:
   - Sheet ID
   - Drive folder ID
   - Slack webhook URL
   - LINE token

### 22. Handle LINE reply token timing

Because reply tokens expire quickly:

1. Keep AI processing fast
2. If processing may exceed the reply window:
   - switch from Reply API to Push API where appropriate
   - or reduce tool latency and search scope

### 23. Escape reply text safely

The current `messages` body is built as a JSON-like string expression. A safer pattern is:

1. Send a proper JSON object body
2. Use structured JSON fields instead of manual string interpolation
3. Ensure newlines and quotes in the AI output are escaped properly

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| This workflow is intended for tutoring schools, cram schools, and independent teachers who want to provide 24/7 Q&A support via LINE. | Project description |
| The described data flow is: LINE message → history from Google Sheets → material search in Google Drive → AI answer in Japanese → optional Slack escalation → logging to Google Sheets. | Functional intent |
| The provided JSON does not include actual nodes for `save_progress` or `escalate_to_teacher`, even though the AI prompt expects them. | Implementation gap |
| The provided JSON also does not include a complete Google Drive search implementation, despite the presence of a `search_materials` workflow tool node. | Implementation gap |
| Suggested spreadsheet columns from the description: student ID, timestamp, question, answer, resolved (yes/no). | Data schema |
| Required external services: LINE Messaging API, Google Sheets, Google Drive, Slack incoming webhook, Gemini API. | Dependencies |
| Setup note preserved from canvas: Add your LINE Messaging API credentials; set the Google Sheets ID, Google Drive folder ID, Slack webhook URL, and activate/share the bot link with students. | From workflow note |
| Workflow title in JSON differs slightly from the supplied title: `Support student questions via LINE with an AI Agent, Google Drive, and Slack`. | Naming note |

## Final implementation warning

As supplied, this workflow is a strong partial scaffold, not a complete end-to-end production build. The inbound LINE handling, history loading, Gemini response generation, and reply delivery are present. The promised Google Drive search, Slack escalation, and Google Sheets progress logging require additional nodes or separate sub-workflows before the workflow can fully match its own description.