Generate AI case studies from Feedspace testimonials with Google Gemini and publish to WordPress

https://n8nworkflows.xyz/workflows/generate-ai-case-studies-from-feedspace-testimonials-with-google-gemini-and-publish-to-wordpress-13012


# Generate AI case studies from Feedspace testimonials with Google Gemini and publish to WordPress

## 1. Workflow Overview

**Purpose:** Automatically receive **Feedspace** testimonial webhooks, transform **text testimonials** into **SEO-optimized HTML case studies** using a **Google Gemini-powered AI Agent with memory**, then **publish** the result as a **WordPress post**, and finally respond to the webhook caller.

**Target use cases:**
- Marketing/SEO teams using Feedspace who want continuous case study generation from incoming testimonials.
- Content automation pipelines publishing directly to WordPress with consistent structure but varied angles/tones.

**Logical blocks**
1.1 **Webhook Reception & Normalization**  
Receives the webhook, extracts/normalizes the testimonial fields needed downstream.

1.2 **Filtering (Text Testimonials Only)**  
Allows only specific Feedspace event types to proceed (text feedback); otherwise returns an error-style response payload.

1.3 **AI Case Study Generation (Gemini + Memory)**  
Uses an AI Agent connected to Google Gemini and a buffer memory to generate varied case studies (angle/tone/title formula rotation) in strict JSON.

1.4 **Output Parsing, Validation & Logging**  
Attempts to parse the AI output as JSON; falls back to manual extraction if malformed; logs strategy metadata for monitoring.

1.5 **WordPress Publishing & Webhook Response**  
Maps parsed content to WordPress fields, creates a post, then returns a success response to the original webhook.

---

## 2. Block-by-Block Analysis

### 1.1 Webhook Reception & Normalization

**Overview:** Receives a POST webhook from Feedspace and extracts the essential fields (review text, reviewer name, rating, and event type) into a simplified JSON payload.

**Nodes involved:**
- Feedspace Webhook
- Extract Testimonial Data

#### Node: **Feedspace Webhook**
- **Type / role:** `Webhook` (n8n-nodes-base.webhook) — entry point for Feedspace events.
- **Configuration (interpreted):**
  - **HTTP Method:** POST
  - **Path:** `fs-wp` (final endpoint is your n8n base URL + `/webhook/fs-wp` for test or `/webhook/<id>/fs-wp` depending on environment)
  - **Response mode:** `lastNode` (the workflow’s final response node determines the HTTP response)
  - **On error:** `continueRegularOutput` (execution continues even if this node errors; generally unusual for a webhook node but can prevent hard failures)
- **Inputs/outputs:** No inputs; outputs to **Extract Testimonial Data**.
- **Edge cases / failures:**
  - Incorrect webhook URL configured in Feedspace (404 / no trigger).
  - Unexpected payload shape (downstream expressions fail).
  - Large payloads or rate spikes (execution load).

#### Node: **Extract Testimonial Data**
- **Type / role:** `Set` (n8n-nodes-base.set) — normalizes inbound webhook payload into stable fields.
- **Configuration (interpreted):** Creates these fields:
  - `review` = `{{$json.body.data.response.comment}}`
  - `review_user` = `{{$json.body.data.response.Name}}`
  - `feedback_type` = `{{$json.body.type}}`
  - `rating` = `{{$json.body.data.response.value}}`
- **Inputs/outputs:** Input from **Feedspace Webhook**, output to **If**.
- **Key expressions/variables:** Uses `$json.body...` structure, so it assumes Feedspace sends a `body` object shaped exactly as referenced.
- **Edge cases / failures:**
  - Missing `data.response.comment`, `Name`, or `value` → empty fields or expression resolution issues.
  - `rating` stored as **string** (not number). If later logic needs numeric comparisons, conversion would be required.

---

### 1.2 Filtering (Text Testimonials Only)

**Overview:** Ensures only text-based testimonial events proceed to the AI generation block; other event types are routed to an “Error Response” webhook response.

**Nodes involved:**
- If
- Error Response

#### Node: **If**
- **Type / role:** `IF` (n8n-nodes-base.if) — conditional router.
- **Configuration (interpreted):**
  - Condition: `{{$json.feedback_type}}` **equals** `feed.text.received`
- **Inputs/outputs:**
  - Input from **Extract Testimonial Data**
  - **True** → **AI Agent - Generate Case Study & Title**
  - **False** → **Error Response**
- **Edge cases / failures:**
  - Feedspace event type differs (e.g., casing or alternative values) → routes to error branch unexpectedly.
  - If `feedback_type` is undefined → condition fails → error branch.

#### Node: **Error Response**
- **Type / role:** `Respond to Webhook` (n8n-nodes-base.respondToWebhook) — returns early response for disallowed events.
- **Configuration (interpreted):**
  - Respond with: JSON
  - Body: `{{$json}}` (echoes the current item)
- **Inputs/outputs:** Input from **If (false branch)**; terminates response path for those cases.
- **Edge cases / failures:**
  - Because webhook response mode is “lastNode”, ensure this node is indeed the final executed node on the false path (it is).
  - The payload echoed may include internal fields you do not want to expose; consider a sanitized error payload if needed.

---

### 1.3 AI Case Study Generation (Gemini + Memory)

**Overview:** Uses a LangChain AI Agent powered by **Google Gemini** and a **buffer window memory** to generate varied case studies in HTML and return them in strict JSON format.

**Nodes involved:**
- Simple Memory
- Google Gemini Chat Model
- AI Agent - Generate Case Study & Title

#### Node: **Simple Memory**
- **Type / role:** LangChain Memory Buffer Window ( `@n8n/n8n-nodes-langchain.memoryBufferWindow` ) — provides conversation memory across executions.
- **Configuration (interpreted):**
  - **Session ID type:** custom key
  - **Session key:** `feedspace_case_studies_global`
  - Effectively creates a shared memory thread for all executions using that key (global rotation strategy).
- **Connections:** Outputs via `ai_memory` to **AI Agent - Generate Case Study & Title**.
- **Edge cases / failures:**
  - Global session means **all testimonials share the same memory**; this is intended for “variety”, but can mix contexts across different brands/products if reused.
  - Memory growth and relevance: buffer window size is node-implementation dependent; older context may be truncated.

#### Node: **Google Gemini Chat Model**
- **Type / role:** LangChain Chat Model ( `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` ) — LLM provider.
- **Configuration (interpreted):**
  - Temperature: **0.7**
  - Max output tokens: **3000**
- **Connections:** Provides `ai_languageModel` input to the **AI Agent**.
- **Credential requirements:** Google Gemini / Google AI credentials configured in n8n for this node.
- **Edge cases / failures:**
  - Auth/permission issues, quota exhaustion, or model availability errors.
  - Output may exceed token limit → truncation → malformed JSON downstream.

#### Node: **AI Agent - Generate Case Study & Title**
- **Type / role:** LangChain Agent ( `@n8n/n8n-nodes-langchain.agent` ) — orchestrates prompt + memory + model to produce structured JSON.
- **Configuration (interpreted):**
  - Prompt injects values from **Extract Testimonial Data**:
    - Customer name, rating, review text
  - Strong constraints:
    - Must output **ONLY** a JSON object (no markdown, no backticks).
    - Must generate:
      - `title` (50–60 chars)
      - `content` (HTML with `<h2>`, `<p>`, `<blockquote><p>`, `<ul><li>`)
      - `meta` with angle/tone/title_formula/features_highlighted/reasoning
  - System message defines allowed **angles**, **tones**, and **title formulas**, and enforces variation using memory.
  - **On error:** `continueRegularOutput` (workflow continues even if the agent errors; downstream parse node must handle unexpected shapes).
- **Inputs/outputs:**
  - Inputs:
    - Main flow from **If (true branch)**
    - `ai_languageModel` from **Google Gemini Chat Model**
    - `ai_memory` from **Simple Memory**
  - Output: to **Parse AI Agent Output**
- **Edge cases / failures:**
  - The model may still return non-JSON or JSON-with-markdown fences → handled downstream.
  - Memory-based “variety” may be weak if memory is empty or truncated.
  - If Feedspace fields are empty, the agent may hallucinate missing details.

---

### 1.4 Output Parsing, Validation & Logging

**Overview:** Converts the AI agent output into a reliable `{title, content, meta}` object, with fallbacks for malformed JSON, and logs strategy metadata for audit/monitoring.

**Nodes involved:**
- Parse AI Agent Output
- Memory Context Logger

#### Node: **Parse AI Agent Output**
- **Type / role:** `Code` (n8n-nodes-base.code) — robust JSON parsing and fallback extraction.
- **Configuration (interpreted):**
  - Reads first input item as `agentOutput`.
  - Determines raw text from `agentOutput.output || agentOutput.text || JSON.stringify(agentOutput)`.
  - Removes markdown code fences (```json / ```).
  - Primary attempt: `JSON.parse(cleanedText)`.
  - Fallback (if parsing fails):
    - Regex extracts `"title": "..."` and `"content": "..."`.
    - Attempts to unescape sequences (`\\n`, `\\"`, etc.).
    - Attempts to parse `meta` if present; else sets default meta with parse error marker.
  - Outputs:
    - `title`, `content`, `meta`
    - `warning` if manual extraction was used
    - `debug_raw` first 200 chars of original raw output
- **Inputs/outputs:** Input from **AI Agent**, output to **Memory Context Logger**.
- **Edge cases / failures:**
  - Regex extraction can break if:
    - JSON uses single quotes, unescaped quotes inside content, or different ordering.
    - Content includes `"meta"`-like strings confusing the regex boundary.
  - Truncated JSON (token limits) may lead to partial extraction.

#### Node: **Memory Context Logger**
- **Type / role:** `Code` (n8n-nodes-base.code) — logs structured summary of generated strategy.
- **Configuration (interpreted):**
  - Builds a summary with:
    - Title
    - `meta.angle`, `meta.tone`, `meta.title_formula`, `meta.features_highlighted`, `meta.reasoning`
    - Timestamp
  - `console.log` output for observability
  - Pass-through: returns original caseStudy object unchanged
- **Inputs/outputs:** Input from **Parse AI Agent Output**, output to **Prepare WordPress Data**.
- **Edge cases / failures:**
  - If `meta` is missing or malformed, defaults are applied in the summary; publishing still proceeds.

---

### 1.5 WordPress Publishing & Webhook Response

**Overview:** Maps AI output into WordPress post fields, creates a WordPress post, then responds to the webhook with a success JSON payload.

**Nodes involved:**
- Prepare WordPress Data
- Create WordPress Post
- Respond to Webhook

#### Node: **Prepare WordPress Data**
- **Type / role:** `Set` (n8n-nodes-base.set) — maps parsed AI fields to WordPress-ready keys.
- **Configuration (interpreted):**
  - `Post Title` = `{{$json.title}}`
  - `Post content` = `{{$json.content}}`
  - `case_study_meta` = `{{JSON.stringify($json.meta)}}` (prepared but **not used** by the WordPress node as configured)
- **Inputs/outputs:** Input from **Memory Context Logger**, output to **Create WordPress Post**.
- **Edge cases / failures:**
  - If `content` contains invalid HTML for your theme/editor, WordPress may sanitize or render unexpectedly.
  - `case_study_meta` is unused unless you extend the WordPress node to store it (e.g., custom fields).

#### Node: **Create WordPress Post**
- **Type / role:** `WordPress` (n8n-nodes-base.wordpress) — creates a post.
- **Configuration (interpreted):**
  - Operation: create post (implied by node configuration)
  - Title: `{{$json['Post Title']}}`
  - Additional field: Content = `{{$json['Post content']}}`
  - Retries: enabled (`retryOnFail: true`), **maxTries=3**, **5s** wait between tries
  - **On error:** `continueRegularOutput` (so the workflow can still proceed to respond, but you may return “success” even if WP failed unless you add checks)
- **Inputs/outputs:** Input from **Prepare WordPress Data**, output to **Respond to Webhook**.
- **Credential requirements:** WordPress credentials (typically application password or OAuth/plugin-based depending on n8n setup).
- **Edge cases / failures:**
  - Auth errors (401/403), wrong site URL, REST API disabled, insufficient permissions.
  - Timeouts, rate limits, intermittent 5xx → mitigated by retry.
  - Posting succeeds but content formatting differs (block editor conversions).

#### Node: **Respond to Webhook**
- **Type / role:** `Respond to Webhook` (n8n-nodes-base.respondToWebhook) — final success response.
- **Configuration (interpreted):**
  - Respond with: JSON
  - Body: `{ "status": "success", "message": "Notification sent successfully", "timestamp": $now.toISO() }`
- **Inputs/outputs:** Input from **Create WordPress Post**; ends the webhook request.
- **Edge cases / failures:**
  - Response always returns “success” regardless of actual WP result unless you inspect WP node output and conditionally respond.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Feedspace Webhook | Webhook | Receives Feedspace POST events | — | Extract Testimonial Data | ## Webhook & Notification; Receives Feedspace webhook, extracts data and checks if text feedback |
| Extract Testimonial Data | Set | Normalizes webhook payload to stable fields | Feedspace Webhook | If | ## Webhook & Notification; Receives Feedspace webhook, extracts data and checks if text feedback |
| If | If | Filters only `feed.text.received` events | Extract Testimonial Data | AI Agent - Generate Case Study & Title; Error Response | ## Webhook & Notification; Receives Feedspace webhook, extracts data and checks if text feedback |
| Error Response | Respond to Webhook | Returns JSON for non-matching event types | If (false) | — | ## Webhook & Notification; Receives Feedspace webhook, extracts data and checks if text feedback |
| Simple Memory | Memory Buffer Window (LangChain) | Provides cross-run memory for variation | — | AI Agent - Generate Case Study & Title (ai_memory) | ## Generate case study with AI Agent |
| Google Gemini Chat Model | Google Gemini Chat Model (LangChain) | LLM provider for the agent | — | AI Agent - Generate Case Study & Title (ai_languageModel) | ## Generate case study with AI Agent |
| AI Agent - Generate Case Study & Title | LangChain Agent | Generates JSON: title/content/meta with variation strategy | If (true); Simple Memory; Google Gemini Chat Model | Parse AI Agent Output | ## Generate case study with AI Agent |
| Parse AI Agent Output | Code | Parses/repairs model output into usable JSON fields | AI Agent - Generate Case Study & Title | Memory Context Logger |  |
| Memory Context Logger | Code | Logs generated strategy metadata; passes through | Parse AI Agent Output | Prepare WordPress Data |  |
| Prepare WordPress Data | Set | Maps fields to WordPress title/content; serializes meta | Memory Context Logger | Create WordPress Post | ## Create Wordpress Post; Prepare the wordpress data and creates a post on WP. |
| Create WordPress Post | WordPress | Creates WP post; retries on fail | Prepare WordPress Data | Respond to Webhook | ## Create Wordpress Post; Prepare the wordpress data and creates a post on WP. |
| Respond to Webhook | Respond to Webhook | Returns success JSON to Feedspace | Create WordPress Post | — |  |
| Sticky Note | Sticky Note | Comment block | — | — | ## Webhook & Notification; Receives Feedspace webhook, extracts data and checks if text feedback |
| Sticky Note1 | Sticky Note | Comment block | — | — | ## Generate case study with AI Agent |
| Sticky Note2 | Sticky Note | Comment block | — | — | ## Create Wordpress Post; Prepare the wordpress data and creates a post on WP. |
| Sticky Note3 | Sticky Note | Comment block | — | — | ## Generate AI Case Studies from Feedspace Testimonials; **Who is this for?** Teams using Feedspace who want to automatically turn customer testimonials into SEO-optimized case studies and publish them on WordPress.; **Setup steps**; - Connect to Feedspace; - Activate the workflow and copy the Production webhook URL; - Go to Feedspace → Automations → Webhooks; - Paste the webhook URL and activate it; - See https://www.feedspace.io/help/automation/ for more information; - Add your AI API credentials to the AI model node; - Connect your WordPress account in the WordPress node; - Send testimonial data to the webhook in this format:; - Reviewer name; - Rating; - Text feedback; - Event or feedback type; - Activate the workflow; **How it works**; 1. Receives testimonial data through feedpsace webhook; 2. Extracts reviewer name, rating, feedback, and event type; 3. Filters for text-based testimonials; 4. Uses an AI agent to:; 5. Choose a unique case study angle and tone; 6. Generate structured HTML content; 7. Create an SEO-optimized title; 8. Parses and validates the AI output; 9. Publishes the generated case study to WordPress as a post |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow** in n8n  
   - Name it: *Generate AI-Powered Case Studies from Testimonials and Publish to WordPress* (or your preferred name).

2) **Add “Webhook” node** (Entry)  
   - Node type: **Webhook**
   - HTTP Method: **POST**
   - Path: `fs-wp`
   - Response mode: **Last Node**
   - Activate workflow later after configuration.
   - Copy the **Production webhook URL** for Feedspace.

3) **Add “Set” node: “Extract Testimonial Data”**  
   - Node type: **Set**
   - Add fields:
     - `review` (String) → expression: `{{$json.body.data.response.comment}}`
     - `review_user` (String) → `{{$json.body.data.response.Name}}`
     - `feedback_type` (String) → `{{$json.body.type}}`
     - `rating` (String) → `{{$json.body.data.response.value}}`
   - Connect: **Webhook → Extract Testimonial Data**

4) **Add “If” node**  
   - Node type: **If**
   - Condition (String):
     - Left: `{{$json.feedback_type}}`
     - Operation: **equals**
     - Right: `feed.text.received`
   - Connect: **Extract Testimonial Data → If**

5) **Add “Respond to Webhook” node: “Error Response”** (for the false path)  
   - Node type: **Respond to Webhook**
   - Respond with: **JSON**
   - Response body: `{{$json}}`
   - Connect: **If (false) → Error Response**

6) **Add memory node: “Simple Memory”**  
   - Node type: **Memory Buffer Window (LangChain)**
   - Session ID type: **Custom key**
   - Session key: `feedspace_case_studies_global`
   - No main connection required; it connects to the agent via the AI memory port.

7) **Add LLM node: “Google Gemini Chat Model”**  
   - Node type: **Google Gemini Chat Model (LangChain)**
   - Set:
     - Temperature: **0.7**
     - Max output tokens: **3000**
   - **Credentials:** configure your Google Gemini / Google AI credentials for this node in n8n.

8) **Add agent node: “AI Agent - Generate Case Study & Title”**  
   - Node type: **AI Agent (LangChain)**
   - Prompt type: “Define” (or equivalent “custom/system+user prompt” setup depending on UI)
   - **System message:** paste the provided system message defining angles/tones/title formulas and JSON-only response rules.
   - **User prompt/text:** paste the instruction block that injects:
     - `Customer: {{ $('Extract Testimonial Data').item.json.review_user }}`
     - `Rating: {{ $('Extract Testimonial Data').item.json.rating }}`
     - `Review: {{ $('Extract Testimonial Data').item.json.review }}`
   - Connect AI ports:
     - **Google Gemini Chat Model → Agent** (AI language model connection)
     - **Simple Memory → Agent** (AI memory connection)
   - Connect main flow:
     - **If (true) → AI Agent**

9) **Add “Code” node: “Parse AI Agent Output”**  
   - Node type: **Code**
   - Paste the JS that:
     - Cleans code fences
     - Parses JSON
     - Fallback regex extraction
     - Outputs `{title, content, meta, warning, debug_raw}`
   - Connect: **AI Agent → Parse AI Agent Output**

10) **Add “Code” node: “Memory Context Logger”**  
   - Node type: **Code**
   - Paste the JS logger that prints the strategy summary and passes data through.
   - Connect: **Parse AI Agent Output → Memory Context Logger**

11) **Add “Set” node: “Prepare WordPress Data”**  
   - Node type: **Set**
   - Fields:
     - `Post Title` (String) → `{{$json.title}}`
     - `Post content` (String) → `{{$json.content}}`
     - `case_study_meta` (String) → `{{JSON.stringify($json.meta)}}`
   - Connect: **Memory Context Logger → Prepare WordPress Data**

12) **Add “WordPress” node: “Create WordPress Post”**  
   - Node type: **WordPress**
   - Operation: **Create Post**
   - Title: `{{$json["Post Title"]}}`
   - Content (additional field): `{{$json["Post content"]}}`
   - Retries:
     - Enable retry on fail
     - Max tries: **3**
     - Wait: **5000 ms**
   - **Credentials:** configure WordPress credentials (site URL + authentication method supported by your n8n WP node).
   - Connect: **Prepare WordPress Data → Create WordPress Post**

13) **Add “Respond to Webhook” node: “Respond to Webhook”**  
   - Node type: **Respond to Webhook**
   - Respond with: **JSON**
   - Body:
     - `{"status":"success","message":"Notification sent successfully","timestamp": {{$now.toISO()}} }`
   - Connect: **Create WordPress Post → Respond to Webhook**

14) **Activate the workflow**  
   - In Feedspace: Automations → Webhooks → paste the **Production webhook URL** and enable.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Feedspace webhook setup reference | https://www.feedspace.io/help/automation/ |
| Sticky note project description: “Generate AI Case Studies from Feedspace Testimonials… setup steps… how it works…” | Applies to the whole workflow (see Sticky Note3 content in the table) |
| Important behavior: WordPress success response is always returned even if WP post creation fails (due to continue-on-error + unconditional success response) | Consider adding an IF check on WordPress node output and responding with failure when appropriate |
| Memory is global (`feedspace_case_studies_global`) so variation is enforced across all runs | If running multiple brands/sites, use a dynamic session key (e.g., per site or per Feedspace space) |

