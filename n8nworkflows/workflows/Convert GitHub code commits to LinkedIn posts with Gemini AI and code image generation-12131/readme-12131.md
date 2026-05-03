Convert GitHub code commits to LinkedIn posts with Gemini AI and code image generation

https://n8nworkflows.xyz/workflows/convert-github-code-commits-to-linkedin-posts-with-gemini-ai-and-code-image-generation-12131


# Convert GitHub code commits to LinkedIn posts with Gemini AI and code image generation

## 1. Workflow Overview

**Purpose:** This workflow monitors GitHub `push` events, extracts changed code files, uses **Gemini (via OpenRouter)** to generate a short technical LinkedIn post plus a highlighted code snippet, renders that snippet into a “Mac window” style code image (via **HCTI**), uploads the image to a GitHub repository for hosting, then publishes the post to **LinkedIn** with the image attached.

**Target use cases:**
- Auto-promote engineering work (commits) as short, consistent LinkedIn posts.
- Create visually attractive code snippet images for social sharing.
- Maintain hosted social assets in a dedicated GitHub “image repo”.

### 1.1 Monitor & Fetch (GitHub → changed files → raw file content)
Triggered by a GitHub push webhook, extracts added/modified file paths, downloads each file, and converts binary to usable text and/or image payload.

### 1.2 AI Content Creation (Code → structured post + snippet)
Feeds extracted code text to a LangChain Agent node, backed by OpenRouter Gemini, enforcing a strict JSON structure with a structured output parser.

### 1.3 Image Generation (Snippet → HTML → PNG)
Builds HTML/CSS for a code “window”, calculates dimensions, then calls HCTI to render an image and downloads the resulting file.

### 1.4 Store & Publish (Upload image → merge → LinkedIn post)
Uploads the generated image back to a GitHub repository, merges post text + image metadata, and creates a LinkedIn post with image media.

---

## 2. Block-by-Block Analysis

### Block 1 — Monitor & Fetch
**Overview:** Listens to GitHub `push` events, extracts changed file paths, downloads each file, and prepares both text and binary representations for downstream AI and upload steps.

**Nodes involved:**
- Main Info (Sticky Note)
- Section 1 (Sticky Note)
- Github Trigger1
- Extract Modified Files
- GitHub File Download
- Extract from File
- Extract from File1

#### Main Info (Sticky Note)
- **Type/role:** Sticky Note (documentation only).
- **Key content:** Explains end-to-end behavior and setup checklist (GitHub/LinkedIn/OpenRouter/HCTI credentials, owner/repo fields, LinkedIn URN).
- **Connections:** None.
- **Failure modes:** None (non-executable).

#### Section 1 (Sticky Note)
- **Type/role:** Sticky Note describing “Monitor & Fetch”.
- **Connections:** None.

#### Github Trigger1
- **Type/role:** `githubTrigger` — entry point webhook for GitHub events.
- **Configuration (interpreted):**
  - **Owner:** `your-github-username` (must be replaced)
  - **Repository:** `your-source-repo-name` (must be replaced)
  - **Events:** `push`
- **Output:** Emits the GitHub webhook payload (notably `body.head_commit.*`).
- **Connections:** → `Extract Modified Files`
- **Credentials:** GitHub API credential (token/OAuth depending on n8n setup).
- **Edge cases / failures:**
  - Missing/invalid credentials or insufficient repo permissions.
  - Push events without `head_commit` (some GitHub events or unusual payloads).
  - Multiple commits in one push: this workflow uses **only** `head_commit` for file lists.

#### Extract Modified Files
- **Type/role:** Code node — normalizes the webhook payload into one n8n item per changed file.
- **Notes (from node):** “Needed to extract the path of different types of changes in Github, be it modified, added, etc.”
- **Key logic:**
  - Reads `const headCommit = $json.body.head_commit;`
  - Collects `headCommit.added` and `headCommit.modified`
  - Returns `[{ json: { filePath } }, ...]`
  - If no changes: `return []` (graceful stop)
- **Connections:** ← `Github Trigger1` ; → `GitHub File Download`
- **Edge cases / failures:**
  - If `body.head_commit` is undefined, code throws (TypeError). A defensive check is recommended.
  - Deleted/removed files are ignored (`removed` array not included).
  - Large number of changed files will create many items (rate limits downstream).

#### GitHub File Download
- **Type/role:** GitHub node — downloads file content by path.
- **Configuration:**
  - Resource: **file**
  - Operation: **get**
  - `filePath`: `={{ $json.filePath }}`
  - Owner/Repo: same placeholders as trigger (must be updated)
- **Output:** File content in binary (and metadata) as returned by the GitHub node.
- **Connections:** ← `Extract Modified Files` ; → `Extract from File` and → `Extract from File1` (fan-out)
- **Credentials:** GitHub API credential.
- **Edge cases / failures:**
  - Path may refer to non-text/binary files; downstream text extraction may fail.
  - GitHub “get file” behavior depends on node implementation; for large files, API limitations may apply.
  - If the push includes file renames or submodule pointers, fetch may fail.

#### Extract from File
- **Type/role:** Extract From File node — converts downloaded file to **text** for AI analysis.
- **Configuration:** Operation: `text`
- **Input/Output:**
  - Input: binary file from `GitHub File Download`
  - Output: extracted text in a field that is later referenced as `data` by the agent input (`{{ $json.data }}`)
- **Connections:** ← `GitHub File Download` ; → `LinkedIn Content Creator`
- **Edge cases / failures:**
  - Non-text files or unknown encodings can produce empty/garbled text.
  - Very large files can bloat prompts and exceed model limits.

#### Extract from File1
- **Type/role:** Extract From File node — converts binary to a property for upload.
- **Configuration:** Operation: `binaryToPropery` (node label suggests converting binary to JSON property; exact behavior depends on node version)
- **Connections:** ← `GitHub File Download` ; → `Upload to Image Repo`
- **Edge cases / failures:**
  - The output expected by `Upload to Image Repo` is `{{$json.data}}`. If this node outputs a different property name (varies by n8n version/config), the upload will fail.
  - If the downloaded content is not the generated image (and it isn’t—this is still the **source code** file), the upload step may be conceptually miswired (see Block 4 notes).

---

### Block 2 — AI Content Creation
**Overview:** Uses a LangChain Agent with Gemini (OpenRouter) to analyze extracted code text and output structured JSON containing: post title, short post body, hashtags string, and a code snippet + language.

**Nodes involved:**
- Section 2 (Sticky Note)
- OpenRouter Chat Model
- Structured Output Parser
- LinkedIn Content Creator
- Edit Fields

#### Section 2 (Sticky Note)
- **Type/role:** Sticky Note describing AI content generation.

#### OpenRouter Chat Model
- **Type/role:** LangChain chat model connector (`lmChatOpenRouter`).
- **Configuration:**
  - Model: `google/gemini-2.5-flash`
  - Response format option: `json_object` (encourages JSON-compliant output)
- **Connections:** Provides **AI language model** input to `LinkedIn Content Creator` (special `ai_languageModel` connection).
- **Credentials:** OpenRouter API credential.
- **Edge cases / failures:**
  - OpenRouter auth issues, model availability, rate limits.
  - “json_object” is best-effort; malformed JSON can still happen (parser mitigates this).

#### Structured Output Parser
- **Type/role:** LangChain structured output parser.
- **Configuration:** Manual JSON schema requiring:
  - `post_title` (string)
  - `post_content` (string)
  - `code_snippet` (string)
  - `code_language` (string)
  - `hashtags` (string)
  - `character_count` (integer)
- **Connections:** Supplies `ai_outputParser` to `LinkedIn Content Creator`.
- **Edge cases / failures:**
  - If the model output doesn’t match schema (missing fields, wrong types), the agent will fail.
  - `character_count` returned as string instead of integer is a common mismatch.

#### LinkedIn Content Creator
- **Type/role:** LangChain Agent node — orchestrates prompt + model + structured parsing.
- **Configuration:**
  - Prompt input text: `=Data\n{{ $json.data }}`
    - Expects `data` field from `Extract from File` (text of source code).
  - System message: LinkedIn content strategist; constraints:
    - Post within **400 characters**
    - No code included in post body
    - Returns JSON with specific fields
    - Hashtags must be a **single string** of space-separated hashtags
  - Output parser enabled (`hasOutputParser: true`)
- **Connections:** ← `Extract from File` ; → `Edit Fields`
- **Edge cases / failures:**
  - If extracted code text is empty, the model may produce generic content.
  - If the code is huge, token limits may truncate or error.
  - If the agent returns content > 400 characters, it violates the instruction but may still happen.

#### Edit Fields
- **Type/role:** Set node — flattens `output.*` fields into top-level fields for simpler downstream usage.
- **Configuration:** Assignments:
  - `post_title = {{$json.output.post_title}}`
  - `post_content = {{$json.output.post_content}}`
  - `code_snippet = {{$json.output.code_snippet}}`
  - `code_language = {{$json.output.code_language}}`
  - `hashtags = {{$json.output.hashtags}}`
  - `character_count = {{$json.output.character_count}}` (stored as number)
- **Connections:** ← `LinkedIn Content Creator` ; → `Create Code HTML`
- **Edge cases / failures:**
  - If the agent output is not under `output` (node/version differences), these expressions resolve to null.
  - `character_count` type mismatch can break numeric assignment.

---

### Block 3 — Image Generation
**Overview:** Converts the selected snippet into syntax-highlighted HTML with calculated viewport size, then uses HCTI to render it to an image and downloads the image file.

**Nodes involved:**
- Section 3 (Sticky Note)
- Create Code HTML
- Generate Code Image
- GET the genarted Image

#### Section 3 (Sticky Note)
- **Type/role:** Sticky Note describing image generation.

#### Create Code HTML
- **Type/role:** Code node — builds HTML/CSS and computes image dimensions.
- **Key variables/logic:**
  - Reads:
    - `codeSnippet = $json.code_snippet || ''`
    - `language = $json.code_language || 'javascript'`
  - Computes:
    - `estimatedWidth`, `estimatedHeight` based on line count and max line length
    - Uses `Math.ceil` to avoid non-integer viewport values (explicitly to prevent API errors)
  - Generates HTML:
    - Background gradient
    - “Mac dots” header
    - PrismJS theme `prism-tomorrow`
    - Loads Prism component script for `prism-${language}.min.js`
    - Escapes `<` and `>` in code
  - Returns:
    - `html`
    - `estimatedWidth`, `estimatedHeight`
- **Connections:** ← `Edit Fields` ; → `Generate Code Image`
- **Edge cases / failures:**
  - If `language` is not supported by Prism component CDN path, the script 404s and highlighting may fail (image still renders).
  - Very long lines can push width; it clamps to max 1200 content width, but viewport sent later clamps to 1920.
  - Snippet containing `&` isn’t escaped (only `<` and `>`), potentially affecting HTML rendering.

#### Generate Code Image
- **Type/role:** HTTP Request node — calls HCTI image generation API.
- **Configuration:**
  - POST `https://hcti.io/v1/image`
  - Timeout: 30s
  - Response format: JSON
  - Sends body params:
    - `html = {{$json.html}}`
    - `google_fonts = Fira Code`
    - `viewport_width = {{ Math.min($json.estimatedWidth || 800, 1920) }}`
    - `viewport_height = {{ Math.min($json.estimatedHeight || 400, 1080) }}`
    - `device_scale = 2`
  - Headers: `Content-Type: application/json`
  - Auth: set to `genericCredentialType`, using `httpBasicAuth` credential (HCTI typically uses Basic Auth with API key/secret)
- **Connections:** ← `Create Code HTML` ; → `GET the genarted Image`
- **Edge cases / failures:**
  - Wrong HCTI credentials → 401.
  - HCTI can reject large HTML or too-large viewport.
  - If HCTI returns error JSON without `url`, next step fails.

#### GET the genarted Image
- **Type/role:** HTTP Request node — downloads the rendered image file.
- **Configuration:**
  - URL: `={{ $json.url }}`
  - Response format: `file` (binary)
  - `alwaysOutputData: true` (continues even if some error conditions happen)
- **Connections:** ← `Generate Code Image` ; → `Merge` (input index 1)
- **Edge cases / failures:**
  - If `$json.url` missing/empty, request fails.
  - CDN/download transient failures (timeouts, 403) lead to missing binary for upload/LinkedIn.

---

### Block 4 — Store & Publish
**Overview:** Uploads an image file to a GitHub “image repo”, merges the GitHub upload result with the generated image payload, then posts to LinkedIn with text + image.

**Nodes involved:**
- Section 4 (Sticky Note)
- Upload to Image Repo
- Merge
- Post to LinkedIn

#### Section 4 (Sticky Note)
- **Type/role:** Sticky Note describing upload + publishing.

#### Upload to Image Repo
- **Type/role:** GitHub node — creates/updates a file in a separate repository intended for image hosting.
- **Configuration:**
  - Resource: **file**
  - Operation: (implied) **create/update** via “fileContent” + “commitMessage” fields (the node UI typically maps this to “create or update file”)
  - Owner: `your-github-username`
  - Repository: `your-image-storage-repo`
  - `filePath`: `={{ $('GitHub File Download').item.json.filePath }}`
  - `fileContent`: `={{ $json.data }}`
  - Commit message: `Upload generated image for {{ $('GitHub File Download').item.json.filePath }}`
- **Connections:** ← `Extract from File1` ; → `Merge` (input index 0)
- **Edge cases / failures (important):**
  - **Wiring/data mismatch risk:** This node is fed by `Extract from File1`, which originates from **GitHub File Download** (source code file), not from **GET the generated Image** (the actual PNG). As configured, it may upload the *source code content* into the image repo, not the generated image.
  - `filePath` uses the original code path; if it contains folders or extensions (e.g., `.bicep`), you likely want to transform it to something like `images/<name>.png`.
  - GitHub file content for binary needs correct base64 handling; ensure `fileContent` is what the GitHub node expects (often base64 without metadata).
  - Permission issues on target repo.

#### Merge
- **Type/role:** Merge node — combines data from two branches to assemble everything needed for LinkedIn.
- **Configuration:**
  - Mode: `combine`
  - Combine by: `combineAll`
  - Inputs:
    - Input 0: from `Upload to Image Repo`
    - Input 1: from `GET the genarted Image`
- **Connections:** → `Post to LinkedIn`
- **Edge cases / failures:**
  - If one branch produces zero items (e.g., upload failed), combineAll may produce unexpected results or none.
  - Field collisions: later items may overwrite keys depending on merge behavior.

#### Post to LinkedIn
- **Type/role:** LinkedIn node — posts text + image.
- **Configuration:**
  - `person`: `your-linkedin-urn` (must be replaced with the author URN)
  - Visibility: PUBLIC
  - `shareMediaCategory`: IMAGE
  - Text body expression:
    - Title/content/hashtags from `LinkedIn Content Creator`:
      - `$('LinkedIn Content Creator').item.json.output.post_title`
      - `$('LinkedIn Content Creator').item.json.output.post_content`
      - `$('LinkedIn Content Creator').item.json.output.hashtags`
    - Adds “Link to Github: {{ $json.content._links.html }}”
      - This assumes the merged item includes a `content._links.html` field (commonly returned by GitHub file create/update responses).
- **Connections:** ← `Merge`
- **Credentials:** LinkedIn OAuth2 credential.
- **Edge cases / failures:**
  - LinkedIn requires correct permissions/scopes for posting (UGC/share permissions).
  - Media upload: LinkedIn node typically needs the binary image present in the item; this workflow’s merge must ensure the binary from `GET the genarted Image` is the active binary property the node expects.
  - If `content._links.html` doesn’t exist (upload failed or response differs), link expression becomes blank.
  - Post length constraint (400 chars) isn’t enforced at node level; LinkedIn limits may differ.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Info | Sticky Note | Workflow description & setup checklist |  |  | ## 🚀 Github Code to LinkedIn Publisher / Setup steps include credentials, owner/repo fields, LinkedIn URN, HCTI account |
| Section 1 | Sticky Note | Describes Monitor & Fetch block |  |  | ### 1. Monitor & Fetch Watches for `push` events, extracts modified files, and downloads the raw code content. |
| Github Trigger1 | GitHub Trigger | Receives `push` webhook events |  | Extract Modified Files | ### 1. Monitor & Fetch Watches for `push` events, extracts modified files, and downloads the raw code content. |
| Extract Modified Files | Code | Builds items for added/modified file paths | Github Trigger1 | GitHub File Download | ### 1. Monitor & Fetch Watches for `push` events, extracts modified files, and downloads the raw code content. |
| GitHub File Download | GitHub | Downloads each changed file by path | Extract Modified Files | Extract from File; Extract from File1 | ### 1. Monitor & Fetch Watches for `push` events, extracts modified files, and downloads the raw code content. |
| Extract from File | Extract From File | Converts downloaded file to text (`data`) | GitHub File Download | LinkedIn Content Creator | ### 1. Monitor & Fetch Watches for `push` events, extracts modified files, and downloads the raw code content. |
| Extract from File1 | Extract From File | Converts binary to a property for upload | GitHub File Download | Upload to Image Repo | ### 4. Store & Publish Uploads the generated image to your repo for hosting, then combines text + image for the final LinkedIn post. |
| Section 2 | Sticky Note | Describes AI content creation |  |  | ### 2. AI Content Creation Uses an LLM to analyze the code, write a LinkedIn post, and select the best snippet. |
| OpenRouter Chat Model | LangChain Chat Model (OpenRouter) | Provides Gemini model to agent |  | LinkedIn Content Creator (ai_languageModel) | ### 2. AI Content Creation Uses an LLM to analyze the code, write a LinkedIn post, and select the best snippet. |
| Structured Output Parser | LangChain Output Parser (Structured) | Enforces JSON schema for agent output |  | LinkedIn Content Creator (ai_outputParser) | ### 2. AI Content Creation Uses an LLM to analyze the code, write a LinkedIn post, and select the best snippet. |
| LinkedIn Content Creator | LangChain Agent | Generates post + snippet as structured JSON | Extract from File | Edit Fields | ### 2. AI Content Creation Uses an LLM to analyze the code, write a LinkedIn post, and select the best snippet. |
| Edit Fields | Set | Flattens `output.*` fields | LinkedIn Content Creator | Create Code HTML | ### 2. AI Content Creation Uses an LLM to analyze the code, write a LinkedIn post, and select the best snippet. |
| Section 3 | Sticky Note | Describes image generation |  |  | ### 3. Image Generation Creates HTML/CSS for a "pretty" code window and converts it to an image via API. |
| Create Code HTML | Code | Builds HTML/CSS and viewport sizing | Edit Fields | Generate Code Image | ### 3. Image Generation Creates HTML/CSS for a "pretty" code window and converts it to an image via API. |
| Generate Code Image | HTTP Request | Calls HCTI to render HTML → image | Create Code HTML | GET the genarted Image | ### 3. Image Generation Creates HTML/CSS for a "pretty" code window and converts it to an image via API. |
| GET the genarted Image | HTTP Request | Downloads rendered image as binary | Generate Code Image | Merge | ### 3. Image Generation Creates HTML/CSS for a "pretty" code window and converts it to an image via API. |
| Section 4 | Sticky Note | Describes storing and publishing |  |  | ### 4. Store & Publish Uploads the generated image to your repo for hosting, then combines text + image for the final LinkedIn post. |
| Upload to Image Repo | GitHub | Uploads a file to image hosting repo | Extract from File1 | Merge | ### 4. Store & Publish Uploads the generated image to your repo for hosting, then combines text + image for the final LinkedIn post. |
| Merge | Merge | Combines upload response + image binary | Upload to Image Repo; GET the genarted Image | Post to LinkedIn | ### 4. Store & Publish Uploads the generated image to your repo for hosting, then combines text + image for the final LinkedIn post. |
| Post to LinkedIn | LinkedIn | Publishes LinkedIn post with image | Merge |  | ### 4. Store & Publish Uploads the generated image to your repo for hosting, then combines text + image for the final LinkedIn post. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: `Github Code to LinkedIn Publisher`
   - Keep it inactive until credentials and repos are configured.

2. **Add Sticky Notes (optional but recommended)**
   - Add four section notes: “Monitor & Fetch”, “AI Content Creation”, “Image Generation”, “Store & Publish”.
   - Add a main note listing required credentials and the fields you must customize.

3. **Add “GitHub Trigger” node**
   - Node: **GitHub Trigger**
   - Events: `push`
   - Set **Owner** and **Repository** to your source code repo.
   - Credentials: **GitHub API** (token/OAuth with webhook permissions).
   - Connect to next step.

4. **Add “Extract Modified Files” (Code node)**
   - Node: **Code**
   - Paste logic to read `body.head_commit.added` and `body.head_commit.modified`, emit items with `json.filePath`.
   - Connect: `GitHub Trigger` → `Extract Modified Files`.

5. **Add “GitHub File Download” (GitHub node)**
   - Node: **GitHub**
   - Resource: **File**
   - Operation: **Get**
   - Owner/Repository: same as source repo
   - `filePath`: `={{ $json.filePath }}`
   - Connect: `Extract Modified Files` → `GitHub File Download`.

6. **Add “Extract from File” (text)**
   - Node: **Extract From File**
   - Operation: `text`
   - Connect: `GitHub File Download` → `Extract from File`.

7. **Add AI model connector: “OpenRouter Chat Model”**
   - Node: **OpenRouter Chat Model** (LangChain)
   - Model: `google/gemini-2.5-flash`
   - Options: Response format `json_object`
   - Credentials: **OpenRouter API** (API key).
   - This node connects via the **AI Language Model** connection type to the agent.

8. **Add “Structured Output Parser”**
   - Node: **Structured Output Parser** (LangChain)
   - Schema: manual JSON schema with fields:
     - `post_title` string, `post_content` string, `code_snippet` string, `code_language` string, `hashtags` string, `character_count` integer
   - This connects via the **AI Output Parser** connection type to the agent.

9. **Add “LinkedIn Content Creator” (LangChain Agent)**
   - Node: **Agent**
   - Prompt (input): `Data\n{{ $json.data }}`
   - System message: copy the constraints (hook, explain value, CTA, ≤400 chars, no code in post, return JSON with specified fields; hashtags as space-separated string).
   - Enable structured parsing.
   - Connect:
     - Main: `Extract from File` → `LinkedIn Content Creator`
     - AI Language Model: `OpenRouter Chat Model` → `LinkedIn Content Creator`
     - AI Output Parser: `Structured Output Parser` → `LinkedIn Content Creator`

10. **Add “Edit Fields” (Set node)**
    - Node: **Set**
    - Map from the agent’s structured output into top-level fields:
      - `post_title`, `post_content`, `code_snippet`, `code_language`, `hashtags`, `character_count`
    - Connect: `LinkedIn Content Creator` → `Edit Fields`.

11. **Add “Create Code HTML” (Code node)**
    - Node: **Code**
    - Use code that:
      - reads `code_snippet` and `code_language`
      - calculates `estimatedWidth/estimatedHeight` (integers)
      - builds Prism-based HTML with “Mac window” styling
      - outputs `html`, `estimatedWidth`, `estimatedHeight`
    - Connect: `Edit Fields` → `Create Code HTML`.

12. **Add “Generate Code Image” (HTTP Request)**
    - Node: **HTTP Request**
    - Method: POST
    - URL: `https://hcti.io/v1/image`
    - Response: JSON
    - Timeout: ~30s
    - Body params: `html`, `google_fonts`, `viewport_width`, `viewport_height`, `device_scale`
    - Header: `Content-Type: application/json`
    - Auth: **HTTP Basic Auth** (HCTI credentials)
    - Connect: `Create Code HTML` → `Generate Code Image`.

13. **Add “GET the generated Image” (HTTP Request)**
    - Node: **HTTP Request**
    - Method: GET
    - URL: `={{ $json.url }}`
    - Response: `file` (binary)
    - Connect: `Generate Code Image` → `GET the generated Image`.

14. **Add “Upload to Image Repo” (GitHub)**
    - Node: **GitHub**
    - Target: a dedicated repo (e.g., `your-image-storage-repo`)
    - Operation: create/update file (as supported by GitHub node)
    - **Important configuration choices to decide:**
      - `filePath`: choose a deterministic image path, e.g. `images/{{ $json.filePath }}.png` (recommended), rather than reusing source path.
      - `fileContent`: must be the **downloaded image binary** converted to what the GitHub node expects (often base64). In the provided workflow, this is likely miswired.
    - Credentials: GitHub API with write access to image repo.

15. **Add “Merge” node**
    - Node: **Merge**
    - Mode: `combine`
    - Combine by: `combineAll`
    - Connect:
      - `Upload to Image Repo` → `Merge` input 0
      - `GET the generated Image` → `Merge` input 1

16. **Add “Post to LinkedIn” node**
    - Node: **LinkedIn**
    - Person URN: set to your `urn:li:person:...`
    - Visibility: PUBLIC
    - Share media category: IMAGE
    - Text: combine title, content, hashtags; optionally include GitHub link from upload response.
    - Credentials: **LinkedIn OAuth2** with posting permissions.
    - Connect: `Merge` → `Post to LinkedIn`.

17. **Validate data flow end-to-end**
    - Test with a push that modifies a small text file.
    - Confirm:
      - AI output matches schema
      - HCTI returns `url`
      - LinkedIn node receives the correct binary image
      - GitHub upload points to the image (not the source code)

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Disclaimer provided by user |
| PrismJS theme and components are loaded via CDN (`cdnjs.cloudflare.com`) | Used in “Create Code HTML” to render highlighted code |
| Google Font “Fira Code” is referenced in HTML and also passed to HCTI `google_fonts` | Ensures consistent monospace rendering in generated image |

