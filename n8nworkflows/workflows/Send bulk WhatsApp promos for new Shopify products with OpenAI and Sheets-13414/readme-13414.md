Send bulk WhatsApp promos for new Shopify products with OpenAI and Sheets

https://n8nworkflows.xyz/workflows/send-bulk-whatsapp-promos-for-new-shopify-products-with-openai-and-sheets-13414


# Send bulk WhatsApp promos for new Shopify products with OpenAI and Sheets

## 1. Workflow Overview

**Purpose:**  
This workflow automatically promotes newly created Shopify products to a list of Shopify customers via WhatsApp. It generates a short product pitch from the product description using OpenAI, detects/handles product image links, verifies customer WhatsApp numbers, sends messages in bulk, logs outcomes in Google Sheets, and throttles execution with a wait between batches.

**Target use cases:**
- Bulk WhatsApp promotion when a new product is added in Shopify
- Semi-safe outbound messaging where numbers are verified before sending
- Tracking “sent vs not sent” outcomes in Google Sheets

### 1.1 Product Event Reception (Shopify)
Triggered by a Shopify event (new product), then prepares product/media data.

### 1.2 Media Handling & Branching
Detects whether there are multiple image links and branches:
- **Multiple images:** loop over images and proceed with AI generation
- **No/invalid images path:** send a notification email and/or a fallback WhatsApp action

### 1.3 AI Copy Generation + Product Context
Uses OpenAI to shorten the product HTML description into a WhatsApp-friendly message, optionally enriched by fetching product data from Shopify as an AI tool.

### 1.4 Customer Retrieval & Cleanup
Pulls all customers from Shopify, cleans/transforms the dataset, then iterates customer-by-customer.

### 1.5 WhatsApp Verification → Send → Logging
For each customer:
- verify WhatsApp number (Rapiwa)
- if verified: send WhatsApp promo and log to “Verified & Sent”
- else: log to “Unverified & Not sent”
Then waits (rate-limiting) before continuing.

---

## 2. Block-by-Block Analysis

### Block A — Shopify Product Trigger & Image Detection
**Overview:** Receives the “new product” signal from Shopify, then inspects product data to identify usable image links and decide the next path.

**Nodes involved:**
- Shopify Trigger
- Code (image link detect)
- If (separate image link)
- (Fallback branch) Rapiwa (Send Whatsapp message)
- (Fallback branch) Send a Notification mail

#### Node: **Shopify Trigger**
- **Type / role:** `n8n-nodes-base.shopifyTrigger` — entry point via Shopify webhook.
- **Configuration (interpreted):** Must be configured to trigger on a product event (typically *Product created*). Uses a generated `webhookId`.
- **Connections:**  
  Output → **Code (image link detect)**
- **Potential failures / edge cases:**
  - Shopify credentials missing/invalid
  - Wrong topic/event configured (won’t trigger)
  - Payload shape differences (e.g., images missing)

#### Node: **Code (image link detect)**
- **Type / role:** `n8n-nodes-base.code` — parses incoming product payload and extracts image URL(s).
- **Configuration:** Parameters are empty in the JSON export, so the actual JS logic is not available here. In practice this node should:
  - read product images from the trigger payload (e.g., `images[]`, `image.src`, etc.)
  - output a normalized field like `imageLinks` (array) or `imageLink` (string)
- **Connections:**  
  Output → **If (separate image link)**
- **Potential failures / edge cases:**
  - Code expects fields not present (expression/JS errors)
  - Empty images array; malformed URLs

#### Node: **If (separate image link)**
- **Type / role:** `n8n-nodes-base.if` — branching based on whether multiple image links exist / need splitting.
- **Configuration:** Not shown (empty params). Typically checks something like:
  - `{{$json.imageLinks.length > 1}}` or `{{$json.imageLink.includes(',')}}`
- **Connections (two branches):**
  - **True** → **Loop Over Image Link**
  - **False** → **Send a Notification mail** AND **Rapiwa (Send Whatsapp message)** (both are connected from the false branch)
- **Edge cases:**
  - If condition uses a field that doesn’t exist, the node may default false or error depending on expression
  - If both fallback nodes run in parallel, ensure they can tolerate missing data

#### Node: **Rapiwa (Send Whatsapp message)** *(fallback path)*
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` — WhatsApp send action (custom/community node).
- **Configuration:** Empty in export; must be configured with:
  - Rapiwa credentials / API key
  - recipient number (likely admin/owner) and message template
- **Connections:** No outgoing connections shown from this node in this branch.
- **Failures / edge cases:**
  - Invalid credentials, rate limits, invalid phone format
  - Sending to a number without WhatsApp capability

#### Node: **Send a Notification mail**
- **Type / role:** `n8n-nodes-base.gmail` — sends an email alert when image handling fails/doesn’t match expected structure.
- **Configuration:** Empty in export; must specify:
  - Gmail OAuth2 credentials
  - To/Subject/Body (likely including product info and error context)
- **Connections:** No outgoing connections shown from this node in this branch.
- **Failures / edge cases:**
  - Gmail auth expired
  - Missing required fields (to/subject/body)

---

### Block B — Image Loop → URL Build → AI Copy Generation (with Shopify tool context)
**Overview:** For products with multiple image links, iterates each image link, builds product URL(s), generates a shortened marketing description with OpenAI, and then proceeds to customer retrieval.

**Nodes involved:**
- Loop Over Image Link
- Create Product URL
- Get a product in Shopify
- Create product description (HTML) into a short

#### Node: **Loop Over Image Link**
- **Type / role:** `n8n-nodes-base.splitInBatches` — loops over an array of image links (batch iteration).
- **Configuration:** Empty in export; should define:
  - batch size (often 1 for per-image processing)
  - input field containing list (must be prepared by the prior Code node)
- **Connections:**
  - Output (loop items) → **Create Product URL**
  - Second output (when complete) is unused (empty)
- **Edge cases:**
  - Input not an array; node won’t iterate as expected
  - Very large list (performance)

#### Node: **Create Product URL**
- **Type / role:** `n8n-nodes-base.set` — sets/overwrites fields (e.g., product URL, image URL per iteration).
- **Configuration:** Empty in export; typically sets:
  - `productUrl` from Shopify handle: `https://yourstore.com/products/{{$json.handle}}`
  - ensures `imageUrl` points to current loop item
- **Connections:**  
  Output → **Create product description (HTML) into a short**
- **Edge cases:**
  - Missing store domain/handle
  - Overwriting fields needed later

#### Node: **Get a product in Shopify**
- **Type / role:** `n8n-nodes-base.shopifyTool` — Shopify “tool” used by the AI node (LangChain tool connector).
- **Configuration:** Empty in export; should define:
  - Operation like “Get Product” by ID/handle from incoming JSON
  - Shopify credentials
- **Connections:**  
  Provides **ai_tool** connection → **Create product description (HTML) into a short**
- **Version-specific note:** Uses the `ai_tool` connection type (AI tool calling), requiring n8n’s AI/LangChain capable versions and the OpenAI LangChain node usage patterns.
- **Edge cases:**
  - Missing product identifier
  - Shopify API errors/permissions

#### Node: **Create product description (HTML) into a short**
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — generates a short WhatsApp-friendly promo text from HTML product description.
- **Configuration:** Empty in export; must set:
  - OpenAI credentials
  - prompt/instructions (e.g., “Summarize in 1–2 sentences, include CTA + URL, avoid spammy wording”)
  - input variable mapping from product HTML/body
  - (optional) allow tool usage: connected Shopify tool can enrich context
- **Connections:**  
  Output → **Get All Customer Data In Shopify Store**
- **Failures / edge cases:**
  - OpenAI auth/rate limits
  - Prompt produces overly long output; may exceed WhatsApp limits or your template constraints
  - Tool calling may fail if tool not configured properly

---

### Block C — Customer Retrieval & Normalization
**Overview:** Pulls customers from Shopify using HTTP, cleans/transforms the dataset into a consistent list (names, phone numbers), then feeds the customer loop.

**Nodes involved:**
- Get All Customer Data In Shopify Store
- Clean Customer Data In Shopify Store
- Loop Over Customer

#### Node: **Get All Customer Data In Shopify Store**
- **Type / role:** `n8n-nodes-base.httpRequest` — fetches customer data via Shopify REST/GraphQL endpoint.
- **Configuration:** Empty in export; should include:
  - method: GET
  - URL like `https://{shop}.myshopify.com/admin/api/{version}/customers.json?limit=250` (plus pagination handling if needed)
  - headers: `X-Shopify-Access-Token`
- **Connections:**  
  Output → **Clean Customer Data In Shopify Store**
- **Edge cases:**
  - Pagination not handled: only first page retrieved
  - Token/permissions insufficient (customers scope)
  - Shopify API versioning changes

#### Node: **Clean Customer Data In Shopify Store**
- **Type / role:** `n8n-nodes-base.code` — normalizes customer records (extract phone, name, etc.).
- **Configuration:** Empty in export; typically:
  - filters customers without phone
  - normalizes phone format to E.164 (e.g., `+212...`)
  - removes duplicates
  - outputs an array suitable for SplitInBatches
- **Connections:**  
  Output → **Loop Over Customer**
- **Edge cases:**
  - Invalid phone formats, country code assumptions
  - Duplicate customers cause repeated sends unless deduped

#### Node: **Loop Over Customer**
- **Type / role:** `n8n-nodes-base.splitInBatches` — iterates customers in batches for verification/sending.
- **Configuration:** Empty in export; should set:
  - batch size (often 1–10 depending on rate limits)
- **Connections:**
  - Second output (per-batch continuation) → **Rapiwa (verify whatsapp number)**  
  - First output is unused (empty) in this workflow export
- **Important note:** The connection uses **index 1** output; this implies the workflow author intended a specific SplitInBatches output behavior. Ensure your n8n version matches SplitInBatches v3 behavior and that the correct output is wired (commonly output 0 is “Items”, output 1 is “No items left” in some patterns—verify in your UI).
- **Edge cases:**
  - Miswired output can prevent iteration entirely
  - Too-large batch size can trigger WhatsApp/API rate limits

---

### Block D — WhatsApp Verification → Conditional Send → Logging → Throttle
**Overview:** Verifies each customer number, sends WhatsApp promo only if verified, logs to Sheets based on outcome, then waits to throttle and continues looping.

**Nodes involved:**
- Rapiwa (verify whatsapp number)
- If
- Rapiwa (send whatsapp message)
- Save data in Sheet Verified & Sent
- Save data Sheet Unverified & Not sent
- Wait

#### Node: **Rapiwa (verify whatsapp number)**
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` — checks if the phone number is valid/registered for WhatsApp.
- **Configuration:** Empty; must map:
  - input phone number field from customer record
  - verification operation/action in Rapiwa node
- **Connections:**  
  Output → **If**
- **Edge cases:**
  - False negatives if number formatting wrong
  - Provider downtime / rate limits

#### Node: **If**
- **Type / role:** `n8n-nodes-base.if` — routes verified vs unverified.
- **Configuration:** Empty; typically checks verification response like:
  - `{{$json.isWhatsapp === true}}` or provider-specific field
- **Connections:**
  - True → **Rapiwa (send whatsapp message)**
  - False → **Save data Sheet Unverified & Not sent**
- **Edge cases:**
  - If verification schema changes, condition breaks
  - Missing fields cause incorrect routing

#### Node: **Rapiwa (send whatsapp message)** *(main send path)*
- **Type / role:** `n8n-nodes-rapiwa.rapiwa` — sends the promotional message to a verified number.
- **Configuration:** Empty; must set:
  - recipient phone from customer data
  - message body combining:
    - OpenAI-generated short description
    - product URL
    - optionally image URL
- **Connections:**  
  Output → **Save data in Sheet Verified & Sent**
- **Edge cases:**
  - Message template requirements (approved templates if using official WhatsApp Business API)
  - Media sending constraints (image URL must be public HTTPS)
  - Rate limits / blocked recipients

#### Node: **Save data in Sheet Verified & Sent**
- **Type / role:** `n8n-nodes-base.googleSheets` — logs successful/attempted sends for verified numbers.
- **Configuration:** Empty; must define:
  - Google Sheets OAuth2 credential
  - spreadsheet + sheet/tab
  - append row mapping (customer id/name/phone, product id/url, timestamp, message id/status)
- **Connections:**  
  Output → **Wait**
- **Edge cases:**
  - Sheets quota limits
  - Wrong column mapping leading to failed append

#### Node: **Save data Sheet Unverified & Not sent**
- **Type / role:** `n8n-nodes-base.googleSheets` — logs numbers that failed verification (no send).
- **Configuration:** Empty; similar mapping but with status “unverified”.
- **Connections:**  
  Output → **Wait**
- **Edge cases:** same as above

#### Node: **Wait**
- **Type / role:** `n8n-nodes-base.wait` — throttling/rate control before continuing loop.
- **Configuration:** Empty in export; should be set to a fixed delay (e.g., 2–10 seconds) or a schedule-based resume. Has a `webhookId` for resume behavior.
- **Connections:**  
  Output → **Loop Over Customer** (continues iteration)
- **Edge cases:**
  - Too short delay → WhatsApp provider bans/rate-limits
  - If using “Wait for webhook”, ensure resume webhook is correctly invoked (otherwise stuck)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Shopify Trigger | n8n-nodes-base.shopifyTrigger | Entry point on Shopify product event | — | Code (image link detect) |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| Code (image link detect) | n8n-nodes-base.code | Extract/normalize image link(s) from product payload | Shopify Trigger | If (separate image link) |  |
| If (separate image link) | n8n-nodes-base.if | Branch based on image link structure | Code (image link detect) | Loop Over Image Link; Send a Notification mail; Rapiwa (Send Whatsapp message) |  |
| Loop Over Image Link | n8n-nodes-base.splitInBatches | Iterate over image URLs | If (separate image link) | Create Product URL |  |
| Create Product URL | n8n-nodes-base.set | Build product URL / set fields | Loop Over Image Link | Create product description (HTML) into a short |  |
| Get a product in Shopify | n8n-nodes-base.shopifyTool | Shopify tool for AI enrichment | — | (ai_tool) Create product description (HTML) into a short |  |
| Create product description (HTML) into a short | @n8n/n8n-nodes-langchain.openAi | Generate short promo copy from HTML | Create Product URL; (ai_tool) Get a product in Shopify | Get All Customer Data In Shopify Store |  |
| Get All Customer Data In Shopify Store | n8n-nodes-base.httpRequest | Fetch customers from Shopify API | Create product description (HTML) into a short | Clean Customer Data In Shopify Store |  |
| Clean Customer Data In Shopify Store | n8n-nodes-base.code | Normalize/dedupe customer records | Get All Customer Data In Shopify Store | Loop Over Customer |  |
| Loop Over Customer | n8n-nodes-base.splitInBatches | Iterate over customers in batches | Clean Customer Data In Shopify Store; Wait | Rapiwa (verify whatsapp number) |  |
| Rapiwa (verify whatsapp number) | n8n-nodes-rapiwa.rapiwa | Verify WhatsApp capability of phone | Loop Over Customer | If |  |
| If | n8n-nodes-base.if | Route verified vs unverified | Rapiwa (verify whatsapp number) | Rapiwa (send whatsapp message); Save data Sheet Unverified & Not sent |  |
| Rapiwa (send whatsapp message) | n8n-nodes-rapiwa.rapiwa | Send promo WhatsApp message to customer | If | Save data in Sheet Verified & Sent |  |
| Save data in Sheet Verified & Sent | n8n-nodes-base.googleSheets | Log verified & sent | Rapiwa (send whatsapp message) | Wait |  |
| Save data Sheet Unverified & Not sent | n8n-nodes-base.googleSheets | Log unverified & not sent | If | Wait |  |
| Wait | n8n-nodes-base.wait | Throttle and continue loop | Save data in Sheet Verified & Sent; Save data Sheet Unverified & Not sent | Loop Over Customer |  |
| Rapiwa (Send Whatsapp message) | n8n-nodes-rapiwa.rapiwa | Fallback WhatsApp message (likely admin alert) | If (separate image link) | — |  |
| Send a Notification mail | n8n-nodes-base.gmail | Fallback email notification | If (separate image link) | — |  |
| Sticky Note5 | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| Sticky Note | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Comment container (empty) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
- Name it: **Automated Bulk Promotion for Shopify New Products**
- Keep it inactive until credentials and rate limits are verified.

2) **Add Shopify Trigger**
- Node: **Shopify Trigger**
- Configure credentials: Shopify Admin (private app/custom app token)
- Event/topic: **Product created** (or equivalent “new product” event)
- Save to generate webhook; ensure Shopify is able to call it.

3) **Add Code node: image extraction**
- Node: **Code** → name: `Code (image link detect)`
- Implement logic to extract image URLs from trigger payload and output:
  - `imageLinks`: array of URLs (preferred), and product identifiers (id/handle/title/body_html)
- Connect: **Shopify Trigger → Code (image link detect)**

4) **Add IF node: decide if links should be split**
- Node: **If** → name: `If (separate image link)`
- Condition example (adapt to your output):
  - True if `{{$json.imageLinks && $json.imageLinks.length > 0}}`
  - Optionally check `> 1` if you only want the loop when multiple images exist
- Connect: **Code (image link detect) → If (separate image link)**

5) **True branch: Split image links**
- Node: **Split In Batches** → name: `Loop Over Image Link`
- Batch size: `1`
- Ensure input items are one per image link (you may need your Code node to output one item per image, or use an Item Lists node to split arrays—your current workflow assumes SplitInBatches can iterate as wired).
- Connect: **If (separate image link) [true] → Loop Over Image Link**

6) **Set node: build product URL**
- Node: **Set** → name: `Create Product URL`
- Add fields such as:
  - `productUrl`: `https://YOUR_STORE_DOMAIN/products/{{$json.handle}}`
  - `imageUrl`: map current image link item
- Connect: **Loop Over Image Link → Create Product URL**

7) **OpenAI node (LangChain): generate short promo**
- Node: **OpenAI (LangChain)** → name: `Create product description (HTML) into a short`
- Credentials: OpenAI API key
- Prompt: instruct to convert HTML description to a short WhatsApp promo, include product name + URL, keep concise.
- Map inputs: product title/body_html/productUrl/imageUrl
- Connect: **Create Product URL → OpenAI node**

8) **(Optional/Recommended) Add Shopify Tool for AI enrichment**
- Node: **Shopify Tool** → name: `Get a product in Shopify`
- Configure operation: fetch product by ID/handle from incoming JSON
- Credentials: Shopify Admin
- Connect via AI tool connection: **Get a product in Shopify (ai_tool) → OpenAI node**

9) **Fetch customers from Shopify**
- Node: **HTTP Request** → name: `Get All Customer Data In Shopify Store`
- Configure:
  - GET customers endpoint (REST or GraphQL)
  - Auth header `X-Shopify-Access-Token`
  - Handle pagination (recommended): loop pages until no `Link: rel="next"` (not present in this JSON; you should implement to avoid partial coverage)
- Connect: **OpenAI node → HTTP Request**

10) **Clean customer dataset**
- Node: **Code** → name: `Clean Customer Data In Shopify Store`
- Transform:
  - Extract phone, first/last name, customer id
  - Normalize phone to E.164
  - Remove empty/duplicates
- Connect: **HTTP Request → Clean Customer Data**

11) **Loop over customers**
- Node: **Split In Batches** → name: `Loop Over Customer`
- Batch size: start with `1` to be safe
- Connect: **Clean Customer Data → Loop Over Customer**
- Ensure the correct output is wired to the next node (verify in your n8n UI which output is “Items” vs “No items left”).

12) **Verify WhatsApp number (Rapiwa)**
- Node: **Rapiwa**
- Name: `Rapiwa (verify whatsapp number)`
- Configure credentials (Rapiwa account/token)
- Operation: verify/check number
- Phone field: map from cleaned customer phone
- Connect: **Loop Over Customer → Rapiwa (verify whatsapp number)**

13) **IF verified**
- Node: **If** → name: `If`
- Condition based on Rapiwa response (provider-specific), e.g. `{{$json.valid === true}}`
- Connect: **Rapiwa verify → If**

14) **True branch: send WhatsApp promo**
- Node: **Rapiwa**
- Name: `Rapiwa (send whatsapp message)`
- Operation: send message
- To: customer phone
- Body: include OpenAI short text + product URL; optionally include image URL if supported
- Connect: **If [true] → Rapiwa send**

15) **Log verified & sent in Google Sheets**
- Node: **Google Sheets**
- Name: `Save data in Sheet Verified & Sent`
- Credentials: Google OAuth2
- Operation: Append
- Spreadsheet + sheet tab: select your “Verified & Sent”
- Map columns: customer info, phone, product id/url, timestamp, status, message id
- Connect: **Rapiwa send → Sheets (Verified & Sent)**

16) **False branch: log unverified**
- Node: **Google Sheets**
- Name: `Save data Sheet Unverified & Not sent`
- Operation: Append
- Sheet tab: “Unverified & Not sent”
- Map columns similarly with status `unverified`
- Connect: **If [false] → Sheets (Unverified & Not sent)**

17) **Add Wait (throttle)**
- Node: **Wait**
- Configure a delay (e.g., 3–10 seconds) to reduce provider rate limiting
- Connect:
  - **Sheets (Verified & Sent) → Wait**
  - **Sheets (Unverified & Not sent) → Wait**
- Connect: **Wait → Loop Over Customer** (to continue batching)

18) **Fallback branch from image IF (optional alerts)**
- From **If (separate image link) [false]** connect:
  - **Gmail** node `Send a Notification mail` (to notify you that image parsing/splitting didn’t match)
  - **Rapiwa** node `Rapiwa (Send Whatsapp message)` (likely to notify an admin number)
- Configure Gmail OAuth2 + message content.
- Configure Rapiwa send to your admin number.

19) **Test execution**
- Trigger with a test product creation in Shopify.
- Confirm:
  - image extraction works
  - OpenAI output is concise
  - customer loop actually iterates (no miswired SplitInBatches output)
  - verification and send behave as expected
  - Sheets logging works
  - Wait throttles properly

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but their content is empty in the provided workflow export. | No additional context was embedded in the workflow comments. |
| Several nodes have empty parameters in the export (notably Code, If, HTTP Request, OpenAI, Rapiwa, Sheets). You must configure them manually as described above; otherwise the workflow cannot run. | Applies across most functional nodes; likely exported from a template with placeholders. |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.