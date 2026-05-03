Generate blog posts and social media content with GPT-4.1 and Pexels images

https://n8nworkflows.xyz/workflows/generate-blog-posts-and-social-media-content-with-gpt-4-1-and-pexels-images-12123


# Generate blog posts and social media content with GPT-4.1 and Pexels images

## 1. Workflow Overview

**Purpose:**  
This workflow collects a user’s content requirements via an n8n Form, generates written content with OpenAI (GPT-4.1-mini), finds a relevant royalty-free image via the Pexels API, and then regenerates the final content as **HTML** including the selected image. The output is displayed in an n8n HTML viewer node.

**Typical use cases:**
- Quickly drafting blog posts, landing pages, newsletters, or social media posts
- Auto-matching a representative header/hero image from Pexels
- Producing ready-to-publish HTML snippets

### 1.1 Input Reception (Form)
User provides: topic/keyword, content type, tone, and length.

### 1.2 AI Content Draft Generation
A LangChain Agent generates an initial structured response (intended to be JSON).

### 1.3 Image Keyword Extraction + Pexels Search
Another OpenAI node extracts a single image-search keyword from the draft, then the Pexels API is queried.

### 1.4 Final HTML Content Generation (Content + Image)
A second LangChain Agent produces polished **HTML** content using the original draft and the chosen Pexels image URL.

### 1.5 Result Display
The generated HTML is rendered for viewing.

### 1.6 Alternative Path (Token Optimization)
An optional branch creates both content and image keyword in one model call, then parses JSON in a Code node. (In the provided workflow JSON, this branch is not connected to the main output.)

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception (Form)
**Overview:** Collects user inputs that parameterize all downstream AI prompts (topic, type, tone, length).  
**Nodes involved:** `On form submission`

#### Node: On form submission
- **Type / role:** `n8n-nodes-base.formTrigger` — workflow entrypoint via hosted form submission.
- **Key configuration:**
  - Form title: **“AI Content Generator”**
  - Fields:
    - **Topic or Keyword** (text)
    - **Content Type** (dropdown: Blog Post, Social Media Post, Landing Page, Email Newsletter)
    - **Tone** (dropdown: Professional, Casual, Friendly, Educational)
    - **Content Length** (dropdown: Short, Medium, Long)
- **Key variables used downstream:**
  - `$json['Topic or Keyword']`, `$json['Content Type']`, `$json.Tone`, `$json['Content Length']`
- **Connections:**
  - **Output →** `Create Content with AI`
- **Failure/edge cases:**
  - Empty topic can yield poor outputs or prompt-following issues (consider adding required validation).
  - “Content Length” is “Short/Medium/Long” but prompts say “words”; the model may interpret loosely (not a strict word count).

---

### Block 2 — AI Content Draft Generation
**Overview:** Produces initial content based on the form inputs, intended to be structured JSON.  
**Nodes involved:** `OpenAI 4.1 mini`, `Create Content with AI`

#### Node: OpenAI 4.1 mini
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — provides the chat model to the agent.
- **Configuration choices:**
  - Model: **gpt-4.1-mini**
  - Credentials: `openAiApi Credential`
- **Connections:**
  - **AI language model →** `Create Content with AI` (as its LM provider)
- **Failure/edge cases:**
  - Missing/invalid OpenAI credentials, model not available in your account/region.
  - Rate limits (429) or token limits if users request “Long” and content type is large.

#### Node: Create Content with AI
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — agent that drafts the content.
- **Prompting:**
  - User prompt (templated from form):
    - “Create a {Content Type} about {Topic} … Tone … Length … Remember to respond with valid JSON only.”
  - System message enforces JSON-like structure:
    - **Title:** …
    - **Content:** …
- **Important note:** The system message shows **“Title: … Content: …”** but does not explicitly enforce JSON syntax (quotes/braces). The user prompt says “valid JSON only”, but models sometimes output pseudo-JSON.
- **Connections:**
  - **Input ←** `On form submission`
  - **LM ←** `OpenAI 4.1 mini`
  - **Output →** `Find suitable content keyword`
- **Failure/edge cases:**
  - If the model returns non-JSON or wraps in Markdown code fences, downstream steps that assume clean text may break.
  - Content length instruction uses “words” but form provides “Short/Medium/Long”—can cause ambiguity.

---

### Block 3 — Image Keyword Extraction + Pexels Search
**Overview:** Extracts a single keyword suitable for Pexels search from the generated content, then queries Pexels for images.  
**Nodes involved:** `Find suitable content keyword`, `Pexels Image Search`

#### Node: Find suitable content keyword
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — direct OpenAI call (not an agent) to produce one keyword.
- **Configuration choices:**
  - Model: **gpt-4.1-mini**
  - System prompt: “Give one keyword … Give only the keyword, no explanation.”
  - Input reference: `{{ $json.output }}`
    - This assumes the previous agent output is in `$json.output` (agent nodes commonly place generated text there).
- **Connections:**
  - **Input ←** `Create Content with AI`
  - **Output →** `Pexels Image Search`
- **Failure/edge cases:**
  - If `Create Content with AI` output is not where expected (field name mismatch), keyword prompt may receive empty input.
  - Model may output multi-word phrases; Pexels accepts phrases, but your workflow expects “one keyword”.

#### Node: Pexels Image Search
- **Type / role:** `n8n-nodes-base.httpRequest` — calls Pexels REST API.
- **Configuration choices:**
  - Method: implied GET (default for HTTP Request when only URL + query is used)
  - URL: `https://api.pexels.com/v1/search`
  - Query params:
    - `query = {{ $json.output[0].content[0].text }}`
    - `per_page = 5`
    - `orientation = landscape`
  - Headers:
    - `Authorization = {{ INSERT YOUR PEXELS API KEY HERE }}`
- **Important note about the `query` expression:**  
  The expression references `$json.output[0].content[0].text`, which matches **some** OpenAI response formats, but not others. Depending on the exact node output shape, the keyword might instead be available as a simple string (often `$json.output`).
- **Connections:**
  - **Input ←** `Find suitable content keyword`
  - **Output →** `Create Suitable Content Including Image`
- **Failure/edge cases:**
  - 401/403 if Authorization header is wrong.
  - 429 rate limit (Pexels free: 200 requests/hour per sticky note).
  - Empty `photos` array if no results; downstream node uses `photos[0].url` and would error.
  - Schema mismatch if query expression is wrong → Pexels returns irrelevant results or the request may still succeed with an empty query.

---

### Block 4 — Final HTML Content Generation (Content + Image)
**Overview:** Produces final polished HTML using the initial content draft plus the selected Pexels image URL.  
**Nodes involved:** `OpenAi 4.1 Mini`, `Create Suitable Content Including Image`

#### Node: OpenAi 4.1 Mini
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` — model provider for the final agent.
- **Configuration choices:**
  - Model: **gpt-4.1-mini**
  - Credentials: `openAiApi Credential`
- **Connections:**
  - **AI language model →** `Create Suitable Content Including Image`
- **Failure/edge cases:** Same as other OpenAI LM node (auth, limits, availability).

#### Node: Create Suitable Content Including Image
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — generates HTML combining text + image.
- **Configuration choices:**
  - System message (key parts):
    - “You are an expert in {Content Type} and develop beautiful HTML posting.”
    - Uses original form data via: `$('On form submission').item.json[...]`
    - Injects the initial generated content via: `$('Create Content with AI').item.json.output`
    - Injects image URL via: `{{ $json.photos[0].url }}`
    - Constraints:
      - Output only, no explanation
      - Don’t start with ```html
- **Connections:**
  - **Input ←** `Pexels Image Search` (to access `photos[0].url`)
  - **LM ←** `OpenAi 4.1 Mini`
  - **Output →** `View the Result`
- **Failure/edge cases:**
  - If Pexels returns no photos, `photos[0]` is undefined → expression error.
  - If `Create Content with AI` output is not accessible via `.item.json.output`, the prompt will embed `undefined` and degrade output quality.
  - HTML may not be sanitized; if later published, consider security implications (XSS) depending on where it’s rendered.

---

### Block 5 — Result Display
**Overview:** Renders the HTML output for inspection.  
**Nodes involved:** `View the Result`

#### Node: View the Result
- **Type / role:** `n8n-nodes-base.html` — displays HTML content in n8n.
- **Configuration choices:**
  - HTML: `{{ $json.output }}`
- **Connections:**
  - **Input ←** `Create Suitable Content Including Image`
- **Failure/edge cases:**
  - If previous node outputs HTML under a different field name than `output`, nothing displays.
  - Very large HTML may be truncated depending on UI constraints.

---

### Block 6 — Alternative Token Optimization Path (Not wired into main path)
**Overview:** Generates both the content and a Pexels keyword in a single AI call, then parses the JSON in a Code node. In the provided workflow, this branch does not continue to Pexels/HTML generation automatically.  
**Nodes involved:** `Create Content and Find Suitable Image`, `OpenAI 4.1 mini for Content and Image`, `Extract Content and Image Keyword`

#### Node: OpenAI 4.1 mini for Content and Image
- **Type / role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
- **Configuration choices:** model `gpt-4.1-mini`, OpenAI credentials.
- **Connections:** **AI language model →** `Create Content and Find Suitable Image`

#### Node: Create Content and Find Suitable Image
- **Type / role:** `@n8n/n8n-nodes-langchain.agent`
- **Configuration choices:**
  - Prompts similar to the main content generator but system message requires:
    - `Title`
    - `Content`
    - `Image_keyword` (one word)
- **Connections:**
  - **LM ←** `OpenAI 4.1 mini for Content and Image`
  - **Output →** `Extract Content and Image Keyword`
- **Failure/edge cases:**
  - Same “valid JSON” risk as the other agent.
  - If keyword is not a single word, you may want to normalize it before searching Pexels.

#### Node: Extract Content and Image Keyword
- **Type / role:** `n8n-nodes-base.code` — parses/cleans model output into structured fields.
- **Configuration choices (interpreted):**
  - Reads all incoming items and for each:
    - Strips leading/trailing Markdown code fences like ```json … ```
    - `JSON.parse(raw)`
    - Outputs `{ Title, Content, Image_keyword }`
- **Connections:** No downstream connections in the JSON (`main` output is empty).
- **Failure/edge cases:**
  - `JSON.parse` will throw if the AI response is not strict JSON (common failure).
  - If the model uses different key casing (e.g., `image_keyword`), fields become undefined.
  - Doesn’t guard against missing `item.json.output`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | Form Trigger | Entry point; collects user parameters | — | Create Content with AI | ## 1.Create RAW content from the the user's input |
| OpenAI 4.1 mini | LangChain Chat Model (OpenAI) | Provides GPT-4.1-mini to agent | — | Create Content with AI (ai_languageModel) | ## 1.Create RAW content from the the user's input |
| Create Content with AI | LangChain Agent | Generates initial content draft | On form submission; OpenAI 4.1 mini (LM) | Find suitable content keyword | ## 1.Create RAW content from the the user's input |
| Find suitable content keyword | OpenAI (LangChain) | Extracts one keyword for image search | Create Content with AI | Pexels Image Search | ## 2. Generate suitable image for the content |
| Pexels Image Search | HTTP Request | Searches Pexels for images | Find suitable content keyword | Create Suitable Content Including Image | ## 2. Generate suitable image for the content |
| OpenAi 4.1 Mini | LangChain Chat Model (OpenAI) | Provides GPT-4.1-mini to HTML agent | — | Create Suitable Content Including Image (ai_languageModel) | ## 3. Regenerate the content with picture and HTML format |
| Create Suitable Content Including Image | LangChain Agent | Generates final HTML using content + image URL | Pexels Image Search; OpenAi 4.1 Mini (LM) | View the Result | ## 3. Regenerate the content with picture and HTML format |
| View the Result | HTML | Renders generated HTML | Create Suitable Content Including Image | — | ## 4. Get the results |
| Create Content and Find Suitable Image | LangChain Agent | (Alternative) Generate content + image keyword in one call | OpenAI 4.1 mini for Content and Image (LM) | Extract Content and Image Keyword | ## Alternative way to optimize token usage |
| OpenAI 4.1 mini for Content and Image | LangChain Chat Model (OpenAI) | Model provider for alternative agent | — | Create Content and Find Suitable Image (ai_languageModel) | ## Alternative way to optimize token usage |
| Extract Content and Image Keyword | Code | Parses AI JSON and extracts fields | Create Content and Find Suitable Image | — | ## Alternative way to optimize token usage |
| OpenAI 4.1 mini (node) / OpenAi 4.1 Mini / OpenAI 4.1 mini for Content and Image | Sticky Note (global) | Informational | — | — | (This workflow also includes a general note: setup steps + customization; see Notes section) |
| Sticky Note | Sticky Note | Documentation | — | — | ## AI Content Generator with Auto Pexels Image Matching… (full content in Notes) |
| Sticky Note1 | Sticky Note | Block label | — | — | ## 1.Create RAW content from the the user's input |
| Sticky Note2 | Sticky Note | Block label | — | — | ## 2. Generate suitable image for the content |
| Sticky Note3 | Sticky Note | Block label | — | — | ## 3. Regenerate the content with picture and HTML format |
| Sticky Note4 | Sticky Note | Block label | — | — | ## 4. Get the results |
| Sticky Note5 | Sticky Note | Block label | — | — | ## Alternative way to optimize token usage |

> Note: Sticky Note rows are included here for completeness; they are not executable nodes.

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
2. **Add “On form submission” (Form Trigger)**
   - Title: `AI Content Generator`
   - Add fields:
     1) Text: `Topic or Keyword` (placeholder: “Example: Fitness, Business Tips”)  
     2) Dropdown: `Content Type` options: Blog Post, Social Media Post, Landing Page, Email Newsletter  
     3) Dropdown: `Tone` options: Professional, Casual, Friendly, Educational  
     4) Dropdown: `Content Length` options: Short, Medium, Long
3. **Add “OpenAI 4.1 mini” (LangChain Chat Model → OpenAI)**
   - Model: `gpt-4.1-mini`
   - Set **OpenAI API credential** (create/select an `openAiApi` credential).
4. **Add “Create Content with AI” (LangChain Agent)**
   - Connect **Form Trigger → Agent** (main connection).
   - In the Agent node, set:
     - Prompt text:
       - `Create a {{ $json['Content Type'] }} about: {{ $json['Topic or Keyword'] }}`
       - `Tone: {{ $json.Tone }}`
       - `Length: {{ $json['Content Length'] }} words`
       - `Remember to respond with valid JSON only.`
     - System message (ensure it demands JSON structure with Title and Content).
   - Connect **OpenAI 4.1 mini → Create Content with AI** via the **ai_languageModel** connection.
5. **Add “Find suitable content keyword” (OpenAI / LangChain OpenAI node)**
   - Model: `gpt-4.1-mini`
   - Credentials: same OpenAI credential
   - System message: instruct to output **one keyword only**, using the drafted content as context (reference the prior node output).
   - Connect **Create Content with AI → Find suitable content keyword**.
6. **Add “Pexels Image Search” (HTTP Request)**
   - URL: `https://api.pexels.com/v1/search`
   - Enable **Send Query Parameters** and add:
     - `query`: expression referencing the keyword from previous node  
       - (If your OpenAI node outputs a plain string, use `{{ $json.output }}`; adjust as needed.)
     - `per_page`: `5`
     - `orientation`: `landscape`
   - Enable **Send Headers**:
     - Header `Authorization`: set to your Pexels API key value (e.g., `YOUR_PEXELS_KEY`)
   - Connect **Find suitable content keyword → Pexels Image Search**.
7. **Add “OpenAi 4.1 Mini” (LangChain Chat Model → OpenAI)** for the final HTML generation
   - Model: `gpt-4.1-mini`
   - Credentials: OpenAI credential
8. **Add “Create Suitable Content Including Image” (LangChain Agent)**
   - System message should:
     - Reference the original form fields via `$('On form submission').item.json[...]`
     - Include the draft content from `$('Create Content with AI').item.json.output`
     - Include an image URL from the Pexels response (e.g., `{{ $json.photos[0].url }}`)
     - Require **HTML only** output and no code fences
   - Connect **Pexels Image Search → Create Suitable Content Including Image** (main)
   - Connect **OpenAi 4.1 Mini → Create Suitable Content Including Image** (ai_languageModel)
9. **Add “View the Result” (HTML node)**
   - HTML field: `{{ $json.output }}`
   - Connect **Create Suitable Content Including Image → View the Result**
10. **(Optional) Add the alternative optimization branch**
   - Add `OpenAI 4.1 mini for Content and Image` (LM Chat OpenAI) with `gpt-4.1-mini`
   - Add `Create Content and Find Suitable Image` (Agent) that returns JSON including `Image_keyword`
   - Add `Extract Content and Image Keyword` (Code) to strip code fences and `JSON.parse`
   - If you want it to replace the main path, you must:
     - Connect it to Pexels (using `Image_keyword`)
     - Update the Pexels `query` expression and HTML agent references to use the parsed fields.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “AI Content Generator with Auto Pexels Image Matching” overview and steps: form → AI draft → keyword → Pexels → final HTML → display | Sticky note content (documentation) |
| OpenAI API key setup | https://platform.openai.com |
| Pexels API key setup (free tier mentions 200 requests/hour) | https://www.pexels.com/api |
| Customization: optimize token usage by combining steps; update expressions in Pexels and final content node accordingly | Mentioned in sticky note |
| Suggestion: switch to more advanced models (e.g., GPT-4.1, GPT-5.1, GPT-5.2) for deeper analysis | Mentioned in sticky note |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.