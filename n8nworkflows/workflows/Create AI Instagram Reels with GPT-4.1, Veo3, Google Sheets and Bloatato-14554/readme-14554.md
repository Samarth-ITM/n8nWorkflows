Create AI Instagram Reels with GPT-4.1, Veo3, Google Sheets and Bloatato

https://n8nworkflows.xyz/workflows/create-ai-instagram-reels-with-gpt-4-1--veo3--google-sheets-and-bloatato-14554


# Create AI Instagram Reels with GPT-4.1, Veo3, Google Sheets and Bloatato

### 1. Workflow Overview

This workflow automates the end-to-end creation and publishing of AI-generated short-form transformation videos on Instagram. It runs on a scheduled basis (twice daily), uses GPT-4.1-mini to generate creative BEFORE/AFTER video concepts and cinematic prompts, renders those prompts into actual video via the VEO 3 API (kie.ai), uploads the result to Bloatato for media hosting, publishes the video to Instagram with a viral-ready caption, updates tracking status in Google Sheets, and sends multi-channel notifications (Telegram, WhatsApp, Slack).

**Logical Blocks:**

| Block | Name | Purpose |
|-------|------|---------|
| 1 | **Idea Generation & Storage** | Scheduled trigger fires every 12 hours → GPT-4.1-mini generates one original BEFORE/AFTER transformation concept with caption, environment, sound → structured output is parsed and saved to Google Sheets |
| 2 | **Professional Scripting** | A master JSON schema for VEO 3 is prepared → a second GPT-4.1-mini agent converts the idea + environment + sound into a fully structured cinematic video prompt conforming to that schema → output is formatted as a valid JSON body for the VEO 3 API |
| 3 | **Video Generation & Retrieval** | The structured prompt is sent to the VEO 3 generation endpoint → the workflow waits 3 minutes for initial rendering, then polls every 30 seconds until the video status is `SUCCESS` → a retry loop handles the still-processing case |
| 4 | **Publishing** | The finished video URL is uploaded to Bloatato's media endpoint → Bloatato is used to publish the video to Instagram with the idea text as caption → the Google Sheet row is updated from "for production" to "Publish" |
| 5 | **Notification** | Upon successful publishing, parallel notifications are sent via Telegram, WhatsApp (Rapiwa), and Slack |

---

### 2. Block-by-Block Analysis

---

#### Block 1: Idea Generation & Storage

**Overview:** A scheduled trigger launches the workflow twice daily. An AI agent powered by GPT-4.1-mini generates a single, original BEFORE/AFTER transformation video concept with a viral caption, environment description, and sound design. The output is parsed via a structured output parser and appended as a new row to a Google Sheet for persistent tracking.

**Nodes Involved:**
- Start Daily Two Times
- OpenAI Chat Model
- Make a Creative Video Idea
- Tool (Make More Creative Thinking in Action)
- Parse AI Output
- Save Idea in google sheet

---

**Start Daily Two Times**
- **Type:** Schedule Trigger (n8n-nodes-base.scheduleTrigger) v1.2
- **Technical Role:** Workflow entry point; fires every 12 hours.
- **Configuration:** Interval-based schedule, `hoursInterval: 12`.
- **Key expressions/variables:** None.
- **Input connections:** None (trigger node).
- **Output connections:** → Make a Creative Video Idea (main, index 0).
- **Edge cases:** If the workflow execution takes longer than 12 hours, a second execution may overlap. No concurrency guard is in place.

---

**OpenAI Chat Model**
- **Type:** LangChain Chat Model (n8n-nodes-langchain.lmChatOpenAi) v1.3
- **Technical Role:** Provides the LLM backend for the "Make a Creative Video Idea" agent.
- **Configuration:** Model set to `gpt-4.1-mini`; no built-in tools enabled; no additional options.
- **Key expressions/variables:** None.
- **Input connections:** None (sub-node connected via ai_languageModel).
- **Output connections:** → Make a Creative Video Idea (ai_languageModel, index 0).
- **Credential:** OpenAI API key (`openAiApi`).
- **Edge cases:** Rate limiting on OpenAI API; token limit for `gpt-4.1-mini`; invalid/expired API key will cause the entire agent to fail.

---

**Tool (Make More Creative Thinking in Action)**
- **Type:** LangChain Think Tool (n8n-nodes-langchain.toolThink) v1
- **Technical Role:** Provides an internal "thinking" scratchpad tool that the AI agent can invoke to reason through creative decisions before producing final output.
- **Configuration:** Description states it records internal reasoning; does not retrieve external information or modify databases.
- **Key expressions/variables:** None.
- **Input connections:** None (sub-node connected via ai_tool).
- **Output connections:** → Make a Creative Video Idea (ai_tool, index 0).
- **Edge cases:** If the agent over-uses the think tool, it may consume tokens without producing output.

---

**Parse AI Output**
- **Type:** LangChain Structured Output Parser (n8n-nodes-langchain.outputParserStructured) v1.2
- **Technical Role:** Enforces structured JSON output from the idea-generation agent, matching the schema with fields: Caption, Idea, Environment, Sound, Status.
- **Configuration:** JSON schema example provided:
  ```json
  [{
    "Caption": "Stray dog gets a second chance 🐶❤️ #dogrescue ...",
    "Idea": "A homeless injured dog is rescued from the street and slowly recovers with human care",
    "Environment": "Urban street to cozy shelter transition, warm lighting, emotional storytelling, before-and-after transformation",
    "Sound": "Soft piano music with gentle city ambience fading into calm indoor sounds",
    "Status": "Ready for production"
  }]
  ```
- **Key expressions/variables:** None.
- **Input connections:** None (sub-node connected via ai_outputParser).
- **Output connections:** → Make a Creative Video Idea (ai_outputParser, index 0).
- **Edge cases:** If GPT-4.1-mini deviates from the expected schema, the parser may throw a validation error. The agent's system prompt includes strict formatting instructions to mitigate this.

---

**Make a Creative Video Idea**
- **Type:** LangChain Agent (n8n-nodes-langchain.agent) v1.9
- **Technical Role:** Core AI agent that generates a single original BEFORE/AFTER transformation concept optimized for vertical TikTok/Reels content.
- **Configuration:**
  - **Prompt type:** Defined (not chat-triggered).
  - **User prompt:** Instructs the agent to generate a highly original visual concept centered on a non-human subject, with clear BEFORE/AFTER states, transition, style descriptors, color palette, camera movement, and lighting. Output must be a single-line JSON array.
  - **System message:** Detailed role ("AI specialized in generating one original BEFORE/AFTER transformation video concept"), with rules for exactly one idea, theme (Before/After transformation), idea max 15 words, caption with emoji + 12 hashtags (4 topic + 4 all-time popular + 4 trending), environment max 20 words, sound max 15 words, status always "for production". Output format: single-line JSON array.
  - **Has output parser:** Yes (connected to Parse AI Output).
  - **Connected language model:** OpenAI Chat Model.
  - **Connected tool:** Tool (Make More Creative Thinking in Action).
- **Key expressions/variables:** None in the prompt itself (the agent generates content from its instructions).
- **Input connections:** Start Daily Two Times (main), OpenAI Chat Model (ai_languageModel), Parse AI Output (ai_outputParser), Tool (Make More Creative Thinking in Action) (ai_tool).
- **Output connections:** → Save Idea in google sheet (main, index 0).
- **Edge cases:** Agent may generate more than one idea despite instructions; may include humans or brands; may not format as single-line JSON. The structured output parser helps enforce the schema.

---

**Save Idea in google sheet**
- **Type:** Google Sheets (n8n-nodes-base.googleSheets) v4.5
- **Technical Role:** Persists the AI-generated idea, caption, environment, sound, and status to a Google Sheet for tracking.
- **Configuration:**
  - **Operation:** Append.
  - **Document:** `1o8a6roAbvIUyJ6yZ5u3fsom3ubr9g6A0lCzfx961V-M` (named "Save Video Idea").
  - **Sheet:** `gid=0` (Sheet1).
  - **Mapping mode:** Define below.
  - **Matching columns:** `id`.
  - **Column mappings:**
    - `idea` ← `{{ $json.output[0].Idea }}`
    - `caption` ← `{{ $json.output[0].Caption }}`
    - `production` ← `{{ $json.output[0].Status }}`
    - `sound_prompt` ← `{{ $json.output[0].Sound }}`
    - `environment_prompt` ← `{{ $json.output[0].Environment }}`
- **Key expressions/variables:** References the agent's output via `$json.output[0]`.
- **Input connections:** Make a Creative Video Idea (main, index 0).
- **Output connections:** → Make Professional Prompt (main, index 0).
- **Credential:** Google Sheets OAuth2 API (`googleSheetsOAuth2Api`).
- **Edge cases:** Google Sheets API rate limits; OAuth2 token expiration; sheet permissions; if the agent output structure doesn't match `output[0].Idea` etc., expressions will resolve to undefined.

---

#### Block 2: Professional Scripting

**Overview:** This block prepares a comprehensive master JSON schema (the VEO 3 video prompt template) and feeds it alongside the saved idea data into a second AI agent. That agent generates a structured cinematic video prompt conforming exactly to the VEO 3 schema, with a `title` and a `final_prompt` (stringified JSON). The output is then code-processed to ensure proper JSON escaping before being sent to the video generation API.

**Nodes Involved:**
- Make Professional Prompt
- OpenAI Chat Model1
- AI Agent (Create Video Script)
- Think
- Structured Output Parser
- Code (Format Prompt)

---

**Make Professional Prompt**
- **Type:** Set (n8n-nodes-base.set) v3.4
- **Technical Role:** Prepares the master JSON schema and video generation parameters for the scripting agent.
- **Configuration:**
  - **Assignments:**
    - `json_master` (string): A comprehensive JSON object defining the VEO 3 prompt schema with fields: description, style, camera (type, movement, lens), lighting (type, sources, FX), environment (location, set_pieces, mood), elements, subject (character, product), motion, VFX, audio, ending, text, format, keywords.
    - `model` (string): `"veo3_fast"`
    - `aspectRatio` (string): `"9:16"`
- **Key expressions/variables:** All values are static strings defined inline.
- **Input connections:** Save Idea in google sheet (main, index 0).
- **Output connections:** → AI Agent (Create Video Script) (main, index 0).
- **Edge cases:** If the `json_master` string is malformed JSON, downstream agents may produce invalid prompts. The string is embedded as a raw string value, not a parsed object.

---

**OpenAI Chat Model1**
- **Type:** LangChain Chat Model (n8n-nodes-langchain.lmChatOpenAi) v1.3
- **Technical Role:** Provides the LLM backend for the "AI Agent (Create Video Script)" agent.
- **Configuration:** Model set to `gpt-4.1-mini`; no built-in tools; no additional options.
- **Key expressions/variables:** None.
- **Input connections:** None (sub-node).
- **Output connections:** → AI Agent (Create Video Script) (ai_languageModel, index 0).
- **Credential:** OpenAI API key (`openAiApi`).
- **Edge cases:** Same as OpenAI Chat Model above.

---

**Think**
- **Type:** LangChain Think Tool (n8n-nodes-langchain.toolThink) v1
- **Technical Role:** Internal reasoning scratchpad for the scripting agent to review and refine its structured prompt before outputting.
- **Configuration:** Description states it records internal reasoning; the agent is explicitly instructed to "Use the Think tool to review your output."
- **Key expressions/variables:** None.
- **Input connections:** None (sub-node).
- **Output connections:** → AI Agent (Create Video Script) (ai_tool, index 0).
- **Edge cases:** Same as Tool (Make More Creative Thinking in Action) above.

---

**Structured Output Parser**
- **Type:** LangChain Structured Output Parser (n8n-nodes-langchain.outputParserStructured) v1.3
- **Technical Role:** Enforces the output schema of the scripting agent to produce exactly `{ "title": "string", "final_prompt": "string" }`.
- **Configuration:** JSON schema example:
  ```json
  { "title": "string", "final_prompt": "string" }
  ```
- **Key expressions/variables:** None.
- **Input connections:** None (sub-node).
- **Output connections:** → AI Agent (Create Video Script) (ai_outputParser, index 0).
- **Edge cases:** If the agent fails to produce valid JSON with both keys, the parser will raise an error. The `final_prompt` must be a single-line stringified JSON, which requires careful escaping.

---

**AI Agent (Create Video Script)**
- **Type:** LangChain Agent (n8n-nodes-langchain.agent) v2
- **Technical Role:** Converts the raw idea, environment, and sound into a structured, VEO 3-compatible cinematic video prompt.
- **Configuration:**
  - **Prompt type:** Defined.
  - **User prompt:** Instructs the agent to create a BEFORE/AFTER transformation video prompt using:
    - `idea`: `{{ $('Save Idea in google sheet').item.json.idea }}`
    - `environment`: `{{ $('Save Idea in google sheet').item.json.environment_prompt }}`
    - `sound`: `{{ $('Save Idea in google sheet').item.json.sound_prompt }}`
  - Rules: cinematic style, vertical 9:16, include BEFORE/AFTER/TRANSITION/CAMERA/LIGHTING/COLOR PALETTE/MOOD, default model veo3_fast, output one JSON with `title` and `final_prompt`, use Think tool.
  - **System message:** Extensive structured prompt following the "A-G-N" (Ask-Guidance-Notation) pattern. Role: Creative Director. Output must be valid JSON, `title` max 15 words, `final_prompt` must be a single-line stringified JSON matching `{{ $json.json_master }}`. No markdown or explanations. Escaped inner quotes required.
  - **Has output parser:** Yes (Structured Output Parser).
  - **Connected language model:** OpenAI Chat Model1.
  - **Connected tool:** Think.
- **Key expressions/variables:** References `Save Idea in google sheet` node for idea, environment_prompt, and sound_prompt. References `$json.json_master` from Make Professional Prompt.
- **Input connections:** Make Professional Prompt (main), OpenAI Chat Model1 (ai_languageModel), Structured Output Parser (ai_outputParser), Think (ai_tool).
- **Output connections:** → Code (Format Prompt) (main, index 0).
- **Edge cases:** Cross-node expression references (`$('Save Idea in google sheet')`) may fail if that node hasn't produced output yet or the data structure doesn't match. The `final_prompt` being a stringified JSON inside JSON requires double-escaping, which the system prompt addresses.

---

**Code (Format Prompt)**
- **Type:** Code (n8n-nodes-base.code) v2
- **Technical Role:** Extracts the `final_prompt` from the agent's output, stringifies it, and packages it with model and aspect ratio into the body format required by the VEO 3 API.
- **Configuration:**
  - **JavaScript code:**
    ```javascript
    const structuredPrompt = $input.first().json.output.final_prompt;
    return {
      json: {
        prompt: JSON.stringify(structuredPrompt),
        model: "veo3_fast",
        aspectRatio: "9:16"
      }
    };
    ```
- **Key expressions/variables:** `$input.first().json.output.final_prompt` — reads the agent's parsed output.
- **Input connections:** AI Agent (Create Video Script) (main, index 0).
- **Output connections:** → HTTP (Create Video with VEO3) (main, index 0).
- **Edge cases:** If `final_prompt` is already a string rather than an object, `JSON.stringify()` will double-stringify it. If the output structure is unexpected (e.g., `output` is an array), the code will fail.

---

#### Block 3: Video Generation & Retrieval

**Overview:** The formatted prompt is sent to the VEO 3 API for video rendering. After an initial 3-minute wait, the workflow polls the API for status. If the video is still processing, it waits 30 seconds and retries. Once the status is `SUCCESS`, the flow proceeds to the publishing block.

**Nodes Involved:**
- HTTP (Create Video with VEO3)
- Wait 3m for Rendering video
- HTTP (Download Video from VEO3)
- If (Check Video Status)
- Wait 30s if video still processing

---

**HTTP (Create Video with VEO3)**
- **Type:** HTTP Request (n8n-nodes-base.httpRequest) v4.2
- **Technical Role:** Sends the structured video prompt to the VEO 3 generation API.
- **Configuration:**
  - **Method:** POST.
  - **URL:** `https://api.kie.ai/api/v1/veo/generate`.
  - **Authentication:** Generic credential type — Bearer Auth (`httpBearerAuth`).
  - **Headers:** `Authorization: Bearer YOUR_TOKEN_HERE` (placeholder; must be replaced with actual token).
  - **Body type:** JSON (specified).
  - **JSON Body:**
    ```json
    {
      "prompt": {{ $json.prompt }},
      "model": "{{ $('Make Professional Prompt').item.json.model }}",
      "aspectRatio": "{{ $('Make Professional Prompt').item.json.aspectRatio }}"
    }
    ```
- **Key expressions/variables:** `$json.prompt` from Code node, references to Make Professional Prompt for model and aspectRatio.
- **Input connections:** Code (Format Prompt) (main, index 0).
- **Output connections:** → Wait 3m for Rendering video (main, index 0).
- **Credential:** Bearer Auth (`httpBearerAuth`, named "Bearer Auth account 3").
- **Edge cases:** Invalid/expired bearer token; VEO 3 API rate limits; prompt content violating VEO 3 content policies; the `$json.prompt` expression is injected without quotes in the JSON body — n8n handles this as an expression within the JSON body field.

---

**Wait 3m for Rendering video**
- **Type:** Wait (n8n-nodes-base.wait) v1.1
- **Technical Role:** Pauses execution for 3 minutes to allow VEO 3 to begin rendering the video before polling.
- **Configuration:** Amount: 3 (unit not shown but defaults to minutes in the UI context based on the node naming).
- **Webhook ID:** `f8f1a8a7-0870-4f09-b732-425a8937f229`.
- **Key expressions/variables:** None.
- **Input connections:** HTTP (Create Video with VEO3) (main, index 0).
- **Output connections:** → HTTP (Download Video from VEO3) (main, index 0).
- **Edge cases:** 3 minutes may be insufficient for complex prompts; the polling loop handles extended wait times. The Wait node uses a webhook to resume execution, which requires the n8n instance to be publicly accessible or properly configured for webhooks.

---

**HTTP (Download Video from VEO3)**
- **Type:** HTTP Request (n8n-nodes-base.httpRequest) v4.2
- **Technical Role:** Checks the status and retrieves the video URL from the VEO 3 API using the task ID from the previous creation step.
- **Configuration:**
  - **Method:** GET (default).
  - **URL:** `https://api.kie.ai/api/v1/veo/record-info`.
  - **Authentication:** Generic credential type — Header Auth (`httpHeaderAuth`).
  - **Query parameters:** `taskId` = `{{ $('HTTP (Create Video with VEO3)').item.json.data.taskId }}`.
- **Key expressions/variables:** Cross-node reference to `HTTP (Create Video with VEO3)` for the `taskId`.
- **Input connections:** Wait 3m for Rendering video (main, index 0).
- **Output connections:** → If (Check Video Status) (main, index 0).
- **Credential:** HTTP Header Auth (`httpHeaderAuth`, named "Veo3").
- **Edge cases:** If the creation step failed and `data.taskId` is undefined, the query parameter will be empty and the API call will fail. The API may return an error status rather than `SUCCESS` or `PROCESSING`.

---

**If (Check Video Status)**
- **Type:** IF (n8n-nodes-base.if) v2.2
- **Technical Role:** Branches based on whether the VEO 3 video status equals `SUCCESS`.
- **Configuration:**
  - **Conditions:** Single condition — `$json.data.status` equals (string, case-sensitive, strict type validation) `SUCCESS`.
  - **Combinator:** AND (single condition, so irrelevant).
- **Key expressions/variables:** `$json.data.status`.
- **Input connections:** HTTP (Download Video from VEO3) (main, index 0), Wait 30s if video still processing (main, index 0).
- **Output connections:**
  - TRUE (index 0) → HTTP (Upload Bloatato Media)
  - FALSE (index 1) → Wait 30s if video still processing
- **Edge cases:** If the API returns a status other than `SUCCESS` or `PROCESSING` (e.g., `FAILED`, `ERROR`), the workflow will loop indefinitely on the 30-second retry. There is no maximum retry count or timeout.

---

**Wait 30s if video still processing**
- **Type:** Wait (n8n-nodes-base.wait) v1.1
- **Technical Role:** Pauses execution for 30 seconds before re-checking video status, creating a polling loop.
- **Configuration:** Amount: 30 (seconds based on node naming).
- **Webhook ID:** `0f2d17ef-dde0-41d6-b29e-0896907f75a0`.
- **Key expressions/variables:** None.
- **Input connections:** If (Check Video Status) FALSE branch (main, index 1).
- **Output connections:** → If (Check Video Status) (main, index 0).
- **Edge cases:** Infinite loop if video never reaches `SUCCESS` status. No circuit breaker or max-retry logic. Each iteration consumes a webhook resume call.

---

#### Block 4: Publishing

**Overview:** Once the video is successfully generated, its URL is uploaded to Bloatato's media hosting, then published to Instagram via Bloatato's posting API. Finally, the Google Sheet row is updated to reflect the "Publish" status.

**Nodes Involved:**
- HTTP (Upload Bloatato Media)
- HTTP (Publish to Instagram)
- Update Status to "Publish"

---

**HTTP (Upload Bloatato Media)**
- **Type:** HTTP Request (n8n-nodes-base.httpRequest) v4.2
- **Technical Role:** Uploads the rendered video URL to Bloatato's media hosting service.
- **Configuration:**
  - **Method:** POST.
  - **URL:** `https://backend.blotato.com/v2/media`.
  - **Headers:** `Authorization: Bearer YOUR_TOKEN_HERE` (placeholder).
  - **Body parameters:** `url` = `{{ $json["data"]["videoUrl"] }}`.
- **Key expressions/variables:** `$json.data.videoUrl` from the VEO 3 download response.
- **Input connections:** If (Check Video Status) TRUE branch (main, index 0).
- **Output connections:** → HTTP (Publish to Instagram) (main, index 0).
- **Credential:** None explicitly configured in credentials; bearer token is hardcoded in header (should be moved to credential).
- **Edge cases:** If `videoUrl` is undefined or the URL has expired, the upload will fail. Hardcoded bearer token is a security risk; should be replaced with a proper credential.

---

**HTTP (Publish to Instagram)**
- **Type:** HTTP Request (n8n-nodes-base.httpRequest) v4.2
- **Technical Role:** Publishes the video to Instagram via Bloatato's posting API.
- **Configuration:**
  - **Method:** POST.
  - **URL:** `https://backend.blotato.com/v2/posts`.
  - **Authentication:** Generic credential type — Header Auth (`httpHeaderAuth`).
  - **Body type:** JSON (specified).
  - **JSON Body:**
    ```json
    {
      "post": {
        "target": { "targetType": "instagram" },
        "content": {
          "text": {{ $('Save Idea in google sheet').first().json.idea }},
          "platform": "instagram",
          "mediaUrls": ["{{ $json.url }}"]
        },
        "accountId": "add_your_instagram_ID"
      }
    }
    ```
- **Key expressions/variables:** `$('Save Idea in google sheet').first().json.idea` for caption text, `$json.url` for the Bloatato media URL.
- **Input connections:** HTTP (Upload Bloatato Media) (main, index 0).
- **Output connections:** → Update Status to "Publish" (main, index 0).
- **Credential:** HTTP Header Auth (`httpHeaderAuth`, named "Veo3" — note: credential name appears reused; verify correct credentials in production).
- **Edge cases:** `accountId` is a placeholder ("add_your_instagram_ID") and must be replaced. The `text` field uses the idea (not the caption), which may not be the desired Instagram post caption. Bloatato API may reject if the media URL is invalid or the Instagram account is not connected.

---

**Update Status to "Publish"**
- **Type:** Google Sheets (n8n-nodes-base.googleSheets) v4.5
- **Technical Role:** Updates the Google Sheet row to mark the idea as published.
- **Configuration:**
  - **Operation:** Append or Update.
  - **Document:** Same as Save Idea (`1o8a6roAbvIUyJ6yZ5u3fsom3ubr9g6A0lCzfx961V-M`).
  - **Sheet:** Sheet1.
  - **Matching columns:** `idea`.
  - **Column mappings:**
    - `idea` ← `{{ $('Save Idea in google sheet').first().json.idea }}` (used as matching key).
    - `production` ← `"Publish"`.
  - **Removed columns from schema:** caption, environment_prompt, sound_prompt (not updated).
- **Key expressions/variables:** Cross-reference to the Save Idea node for the idea value.
- **Input connections:** HTTP (Publish to Instagram) (main, index 0).
- **Output connections:** → Send Telegram Notification, Rapiwa (Send WhatsApp Notification), Send Slack Notification (all main, index 0 — parallel).
- **Credential:** Google Sheets OAuth2 API.
- **Edge cases:** If the idea string has been modified or there are duplicate ideas, the append-or-update operation may match the wrong row or create a duplicate. The matching is case-sensitive.

---

#### Block 5: Notification

**Overview:** After successful publishing, three parallel notifications are dispatched to Telegram, WhatsApp (via Rapiwa), and Slack, each containing the publication timestamp and the video idea title.

**Nodes Involved:**
- Send Telegram Notification
- Rapiwa (Send WhatsApp Notification)
- Send Slack Notification

---

**Send Telegram Notification**
- **Type:** Telegram (n8n-nodes-base.telegram) v1.2
- **Technical Role:** Sends a notification message to a Telegram chat when a video is published.
- **Configuration:**
  - **Chat ID:** `"add_your_ID"` (placeholder).
  - **Text:**
    ```
    New Shorts Video Publish:
    Time: {{ $now.format("DD-MMM-YYYY t") }}
    Title: {{ $('Save Idea in google sheet').first().json.idea }}
    ```
- **Key expressions/variables:** `$now.format()` for timestamp, cross-reference to Save Idea for title.
- **Input connections:** Update Status to "Publish" (main, index 0).
- **Output connections:** None (terminal node).
- **Credential:** Telegram API (`telegramApi`).
- **Edge cases:** Invalid or placeholder chat ID will cause a 400 error. Bot must have permission to send messages to the specified chat.

---

**Rapiwa (Send WhatsApp Notification)**
- **Type:** Rapiwa (n8n-nodes-rapiwa.rapiwa) v1
- **Technical Role:** Sends a WhatsApp notification via the Rapiwa service.
- **Configuration:**
  - **Number:** `"add_your_whatsapp_number"` (placeholder).
  - **Message:**
    ```
    New Shorts Video Publish:
    Time: {{ $now.format("DD-MMM-YYYY t") }}
    Title: {{ $('Save Idea in google sheet').first().json.idea }}
    ```
- **Key expressions/variables:** Same as Telegram.
- **Input connections:** Update Status to "Publish" (main, index 0).
- **Output connections:** None (terminal node).
- **Credential:** Rapiwa API (`rapiwaApi`, named "Rapiwa (Shakil)").
- **Edge cases:** Invalid phone number; Rapiwa API rate limits or subscription limits; WhatsApp may block automated messages.

---

**Send Slack Notification**
- **Type:** Slack (n8n-nodes-base.slack) v2.4
- **Technical Role:** Sends a notification to a Slack user when a video is published.
- **Configuration:**
  - **Authentication:** OAuth2.
  - **Select:** User.
  - **User:** `@add_your_username` (placeholder, mode: username).
  - **Text:** Same format as Telegram/WhatsApp notifications.
- **Key expressions/variables:** Same timestamp and title expressions.
- **Input connections:** Update Status to "Publish" (main, index 0).
- **Output connections:** None (terminal node).
- **Credential:** Slack OAuth2 API (`slackOAuth2Api`).
- **Edge cases:** Invalid username; Slack workspace not connected; OAuth2 token expiration; app not installed in workspace.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|---------------|---------------|----------------|-------------|
| Start Daily Two Times | Schedule Trigger | Workflow entry; triggers every 12 hours | — | Make a Creative Video Idea | Idea Generation & Storage Group |
| OpenAI Chat Model | LangChain LLM ChatOpenAi | LLM backend for idea generation agent | — | Make a Creative Video Idea | Idea Generation & Storage Group |
| Tool (Make More Creative Thinking in Action) | LangChain Think Tool | Internal reasoning scratchpad for idea agent | — | Make a Creative Video Idea | Idea Generation & Storage Group |
| Parse AI Output | LangChain Structured Output Parser | Enforces JSON schema for idea output | — | Make a Creative Video Idea | Idea Generation & Storage Group |
| Make a Creative Video Idea | LangChain Agent | Generates one BEFORE/AFTER transformation concept with caption, environment, sound | Start Daily Two Times; OpenAI Chat Model; Parse AI Output; Tool (Make More Creative Thinking in Action) | Save Idea in google sheet | Idea Generation & Storage Group |
| Save Idea in google sheet | Google Sheets | Persists idea, caption, status, environment, sound to spreadsheet | Make a Creative Video Idea | Make Professional Prompt | Idea Generation & Storage Group |
| Make Professional Prompt | Set | Prepares VEO 3 master schema + model + aspect ratio | Save Idea in google sheet | AI Agent (Create Video Script) | Professional Scripting Group |
| OpenAI Chat Model1 | LangChain LLM ChatOpenAi | LLM backend for video scripting agent | — | AI Agent (Create Video Script) | Professional Scripting Group |
| Think | LangChain Think Tool | Internal reasoning scratchpad for scripting agent | — | AI Agent (Create Video Script) | Professional Scripting Group |
| Structured Output Parser | LangChain Structured Output Parser | Enforces {title, final_prompt} schema for scripting agent | — | AI Agent (Create Video Script) | Professional Scripting Group |
| AI Agent (Create Video Script) | LangChain Agent | Converts idea into structured VEO 3-compatible cinematic prompt | Make Professional Prompt; OpenAI Chat Model1; Structured Output Parser; Think | Code (Format Prompt) | Professional Scripting Group |
| Code (Format Prompt) | Code | Stringifies final_prompt and packages with model/aspect ratio | AI Agent (Create Video Script) | HTTP (Create Video with VEO3) | Professional Scripting Group |
| HTTP (Create Video with VEO3) | HTTP Request | Sends prompt to VEO 3 API to generate video | Code (Format Prompt) | Wait 3m for Rendering video | Video Generation & Retrieval Group |
| Wait 3m for Rendering video | Wait | Pauses 3 minutes for initial rendering | HTTP (Create Video with VEO3) | HTTP (Download Video from VEO3) | Video Generation & Retrieval Group |
| HTTP (Download Video from VEO3) | HTTP Request | Polls VEO 3 for video status and URL | Wait 3m for Rendering video | If (Check Video Status) | Video Generation & Retrieval Group |
| If (Check Video Status) | IF | Branches on whether video status is SUCCESS | HTTP (Download Video from VEO3); Wait 30s if video still processing | HTTP (Upload Bloatato Media) [TRUE]; Wait 30s if video still processing [FALSE] | Video Generation & Retrieval Group |
| Wait 30s if video still processing | Wait | Pauses 30 seconds before re-polling | If (Check Video Status) [FALSE] | If (Check Video Status) | Video Generation & Retrieval Group |
| HTTP (Upload Bloatato Media) | HTTP Request | Uploads video URL to Bloatato media hosting | If (Check Video Status) [TRUE] | HTTP (Publish to Instagram) | Publishing Group |
| HTTP (Publish to Instagram) | HTTP Request | Publishes video to Instagram via Bloatato | HTTP (Upload Bloatato Media) | Update Status to "Publish" | Publishing Group |
| Update Status to "Publish" | Google Sheets | Updates spreadsheet row status to Publish | HTTP (Publish to Instagram) | Send Telegram Notification; Rapiwa (Send WhatsApp Notification); Send Slack Notification | Publishing Group |
| Send Telegram Notification | Telegram | Sends publish notification via Telegram | Update Status to "Publish" | — | Notification Group |
| Rapiwa (Send WhatsApp Notification) | Rapiwa | Sends publish notification via WhatsApp | Update Status to "Publish" | — | Notification Group |
| Send Slack Notification | Slack | Sends publish notification via Slack | Update Status to "Publish" | — | Notification Group |

**Additional Sticky Note Coverage:**

The Overview sticky note covers all nodes in the workflow with the following content:

| Node Name | Sticky Note |
|-----------|-------------|
| All nodes | **Overview:** This workflow automates the creation and publishing of viral short-form videos for social media. It leverages AI to generate a unique "before/after" transformation concept, uses the VEO 3 API to render the video, and then publishes it to Instagram, complete with a viral-ready caption and hashtags. **Workflow Steps:** Scheduled Trigger – Runs automatically on a set schedule to generate new content. AI Idea Generation – GPT creates viral "before/after" concepts, captions, and video prompts. Data Storage – Saves ideas, captions, and prompts in Google Sheets. Structured Prompting – Converts ideas into JSON prompts for VEO 3 video generation. Video Generation – Sends prompts to VEO 3 API for cinematic video rendering. Polling for Completion – Checks video status every 30 seconds until ready. Media Upload – Uploads finished video to Bloatato for social media hosting. Social Publishing – Posts video and caption to Instagram via Bloatato. Status Update – Marks idea as "Publish" in Google Sheets. Multi-Channel Notifications – Sends final success alerts via WhatsApp, Telegram, and Slack. **Requirements:** n8n – Automation platform for the workflow. APIs & Keys – OpenAI (ideas & scripts), Google Sheets (tracking), VEO 3 (video generation), Bloatato (media hosting & Instagram), plus credentials for Instagram, Rapiwa, Telegram, and Slack. **Customization Options:** Content Theme – Change AI prompts for tutorials, product reveals, educational clips, etc. Schedule – Adjust the Cron node frequency. Platforms – Swap Instagram/Bloatato for TikTok, YouTube Shorts, or Facebook Reels. AI Model – Switch GPT to Anthropic Claude, Google Gemini, or others. Video Settings – Modify `aspectRatio` for vertical (9:16) or horizontal (16:9) formats. Notifications – Add/remove channels like email, Slack, WhatsApp, or Telegram. **Useful Links:** n8n Documentation: [n8n.io](https://docs.n8n.io/) · OpenAI API: [platform.openai.com](https://platform.openai.com/docs/api-reference) · Google Sheets API: [developers.google.com](https://developers.google.com/sheets/api) · VEO 3 API: [kie.ai](https://kie.ai/) · Bloatato API: [backend.blotato.com](https://backend.blotato.com/) · Instagram Graph API: [developers.facebook.com](https://developers.facebook.com/docs/instagram-api/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Automated AI Video Creation with Veo3 for Instagram Publishing".

2. **Add a Schedule Trigger node:**
   - Type: Schedule Trigger
   - Name: "Start Daily Two Times"
   - Configuration: Interval-based, every 12 hours
   - This is the entry point of the workflow.

3. **Add an OpenAI Chat Model node (for idea generation):**
   - Type: LangChain → Chat Model → OpenAI
   - Name: "OpenAI Chat Model"
   - Configuration: Model = `gpt-4.1-mini`, no built-in tools, no additional options
   - Credential: Create or select an OpenAI API credential

4. **Add a Think Tool node (for idea agent reasoning):**
   - Type: LangChain → Tool → Think
   - Name: "Tool (Make More Creative Thinking in Action)"
   - Configuration: Default description (records internal reasoning)

5. **Add a Structured Output Parser node (for idea schema):**
   - Type: LangChain → Output Parser → Structured
   - Name: "Parse AI Output"
   - Configuration: JSON schema example set to an array with objects containing: `Caption` (string), `Idea` (string), `Environment` (string), `Sound` (string), `Status` (string)

6. **Add an AI Agent node (for idea generation):**
   - Type: LangChain → Agent
   - Name: "Make a Creative Video Idea"
   - Version: 1.9
   - Configuration:
     - Prompt type: Define
     - User prompt: Full prompt text instructing generation of a BEFORE/AFTER transformation concept with style, color palette, camera, lighting descriptors; output as a single-line JSON array
     - System message: Detailed role and rules (one idea per request, BEFORE/AFTER theme, max 15 words for idea, caption with emoji + 12 hashtags, max 20 words for environment, max 15 words for sound, Status always "for production", single-line JSON array output format)
     - Has Output Parser: Enabled
   - Sub-node connections:
     - Connect "OpenAI Chat Model" → ai_languageModel
     - Connect "Parse AI Output" → ai_outputParser
     - Connect "Tool (Make More Creative Thinking in Action)" → ai_tool
   - Main connection: Connect "Start Daily Two Times" → this node (main input)

7. **Add a Google Sheets node (to save the idea):**
   - Type: Google Sheets
   - Name: "Save Idea in google sheet"
   - Version: 4.5
   - Configuration:
     - Operation: Append
     - Document: Your Google Sheet ID (replace with your own)
     - Sheet: Sheet1
     - Mapping mode: Define below
     - Matching columns: `id`
     - Column mappings: `idea` = `{{ $json.output[0].Idea }}`, `caption` = `{{ $json.output[0].Caption }}`, `production` = `{{ $json.output[0].Status }}`, `sound_prompt` = `{{ $json.output[0].Sound }}`, `environment_prompt` = `{{ $json.output[0].Environment }}`
   - Credential: Google Sheets OAuth2 API
   - Main connection: Connect "Make a Creative Video Idea" → this node

8. **Add a Set node (to prepare the professional prompt):**
   - Type: Set (now called Edit Fields)
   - Name: "Make Professional Prompt"
   - Version: 3.4
   - Configuration — three assignments:
     - `json_master` (string): Paste the full VEO 3 master JSON schema (containing description, style, camera, lighting, environment, elements, subject, motion, VFX, audio, ending, text, format, keywords)
     - `model` (string): `"veo3_fast"`
     - `aspectRatio` (string): `"9:16"`
   - Main connection: Connect "Save Idea in google sheet" → this node

9. **Add a second OpenAI Chat Model node (for scripting agent):**
   - Type: LangChain → Chat Model → OpenAI
   - Name: "OpenAI Chat Model1"
   - Configuration: Model = `gpt-4.1-mini`, no built-in tools
   - Credential: Same OpenAI API credential

10. **Add a second Think Tool node (for scripting agent):**
    - Type: LangChain → Tool → Think
    - Name: "Think"
    - Configuration: Default description (records internal reasoning)

11. **Add a second Structured Output Parser node (for scripting schema):**
    - Type: LangChain → Output Parser → Structured
    - Name: "Structured Output Parser"
    - Version: 1.3
    - Configuration: JSON schema example = `{ "title": "string", "final_prompt": "string" }`

12. **Add a second AI Agent node (for video script creation):**
    - Type: LangChain → Agent
    - Name: "AI Agent (Create Video Script)"
    - Version: 2
    - Configuration:
      - Prompt type: Define
      - User prompt: Instructions to create a BEFORE/AFTER transformation video prompt using the idea, environment, and sound from the Save Idea node. References: `{{ $('Save Idea in google sheet').item.json.idea }}`, `{{ $('Save Idea in google sheet').item.json.environment_prompt }}`, `{{ $('Save Idea in google sheet').item.json.sound_prompt }}`. Rules: cinematic, 9:16 vertical, include all scene elements, output one JSON with `title` and `final_prompt`, use Think tool.
      - System message: Full AGN-pattern system prompt — Role: Creative Director, output valid JSON, title max 15 words, final_prompt is a single-line stringified JSON matching `{{ $json.json_master }}`, no markdown, escaped inner quotes
      - Has Output Parser: Enabled
    - Sub-node connections:
      - Connect "OpenAI Chat Model1" → ai_languageModel
      - Connect "Structured Output Parser" → ai_outputParser
      - Connect "Think" → ai_tool
    - Main connection: Connect "Make Professional Prompt" → this node (main input)

13. **Add a Code node (to format the prompt):**
    - Type: Code
    - Name: "Code (Format Prompt)"
    - Version: 2
    - Configuration: JavaScript code:
      ```javascript
      const structuredPrompt = $input.first().json.output.final_prompt;
      return {
        json: {
          prompt: JSON.stringify(structuredPrompt),
          model: "veo3_fast",
          aspectRatio: "9:16"
        }
      };
      ```
    - Main connection: Connect "AI Agent (Create Video Script)" → this node

14. **Add an HTTP Request node (to create video with VEO 3):**
    - Type: HTTP Request
    - Name: "HTTP (Create Video with VEO3)"
    - Version: 4.2
    - Configuration:
      - Method: POST
      - URL: `https://api.kie.ai/api/v1/veo/generate`
      - Authentication: Generic Credential Type → Bearer Auth
      - Headers: `Authorization` = `Bearer YOUR_TOKEN_HERE` (replace with actual token)
      - Body: JSON, specified as:
        ```
        { "prompt": {{ $json.prompt }}, "model": "{{ $('Make Professional Prompt').item.json.model }}", "aspectRatio": "{{ $('Make Professional Prompt').item.json.aspectRatio }}" }
        ```
    - Credential: Create an HTTP Bearer Auth credential with your VEO 3 API token
    - Main connection: Connect "Code (Format Prompt)" → this node

15. **Add a Wait node (3-minute initial delay):**
    - Type: Wait
    - Name: "Wait 3m for Rendering video"
    - Version: 1.1
    - Configuration: Amount = 3 (minutes)
    - Main connection: Connect "HTTP (Create Video with VEO3)" → this node

16. **Add an HTTP Request node (to check video status/download):**
    - Type: HTTP Request
    - Name: "HTTP (Download Video from VEO3)"
    - Version: 4.2
    - Configuration:
      - Method: GET (default)
      - URL: `https://api.kie.ai/api/v1/veo/record-info`
      - Authentication: Generic Credential Type → Header Auth
      - Query parameters: `taskId` = `{{ $('HTTP (Create Video with VEO3)').item.json.data.taskId }}`
    - Credential: Create an HTTP Header Auth credential named "Veo3" with the appropriate API key header
    - Main connection: Connect "Wait 3m for Rendering video" → this node

17. **Add an IF node (to check video status):**
    - Type: IF
    - Name: "If (Check Video Status)"
    - Version: 2.2
    - Configuration:
      - Condition: `$json.data.status` equals (string, strict type, case-sensitive) `SUCCESS`
    - Main connections:
      - Input: Connect "HTTP (Download Video from VEO3)" → this node
      - TRUE output → HTTP (Upload Bloatato Media) (step 20)
      - FALSE output → Wait 30s node (step 19)

18. **Add a Wait node (30-second retry delay):**
    - Type: Wait
    - Name: "Wait 30s if video still processing"
    - Version: 1.1
    - Configuration: Amount = 30 (seconds)
    - Main connections:
      - Input: Connect "If (Check Video Status)" FALSE branch → this node
      - Output: Connect this node → "If (Check Video Status)" (loop back)

19. **Add an HTTP Request node (to upload to Bloatato):**
    - Type: HTTP Request
    - Name: "HTTP (Upload Bloatato Media)"
    - Version: 4.2
    - Configuration:
      - Method: POST
      - URL: `https://backend.blotato.com/v2/media`
      - Headers: `Authorization` = `Bearer YOUR_TOKEN_HERE` (replace with Bloatato API token)
      - Body parameters: `url` = `{{ $json["data"]["videoUrl"] }}`
    - Credential: Configure a Bearer Auth credential for Bloatato
    - Main connections:
      - Input: Connect "If (Check Video Status)" TRUE branch → this node
      - Output: Connect this node → "HTTP (Publish to Instagram)"

20. **Add an HTTP Request node (to publish to Instagram):**
    - Type: HTTP Request
    - Name: "HTTP (Publish to Instagram)"
    - Version: 4.2
    - Configuration:
      - Method: POST
      - URL: `https://backend.blotato.com/v2/posts`
      - Authentication: Generic Credential Type → Header Auth
      - Body: JSON, specified as:
        ```
        {
          "post": {
            "target": { "targetType": "instagram" },
            "content": {
              "text": {{ $('Save Idea in google sheet').first().json.idea }},
              "platform": "instagram",
              "mediaUrls": ["{{ $json.url }}"]
            },
            "accountId": "add_your_instagram_ID"
          }
        }
        ```
      - Replace `"add_your_instagram_ID"` with your actual Bloatato/Instagram account ID
    - Credential: HTTP Header Auth (may reuse "Veo3" credential or create a dedicated one)
    - Main connections:
      - Input: Connect "HTTP (Upload Bloatato Media)" → this node
      - Output: Connect this node → "Update Status to Publish"

21. **Add a Google Sheets node (to update status):**
    - Type: Google Sheets
    - Name: "Update Status to Publish"
    - Version: 4.5
    - Configuration:
      - Operation: Append or Update
      - Document: Same Google Sheet ID as step 7
      - Sheet: Sheet1
      - Matching columns: `idea`
      - Column mappings: `idea` = `{{ $('Save Idea in google sheet').first().json.idea }}`, `production` = `"Publish"`
    - Credential: Same Google Sheets OAuth2 credential
    - Main connection: Connect "HTTP (Publish to Instagram)" → this node
    - Output: Connect this node to all three notification nodes (parallel)

22. **Add a Telegram node:**
    - Type: Telegram
    - Name: "Send Telegram Notification"
    - Version: 1.2
    - Configuration:
      - Chat ID: Your Telegram chat ID (replace placeholder)
      - Text: `New Shorts Video Publish:\nTime: {{ $now.format("DD-MMM-YYYY t") }}\nTitle: {{ $('Save Idea in google sheet').first().json.idea }}`
    - Credential: Create a Telegram API credential (bot token)
    - Input: Connect "Update Status to Publish" → this node

23. **Add a Rapiwa node:**
    - Type: Rapiwa
    - Name: "Rapiwa (Send WhatsApp Notification)"
    - Version: 1
    - Configuration:
      - Number: Your WhatsApp number (replace placeholder)
      - Message: Same format as Telegram notification
    - Credential: Create a Rapiwa API credential
    - Input: Connect "Update Status to Publish" → this node

24. **Add a Slack node:**
    - Type: Slack
    - Name: "Send Slack Notification"
    - Version: 2.4
    - Configuration:
      - Authentication: OAuth2
      - Select: User
      - User: Your Slack username (replace `@add_your_username`)
      - Text: Same format as other notifications
    - Credential: Create a Slack OAuth2 credential (app must be installed in workspace)
    - Input: Connect "Update Status to Publish" → this node

25. **Add Sticky Notes for visual grouping (optional):**
    - "Overview" sticky note covering the entire canvas with the workflow description, steps, requirements, customization options, and useful links
    - "Idea Generation & Storage Group" covering nodes in Block 1
    - "Professional Scripting Group" covering nodes in Block 2
    - "Video Generation & Retrieval Group" covering nodes in Block 3
    - "Publishing Group" covering nodes in Block 4
    - "Notification Group" covering nodes in Block 5

26. **Activate the workflow** once all credentials and placeholders are configured.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|-------------|-----------------|
| n8n Documentation | [https://docs.n8n.io/](https://docs.n8n.io/) |
| OpenAI API Reference | [https://platform.openai.com/docs/api-reference](https://platform.openai.com/docs/api-reference) |
| Google Sheets API | [https://developers.google.com/sheets/api](https://developers.google.com/sheets/api) |
| VEO 3 API (kie.ai) | [https://kie.ai/](https://kie.ai/) |
| Bloatato API (media hosting & Instagram publishing) | [https://backend.blotato.com/](https://backend.blotato.com/) |
| Instagram Graph API | [https://developers.facebook.com/docs/instagram-api/](https://developers.facebook.com/docs/instagram-api/) |
| The workflow uses two separate GPT-4.1-mini instances: one for idea generation and one for video script creation. Both share the same OpenAI credential but operate independently. | Credential reuse note |
| The polling loop (If → Wait 30s → If) has no maximum retry count. If VEO 3 returns a `FAILED` or `ERROR` status, the loop will never exit. Consider adding a max-retry counter or a timeout gate. | Architecture limitation |
| Several nodes contain placeholder values that must be replaced before activation: Bearer tokens in HTTP headers ("YOUR_TOKEN_HERE"), Instagram account ID ("add_your_instagram_ID"), Telegram chat ID ("add_your_ID"), WhatsApp number ("add_your_whatsapp_number"), and Slack username ("@add_your_username"). | Deployment prerequisite |
| The `HTTP (Upload Bloatato Media)` node uses a hardcoded Bearer token in the header parameter rather than a proper n8n credential. For security, migrate this to an HTTP Bearer Auth credential. | Security best practice |
| The `HTTP (Publish to Instagram)` node uses the `idea` field as the Instagram caption text, not the `caption` field that was specifically generated with emojis and hashtags. This may be unintentional — consider switching to `$('Save Idea in google sheet').first().json.caption` for the viral-ready caption. | Potential design issue |
| The Code (Format Prompt) node calls `JSON.stringify(structuredPrompt)` on the `final_prompt` value. If the structured output parser already returns `final_prompt` as a string (not an object), this will double-stringify it. Verify the actual output type at runtime. | Runtime verification needed |
| The Google Sheets "Update Status to Publish" node uses `idea` as the matching column. If multiple rows share the same idea text, all will be updated. Ensure ideas are unique or use a unique row identifier. | Data integrity note |
| The Wait nodes require the n8n instance to support webhook-based execution resumption (the instance must be reachable at its webhook URL). Self-hosted instances behind firewalls or NAT may need tunneling. | Infrastructure requirement |
| Workflow version uses execution order "v1". Ensure compatibility with your n8n version. | Version note |
| The `httpHeaderAuth` credential named "Veo3" is referenced by both the VEO 3 download endpoint and the Bloatato publish endpoint. These are different services and likely require different API keys. Verify credential assignments in production. | Credential assignment warning |