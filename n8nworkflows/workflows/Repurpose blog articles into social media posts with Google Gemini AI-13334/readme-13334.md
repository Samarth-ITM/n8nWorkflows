Repurpose blog articles into social media posts with Google Gemini AI

https://n8nworkflows.xyz/workflows/repurpose-blog-articles-into-social-media-posts-with-google-gemini-ai-13334


# Repurpose blog articles into social media posts with Google Gemini AI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensif ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** This workflow accepts a blog article URL via a webhook, fetches the article content, uses **Google Gemini (LLM)** to generate repurposed social media posts, formats the output into a clean response payload, and returns it to the webhook caller.

**Typical use cases**
- Turn a newly published blog post into platform-ready snippets (LinkedIn/X/Threads, etc.).
- Automate content repurposing for editorial pipelines.
- Provide an internal “URL → social copy” microservice.

### 1.1 Input Reception & Initialization
Receives a URL and prepares configuration variables used downstream (e.g., platform list, tone, length constraints).

### 1.2 Article Retrieval & AI Generation
Downloads blog HTML/text and prompts Gemini to generate multiple social media drafts.

### 1.3 Output Formatting & Webhook Response
Normalizes the generated content into a structured JSON and returns it to the requester.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Input Reception & Initialization

**Overview:** Exposes an HTTP endpoint, receives a blog URL, and sets/normalizes configuration values for the rest of the workflow.

**Nodes involved**
- **Receive Blog URL** (Webhook)
- **Config** (Set)
- (Sticky notes in this area: **Workflow Overview**, **Step 1 - Trigger & Config**)

#### Node: “Receive Blog URL”
- **Type / role:** `Webhook` — entry point; starts the workflow when called.
- **Configuration (interpreted):**
  - The JSON provided doesn’t include specific webhook settings (method/path/response mode). In n8n defaults, you must still set:
    - **HTTP Method** (commonly `POST`)
    - **Path** (unique endpoint path)
    - **Response mode** (often “Using ‘Respond to Webhook’ node” when you have a Respond node downstream, which this workflow does)
- **Key expressions / variables:**
  - Expected to provide the blog URL in the incoming request (commonly in `body.url` or `query.url`). Exact mapping is not specified in JSON.
- **Connections:**
  - **Output →** Config
- **Edge cases / failures:**
  - Missing/invalid URL in request (downstream HTTP request fails).
  - If response mode is misconfigured (e.g., “On Received”), the Respond node may not be used and the caller may not receive generated content.
  - Webhook endpoint authentication not defined—if deployed publicly, consider adding auth or a secret token.

#### Node: “Config”
- **Type / role:** `Set` — defines constants and standardizes input fields for later nodes.
- **Configuration (interpreted):**
  - Parameters are empty in JSON, so the actual fields aren’t defined here. Common practice for this workflow pattern:
    - Set `blogUrl` from webhook input (e.g., `{{$json.body.url}}` or `{{$json.query.url}}`)
    - Set generation settings: `platforms`, `tone`, `language`, `numPosts`, `maxChars`, hashtags rules, etc.
- **Key expressions / variables:**
  - Not present in provided JSON; you must create them explicitly when rebuilding.
- **Connections:**
  - **Input ←** Receive Blog URL
  - **Output →** Fetch Blog Article
- **Edge cases / failures:**
  - Expression errors if referencing non-existent fields (e.g., `body.url` not present).
  - Overwriting incoming fields if “Keep Only Set” is enabled (not shown here). If enabled inadvertently, you might discard needed webhook metadata.

---

### Block 1.2 — Article Retrieval & AI Generation

**Overview:** Downloads the blog content and uses a LangChain LLM chain with Google Gemini as the language model to generate social post drafts.

**Nodes involved**
- **Fetch Blog Article** (HTTP Request)
- **Generate Social Posts** (LangChain Chain LLM)
- **Google Gemini Chat Model** (Gemini chat model connector)
- (Sticky note in this area: **Step 2 - Fetch & Generate**)

#### Node: “Fetch Blog Article”
- **Type / role:** `HTTP Request` — retrieves article content from the given URL.
- **Configuration (interpreted):**
  - Parameters empty in JSON; typically you must set:
    - **URL** = value from Config (e.g., `{{$json.blogUrl}}`)
    - **Method** = `GET`
    - **Response Format** = `String` (to pass HTML/text to the LLM) or `JSON` if your source provides it
    - Optional: follow redirects, set user-agent headers, timeout
- **Key expressions / variables:**
  - URL expression (not provided but implied)
- **Connections:**
  - **Input ←** Config
  - **Output →** Generate Social Posts
- **Edge cases / failures:**
  - 4xx/5xx responses, redirects, bot protection, paywalls.
  - Very large HTML can cause token/size issues later. You may need an HTML-to-text extraction step (not included).
  - Encoding issues or non-text content types.

#### Node: “Google Gemini Chat Model”
- **Type / role:** `lmChatGoogleGemini` — provides Gemini chat completion capability to LangChain nodes.
- **Configuration (interpreted):**
  - Parameters empty in JSON; you must configure:
    - **Credentials:** Google Gemini / Google AI Studio API key (or Google Cloud Vertex AI, depending on node credential type used by your n8n version)
    - **Model name:** e.g., `gemini-1.5-pro` / `gemini-1.5-flash` (exact selection not provided)
    - Optional: temperature, max output tokens, safety settings
- **Connections:**
  - **AI languageModel output →** Generate Social Posts (via `ai_languageModel`)
- **Version-specific requirements:**
  - Requires n8n LangChain nodes (`@n8n/n8n-nodes-langchain`) installed and compatible with your n8n version.
- **Edge cases / failures:**
  - Invalid API key / quota exceeded.
  - Safety filters block content depending on prompt and article text.
  - Model output truncation if token limits are too low.

#### Node: “Generate Social Posts”
- **Type / role:** `chainLlm` — runs a LangChain “LLM chain” to prompt the model and generate content.
- **Configuration (interpreted):**
  - Parameters empty in JSON; typically includes:
    - A **prompt template** using inputs: article content, title, target platforms, tone, constraints.
    - Possibly an **output parser** instruction (“Return valid JSON with fields …”).
- **Key expressions / variables:**
  - Would reference fetched article payload (e.g., `{{$json.body}}` or `{{$json}}` depending on HTTP node response).
  - Would reference config fields set earlier (platform list, number of posts, etc.).
- **Connections:**
  - **Main input ←** Fetch Blog Article
  - **AI model input ←** Google Gemini Chat Model (ai_languageModel)
  - **Main output →** Format Social Media Output
- **Edge cases / failures:**
  - Prompt not robust: model may return non-JSON, mixed formatting, or ignore character limits.
  - Too-long article content causes token overflow or high latency/cost.
  - If HTTP node returns HTML, the model may produce noisy output unless you strip boilerplate.

---

### Block 1.3 — Output Formatting & Webhook Response

**Overview:** Converts the raw LLM result into a predictable structured payload and returns it to the webhook caller.

**Nodes involved**
- **Format Social Media Output** (Code)
- **Return Generated Content** (Respond to Webhook)
- (Sticky note in this area: **Step 3 - Format & Respond**)

#### Node: “Format Social Media Output”
- **Type / role:** `Code` — post-processes LLM output (e.g., parse JSON, split posts, enforce schema).
- **Configuration (interpreted):**
  - Parameters empty in JSON; expected responsibilities:
    - Read the LLM output text field produced by the chain node.
    - Normalize into something like:
      - `posts: [{ platform, text, hashtags, callToAction }]`
      - `sourceUrl`, `generatedAt`, etc.
    - Handle non-JSON LLM outputs safely (fallback parsing).
- **Key expressions / variables:**
  - Access to previous node output via `$json` (exact field names depend on Generate Social Posts node output structure).
- **Connections:**
  - **Input ←** Generate Social Posts
  - **Output →** Return Generated Content
- **Edge cases / failures:**
  - JSON parse errors if the LLM output is not valid JSON.
  - Missing fields (e.g., expected `posts` array absent).
  - Character limit enforcement may require truncation logic.

#### Node: “Return Generated Content”
- **Type / role:** `Respond to Webhook` — sends the final HTTP response back to the webhook caller.
- **Configuration (interpreted):**
  - Parameters empty in JSON; commonly:
    - Response body = the formatted JSON from previous node
    - Status code = `200`
    - Response headers = `Content-Type: application/json`
- **Connections:**
  - **Input ←** Format Social Media Output
- **Edge cases / failures:**
  - If webhook node isn’t configured to “Respond via Respond to Webhook node”, the response may not be delivered correctly.
  - Large response payloads can hit client/proxy limits.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Overview | Sticky Note | Documentation / grouping | — | — |  |
| Step 1 - Trigger & Config | Sticky Note | Documentation / grouping | — | — |  |
| Step 2 - Fetch & Generate | Sticky Note | Documentation / grouping | — | — |  |
| Step 3 - Format & Respond | Sticky Note | Documentation / grouping | — | — |  |
| Receive Blog URL | Webhook | Entry point: receives blog URL request | — | Config |  |
| Config | Set | Normalize input + define generation settings | Receive Blog URL | Fetch Blog Article |  |
| Fetch Blog Article | HTTP Request | Download blog content | Config | Generate Social Posts |  |
| Generate Social Posts | LangChain Chain LLM | Prompt LLM to create social posts | Fetch Blog Article | Format Social Media Output |  |
| Google Gemini Chat Model | Google Gemini Chat Model (LangChain) | LLM provider for chain | — | Generate Social Posts (ai_languageModel) |  |
| Format Social Media Output | Code | Parse/shape LLM output into stable schema | Generate Social Posts | Return Generated Content |  |
| Return Generated Content | Respond to Webhook | Return final payload to caller | Format Social Media Output | — |  |

> Note: All sticky note contents are empty in the provided workflow, so the “Sticky Note” column is blank.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **“Repurpose blog articles into social media posts with Google Gemini”**
   - Ensure workflow setting **Execution Order** is `v1` (matches provided JSON).

2. **Add Trigger: “Receive Blog URL” (Webhook)**
   - Node type: **Webhook**
   - Set **HTTP Method**: `POST` (recommended)
   - Set **Path**: e.g., `repurpose-blog`
   - Set **Response Mode**: **Respond to Webhook node**
   - Decide how the caller provides the URL:
     - Option A (body): `{"url":"https://example.com/blog-post"}`
     - Option B (query): `?url=https://example.com/blog-post`

3. **Add “Config” (Set node)**
   - Node type: **Set**
   - Add fields such as (example structure; adjust to your needs):
     - `blogUrl` = expression pointing to webhook input:
       - If body: `{{$json.body.url}}`
       - If query: `{{$json.query.url}}`
     - `platforms` = e.g. `["LinkedIn","X"]`
     - `tone` = e.g. `"clear, practical, non-hype"`
     - `numPostsPerPlatform` = e.g. `2`
     - `language` = e.g. `"en"`
     - `includeHashtags` = `true`
   - Connect: **Receive Blog URL → Config**

4. **Add “Fetch Blog Article” (HTTP Request)**
   - Node type: **HTTP Request**
   - **Method:** `GET`
   - **URL:** `{{$json.blogUrl}}`
   - **Response Format:** `String` (recommended for HTML/text)
   - Optional hardening:
     - Set a **User-Agent** header
     - Increase timeout
     - Follow redirects
   - Connect: **Config → Fetch Blog Article**

5. **Add “Google Gemini Chat Model”**
   - Node type: **Google Gemini Chat Model** (LangChain)
   - Configure **Credentials**:
     - Add your **Google Gemini / Google AI Studio API key** in n8n credentials
   - Choose a **Model** (examples): `gemini-1.5-flash` (faster) or `gemini-1.5-pro` (higher quality)
   - Optional: set temperature / max tokens
   - This node will be connected to the chain node via the AI language model connection.

6. **Add “Generate Social Posts” (LangChain Chain LLM)**
   - Node type: **Chain LLM**
   - Configure the **prompt** to include:
     - The fetched article text (from HTTP node output)
     - Constraints from Config (platforms, tone, counts, character limits)
     - Output requirements (strongly recommend forcing JSON)
   - Example prompt requirements (conceptual):
     - “Generate N posts per platform… return JSON: {sourceUrl, posts:[{platform,text}]}”
   - Connect:
     - **Fetch Blog Article → Generate Social Posts** (main)
     - **Google Gemini Chat Model → Generate Social Posts** (AI Language Model connection)

7. **Add “Format Social Media Output” (Code node)**
   - Node type: **Code**
   - Implement logic to:
     - Read the LLM output field (depends on chain output)
     - Attempt `JSON.parse` if the LLM returns JSON
     - Fallback: store raw text under a `raw` field if parsing fails
     - Return a clean object, e.g.:
       - `{ sourceUrl, posts, raw, model, generatedAt }`
   - Connect: **Generate Social Posts → Format Social Media Output**

8. **Add “Return Generated Content” (Respond to Webhook)**
   - Node type: **Respond to Webhook**
   - Configure response:
     - Body: use incoming JSON from Code node
     - Status code: `200`
     - Content-Type: `application/json` (if configurable)
   - Connect: **Format Social Media Output → Return Generated Content**

9. **(Optional) Add sticky notes**
   - Add 4 sticky notes titled:
     - “Workflow Overview”
     - “Step 1 - Trigger & Config”
     - “Step 2 - Fetch & Generate”
     - “Step 3 - Format & Respond”
   - Content can remain empty to match the provided workflow.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but contain no text in the provided workflow. | Applies to “Workflow Overview” and Steps 1–3 sticky notes. |
| Consider adding HTML-to-text extraction before prompting Gemini for more consistent results. | General robustness improvement (not present in workflow). |
| Consider validating the incoming URL and handling HTTP errors with explicit branches. | General reliability improvement (not present in workflow). |