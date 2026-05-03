Generate product ideas from website content with FireCrawl and GPT-4.1

https://n8nworkflows.xyz/workflows/generate-product-ideas-from-website-content-with-firecrawl-and-gpt-4-1-12047


# Generate product ideas from website content with FireCrawl and GPT-4.1

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Generate product ideas from website content with FireCrawl and GPT-4.1  
**Internal name:** AI Website Analyzer to Product Ideas with FireCrawl

**Purpose:**  
Collect a website URL via an n8n form, scrape the site’s main content using Firecrawl (markdown output), then use an OpenAI chat model (GPT‑4.1) through an AI Agent node to generate **3 complementary product/service ideas**, returning **structured HTML** and rendering it in an HTML viewer node.

**Primary use cases:**
- Rapid business/market understanding from a website’s content
- Brainstorming adjacent offerings and revenue streams for SMBs
- Producing a shareable HTML output (report-style) directly in n8n

### Logical Blocks
1. **1.1 Input Reception (Form Trigger)**
   - Captures the website URL from a user-submitted form.
2. **1.2 Website Scraping (Firecrawl HTTP Request)**
   - Sends URL to Firecrawl `/v1/scrape`, retrieves main content as markdown.
3. **1.3 AI Analysis & HTML Generation (LangChain AI Agent + OpenAI Model)**
   - Injects the scraped markdown + original URL into a prompt and generates HTML with 3 ideas.
4. **1.4 Result Rendering (HTML Node)**
   - Displays the generated HTML in n8n.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception (Form Trigger)
**Overview:**  
Collects a single input field (`url`) via an n8n hosted form and starts the workflow when the form is submitted.

**Nodes involved:**
- **On form submission** (Form Trigger)
- (Sticky note context: **Sticky Note1** also labels this section visually)

#### Node: On form submission
- **Type / role:** `n8n-nodes-base.formTrigger` — workflow entry point via n8n Form.
- **Configuration (interpreted):**
  - Form title: **“Website URL”**
  - Single field:
    - Label: `url`
    - Placeholder: `https://tenwasap.com`
- **Key variables/expressions:**
  - Output includes `{{$json.url}}` as the submitted value.
- **Connections:**
  - **Outgoing:** to **Scrape Website Content**
  - **Incoming:** none (trigger)
- **Version-specific notes:**
  - TypeVersion `2.3` (behavior consistent with modern n8n Forms).
- **Edge cases / failure modes:**
  - Invalid or missing URL (no validation is configured here); will propagate to Firecrawl and likely fail there.
  - Users may submit non-HTTP(S) URLs; Firecrawl may reject.

---

### 1.2 Website Scraping (Firecrawl HTTP Request)
**Overview:**  
Calls Firecrawl’s scrape endpoint with the submitted URL, requesting **main content only** and **markdown output**.

**Nodes involved:**
- **Scrape Website Content** (HTTP Request)
- Sticky note: **Sticky Note1** (“1. Get the website information”)

#### Node: Scrape Website Content
- **Type / role:** `n8n-nodes-base.httpRequest` — external API call to Firecrawl.
- **Configuration (interpreted):**
  - Method: `POST`
  - URL: `https://api.firecrawl.dev/v1/scrape`
  - Timeout: **30,000 ms**
  - Sends JSON body:
    - `url`: from the form submission (`{{$json.url}}` from incoming item)
    - `formats`: `["markdown"]`
    - `onlyMainContent`: `true` (tries to exclude nav/footer/boilerplate)
  - Headers:
    - `Authorization: Bearer INSERT YOUR API KEY HERE` (must be replaced)
    - `Content-Type: application/json`
- **Key variables/expressions:**
  - Body uses expression referencing current item: `{{ $json.url }}`
- **Connections:**
  - **Incoming:** from **On form submission**
  - **Outgoing:** to **AI Agent**
- **Version-specific notes:**
  - TypeVersion `4.3` (HTTP Request node with modern options and JSON body mode).
- **Edge cases / failure modes:**
  - **Auth failure (401/403):** API key missing/invalid or “Bearer ” removed.
  - **Timeouts:** large websites or slow responses; 30s may be insufficient.
  - **Invalid URL / blocked site:** Firecrawl may return an error payload, or missing `data.markdown`.
  - **Unexpected response structure:** downstream AI Agent expects `{{$json.data.markdown}}`; if absent, prompt will degrade or error depending on n8n expression handling.
- **Credential note:**
  - Firecrawl key is not stored as an n8n credential here; it’s hardcoded in headers. Consider using n8n Credentials or environment variables for security.

---

### 1.3 AI Analysis & HTML Generation (AI Agent + OpenAI Chat Model)
**Overview:**  
Uses an AI Agent node with an OpenAI chat model (GPT‑4.1) to analyze scraped website markdown and generate **3 feasible product/service ideas**, formatted as **“structured beautiful HTML”**.

**Nodes involved:**
- **OpenAI Chat Model** (LangChain Chat Model)
- **AI Agent** (LangChain Agent)
- Sticky note: **Sticky Note2** (“2. Summarize the website information and change to HTML format”)

#### Node: OpenAI Chat Model
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the LLM to the agent.
- **Configuration (interpreted):**
  - Model: **gpt-4.1**
  - No special model options configured (defaults apply).
  - Built-in tools: none enabled.
- **Credentials:**
  - Uses `openAiApi` credential named **“openAiApi Credential”** (must exist in the n8n instance).
- **Connections:**
  - **Outgoing (ai_languageModel):** to **AI Agent** (supplies the model)
  - **Incoming:** none (it’s a resource node for the agent)
- **Version-specific notes:**
  - TypeVersion `1.3`
- **Edge cases / failure modes:**
  - Missing/invalid OpenAI credential (401).
  - Model availability / permission issues (OpenAI account/model access).
  - Rate limits or quota exhaustion.
  - Latency/timeouts depending on hosting/network.

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates prompt execution using the attached chat model.
- **Configuration (interpreted):**
  - Prompt is explicitly defined (`promptType: define`).
  - Prompt content includes:
    - URL: `{{ $('On form submission').item.json.url }}`
      - Uses a **cross-node reference** to the trigger node’s item.
    - Website information: `{{ $json.data.markdown }}`
      - Uses the current input (from Firecrawl) expecting `data.markdown`.
  - Output requirements:
    - Produce **3 complementary** product/service ideas.
    - For each idea: name, description, target audience, revenue model, why it fits.
    - Default language English.
    - Must be practical for SMBs.
    - Must output **HTML only** (explicitly says no ```html fences).
- **Connections:**
  - **Incoming (main):** from **Scrape Website Content**
  - **Incoming (ai_languageModel):** from **OpenAI Chat Model**
  - **Outgoing (main):** to **View the result in HTML**
- **Version-specific notes:**
  - TypeVersion `3`
- **Edge cases / failure modes:**
  - If Firecrawl response doesn’t include `data.markdown`, the prompt may contain blank/undefined content, producing low-quality output.
  - If the form trigger item isn’t accessible as written (e.g., changed node name), `$('On form submission')...` will break.
  - Model may sometimes return non-HTML or partially formatted HTML; downstream viewer will still render but might look broken.
  - Large markdown may exceed model context limits; consider truncation or summarization if needed.

---

### 1.4 Result Rendering (HTML Node)
**Overview:**  
Takes the AI Agent output and renders it as HTML in the n8n UI.

**Nodes involved:**
- **View the result in HTML**
- Sticky note: **Sticky Note3** (“3. Get the results”)

#### Node: View the result in HTML
- **Type / role:** `n8n-nodes-base.html` — HTML rendering node (display).
- **Configuration (interpreted):**
  - HTML content: `{{ $json.output }}`
    - Assumes the AI Agent outputs a field named `output` containing HTML.
- **Connections:**
  - **Incoming:** from **AI Agent**
  - **Outgoing:** none
- **Version-specific notes:**
  - TypeVersion `1.2`
- **Edge cases / failure modes:**
  - If AI Agent output field name differs (not `output`), nothing renders.
  - If output contains malformed HTML, display may be broken.
  - If the AI returns very large HTML, UI rendering may be slow.

---

## Sticky Notes (Documentation Nodes)
These are non-executing nodes but provide essential operator guidance.

### Sticky Note (main instructions)
- Explains workflow steps, setup requirements, customization ideas.
- Includes links to Firecrawl and OpenAI API key pages.

### Sticky Note4 (video reference)
- Contains a YouTube embed reference: `@[youtube](2jXR8mb-DeY)`

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | n8n-nodes-base.stickyNote | Documentation / instructions | — | — | ## AI Website Analyzer for Product Ideas with FireCrawl; How it works (1–3); How to Set Up (domain valid, Firecrawl key: https://www.firecrawl.dev/app/api-keys, OpenAI key: https://platform.openai.com/settings/organization/api-keys); Customize (webhook, formats, model suggestions) |
| On form submission | n8n-nodes-base.formTrigger | Entry point: collect URL | — | Scrape Website Content | ## 1. Get the website information |
| Scrape Website Content | n8n-nodes-base.httpRequest | Firecrawl scrape API call (markdown main content) | On form submission | AI Agent | ## 1. Get the website information |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation / block label | — | — | ## 1. Get the website information |
| AI Agent | @n8n/n8n-nodes-langchain.agent | LLM-driven analysis + HTML generation | Scrape Website Content; OpenAI Chat Model (ai_languageModel) | View the result in HTML | ## 2. Summarize the website information and change to HTML format |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4.1 chat model to agent | — | AI Agent (ai_languageModel) | ## 2. Summarize the website information and change to HTML format |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation / block label | — | — | ## 2. Summarize the website information and change to HTML format |
| View the result in HTML | n8n-nodes-base.html | Render AI output as HTML | AI Agent | — | ## 3. Get the results |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation / block label | — | — | ## 3. Get the results |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation / video link | — | — | ## Here's how; @[youtube](2jXR8mb-DeY) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **AI Website Analyzer to Product Ideas with FireCrawl** (or your preferred name).
   - Ensure workflow execution order is default (v1 is fine).

2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Name: **On form submission**
   - Configure:
     - Form Title: `Website URL`
     - Add a field:
       - Label: `url`
       - Placeholder: `https://tenwasap.com`
   - This node is the entry point.

3. **Add an HTTP Request node for Firecrawl**
   - Node type: **HTTP Request**
   - Name: **Scrape Website Content**
   - Connect: **On form submission → Scrape Website Content**
   - Configure:
     - Method: `POST`
     - URL: `https://api.firecrawl.dev/v1/scrape`
     - Timeout: `30000` ms
     - Send Headers: enabled
     - Send Body: enabled
     - Body Content Type / Specify Body: **JSON**
     - JSON body (as expression or JSON with expressions):
       - `url`: `{{$json.url}}`
       - `formats`: `["markdown"]`
       - `onlyMainContent`: `true`
     - Headers:
       - `Authorization`: `Bearer <YOUR_FIRECRAWL_API_KEY>`
       - `Content-Type`: `application/json`
   - **Credential setup:** Firecrawl is handled via header here (no n8n credential required unless you choose to implement one).

4. **Add an OpenAI Chat Model node (LangChain)**
   - Node type: **OpenAI Chat Model** (LangChain)
   - Name: **OpenAI Chat Model**
   - Configure:
     - Model: **gpt-4.1**
   - **Create/attach OpenAI credentials:**
     - In n8n Credentials, create **OpenAI API** credential.
     - Paste your OpenAI API key (from: https://platform.openai.com/settings/organization/api-keys).
     - Select this credential in the node.

5. **Add an AI Agent node (LangChain Agent)**
   - Node type: **AI Agent**
   - Name: **AI Agent**
   - Connect (main): **Scrape Website Content → AI Agent**
   - Connect (ai_languageModel): **OpenAI Chat Model → AI Agent**
     - Use the **AI language model** connection type/port.
   - Configure the agent prompt (Define/Custom prompt) to include:
     - URL from the trigger node (cross-node reference):
       - `{{ $('On form submission').item.json.url }}`
     - Markdown from Firecrawl response:
       - `{{ $json.data.markdown }}`
     - Instructions to generate **3 ideas** and output **HTML only**.

6. **Add an HTML node to display results**
   - Node type: **HTML**
   - Name: **View the result in HTML**
   - Connect: **AI Agent → View the result in HTML**
   - Configure HTML field:
     - `{{ $json.output }}`
   - If your AI Agent returns a different field name, adjust accordingly (e.g., `{{$json.text}}`).

7. **(Optional) Add Sticky Notes for operator guidance**
   - Add sticky notes with:
     - Setup steps and links (Firecrawl + OpenAI keys)
     - Block labels (“1…2…3…”) and the YouTube reference if desired

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Firecrawl API key page (free tier mentions up to 500 API calls) | https://www.firecrawl.dev/app/api-keys |
| OpenAI API key page | https://platform.openai.com/settings/organization/api-keys |
| Video reference from sticky note | @[youtube](2jXR8mb-DeY) |
| Security note: Firecrawl key is hardcoded in HTTP headers in this workflow; prefer credentials/env vars for production | Applies to “Scrape Website Content” node configuration |
| Output formatting note: AI is instructed to return HTML without code fences; HTML node renders `{{$json.output}}` | Applies to “AI Agent” → “View the result in HTML” |

