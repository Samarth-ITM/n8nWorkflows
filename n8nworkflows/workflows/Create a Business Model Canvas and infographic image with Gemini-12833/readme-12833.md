Create a Business Model Canvas and infographic image with Gemini

https://n8nworkflows.xyz/workflows/create-a-business-model-canvas-and-infographic-image-with-gemini-12833


# Create a Business Model Canvas and infographic image with Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:** Collects a user’s business idea through a 4-phase multi-page form, uses an LLM (via AWS Bedrock) to generate a structured **Business Model Canvas (BMC)** text, validates whether the LLM returned an error JSON, then uses **Google Gemini image generation** to render a **single infographic image** of the BMC and returns it as a downloadable/viewable binary link.

**Typical use cases**
- Lead magnet / intake form that returns a one-page BMC.
- Rapid business ideation support for founders, consultants, incubators.
- Automated generation of a presentation-ready canvas image.

### 1.1 Input Reception (Multi-step form)
Users complete 4 sequential form pages (Phase 1–4). Each page collects specific information.

### 1.2 Response Normalization
All phase responses are merged and reformatted into a structured text payload for the AI agent.

### 1.3 AI Canvas Generation + Validation
An AI agent prompts a Bedrock-connected chat model to either:
- Return **ERROR JSON** (if inputs are invalid), or
- Return a **9-block BMC** text.
A code node inspects the output and sets a boolean `isError`.

### 1.4 Conditional Routing + Infographic Generation
If `isError` is true, the workflow ends with an error completion page.
Otherwise Gemini generates an infographic image and the workflow returns a completion page containing a link to the image binary.

---

## 2. Block-by-Block Analysis

### Block 1 — Step1: Collect business idea details
**Overview:** Presents a 4-page interview form. Each form page submits to the next, culminating in a dataset used for canvas generation.

**Nodes involved:**
- Sticky Note (Step1)
- On form submission
- Form2
- Form3
- Form4

#### Sticky Note (Step1 - Collect business idea details)
- **Type/role:** Sticky Note (documentation)
- **Content:**  
  “Step1 - Collect business idea details / 4-page form collecting business idea details”

#### On form submission
- **Type/role:** `formTrigger` — workflow entry point, hosts the first form page.
- **Configuration highlights:**
  - Title: “Business Model Canvas Interview”
  - Ignore bots: enabled
  - Button label: “Next”
  - Fields (Phase 1 / “Grasping the Core”):
    - Target customer (required textarea)
    - Challenges/pain points (required textarea)
    - Current coping methods (required textarea)
    - Value proposition / transformation (required textarea)
    - Strengths & motivation (required textarea)
  - Includes an HTML heading as the first “field”
- **Connections:**
  - Output → **Form2**
- **Edge cases / failures:**
  - Users abandon mid-form: later phases won’t run.
  - Field naming: downstream code relies on reading this node’s `.item.json`; changes to field names affect prompt content.

#### Form2
- **Type/role:** `form` — second page in the multi-step form.
- **Configuration highlights:**
  - Title: “Phase 2: Customer Touchpoints (External Environment)”
  - Button label: “Next”
  - Fields: information sources, discovery, purchase/signup location, relationship type (checkbox, required), repeat/referral mechanisms.
- **Connections:**
  - Input ← On form submission
  - Output → **Form3**
- **Edge cases:**
  - Checkbox values come through as arrays; the AI prompt will include them as-is.

#### Form3
- **Type/role:** `form` — third page.
- **Configuration highlights:**
  - Title: “Phase 3: Means of Delivery (Internal Environment)”
  - Fields: essential activities (required), resources (checkbox required), missing/collaborators (optional).
- **Connections:**
  - Input ← Form2
  - Output → **Form4**

#### Form4
- **Type/role:** `form` — fourth page.
- **Configuration highlights:**
  - Title: “Phase 4: Money Flow”
  - Fields: revenue model (checkbox required), revenue growth focus (checkbox required), price range (required), cost structure (required).
- **Connections:**
  - Input ← Form3
  - Output → **Format Interview Responses**

---

### Block 2 — Step2: Structure responses into Business Model Canvas
**Overview:** Merges all phase answers, formats them into a phase-based structure, then calls an AI agent (connected to AWS Bedrock chat model) to generate the 9 BMC elements. Output is validated for an ERROR JSON pattern.

**Nodes involved:**
- Sticky Note1 (Step2)
- Format Interview Responses
- AWS Bedrock Chat Model
- AI Canvas Generator
- Validate Canvas Output

#### Sticky Note1
- **Type/role:** Sticky Note (documentation)
- **Content:**  
  “Step2 - Structure responses into Business Model Canvas … (Format → AI generation → Validate output format for error handling)”

#### Format Interview Responses
- **Type/role:** `code` — data shaping/normalization.
- **Configuration choices (interpreted):**
  - Reads `.item.json` from each form node by name:
    - `$('On form submission').item.json`
    - `$('Form2').item.json`
    - `$('Form3').item.json`
    - `$('Form4').item.json`
  - Merges all answers into `allData` (spread merge).
  - Creates `formatted`: an array of phases, each with `phase` label and `data: Object.entries(phaseX)`.
- **Outputs:**
  - `{ allData, formatted }`
- **Connections:**
  - Input ← Form4
  - Output → AI Canvas Generator
- **Key variables/structures:**
  - `formatted` is later mapped into a prompt with sections and Q/A pairs.
- **Edge cases / failures:**
  - If any upstream form node name changes, `$('<name>')` lookups will fail.
  - If form nodes don’t have an item (e.g., user didn’t reach that page), `.item.json` access can throw.
  - Includes meta fields like `submittedAt`, `formMode` which are filtered later in the agent prompt (not here).

#### AWS Bedrock Chat Model
- **Type/role:** LangChain chat model wrapper `lmChatAwsBedrock` — provides the LLM used by the agent.
- **Configuration choices:**
  - Model: `jp.anthropic.claude-sonnet-4-5-20250929-v1:0`
  - Model source: inference profile
- **Credentials:** AWS account credentials for Bedrock.
- **Connections:**
  - AI languageModel output → AI Canvas Generator (agent’s model input)
- **Edge cases / failures:**
  - Bedrock auth/permissions (model invocation denied).
  - Region/model availability mismatches.
  - Timeouts or token limits if users provide very long answers.

#### AI Canvas Generator
- **Type/role:** LangChain `agent` — generates BMC text (or an ERROR JSON) using the connected chat model.
- **Configuration choices (interpreted):**
  - **User text prompt** is built from `formatted` and rendered as:
    - Multiple sections `## <phase>` with `Q:` and `A:` pairs
    - Filters out questions named `submittedAt` and `formMode`
  - **System message** enforces a two-step behavior:
    1. Validate answers (relevance, completeness, coherence). If invalid → output **only** a specific JSON with `status: "ERROR"`, `reason`, `message`.
    2. If valid → produce the 9 BMC elements in a prescribed labeled format.
- **Connections:**
  - Main input ← Format Interview Responses
  - Model input ← AWS Bedrock Chat Model (via `ai_languageModel` connection)
  - Output → Validate Canvas Output
- **Edge cases / failures:**
  - The agent output is expected as text in `item.json.output`. If the node returns a different shape (version/provider differences), downstream validation may break.
  - The system prompt asks for “Cost Structure (CS)” but also uses “Customer Segments (CS)”; abbreviation collision can confuse humans (the model generally still labels full headings).
  - If the model outputs JSON but with different spacing/escaping, simplistic detection may fail.

#### Validate Canvas Output
- **Type/role:** `code` — parses AI output to decide error vs success.
- **Logic (interpreted):**
  - Reads: `const output = $('AI Canvas Generator').item.json.output;`
  - If `output` contains `"status": "ERROR"` (two string patterns), then:
    - Extracts `"message"` via regex `/"message":\s*"([^"]+)"/`
    - Returns `{ isError: true, errorMessage }`
  - Else returns `{ isError: false, canvas: output }`
- **Connections:**
  - Input ← AI Canvas Generator
  - Output → If_is_error
- **Edge cases / failures:**
  - If the agent returns structured JSON (object) instead of a string, `.includes()` will throw.
  - Regex extraction fails if message contains escaped quotes or multiline JSON; fallback message is used.
  - False negatives if the model returns error JSON without exact `"status": "ERROR"` substring (e.g., lowercase, extra formatting).

---

### Block 3 — Step3: Generate infographic with Gemini + show result
**Overview:** Routes execution based on `isError`. If error → show a completion page with an error message. If success → generate an image using Gemini and return a completion page with a binary link.

**Nodes involved:**
- Sticky Note2 (Step3)
- If_is_error
- Generate an image
- Completed
- Error End

#### Sticky Note2
- **Type/role:** Sticky Note (documentation)
- **Content:**  
  “Step3 - Generate infographic with Gemini / Checks validation result, then generates canvas image or displays error message”

#### If_is_error
- **Type/role:** `if` — branching control.
- **Condition:**
  - `{{ $json.isError }}` equals `true` (boolean strict)
- **Connections:**
  - Input ← Validate Canvas Output
  - True output → Error End
  - False output → Generate an image
- **Edge cases / failures:**
  - If `isError` missing or not boolean, strict validation may route unexpectedly.

#### Generate an image
- **Type/role:** Google Gemini node `googleGemini` — image generation.
- **Configuration choices:**
  - Resource: `image`
  - Model: `models/gemini-3-pro-image-preview` (as selected in the node)
  - Prompt: combines:
    - `{{ $json.canvas }}` (the canvas text)
    - Detailed layout constraints for a standard 9-block BMC grid
    - Style rules (icons, headers, borders, bullets, avoid overflow)
- **Credentials:** Google Gemini / PaLM API credential.
- **Connections:**
  - Input ← If_is_error (false branch)
  - Output → Completed
- **Edge cases / failures:**
  - Image generation model availability/quotas.
  - Prompt too long (if canvas is verbose).
  - Output may not respect strict layout requirements (model limitation); could produce illegible text or overflow despite instructions.
  - Binary data must be produced for the completion node link to work.

#### Completed
- **Type/role:** `form` (completion operation) — final success page; returns a binary link to the generated image.
- **Configuration choices:**
  - Operation: `completion`
  - Respond with: `returnBinary`
  - Completion message builds an internal n8n binary download/view link:
    - `/rest/binary-data?id={{ encodeURIComponent($('Generate an image').first().binary.data.id) }}&action=view&fileName={{ $('Generate an image').first().binary.data.fileName }}&mimeType={{ encodeURIComponent($('Generate an image').first().binary.data.mimeType) }}`
- **Connections:**
  - Input ← Generate an image
- **Edge cases / failures:**
  - If the Gemini node output path differs (no `binary.data.*`), the link renders broken.
  - The internal `/rest/binary-data` route depends on n8n hosting/base URL and auth settings.

#### Error End
- **Type/role:** `form` (completion operation) — final error page.
- **Configuration choices:**
  - Operation: `completion`
  - Message shows `{{ $json.errorMessage }}`
- **Connections:**
  - Input ← If_is_error (true branch)
- **Edge cases:**
  - If `errorMessage` missing, the page will show blank.

---

### Block 4 — Main note (global documentation)
**Overview:** Embedded documentation describing the overall goal and setup steps.

**Nodes involved:**
- Sticky Note3 (Main)

#### Sticky Note3
- **Type/role:** Sticky Note (documentation)
- **Key content:**
  - Explains the 3-step flow (form → AI canvas → Gemini image)
  - Setup steps:
    - Connect LLM provider credentials (AWS Bedrock, OpenAI, Anthropic, Azure OpenAI, Google Vertex AI, Ollama)
    - Set LLM credentials in “AWS Bedrock Chat Model” (or replace provider)
    - Set Gemini API credentials in “Generate an image”
    - Activate workflow and share production URL

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | n8n-nodes-base.formTrigger | Entry point; Phase 1 form collection | — | Form2 | Step1 - Collect business idea details / 4-page form collecting business idea details |
| Form2 | n8n-nodes-base.form | Phase 2 form page | On form submission | Form3 | Step1 - Collect business idea details / 4-page form collecting business idea details |
| Form3 | n8n-nodes-base.form | Phase 3 form page | Form2 | Form4 | Step1 - Collect business idea details / 4-page form collecting business idea details |
| Form4 | n8n-nodes-base.form | Phase 4 form page | Form3 | Format Interview Responses | Step1 - Collect business idea details / 4-page form collecting business idea details |
| Format Interview Responses | n8n-nodes-base.code | Merge/format all form answers | Form4 | AI Canvas Generator | Step2 - Structure responses into Business Model Canvas / Formats input and generates 9 canvas elements using AI. (Format → AI generation → Validate output format for error handling) |
| AWS Bedrock Chat Model | @n8n/n8n-nodes-langchain.lmChatAwsBedrock | Provides LLM for canvas generation | — | AI Canvas Generator (ai_languageModel) | Step2 - Structure responses into Business Model Canvas / Formats input and generates 9 canvas elements using AI. (Format → AI generation → Validate output format for error handling) |
| AI Canvas Generator | @n8n/n8n-nodes-langchain.agent | Generates BMC text or error JSON | Format Interview Responses + AWS Bedrock Chat Model | Validate Canvas Output | Step2 - Structure responses into Business Model Canvas / Formats input and generates 9 canvas elements using AI. (Format → AI generation → Validate output format for error handling) |
| Validate Canvas Output | n8n-nodes-base.code | Detects error JSON vs canvas text | AI Canvas Generator | If_is_error | Step2 - Structure responses into Business Model Canvas / Formats input and generates 9 canvas elements using AI. (Format → AI generation → Validate output format for error handling) |
| If_is_error | n8n-nodes-base.if | Branch: error vs image generation | Validate Canvas Output | Error End; Generate an image | Step3 - Generate infographic with Gemini / Checks validation result, then generates canvas image or displays error message |
| Generate an image | @n8n/n8n-nodes-langchain.googleGemini | Generates BMC infographic image | If_is_error (false) | Completed | Step3 - Generate infographic with Gemini / Checks validation result, then generates canvas image or displays error message |
| Completed | n8n-nodes-base.form | Success completion; returns binary link | Generate an image | — | Step3 - Generate infographic with Gemini / Checks validation result, then generates canvas image or displays error message |
| Error End | n8n-nodes-base.form | Error completion; displays error message | If_is_error (true) | — | Step3 - Generate infographic with Gemini / Checks validation result, then generates canvas image or displays error message |
| Sticky Note | n8n-nodes-base.stickyNote | Comment | — | — | |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment | — | — | |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment | — | — | |
| Sticky Note3 | n8n-nodes-base.stickyNote | Comment | — | — | |

---

## 4. Reproducing the Workflow from Scratch

1. **Create node: “On form submission”**
   - Type: **Form Trigger**
   - Set **Form Title**: “Business Model Canvas Interview”
   - Options: enable **Ignore Bots**, set button label “Next”
   - Add fields for Phase 1:
     - HTML heading (optional)
     - 5 required textareas: target customer, pain points, coping methods, value proposition, strengths/motivation
   - Connect **On form submission → Form2**

2. **Create node: “Form2”**
   - Type: **Form**
   - Title: “Phase 2: Customer Touchpoints (External Environment)”
   - Add required textareas for info sources, discovery, purchase/signup location
   - Add required checkbox for relationship type (+ options)
   - Add optional textarea for repeat/referral
   - Connect **Form2 → Form3**

3. **Create node: “Form3”**
   - Type: **Form**
   - Title: “Phase 3: Means of Delivery (Internal Environment)”
   - Add required textarea essential activities
   - Add required checkbox resources (+ options)
   - Add optional textarea missing/collaborators
   - Connect **Form3 → Form4**

4. **Create node: “Form4”**
   - Type: **Form**
   - Title: “Phase 4: Money Flow”
   - Add required checkbox revenue model (+ options)
   - Add required checkbox revenue growth focus (+ options)
   - Add required textarea price range
   - Add required textarea cost structure
   - Connect **Form4 → Format Interview Responses**

5. **Create node: “Format Interview Responses”**
   - Type: **Code**
   - Paste logic that:
     - Reads each form node’s `.item.json` by node name
     - Produces `{ allData, formatted }` where `formatted` is an array of phases with `Object.entries(...)`
   - Connect **Format Interview Responses → AI Canvas Generator**

6. **Create node: “AWS Bedrock Chat Model”**
   - Type: **LangChain → AWS Bedrock Chat Model**
   - Select your Bedrock model (this workflow uses: `jp.anthropic.claude-sonnet-4-5-20250929-v1:0`)
   - **Credentials:** configure AWS credentials with Bedrock invoke permissions.
   - Connect the node’s **AI Language Model** output to **AI Canvas Generator** (language model input).

   *Alternative:* Replace this node with any other n8n LangChain chat model node (OpenAI, Azure OpenAI, Anthropic, Vertex, Ollama). Ensure the agent still outputs text in a field compatible with the next step.

7. **Create node: “AI Canvas Generator”**
   - Type: **LangChain Agent**
   - Set **System Message** to:
     - Validate inputs; if invalid, return only a strict ERROR JSON with `status: "ERROR"`, `reason`, `message`
     - If valid, output the 9 BMC elements with clear headings
   - Set **User Text** to render the `formatted` array into multi-phase Q/A text, filtering out `submittedAt` and `formMode`.
   - Connect **AI Canvas Generator → Validate Canvas Output**

8. **Create node: “Validate Canvas Output”**
   - Type: **Code**
   - Read `$('AI Canvas Generator').item.json.output`
   - If it contains `"status": "ERROR"` → return `{ isError: true, errorMessage }`
   - Else return `{ isError: false, canvas: output }`
   - Connect **Validate Canvas Output → If_is_error**

9. **Create node: “If_is_error”**
   - Type: **IF**
   - Condition: boolean equals
     - Left: `{{ $json.isError }}`
     - Right: `true`
   - True branch → **Error End**
   - False branch → **Generate an image**

10. **Create node: “Generate an image”**
   - Type: **Google Gemini** (image generation resource)
   - Model: select an image-capable Gemini model (this workflow uses `models/gemini-3-pro-image-preview`)
   - Prompt:
     - Include `{{ $json.canvas }}`
     - Add strict instructions for BMC 9-block grid layout, icons, colors, non-overflow text, bullet points
   - **Credentials:** configure Google Gemini/PaLM API key/credential in n8n.
   - Connect **Generate an image → Completed**

11. **Create node: “Completed”**
   - Type: **Form** with operation **Completion**
   - Set **Respond With**: `returnBinary`
   - Completion message: include an `<a>` tag that references the binary created by “Generate an image” using:
     - `$('Generate an image').first().binary.data.id`, `.fileName`, `.mimeType`
     - Build URL `/rest/binary-data?...&action=view...`
   - No further connections required.

12. **Create node: “Error End”**
   - Type: **Form** with operation **Completion**
   - Completion message should display: `{{ $json.errorMessage }}`

13. **(Optional) Add Sticky Notes**
   - Add documentation notes for Step1/Step2/Step3 and the global “Main” explanation.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Transform your business idea into a professional Business Model Canvas infographic. Flow: multi-page form → AI structures into 9 BMC elements → Gemini generates infographic image. | Sticky Note3 (Main) |
| Setup: connect LLM credentials (AWS Bedrock, OpenAI, Anthropic, Azure OpenAI, Google Vertex AI, or Ollama) for canvas generation; connect Google Gemini API for image generation; activate workflow and share production URL. | Sticky Note3 (Main) |
| The success page uses an internal n8n binary URL endpoint `/rest/binary-data` to view the generated image. | Completed node completion message (internal link) |