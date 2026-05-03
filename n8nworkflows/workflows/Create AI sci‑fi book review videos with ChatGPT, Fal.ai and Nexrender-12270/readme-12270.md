Create AI sci‑fi book review videos with ChatGPT, Fal.ai and Nexrender

https://n8nworkflows.xyz/workflows/create-ai-sci-fi-book-review-videos-with-chatgpt--fal-ai-and-nexrender-12270


# Create AI sci‑fi book review videos with ChatGPT, Fal.ai and Nexrender

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Generate short sci‑fi book review voice lines with OpenAI (ChatGPT), then trigger a Nexrender Cloud render that uses Fal.ai GenAI (via Nexrender “nx:genai”) to generate a podcast-style video segment and render a final vertical video suitable for TikTok/Instagram Reels.

**Target use cases**
- Automated “book review” short-form video production
- Rapid experimentation with LLM scripts + GenAI video generation + After Effects templating
- Batch or scheduled content creation (this demo uses manual trigger)

### Logical blocks
1.1 **Workflow start & book selection**  
Manual start, then pick one random book from a predefined list.

1.2 **LLM voice lines generation**  
Send the selected book metadata to OpenAI to generate 3–5 sentences of review voice lines.

1.3 **LLM output sanitization**  
Clean the LLM text so it can be safely embedded in downstream JSON/templates.

1.4 **Render job creation & execution (Nexrender + Fal.ai)**  
Create a Nexrender render job using an After Effects template, pass text assets (title/author), call Nexrender GenAI (Fal.ai) to generate a video asset, then wait for the render job to finish.

---

## 2. Block-by-Block Analysis

### 1.1 Workflow start & book selection

**Overview:** Starts the workflow manually and selects one random sci‑fi book (title/author/description) from an embedded list.

**Nodes involved**
- **Run book review**
- **Grab one book for review**

#### Node: Run book review
- **Type / role:** `Manual Trigger` — entry point for interactive runs.
- **Configuration choices:** No parameters; click “Execute workflow” to run.
- **Inputs / outputs:**  
  - Input: none  
  - Output → **Grab one book for review**
- **Type version:** 1
- **Edge cases / failures:** None (except instance-level execution restrictions).

#### Node: Grab one book for review
- **Type / role:** `Code` — builds a dataset and returns a single selected item.
- **Configuration choices (interpreted):**
  - Defines an in-node array of ~50 books with `title`, `author`, `description`.
  - Shuffles the array with `sort(() => Math.random() - 0.5)` and returns the first book object.
- **Key variables/logic:**
  - Returns **one item** shaped like:
    - `json.title`
    - `json.author`
    - `json.description`
- **Inputs / outputs:**  
  - Input: trigger item  
  - Output → **Generate voicelines**
- **Type version:** 2
- **Edge cases / failures:**
  - `sort(() => Math.random() - 0.5)` is a common quick shuffle but not perfectly uniform; acceptable for demo randomness.
  - If edited incorrectly and returns a plain object rather than an n8n item, downstream nodes may fail. In n8n Code nodes, ensure the returned value is an item/array of items in the expected format.

---

### 1.2 LLM voice lines generation

**Overview:** Uses OpenAI to generate short podcast-style voice lines for the chosen book, constrained to max 5 sentences and “voice lines only”.

**Nodes involved**
- **Generate voicelines**

#### Node: Generate voicelines
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — chat completion via OpenAI.
- **Configuration choices (interpreted):**
  - **Model:** `gpt-5`
  - **System message:** Instructs the assistant to:
    - Create *only* voice lines (no extra formatting)
    - Max 5 sentences
    - Avoid intro lines
    - Fit the full story into 3–4 sentences
    - Include a score and thoughts on writing
  - **User message:** Injects runtime book fields:
    - `Book title: {{ $json.title }}`
    - `Book author: {{ $json.author }}`
    - `Book description: {{ $json.description }}`
- **Key expressions/variables:**
  - `{{ $json.title }}`, `{{ $json.author }}`, `{{ $json.description }}`
- **Inputs / outputs:**  
  - Input ← **Grab one book for review**  
  - Output → **Sanitize text**
- **Credentials:** OpenAI API (named “Book review demo account”)
- **Type version:** 1.8
- **Edge cases / failures:**
  - OpenAI auth errors (invalid key, revoked key, wrong org/project settings).
  - Model availability/permission issues for `gpt-5`.
  - Output format drift: despite instructions, the LLM might add bullets/quotes; the next node partially mitigates this.
  - Rate limits / timeouts for larger workloads.

---

### 1.3 LLM output sanitization

**Overview:** Normalizes the LLM output to reduce JSON/template escaping issues downstream (quotes and newlines).

**Nodes involved**
- **Sanitize text**

#### Node: Sanitize text
- **Type / role:** `Code` — post-process the OpenAI response content.
- **Configuration choices (interpreted):**
  - Iterates through all incoming items and modifies:
    - `item.json.message.content`
  - Replaces:
    - All double quotes (`"`) with single quotes (`'`)
    - Actual newlines (`\n`) with the literal sequence `\\n` (so it can be embedded safely)
- **Key code behavior:**
  - `replaceAll("\"", "'")`
  - `replaceAll("\n","\\n")`
- **Inputs / outputs:**  
  - Input ← **Generate voicelines**  
  - Output → **Initiate a render**
- **Type version:** 2
- **Edge cases / failures:**
  - If the OpenAI node output structure changes (e.g., no `json.message.content`), this throws an error (`Cannot read properties of undefined`).
  - Converting quotes can subtly alter meaning (e.g., quoted titles); consider more robust escaping if needed.
  - Replacing newlines assumes downstream wants a single-line string with escaped newlines.

---

### 1.4 Render job creation & execution (Nexrender + Fal.ai)

**Overview:** Creates a Nexrender job referencing an uploaded AE template, supplies dynamic text assets (author/title) and a GenAI-generated video layer using Fal.ai, then waits for the job to complete.

**Nodes involved**
- **Initiate a render**
- **Get job**

#### Node: Initiate a render
- **Type / role:** `@nexrender/n8n-nodes-nexrender.nexrender` — creates a render job in Nexrender Cloud.
- **Configuration choices (interpreted):**
  - **Operation:** `create`
  - **Request body:** Constructs a Nexrender job JSON with:
    - **template**
      - `template.id`: `01K8NGN0QYA0RX5AT041BP4KF8` (must be replaced with your uploaded template ID)
      - `composition`: `"Final 2"`
    - **fonts**: includes `"This Is The Future.ttf"` and `"This Is The Future Italic.ttf"` (expected to be uploaded/available with the template)
    - **assets**
      1) Text asset → AE layer `var_author` set to selected book author  
         - Uses expression: `{{ $('Grab one book for review').item.json.author }}`
      2) Text asset → AE layer `var_title` set to selected book title  
         - Uses expression: `{{ $('Grab one book for review').item.json.title }}`
      3) Data asset → sets `"Effects.Style Selection.Slider"` on layer `"Global Control"` to `4`
      4) **Function asset (GenAI)** → `name: "nx:genai"`  
         - Provider: `fal`  
         - Key: `${secrets.FAL_KEY}` (Nexrender secret must exist)  
         - Model: `infinitalk/single-text`  
         - Type: `video`  
         - Target AE layer: `var_video`  
         - Data:
           - `image_url`: fixed example cover image URL
           - `text_input`: book description from the selected book
           - `voice`: `"Eric"`
           - `prompt`: fixed prompt describing a bearded man recording a podcast
           - `num_frames`: `721`
           - `resolution`: `"720p"`
           - `acceleration`: `"high"`
      5) Data asset → sets work area duration to match generated video duration  
         - `composition`: `"Final 2"`
         - `layerName`: `var_video`
         - `property`: `containingComp.workAreaDuration`
         - `expression`: `"layer.source.duration"`
- **Key expressions/variables used:**
  - `{{ $('Grab one book for review').item.json.author }}`
  - `{{ $('Grab one book for review').item.json.title }}`
  - `{{ $('Grab one book for review').item.json.description }}`
  - `${secrets.FAL_KEY}` (resolved inside Nexrender Cloud, not by n8n)
- **Inputs / outputs:**
  - Input ← **Sanitize text** (note: the sanitized voice lines are not actually referenced in the Nexrender request body as provided)
  - Output → **Get job** (returns created job, including `id`)
- **Credentials:** Nexrender API (named “Nexrender Book Review Account”)
- **Type version:** 1
- **Edge cases / failures:**
  - **Template mismatch:** `template.id`, composition name, or layer names (`var_author`, `var_title`, `var_video`, `Global Control`) not present → render fails.
  - **Missing fonts** → text rendering differences or failures depending on template/font handling.
  - **Secrets not set:** `FAL_KEY` missing in Nexrender secrets → GenAI step fails.
  - **Fal.ai limits:** model availability, rate limits, content moderation, timeouts.
  - **Data type mismatches:** AE property paths must match exactly (e.g., `Effects.Style Selection.Slider`).
  - **Important design note:** The generated OpenAI voice lines are not passed into the render job in this JSON. If you intended narration/subtitles from the LLM, you need an additional text asset mapping (e.g., layer `var_script`) using `item.json.message.content`.

#### Node: Get job
- **Type / role:** `@nexrender/n8n-nodes-nexrender.nexrender` — retrieves job status and optionally waits for completion.
- **Configuration choices (interpreted):**
  - **Operation:** `get`
  - **Job ID:** `={{ $json.id }}` (from the “create job” response)
  - **Wait until done:** `true` (polls until final status)
- **Inputs / outputs:**
  - Input ← **Initiate a render**
  - Output: final job object including status and (typically) result/download links depending on Nexrender configuration
- **Credentials:** same Nexrender API credentials
- **Type version:** 1
- **Edge cases / failures:**
  - Long renders may exceed n8n execution time limits depending on instance settings.
  - Polling can fail on network errors; consider retry policies.
  - If job creation fails and returns no `id`, the expression `{{$json.id}}` will be undefined.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run book review | Manual Trigger | Manual entry point | — | Grab one book for review | In this demo, we provide a list of 50 sci-fi books the AI can review. It does so by picking a random book and then generating voicelines for the GenAI function. / Install our n8n community node in your n8n instance, follow the guide from GitHub: https://github.com/nexrender/nexrender-n8n-node / You will also need: Nexrender trial https://www.nexrender.com/get-trial , FAL.ai key https://fal.ai/ , OpenAI key https://platform.openai.com/settings/organization/api-keys / Quickstart https://docs.nexrender.com/cloud/quickstart (upload template/fonts, set **FAL_KEY** secret) / Contact: support@nexrender.com |
| Grab one book for review | Code | Select random book metadata | Run book review | Generate voicelines | In this demo, we provide a list of 50 sci-fi books the AI can review. It does so by picking a random book and then generating voicelines for the GenAI function. / Install node: https://github.com/nexrender/nexrender-n8n-node / Credentials links as above / Quickstart: https://docs.nexrender.com/cloud/quickstart / support@nexrender.com |
| Generate voicelines | OpenAI (LangChain) | Generate short review voice lines | Grab one book for review | Sanitize text | In this demo, we provide a list of 50 sci-fi books the AI can review. It does so by picking a random book and then generating voicelines for the GenAI function. / Install node: https://github.com/nexrender/nexrender-n8n-node / Credentials links as above / Quickstart: https://docs.nexrender.com/cloud/quickstart / support@nexrender.com |
| Sanitize text | Code | Clean LLM output for downstream use | Generate voicelines | Initiate a render | Here we process and clean up the output from LLM, ready for use in our GenAI workflow. |
| Initiate a render | Nexrender | Create Nexrender job; invoke Fal.ai GenAI video; map AE assets | Sanitize text | Get job | This is where the magic happens! Double-click on the **Initiate a render** node to see how the job request looks like. / Update `template.id` to your uploaded template. |
| Get job | Nexrender | Wait for render completion and fetch results | Initiate a render | — | Here we wait for the job to finish rendering. Once we get a result, we can publish the video to social sites, send it via e-mail, post in Slack channel, ... |
| Sticky Note | Sticky Note | Comment (credentials + setup) | — | — | In this demo, we provide a list of 50 sci-fi books… (see full note content in Section 5). |
| Sticky Note1 | Sticky Note | Comment (sanitization) | — | — | Here we process and clean up the output from LLM, ready for use in our GenAI workflow. |
| Sticky Note2 | Sticky Note | Comment (wait/publish ideas) | — | — | Here we wait for the job to finish rendering… |
| Sticky Note3 | Sticky Note | Comment (render request/template id) | — | — | This is where the magic happens… update `template.id`… |
| Sticky Note4 | Sticky Note | Comment (video intro link) | — | — | ## Video Introduction / @[youtube](EZHOHXu4PyA) |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**
   - Name it: `Demo: book review` (or your preferred name).
   - Keep workflow **Inactive** while configuring.

2) **Add Manual Trigger**
   - Node: **Manual Trigger**
   - Name: `Run book review`

3) **Add Code node to select a random book**
   - Node: **Code** (JavaScript)
   - Name: `Grab one book for review`
   - Paste logic that:
     - Defines an array of book objects with `title`, `author`, `description`
     - Returns one random object (as an n8n item)
   - Connect: `Run book review` → `Grab one book for review`

4) **Add OpenAI node (LangChain OpenAI Chat)**
   - Node: **OpenAI** (`@n8n/n8n-nodes-langchain.openAi`)
   - Name: `Generate voicelines`
   - Credentials: configure an **OpenAI API key** in n8n credentials.
   - Model: select `gpt-5` (or another available model).
   - Messages:
     - **System**: instructions to produce only voice lines, max 5 sentences, include score + writing thoughts, no intro.
     - **User**: include expressions:
       - `Book title: {{ $json.title }}`
       - `Book author: {{ $json.author }}`
       - `Book description: {{ $json.description }}`
   - Connect: `Grab one book for review` → `Generate voicelines`

5) **Add Code node to sanitize text**
   - Node: **Code**
   - Name: `Sanitize text`
   - Implement transformations on `item.json.message.content`:
     - Replace `"` with `'`
     - Replace newline characters with `\\n`
   - Connect: `Generate voicelines` → `Sanitize text`

6) **Install the Nexrender community node**
   - Install `@nexrender/n8n-nodes-nexrender` following: https://github.com/nexrender/nexrender-n8n-node
   - Restart n8n if required.

7) **Prepare Nexrender Cloud**
   - Create/obtain a Nexrender account (trial: https://www.nexrender.com/get-trial).
   - Follow quickstart: https://docs.nexrender.com/cloud/quickstart
   - Upload the provided After Effects template + required fonts.
   - In Nexrender secrets, set:
     - `FAL_KEY` = your Fal.ai API key (https://fal.ai/)
   - Generate a Nexrender API key for n8n credential setup.

8) **Add Nexrender node to create a render job**
   - Node: **Nexrender**
   - Name: `Initiate a render`
   - Credentials: Nexrender API key credential in n8n.
   - Operation: **Create**
   - Body: configure a job request that includes:
     - `template.id` (your uploaded template ID)
     - `template.composition` (e.g., `Final 2`)
     - `fonts` list that matches what your template needs
     - `assets`:
       - Text → `var_author` = `{{ $('Grab one book for review').item.json.author }}`
       - Text → `var_title` = `{{ $('Grab one book for review').item.json.title }}`
       - (Optional) Text layer for narration/subtitles using sanitized LLM output:
         - e.g., `var_script` = `{{ $('Sanitize text').item.json.message.content }}`
       - Function → `nx:genai` configured for Fal.ai and your desired model/settings
       - Data expression to set AE work area duration to the generated video duration
   - Connect: `Sanitize text` → `Initiate a render`

9) **Add Nexrender node to wait for completion**
   - Node: **Nexrender**
   - Name: `Get job`
   - Operation: **Get**
   - Job ID: `{{ $json.id }}`
   - Enable: **Wait Until Done**
   - Connect: `Initiate a render` → `Get job`

10) **(Optional) Add delivery/publishing nodes**
   - After `Get job`, add nodes to:
     - Download the render result
     - Upload to storage/CDN
     - Post to Slack, send email, or schedule social publishing

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Install the Nexrender n8n community node | https://github.com/nexrender/nexrender-n8n-node |
| Request Nexrender trial access | https://www.nexrender.com/get-trial |
| Fal.ai API key | https://fal.ai/ |
| OpenAI API keys | https://platform.openai.com/settings/organization/api-keys |
| Nexrender Cloud quickstart (upload template/fonts, configure secrets like **FAL_KEY**) | https://docs.nexrender.com/cloud/quickstart |
| Support email | support@nexrender.com |
| Video introduction | `@[youtube](EZHOHXu4PyA)` |
| Important implementation note: in the provided workflow, the OpenAI-generated voice lines are sanitized but not actually injected into the Nexrender job request body. If you want narration/subtitles driven by the LLM, add a text asset mapping to a dedicated AE layer. | Applies to the current JSON design |

