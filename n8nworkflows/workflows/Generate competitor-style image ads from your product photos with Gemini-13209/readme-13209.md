Generate competitor-style image ads from your product photos with Gemini

https://n8nworkflows.xyz/workflows/generate-competitor-style-image-ads-from-your-product-photos-with-gemini-13209


# Generate competitor-style image ads from your product photos with Gemini

## 1. Workflow Overview

This workflow, **“Generate competitor-style image ads from your product photos with Gemini”** (internal name: **Image Ad Cloner AI Agent**), takes a user-submitted product image and a competitor ad image URL, then uses Gemini API endpoints to generate a new ad image that mimics the competitor’s visual style while replacing the featured product and brand elements with the user’s own product.

Typical use cases include:
- Rapid ad creative iteration
- Competitive creative benchmarking
- Re-skinning ad concepts for internal testing
- Producing AI-generated ad variants from reference visuals

The workflow is organized into three main logical blocks:

### 1.1 Input Reception and Image Preparation
The workflow starts with a form submission. The user provides:
- A competitor image link
- A product image file
- Optional additional change instructions

The uploaded and downloaded images are then converted into base64-compatible content so they can be embedded directly in Gemini API requests.

### 1.2 Prompt Engineering and AI Generation
A detailed prompt is assembled to instruct Gemini to analyze both images, identify branding/text/product placement changes, and produce a rewritten generation prompt. That rewritten prompt is then passed to an image-generation Gemini endpoint together with both images.

### 1.3 Result Processing and Output Preparation
After the image generation request, the workflow pauses in a Wait node, then extracts the generated inline image data from the Gemini response, converts it back into binary file format, and prepares it as the final output.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and Image Preparation

**Overview:**  
This block collects the user’s input through an n8n form, retrieves the competitor image from the submitted URL, and converts both the uploaded product image and downloaded competitor image into base64 text suitable for inline API payloads.

**Nodes Involved:**  
- `form_trigger`
- `convert_product_image_to_base64`
- `download_image`
- `convert_ad_image_to_base64`

### Node: `form_trigger`

- **Type and technical role:**  
  `n8n-nodes-base.formTrigger`  
  Entry-point node that exposes a form and starts the workflow when a user submits it.

- **Configuration choices:**  
  - Form title: **Facebook Ad Thief Form**
  - Description explains that the user must upload a product image and submit a competitor image URL
  - Fields:
    - `Competitor Image Link` — required text field
    - `Your Product Image` — required single file upload
    - `Additional changes` — optional text field

- **Key expressions or variables used:**  
  Downstream nodes reference:
  - `$('form_trigger').item.json['Competitor Image Link']`
  - `$('form_trigger').item.json['Additional changes']`
  - Uploaded file binary property is used by `convert_product_image_to_base64`

- **Input and output connections:**  
  - Input: none
  - Output: `convert_product_image_to_base64`

- **Version-specific requirements:**  
  Type version `2.2` of the Form Trigger node.

- **Edge cases or potential failure types:**  
  - Invalid or inaccessible competitor URL later causes HTTP download failure
  - Missing uploaded file prevents product image conversion
  - Unexpected file type may later cause MIME mismatch if Gemini expects a supported image format

- **Sub-workflow reference:**  
  None

---

### Node: `convert_product_image_to_base64`

- **Type and technical role:**  
  `n8n-nodes-base.extractFromFile`  
  Converts the uploaded binary product image into a text/base64 property.

- **Configuration choices:**  
  - Operation: `binaryToPropery`
  - Binary property name: `Your_Product_Image`

- **Key expressions or variables used:**  
  Produces `json.data`, which is later referenced as:
  - `$node['convert_product_image_to_base64'].json.data`

- **Input and output connections:**  
  - Input: `form_trigger`
  - Output: `download_image`

- **Version-specific requirements:**  
  Type version `1`.

- **Edge cases or potential failure types:**  
  - If the uploaded file is not stored under `Your_Product_Image`, conversion fails
  - Large files may create oversized request payloads later
  - If the uploaded file is not PNG but later sent as `image/png`, the declared MIME type may not match the actual content

- **Sub-workflow reference:**  
  None

---

### Node: `download_image`

- **Type and technical role:**  
  `n8n-nodes-base.httpRequest`  
  Fetches the competitor ad image from the submitted URL.

- **Configuration choices:**  
  - URL dynamically read from the form:
    `={{ $('form_trigger').item.json['Competitor Image Link'] }}`

- **Key expressions or variables used:**  
  - Competitor image URL from the form submission

- **Input and output connections:**  
  - Input: `convert_product_image_to_base64`
  - Output: `convert_ad_image_to_base64`

- **Version-specific requirements:**  
  Type version `4.2`.

- **Edge cases or potential failure types:**  
  - URL returns HTML instead of an image
  - Direct Facebook URLs often require authentication, redirects, or anti-bot handling
  - 403/404/timeout/network failures
  - If response is not binary image data, the next conversion node may fail or produce invalid base64

- **Sub-workflow reference:**  
  None

---

### Node: `convert_ad_image_to_base64`

- **Type and technical role:**  
  `n8n-nodes-base.extractFromFile`  
  Converts the downloaded competitor ad image into base64 text for inline Gemini request use.

- **Configuration choices:**  
  - Operation: `binaryToPropery`
  - Uses default binary property from the previous HTTP response

- **Key expressions or variables used:**  
  Produces `json.data`, later used as:
  - `$node['convert_ad_image_to_base64'].json.data`

- **Input and output connections:**  
  - Input: `download_image`
  - Output: `build_prompt`

- **Version-specific requirements:**  
  Type version `1`.

- **Edge cases or potential failure types:**  
  - Failure if previous node returned no binary
  - Failure if downloaded content is not a valid file/image
  - Large binary payload may exceed downstream Gemini request size limits

- **Sub-workflow reference:**  
  None

---

## 2.2 Prompt Engineering and AI Generation

**Overview:**  
This block builds a detailed instruction set, sends it to a Gemini reasoning model to produce a refined generation prompt, then uses that prompt with an image-generation Gemini model to create the final ad image.

**Nodes Involved:**  
- `build_prompt`
- `generate_ad_image_prompt`
- `generate_ad_image`
- `Wait`

### Node: `build_prompt`

- **Type and technical role:**  
  `n8n-nodes-base.set`  
  Creates a `prompt` field containing detailed image-editing instructions.

- **Configuration choices:**  
  - Assigns a single string field: `prompt`
  - The prompt instructs the model to:
    - Analyze both the user’s product image and competitor ad image
    - Replicate the competitor style
    - Replace competitor packaging/branding with the user’s product
    - Detect partial branding text and label fragments
    - Preserve ad copy style while swapping brand references
    - Include any user-provided additional changes

- **Key expressions or variables used:**  
  - `{{ $('form_trigger').item.json['Additional changes'] }}` embedded inside the prompt

- **Input and output connections:**  
  - Input: `convert_ad_image_to_base64`
  - Output: `generate_ad_image_prompt`

- **Version-specific requirements:**  
  Type version `3.4`.

- **Edge cases or potential failure types:**  
  - Empty “Additional changes” is acceptable, but results in a less customized prompt
  - Prompt quality strongly affects output quality
  - The prompt contains awkward or incomplete phrases such as partial quoted fragments; this may reduce consistency or create ambiguous model behavior

- **Sub-workflow reference:**  
  None

---

### Node: `generate_ad_image_prompt`

- **Type and technical role:**  
  `n8n-nodes-base.httpRequest`  
  Calls Gemini 2.5 Pro to generate or refine the prompt/content strategy before image generation.

- **Configuration choices:**  
  - Method: `POST`
  - Endpoint:  
    `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-pro:generateContent`
  - Body sent as JSON
  - Authentication: generic credential type with header auth
  - Retries enabled
  - Wait between retries: 5000 ms

- **Key expressions or variables used:**  
  JSON body includes:
  - Prompt text from `$json.prompt`
  - Product image base64 from `$node['convert_product_image_to_base64'].json.data`
  - Competitor image base64 from `$node['convert_ad_image_to_base64'].json.data`

  The request sends three parts:
  1. Text instruction
  2. Product image as inline data with MIME `image/png`
  3. Competitor image as inline data with MIME `image/jpeg`

- **Input and output connections:**  
  - Input: `build_prompt`
  - Output: `generate_ad_image`

- **Version-specific requirements:**  
  Type version `4.2`.

- **Edge cases or potential failure types:**  
  - Invalid Gemini credentials or header auth
  - Unsupported model name or API version changes
  - MIME type mismatch between declared and actual image
  - Request body too large due to base64 image sizes
  - Response format changes could break downstream expression access
  - Rate limiting or quota exhaustion
  - Retry only helps on transient failures, not malformed requests

- **Sub-workflow reference:**  
  None

---

### Node: `generate_ad_image`

- **Type and technical role:**  
  `n8n-nodes-base.httpRequest`  
  Calls Gemini Flash Image generation using the prompt text produced by the prior Gemini response.

- **Configuration choices:**  
  - Method: `POST`
  - Endpoint:  
    `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent`
  - Body sent as JSON
  - Authentication: generic credential type with header auth
  - Retries enabled
  - Wait between retries: 5000 ms

- **Key expressions or variables used:**  
  The request body uses:
  - Prompt text from the previous Gemini response:  
    `$json.candidates.first().content.parts.first().text`
  - Product image base64 from `convert_product_image_to_base64`
  - Competitor image base64 from `convert_ad_image_to_base64`

- **Input and output connections:**  
  - Input: `generate_ad_image_prompt`
  - Output: `Wait`

- **Version-specific requirements:**  
  Type version `4.2`.

- **Edge cases or potential failure types:**  
  - If `generate_ad_image_prompt` returns no `candidates`, the expression fails
  - If the first part is not text, the prompt extraction fails
  - Image-generation endpoint may return text-only or alternate response structures
  - Model availability, quota, or regional restrictions
  - MIME mismatch and oversized payload risks still apply

- **Sub-workflow reference:**  
  None

---

### Node: `Wait`

- **Type and technical role:**  
  `n8n-nodes-base.wait`  
  Pauses execution before result processing.

- **Configuration choices:**  
  - Default parameters; no explicit wait condition shown
  - A webhook ID exists, indicating the node can resume later through n8n’s wait mechanism

- **Key expressions or variables used:**  
  None

- **Input and output connections:**  
  - Input: `generate_ad_image`
  - Output: `set_result`

- **Version-specific requirements:**  
  Type version `1.1`.

- **Edge cases or potential failure types:**  
  - As configured, this node may be unnecessary if Gemini returns synchronously
  - If the workflow expects a manual/webhook resume and none occurs, execution may remain paused
  - Misunderstanding this node’s behavior can make the workflow appear “stuck”

- **Sub-workflow reference:**  
  None

---

## 2.3 Result Processing and Output Preparation

**Overview:**  
This block parses the Gemini image-generation response, extracts the returned inline image data, and converts it into a binary file for output or download.

**Nodes Involved:**  
- `set_result`
- `get_image`

### Node: `set_result`

- **Type and technical role:**  
  `n8n-nodes-base.set`  
  Extracts the generated image base64 from the Gemini response and stores it in a clean property.

- **Configuration choices:**  
  - Creates `image_result`
  - Expression filters the Gemini response parts for a part containing `inlineData`

- **Key expressions or variables used:**  
  - `{{ $json.candidates[0].content.parts.filter(item => item.inlineData).first().inlineData.data }}`

- **Input and output connections:**  
  - Input: `Wait`
  - Output: `get_image`

- **Version-specific requirements:**  
  Type version `3.4`.

- **Edge cases or potential failure types:**  
  - If the response contains no inline image data, the expression fails
  - If the API returns `inline_data` instead of `inlineData` or changes structure, extraction breaks
  - If candidate index `0` does not exist, execution fails

- **Sub-workflow reference:**  
  None

---

### Node: `get_image`

- **Type and technical role:**  
  `n8n-nodes-base.convertToFile`  
  Converts the extracted base64 string into binary file output.

- **Configuration choices:**  
  - Operation: `toBinary`
  - Source property: `image_result`

- **Key expressions or variables used:**  
  Reads `image_result` from the previous node.

- **Input and output connections:**  
  - Input: `set_result`
  - Output: none

- **Version-specific requirements:**  
  Type version `1.1`.

- **Edge cases or potential failure types:**  
  - If `image_result` is empty or malformed, binary conversion fails
  - No explicit file name or MIME type is set, so downstream consumers may need additional metadata

- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| form_trigger | Form Trigger | Workflow entry form for URL, image upload, and optional instructions |  | convert_product_image_to_base64 | # Image Ad Cloner AI Agent  This n8n workflow automates “cloning” a competitor ad style and re-generating it using your product image. A user uploads your product image and provides a competitor ad URL via a form trigger; the workflow downloads both images, converts them to inline/base64 parts, builds a detailed generation prompt, sends the request to a generative-vision model (Gemini-style endpoints), waits for the result, converts the output back to a file and returns it. Use-cases: rapid ad iteration, creative A/B variants, and scaled ad re-skins.  \n## How it works (step-by-step) 1. `form_trigger` — user submits: competitor image URL, their product file, and optional changes. 2. `convert_product_image_to_base64` — converts uploaded product file to base64 inline data. 3. `download_image` — fetches the competitor ad image from the provided URL. 4. `convert_ad_image_to_base64` — converts downloaded competitor image to base64 inline data. 5. `build_prompt` — assembles a careful prompt (handles partial text, labels, placements, CTA swaps, additional changes). 6. `generate_ad_image_prompt` → `generate_ad_image` — two HTTP Request nodes call generative endpoints (first to prepare prompt/content and then flash-image generation). These include inline image data and model-specific parameters. 7. `Wait` — allows async generation to complete (workflow uses a wait/webhook pattern). 8. `set_result` → `get_image` — extract image result, convert binary to file for downstream use (download or return to the requester). 9. `Sticky Note` — visual diagram / docs inside workflow.  \n👉 [Demo & Setup Video](https://drive.google.com/file/d/1PLFPHSXQ-DaNAKeBQv7G9SzaXT0c3hRt/view?usp=sharing)  \n👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC) |
| convert_product_image_to_base64 | Extract From File | Converts uploaded product image binary into base64 text | form_trigger | download_image | # Image Ad Cloner AI Agent  This n8n workflow automates “cloning” a competitor ad style and re-generating it using your product image. A user uploads your product image and provides a competitor ad URL via a form trigger; the workflow downloads both images, converts them to inline/base64 parts, builds a detailed generation prompt, sends the request to a generative-vision model (Gemini-style endpoints), waits for the result, converts the output back to a file and returns it. Use-cases: rapid ad iteration, creative A/B variants, and scaled ad re-skins.  \n## How it works (step-by-step) 1. `form_trigger` — user submits: competitor image URL, their product file, and optional changes. 2. `convert_product_image_to_base64` — converts uploaded product file to base64 inline data. 3. `download_image` — fetches the competitor ad image from the provided URL. 4. `convert_ad_image_to_base64` — converts downloaded competitor image to base64 inline data. 5. `build_prompt` — assembles a careful prompt (handles partial text, labels, placements, CTA swaps, additional changes). 6. `generate_ad_image_prompt` → `generate_ad_image` — two HTTP Request nodes call generative endpoints (first to prepare prompt/content and then flash-image generation). These include inline image data and model-specific parameters. 7. `Wait` — allows async generation to complete (workflow uses a wait/webhook pattern). 8. `set_result` → `get_image` — extract image result, convert binary to file for downstream use (download or return to the requester). 9. `Sticky Note` — visual diagram / docs inside workflow.  \n## 📥 INPUT & IMAGE PREPARATION This section collects the required inputs and prepares images for AI processing. |
| download_image | HTTP Request | Downloads competitor image from submitted link | convert_product_image_to_base64 | convert_ad_image_to_base64 | # Image Ad Cloner AI Agent  This n8n workflow automates “cloning” a competitor ad style and re-generating it using your product image. A user uploads your product image and provides a competitor ad URL via a form trigger; the workflow downloads both images, converts them to inline/base64 parts, builds a detailed generation prompt, sends the request to a generative-vision model (Gemini-style endpoints), waits for the result, converts the output back to a file and returns it. Use-cases: rapid ad iteration, creative A/B variants, and scaled ad re-skins.  \n## How it works (step-by-step) 1. `form_trigger` — user submits: competitor image URL, their product file, and optional changes. 2. `convert_product_image_to_base64` — converts uploaded product file to base64 inline data. 3. `download_image` — fetches the competitor ad image from the provided URL. 4. `convert_ad_image_to_base64` — converts downloaded competitor image to base64 inline data. 5. `build_prompt` — assembles a careful prompt (handles partial text, labels, placements, CTA swaps, additional changes). 6. `generate_ad_image_prompt` → `generate_ad_image` — two HTTP Request nodes call generative endpoints (first to prepare prompt/content and then flash-image generation). These include inline image data and model-specific parameters. 7. `Wait` — allows async generation to complete (workflow uses a wait/webhook pattern). 8. `set_result` → `get_image` — extract image result, convert binary to file for downstream use (download or return to the requester). 9. `Sticky Note` — visual diagram / docs inside workflow.  \n## 📥 INPUT & IMAGE PREPARATION This section collects the required inputs and prepares images for AI processing. |
| convert_ad_image_to_base64 | Extract From File | Converts downloaded competitor image binary into base64 text | download_image | build_prompt | # Image Ad Cloner AI Agent  This n8n workflow automates “cloning” a competitor ad style and re-generating it using your product image. A user uploads your product image and provides a competitor ad URL via a form trigger; the workflow downloads both images, converts them to inline/base64 parts, builds a detailed generation prompt, sends the request to a generative-vision model (Gemini-style endpoints), waits for the result, converts the output back to a file and returns it. Use-cases: rapid ad iteration, creative A/B variants, and scaled ad re-skins.  \n## How it works (step-by-step) 1. `form_trigger` — user submits: competitor image URL, their product file, and optional changes. 2. `convert_product_image_to_base64` — converts uploaded product file to base64 inline data. 3. `download_image` — fetches the competitor ad image from the provided URL. 4. `convert_ad_image_to_base64` — converts downloaded competitor image to base64 inline data. 5. `build_prompt` — assembles a careful prompt (handles partial text, labels, placements, CTA swaps, additional changes). 6. `generate_ad_image_prompt` → `generate_ad_image` — two HTTP Request nodes call generative endpoints (first to prepare prompt/content and then flash-image generation). These include inline image data and model-specific parameters. 7. `Wait` — allows async generation to complete (workflow uses a wait/webhook pattern). 8. `set_result` → `get_image` — extract image result, convert binary to file for downstream use (download or return to the requester). 9. `Sticky Note` — visual diagram / docs inside workflow.  \n## 📥 INPUT & IMAGE PREPARATION This section collects the required inputs and prepares images for AI processing. |
| build_prompt | Set | Builds the instruction prompt for Gemini analysis and ad recreation | convert_ad_image_to_base64 | generate_ad_image_prompt | # Image Ad Cloner AI Agent  This n8n workflow automates “cloning” a competitor ad style and re-generating it using your product image. A user uploads your product image and provides a competitor ad URL via a form trigger; the workflow downloads both images, converts them to inline/base64 parts, builds a detailed generation prompt, sends the request to a generative-vision model (Gemini-style endpoints), waits for the result, converts the output back to a file and returns it. Use-cases: rapid ad iteration, creative A/B variants, and scaled ad re-skins.  \n## How it works (step-by-step) 1. `form_trigger` — user submits: competitor image URL, their product file, and optional changes. 2. `convert_product_image_to_base64` — converts uploaded product file to base64 inline data. 3. `download_image` — fetches the competitor ad image from the provided URL. 4. `convert_ad_image_to_base64` — converts downloaded competitor image to base64 inline data. 5. `build_prompt` — assembles a careful prompt (handles partial text, labels, placements, CTA swaps, additional changes). 6. `generate_ad_image_prompt` → `generate_ad_image` — two HTTP Request nodes call generative endpoints (first to prepare prompt/content and then flash-image generation). These include inline image data and model-specific parameters. 7. `Wait` — allows async generation to complete (workflow uses a wait/webhook pattern). 8. `set_result` → `get_image` — extract image result, convert binary to file for downstream use (download or return to the requester). 9. `Sticky Note` — visual diagram / docs inside workflow.  \n## 🧠 PROMPT ENGINEERING & AI GENERATION This section constructs the instructions and generates the new advertisement. |
| generate_ad_image_prompt | HTTP Request | Sends prompt plus both images to Gemini Pro to produce refined generation instructions | build_prompt | generate_ad_image | # Image Ad Cloner AI Agent  This n8n workflow automates “cloning” a competitor ad style and re-generating it using your product image. A user uploads your product image and provides a competitor ad URL via a form trigger; the workflow downloads both images, converts them to inline/base64 parts, builds a detailed generation prompt, sends the request to a generative-vision model (Gemini-style endpoints), waits for the result, converts the output back to a file and returns it. Use-cases: rapid ad iteration, creative A/B variants, and scaled ad re-skins.  \n## How it works (step-by-step) 1. `form_trigger` — user submits: competitor image URL, their product file, and optional changes. 2. `convert_product_image_to_base64` — converts uploaded product file to base64 inline data. 3. `download_image` — fetches the competitor ad image from the provided URL. 4. `convert_ad_image_to_base64` — converts downloaded competitor image to base64 inline data. 5. `build_prompt` — assembles a careful prompt (handles partial text, labels, placements, CTA swaps, additional changes). 6. `generate_ad_image_prompt` → `generate_ad_image` — two HTTP Request nodes call generative endpoints (first to prepare prompt/content and then flash-image generation). These include inline image data and model-specific parameters. 7. `Wait` — allows async generation to complete (workflow uses a wait/webhook pattern). 8. `set_result` → `get_image` — extract image result, convert binary to file for downstream use (download or return to the requester). 9. `Sticky Note` — visual diagram / docs inside workflow.  \n## 🧠 PROMPT ENGINEERING & AI GENERATION This section constructs the instructions and generates the new advertisement. |
| generate_ad_image | HTTP Request | Sends refined prompt plus both images to Gemini image generation model | generate_ad_image_prompt | Wait | # Image Ad Cloner AI Agent  This n8n workflow automates “cloning” a competitor ad style and re-generating it using your product image. A user uploads your product image and provides a competitor ad URL via a form trigger; the workflow downloads both images, converts them to inline/base64 parts, builds a detailed generation prompt, sends the request to a generative-vision model (Gemini-style endpoints), waits for the result, converts the output back to a file and returns it. Use-cases: rapid ad iteration, creative A/B variants, and scaled ad re-skins.  \n## How it works (step-by-step) 1. `form_trigger` — user submits: competitor image URL, their product file, and optional changes. 2. `convert_product_image_to_base64` — converts uploaded product file to base64 inline data. 3. `download_image` — fetches the competitor ad image from the provided URL. 4. `convert_ad_image_to_base64` — converts downloaded competitor image to base64 inline data. 5. `build_prompt` — assembles a careful prompt (handles partial text, labels, placements, CTA swaps, additional changes). 6. `generate_ad_image_prompt` → `generate_ad_image` — two HTTP Request nodes call generative endpoints (first to prepare prompt/content and then flash-image generation). These include inline image data and model-specific parameters. 7. `Wait` — allows async generation to complete (workflow uses a wait/webhook pattern). 8. `set_result` → `get_image` — extract image result, convert binary to file for downstream use (download or return to the requester). 9. `Sticky Note` — visual diagram / docs inside workflow.  \n## 🧠 PROMPT ENGINEERING & AI GENERATION This section constructs the instructions and generates the new advertisement. |
| Wait | Wait | Pauses execution before response parsing | generate_ad_image | set_result | # Image Ad Cloner AI Agent  This n8n workflow automates “cloning” a competitor ad style and re-generating it using your product image. A user uploads your product image and provides a competitor ad URL via a form trigger; the workflow downloads both images, converts them to inline/base64 parts, builds a detailed generation prompt, sends the request to a generative-vision model (Gemini-style endpoints), waits for the result, converts the output back to a file and returns it. Use-cases: rapid ad iteration, creative A/B variants, and scaled ad re-skins.  \n## How it works (step-by-step) 1. `form_trigger` — user submits: competitor image URL, their product file, and optional changes. 2. `convert_product_image_to_base64` — converts uploaded product file to base64 inline data. 3. `download_image` — fetches the competitor ad image from the provided URL. 4. `convert_ad_image_to_base64` — converts downloaded competitor image to base64 inline data. 5. `build_prompt` — assembles a careful prompt (handles partial text, labels, placements, CTA swaps, additional changes). 6. `generate_ad_image_prompt` → `generate_ad_image` — two HTTP Request nodes call generative endpoints (first to prepare prompt/content and then flash-image generation). These include inline image data and model-specific parameters. 7. `Wait` — allows async generation to complete (workflow uses a wait/webhook pattern). 8. `set_result` → `get_image` — extract image result, convert binary to file for downstream use (download or return to the requester). 9. `Sticky Note` — visual diagram / docs inside workflow.  \n## 📦 RESULT PROCESSING & OUTPUT This section processes the generated image and prepares the final result. |
| set_result | Set | Extracts generated image base64 from Gemini response | Wait | get_image | # Image Ad Cloner AI Agent  This n8n workflow automates “cloning” a competitor ad style and re-generating it using your product image. A user uploads your product image and provides a competitor ad URL via a form trigger; the workflow downloads both images, converts them to inline/base64 parts, builds a detailed generation prompt, sends the request to a generative-vision model (Gemini-style endpoints), waits for the result, converts the output back to a file and returns it. Use-cases: rapid ad iteration, creative A/B variants, and scaled ad re-skins.  \n## How it works (step-by-step) 1. `form_trigger` — user submits: competitor image URL, their product file, and optional changes. 2. `convert_product_image_to_base64` — converts uploaded product file to base64 inline data. 3. `download_image` — fetches the competitor ad image from the provided URL. 4. `convert_ad_image_to_base64` — converts downloaded competitor image to base64 inline data. 5. `build_prompt` — assembles a careful prompt (handles partial text, labels, placements, CTA swaps, additional changes). 6. `generate_ad_image_prompt` → `generate_ad_image` — two HTTP Request nodes call generative endpoints (first to prepare prompt/content and then flash-image generation). These include inline image data and model-specific parameters. 7. `Wait` — allows async generation to complete (workflow uses a wait/webhook pattern). 8. `set_result` → `get_image` — extract image result, convert binary to file for downstream use (download or return to the requester). 9. `Sticky Note` — visual diagram / docs inside workflow.  \n## 📦 RESULT PROCESSING & OUTPUT This section processes the generated image and prepares the final result. |
| get_image | Convert to File | Converts generated base64 string back into binary file output | set_result |  | # Image Ad Cloner AI Agent  This n8n workflow automates “cloning” a competitor ad style and re-generating it using your product image. A user uploads your product image and provides a competitor ad URL via a form trigger; the workflow downloads both images, converts them to inline/base64 parts, builds a detailed generation prompt, sends the request to a generative-vision model (Gemini-style endpoints), waits for the result, converts the output back to a file and returns it. Use-cases: rapid ad iteration, creative A/B variants, and scaled ad re-skins.  \n## How it works (step-by-step) 1. `form_trigger` — user submits: competitor image URL, their product file, and optional changes. 2. `convert_product_image_to_base64` — converts uploaded product file to base64 inline data. 3. `download_image` — fetches the competitor ad image from the provided URL. 4. `convert_ad_image_to_base64` — converts downloaded competitor image to base64 inline data. 5. `build_prompt` — assembles a careful prompt (handles partial text, labels, placements, CTA swaps, additional changes). 6. `generate_ad_image_prompt` → `generate_ad_image` — two HTTP Request nodes call generative endpoints (first to prepare prompt/content and then flash-image generation). These include inline image data and model-specific parameters. 7. `Wait` — allows async generation to complete (workflow uses a wait/webhook pattern). 8. `set_result` → `get_image` — extract image result, convert binary to file for downstream use (download or return to the requester). 9. `Sticky Note` — visual diagram / docs inside workflow.  \n## 📦 RESULT PROCESSING & OUTPUT This section processes the generated image and prepares the final result. |
| Sticky Note | Sticky Note | In-canvas documentation and project notes |  |  | # Image Ad Cloner AI Agent  This n8n workflow automates “cloning” a competitor ad style and re-generating it using your product image. A user uploads your product image and provides a competitor ad URL via a form trigger; the workflow downloads both images, converts them to inline/base64 parts, builds a detailed generation prompt, sends the request to a generative-vision model (Gemini-style endpoints), waits for the result, converts the output back to a file and returns it. Use-cases: rapid ad iteration, creative A/B variants, and scaled ad re-skins.  \n## How it works (step-by-step) 1. `form_trigger` — user submits: competitor image URL, their product file, and optional changes. 2. `convert_product_image_to_base64` — converts uploaded product file to base64 inline data. 3. `download_image` — fetches the competitor ad image from the provided URL. 4. `convert_ad_image_to_base64` — converts downloaded competitor image to base64 inline data. 5. `build_prompt` — assembles a careful prompt (handles partial text, labels, placements, CTA swaps, additional changes). 6. `generate_ad_image_prompt` → `generate_ad_image` — two HTTP Request nodes call generative endpoints (first to prepare prompt/content and then flash-image generation). These include inline image data and model-specific parameters. 7. `Wait` — allows async generation to complete (workflow uses a wait/webhook pattern). 8. `set_result` → `get_image` — extract image result, convert binary to file for downstream use (download or return to the requester). 9. `Sticky Note` — visual diagram / docs inside workflow.  \n👉 [Demo & Setup Video](https://drive.google.com/file/d/1PLFPHSXQ-DaNAKeBQv7G9SzaXT0c3hRt/view?usp=sharing)  \n👉 [Course](https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC) |
| Sticky Note1 | Sticky Note | In-canvas label for input and image preparation section |  |  | ## 📥  INPUT & IMAGE PREPARATION This section collects the required inputs and prepares images for AI processing. |
| Sticky Note2 | Sticky Note | In-canvas label for prompt engineering and AI generation section |  |  | ## 🧠  PROMPT ENGINEERING & AI GENERATION This section constructs the instructions and generates the new advertisement. |
| Sticky Note3 | Sticky Note | In-canvas label for result processing and output section |  |  | ## 📦 RESULT PROCESSING & OUTPUT This section processes the generated image and prepares the final result. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like **Image Ad Cloner AI Agent**.
   - Keep execution order as default or set it to `v1` if you want parity with the exported workflow.

2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Set title to: **Facebook Ad Thief Form**
   - Set description to:  
     `Upload your product image and submit a competitor image url for the ads you want to clone/spin for your product`
   - Add fields:
     1. Text field: **Competitor Image Link** — required
     2. File field: **Your Product Image** — required, single file only
     3. Text field: **Additional changes** — optional

3. **Add an Extract From File node for the uploaded product image**
   - Node type: **Extract From File**
   - Name: `convert_product_image_to_base64`
   - Operation: **Binary to Property**
   - Binary property name: `Your_Product_Image`
   - Connect: `form_trigger` → `convert_product_image_to_base64`

4. **Add an HTTP Request node to fetch the competitor image**
   - Node type: **HTTP Request**
   - Name: `download_image`
   - Method: `GET` is implicit/default unless changed
   - URL expression:  
     `{{ $('form_trigger').item.json['Competitor Image Link'] }}`
   - Ensure the node is configured to handle file/image responses correctly in your n8n version if needed
   - Connect: `convert_product_image_to_base64` → `download_image`

5. **Add another Extract From File node for the downloaded competitor image**
   - Node type: **Extract From File**
   - Name: `convert_ad_image_to_base64`
   - Operation: **Binary to Property**
   - Leave binary property at the downloaded file default, unless your HTTP Request node outputs to a custom binary property
   - Connect: `download_image` → `convert_ad_image_to_base64`

6. **Add a Set node to build the prompt**
   - Node type: **Set**
   - Name: `build_prompt`
   - Create a string field named `prompt`
   - Paste a prompt instructing the model to:
     - Compare the two images
     - Reproduce the competitor ad style
     - Replace competitor packaging with the user’s own product
     - Detect and replace visible branding, CTA text, and label fragments
     - Apply optional additional changes from the form
   - Include this expression inside the prompt for user customization:  
     `{{ $('form_trigger').item.json['Additional changes'] }}`
   - Connect: `convert_ad_image_to_base64` → `build_prompt`

7. **Create Gemini credentials**
   - The exported workflow uses header-based authentication.
   - In n8n, create an auth credential compatible with **HTTP Header Auth**.
   - Configure it with the header expected by your Gemini endpoint, typically an API key header such as:
     - Header name: `x-goog-api-key`
     - Value: your Gemini API key  
   - The export also references a bearer credential, but the node configuration uses `genericAuthType: httpHeaderAuth`. Reproduce the actual working auth method you use in your environment.
   - Verify which authentication method your Gemini API setup expects before testing.

8. **Add the first Gemini HTTP Request node**
   - Node type: **HTTP Request**
   - Name: `generate_ad_image_prompt`
   - Method: `POST`
   - URL:  
     `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-pro:generateContent`
   - Send body: enabled
   - Body type: JSON
   - Authentication: select your Gemini header auth credential
   - Enable retry on fail
   - Wait between retries: `5000` ms

   Use a JSON body shaped like this logically:
   - `contents[0].parts[0]` = prompt text
   - `contents[0].parts[1]` = product image inline data
   - `contents[0].parts[2]` = competitor image inline data

   Recreate the expressions:
   - Prompt text from `build_prompt`: `$json.prompt`
   - Product image data: `$node['convert_product_image_to_base64'].json.data`
   - Competitor image data: `$node['convert_ad_image_to_base64'].json.data`

   MIME types used in the original:
   - Product image: `image/png`
   - Competitor image: `image/jpeg`

   Connect: `build_prompt` → `generate_ad_image_prompt`

9. **Add the second Gemini HTTP Request node**
   - Node type: **HTTP Request**
   - Name: `generate_ad_image`
   - Method: `POST`
   - URL:  
     `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image:generateContent`
   - Send body: enabled
   - Body type: JSON
   - Authentication: same Gemini header auth credential
   - Enable retry on fail
   - Wait between retries: `5000` ms

   Configure the body to send:
   - The first Gemini response’s text as the new prompt
   - The same product image inline data
   - The same competitor image inline data

   Main expression for prompt text:
   - `$json.candidates.first().content.parts.first().text`

   Connect: `generate_ad_image_prompt` → `generate_ad_image`

10. **Add a Wait node**
    - Node type: **Wait**
    - Name: `Wait`
    - Leave default settings if you want to match the exported workflow
    - Connect: `generate_ad_image` → `Wait`

    Important note:
    - If your Gemini generation call is fully synchronous, you may be able to remove this node and connect directly to the parsing step.
    - If you keep it, make sure you understand how and when the workflow resumes.

11. **Add a Set node to extract the image result**
    - Node type: **Set**
    - Name: `set_result`
    - Add a string field named `image_result`
    - Use the expression:  
      `{{ $json.candidates[0].content.parts.filter(item => item.inlineData).first().inlineData.data }}`
    - Connect: `Wait` → `set_result`

12. **Add a Convert to File node**
    - Node type: **Convert to File**
    - Name: `get_image`
    - Operation: **To Binary**
    - Source property: `image_result`
    - Connect: `set_result` → `get_image`

13. **Optionally add sticky notes**
    - Add one documentation note describing the full workflow and resource links
    - Add section notes for:
      - Input & Image Preparation
      - Prompt Engineering & AI Generation
      - Result Processing & Output

14. **Test the form**
    - Open the form URL exposed by the Form Trigger
    - Submit:
      - A direct image URL for the competitor creative
      - A valid product image file
      - Optional change text

15. **Validate the HTTP responses**
    - Confirm `download_image` actually downloads an image, not HTML
    - Confirm both Extract nodes produce `json.data`
    - Confirm `generate_ad_image_prompt` returns `candidates[0].content.parts[0].text`
    - Confirm `generate_ad_image` returns an inline image in `candidates[0].content.parts`

16. **Harden the workflow for production**
    - Add MIME detection instead of hardcoding `image/png` and `image/jpeg`
    - Add an IF node after `download_image` to reject non-image responses
    - Add error handling for missing Gemini response fields
    - Optionally set output filename and MIME type in `get_image`
    - Consider removing or reconfiguring `Wait` if it is not needed

### Credential setup summary

- **Gemini / Google Generative Language API**
  - Likely best configured with **HTTP Header Auth**
  - Typical header:
    - `x-goog-api-key: <YOUR_API_KEY>`
  - Ensure the enabled models are accessible from your account/project
  - Verify that `gemini-2.5-pro` and `gemini-2.5-flash-image` are valid and available in your environment

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and does not expose reusable sub-workflow interfaces.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Demo & Setup Video | https://drive.google.com/file/d/1PLFPHSXQ-DaNAKeBQv7G9SzaXT0c3hRt/view?usp=sharing |
| Course | https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC |
| Internal branding used in the canvas | “Image Ad Cloner AI Agent” |
| Main caution | The workflow assumes competitor image URLs are directly downloadable images; many social media links are not |
| Main technical caution | The workflow hardcodes MIME types for uploaded/downloaded images, which may not match actual file types |
| Main operational caution | The Wait node is present but not clearly configured for a specific resume strategy, so it may require adjustment before production use |