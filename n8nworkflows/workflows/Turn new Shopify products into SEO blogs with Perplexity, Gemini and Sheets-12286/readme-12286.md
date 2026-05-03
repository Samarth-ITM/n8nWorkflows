Turn new Shopify products into SEO blogs with Perplexity, Gemini and Sheets

https://n8nworkflows.xyz/workflows/turn-new-shopify-products-into-seo-blogs-with-perplexity--gemini-and-sheets-12286


# Turn new Shopify products into SEO blogs with Perplexity, Gemini and Sheets

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Turn new Shopify products into SEO blogs with Perplexity, Gemini and Sheets

**Purpose:**  
This workflow detects newly created Shopify products, logs them to Google Sheets, performs SEO/market research using Perplexity, generates a structured product-focused article outline via a Gemini-powered AI agent, converts that structure into styled HTML, stores the generated HTML back in Sheets, and then (via a secondary path) posts “ready” blog entries to Shopify and marks them as posted.

**Target use cases**
- Automated product-to-blog content pipeline for Shopify stores
- SEO content ops with traceability (Google Sheets as a queue + audit log)
- Separation of “generation” and “publishing” phases using status fields

### 1.1 Product Intake & Logging (Shopify → Sheets)
New product created in Shopify triggers the flow; essential fields are normalized and appended to a tracking Google Sheet with status `product data`.

### 1.2 Research (Perplexity)
Perplexity receives product context and returns structured JSON research: intent, pain points, FAQs, keyword ideas.

### 1.3 AI Article Structure Generation (LangChain Agent + Gemini + Parsers)
A LangChain agent uses Gemini as the LLM, plus memory and output parsers to produce strict JSON for an article structure (title/intro/problem/solution/features/specs/usage/comparison/CTA).

### 1.4 HTML Assembly & Persistence (Code → Sheets)
A code node converts the structured JSON into a branded, Shopify-friendly HTML article template, escaping content as needed, then updates the same sheet row to `blog generated` and stores `ai_blog` HTML.

### 1.5 Publishing Queue & Posting (Sheets → Shopify Admin API → Sheets)
A separate branch checks for items with status `blog generated`, posts them to Shopify as blog articles, then updates status to `posted`.

---

## 2. Block-by-Block Analysis

### Block A — Product Intake & Normalization

**Overview:**  
Captures the Shopify “product created” webhook payload, extracts key fields, and standardizes the data shape for later steps.

**Nodes involved:**
- Shopify Trigger
- input set

#### Node: Shopify Trigger
- **Type / role:** `shopifyTrigger` — Webhook trigger on Shopify Admin events.
- **Configuration choices:**
  - Topic: `products/create` (fires when a product is created).
  - Authentication: Admin API Access Token (`accessToken`).
- **Inputs/Outputs:**
  - **Output:** Shopify product object (includes `id`, `title`, `body_html`, `vendor`, etc.).
  - Connected to: **input set**
- **Edge cases / failures:**
  - Shopify webhook delivery delays/duplicates (Shopify may retry).
  - Credential permission issues (Admin API scope lacking products read).
  - Payload differences depending on Shopify API version/store config.

#### Node: input set
- **Type / role:** `set` — Normalize payload to only needed fields.
- **Configuration choices (interpreted):**
  - Creates fields:
    - `id` (number) = `{{$json.id}}`
    - `title` (string) = `{{$json.title}}`
    - `body_html` = `{{$json.body_html}}`
    - `vendor` = `{{$json.vendor}}`
    - `product_type` = `{{$json.product_type}}`
    - `handle` = `{{$json.handle}}`
    - `images` (string) = `{{$json.images[0].src}}`
- **Connections:**
  - Input: Shopify Trigger
  - Output: **input update** (Google Sheets append)
- **Key expressions/variables:**
  - `$json.images[0].src`
- **Edge cases / failures:**
  - If product has no images, `$json.images[0].src` can throw/resolve to `undefined` depending on n8n expression behavior. Safer would be optional chaining or conditional logic.

---

### Block B — Sheet Logging (Append Row)

**Overview:**  
Appends a new row to Google Sheets as a tracking record and sets initial status.

**Nodes involved:**
- input update

#### Node: input update
- **Type / role:** `googleSheets` — Append a row to a tracking sheet.
- **Configuration choices:**
  - Operation: **Append**
  - Document: `shopify_to_blog` (spreadsheet ID `1mKEh5BO...`)
  - Sheet tab: `data` (`gid=0`)
  - Mapping mode: define columns explicitly
  - Matching columns: `id` (present, though append doesn’t need matching—this schema is reused later)
  - Sets:
    - `id, title, handle, images, vendor, body_html, product_type`
    - `status` = `"product data"`
- **Connections:**
  - Input: input set
  - Output: **web search**
- **Edge cases / failures:**
  - OAuth token expiry / missing permissions.
  - Sheet schema mismatch (missing columns like `status`, `ai_blog`).
  - Duplicate product IDs: append will create multiple rows for same `id` unless deduped.

---

### Block C — Market/SEO Research via Perplexity

**Overview:**  
Performs web-informed research and returns structured JSON for downstream writing.

**Nodes involved:**
- web search

#### Node: web search
- **Type / role:** `perplexity` — Chat-style research request.
- **Configuration choices:**
  - Prompt forces **valid JSON only** with a fixed schema:
    - `productSummary`, audiences, goals, pain points, usage scenarios, objections, FAQ ideas
    - `seoResearch` including intent, primary/secondary keywords, angle ideas
  - Uses product fields via expressions:
    - `{{ $json.title }}`, `{{ $json.body_html }}`, `{{ $json.vendor }}`, `{{ $json.product_type }}`
  - Simplify output: enabled (`simplify: true`)
- **Connections:**
  - Input: input update (row data)
  - Output: **content generator**
- **Edge cases / failures:**
  - Model may return invalid JSON despite instructions (commas, code fences, extra text).
  - Rate limits / Perplexity API errors.
  - HTML in `body_html` may inflate prompt length; truncation could help.

---

### Block D — AI Article JSON Generation (Agent + Gemini + Memory + Parsers)

**Overview:**  
A LangChain agent uses Gemini to generate a strict structured JSON article object, relying on an autofixing parser + structured schema.

**Nodes involved:**
- Gemini
- Memory
- Structured
- Gemini3
- Output Parser1
- content generator

#### Node: Gemini
- **Type / role:** `lmChatGoogleGemini` — Primary language model for the agent.
- **Configuration choices:**
  - Default options (no temperature/model overrides shown).
- **Connections:**
  - Output (AI Language Model): to **content generator** (as `ai_languageModel`)
- **Edge cases / failures:**
  - API key restrictions, quota, model availability.
  - Output variability if defaults change.

#### Node: Memory
- **Type / role:** `memoryBufferWindow` — Provides short-term conversational memory to agent.
- **Configuration choices:**
  - `sessionIdType`: custom key
  - `sessionKey`: `={{ $now.minute }}`
    - This groups runs by minute, which can accidentally mix multiple items processed within the same minute.
- **Connections:**
  - Output (AI memory): to **content generator**
- **Edge cases / failures:**
  - Collisions when multiple executions occur in same minute (cross-talk between runs).

#### Node: Structured
- **Type / role:** `outputParserStructured` — Enforces output schema.
- **Configuration choices:**
  - Manual JSON schema matching the required article structure fields.
- **Connections:**
  - Provides `ai_outputParser` input to **Output Parser1**
- **Edge cases / failures:**
  - If the AI output deviates strongly, downstream parser may fail or over-correct.

#### Node: Gemini3
- **Type / role:** `lmChatGoogleGemini` — LLM used by the autofixing parser to repair malformed output.
- **Connections:**
  - Output (AI Language Model): to **Output Parser1**
- **Edge cases / failures:**
  - Autofix may “hallucinate” corrections that pass schema but distort meaning.

#### Node: Output Parser1
- **Type / role:** `outputParserAutofixing` — Attempts to coerce AI output into valid structured JSON.
- **Connections:**
  - Receives:
    - LLM from **Gemini3**
    - Schema from **Structured**
  - Outputs parser hook to **content generator** (`ai_outputParser`)
- **Edge cases / failures:**
  - If output is too broken, autofix can still fail.
  - Parsing latency and extra token cost.

#### Node: content generator
- **Type / role:** `langchain.agent` — Orchestrates prompt + tools (LLM, memory, output parser).
- **Configuration choices:**
  - **System message**: strict JSON-only response with character limits and required fields.
  - **User text prompt**: injects:
    - product data from `$('input update').item.json.*`
    - web research from Perplexity: `{{ $json.message }}`
  - `hasOutputParser: true` ensures parsed output is attached (commonly in `json.output`).
- **Connections:**
  - Input: web search
  - Output: **HTML structuring**
  - AI model input: **Gemini**
  - AI memory input: **Memory**
  - Output parser input: **Output Parser1**
- **Edge cases / failures:**
  - References to `$('input update').item.json...` assume that node exists in the execution context and the correct item index is used. If multiple items flow, `.item` vs `.first()` can cause mismatches.
  - Character limit constraints are not programmatically enforced—model may exceed limits.

---

### Block E — HTML Assembly & Sheet Update (Generation Phase)

**Overview:**  
Converts the structured JSON into styled HTML, escapes problematic characters, writes `ai_blog` and updates the row status to `blog generated`.

**Nodes involved:**
- HTML structuring
- update content
- error handling

#### Node: HTML structuring
- **Type / role:** `code` — JavaScript transformation to Shopify-ready HTML.
- **Configuration choices (interpreted):**
  - Reads:
    - AI structured output: `$input.first().json.output`
    - Product data: `$('input update').first().json`
  - Determines `productImage`:
    - Expects `productData.images` to be an array; otherwise falls back to placeholder image URL.
    - **However:** earlier `input set` stored `images` as a string (`images[0].src`), and `input update` stores that string. This creates a likely mismatch: `productData.images.length` is string length, and `productData.images[0].src` will be `undefined`.
  - Cleans text (whitespace normalization) and escapes JSON-breaking characters.
  - Builds a full HTML template (hero, image, cards, lists, table, CTA).
  - Returns a JSON object shaped like:
    - `json.article.title`
    - `json.article.body_html`
    - `json.article.excerpt`
    - `json.article.published`
    - `json.article.tags`
- **Connections:**
  - Input: content generator
  - Output: update content
- **Edge cases / failures:**
  - If `aiData.problem_pointers` etc. are missing or not arrays, `.map()` will throw.
  - The image handling is inconsistent with upstream data shape (string vs array).
  - Over-escaping: `body_html` is escaped again with `makeJsonSafe`, which might not be necessary if later nodes send JSON with proper encoding; could lead to Shopify rendering escaped characters literally.

#### Node: update content
- **Type / role:** `googleSheets` — Update row to store HTML and status.
- **Configuration choices:**
  - Operation: **Update**
  - Matching column: `id`
  - Updates:
    - `status` = `blog generated`
    - `ai_blog` = `{{$json.article.body_html}}`
    - `id` = `{{ $('input update').item.json.id }}`
- **Connections:**
  - Input: HTML structuring
  - Output: error handling
- **Edge cases / failures:**
  - If the sheet contains duplicate IDs, update may affect an unexpected row (depends on n8n node behavior for matching).
  - Very large HTML may hit Google Sheets cell limits.

#### Node: error handling
- **Type / role:** `switch` — Routes based on presence of `id`.
- **Configuration choices:**
  - Output “available” if `$json.id` exists
  - Output “no output” if `$json.id` does not exist
  - Both outputs are connected:
    - “available” → get product data
    - “no output” → input update
- **Connections:**
  - Input: update content
  - Outputs: get product data, input update
- **Edge cases / failures:**
  - This is not true error handling (it does not catch node execution errors); it only checks for a field.
  - Because “no output” loops back into `input update`, it can create unintended re-append loops if `id` is missing for some reason.

---

### Block F — Publishing Phase (Sheets Queue → Shopify Post → Status Update)

**Overview:**  
Pulls rows with status `blog generated`, posts them to Shopify as blog articles, then marks as `posted`.

**Nodes involved:**
- get product data
- article creation
- update blog post status
- No Operation

#### Node: get product data
- **Type / role:** `googleSheets` — Read rows from the sheet based on status.
- **Configuration choices:**
  - Filter: `status` equals `blog generated`
- **Connections:**
  - Input: error handling
  - Output: article creation
- **Edge cases / failures:**
  - If many rows match, it may post multiple articles in a single run (intended queue behavior).
  - If `ai_blog` is empty/too large, article creation may fail.

#### Node: article creation
- **Type / role:** `httpRequest` — Calls Shopify Admin REST API to create a blog article.
- **Configuration choices:**
  - Method: POST
  - URL: `https://{store_id}.myshopify.com/admin/api/2025-07/blogs/118631235877/articles.json`
    - Requires replacing `{store_id}` with actual Shopify store subdomain.
    - Blog ID is hard-coded: `118631235877`
  - Authentication: Shopify Access Token credential (`shopifyAccessTokenApi`)
  - JSON body includes:
    - `title`: `{{ $('HTML structuring').item.json.article.title }}`
    - `author`: `{{ $json.vendor }}`
    - `tags`: `"Electronics"` (static)
    - `body_html`: `{{ $json.ai_blog }}`
    - `published_at`: `{{$now}}`
- **Connections:**
  - Input: get product data (sheet row with `ai_blog`)
  - Output: update blog post status
- **Edge cases / failures:**
  - API version `2025-07` must be supported by the store; otherwise 404/unsupported.
  - The expression for `title` references **HTML structuring**, which is *not* in this publishing branch item context. This is likely to resolve to empty or wrong data. Typically you should use `{{ $json.title }}` or parse from the sheet row.
  - `author` uses `$json.vendor` but the sheet read must include `vendor`; ensure it’s present.
  - HTML may be double-escaped (depending on HTML structuring logic), causing display issues.
  - Shopify API rate limits (429).

#### Node: update blog post status
- **Type / role:** `googleSheets` — Marks successfully posted items.
- **Configuration choices:**
  - Operation: Update
  - Matching column: `id`
  - Updates: `status` = `posted`, `id` = `{{ $('input update').item.json.id }}`
    - This likely references the wrong node in this branch; should be `{{ $json.id }}` from the row being processed.
- **Connections:**
  - Input: article creation
  - Output: No Operation
- **Edge cases / failures:**
  - Incorrect `id` expression can mark wrong rows or fail to match any row.

#### Node: No Operation
- **Type / role:** `noOp` — Terminator node (does nothing).
- **Connections:**
  - Input: update blog post status
- **Edge cases / failures:** none.

---

### Block G — Sticky Notes / Metadata

**Overview:**  
Workflow includes informational sticky notes for author credits and setup instructions.

**Nodes involved:**
- Sticky Note
- Sticky Note7
- Sticky Note8
- Sticky Note10
- Sticky Note11

#### Sticky Note
- **Type / role:** `stickyNote` — Contains workflow description and setup checklist.
- **Operational impact:** None (documentation only).

#### Sticky Note10 / Sticky Note11
- **Type / role:** `stickyNote` — Author details + image.
- **Operational impact:** None.

#### Sticky Note7 / Sticky Note8
- **Type / role:** `stickyNote` — Empty content (placeholders/layout).
- **Operational impact:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Shopify Trigger | shopifyTrigger | Entry point: detect new Shopify product | — | input set |  |
| input set | set | Normalize Shopify payload fields | Shopify Trigger | input update |  |
| input update | googleSheets | Append product row to tracking sheet | input set; error handling (no output path) | web search |  |
| web search | perplexity | Web/SEO research returning JSON | input update | content generator |  |
| Gemini | lmChatGoogleGemini | LLM for agent | — | content generator (ai_languageModel) |  |
| Memory | memoryBufferWindow | Agent short-term memory | — | content generator (ai_memory) |  |
| Structured | outputParserStructured | Defines required JSON schema | — | Output Parser1 (ai_outputParser) |  |
| Gemini3 | lmChatGoogleGemini | LLM used for autofixing parser | — | Output Parser1 (ai_languageModel) |  |
| Output Parser1 | outputParserAutofixing | Repairs/validates model output to schema | Gemini3; Structured | content generator (ai_outputParser) |  |
| content generator | langchain.agent | Generates article-structure JSON | web search | HTML structuring |  |
| HTML structuring | code | Convert structured JSON to styled HTML | content generator | update content |  |
| update content | googleSheets | Update row with ai_blog + status=blog generated | HTML structuring | error handling |  |
| error handling | switch | Route based on id existence; triggers publish queue | update content | get product data; input update |  |
| get product data | googleSheets | Fetch queue items where status=blog generated | error handling | article creation |  |
| article creation | httpRequest | Create Shopify blog article via Admin API | get product data | update blog post status |  |
| update blog post status | googleSheets | Mark row status=posted | article creation | No Operation |  |
| No Operation | noOp | End node | update blog post status | — |  |
| Sticky Note7 | stickyNote | Layout/placeholder (empty) | — | — |  |
| Sticky Note8 | stickyNote | Layout/placeholder (empty) | — | — |  |
| Sticky Note | stickyNote | “Shopify Product-to-Blog Automation…” description + setup checklist | — | — |  |
| Sticky Note10 | stickyNote | Author details + contact: [manipritraj@gmail.com](mailto:manipritraj@gmail.com), +91-9334888899 | — | — |  |
| Sticky Note11 | stickyNote | Author image: https://i.ibb.co/kg9fW8P4/Screenshot-2025-12-30-at-12-55-21-AM.png | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Shopify Trigger**
   - Add node: **Shopify Trigger**
   - Event/topic: **products/create**
   - Auth: **Admin API Access Token**
   - Connect Shopify credential with product read access.

2. **Add “input set” (Set node)**
   - Add node: **Set**
   - Add fields:
     - `id` = `{{$json.id}}` (Number)
     - `title` = `{{$json.title}}`
     - `body_html` = `{{$json.body_html}}`
     - `vendor` = `{{$json.vendor}}`
     - `product_type` = `{{$json.product_type}}`
     - `handle` = `{{$json.handle}}`
     - `images` = `{{$json.images[0].src}}` (note: this stores a string URL)
   - Connect: **Shopify Trigger → input set**

3. **Add “input update” (Google Sheets Append)**
   - Add node: **Google Sheets**
   - Operation: **Append**
   - Select Spreadsheet (create one) and sheet tab `data`
   - Ensure columns exist: `id,title,handle,images,status,vendor,body_html,product_type,ai_blog` (ai_blog can be blank initially)
   - Map values:
     - `id,title,handle,images,vendor,body_html,product_type` from `$json`
     - `status` = `product data`
   - Connect: **input set → input update**
   - Credential: Google Sheets OAuth2 (account with access to the file)

4. **Add “web search” (Perplexity)**
   - Add node: **Perplexity**
   - Provide API key credential
   - Prompt: include product fields and require JSON-only response (use the same schema as in the workflow)
   - Connect: **input update → web search**

5. **Add LangChain components**
   1) **Gemini (LLM for agent)**
      - Add node: **Google Gemini Chat Model**
      - Attach Gemini API credential
      - Connect to agent as **ai_languageModel**
   2) **Memory**
      - Add node: **Memory Buffer Window**
      - Set `sessionIdType` = custom key
      - `sessionKey` = `{{$now.minute}}` (or safer: `{{$execution.id}}`)
      - Connect to agent as **ai_memory**
   3) **Structured schema parser**
      - Add node: **Structured Output Parser**
      - Schema: the article JSON structure (article_title, introduction, problem fields, arrays, comparative_advantage, recommendations_and_cta)
   4) **Gemini3 (LLM for autofix)**
      - Add node: **Google Gemini Chat Model**
      - Same credential
   5) **Output Parser Autofixing**
      - Add node: **Autofixing Output Parser**
      - Connect:
        - Gemini3 → Autofixing parser (ai_languageModel)
        - Structured parser → Autofixing parser (ai_outputParser)
      - Connect autofixing parser to agent as **ai_outputParser**
   6) **content generator (AI Agent)**
      - Add node: **AI Agent**
      - System message: enforce JSON-only with character limits (as provided)
      - User prompt: pass:
        - product fields (from the sheet append node output)
        - `websearch_data` from Perplexity output
      - Connect: **web search → content generator**

6. **Add “HTML structuring” (Code node)**
   - Add node: **Code**
   - Paste JS that:
     - Reads `aiData` from agent output
     - Builds styled HTML
     - Returns `{ article: { title, body_html, excerpt, published, tags } }`
   - Connect: **content generator → HTML structuring**
   - Important: align image field handling:
     - If `images` is stored as a string URL in Sheets, use it directly (do not treat it like an array).

7. **Add “update content” (Google Sheets Update)**
   - Add node: **Google Sheets**
   - Operation: **Update**
   - Match on `id`
   - Set:
     - `status` = `blog generated`
     - `ai_blog` = `{{$json.article.body_html}}`
   - Connect: **HTML structuring → update content**

8. **Add “error handling” (Switch)**
   - Add node: **Switch**
   - Rule 1: if `id` exists → output “available”
   - Rule 2: if `id` not exists → output “no output”
   - Connect: **update content → error handling**
   - (Optional) Avoid looping “no output” back to append unless you add safeguards.

9. **Publishing queue: “get product data” (Google Sheets Read)**
   - Add node: **Google Sheets**
   - Operation: **Read/Get Many**
   - Filter: `status = blog generated`
   - Connect: **error handling (available) → get product data**

10. **Create Shopify article: “article creation” (HTTP Request)**
    - Add node: **HTTP Request**
    - Method: POST
    - URL: `https://YOUR_STORE.myshopify.com/admin/api/2025-07/blogs/YOUR_BLOG_ID/articles.json`
    - Auth: predefined credential type → **Shopify Access Token**
    - JSON body should reference the row being posted, e.g.:
      - title: from the generated structure (store it in Sheets or derive it), or from `$json.title`
      - body_html: `{{$json.ai_blog}}`
      - author: `{{$json.vendor}}`
    - Connect: **get product data → article creation**

11. **Mark posted: “update blog post status” (Google Sheets Update)**
    - Add node: **Google Sheets**
    - Operation: **Update**
    - Match on `id`
    - Set: `status = posted`
    - Connect: **article creation → update blog post status**

12. **End**
    - Add node: **No Operation** (optional)
    - Connect: **update blog post status → No Operation**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Shopify Product-to-Blog Automation (AI + Google Sheets) with Perplexity & Gemini — includes setup checklist (Shopify Admin API, Sheets, Perplexity key, Gemini key, Blog ID, status column, enable trigger). | Sticky note content inside the workflow. |
| Author Details: Manish Kumar — AI Engineer helping brands scale with Custom Shopify Apps & other AI Automations. Email: [manipritraj@gmail.com](mailto:manipritraj@gmail.com). Phone: +91-9334888899 | Sticky Note10 |
| Author image | https://i.ibb.co/kg9fW8P4/Screenshot-2025-12-30-at-12-55-21-AM.png |