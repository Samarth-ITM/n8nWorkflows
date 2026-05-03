Convert Fireflies Transcripts to Meeting Summaries with Gemini AI to Slack & ClickUp

https://n8nworkflows.xyz/workflows/convert-fireflies-transcripts-to-meeting-summaries-with-gemini-ai-to-slack---clickup-8592


# Convert Fireflies Transcripts to Meeting Summaries with Gemini AI to Slack & ClickUp

### 1. Workflow Overview

This workflow automates the conversion of Fireflies meeting transcripts into concise meeting summaries and actionable tasks. It then distributes the summary to a Slack channel and creates tasks in ClickUp based on action items extracted from the transcript. The workflow is triggered by a webhook from Fireflies upon transcription completion.

Logical blocks:

- **1.1 Webhook & Transcript Retrieval:** Receives a webhook notification from Fireflies when transcription completes and fetches the full transcript by meeting ID.
- **1.2 Transcript Processing:** Splits the transcript into sentences, then aggregates them back into a single text block for AI processing.
- **1.3 AI Processing:** Sends the aggregated transcript to the Google Gemini AI model with instructions to generate a concise summary and a list of action items.
- **1.4 Post-processing AI Output:** Parses and cleans the AI response, extracts action items as structured data.
- **1.5 Outputs:** Sends the meeting summary to Slack and creates tasks in ClickUp for each action item.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Webhook & Transcript Retrieval

- **Overview:**  
  This block listens for a Fireflies webhook signaling transcription completion, then fetches the full transcript using the meeting ID from the webhook payload.

- **Nodes Involved:**  
  - Firefly Webhook  
  - Get a transcript

- **Node Details:**

  - **Firefly Webhook**  
    - Type: Webhook (HTTP Trigger)  
    - Role: Entry point; receives POST requests from Fireflies with event data including meeting ID.  
    - Configuration: HTTP POST method, path set to `fireflies-n8n-core47-agent`.  
    - Inputs: External HTTP POST.  
    - Outputs: JSON with eventType and meetingId in body.  
    - Edge cases: Missing or malformed webhook payload, network failures, security considerations (e.g., validating source).  
    - Notes: Triggers the workflow only on “Transcription completed” events.

  - **Get a transcript**  
    - Type: Fireflies Node (custom integration)  
    - Role: Fetches the full transcript matching the provided meeting ID.  
    - Configuration: Uses expression `{{$json.body.meetingId}}` to extract meetingId from webhook data.  
    - Credentials: Fireflies API credentials required.  
    - Inputs: Meeting ID from webhook.  
    - Outputs: Transcript data including `data.sentences`.  
    - Edge cases: Invalid meetingId, API authentication errors, transcript not ready or unavailable.

---

#### 2.2 Transcript Processing

- **Overview:**  
  Splits the transcript sentences into individual items, then aggregates all sentences back into a single text block to prepare for summarization.

- **Nodes Involved:**  
  - Split Out  
  - Aggregate

- **Node Details:**

  - **Split Out**  
    - Type: Split Out (array splitter)  
    - Role: Splits the transcript into multiple outputs, each representing a sentence from `data.sentences`.  
    - Configuration: Splits on field `data.sentences`.  
    - Inputs: Transcript JSON with sentences array.  
    - Outputs: Individual sentence items.  
    - Edge cases: Empty sentences array, data format mismatches.

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Combines all sentence items into a single aggregated field `raw_text` for AI input.  
    - Configuration: Aggregates the `raw_text` field from each input item.  
    - Inputs: Multiple sentence items.  
    - Outputs: Single item with aggregated raw_text string.  
    - Edge cases: No sentences to aggregate, malformed text fields.

---

#### 2.3 AI Processing

- **Overview:**  
  Uses Google Gemini AI via LangChain integration to generate a meeting summary and action item list from the aggregated transcript text.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - Summary Generator

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini LLM Node  
    - Role: Provides AI language model capability to process prompts and generate responses.  
    - Configuration: Model set to `models/gemini-2.0-flash-thinking-exp-1219`.  
    - Credentials: Google Palm API credentials required.  
    - Inputs: Prompt text (from Summary Generator).  
    - Outputs: AI-generated response.  
    - Edge cases: API authentication errors, rate limits, model unavailability, network issues.

  - **Summary Generator**  
    - Type: LangChain Agent  
    - Role: Constructs prompt instructing AI to generate a 5-6 line summary and a list of action items with titles and descriptions.  
    - Configuration: Prompt includes detailed instructions and embedded transcript text using expression `{{$json.raw_text}}`.  
    - Inputs: Aggregated transcript text.  
    - Outputs: JSON string with keys for summary and action items.  
    - Edge cases: Prompt formatting errors, AI response malformation, empty or irrelevant AI output.

---

#### 2.4 Post-processing AI Output

- **Overview:**  
  Cleans and parses the AI response JSON, then extracts action items into structured objects for task creation.

- **Nodes Involved:**  
  - Cleans AI response  
  - Extracts action items

- **Node Details:**

  - **Cleans AI response**  
    - Type: Code (JavaScript)  
    - Role: Removes code fences (```json ... ```) from AI response and parses JSON to proper object.  
    - Configuration: JS code uses regex replacement and JSON.parse on the AI output field.  
    - Inputs: Raw AI response string.  
    - Outputs: Parsed JSON object with summary and action_items keys.  
    - Edge cases: Malformed AI output, JSON parse errors, missing keys.

  - **Extracts action items**  
    - Type: Code (JavaScript)  
    - Role: Extracts action items array from parsed JSON, normalizes each item to have `title` and `description` fields.  
    - Configuration: JS code maps over `action_items` array converting `name` to `title`.  
    - Inputs: Parsed AI JSON.  
    - Outputs: Array of objects each with `title` and `description` for task creation.  
    - Edge cases: Missing or empty action_items, wrong data structure, empty titles.

---

#### 2.5 Outputs

- **Overview:**  
  Posts the meeting summary to a Slack channel and creates tasks in ClickUp for each extracted action item.

- **Nodes Involved:**  
  - Send a message (Slack)  
  - Create a task (ClickUp)

- **Node Details:**

  - **Send a message**  
    - Type: Slack  
    - Role: Sends the generated summary text to a specified Slack channel.  
    - Configuration: Message text set to `{{$json.summary}}`, channel ID configured (e.g., `C099QR902TU`), OAuth2 authentication.  
    - Inputs: Parsed AI summary JSON.  
    - Outputs: Slack message post confirmation.  
    - Edge cases: Slack API auth errors, invalid channel ID, message formatting issues.

  - **Create a task**  
    - Type: ClickUp  
    - Role: Creates a task for each action item with title and description.  
    - Configuration:  
      - List, team, and space IDs configured (e.g., list: `901810841552`)  
      - Task name from `{{$json.title}}`  
      - Task content from `{{$json.description}}`  
      - OAuth2 authentication.  
    - Inputs: Array of action items from Extracts action items node.  
    - Outputs: ClickUp task creation responses.  
    - Edge cases: API auth failures, invalid folder/list IDs, empty task data.

---

### 3. Summary Table

| Node Name          | Node Type                  | Functional Role                      | Input Node(s)           | Output Node(s)                 | Sticky Note                                                  |
|--------------------|----------------------------|------------------------------------|------------------------|-------------------------------|--------------------------------------------------------------|
| Firefly Webhook    | Webhook                    | Entry trigger on transcription done | None                   | Get a transcript              | ## Webhook + Transcript Retrieval<br>Webhook: Trigger when Fireflies finishes a meeting transcription. Receives meetingId.<br>Get a transcript: Fetches the full transcript from Fireflies using the provided meetingId. |
| Get a transcript   | Fireflies Node             | Fetches meeting transcript          | Firefly Webhook        | Split Out                     | Same as above                                                |
| Split Out          | Split Out                  | Splits transcript into sentences    | Get a transcript       | Aggregate                     | ## Transcript Processing<br>Split Out: Splits transcript into individual sentences (data.sentences).<br>Aggregate: Rejoins sentences into one block of raw text for summarization. |
| Aggregate          | Aggregate                  | Aggregates sentences into raw text  | Split Out              | Summary Generator             | Same as above                                                |
| Google Gemini Chat Model | LangChain LLM (Google Gemini) | AI model provider for summary generation | Summary Generator (ai_languageModel) | Summary Generator             | ## AI Processing<br>Summary Generator: Sends transcript text to the LLM with instructions to generate summary and action items.<br>Google Gemini Chat Model: Connected as the LLM provider to power the AI Agent. |
| Summary Generator  | LangChain Agent            | Generates meeting summary & tasks   | Aggregate              | Cleans AI response            | Same as above                                                |
| Cleans AI response | Code                       | Parses and cleans AI JSON output    | Summary Generator      | Send a message, Extracts action items | ## Post-processing AI Output<br>Cleans AI response: Parses JSON, removes code fences.<br>Extracts action items: Extracts action items for tasks. |
| Extracts action items | Code                     | Extracts and formats action items   | Cleans AI response     | Create a task                 | Same as above                                                |
| Send a message     | Slack                      | Posts summary message to Slack      | Cleans AI response     | None                         | ## Outputs<br>Slack: Posts summary to team channel.<br>ClickUp: Creates tasks for each action item. |
| Create a task      | ClickUp                    | Creates tasks based on action items | Extracts action items  | None                         | Same as above                                                |
| Sticky Note        | Sticky Note                | Documentation                      | None                   | None                         | ## Webhook + Transcript Retrieval<br>Webhook: Trigger when Fireflies finishes a meeting transcription. Receives meetingId.<br>Get a transcript: Fetches the full transcript from Fireflies using the provided meetingId. |
| Sticky Note1       | Sticky Note                | Documentation                      | None                   | None                         | ## Transcript Processing<br>Split Out: Splits transcript into individual sentences (data.sentences).<br>Aggregate: Rejoins sentences into one block of raw text for summarization. |
| Sticky Note2       | Sticky Note                | Documentation                      | None                   | None                         | ## AI Processing<br>Summary Generator: Sends transcript text to the LLM with instructions to generate:<br>◉ A summary (5–6 lines, simple wording).<br>◉ A list of action items.<br>Google Gemini Chat Model: AI provider. |
| Sticky Note3       | Sticky Note                | Documentation                      | None                   | None                         | ## Post-processing AI Output<br>Cleans AI response: Parses JSON, removes code fences.<br>Extracts action items for task creation. |
| Sticky Note4       | Sticky Note                | Documentation                      | None                   | None                         | ## Outputs<br>Slack: Posts summary.<br>ClickUp: Creates tasks for each action item. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `fireflies-n8n-core47-agent`  
   - Purpose: Receive Fireflies transcription completion events, extract `meetingId` from POST body.

2. **Create Fireflies Node**  
   - Type: Fireflies API Node (custom integration)  
   - Operation: Get transcript by `transcriptId`  
   - Parameter: `transcriptId` set to expression `{{$json.body.meetingId}}` from webhook output  
   - Credentials: Set up Fireflies API credentials with appropriate API key.

3. **Create Split Out Node**  
   - Type: Split Out  
   - Field to split: `data.sentences` (array of sentences from transcript)  
   - Purpose: Split transcript into individual sentence items for processing.

4. **Create Aggregate Node**  
   - Type: Aggregate  
   - Aggregate field: `raw_text` (aggregate sentences into one text block)  
   - Purpose: Join all sentences back into a single string of text for AI input.

5. **Create LangChain Google Gemini Chat Model Node**  
   - Type: LangChain LLM Node  
   - Model Name: `models/gemini-2.0-flash-thinking-exp-1219`  
   - Credentials: Google Palm API credentials with required access.

6. **Create Summary Generator Node**  
   - Type: LangChain Agent  
   - Prompt:  
     ```
     Please generate a summary of this meeting transcript in 5-6 lines maximum. The main purpose is that the summary should be sent to the team in simple and easy words with medium length text, not very long, not very short.

     Also create a list of action items from the meeting and output it named description. Also give under 100 character title of each action item that I will use as the name.

     Here is the meeting Transcript:

     {{ $json.raw_text }}

     Return your answer as JSON with keys
     ```
   - Connect AI language model input to Google Gemini Chat Model node.

7. **Create Code Node "Cleans AI response"**  
   - Type: Code  
   - JavaScript code:  
     ```js
     return [
       {
         json: JSON.parse(
           $input.first().json.output.replace(/```json|```/g, '')
         ),
       },
     ];
     ```
   - Purpose: Remove markdown code fences and parse AI output JSON.

8. **Create Code Node "Extracts action items"**  
   - Type: Code  
   - JavaScript code:  
     ```js
     const items = $input.first().json.action_items || [];

     if (!Array.isArray(items) || items.length === 0) {
       return [];
     }

     // Convert each {name, description} into {title, description}
     return items.map(ai => ({
       json: {
         title: ai.name || "Untitled",
         description: ai.description || ""
       }
     }));
     ```
   - Purpose: Extract and normalize action items for task creation.

9. **Create Slack Node "Send a message"**  
   - Type: Slack  
   - Text: `{{$json.summary}}` (summary from cleaned AI response)  
   - Channel ID: Set to target Slack channel (e.g., `C099QR902TU`)  
   - Authentication: OAuth2 credentials for Slack API.

10. **Create ClickUp Node "Create a task"**  
    - Type: ClickUp  
    - List ID: `901810841552` (specify your target list)  
    - Team ID: `90181586895`  
    - Space ID: `90186124564`  
    - Folderless: true (no folder)  
    - Task name: `{{$json.title}}` (from action item title)  
    - Task content: `{{$json.description}}` (from action item description)  
    - Authentication: OAuth2 for ClickUp API.

11. **Connect nodes:**  
    - Webhook → Get a transcript → Split Out → Aggregate → Summary Generator → Cleans AI response  
    - Cleans AI response → Send a message  
    - Cleans AI response → Extracts action items → Create a task  
    - Summary Generator’s AI languageModel input → Google Gemini Chat Model

12. **Credentials Setup:**  
    - Fireflies API with API key.  
    - Google Palm API (Google Gemini) with API key.  
    - Slack OAuth2 with appropriate scopes to post messages.  
    - ClickUp OAuth2 with permissions to create tasks.

13. **Test Execution:**  
    - Trigger Fireflies webhook with sample meeting ID.  
    - Validate transcript retrieval, AI summarization, Slack message posting, and task creation in ClickUp.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                |
|----------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow integrates Fireflies transcription with Google Gemini AI to automate meeting summarization and task management.   | Core workflow purpose                                                                          |
| Slack channel and ClickUp space IDs must be customized per user's environment.                                                   | Slack and ClickUp node configuration                                                          |
| Fireflies webhook must be configured in Fireflies app to point to the n8n webhook URL for transcription completed events.       | Fireflies integration setup                                                                   |
| Google Gemini model used is `models/gemini-2.0-flash-thinking-exp-1219` via LangChain n8n node requiring Google PaLM API access.| Google Gemini AI via LangChain in n8n                                                         |
| For code nodes parsing AI output, ensure stable and predictable AI response formatting to avoid JSON parsing errors.             | AI output post-processing best practices                                                      |
| Slack OAuth2 must have chat:write scope to post messages; ClickUp OAuth2 must have task:create scope.                            | Credential scope requirements                                                                 |
| Sticky notes in the workflow provide detailed explanations for each logical block.                                                | Workflow documentation within n8n editor                                                      |

---

**Disclaimer:**  
The text provided is exclusively derived from a fully automated n8n workflow. It strictly complies with content policies, containing no illegal or offensive material. All data processed are legal and public.