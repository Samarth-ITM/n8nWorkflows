Generate blog posts and social media posts with GPT‑4.1, Tavily and WordPress

https://n8nworkflows.xyz/workflows/generate-blog-posts-and-social-media-posts-with-gpt-4-1--tavily-and-wordpress-12858


# Generate blog posts and social media posts with GPT‑4.1, Tavily and WordPress

## 1. Workflow Overview

**Purpose:**  
This workflow generates (1) a blog post and (2) an on-brand image using LLMs + optional web research, then publishes the content to WordPress and multiple social channels (Telegram, Facebook, X, LinkedIn). It can also upload the generated image to Google Drive and log the publication details in Google Sheets.

**Primary use cases:**
- Automated content marketing (blog + social) from a topic/audience brief
- Research-assisted writing (via Tavily/SerpAPI tools)
- Multi-channel distribution + asset logging

### 1.1 Entry / Trigger Layer (multiple entry points)
The workflow can start from several triggers (manual/other workflow, social trigger reference, RSS, Google Sheets, Airtable, Postgres). All entry points converge into the “Blog Post Agent”.

### 1.2 Blog Post Generation (LLM + optional web research)
A LangChain Agent produces the long-form blog post. It is connected to an OpenAI chat model and has SerpAPI available as a tool (Tavily is present but not connected).

### 1.3 Image Prompt Generation (LLM)
A second LangChain Agent converts the blog output into an image title + prompt, using an OpenRouter chat model.

### 1.4 Image Generation + File Conversion
An HTTP request calls OpenAI Images (`gpt-image-1`) to generate a base64 image, then converts it into binary for downstream publishing/upload.

### 1.5 Publishing + Distribution + Logging
The workflow “splits out” the generated content (as configured), then fans out to:
- WordPress post creation
- Telegram photo send
- Facebook post
- X post
- LinkedIn post
- Google Drive upload, then Google Sheets logging

---

## 2. Block-by-Block Analysis

### Block A — Triggers / Inputs

**Overview:**  
Provides multiple ways to start the same content generation pipeline. Most triggers connect directly to the Blog Post Agent; one trigger (“When Executed by Another Workflow”) supplies explicit inputs (blogTopic, targetAudience, chatID).

**Nodes involved:**
- When Executed by Another Workflow
- Facebook Reference
- RSS Feed Trigger
- Google Sheets Trigger
- Airtable Trigger
- Postgres Trigger

#### Node: **When Executed by Another Workflow**
- **Type / role:** Execute Workflow Trigger (sub-workflow entry point)
- **Configuration (interpreted):**
  - Expects workflow inputs: `blogTopic`, `targetAudience`, `chatID`
- **Key variables:**
  - Used later in Telegram (`chatID`) and logging (`blogTopic`, `targetAudience`)
- **Connections:**
  - **Output →** Blog Post Agent
- **Failure/edge cases:**
  - If parent workflow does not pass `chatID`, Telegram send will fail (missing chat id).
  - If `blogTopic`/`targetAudience` empty, the blog agent prompt quality may degrade.

#### Node: **Facebook Reference**
- **Type / role:** Facebook Trigger (webhook-based)
- **Configuration:** Default options (not detailed); used as an alternative start trigger.
- **Connections:** **Output →** Blog Post Agent
- **Failure/edge cases:** Facebook app permissions / webhook validation / expired tokens.

#### Node: **RSS Feed Trigger**
- **Type / role:** RSS polling trigger
- **Configuration:** Polls every minute.
- **Connections:** **Output →** Blog Post Agent
- **Failure/edge cases:** Feed downtime, malformed RSS items, rate limits.

#### Node: **Google Sheets Trigger**
- **Type / role:** Google Sheets polling trigger (rowAdded)
- **Configuration:** Polls every minute; **sheetName/documentId are blank** (must be configured).
- **Connections:** **Output →** Blog Post Agent
- **Failure/edge cases:** Not functional until document + sheet are selected; Google auth scopes.

#### Node: **Airtable Trigger**
- **Type / role:** Airtable polling trigger
- **Configuration:** baseId/tableId blank (must be configured).
- **Connections:** **Output →** Blog Post Agent
- **Failure/edge cases:** Not functional until configured; API limits; field mapping changes.

#### Node: **Postgres Trigger**
- **Type / role:** Postgres trigger (DB event-driven or polling depending on n8n setup)
- **Configuration:** Schema `public`; tableName blank (must be configured).
- **Connections:** **Output →** Blog Post Agent
- **Failure/edge cases:** DB permissions, connectivity, replication slot/trigger support, missing table selection.

---

### Block B — Research Tools (available to agent)

**Overview:**  
Provides optional external search capability for the LLM agent(s). SerpAPI is actually connected as a tool to the Blog Post Agent; Tavily is defined but unused in current connections.

**Nodes involved:**
- SerpAPI
- Tavily

#### Node: **SerpAPI**
- **Type / role:** LangChain tool (SerpAPI search)
- **Configuration:** Default options; requires SerpAPI credentials in n8n.
- **Connections:**
  - **ai_tool →** Blog Post Agent
- **Failure/edge cases:** Missing/invalid SerpAPI key; quota limits; result variability.

#### Node: **Tavily**
- **Type / role:** LangChain tool implemented via HTTP Request
- **Configuration (interpreted):**
  - POST `https://api.tavily.com/search`
  - Body includes placeholders: `{searchTerm}` (tool input)
  - Search parameters: advanced depth, 1 result, 7 days window, include_answer=true
  - Auth via HTTP Header credential (generic)
- **Connections:** **None** (currently not attached to any agent)
- **Failure/edge cases:**
  - Not usable unless connected to an agent as an `ai_tool`.
  - Auth header must match Tavily requirements (typically `Authorization: Bearer ...`).

---

### Block C — Blog Post Generation (Agent + LLM)

**Overview:**  
Generates the full blog post text. The agent is powered by the OpenAI Chat Model node and can call SerpAPI for research.

**Nodes involved:**
- Blog Post Agent
- OpenAI Chat Model
- (Tool dependency) SerpAPI

#### Node: **Blog Post Agent**
- **Type / role:** LangChain Agent (text generation/orchestration)
- **Configuration (interpreted):**
  - Uses connected chat model (OpenAI Chat Model via `ai_languageModel`)
  - Has SerpAPI available as a tool
  - Prompt/system instructions are not visible in provided JSON (likely stored in node UI defaults or omitted)
- **Inputs:**
  - From any trigger node
- **Outputs:**
  - **Main output →** Image Prompt Agent
  - Output content later referenced in logging: `$('Blog Post Agent').item.json.output`
- **Failure/edge cases:**
  - If the agent output is not a string or is missing `output`, downstream nodes referencing it may fail.
  - Research tool may return unexpected formats; agent prompt should instruct how to cite/use results.

#### Node: **OpenAI Chat Model**
- **Type / role:** LangChain Chat Model provider (OpenAI)
- **Configuration (interpreted):**
  - Model: `gpt-4.1-mini`
  - Credentials: OpenAI API credential (“n8n free OpenAI API credits”)
  - “builtInTools” present but empty (no built-in tools configured here)
- **Connections:**
  - **ai_languageModel →** Blog Post Agent
- **Failure/edge cases:**
  - Missing/expired OpenAI key; org/project restrictions; rate limits.
  - Model availability changes (ensure `gpt-4.1-mini` is enabled for the key).

---

### Block D — Image Prompt Generation (Agent + LLM)

**Overview:**  
Transforms the blog post into an image concept: typically a `title` and a detailed `prompt` suitable for an image model.

**Nodes involved:**
- Image Prompt Agent
- OpenRouter Chat Model

#### Node: **Image Prompt Agent**
- **Type / role:** LangChain Agent (prompt engineering step)
- **Configuration:** Options empty; relies on connected LLM.
- **Input:** From Blog Post Agent
- **Output:** To Generate Image
  - Downstream references:
    - `$('Image Prompt Agent').item.json.output.title` (file name, sheet title)
    - `$json.output.prompt` used in image generation request
- **Failure/edge cases:**
  - If agent returns output not shaped as `{ title, prompt }`, downstream expressions will break.
  - Quotation sanitization is applied later; if prompt is not a string, `.replace()` fails.

#### Node: **OpenRouter Chat Model**
- **Type / role:** LangChain Chat Model provider (OpenRouter)
- **Configuration:** Options empty (model not explicitly set here in JSON)
- **Connections:**
  - **ai_languageModel →** Image Prompt Agent
- **Failure/edge cases:**
  - Requires OpenRouter credentials (not shown in JSON).
  - If no model is selected in node UI, executions may fail.

---

### Block E — Image Generation (OpenAI Images) + Binary Conversion

**Overview:**  
Calls OpenAI Images API to generate an image from the agent-produced prompt, then converts the base64 response into a binary file for publishing/upload.

**Nodes involved:**
- Generate Image
- Convert to Binary

#### Node: **Generate Image**
- **Type / role:** HTTP Request (OpenAI Images generation endpoint)
- **Configuration (interpreted):**
  - POST `https://api.openai.com/v1/images/generations`
  - Auth: Generic credential via HTTP Header (must include `Authorization: Bearer <OPENAI_API_KEY>`)
  - Body params:
    - `model`: `gpt-image-1`
    - `prompt`: `={{ $json.output.prompt.replace(/\"/g, '') }}`
    - `size`: `1024x1024`
- **Input:** From Image Prompt Agent
- **Output:** To Convert to Binary
- **Failure/edge cases:**
  - If `$json.output.prompt` missing or not a string, expression fails.
  - API may return errors for policy/content restrictions, invalid key, or quota.
  - Response structure assumptions: expects `data[0].b64_json`.

#### Node: **Convert to Binary**
- **Type / role:** ConvertToFile (toBinary)
- **Configuration (interpreted):**
  - Operation: “toBinary”
  - Source property: `data[0].b64_json` (base64 image)
- **Input:** From Generate Image
- **Output:** To Split Out
- **Failure/edge cases:**
  - If OpenAI returns a different schema (or empty data array), conversion fails.
  - Large payloads could hit n8n memory limits depending on instance settings.

---

### Block F — Fan-out Distribution + Upload/Log

**Overview:**  
Splits the generated content and sends it to multiple destinations: WordPress, Telegram, Facebook, X, LinkedIn, Google Drive, then logs to Google Sheets.

**Nodes involved:**
- Split Out
- Create a post (WordPress)
- Telegram
- Facebook
- X
- LinkedIn
- Upload (Google Drive)
- Image Log (Google Sheets)

#### Node: **Split Out**
- **Type / role:** Split Out (itemization)
- **Configuration (interpreted):**
  - Field to split: `choices[0].message.content`
- **Input:** From Convert to Binary
- **Output:** To Create a post, Telegram, Facebook, X, LinkedIn, Upload (parallel fan-out)
- **Important note / edge case:**
  - This configuration suggests the incoming JSON contains `choices[0].message.content` (typical Chat Completions schema), but the upstream chain is **OpenAI Images → Convert to Binary**, which typically yields `data[0].b64_json`, not `choices[...]`.
  - If `choices[0].message.content` does not exist, Split Out will likely output nothing or error depending on n8n behavior/version.
  - In practice, you may want Split Out to operate on the blog post content (from Blog Post Agent) rather than the image response.

#### Node: **Create a post**
- **Type / role:** WordPress node (create post)
- **Configuration:** Additional fields empty (title/content/status likely must be configured in UI)
- **Input:** From Split Out
- **Output:** None shown
- **Failure/edge cases:**
  - Missing WordPress credentials, wrong site URL, insufficient role permissions.
  - If required fields (title/content) are not mapped, post creation fails.

#### Node: **Telegram**
- **Type / role:** Telegram sendPhoto
- **Configuration (interpreted):**
  - Operation: Send Photo
  - `chatId`: `={{ $('When Executed by Another Workflow').item.json.chatID }}`
  - `binaryData`: true (expects incoming binary image)
- **Input:** From Split Out
- **Output:** None
- **Failure/edge cases:**
  - If workflow started from triggers other than “When Executed…”, `chatID` may be undefined.
  - If binary property name not what Telegram expects (default `data` vs custom), sending can fail.

#### Node: **Facebook**
- **Type / role:** Facebook Graph API node (POST)
- **Configuration:** Options empty; endpoint/action must be configured in node UI.
- **Input:** From Split Out
- **Failure/edge cases:** Token permissions, API versioning, missing required fields.

#### Node: **X**
- **Type / role:** Twitter/X node
- **Configuration:** Additional fields empty; action must be configured.
- **Input:** From Split Out
- **Failure/edge cases:** Elevated API access needs, rate limits, content length constraints.

#### Node: **LinkedIn**
- **Type / role:** LinkedIn node (communityManagement auth)
- **Configuration (interpreted):**
  - Text: `={{ $json['choices[0].message.content'] }}`
  - Person: placeholder `[CONFIGURE_YOUR_LINKEDIN_PERSON_ID]`
- **Input:** From Split Out
- **Failure/edge cases:**
  - The expression references `choices[0].message.content` again—likely invalid given upstream.
  - Person ID must be replaced; auth scopes for posting required.

#### Node: **Upload**
- **Type / role:** Google Drive file upload
- **Configuration (interpreted):**
  - File name: `={{ $('Image Prompt Agent').item.json.output.title }}.png`
  - Folder: “AI Image Generation” (specific folder ID provided)
  - Drive: My Drive
- **Input:** From Split Out (expects binary)
- **Output →** Image Log
- **Failure/edge cases:**
  - Google credential/scopes; folder permission; filename invalid characters.
  - If input binary property is missing, upload fails.

#### Node: **Image Log**
- **Type / role:** Google Sheets append row (logging)
- **Configuration (interpreted):**
  - Spreadsheet: “Marketing Team Log”
  - Sheet: “Sheet1”
  - Appends mapped columns:
    - ID: `={{ $json.id }}`
    - Link: `={{ $json.webViewLink }}`
    - Post: `={{ $('Blog Post Agent').item.json.output }}`
    - Type: `"Post"`
    - Title: `={{ $('Image Prompt Agent').item.json.output.title }}`
    - Request: `={{ $('When Executed by Another Workflow').item.json.blogTopic }} for {{ $('When Executed by Another Workflow').item.json.targetAudience }}`
- **Input:** From Upload (uses Drive upload response fields like `id`, `webViewLink`)
- **Failure/edge cases:**
  - If workflow is not started via “When Executed…”, Request field may evaluate to `undefined`.
  - Sheet schema must match columns; Google API limits.

---

### Block G — “Chat Models” (available but not wired)

**Overview:**  
These nodes provide alternative LLM providers, but in this workflow JSON they are not connected (no outgoing connections to agents), except OpenAI and OpenRouter.

**Nodes involved (all unused in connections):**
- Mistral Cloud Chat Model (pixtral-large-latest)
- Google Gemini Chat Model
- DeepSeek Chat Model
- Azure OpenAI Chat Model
- Ollama Chat Model
- xAI Grok Chat Model
- AWS Bedrock Chat Model
- Groq Chat Model
- Anthropic Chat Model (Claude 3.7 Sonnet)
- Hugging Face Inference Model

**Common edge cases:** missing credentials, region constraints, model name changes, provider rate limits.  
**Practical note:** To use one, connect its `ai_languageModel` output to the Agent node you want to power.

---

### Block H — Alternative Image Providers (examples / unused)

**Overview:**  
These are sample HTTP nodes for other image generation providers. They are not connected to the main execution path.

**Nodes involved (unused):**
- Clipdrop API
- APITemplate.io
- Ideogram API
- Replicate API
- Imagen Google API
- HuggingFace API
- Runway Images
- Kling Images
- Leonardo Images
- Nano Banana

**Common edge cases:** placeholder tokens/URLs in node parameters, async prediction flows (Replicate/Runway often require polling), content-type mismatches, and returning URLs rather than raw image data (requiring additional “download binary” step).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When Executed by Another Workflow | Execute Workflow Trigger | Sub-workflow entry with inputs | — | Blog Post Agent | ## 1. Choose Your Trigger  \n\n- **Scheduled automatic generation:** Run the parent workflow on a schedule (for example, once per day at 9 AM) to publish new content regularly.\n- **Google Sheets trigger:** Generate content when a new row with a reference link or topic is added to your sheet.  \n- **Airtable trigger:** Start the workflow when records in a selected Airtable base/table change (for example, a new idea, brief, or status update), so your posts react instantly to updates in your Airtable content board.\n\n- **Postgres trigger:** Fire the workflow when new rows are inserted or existing rows are updated in a connected PostgreSQL table, letting you drive content generation from events in your app database or Supabase‑style back end.\n\n- **Manual start:** Hit **Execute workflow** whenever you want to spin up a batch of posts on demand. |
| Facebook Reference | Facebook Trigger | Alternative trigger | — | Blog Post Agent | ## 1. Choose Your Trigger  \n\n- **Scheduled automatic generation:** Run the parent workflow on a schedule (for example, once per day at 9 AM) to publish new content regularly.\n- **Google Sheets trigger:** Generate content when a new row with a reference link or topic is added to your sheet.  \n- **Airtable trigger:** Start the workflow when records in a selected Airtable base/table change (for example, a new idea, brief, or status update), so your posts react instantly to updates in your Airtable content board.\n\n- **Postgres trigger:** Fire the workflow when new rows are inserted or existing rows are updated in a connected PostgreSQL table, letting you drive content generation from events in your app database or Supabase‑style back end.\n\n- **Manual start:** Hit **Execute workflow** whenever you want to spin up a batch of posts on demand. |
| RSS Feed Trigger | RSS Feed Read Trigger | Alternative trigger (RSS) | — | Blog Post Agent | ## 1. Choose Your Trigger  \n\n- **Scheduled automatic generation:** Run the parent workflow on a schedule (for example, once per day at 9 AM) to publish new content regularly.\n- **Google Sheets trigger:** Generate content when a new row with a reference link or topic is added to your sheet.  \n- **Airtable trigger:** Start the workflow when records in a selected Airtable base/table change (for example, a new idea, brief, or status update), so your posts react instantly to updates in your Airtable content board.\n\n- **Postgres trigger:** Fire the workflow when new rows are inserted or existing rows are updated in a connected PostgreSQL table, letting you drive content generation from events in your app database or Supabase‑style back end.\n\n- **Manual start:** Hit **Execute workflow** whenever you want to spin up a batch of posts on demand. |
| Google Sheets Trigger | Google Sheets Trigger | Alternative trigger (new row) | — | Blog Post Agent | ## 1. Choose Your Trigger  \n\n- **Scheduled automatic generation:** Run the parent workflow on a schedule (for example, once per day at 9 AM) to publish new content regularly.\n- **Google Sheets trigger:** Generate content when a new row with a reference link or topic is added to your sheet.  \n- **Airtable trigger:** Start the workflow when records in a selected Airtable base/table change (for example, a new idea, brief, or status update), so your posts react instantly to updates in your Airtable content board.\n\n- **Postgres trigger:** Fire the workflow when new rows are inserted or existing rows are updated in a connected PostgreSQL table, letting you drive content generation from events in your app database or Supabase‑style back end.\n\n- **Manual start:** Hit **Execute workflow** whenever you want to spin up a batch of posts on demand. |
| Airtable Trigger | Airtable Trigger | Alternative trigger (Airtable change) | — | Blog Post Agent | ## 1. Choose Your Trigger  \n\n- **Scheduled automatic generation:** Run the parent workflow on a schedule (for example, once per day at 9 AM) to publish new content regularly.\n- **Google Sheets trigger:** Generate content when a new row with a reference link or topic is added to your sheet.  \n- **Airtable trigger:** Start the workflow when records in a selected Airtable base/table change (for example, a new idea, brief, or status update), so your posts react instantly to updates in your Airtable content board.\n\n- **Postgres trigger:** Fire the workflow when new rows are inserted or existing rows are updated in a connected PostgreSQL table, letting you drive content generation from events in your app database or Supabase‑style back end.\n\n- **Manual start:** Hit **Execute workflow** whenever you want to spin up a batch of posts on demand. |
| Postgres Trigger | Postgres Trigger | Alternative trigger (DB change) | — | Blog Post Agent | ## 1. Choose Your Trigger  \n\n- **Scheduled automatic generation:** Run the parent workflow on a schedule (for example, once per day at 9 AM) to publish new content regularly.\n- **Google Sheets trigger:** Generate content when a new row with a reference link or topic is added to your sheet.  \n- **Airtable trigger:** Start the workflow when records in a selected Airtable base/table change (for example, a new idea, brief, or status update), so your posts react instantly to updates in your Airtable content board.\n\n- **Postgres trigger:** Fire the workflow when new rows are inserted or existing rows are updated in a connected PostgreSQL table, letting you drive content generation from events in your app database or Supabase‑style back end.\n\n- **Manual start:** Hit **Execute workflow** whenever you want to spin up a batch of posts on demand. |
| Tavily | LangChain Tool (HTTP Request) | Web research tool (unused) | — | — | # Tools for internet  research |
| SerpAPI | LangChain Tool (SerpAPI) | Web research tool for Blog agent | — | Blog Post Agent (ai_tool) | # Tools for internet  research |
| Blog Post Agent | LangChain Agent | Generates blog post | All triggers | Image Prompt Agent | # Content Generation |
| OpenAI Chat Model | LangChain Chat Model (OpenAI) | LLM provider for Blog agent | — | Blog Post Agent (ai_languageModel) | # Chat Models |
| Image Prompt Agent | LangChain Agent | Generates image title + prompt | Blog Post Agent | Generate Image | # Image Prompt |
| OpenRouter Chat Model | LangChain Chat Model (OpenRouter) | LLM provider for Image Prompt agent | — | Image Prompt Agent (ai_languageModel) | # Chat Models |
| Generate Image | HTTP Request | Calls OpenAI Images API | Image Prompt Agent | Convert to Binary | # Generate Image with your favourite model |
| Convert to Binary | Convert to File | Base64 → binary image | Generate Image | Split Out | # Generate Image with your favourite model |
| Split Out | Split Out | Splits content field for fan-out | Convert to Binary | Create a post; Telegram; Facebook; X; LinkedIn; Upload | # Post Content to social media |
| Create a post | WordPress | Publish blog post | Split Out | — | # Post to Wordpress |
| Telegram | Telegram | Post image to Telegram | Split Out | — | # Post Content to social media |
| Facebook | Facebook Graph API | Post to Facebook | Split Out | — | # Post Content to social media |
| X | Twitter/X | Post to X | Split Out | — | # Post Content to social media |
| LinkedIn | LinkedIn | Post to LinkedIn | Split Out | — | # Post Content to social media |
| Upload | Google Drive | Upload image to Drive | Split Out | Image Log | # Upload to Drive & Sheets |
| Image Log | Google Sheets | Append log row | Upload | — | # Upload to Drive & Sheets |
| Mistral Cloud Chat Model | LangChain Chat Model (Mistral) | Alternative LLM (unused) | — | — | # Chat Models |
| Google Gemini Chat Model | LangChain Chat Model (Gemini) | Alternative LLM (unused) | — | — | # Chat Models |
| DeepSeek Chat Model | LangChain Chat Model (DeepSeek) | Alternative LLM (unused) | — | — | # Chat Models |
| Azure OpenAI Chat Model | LangChain Chat Model (Azure OpenAI) | Alternative LLM (unused) | — | — | # Chat Models |
| Ollama Chat Model | LangChain Chat Model (Ollama) | Alternative local LLM (unused) | — | — | # Chat Models |
| xAI Grok Chat Model | LangChain Chat Model (xAI) | Alternative LLM (unused) | — | — | # Chat Models |
| AWS Bedrock Chat Model | LangChain Chat Model (Bedrock) | Alternative LLM (unused) | — | — | # Chat Models |
| Groq Chat Model | LangChain Chat Model (Groq) | Alternative LLM (unused) | — | — | # Chat Models |
| Anthropic Chat Model | LangChain Chat Model (Anthropic) | Alternative LLM (unused) | — | — | # Chat Models |
| Hugging Face Inference Model | LangChain HF Inference | Alternative LLM (unused) | — | — | # Chat Models |
| Clipdrop API | HTTP Request | Alt image generation (unused) | — | — | # Generate Image with your favourite model |
| APITemplate.io | APITemplate.io | Alt image/template provider (unused) | — | — | # Generate Image with your favourite model |
| Ideogram API | HTTP Request | Alt image generation (unused) | — | — | # Generate Image with your favourite model |
| Replicate API | HTTP Request | Alt image generation (unused) | — | — | # Generate Image with your favourite model |
| Imagen Google API | HTTP Request | Alt image generation (unused) | — | — | # Generate Image with your favourite model |
| HuggingFace API | HTTP Request | Example HF endpoint call (unused) | — | — | # Generate Image with your favourite model |
| Runway Images | HTTP Request | Alt image generation (unused) | — | — | # Generate Image with your favourite model |
| Kling Images | HTTP Request | Alt image generation (unused) | — | — | # Generate Image with your favourite model |
| Leonardo Images | HTTP Request | Alt image generation (unused) | — | — | # Generate Image with your favourite model |
| Nano Banana | HTTP Request | Alt provider (unused) | — | — | # Generate Image with your favourite model |
| Sticky Note | Sticky Note | Comment | — | — |  |
| Sticky Note1 | Sticky Note | Comment | — | — |  |
| Sticky Note2 | Sticky Note | Comment | — | — |  |
| Sticky Note3 | Sticky Note | Comment | — | — |  |
| Sticky Note4 | Sticky Note | Comment | — | — |  |
| Sticky Note5 (disabled) | Sticky Note | Comment | — | — |  |
| Sticky Note6 | Sticky Note | Comment | — | — |  |
| Sticky Note7 | Sticky Note | Comment | — | — |  |
| Sticky Note8 | Sticky Note | Comment | — | — |  |
| Sticky Note9 | Sticky Note | Comment | — | — |  |
| Sticky Note11 | Sticky Note | Comment | — | — |  |
| Sticky Note12 | Sticky Note | Comment | — | — |  |
| Sticky Note13 | Sticky Note | Comment | — | — |  |
| Sticky Note14 | Sticky Note | Comment | — | — |  |
| Sticky Note15 | Sticky Note | Comment | — | — |  |
| Sticky Note16 | Sticky Note | Comment | — | — |  |
| Sticky Note17 | Sticky Note | Comment | — | — |  |

> Note: Sticky Note nodes are included in the table to satisfy “do not skip any nodes”. They don’t participate in execution logic.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   “Generate blog posts and social media posts with GPT‑4.1, Tavily and WordPress”.

2. **Add an entry trigger (sub-workflow style):**
   - Add node: **Execute Workflow Trigger**  
     Name: **When Executed by Another Workflow**
   - Define inputs:
     - `blogTopic` (string)
     - `targetAudience` (string)
     - `chatID` (string/number for Telegram)

3. **(Optional) Add alternative triggers** and connect each to the same next step:
   - **Facebook Trigger**, **RSS Feed Read Trigger**, **Google Sheets Trigger**, **Airtable Trigger**, **Postgres Trigger**
   - Configure each trigger’s credentials and required identifiers (Sheet doc + tab, Airtable base/table, Postgres table, RSS URL, Facebook app/webhook), then connect each trigger’s output to the Blog agent.

4. **Add the Blog generation agent:**
   - Add node: **LangChain Agent**  
     Name: **Blog Post Agent**
   - Configure its prompt in the node UI to produce your desired blog format (headings, CTA, length, audience adaptation).

5. **Add and connect the blog LLM provider:**
   - Add node: **OpenAI Chat Model** (LangChain)  
     Name: **OpenAI Chat Model**
   - Set model: `gpt-4.1-mini`
   - Attach OpenAI credentials.
   - Connect: **OpenAI Chat Model** → **Blog Post Agent** using the **ai_languageModel** connection.

6. **Add web research tool (SerpAPI) for the blog agent:**
   - Add node: **SerpAPI Tool**  
     Name: **SerpAPI**
   - Add SerpAPI credentials.
   - Connect: **SerpAPI** → **Blog Post Agent** using the **ai_tool** connection.
   - (Optional) Add **Tavily** tool:
     - Add node: **Tool HTTP Request** named **Tavily**
     - Configure POST to Tavily search with a `{searchTerm}` placeholder
     - Add header auth credential
     - Connect it to **Blog Post Agent** as **ai_tool** if you want the agent to use it.

7. **Add the image-prompt agent:**
   - Add node: **LangChain Agent**  
     Name: **Image Prompt Agent**
   - Configure prompt to output a structured object including at least:
     - `title`
     - `prompt` (detailed, visual style guidance)
   - Connect: **Blog Post Agent** → **Image Prompt Agent** (main).

8. **Add and connect the image-prompt LLM provider:**
   - Add node: **OpenRouter Chat Model**  
     Name: **OpenRouter Chat Model**
   - Select a model in the node UI (required in practice).
   - Add OpenRouter credentials.
   - Connect: **OpenRouter Chat Model** → **Image Prompt Agent** via **ai_languageModel**.

9. **Add image generation (OpenAI Images HTTP):**
   - Add node: **HTTP Request**  
     Name: **Generate Image**
   - Method: POST  
     URL: `https://api.openai.com/v1/images/generations`
   - Authentication: **Generic Credential → HTTP Header Auth**
     - Header should be `Authorization: Bearer <OPENAI_API_KEY>`
   - Body parameters:
     - `model` = `gpt-image-1`
     - `prompt` = expression: `{{ $json.output.prompt.replace(/\"/g, '') }}`
     - `size` = `1024x1024`
   - Connect: **Image Prompt Agent** → **Generate Image** (main).

10. **Convert OpenAI base64 image to binary:**
    - Add node: **Convert to File**
      Name: **Convert to Binary**
    - Operation: `toBinary`
    - Source property: `data[0].b64_json`
    - Connect: **Generate Image** → **Convert to Binary**.

11. **Add fan-out node (review needed):**
    - Add node: **Split Out**
      Name: **Split Out**
    - Field to split out: `choices[0].message.content`
    - Connect: **Convert to Binary** → **Split Out**
    - Important: If you want to distribute the *blog post text*, adjust this step to split the actual blog output (or remove Split Out and map fields directly). As provided, it may not match the upstream schema.

12. **Add WordPress publishing:**
    - Add node: **WordPress**  
      Name: **Create a post**
    - Configure credentials (WordPress API/application password or OAuth depending on your setup).
    - Map title/content from the correct incoming fields.
    - Connect: **Split Out** → **Create a post**.

13. **Add social posting nodes (configure only what you use):**
    - **Telegram** node:
      - Operation: sendPhoto
      - chatId: `{{ $('When Executed by Another Workflow').item.json.chatID }}`
      - Binary Data: enabled
    - **Facebook Graph API** node: configure endpoint/action + auth
    - **Twitter/X** node: configure action + auth
    - **LinkedIn** node:
      - Person: set your real person ID
      - Text mapping currently: `{{ $json['choices[0].message.content'] }}`
    - Connect: **Split Out** → each social node.

14. **Add Google Drive upload + Google Sheets logging (optional):**
    - Add node: **Google Drive** named **Upload**
      - File name: `{{ $('Image Prompt Agent').item.json.output.title }}.png`
      - Choose Drive + folder
      - Ensure it receives binary from upstream
    - Add node: **Google Sheets** named **Image Log**
      - Operation: Append
      - Select Spreadsheet + Sheet
      - Map columns (ID, Link, Post, etc.) similar to the workflow
    - Connect: **Split Out** → **Upload** → **Image Log**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Set up steps note: “Setup is intentionally quick and simple ✨” | Workflow guidance block |
| Trigger guidance (schedule, Sheets, Airtable, Postgres, manual) | Provided in Sticky Note: “## 1. Choose Your Trigger” |
| Connect chat models guidance (swap providers for cost/style) | Provided in Sticky Note: “## 2. Connect Chat Models” |
| Customize prompts guidance (brand voice + image style) | Provided in Sticky Note: “## 3. Customize Prompts” |
| Connect image generation model guidance (OpenAI Images + binary conversion; can swap providers) | Provided in Sticky Note: “## 4. Connect Image Generation Model” |
| Link social media & CMS guidance | Provided in Sticky Note: “## 5. Link Your Social Media & CMS” |
| Optional Google Sheets & Drive guidance | Provided in Sticky Note: “## 6. Optionally Connect Google Sheets & Drive” |
| Credit / profile link: “[made with ❤️ by N8ner 👈 click! Feel free to message me!](https://community.n8n.io/u/n8ner/badges)” | https://community.n8n.io/u/n8ner/badges |
| Image link embedded in sticky note | https://i.ibb.co/cSGCGn3H/Replace-image-1-2k-202601191958.jpg#full-width |

