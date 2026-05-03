Generate Instagram carousels from Telegram prompts using OpenAI and Kie AI

https://n8nworkflows.xyz/workflows/generate-instagram-carousels-from-telegram-prompts-using-openai-and-kie-ai-12510


# Generate Instagram carousels from Telegram prompts using OpenAI and Kie AI

## 1. Workflow Overview

This workflow listens for a Telegram message, verifies the sender, turns the message into a **5-slide Instagram carousel plan** (copy + per-slide image prompts) using **OpenAI**, generates the **carousel images** via **Kie AI**, then sends back **the images + an Instagram caption** to a target Telegram chat.

### 1.1 Input Reception & Access Control
Receives Telegram messages and blocks unauthorized users by checking the chat ID.

### 1.2 Content & Prompt Generation (OpenAI)
Applies brand style defaults, then uses an AI Agent (OpenAI chat model + structured output parser) to produce **exactly 5 image prompts** (with overlay text instructions). In parallel, a second OpenAI node produces a short SEO-friendly Instagram caption.

### 1.3 Visual Creation (Kie AI)
Splits the 5 prompts, submits them to Kie AI as a generation task, waits/polls until completion, then retrieves result URLs.

### 1.4 Delivery Back to Telegram
Downloads an image and sends it as a Telegram photo, and sends the generated caption as a Telegram text message.

---

## 2. Block-by-Block Analysis

### Block 1 — Receive & Verify Input
**Overview:** Triggers on Telegram updates and allows execution only if the sender matches a predefined chat ID. Prevents random users from consuming OpenAI/Kie quota.

**Nodes involved:**  
- Start on Telegram Message  
- Verify User ID  

#### Node: Start on Telegram Message
- **Type / role:** `telegramTrigger` — entry point, receives messages.
- **Key configuration:**
  - Updates: `channel_post`, `message`
- **Credentials:** Telegram API credential (`AgentWillyBot`)
- **Outputs:** To **Verify User ID**
- **Failure/edge cases:**
  - Bot not added to channel / insufficient permissions (no updates received)
  - Invalid Telegram token / revoked credential
  - Messages without `message.text` (photos/stickers) will later break “Set Brand Style”

#### Node: Verify User ID
- **Type / role:** `if` — simple access control gate.
- **Key configuration:**
  - Condition: `{{$json.message.chat.id}}` **equals** `12345667`
- **Outputs:**
  - **True branch** → Set Brand Style
  - **False branch** → no connection (execution ends silently)
- **Failure/edge cases:**
  - For `channel_post` updates, the structure may differ; `message.chat.id` may be absent (expression evaluates to null → fails condition)
  - Must replace placeholder ID with your real chat ID, otherwise nothing passes

---

### Block 2 — Generate Content & Prompts
**Overview:** Enriches input with brand style + topic, generates structured 5-slide prompts with an AI Agent, then generates an Instagram caption from the resulting prompts.

**Nodes involved:**  
- Set Brand Style  
- OpenAI Chat Model  
- Structured Output Parser  
- Generate Carousel Content  
- Generate Caption  
- Send Caption to Telegram  

#### Node: Set Brand Style
- **Type / role:** `set` — defines variables used downstream.
- **Key configuration:**
  - `style` (static string): “Dark Mode, Electric Teal & White accents, … aspect ratio 4:5…”
  - `topic` (expression): `{{$json.message.text}}`
- **Outputs:** To Generate Carousel Content
- **Failure/edge cases:**
  - If incoming update has no `message.text`, `topic` becomes empty/undefined; the agent prompt becomes low quality or fails.

#### Node: OpenAI Chat Model
- **Type / role:** LangChain chat model connector (`lmChatOpenAi`) used by the Agent node.
- **Key configuration:**
  - Model: `gpt-4.1`
  - Responses API disabled
- **Credentials:** OpenAI (`Sumopod AI`)
- **Connections:**
  - Outputs via `ai_languageModel` → Generate Carousel Content
- **Failure/edge cases:**
  - Model access not enabled on the OpenAI account
  - Rate limiting / quota exceeded
  - Network timeouts

#### Node: Structured Output Parser
- **Type / role:** LangChain structured parser — forces valid JSON output from the agent.
- **Key configuration:**
  - Schema example:
    - `{ "prompts": ["{{prompt1}}", ... "{{prompt5}}"] }`
  - This implies the agent must output a JSON object with a `prompts` array of 5 strings.
- **Connections:**
  - Outputs via `ai_outputParser` → Generate Carousel Content
- **Failure/edge cases:**
  - If the agent outputs non-JSON or wrong shape, parsing fails and the workflow stops at the agent node.

#### Node: Generate Carousel Content
- **Type / role:** LangChain Agent (`@n8n/n8n-nodes-langchain.agent`) — generates the 5-slide carousel plan as image prompts containing overlay copy.
- **Key configuration choices (interpreted):**
  - Prompt type: “define”
  - System message instructs:
    - 5-slide structure (Hook / Problem / Process / Result / CTA)
    - Must extrapolate and simulate details if needed
    - Must incorporate brand style
    - **Output must be valid JSON** with `prompts` in English, including background + overlay text
  - Has output parser enabled (Structured Output Parser)
  - Uses input vars:
    - `{{$json.topic}}`
    - `{{$json.style}}`
- **Inputs:**
  - Main: from Set Brand Style
  - AI language model: from OpenAI Chat Model
  - Output parser: from Structured Output Parser
- **Outputs:**
  - Main → Generate Caption
  - Main → Split Prompts
- **Failure/edge cases:**
  - Prompt injection from Telegram text (user can try to override instructions)
  - Output length too large → higher token usage/cost, possible truncation
  - If the model returns fewer/more than 5 prompts, downstream nodes may fail (caption expects indices 0–4)

#### Node: Generate Caption
- **Type / role:** OpenAI node (LangChain OpenAI) to produce the final Instagram caption.
- **Key configuration:**
  - Model: `gpt-4.1`
  - System message: caption rules (hook, SEO keywords, short paragraphs, CTA, hashtags)
  - User content includes Prompt 1..5 built from:
    - `{{$json.output.prompts[0]}}` … `[4]`
  - Output constraint: “Only provide the ready-to-post caption under 450 characters.”
- **Outputs:** Send Caption to Telegram
- **Failure/edge cases:**
  - If `output.prompts` missing or <5 items → expression errors
  - Model may exceed 450 chars (not technically enforced, only instructed)

#### Node: Send Caption to Telegram
- **Type / role:** `telegram` — sends the caption as a text message.
- **Key configuration:**
  - chatId: `<YOURCHATID>` (must be replaced)
  - text: `{{$json.output[0].content[0].text}}`
  - appendAttribution: false
- **Inputs:** from Generate Caption
- **Failure/edge cases:**
  - Wrong `chatId` → message not delivered
  - Telegram formatting: if caption contains unsupported characters/length, Telegram may reject (rare)

---

### Block 3 — Create Visuals (Kie AI)
**Overview:** Takes the 5 prompts, submits them to Kie AI to generate images, waits, polls status until success, then extracts URLs for download.

**Nodes involved:**  
- Split Prompts  
- Generate Carousel Images  
- Wait for Image Generation  
- Retrieve Carousel Images  
- Check Image Generation Status  
- Download Images  

#### Node: Split Prompts
- **Type / role:** `splitOut` — splits an array into multiple items.
- **Key configuration:**
  - Field to split out: `output.prompts`
- **Outputs:** Generate Carousel Images
- **Important behavior note:** After split, each item typically contains a *single prompt element*. However, the next node uses `JSON.stringify($json['output.prompts'])`, which assumes the full array still exists. Depending on n8n’s splitOut output structure, this can be inconsistent.
- **Failure/edge cases:**
  - If split produces items without `output.prompts` as an array, the Kie request body may be wrong.
  - If you actually want to generate all 5 images in one Kie task, you may not need Split Prompts at all.

#### Node: Generate Carousel Images
- **Type / role:** `httpRequest` — calls Kie AI “createTask”.
- **Key configuration:**
  - POST `https://api.kie.ai/api/v1/jobs/createTask`
  - Auth: HTTP Header Auth (`Kie AI Auth`)
  - JSON body includes:
    - model: `nano-banana-pro`
    - input.prompt: `{{ JSON.stringify($json['output.prompts']) }}`
    - aspect_ratio: `4:5`, resolution: `1K`, output_format: `png`
- **Outputs:** Wait for Image Generation
- **Failure/edge cases:**
  - Invalid API key / header name mismatch
  - Wrong payload format (Kie may expect an array vs stringified JSON; current implementation sends a JSON string)
  - Model name deprecated/invalid

#### Node: Wait for Image Generation
- **Type / role:** `wait` — delays before polling Kie status.
- **Key configuration:** default wait behavior (no explicit duration shown).
- **Outputs:** Retrieve Carousel Images
- **Failure/edge cases:**
  - If no wait duration is set, behavior depends on node defaults; may poll too quickly or not as intended.

#### Node: Retrieve Carousel Images
- **Type / role:** `httpRequest` — polls Kie AI task status.
- **Key configuration:**
  - GET `https://api.kie.ai/api/v1/jobs/recordInfo`
  - Query parameter: `taskId={{$json.data.taskId}}`
  - Auth: HTTP Header Auth (`Kie AI Auth`)
- **Outputs:** Check Image Generation Status
- **Failure/edge cases:**
  - If previous node didn’t return `data.taskId`, query is invalid
  - Kie API may return non-200 for “still processing”; should be handled

#### Node: Check Image Generation Status
- **Type / role:** `if` — checks whether the Kie task is complete.
- **Key configuration:**
  - Condition: `{{$json.data.state}}` equals `success`
- **Outputs:**
  - True → Download Images
  - False → Wait for Image Generation (poll loop)
- **Failure/edge cases:**
  - If Kie returns `failed` or `error`, this loop will continue forever (no terminal failure branch configured)
  - Missing `data.state` causes condition to fail → endless waiting loop

#### Node: Download Images
- **Type / role:** `httpRequest` — downloads generated image content.
- **Key configuration:**
  - URL: `{{ JSON.parse($json.data.resultJson).resultUrls[0] }}`
- **Outputs:** Send Images to Telegram
- **Failure/edge cases:**
  - Downloads only the first URL (`[0]`), even if 5 images exist
  - If `resultJson` isn’t valid JSON, `JSON.parse` throws
  - For Telegram binary upload, this node typically must be set to “Download”/binary response; not shown here, so it may return text instead of binary

---

### Block 4 — Deliver Results
**Overview:** Sends the downloaded image(s) and the caption to a Telegram chat.

**Nodes involved:**  
- Send Images to Telegram  
- Send Caption to Telegram  

#### Node: Send Images to Telegram
- **Type / role:** `telegram` — sends image to Telegram.
- **Key configuration:**
  - Operation: `sendPhoto`
  - chatId: `<YOURCHATID>` (must be replaced)
  - binaryData: true
- **Input:** from Download Images
- **Failure/edge cases:**
  - If Download Images didn’t produce proper binary property, sendPhoto fails (“binary data missing”)
  - Only one image is sent (first URL only)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Start on Telegram Message | telegramTrigger | Receive Telegram updates (entry point) | — | Verify User ID | ## Step 1: Receive & Verify Input |
| Verify User ID | if | Access control by chat.id | Start on Telegram Message | Set Brand Style | ## Step 1: Receive & Verify Input |
| Set Brand Style | set | Define `style` and `topic` variables | Verify User ID | Generate Carousel Content | ## Step 2: Generate Content & Prompts |
| OpenAI Chat Model | lmChatOpenAi | LLM provider for Agent | — | Generate Carousel Content (ai_languageModel) | ## Step 2: Generate Content & Prompts |
| Structured Output Parser | outputParserStructured | Enforce JSON schema output (`prompts` array) | — | Generate Carousel Content (ai_outputParser) | ## Step 2: Generate Content & Prompts |
| Generate Carousel Content | agent | Create 5-slide carousel prompts JSON | Set Brand Style (+ AI model/parser inputs) | Generate Caption; Split Prompts | ## Step 2: Generate Content & Prompts |
| Generate Caption | openAi | Write IG caption from prompts | Generate Carousel Content | Send Caption to Telegram | ## Step 2: Generate Content & Prompts |
| Send Caption to Telegram | telegram | Deliver caption text | Generate Caption | — | ## Step 4: Deliver Results |
| Split Prompts | splitOut | Split `output.prompts` array into items | Generate Carousel Content | Generate Carousel Images | ## Step 3: Create Visuals |
| Generate Carousel Images | httpRequest | Submit prompts to Kie AI createTask | Split Prompts | Wait for Image Generation | ## Step 3: Create Visuals |
| Wait for Image Generation | wait | Delay between polling attempts | Generate Carousel Images; Check Image Generation Status (false) | Retrieve Carousel Images | ## Step 3: Create Visuals |
| Retrieve Carousel Images | httpRequest | Poll Kie AI recordInfo by taskId | Wait for Image Generation | Check Image Generation Status | ## Step 3: Create Visuals |
| Check Image Generation Status | if | Loop until `state=success` | Retrieve Carousel Images | Download Images (true); Wait for Image Generation (false) | ## Step 3: Create Visuals |
| Download Images | httpRequest | Download first generated image URL | Check Image Generation Status (true) | Send Images to Telegram | ## Step 3: Create Visuals |
| Send Images to Telegram | telegram | Send downloaded image to Telegram | Download Images | — | ## Step 4: Deliver Results |
| Sticky Note | stickyNote | Comment block | — | — | ## How it works… (full note content preserved in section 5) |
| Sticky Note1 | stickyNote | Section label | — | — | ## Step 1: Receive & Verify Input |
| Sticky Note2 | stickyNote | Section label | — | — | ## Step 2: Generate Content & Prompts |
| Sticky Note3 | stickyNote | Section label | — | — | ## Step 3: Create Visuals |
| Sticky Note4 | stickyNote | Section label | — | — | ## Step 4: Deliver Results |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Telegram Trigger**
   - Add node: **Telegram Trigger**
   - Updates: enable `message` and `channel_post`
   - Connect Telegram credentials (create bot with BotFather, paste token into n8n Telegram credential)

2) **Add access control**
   - Add node: **IF**
   - Condition: Number → `{{$json.message.chat.id}}` equals **YOUR_CHAT_ID**
   - Connect: Telegram Trigger → IF (true branch continues)

3) **Set brand style + topic**
   - Add node: **Set**
   - Create fields:
     - `style` (string): your visual identity instructions (colors, typography, vibe, 4:5, etc.)
     - `topic` (string expression): `{{$json.message.text}}`
   - Connect: IF (true) → Set

4) **Create the AI Agent for carousel prompts**
   - Add node: **AI Agent (LangChain Agent)**
   - Prompt type: “Define”
   - System message: include the 5-slide structure and require JSON output with `prompts` (5 strings) in English containing background + overlay text.
   - Enable “Use Output Parser”
   - Connect: Set → AI Agent

5) **Attach OpenAI chat model to the agent**
   - Add node: **OpenAI Chat Model (LangChain)**
   - Model: `gpt-4.1`
   - Configure OpenAI credential (API key)
   - Connect its **AI Language Model** output to the Agent’s **ai_languageModel** input.

6) **Attach Structured Output Parser**
   - Add node: **Structured Output Parser**
   - Provide schema example:
     - `{ "prompts": ["{{prompt1}}","{{prompt2}}","{{prompt3}}","{{prompt4}}","{{prompt5}}"] }`
   - Connect its **ai_outputParser** output to the Agent’s **ai_outputParser** input.

7) **Generate caption with OpenAI**
   - Add node: **OpenAI (LangChain OpenAI)**
   - Model: `gpt-4.1`
   - System message: caption constraints (hook, SEO, line breaks, CTA, hashtags, <450 chars)
   - User content: include Prompt 1..5 using:
     - `{{$json.output.prompts[0]}}` … `{{$json.output.prompts[4]}}`
   - Connect: Agent → OpenAI (caption)

8) **Send caption back to Telegram**
   - Add node: **Telegram**
   - Operation: Send Message
   - chatId: **YOUR_CHAT_ID**
   - Text expression: `{{$json.output[0].content[0].text}}`
   - Connect: Caption node → Telegram (message)

9) **(Visual generation) Decide whether to split prompts**
   - If Kie expects **all prompts in one task**, *do not use Split Out* and pass the full prompts array directly.
   - If Kie expects **one prompt per task**, keep Split Out and adjust the Kie request to use the single prompt item.

10) **Split prompts (as in provided workflow)**
   - Add node: **Split Out**
   - Field: `output.prompts`
   - Connect: Agent → Split Out

11) **Submit Kie AI createTask**
   - Add node: **HTTP Request**
   - Method: POST
   - URL: `https://api.kie.ai/api/v1/jobs/createTask`
   - Authentication: **HTTP Header Auth** (create credential with Kie API key header/value as required by Kie)
   - Body (JSON): include model and input params (prompt(s), aspect_ratio 4:5, resolution 1K, png)
   - Connect: Split Out → HTTP Request

12) **Wait + poll loop**
   - Add node: **Wait**
   - Configure a sensible delay (e.g., 10–30s) to avoid hammering the API
   - Connect: createTask → Wait
   - Add node: **HTTP Request** (GET recordInfo)
     - URL: `https://api.kie.ai/api/v1/jobs/recordInfo`
     - Query: `taskId={{$json.data.taskId}}`
     - Auth: same header auth
   - Connect: Wait → recordInfo
   - Add node: **IF**
     - Condition: `{{$json.data.state}}` equals `success`
   - Connect: recordInfo → IF
   - Connect IF false → Wait (loop)

13) **Download image**
   - Add node: **HTTP Request**
   - URL expression: `{{ JSON.parse($json.data.resultJson).resultUrls[0] }}`
   - Configure response to download as **file/binary** (so Telegram sendPhoto can use it)
   - Connect: IF true → Download

14) **Send image to Telegram**
   - Add node: **Telegram**
   - Operation: Send Photo
   - chatId: **YOUR_CHAT_ID**
   - Binary data: enabled (select the binary property produced by the download node)
   - Connect: Download → Send Photo

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This workflow transforms a Telegram text prompt into a 5-slide Instagram carousel… Images are generated via Kie AI and sent back to you on Telegram, ready to post.” | Workflow purpose/behavior (Sticky Note) |
| “Setup steps: 1. Telegram… 2. OpenAI… 3. Kie AI… 4. Branding…” | Setup guidance (Sticky Note) |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.