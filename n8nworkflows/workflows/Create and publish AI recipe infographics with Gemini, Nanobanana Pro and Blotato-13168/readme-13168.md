Create and publish AI recipe infographics with Gemini, Nanobanana Pro and Blotato

https://n8nworkflows.xyz/workflows/create-and-publish-ai-recipe-infographics-with-gemini--nanobanana-pro-and-blotato-13168


# Create and publish AI recipe infographics with Gemini, Nanobanana Pro and Blotato

## 1. Workflow Overview

**Purpose:** This workflow turns a single user-provided **dish name** into a complete **AI-generated recipe infographic** and then **publishes it to Facebook** via **Blotato**. It automates: research → recipe structuring → caption writing → infographic prompt creation → image generation (Nanobanana Pro / GeminiGenAI) → status polling → media upload → Facebook posting.

**Target use cases:**
- Food creators generating consistent “recipe card / infographic” posts quickly
- Social media managers automating content production for a food page
- Rapid ideation/testing of multiple dishes with minimal manual effort

### Logical Blocks
1.1 **Input Reception**: Collect dish name via n8n Form Trigger.  
1.2 **Recipe Research & Compilation (Agent + Tool)**: Agent uses Perplexity tool to research and compile a structured recipe text.  
1.3 **Caption Generation (Agent + Structured Parser)**: Generate Facebook-optimized caption and parse into structured JSON.  
1.4 **Infographic Prompt Generation (Agent + Structured Parser)**: Convert recipe into a Vietnamese prompt for infographic generation; parse prompt JSON.  
1.5 **Image Generation & Polling**: Submit prompt to GeminiGenAI image endpoint, wait, poll status until complete/failed.  
1.6 **Publishing (Blotato)**: Prepare final payload, upload media to Blotato, create Facebook post.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception
**Overview:** Captures a single dish name from a web form. This node is the workflow entry point.  
**Nodes involved:** `Submit Dish Name`

#### Node: Submit Dish Name
- **Type / role:** `Form Trigger` — starts workflow when user submits form.
- **Configuration choices:**
  - Form title: “Enter Dish Name”
  - One field: “Dish Name” (placeholder examples: Pho, Sushi, Pizza, Tacos)
  - Description explains the automation outcome (infographic).
- **Key variables/fields produced:** `$json['Dish Name']`
- **Connections:**
  - **Output →** `AI Agent: Recipe Analyzer`
- **Edge cases / failures:**
  - Empty input (user submits blank) may cause weak/invalid downstream outputs.
  - Non-dish text (e.g., “help”) can generate irrelevant recipe/caption/prompt.

---

### 1.2 Recipe Research & Compilation (Agent + Tool)
**Overview:** Uses a Gemini-based agent to research the dish using Perplexity and produce a comprehensive recipe write-up.  
**Nodes involved:** `Gemini LLM`, `AI Agent: Recipe Analyzer`, `AI Research: Recipe & Ingredients`

#### Node: Gemini LLM
- **Type / role:** `Google Gemini Chat Model` — LLM powering the recipe research agent.
- **Configuration choices:** Default options (model specifics not overridden in node parameters).
- **Credentials:** Google PaLM/Gemini credential (`api google`).
- **Connections:**
  - **AI language model →** `AI Agent: Recipe Analyzer`
- **Edge cases / failures:**
  - Credential misconfiguration / quota exhaustion.
  - Model output variability (may affect downstream parsing/quality).

#### Node: AI Research: Recipe & Ingredients
- **Type / role:** `Perplexity Tool` — tool used by the agent for live research.
- **Configuration choices:**
  - Takes an agent-supplied message (uses `$fromAI(...)` internal binding).
- **Credentials:** Perplexity API credential (`Perplexity account`).
- **Connections:**
  - **Tool output →** consumed by `AI Agent: Recipe Analyzer` as an AI tool.
- **Edge cases / failures:**
  - Perplexity API rate limits/timeouts.
  - Tool output may be unstructured; agent must normalize.

#### Node: AI Agent: Recipe Analyzer
- **Type / role:** `LangChain Agent` — orchestrates recipe compilation, using Perplexity tool and Gemini model.
- **Configuration choices (interpreted):**
  - **Input text:** `{{$json['Dish Name']}}`
  - **System message:** Detailed culinary research assistant instructions, requiring:
    - multiple sources, cross-referencing, authenticity notes, structured format (ingredients, time, steps, tips, variations, nutrition).
    - infographic-friendly structuring.
- **Connections:**
  - **Main input ←** `Submit Dish Name`
  - **AI language model ←** `Gemini LLM`
  - **AI tool ←** `AI Research: Recipe & Ingredients`
  - **Main output →** `AI Agent: Facebook Caption Generator`
- **Outputs:**
  - Produces `item.json.output` (free-form structured markdown-like recipe text).
- **Edge cases / failures:**
  - If the agent does not call the tool (despite instructions), recipe may be generic/outdated.
  - Very long outputs may increase token usage and cost and degrade later steps.

---

### 1.3 Caption Generation (Agent + Structured Parser)
**Overview:** Turns the compiled recipe text into a Facebook-ready caption, then parses it into `{ "Caption": "..." }` JSON for reliable downstream usage.  
**Nodes involved:** `Gemini LLM1`, `AI Agent: Facebook Caption Generator`, `Parser: Caption & Metadata`

#### Node: Gemini LLM1
- **Type / role:** `Google Gemini Chat Model` — LLM powering caption agent.
- **Credentials:** same Google credential as above.
- **Connections:**
  - **AI language model →** `AI Agent: Facebook Caption Generator`

#### Node: Parser: Caption & Metadata
- **Type / role:** `Structured Output Parser` — enforces JSON shape for caption.
- **Configuration choices:**
  - JSON schema example:
    - `Caption: String`
- **Connections:**
  - **AI output parser →** `AI Agent: Facebook Caption Generator`
- **Edge cases / failures:**
  - If the agent returns non-JSON or mismatched keys, parsing fails.

#### Node: AI Agent: Facebook Caption Generator
- **Type / role:** `LangChain Agent` — writes an engagement-optimized Facebook caption.
- **Configuration choices:**
  - **Input text:** `{{ $('AI Agent: Recipe Analyzer').item.json.output }}`
  - **System message:** Strong guidance on hooks, CTA, emoji usage, hashtags (5–15), examples.
  - **hasOutputParser:** enabled; expects parser output.
- **Connections:**
  - **Main input ←** `AI Agent: Recipe Analyzer`
  - **AI language model ←** `Gemini LLM1`
  - **AI output parser ←** `Parser: Caption & Metadata`
  - **Main output →** `AI Agent: Infographic Prompt Generator`
- **Output consumed later as:** `$('AI Agent: Facebook Caption Generator').item.json.output.Caption`
- **Edge cases / failures:**
  - Parser failures if Gemini includes extra commentary outside JSON.
  - Content policy issues (unlikely here, but still possible depending on dish context).
  - Excessively long caption may reduce post performance.

---

### 1.4 Infographic Prompt Generation (Agent + Structured Parser)
**Overview:** Converts the recipe content into a highly specific Vietnamese prompt for image generation, then parses it into `{ "Prompt": "..." }`.  
**Nodes involved:** `Gemini LLM2`, `AI Agent: Infographic Prompt Generator`, `Structured Output Parser`

#### Node: Gemini LLM2
- **Type / role:** `Google Gemini Chat Model` — LLM powering prompt generator agent.
- **Credentials:** same Google credential.
- **Connections:**
  - **AI language model →** `AI Agent: Infographic Prompt Generator`

#### Node: Structured Output Parser
- **Type / role:** `Structured Output Parser` — enforces JSON shape for the image prompt.
- **Configuration choices:**
  - JSON schema example:
    - `Prompt: String`
- **Connections:**
  - **AI output parser →** `AI Agent: Infographic Prompt Generator`
- **Edge cases / failures:**
  - If the agent outputs prompt text without valid JSON, parsing fails.
  - If it returns different key name (e.g., “prompt”), downstream expression breaks.

#### Node: AI Agent: Infographic Prompt Generator
- **Type / role:** `LangChain Agent` — creates the final infographic prompt.
- **Configuration choices:**
  - **Input text:** `{{ $('AI Agent: Recipe Analyzer').item.json.output }}`
  - **System message:** Enforces:
    - “All content… in Vietnamese”
    - 1080×1080, clean editorial infographic style, ingredients + steps layout, badges (time/calories), photorealistic feel.
  - **hasOutputParser:** enabled (must output JSON with `Prompt`).
- **Connections:**
  - **Main input ←** `AI Agent: Facebook Caption Generator`
  - **AI language model ←** `Gemini LLM2`
  - **AI output parser ←** `Structured Output Parser`
  - **Main output →** `Generate Infographic Image`
- **Edge cases / failures:**
  - Prompt can exceed API limits if too verbose.
  - Vietnamese requirement may be violated by model drift; results may be partially English.

---

### 1.5 Image Generation & Polling
**Overview:** Submits the prompt to GeminiGenAI image generation, then loops (wait → fetch status → switch) until the image is completed or failed.  
**Nodes involved:** `Generate Infographic Image`, `Wait for Image Rendering`, `Fetch Generated Image`, `Switch: Image Status (Processing / Done / Failed)`

#### Node: Generate Infographic Image
- **Type / role:** `HTTP Request` — calls GeminiGenAI image generation endpoint.
- **Configuration choices:**
  - **POST** `https://api.geminigen.ai/uapi/v1/generate_image`
  - **Content-Type:** multipart/form-data
  - **Auth:** Header-based generic credential (`GeminiGenAI`)
  - Body parameters:
    - `prompt`: `{{ $json.output.Prompt.replaceAll('\\n', ' ') }}`
    - `model`: `imagen-pro`
    - `aspect_ratio`: `1:1`
    - `style`: `Photorealistic`
- **Connections:**
  - **Main input ←** `AI Agent: Infographic Prompt Generator`
  - **Main output →** `Wait for Image Rendering`
- **Edge cases / failures:**
  - Expression failure if `$json.output.Prompt` missing (parser/agent failed).
  - API auth errors (401/403), content rejection, 5xx errors.
  - If API returns no `uuid`, downstream fetch cannot work.

#### Node: Wait for Image Rendering
- **Type / role:** `Wait` — delays before polling the generation status (webhook-based wait node).
- **Configuration choices:** Defaults (no explicit wait duration shown in parameters).
- **Connections:**
  - **Main input ←** `Generate Infographic Image` and also loopback from Switch “Processing”
  - **Main output →** `Fetch Generated Image`
- **Edge cases / failures:**
  - If wait is too short, excessive polling may hit API rate limits.
  - If wait is too long, workflow completion is delayed.
  - In some n8n environments, wait/resume requires proper public URL/webhook configuration.

#### Node: Fetch Generated Image
- **Type / role:** `HTTP Request` — queries generation history/status and retrieves final image URLs.
- **Configuration choices:**
  - **GET** `https://api.geminigen.ai/uapi/v1/history/{{ $json.uuid }}`
  - **Auth:** same header credential (`GeminiGenAI`)
- **Connections:**
  - **Main input ←** `Wait for Image Rendering`
  - **Main output →** `Switch: Image Status (Processing / Done / Failed)`
- **Edge cases / failures:**
  - If incoming item doesn’t include `uuid` (wrong data passed through loop), request URL becomes invalid.
  - API may return different status encoding (number vs string), impacting switch logic.

#### Node: Switch: Image Status (Processing / Done / Failed)
- **Type / role:** `Switch` — branches based on `$json.status`.
- **Configuration choices (rules):**
  - Output “Processing” if `status == 1` (number equals)
  - Output “Completed” if `status == "2"` (string equals)
  - Output “Failed” if `status == "3"` (string equals)
  - Loose type validation enabled
- **Connections:**
  - **Input ←** `Fetch Generated Image`
  - **Processing →** `Wait for Image Rendering` (loop)
  - **Completed →** `Prepare Facebook Post Data`
  - (No explicit connection for Failed path)
- **Edge cases / failures:**
  - Mixed typing: rule 1 uses number, rules 2–3 use string. If API returns consistent numbers, “Completed/Failed” may never match (though loose validation may help). Safer to normalize types.
  - Failed branch is defined but not connected: failures will effectively stop without handling/notification.

---

### 1.6 Publishing (Blotato)
**Overview:** Extracts the final image URL and caption, uploads the image to Blotato media storage, and publishes a Facebook post through Blotato.  
**Nodes involved:** `Prepare Facebook Post Data`, `Upload media on Blotato`, `Facebook: Create Post`

#### Node: Prepare Facebook Post Data
- **Type / role:** `Set` — builds a clean payload for posting.
- **Configuration choices:**
  - Sets:
    - `URL Image` = `{{ $json.generated_image[0].image_url }}`
    - `Caption` = `{{ $('AI Agent: Facebook Caption Generator').item.json.output.Caption }}`
- **Connections:**
  - **Input ←** Switch “Completed”
  - **Output →** `Upload media on Blotato`
- **Edge cases / failures:**
  - If API response structure differs (no `generated_image[0].image_url`), URL becomes undefined.
  - If caption parser failed earlier, caption expression resolves empty/undefined.

#### Node: Upload media on Blotato
- **Type / role:** `Blotato` node — uploads external media URL to Blotato, returning a hosted URL.
- **Configuration choices:**
  - Resource: `media`
  - Media URL: `{{ $json['URL Image'] }}`
- **Credentials:** Blotato API credential (`Blotato GiangxAI`)
- **Connections:**
  - **Input ←** `Prepare Facebook Post Data`
  - **Output →** `Facebook: Create Post`
- **Edge cases / failures:**
  - Blotato auth/account issues.
  - If the image URL is not publicly reachable, upload can fail.

#### Node: Facebook: Create Post
- **Type / role:** `Blotato` node — creates a Facebook post on a selected page/account.
- **Configuration choices:**
  - Platform: Facebook
  - Account: “Giang VT” (accountId `16978`)
  - Facebook Page: `688227101036478`
  - Text: `{{ $('Prepare Facebook Post Data').item.json.Caption }}`
  - Media URL(s): `{{ $('Upload media on Blotato').item.json.url }}`
- **Credentials:** same Blotato credential.
- **Connections:**
  - **Input ←** `Upload media on Blotato`
  - **No downstream nodes**
- **Edge cases / failures:**
  - Facebook permissions not granted inside Blotato.
  - Page publishing restrictions, rate limits, or content moderation flags.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Submit Dish Name | Form Trigger | Collect dish name and start workflow | — | AI Agent: Recipe Analyzer | ## User Input; Collect a single dish name from the user via a form. This input triggers the entire automation flow.; Output: Dish name |
| Gemini LLM | Google Gemini Chat Model | LLM for recipe research agent | — | AI Agent: Recipe Analyzer (AI model) | ## Recipe Research & Structuring; Automatically research the recipe based on the dish name and normalize the data (ingredients, steps, timing).; Output: Structured recipe data |
| AI Research: Recipe & Ingredients | Perplexity Tool | External research tool for the agent | — (invoked by agent) | AI Agent: Recipe Analyzer (AI tool) | ## Recipe Research & Structuring; Automatically research the recipe based on the dish name and normalize the data (ingredients, steps, timing).; Output: Structured recipe data |
| AI Agent: Recipe Analyzer | LangChain Agent | Research, compile, and structure recipe content | Submit Dish Name | AI Agent: Facebook Caption Generator | ## Recipe Research & Structuring; Automatically research the recipe based on the dish name and normalize the data (ingredients, steps, timing).; Output: Structured recipe data |
| Gemini LLM1 | Google Gemini Chat Model | LLM for caption agent | — | AI Agent: Facebook Caption Generator (AI model) | ## Recipe Content & Caption; Generate a social-ready caption optimized for food content and engagement.; Output: Caption text + metadata |
| Parser: Caption & Metadata | Structured Output Parser | Parse caption into JSON `{Caption}` | — | AI Agent: Facebook Caption Generator (AI parser) | ## Recipe Content & Caption; Generate a social-ready caption optimized for food content and engagement.; Output: Caption text + metadata |
| AI Agent: Facebook Caption Generator | LangChain Agent | Generate Facebook caption from recipe | AI Agent: Recipe Analyzer | AI Agent: Infographic Prompt Generator | ## Recipe Content & Caption; Generate a social-ready caption optimized for food content and engagement.; Output: Caption text + metadata |
| Gemini LLM2 | Google Gemini Chat Model | LLM for infographic prompt agent | — | AI Agent: Infographic Prompt Generator (AI model) | ## Infographic Prompt Builder; Convert the structured recipe into a detailed prompt for generating a cooking infographic.; Output: Image generation prompt |
| Structured Output Parser | Structured Output Parser | Parse infographic prompt into JSON `{Prompt}` | — | AI Agent: Infographic Prompt Generator (AI parser) | ## Infographic Prompt Builder; Convert the structured recipe into a detailed prompt for generating a cooking infographic.; Output: Image generation prompt |
| AI Agent: Infographic Prompt Generator | LangChain Agent | Create Vietnamese image prompt for infographic | AI Agent: Facebook Caption Generator | Generate Infographic Image | ## Infographic Prompt Builder; Convert the structured recipe into a detailed prompt for generating a cooking infographic.; Output: Image generation prompt |
| Generate Infographic Image | HTTP Request | Submit image generation request to GeminiGenAI | AI Agent: Infographic Prompt Generator | Wait for Image Rendering | ## Image Generation; Generate the recipe infographic image and wait until rendering is complete.; Output: Final infographic image |
| Wait for Image Rendering | Wait | Pause between polling attempts | Generate Infographic Image; Switch: Image Status (Processing) | Fetch Generated Image | ## Image Generation; Generate the recipe infographic image and wait until rendering is complete.; Output: Final infographic image |
| Fetch Generated Image | HTTP Request | Poll generation status and retrieve image URL(s) | Wait for Image Rendering | Switch: Image Status (Processing / Done / Failed) | ## Image Generation; Generate the recipe infographic image and wait until rendering is complete.; Output: Final infographic image |
| Switch: Image Status (Processing / Done / Failed) | Switch | Route based on render status | Fetch Generated Image | Wait for Image Rendering; Prepare Facebook Post Data | ## Image Generation; Generate the recipe infographic image and wait until rendering is complete.; Output: Final infographic image |
| Prepare Facebook Post Data | Set | Build final caption + image URL fields | Switch: Image Status (Completed) | Upload media on Blotato | ## Media Upload & Blotato Posting; Upload the generated infographic to Blotato’s media server, then publish the post to Facebook using Blotato as the posting service.; Output: Facebook post published (via Blotato) |
| Upload media on Blotato | Blotato | Upload media URL to Blotato | Prepare Facebook Post Data | Facebook: Create Post | ## Media Upload & Blotato Posting; Upload the generated infographic to Blotato’s media server, then publish the post to Facebook using Blotato as the posting service.; Output: Facebook post published (via Blotato) |
| Facebook: Create Post | Blotato | Publish Facebook post with text + media | Upload media on Blotato | — | ## Media Upload & Blotato Posting; Upload the generated infographic to Blotato’s media server, then publish the post to Facebook using Blotato as the posting service.; Output: Facebook post published (via Blotato) |
| Sticky Note | Sticky Note | Comment | — | — | ## User Input; Collect a single dish name from the user via a form. This input triggers the entire automation flow.; Output: Dish name |
| Sticky Note1 | Sticky Note | Comment | — | — | ## Recipe Research & Structuring; Automatically research the recipe based on the dish name and normalize the data (ingredients, steps, timing).; Output: Structured recipe data |
| Sticky Note2 | Sticky Note | Comment | — | — | ## Recipe Content & Caption; Generate a social-ready caption optimized for food content and engagement.; Output: Caption text + metadata |
| Sticky Note3 | Sticky Note | Comment | — | — | ## Infographic Prompt Builder; Convert the structured recipe into a detailed prompt for generating a cooking infographic.; Output: Image generation prompt |
| Sticky Note4 | Sticky Note | Comment | — | — | ## Image Generation; Generate the recipe infographic image and wait until rendering is complete.; Output: Final infographic image |
| Sticky Note5 | Sticky Note | Comment | — | — | ## Media Upload & Blotato Posting; Upload the generated infographic to Blotato’s media server, then publish the post to Facebook using Blotato as the posting service.; Output: Facebook post published (via Blotato) |
| Sticky Note6 | Sticky Note | Comment / credits & setup links | — | — | # 🛠️ Workflow Setup Guide; Author: [GiangxAI](https://www.youtube.com/@giangxai.official); Setup guide [n8n](https://n8n.partnerlinks.io/giangxai); GeminigenAi: https://geminigen.ai/ ; Kie ai: https://kie.ai?ref=f8cec88ea15f9ecbff52ccbafa41dd6e ; Blotato: https://blotato.com/?ref=giang9s |

---

## 4. Reproducing the Workflow from Scratch

1) **Create Form Trigger**
   1. Add node: **Form Trigger**
   2. Title: **Enter Dish Name**
   3. Description: “Submit any dish you love…”
   4. Add one field:
      - Label: **Dish Name**
      - Placeholder: “e.g. Pho, Sushi, Pizza, Tacos”
   5. This is the **entry node**.

2) **Add Recipe Research Agent (Gemini + Perplexity tool)**
   1. Add node: **Google Gemini Chat Model** (LangChain)
      - Configure Google Gemini/PaLM credentials.
   2. Add node: **Perplexity Tool**
      - Configure Perplexity API credentials.
   3. Add node: **AI Agent (LangChain Agent)** named “AI Agent: Recipe Analyzer”
      - Input text: `{{$json['Dish Name']}}`
      - System message: culinary research assistant instructions (multi-source recipe, structured output).
      - Connect:
        - Form Trigger → Agent (main)
        - Gemini model → Agent (ai_languageModel)
        - Perplexity Tool → Agent (ai_tool)

3) **Add Caption Generator (Gemini + Structured Output Parser)**
   1. Add node: **Google Gemini Chat Model** named “Gemini LLM1”
   2. Add node: **Structured Output Parser** named “Parser: Caption & Metadata”
      - Schema example:
        - `{ "Caption": "String" }`
   3. Add node: **AI Agent** named “AI Agent: Facebook Caption Generator”
      - Input text: `{{ $('AI Agent: Recipe Analyzer').item.json.output }}`
      - System message: Facebook caption best practices + hashtag guidance
      - Enable output parser (node option **hasOutputParser** / connect parser)
   4. Connect:
      - Recipe Analyzer → Caption Agent (main)
      - Gemini LLM1 → Caption Agent (ai_languageModel)
      - Caption Parser → Caption Agent (ai_outputParser)

4) **Add Infographic Prompt Generator (Gemini + Structured Output Parser)**
   1. Add node: **Google Gemini Chat Model** named “Gemini LLM2”
   2. Add node: **Structured Output Parser** named “Structured Output Parser”
      - Schema example:
        - `{ "Prompt": "String" }`
   3. Add node: **AI Agent** named “AI Agent: Infographic Prompt Generator”
      - Input text: `{{ $('AI Agent: Recipe Analyzer').item.json.output }}`
      - System message: Vietnamese-only prompt + infographic layout rules + 1080×1080
      - Enable output parser
   4. Connect:
      - Caption Agent → Infographic Prompt Agent (main)
      - Gemini LLM2 → Infographic Prompt Agent (ai_languageModel)
      - Prompt Parser → Infographic Prompt Agent (ai_outputParser)

5) **Add Image Generation (GeminiGenAI / Nanobanana Pro)**
   1. Add node: **HTTP Request** named “Generate Infographic Image”
      - Method: **POST**
      - URL: `https://api.geminigen.ai/uapi/v1/generate_image`
      - Authentication: **Header Auth** (Generic Credential)
      - Content type: **multipart/form-data**
      - Body fields:
        - `prompt` = `{{ $json.output.Prompt.replaceAll('\\n', ' ') }}`
        - `model` = `imagen-pro`
        - `aspect_ratio` = `1:1`
        - `style` = `Photorealistic`
      - Create Header Auth credential (e.g., `Authorization: Bearer <token>` or vendor-required header).
   2. Add node: **Wait** named “Wait for Image Rendering”
      - Configure wait mode/duration as desired (defaults are acceptable but consider adding a delay).
   3. Add node: **HTTP Request** named “Fetch Generated Image”
      - Method: **GET**
      - URL: `https://api.geminigen.ai/uapi/v1/history/{{ $json.uuid }}`
      - Same Header Auth credential
   4. Add node: **Switch** named “Switch: Image Status (Processing / Done / Failed)”
      - Check `$json.status`
      - Create outputs:
        - Processing when status == 1
        - Completed when status == 2
        - Failed when status == 3
      - (Recommended) make comparisons consistent (all numbers or all strings).
   5. Connect:
      - Infographic Prompt Agent → Generate Infographic Image
      - Generate Infographic Image → Wait
      - Wait → Fetch Generated Image
      - Fetch Generated Image → Switch
      - Switch(Processing) → Wait (loop)
      - Switch(Completed) → next block (posting)

6) **Prepare payload for posting**
   1. Add node: **Set** named “Prepare Facebook Post Data”
      - Add fields:
        - `URL Image` = `{{ $json.generated_image[0].image_url }}`
        - `Caption` = `{{ $('AI Agent: Facebook Caption Generator').item.json.output.Caption }}`
   2. Connect:
      - Switch(Completed) → Prepare Facebook Post Data

7) **Upload to Blotato and publish to Facebook**
   1. Add node: **Blotato** named “Upload media on Blotato”
      - Resource: **media**
      - Media URL: `{{ $json['URL Image'] }}`
      - Configure Blotato API credential.
   2. Add node: **Blotato** named “Facebook: Create Post”
      - Platform: **facebook**
      - Select **Account** and **Facebook Page** in node dropdowns (requires Blotato account linked to Facebook).
      - Text: `{{ $('Prepare Facebook Post Data').item.json.Caption }}`
      - Media URLs: `{{ $('Upload media on Blotato').item.json.url }}`
   3. Connect:
      - Prepare Facebook Post Data → Upload media on Blotato → Facebook: Create Post

8) **(Optional but recommended) Add failure handling**
   - Connect Switch “Failed” output to a notification node (email/Slack) and/or stop with error details.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Author: GiangxAI | https://www.youtube.com/@giangxai.official |
| n8n setup link | https://n8n.partnerlinks.io/giangxai |
| Image generation engine references: GeminigenAI / Kie.ai | https://geminigen.ai/ ; https://kie.ai?ref=f8cec88ea15f9ecbff52ccbafa41dd6e |
| Blotato references (media upload + posting) | https://blotato.com/?ref=giang9s |
| Workflow behavior summary: end-to-end automation from dish name to Facebook post | Included in the “Workflow Setup Guide” sticky note content |

Dislaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.