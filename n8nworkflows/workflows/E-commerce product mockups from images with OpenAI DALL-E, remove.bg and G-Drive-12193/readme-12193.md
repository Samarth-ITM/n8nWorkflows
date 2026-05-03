E-commerce product mockups from images with OpenAI DALL-E, remove.bg and G-Drive

https://n8nworkflows.xyz/workflows/e-commerce-product-mockups-from-images-with-openai-dall-e--remove-bg-and-g-drive-12193


# E-commerce product mockups from images with OpenAI DALL-E, remove.bg and G-Drive

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow generates **e-commerce product mockups** from submitted product images. It accepts a product image via an n8n Form Trigger, optionally routes the user to a notice branch, or proceeds to: **download the image → remove background (remove.bg via HTTP) → upload the PNG to Google Drive → generate a mockup image with OpenAI (DALL·E) → upload mockup to Google Drive → download the final mockup → display it back to the user in an n8n Form page**.

**Target use cases:**
- Create marketing-ready product mockups from raw product photos
- Standardize product cut-outs (transparent PNG) and generate styled scenes
- Return a downloadable result to the user immediately through n8n Forms

### 1.1 Input Reception & Branching
Receives a form submission and decides which processing path to follow.

### 1.2 Main Mockup Generation Path (Path A)
Processes the submitted image through background removal, Drive uploads, OpenAI image generation, and returns the final mockup.

### 1.3 Alternate/Secondary Mockup Generation Path (Path B)
A second, similar chain that starts by downloading an image, then performs background removal + mockup generation again.

### 1.4 User-Facing Responses
Shows either a notice page or a mockup result page (two variants).

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Routing

**Overview:**  
Accepts a user submission from an n8n Form Trigger and routes execution through conditional “If” nodes. These conditions decide whether to run the main image pipeline or an alternate pipeline / notice.

**Nodes involved:**
- **Form submission** (Form Trigger)
- **If**
- **If1**

#### Node: Form submission
- **Type / role:** `n8n-nodes-base.formTrigger` — Entry point that receives form submissions.
- **Configuration (interpreted):** Uses a fixed webhook identifier (`webhookId` present). The form fields are not included in the provided JSON (parameters empty), so field names/validation are unknown.
- **Inputs / outputs:** Entry node → outputs to **If**
- **Edge cases / failures:**
  - Missing expected form fields (downstream expressions may fail if used; not visible here because node parameters are empty in export)
  - Webhook URL not accessible (instance networking / public URL)
- **Version:** 2.3

#### Node: If
- **Type / role:** `n8n-nodes-base.if` — Primary branch decision.
- **Configuration:** Not provided (parameters empty), so the actual condition is unknown.
- **Connections:**
  - **True (Output 0)** → **Remove Image Background** (starts Path A)
  - **False (Output 1)** → **If1** (secondary decision)
- **Edge cases:**
  - If condition uses fields that are missing/null, it may evaluate unexpectedly or throw expression errors (depending on configured rules)
- **Version:** 2.2

#### Node: If1
- **Type / role:** `n8n-nodes-base.if` — Secondary branch decision.
- **Configuration:** Not provided (parameters empty), so the condition is unknown.
- **Connections:**
  - **True (Output 0)** → **HTTP (download image2** (starts Path B)
  - **False (Output 1)** → **Form Notice**
- **Edge cases:** Same as “If” (unknown condition logic).
- **Version:** 2.2

---

### Block 2 — Main Mockup Generation Path (Path A)

**Overview:**  
Takes the submitted image (from the form), removes its background (typically via remove.bg API), uploads the transparent PNG to Google Drive, asks OpenAI to generate a mockup image, uploads the mockup image to Drive, downloads it, and finally displays it to the user.

**Nodes involved:**
- **Remove Image Background**
- **Upload PNG Image**
- **Generate an image**
- **Upload Mockup Image**
- **HTTP (download image)**
- **Form Mockup**

#### Node: Remove Image Background
- **Type / role:** `n8n-nodes-base.httpRequest` — Calls an external HTTP endpoint (typically remove.bg) to remove background.
- **Configuration:** Parameters are empty in the export; in a functioning workflow this typically includes:
  - URL: remove.bg endpoint (e.g., `https://api.remove.bg/v1.0/removebg`)
  - Authentication header: `X-Api-Key: <REMOVE_BG_KEY>`
  - Binary upload of the product image from the form
  - Response as binary (PNG)
- **Connections:** **If (true)** → this node → **Upload PNG Image**
- **Edge cases / failures:**
  - 401/403 (bad API key), 429 (rate limit), 400 (invalid image), 413 (too large), timeouts
  - If the incoming item does not contain binary data, request will fail
- **Version:** 4.2

#### Node: Upload PNG Image
- **Type / role:** `n8n-nodes-base.googleDrive` — Uploads the background-removed PNG to Google Drive.
- **Configuration:** Not provided; typically configured as “Upload” operation with:
  - Destination folder
  - Binary property name (e.g., `data`)
  - File name (often derived from product name/id)
- **Connections:** **Remove Image Background** → **Generate an image**
- **Credentials:** Google Drive OAuth2 or Service Account (depending on node setup).
- **Edge cases:**
  - Auth expired/insufficient scopes
  - Upload fails if binary property name mismatches actual output
- **Version:** 3

#### Node: Generate an image
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — Uses OpenAI image generation (DALL·E) to create a mockup.
- **Configuration:** Parameters empty; typically includes:
  - Model (e.g., DALL·E)
  - Prompt built from product details + styling instructions
  - Possibly input image reference (if using image editing/variations) or prompt-only generation
- **Behavior settings:**
  - `retryOnFail: true`
  - `maxTries: 2`
- **Connections:** **Upload PNG Image** → **Upload Mockup Image**
- **Edge cases:**
  - OpenAI auth errors, quota exceeded
  - Safety policy rejections
  - Prompt/image parameter mismatch
- **Version:** 1.8

#### Node: Upload Mockup Image
- **Type / role:** `n8n-nodes-base.googleDrive` — Uploads the generated mockup to Google Drive.
- **Configuration:** Not shown; usually “Upload” with binary from OpenAI node output.
- **Connections:** **Generate an image** → **HTTP (download image)**
- **Edge cases:** Same Drive upload concerns as above.
- **Version:** 3

#### Node: HTTP (download image)
- **Type / role:** `n8n-nodes-base.httpRequest` — Downloads the uploaded mockup (often via a Drive file link) to provide it to the Form node as binary.
- **Configuration:** Not shown; typically:
  - URL from Google Drive node output (download URL or “webContentLink”)
  - Response format: file/binary
- **Connections:** **Upload Mockup Image** → **Form Mockup**
- **Edge cases:**
  - Drive link not publicly accessible without auth
  - Needs correct “download” URL format
- **Version:** 4.2

#### Node: Form Mockup
- **Type / role:** `n8n-nodes-base.form` — Presents the final mockup to the user (likely with an image preview/download).
- **Configuration:** Empty; in practice this node defines the form page shown after processing.
- **Connections:** Receives from **HTTP (download image)**; no further outputs.
- **Version:** 2.3

---

### Block 3 — Alternate/Secondary Mockup Generation Path (Path B)

**Overview:**  
A second pipeline that starts by downloading an image (likely from a URL provided in the initial form), then performs background removal, Drive upload, OpenAI mockup generation, Drive upload, downloads the result, and displays it in a second mockup form.

**Nodes involved:**
- **HTTP (download image2**
- **Remove Image Background2**
- **Upload PNG Image2**
- **Generate an image2**
- **Upload Mockup Image2**
- **HTTP (download image)3**
- **Form Mockup2**

#### Node: HTTP (download image2
- **Type / role:** `n8n-nodes-base.httpRequest` — Downloads the source image to binary.
- **Configuration:** Not shown; typically uses a URL field from the form submission.
- **Connections:** **If1 (true)** → **Remove Image Background2**
- **Edge cases:**
  - Invalid URL / blocked remote host / SSL errors
  - Non-image content returned
- **Version:** 4.2  
*(Note: the node name appears truncated: “HTTP (download image2”. This is only cosmetic but can be confusing.)*

#### Node: Remove Image Background2
- **Type / role:** `n8n-nodes-base.httpRequest` — Background removal (same as Path A) but for this branch.
- **Connections:** **HTTP (download image2** → **Upload PNG Image2**
- **Edge cases:** Same as Path A remove.bg call.
- **Version:** 4.2

#### Node: Upload PNG Image2
- **Type / role:** `n8n-nodes-base.googleDrive` — Uploads PNG to Drive (branch variant).
- **Connections:** **Remove Image Background2** → **Generate an image2**
- **Edge cases:** Same as Path A Drive upload.
- **Version:** 3

#### Node: Generate an image2
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — Second OpenAI generation step (branch variant).
- **Behavior settings:** `retryOnFail: true`, `maxTries: 2`
- **Connections:** **Upload PNG Image2** → **Upload Mockup Image2**
- **Edge cases:** Same as Path A OpenAI node.
- **Version:** 1.8

#### Node: Upload Mockup Image2
- **Type / role:** `n8n-nodes-base.googleDrive` — Uploads generated mockup (branch variant).
- **Connections:** **Generate an image2** → **HTTP (download image)3**
- **Edge cases:** Same Drive upload concerns.
- **Version:** 3

#### Node: HTTP (download image)3
- **Type / role:** `n8n-nodes-base.httpRequest` — Downloads the mockup image for display.
- **Connections:** **Upload Mockup Image2** → **Form Mockup2**
- **Edge cases:** Same download/link-access concerns as Path A.
- **Version:** 4.2

#### Node: Form Mockup2
- **Type / role:** `n8n-nodes-base.form` — Displays mockup result for Path B.
- **Connections:** End node for this branch.
- **Version:** 2.3

---

### Block 4 — User Notice / Stop Path

**Overview:**  
If conditions are not met in the secondary If, the workflow displays a notice page to the user rather than generating a mockup.

**Nodes involved:**
- **Form Notice**

#### Node: Form Notice
- **Type / role:** `n8n-nodes-base.form` — Displays a notice/notification page.
- **Connections:** **If1 (false)** → this node; ends workflow.
- **Edge cases:** Minimal; primarily configuration/UX related.
- **Version:** 2.3

---

### Block 5 — Sticky Notes (Documentation Layer)

**Overview:**  
The workflow includes multiple sticky notes but all have empty content in the provided JSON export.

**Nodes involved:**
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note5

For each sticky note:
- **Type / role:** `n8n-nodes-base.stickyNote` — Canvas annotation.
- **Configuration:** `content` is empty.
- **Execution:** Sticky notes do not execute and do not affect workflow logic.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Form submission | formTrigger | Entry point (form submission) | — | If |  |
| If | if | Primary routing decision | Form submission | Remove Image Background; If1 |  |
| Remove Image Background | httpRequest | Background removal (remove.bg-like API call) | If | Upload PNG Image |  |
| Upload PNG Image | googleDrive | Store transparent PNG in Drive | Remove Image Background | Generate an image |  |
| Generate an image | openAi (LangChain) | Generate mockup with OpenAI (DALL·E) | Upload PNG Image | Upload Mockup Image |  |
| Upload Mockup Image | googleDrive | Upload generated mockup to Drive | Generate an image | HTTP (download image) |  |
| HTTP (download image) | httpRequest | Download mockup image for display | Upload Mockup Image | Form Mockup |  |
| Form Mockup | form | Display mockup result (Path A) | HTTP (download image) | — |  |
| If1 | if | Secondary routing decision | If | HTTP (download image2; Form Notice |  |
| HTTP (download image2 | httpRequest | Download source image for Path B | If1 | Remove Image Background2 |  |
| Remove Image Background2 | httpRequest | Background removal (Path B) | HTTP (download image2 | Upload PNG Image2 |  |
| Upload PNG Image2 | googleDrive | Store transparent PNG (Path B) | Remove Image Background2 | Generate an image2 |  |
| Generate an image2 | openAi (LangChain) | Generate mockup (Path B) | Upload PNG Image2 | Upload Mockup Image2 |  |
| Upload Mockup Image2 | googleDrive | Upload generated mockup (Path B) | Generate an image2 | HTTP (download image)3 |  |
| HTTP (download image)3 | httpRequest | Download mockup (Path B) for display | Upload Mockup Image2 | Form Mockup2 |  |
| Form Mockup2 | form | Display mockup result (Path B) | HTTP (download image)3 | — |  |
| Form Notice | form | Display notice / stop path | If1 | — |  |
| Sticky Note | stickyNote | Canvas annotation (empty) | — | — |  |
| Sticky Note1 | stickyNote | Canvas annotation (empty) | — | — |  |
| Sticky Note2 | stickyNote | Canvas annotation (empty) | — | — |  |
| Sticky Note3 | stickyNote | Canvas annotation (empty) | — | — |  |
| Sticky Note5 | stickyNote | Canvas annotation (empty) | — | — |  |

---

## 4. Reproducing the Workflow from Scratch

> Note: The exported JSON has **empty parameters** for almost all functional nodes. Steps below describe the required structure and the typical configuration you must add to make it operational.

### 4.1 Create the entry and routing
1. **Add node:** *Form Trigger*  
   - Name: `Form submission`  
   - Configure form fields (recommended):
     - `product_image` (File upload) **or** `image_url` (Text/URL)
     - Optional: `product_name`, `brand`, `mockup_style`, `background`, `aspect_ratio`
2. **Add node:** *If*  
   - Name: `If`  
   - Condition example:
     - If `product_image` binary exists → Path A  
     - Else → Path B (or further check)
3. Connect: `Form submission` → `If`.

4. **Add node:** *If*  
   - Name: `If1`  
   - Condition example:
     - If `image_url` is not empty → Path B  
     - Else → show notice
5. Connect: `If (false)` → `If1`.

### 4.2 Path A (binary upload from form) — remove background → Drive → OpenAI → Drive → download → display
6. **Add node:** *HTTP Request*  
   - Name: `Remove Image Background`  
   - Method: `POST`  
   - URL: `https://api.remove.bg/v1.0/removebg`  
   - Headers: `X-Api-Key: {{$credentials.removeBgApiKey}}` (or store in n8n credential/environment)  
   - Send binary: the uploaded `product_image`  
   - Response: **File/Binary** (PNG)
7. Connect: `If (true)` → `Remove Image Background`.

8. **Add node:** *Google Drive*  
   - Name: `Upload PNG Image`  
   - Operation: Upload  
   - Binary property: output binary from remove.bg (commonly `data`)  
   - File name: e.g. `{{$json.product_name || 'product'}}-cutout.png`  
   - Folder: choose a dedicated folder (e.g., `/mockups/input-png/`)
   - **Credentials:** Google Drive OAuth2 with Drive scope
9. Connect: `Remove Image Background` → `Upload PNG Image`.

10. **Add node:** *OpenAI (LangChain) → Image generation*  
    - Name: `Generate an image`  
    - Configure OpenAI credentials (API key)  
    - Model: DALL·E image generation model available in your n8n/OpenAI setup  
    - Prompt: include style + product context; optionally include the Drive file URL if your approach uses it  
    - Keep: `retryOnFail = true`, `maxTries = 2` (as in JSON)
11. Connect: `Upload PNG Image` → `Generate an image`.

12. **Add node:** *Google Drive*  
    - Name: `Upload Mockup Image`  
    - Operation: Upload  
    - Binary property: image output from OpenAI node  
    - Folder: `/mockups/output/`
13. Connect: `Generate an image` → `Upload Mockup Image`.

14. **Add node:** *HTTP Request*  
    - Name: `HTTP (download image)`  
    - Use it to fetch the image binary you want to show in the form:
      - Either download from the OpenAI-provided URL, or from a Drive download link.
    - Response: File/Binary
15. Connect: `Upload Mockup Image` → `HTTP (download image)`.

16. **Add node:** *Form*  
    - Name: `Form Mockup`  
    - Configure the response page to display the binary image and/or a download link.
17. Connect: `HTTP (download image)` → `Form Mockup`.

### 4.3 Path B (URL download) — download → remove background → Drive → OpenAI → Drive → download → display
18. **Add node:** *HTTP Request*  
    - Name: `HTTP (download image2`  
    - Method: `GET`  
    - URL: `{{$json.image_url}}`  
    - Response: File/Binary
19. Connect: `If1 (true)` → `HTTP (download image2`.

20. **Add node:** *HTTP Request*  
    - Name: `Remove Image Background2`  
    - Same remove.bg configuration as step 6, using the binary from step 18.
21. Connect: `HTTP (download image2` → `Remove Image Background2`.

22. **Add node:** *Google Drive*  
    - Name: `Upload PNG Image2`  
    - Same as step 8, but optionally store in a different folder.
23. Connect: `Remove Image Background2` → `Upload PNG Image2`.

24. **Add node:** *OpenAI (LangChain) → Image generation*  
    - Name: `Generate an image2`  
    - Same concept as step 10 (prompt can differ).
25. Connect: `Upload PNG Image2` → `Generate an image2`.

26. **Add node:** *Google Drive*  
    - Name: `Upload Mockup Image2`
27. Connect: `Generate an image2` → `Upload Mockup Image2`.

28. **Add node:** *HTTP Request*  
    - Name: `HTTP (download image)3`  
    - Download final mockup as binary.
29. Connect: `Upload Mockup Image2` → `HTTP (download image)3`.

30. **Add node:** *Form*  
    - Name: `Form Mockup2`  
    - Displays the mockup for Path B.
31. Connect: `HTTP (download image)3` → `Form Mockup2`.

### 4.4 Notice path
32. **Add node:** *Form*  
    - Name: `Form Notice`  
    - Configure message like: “Please upload an image or provide a valid image URL.”
33. Connect: `If1 (false)` → `Form Notice`.

### 4.5 Credentials to prepare
- **Google Drive credentials:** OAuth2 (recommended) with permission to upload files to target folders.
- **OpenAI credentials:** API key with image generation access.
- **remove.bg API key:** store securely (n8n credential, environment variable, or header value in HTTP Request node).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sticky notes exist but contain no text in the provided workflow export. | Canvas annotations are empty (`content: ""`). |
| The workflow contains two parallel generation branches (Path A and Path B) with duplicated nodes. | Consider consolidating via a single “download/normalize input to binary” step to reduce maintenance. |
| Several nodes have empty parameters in the export; operational behavior depends on configuring HTTP endpoints, binary property names, Drive folders, and OpenAI prompts/models. | Ensure each node’s binary property names match upstream outputs (common failure point). |