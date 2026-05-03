Create WooCommerce products and WordPress posts from product links via Telegram and BrowserAct

https://n8nworkflows.xyz/workflows/create-woocommerce-products-and-wordpress-posts-from-product-links-via-telegram-and-browseract-12360


# Create WooCommerce products and WordPress posts from product links via Telegram and BrowserAct

## 1. Workflow Overview

**Purpose:**  
This workflow listens to Telegram messages, detects when a user sends a product link, scrapes the product page via **BrowserAct**, uses **LLMs (OpenRouter + Gemini)** to generate SEO-focused WooCommerce product copy and a WordPress blog article, then publishes:
- a **WooCommerce product** (with AI-written title/price/description), and  
- a **WordPress post** (a refined article embedding images),  
and finally **attaches scraped images** to the WooCommerce product gallery with SEO alt text.

**Typical use cases:**
- Rapidly turning e-commerce links (AliExpress/Amazon/etc.) into publish-ready WooCommerce products + supporting blog content.
- Lightweight “chat or generate” Telegram bot: if the message is not actionable, respond conversationally or ask for the missing link.

### 1.1 Input Reception & Intent Classification (Telegram → AI router)
Receives a Telegram message, classifies intent (`generate` / `chat` / `noData`), and routes accordingly.

### 1.2 Scraping (BrowserAct)
If `generate`, BrowserAct extracts structured product details from the provided URL.

### 1.3 AI Content Generation (Product + Image SEO + Draft blog)
Transforms scraped data into:
- WooCommerce product content (title, price, descriptions)
- blog draft (headline + HTML)
- per-image SEO metadata (alt text + filename suggestions)

### 1.4 Publishing (WooCommerce + WordPress) and Synchronization
Creates the WooCommerce product and generates a polished WordPress-ready article (with embedded images). Uses a merge gate to proceed once the product exists.

### 1.5 Image Attachment Loop (WooCommerce gallery)
Splits the image list and iteratively updates the WooCommerce product images with SEO alt text.

### 1.6 Conversational Fallback
If message is `chat` or `noData`, a chat agent replies in Telegram (ask for a link or answer normally).

---

## 2. Block-by-Block Analysis

### Block 1 — Telegram Input & Intent Extraction
**Overview:** Receives Telegram messages and uses an AI classifier to output a strict JSON routing object, then routes execution based on intent.  
**Nodes involved:**  
- User Sends Message to Bot  
- Validate user Input  
- Validate user Inputs (Gemini model node)  
- Structured Output Parser  
- Validation Type Switch  
- Process Initialization Alert  

#### Node: **User Sends Message to Bot**
- **Type / role:** Telegram Trigger (`telegramTrigger`) — entry point.
- **Configuration:** Listens to `message` updates.
- **Output:** Telegram update payload; used later for `chat.id` and original text.
- **Edge cases:** Bot not authorized / webhook misconfigured; messages without `text` (stickers, photos) can break downstream expressions referencing `message.text`.

#### Node: **Validate user Input**
- **Type / role:** LangChain Agent — intent classifier + URL extractor.
- **Configuration choices:**
  - Input text: `{{$json.message.text}}`
  - System prompt enforces 3 cases: `generate` with extracted URL, `noData`, or `chat`.
  - **Has output parser** enabled.
- **Key variables/expressions:** reads Telegram message text.
- **Connections:**
  - **Input:** from Telegram Trigger
  - **Output:** to `Validation Type Switch`
  - **AI model:** via `Validate user Inputs`
  - **Output parser:** via `Structured Output Parser`
- **Edge cases:**
  - If user sends a non-text message → classifier input is empty/undefined.
  - The prompt says output keys are `type`/`link`, but downstream Switch checks `$json.output.Type` (capital `Type`)—this mismatch can cause routing failures unless n8n/auto-fix or the model produces `Type`.

#### Node: **Validate user Inputs**
- **Type / role:** Google Gemini Chat Model node (`lmChatGoogleGemini`) used as the LLM backend for the classifier agent/parser.
- **Configuration:** Uses configured Gemini (PaLM) credentials; default options.
- **Connections:** Provides `ai_languageModel` to:
  - `Validate user Input`
  - `Structured Output Parser`
- **Edge cases:** API quota/auth errors; model may return non-JSON without strong prompting (parser attempts to correct).

#### Node: **Structured Output Parser**
- **Type / role:** Structured output parser enforcing classifier schema.
- **Configuration choices:**
  - `autoFix: true`
  - Schema example: `{"type": "generate", "link": "EXTRACTED_URL_HERE"}`
- **Connections:** Feeds parser output into `Validate user Input` as `ai_outputParser`.
- **Edge cases:** If the model returns non-repairable JSON, the workflow will fail here.

#### Node: **Validation Type Switch**
- **Type / role:** Switch — routes by intent type.
- **Configuration choices:**
  - 3 rules checking string equality against `generate`, `chat`, `noData`
  - **Important expression:** `{{$json.output.Type}}` (capital T)
- **Connections:**
  - **Generate branch:** to `Extract Product Details` and `Process Initialization Alert` (both run from the same output)
  - **Chat branch:** to `Chat bot Agent`
  - **NoData branch:** to `Chat bot Agent`
- **Edge cases / integration risks:**
  - Field name mismatch (`type` vs `Type`) may always route to none (or wrong branch).
  - If intent is unexpected value, nothing executes downstream.

#### Node: **Process Initialization Alert**
- **Type / role:** Telegram Send Message — immediate feedback to the user.
- **Configuration choices:**
  - Text: “Okay, give me a few minutes.”
  - chatId: `{{$('User Sends Message to Bot').first().json.message.chat.id}}`
- **Connections:** Runs alongside scraping when `generate`.
- **Edge cases:** Chat ID missing if trigger payload differs; Telegram send failures.

**Sticky note coverage (Step 1 Explanation):**  
“Step 1: Input Analysis & Extraction…” applies to the Telegram trigger and validation/router nodes in this block.

---

### Block 2 — Scraping with BrowserAct
**Overview:** Takes the extracted URL and runs a BrowserAct workflow template to scrape product details.  
**Nodes involved:**  
- Extract Product Details  

#### Node: **Extract Product Details**
- **Type / role:** BrowserAct node (`browserAct`) — executes a BrowserAct “WORKFLOW”.
- **Configuration choices:**
  - `timeout: 7200` seconds (2 hours) — long scrape allowance.
  - BrowserAct `workflowId: 70901224520175622`
  - Workflow input mapping: `input-Product_Link` set to `{{ $json.output.link }}`
  - Not incognito (`open_incognito_mode: false`)
- **Connections:**
  - **Input:** from Switch (generate branch)
  - **Output:** to `Generate Product Data`
- **Edge cases / failure types:**
  - Wrong/expired BrowserAct API key.
  - Workflow ID missing or not accessible in the account.
  - Target site blocks automation / requires captcha/login.
  - Output field expectations: next node reads `$json.output.string` (see next block); if BrowserAct returns a different structure, generation fails.

**Sticky note coverage (Step 2 Explanation):**  
“Step 2: Product Scraping & AI Writing…” applies to scraping + the AI writing nodes in the next block.

---

### Block 3 — AI Product Content + Image SEO List
**Overview:** Uses an LLM agent (OpenRouter model) to transform scraped data into WooCommerce copy, blog draft content, and an image SEO list.  
**Nodes involved:**  
- Generate Product Data  
- Generate Product (OpenRouter model node)  
- Structured Output  
- Check Ouput (Gemini model node; wired as model provider for parser)  

#### Node: **Generate Product Data**
- **Type / role:** LangChain Agent — main copywriting + structuring step.
- **Configuration choices:**
  - Input text: `{{$json.output.string}}` (expects BrowserAct to provide a `string` under `output`)
  - System message defines:
    - WooCommerce title, numeric price formatting (e.g., “EUR38.19 -> 38.19”)
    - Short description hook
    - Rich HTML description with `<h2>` clusters, `<ul>` specs, embed first image at bottom
    - Blog headline and ~300-word review with HTML
    - Image SEO array for every image URL
  - **Has output parser** enabled.
- **Connections:**
  - **AI model:** `Generate Product` (OpenRouter GPT-4.1)
  - **Output parser:** `Structured Output`
  - **Main outputs (fan-out):**
    1) to `Generate Article` (next block)  
    2) to `Await Synchronous Data Arrival` (merge gate, waits for product)  
    3) to `Create a product` (WooCommerce create)
- **Edge cases:**
  - If BrowserAct output is not in `output.string`, the agent gets empty input.
  - Model may output invalid HTML or invalid JSON; parser attempts to fix.
  - Price formatting instruction may be inconsistently followed; can cause WooCommerce validation errors if not numeric.

#### Node: **Generate Product**
- **Type / role:** OpenRouter chat model node (`lmChatOpenRouter`) for the Product Data agent.
- **Configuration:** Model `openai/gpt-4.1`.
- **Connections:** Provides `ai_languageModel` to `Generate Product Data`.
- **Edge cases:** OpenRouter auth, model availability, rate limits.

#### Node: **Structured Output**
- **Type / role:** Structured output parser for `Generate Product Data`.
- **Configuration:**
  - `autoFix: true`
  - Schema includes `woocommerce_data`, `blog_article`, `image_seo_list`
- **Connections:** Provides `ai_outputParser` to `Generate Product Data`.
- **Model backing for parsing:** `Check Ouput` supplies `ai_languageModel` to this parser node.
- **Edge cases:** Unrepairable JSON; missing keys.

#### Node: **Check Ouput**
- **Type / role:** Gemini model node used as the LLM for the parser auto-fix step.
- **Configuration:** Google Gemini (PaLM) credentials; default options.
- **Connections:** `ai_languageModel` → `Structured Output`.
- **Edge cases:** Gemini API failures can break parsing even if OpenRouter generation succeeded.

---

### Block 4 — WooCommerce Product Creation + WordPress Article Refinement
**Overview:** Creates the WooCommerce product, then produces a final blog article HTML (with embedded images) and publishes it to WordPress. A merge node ensures the workflow proceeds to image attachment only when the product exists.  
**Nodes involved:**  
- Create a product  
- Generate Article (agent)  
- Generate  Article (OpenRouter model node)  
- Structured Output3  
- Check Ouputs1 (Gemini model node for parser)  
- Create WordPress Post  
- Await Synchronous Data Arrival  

#### Node: **Create a product**
- **Type / role:** WooCommerce node — create product.
- **Configuration choices:**
  - Name: from `Generate Product Data` → `output.woocommerce_data.title`
  - SKU: `Random_Sku_{{$now.toMillis()}}`
  - Sale price: `output.woocommerce_data.price`
  - Description: short + HTML description concatenated
- **Connections:**
  - **Input:** from `Generate Product Data`
  - **Output:** to `Await Synchronous Data Arrival` (input 1)
- **Edge cases:**
  - WooCommerce credentials/permissions.
  - Price must be numeric string; currency symbols may break.
  - HTML may include unsupported tags or very large content.
  - SKU collisions are unlikely but possible if executed simultaneously with same millisecond (rare).

#### Node: **Generate Article**
- **Type / role:** LangChain Agent — refines blog draft into a “magazine-style” post and embeds `<img>` tags throughout.
- **Configuration choices:**
  - Input text is composed from product fields + image list:
    - Title, price, short description
    - Blog headline + blog HTML draft
    - `images URLs: {{$json.output.image_seo_list}}`
  - System message forces:
    - distribute images; hero image after opening paragraph
    - output strict JSON: `{ blog_headline, blog_html }`
  - **Has output parser** enabled.
- **Connections:**
  - **AI model:** `Generate  Article` (OpenRouter → gemini-2.5-pro)
  - **Output parser:** `Structured Output3`
  - **Main output:** to `Create WordPress Post`
- **Edge cases:**
  - If `image_seo_list` is an array of objects, the agent expects “URLs”; it may still work but could embed wrong `src` (objects rather than strings) unless the model adapts.
  - Article HTML could exceed WordPress limits depending on hosting/security plugins.

#### Node: **Generate  Article**
- **Type / role:** OpenRouter chat model node for the article refinement agent.
- **Configuration:** Model `google/gemini-2.5-pro`.
- **Connections:** `ai_languageModel` → `Generate Article`.
- **Edge cases:** Model availability, OpenRouter rate limits.

#### Node: **Structured Output3**
- **Type / role:** Structured output parser for final article JSON.
- **Configuration:**
  - `autoFix: true`
  - Schema: `blog_headline`, `blog_html`
- **Connections:** `ai_outputParser` → `Generate Article`.
- **Model backing:** `Check Ouputs1` supplies `ai_languageModel`.
- **Edge cases:** Invalid JSON, missing keys.

#### Node: **Check Ouputs1**
- **Type / role:** Gemini model node used by parser auto-fix.
- **Connections:** `ai_languageModel` → `Structured Output3`.
- **Edge cases:** Gemini API failures interrupt parsing.

#### Node: **Create WordPress Post**
- **Type / role:** WordPress node — create post.
- **Configuration choices:**
  - Title: `{{$json.output.blog_headline}}`
  - Content: `{{$json.output.blog_html}}`
- **Connections:** from `Generate Article`.
- **Edge cases:**
  - WordPress credentials/permissions.
  - REST API blocked by security plugin.
  - HTML sanitation may strip styles/tags.

#### Node: **Await Synchronous Data Arrival**
- **Type / role:** Merge node (`chooseBranch`) — acts as a synchronization gate.
- **Configuration:** `useDataOfInput: 2` (use data from Input 2).
- **How it works here:**
  - Input 1: from `Create a product` (ensures product exists)
  - Input 2: from `Generate Product Data` (contains `image_seo_list`)
  - Output: continues with image processing using data from Input 2, but only once the merge condition is satisfied.
- **Edge cases:** If product creation fails, image flow never proceeds; if timings are off or node errors occur, merge may not release items.

**Sticky note coverage (Step 3 Explanation):**  
Applies to WooCommerce + WordPress publishing nodes and their immediate AI generation context.

---

### Block 5 — Image Split + Loop + WooCommerce Update
**Overview:** Iterates through the AI-generated image SEO list and updates the WooCommerce product with an image gallery (including alt text and file name suggestions).  
**Nodes involved:**  
- Split out the images  
- Loop Over Items  
- Update Product Image  

#### Node: **Split out the images**
- **Type / role:** Split Out (`splitOut`) — turns `output.image_seo_list` array into individual items.
- **Configuration:** `fieldToSplitOut: output.image_seo_list`
- **Connections:** from `Await Synchronous Data Arrival` → to `Loop Over Items`
- **Edge cases:** If `image_seo_list` missing or not an array, this node fails.

#### Node: **Loop Over Items**
- **Type / role:** Split In Batches — looping control.
- **Configuration:** Default batch behavior (batch size default).
- **Connections:**
  - Input: from `Split out the images`
  - Output (loop path): output index 1 → `Update Product Image`, then `Update Product Image` returns to this node to fetch next batch.
- **Edge cases:** Infinite loop risk is low with standard split-in-batches pattern, but can happen if connections are miswired or if node receives no termination condition (n8n handles end-of-input automatically when used correctly).

#### Node: **Update Product Image**
- **Type / role:** WooCommerce node — update an existing product’s images (gallery).
- **Configuration choices:**
  - Product ID: `{{$('Create a product').first().json.id}}`
  - Images array (single image per iteration):
    - `src: {{$json.source_url}}`
    - `alt: {{$json.alt_text}}`
    - `name: {{$json.file_name_suggestion}}`
- **Connections:** returns to `Loop Over Items` for next iteration.
- **Edge cases:**
  - If WooCommerce disallows remote image URLs, update may fail or images won’t import.
  - Product ID reference depends on `Create a product` having executed successfully.
  - If the WooCommerce “update images” operation overwrites instead of appending (depends on WooCommerce API behavior and node implementation), you may end up with only the last image. Validate expected behavior in your n8n WooCommerce node version.

**Sticky note coverage (Step 4 Explanation):**  
Applies to image processing and WooCommerce image update nodes.

---

### Block 6 — Conversational / Missing-Link Fallback
**Overview:** Handles `chat` and `noData` intents and responds via Telegram.  
**Nodes involved:**  
- Chat bot Agent  
- Google Gemini (Gemini model node)  
- Send Text Message to Telegram Bot  

#### Node: **Chat bot Agent**
- **Type / role:** LangChain Agent — generates a plain-text reply.
- **Configuration choices:**
  - Input: `Input type : {{$json.output.Type}} | User Input : {{ $('User Sends Message to Bot').item.json.message.text }}`
  - System message:
    - If “Nodata” → ask for link
    - If chat → answer normally
    - Output raw text (no markdown fences)
- **Connections:**
  - AI model: `Google Gemini`
  - Main output: to `Send Text Message to Telegram Bot`
- **Edge cases:** Same `Type` vs `type` mismatch risk; also “Nodata” in prompt vs `noData` in classifier rules can cause incorrect behavior unless the model infers it.

#### Node: **Google Gemini**
- **Type / role:** Gemini model node used by Chat bot Agent.
- **Connections:** `ai_languageModel` → `Chat bot Agent`
- **Edge cases:** API quota/auth errors.

#### Node: **Send Text Message to Telegram Bot**
- **Type / role:** Telegram send message — outputs agent reply.
- **Configuration choices:**
  - Text: `{{$json.output}}`
  - chatId: `{{$('User Sends Message to Bot').item.json.message.chat.id}}`
  - parse_mode: HTML (even though agent is told to output raw text)
- **Edge cases:** If agent returns characters interpreted as HTML, Telegram may reject or format unexpectedly.

**Sticky note coverage (Step 4 Explanation1):**  
Applies to the fallback branch handling missing link / chat responses.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| User Sends Message to Bot | telegramTrigger | Entry point: receive Telegram messages | — | Validate user Input | Step 1: Input Analysis & Extraction |
| Validate user Input | langchain.agent | Classify intent + extract URL (JSON) | User Sends Message to Bot | Validation Type Switch | Step 1: Input Analysis & Extraction |
| Validate user Inputs | lmChatGoogleGemini | LLM backend for intent classifier/parser | — | (AI model to Validate user Input, Structured Output Parser) | Step 1: Input Analysis & Extraction |
| Structured Output Parser | outputParserStructured | Enforce JSON schema for intent output | — | (AI parser to Validate user Input) | Step 1: Input Analysis & Extraction |
| Validation Type Switch | switch | Route: generate vs chat vs noData | Validate user Input | Extract Product Details; Process Initialization Alert; Chat bot Agent | Step 1: Input Analysis & Extraction |
| Process Initialization Alert | telegram | Notify user that processing started | Validation Type Switch (generate) | — | Step 2: Product Scraping & AI Writing |
| Extract Product Details | browserAct | Scrape product page via BrowserAct workflow | Validation Type Switch (generate) | Generate Product Data | Step 2: Product Scraping & AI Writing |
| Generate Product Data | langchain.agent | Generate WooCommerce copy + blog draft + image SEO list | Extract Product Details | Generate Article; Await Synchronous Data Arrival; Create a product | Step 2: Product Scraping & AI Writing |
| Generate Product | lmChatOpenRouter | LLM backend (GPT-4.1) for product generation | — | (AI model to Generate Product Data) | Step 2: Product Scraping & AI Writing |
| Structured Output | outputParserStructured | Enforce JSON schema for product generation | — | (AI parser to Generate Product Data) | Step 2: Product Scraping & AI Writing |
| Check Ouput | lmChatGoogleGemini | LLM used for parser auto-fix | — | (AI model to Structured Output) | Step 2: Product Scraping & AI Writing |
| Create a product | wooCommerce | Create WooCommerce product | Generate Product Data | Await Synchronous Data Arrival | Step 3: Product & Post Creation |
| Await Synchronous Data Arrival | merge | Synchronize: wait for product, pass image list | Create a product; Generate Product Data | Split out the images | Step 3: Product & Post Creation |
| Generate Article | langchain.agent | Refine blog post + embed images, output JSON | Generate Product Data | Create WordPress Post | Step 3: Product & Post Creation |
| Generate  Article | lmChatOpenRouter | LLM backend (Gemini 2.5 Pro via OpenRouter) | — | (AI model to Generate Article) | Step 3: Product & Post Creation |
| Structured Output3 | outputParserStructured | Enforce JSON schema for final article | — | (AI parser to Generate Article) | Step 3: Product & Post Creation |
| Check Ouputs1 | lmChatGoogleGemini | LLM used for parser auto-fix | — | (AI model to Structured Output3) | Step 3: Product & Post Creation |
| Create WordPress Post | wordpress | Publish blog article to WordPress | Generate Article | — | Step 3: Product & Post Creation |
| Split out the images | splitOut | Convert image SEO list array to items | Await Synchronous Data Arrival | Loop Over Items | Step 4: Image Management |
| Loop Over Items | splitInBatches | Iterate images list | Split out the images; Update Product Image | Update Product Image | Step 4: Image Management |
| Update Product Image | wooCommerce | Attach/update product images with SEO alt/name | Loop Over Items | Loop Over Items | Step 4: Image Management |
| Chat bot Agent | langchain.agent | Fallback chat or ask for missing link | Validation Type Switch | Send Text Message to Telegram Bot | Step 2-2: Conversational Fallback |
| Google Gemini | lmChatGoogleGemini | LLM backend for fallback agent | — | (AI model to Chat bot Agent) | Step 2-2: Conversational Fallback |
| Send Text Message to Telegram Bot | telegram | Reply to user in Telegram | Chat bot Agent | — | Step 2-2: Conversational Fallback |
| Documentation | stickyNote | Workspace documentation note | — | — | ## ⚡ Workflow Overview & Setup (incl. https://docs.browseract.com links) |
| Step 1 Explanation | stickyNote | Comment: Step 1 | — | — | ### 🔍 Step 1: Input Analysis & Extraction |
| Step 2 Explanation | stickyNote | Comment: Step 2 | — | — | ### 🛍️ Step 2: Product Scraping & AI Writing |
| Step 3 Explanation | stickyNote | Comment: Step 3 | — | — | ### 📦 Step 3: Product & Post Creation |
| Step 4 Explanation | stickyNote | Comment: Step 4 | — | — | ### 🖼️ Step 4: Image Management |
| Step 4 Explanation1 | stickyNote | Comment: Step 2-2 | — | — | ### 💬 Step 2-2: Conversational Fallback |
| Sticky Note | stickyNote | External video link | — | — | https://www.youtube.com/watch?v=euLP9xdv7J0 |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n  
   - Name it similarly to: *Automate product creation from links to WordPress & WooCommerce using Telegram & BrowserAct*.

2. **Add Telegram Trigger**
   - Node: **Telegram Trigger**  
   - Updates: `message`  
   - Credential: connect your **Telegram Bot** token.

3. **Add Intent Classifier Agent**
   - Node: **AI Agent (LangChain)** named **Validate user Input**  
   - Text input: `{{$json.message.text}}`  
   - Enable **Has Output Parser**  
   - System message: classifier rules producing strict JSON with keys `type` and `link`.

4. **Add Structured Output Parser for classifier**
   - Node: **Structured Output Parser**  
   - Enable `autoFix`  
   - Schema example: `{"type":"generate","link":"EXTRACTED_URL_HERE"}`  
   - Connect it as **AI Output Parser** to the classifier agent.

5. **Add Gemini model for classifier/parsing**
   - Node: **Google Gemini Chat Model** (named **Validate user Inputs**)  
   - Credential: **Google Gemini (PaLM) API**  
   - Connect it as **AI Language Model** to:
     - `Validate user Input`
     - `Structured Output Parser`

6. **Add Switch node for routing**
   - Node: **Switch** named **Validation Type Switch**
   - Create 3 rules:
     - if intent equals `generate`
     - if intent equals `chat`
     - if intent equals `noData`
   - **Important:** Ensure the left value matches your actual parsed JSON path, typically:
     - `{{$json.output.type}}` (recommended)  
     If you keep `{{$json.output.Type}}`, then your classifier must output `Type` exactly.
   - Connect:
     - Trigger → Validate user Input → Switch

7. **Add Telegram “processing started” message (generate path)**
   - Node: **Telegram** named **Process Initialization Alert**
   - Operation: send message
   - Text: “Okay, give me a few minutes.”
   - Chat ID: `{{$('User Sends Message to Bot').first().json.message.chat.id}}`
   - Connect from Switch’s `generate` output to this node.

8. **Add BrowserAct scraping node (generate path)**
   - Node: **BrowserAct**
   - Type: `WORKFLOW`
   - Set `workflowId` to your BrowserAct template/workflow ID (here: `70901224520175622`)
   - Map input variable (e.g., `Product_Link`) to the extracted URL: `{{$json.output.link}}`
   - Set a suitable timeout (this workflow uses 7200 seconds).
   - Credential: **BrowserAct API key**
   - Connect from Switch’s `generate` output to BrowserAct.

9. **Add Product Content Generation agent**
   - Node: **AI Agent** named **Generate Product Data**
   - Input text: point to BrowserAct extracted text (match your BrowserAct output). In this workflow: `{{$json.output.string}}`
   - Enable **Has Output Parser**
   - System message: copywriting instructions producing:
     - `woocommerce_data` (title, price, short_description, html_description)
     - `blog_article` (headline, html_content)
     - `image_seo_list` (source_url, alt_text, file_name_suggestion for each image)

10. **Add OpenRouter model for product generation**
    - Node: **OpenRouter Chat Model** named **Generate Product**
    - Model: `openai/gpt-4.1`
    - Credential: **OpenRouter API**
    - Connect as **AI Language Model** to `Generate Product Data`.

11. **Add Structured Output Parser for product generation**
    - Node: **Structured Output**
    - `autoFix: true`
    - Schema matching the 3 top-level keys above
    - Connect as **AI Output Parser** to `Generate Product Data`.

12. **(Optional but used here) Add Gemini model for parser auto-fix**
    - Node: **Google Gemini Chat Model** named **Check Ouput**
    - Connect as **AI Language Model** to `Structured Output`.

13. **Add WooCommerce “Create Product”**
    - Node: **WooCommerce** named **Create a product**
    - Resource: `Product`, Operation: `Create`
    - Name: `{{$('Generate Product Data').first().json.output.woocommerce_data.title}}`
    - Additional fields:
      - SKU: `Random_Sku_{{$now.toMillis()}}`
      - Sale price: `{{$('Generate Product Data').first().json.output.woocommerce_data.price}}`
      - Description: combine short + html description
    - Credential: WooCommerce REST API (consumer key/secret).
    - Connect from `Generate Product Data` to this node.

14. **Add Article Refinement Agent**
    - Node: **AI Agent** named **Generate Article**
    - Input text: compose from product fields + image list (as in the workflow)
    - Enable **Has Output Parser**
    - System message: magazine-style editor + embed `<img>` tags; output strict JSON: `{blog_headline, blog_html}`
    - Connect from `Generate Product Data` to this node.

15. **Add OpenRouter model for article refinement**
    - Node: **OpenRouter Chat Model** named **Generate  Article**
    - Model: `google/gemini-2.5-pro`
    - Credential: OpenRouter API
    - Connect as **AI Language Model** to `Generate Article`.

16. **Add Structured Output Parser for refined article**
    - Node: **Structured Output3**
    - `autoFix: true`
    - Schema: `blog_headline`, `blog_html`
    - Connect as **AI Output Parser** to `Generate Article`.

17. **Add Gemini model for parser auto-fix**
    - Node: **Google Gemini Chat Model** named **Check Ouputs1**
    - Connect as **AI Language Model** to `Structured Output3`.

18. **Add WordPress “Create Post”**
    - Node: **WordPress** named **Create WordPress Post**
    - Title: `{{$json.output.blog_headline}}`
    - Content: `{{$json.output.blog_html}}`
    - Credential: WordPress API credentials (application password / OAuth / whichever your node uses).
    - Connect from `Generate Article` to WordPress.

19. **Add Merge gate to wait for product creation before image updates**
    - Node: **Merge** named **Await Synchronous Data Arrival**
    - Mode: `Choose Branch`
    - Set “Use data of input” = **Input 2**
    - Connect:
      - `Create a product` → Merge Input 1
      - `Generate Product Data` → Merge Input 2

20. **Add Split Out images**
    - Node: **Split Out** named **Split out the images**
    - Field to split out: `output.image_seo_list`
    - Connect Merge → Split Out.

21. **Add loop controller**
    - Node: **Split In Batches** named **Loop Over Items**
    - Connect Split Out → Split In Batches.

22. **Add WooCommerce Update Product (images)**
    - Node: **WooCommerce** named **Update Product Image**
    - Resource: `Product`, Operation: `Update`
    - Product ID: `{{$('Create a product').first().json.id}}`
    - Images: set one image per item:
      - src: `{{$json.source_url}}`
      - alt: `{{$json.alt_text}}`
      - name: `{{$json.file_name_suggestion}}`
    - Connect:
      - Split In Batches (loop output) → Update Product Image
      - Update Product Image → Split In Batches (to continue loop)

23. **Add fallback chat agent**
    - Node: **AI Agent** named **Chat bot Agent**
    - Input: include intent + original text
    - System message: if noData ask for link; if chat respond normally; output raw text
    - Connect Switch `chat` and `noData` outputs → Chat bot Agent.

24. **Add Gemini model for fallback**
    - Node: **Google Gemini Chat Model** named **Google Gemini**
    - Connect as **AI Language Model** to `Chat bot Agent`.

25. **Add Telegram reply node**
    - Node: **Telegram** named **Send Text Message to Telegram Bot**
    - Text: `{{$json.output}}`
    - chatId: `{{$('User Sends Message to Bot').item.json.message.chat.id}}`
    - Connect Chat bot Agent → Telegram send.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow summary and setup requirements: Telegram, BrowserAct, OpenRouter, Google Gemini (PaLM), WordPress, WooCommerce. Mandatory BrowserAct template: **WordPress & WooCommerce Product Management**. | From “Documentation” sticky note |
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| Video link | https://www.youtube.com/watch?v=euLP9xdv7J0 |