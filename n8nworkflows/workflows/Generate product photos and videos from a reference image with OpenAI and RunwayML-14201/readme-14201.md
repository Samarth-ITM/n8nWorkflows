Generate product photos and videos from a reference image with OpenAI and RunwayML

https://n8nworkflows.xyz/workflows/generate-product-photos-and-videos-from-a-reference-image-with-openai-and-runwayml-14201


# Generate product photos and videos from a reference image with OpenAI and RunwayML

# 1. Workflow Overview

This workflow accepts a user-submitted reference product image and a creative brief through an n8n form, generates a new AI-enhanced product photo using OpenAI, creates a short product marketing video from that image using RunwayML, and emails both results to the user.

Typical use cases include:
- Marketing asset generation for e-commerce products
- Rapid creative concepting from a reference image
- Automated delivery of image and video assets to a requester
- Internal creative ops workflows where non-technical users submit jobs through a form

## 1.1 Input Reception and Job Initialization

The workflow begins with a form submission. The user provides:
- A reference image file
- A product image name
- A descriptive prompt for the desired visual outcome
- An email address for delivery

The original image is immediately uploaded to Google Drive as a backup/reference asset.

## 1.2 Prompt Engineering for Image Generation

An AI Agent powered by GPT-4.1 transforms the user’s plain-language product vision into a more polished, marketing-oriented image-generation prompt suitable for OpenAI image editing.

## 1.3 AI Image Creation

The workflow retrieves the uploaded reference image from Google Drive, sends it together with the generated prompt to OpenAI’s image editing API (`gpt-image-1`), then converts the returned base64 image payload into a binary file usable by downstream nodes.

## 1.4 Public Image Hosting and Video Generation

The generated image is uploaded to ImgBB to obtain a public URL. That URL is then submitted to RunwayML’s image-to-video API, requesting a short, clean product rotation video.

## 1.5 Video Status Polling and Delivery

Because RunwayML rendering is asynchronous, the workflow waits, polls the task status, and loops until the task reaches a terminal successful state. Once ready, it sends an email via Gmail containing:
- A viewer link to the generated image
- A link to the generated video output

---

# 2. Block-by-Block Analysis

## Block 1 — Form Intake and Reference Image Backup

### Overview
This block collects the user request and stores the original uploaded product image in Google Drive. It establishes the initial dataset used by the rest of the workflow and preserves the original file for traceability and reuse.

### Nodes Involved
- A.I Stunning Generator Interface
- Upload to Google Drive

### Node Details

#### 1. A.I Stunning Generator Interface
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point node that exposes a hosted n8n form and triggers the workflow upon submission.
- **Configuration choices:**
  - Form title: `Stelo Link Studio`
  - Attribution disabled
  - Custom CSS empty
  - Uses n8n-generated public form URL when active
  - Required fields:
    - `Image File` as a single file upload
    - `Product Image Name` as text
    - `Prompt A.I` as textarea
    - `Email` as email field
- **Key expressions or variables used:**  
  This node produces values later referenced as:
  - `$('A.I Stunning Generator Interface').item.json['Product Image Name']`
  - `$('A.I Stunning Generator Interface').item.json['Prompt A.I']`
  - `$('A.I Stunning Generator Interface').item.json.Email`
- **Input and output connections:**
  - No input; this is the trigger
  - Output → `Upload to Google Drive`
- **Version-specific requirements:**  
  Uses `typeVersion 2.2`; form field behavior depends on current n8n Form Trigger support.
- **Edge cases or potential failure types:**
  - Invalid or unsupported file type
  - Missing required field values
  - Workflow inactive, making the form unavailable
  - Large file uploads may hit instance or reverse-proxy limits
- **Sub-workflow reference:** None

#### 2. Upload to Google Drive
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the submitted original image file to Google Drive.
- **Configuration choices:**
  - Operation: upload file
  - File name: `{{ $json['Product Image Name'] }} (original)`
  - Target drive: `My Drive`
  - Folder ID: placeholder `YOUR_GOOGLE_DRIVE_FOLDER_ID`
  - Input binary field: `Image_File`
- **Key expressions or variables used:**
  - File name from incoming form field: `$json['Product Image Name']`
- **Input and output connections:**
  - Input ← `A.I Stunning Generator Interface`
  - Output → `AI Agent`
- **Version-specific requirements:**  
  Uses Google Drive node version 3.
- **Edge cases or potential failure types:**
  - Invalid Google Drive OAuth2 credentials
  - Folder ID not replaced or inaccessible
  - Binary field mismatch if form field name changes
  - Google API quota or permission errors
- **Sub-workflow reference:** None

---

## Block 2 — Prompt Engineering for the Product Image

### Overview
This block converts the user’s brief into a professional, image-model-ready prompt. The AI Agent is constrained by a detailed system prompt and uses GPT-4.1 as its language model.

### Nodes Involved
- Chat GPT 4.1
- AI Agent

### Node Details

#### 3. Chat GPT 4.1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the OpenAI chat model used by the AI Agent.
- **Configuration choices:**
  - Model: `gpt-4.1`
  - No advanced options set
- **Key expressions or variables used:** None directly in node parameters beyond model selection.
- **Input and output connections:**
  - No main input/output in workflow chain
  - Connected through `ai_languageModel` to `AI Agent`
- **Version-specific requirements:**  
  Uses node version `1.2`; requires compatible LangChain/OpenAI support in the installed n8n version.
- **Edge cases or potential failure types:**
  - Invalid OpenAI credentials
  - Model access not enabled on account
  - OpenAI API errors or rate limits
- **Sub-workflow reference:** None

#### 4. AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Generates a refined marketing-grade product photo prompt based on user inputs.
- **Configuration choices:**
  - Prompt type: defined in node
  - User text passed to agent:
    - Product name from form
    - Description from `Prompt A.I`
  - Detailed system message instructs the model to produce:
    - Hyper-realistic product photography prompts
    - Minimalist studio look
    - Professional lighting details
    - High-end marketing aesthetic
    - Single ready-to-use output prompt
- **Key expressions or variables used:**
  - `$('A.I Stunning Generator Interface').item.json['Product Image Name']`
  - `$('A.I Stunning Generator Interface').item.json['Prompt A.I']`
  - Downstream output reference: `$('AI Agent').item.json.output`
- **Input and output connections:**
  - Main input ← `Upload to Google Drive`
  - AI model input ← `Chat GPT 4.1`
  - Main output → `Download from Google Drive`
- **Version-specific requirements:**  
  Uses node version `1.9`; depends on the LangChain-based AI Agent implementation available in the n8n release.
- **Edge cases or potential failure types:**
  - OpenAI model authentication or quota errors
  - Prompt output too generic or inconsistent with user intent
  - Expression failure if form field labels are renamed
  - Unexpected agent output format if model behavior drifts
- **Sub-workflow reference:** None

---

## Block 3 — Image Retrieval, OpenAI Image Editing, and File Conversion

### Overview
This block downloads the original reference image from Google Drive, submits it to OpenAI’s image editing API with the AI-generated prompt, and converts the returned base64 response into a binary file that can be uploaded elsewhere.

### Nodes Involved
- Download from Google Drive
- OpenAi Create Image
- Convert to File

### Node Details

#### 5. Download from Google Drive
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Downloads the original file that was just uploaded, making it available as binary input for OpenAI.
- **Configuration choices:**
  - Operation: `download`
  - File ID dynamically pulled from the upload result
- **Key expressions or variables used:**
  - `$('Upload to Google Drive').item.json.id`
- **Input and output connections:**
  - Input ← `AI Agent`
  - Output → `OpenAi Create Image`
- **Version-specific requirements:**  
  Google Drive node version 3.
- **Edge cases or potential failure types:**
  - Uploaded file ID not found
  - Permissions mismatch between upload and download contexts
  - Transient Google Drive API failures
- **Sub-workflow reference:** None

#### 6. OpenAi Create Image
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls OpenAI’s image editing endpoint to generate a new product image from the uploaded reference image and the engineered prompt.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.openai.com/v1/images/edits`
  - Content type: multipart form-data
  - Authentication: generic HTTP header auth
  - Body fields:
    - `image` from binary field `data`
    - `prompt` from `AI Agent` output
    - `model` = `gpt-image-1`
- **Key expressions or variables used:**
  - `$('AI Agent').item.json.output`
- **Input and output connections:**
  - Input ← `Download from Google Drive`
  - Output → `Convert to File`
- **Version-specific requirements:**  
  HTTP Request node version `4.2`. OpenAI account must have access to `gpt-image-1`.
- **Edge cases or potential failure types:**
  - Invalid OpenAI bearer token
  - Missing image generation access for `gpt-image-1`
  - Unsupported file format or image constraints
  - API payload size issues
  - Endpoint returning an unexpected structure
- **Sub-workflow reference:** None

#### 7. Convert to File
- **Type and technical role:** `n8n-nodes-base.convertToFile`  
  Converts a base64 string from OpenAI’s response into binary data for upload to ImgBB.
- **Configuration choices:**
  - Operation: `toBinary`
  - Source property: `data[0].b64_json`
- **Key expressions or variables used:**
  - Reads from the OpenAI response path `data[0].b64_json`
- **Input and output connections:**
  - Input ← `OpenAi Create Image`
  - Output → `Get URL`
- **Version-specific requirements:**  
  Uses node version `1.1`.
- **Edge cases or potential failure types:**
  - OpenAI response missing `data[0].b64_json`
  - Unexpected response format if API changes
  - Invalid base64 payload
- **Sub-workflow reference:** None

---

## Block 4 — Public Hosting and RunwayML Video Creation

### Overview
This block uploads the generated image to ImgBB to obtain a public URL, then asks RunwayML to create a 5-second vertical marketing video based on that hosted image.

### Nodes Involved
- Get URL
- Create Video
- 60 Seconds

### Node Details

#### 8. Get URL
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Uploads the generated image to ImgBB and returns a public URL usable by external APIs.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.imgbb.com/1/upload`
  - Content type: multipart form-data
  - Authentication: generic query auth
  - Body field:
    - `image` from binary field `data`
- **Key expressions or variables used:**  
  No dynamic expression in the body beyond binary mapping.
- **Input and output connections:**
  - Input ← `Convert to File`
  - Output → `Create Video`
- **Version-specific requirements:**  
  HTTP Request node version `4.2`.
- **Edge cases or potential failure types:**
  - Invalid ImgBB API key
  - Wrong binary field name
  - File too large for ImgBB limits
  - Public hosting unavailable or rate-limited
- **Sub-workflow reference:** None

#### 9. Create Video
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Starts an asynchronous RunwayML image-to-video generation task.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.dev.runwayml.com/v1/image_to_video`
  - Authentication: generic HTTP header auth
  - Required custom header:
    - `X-Runway-Version: 2024-11-06`
  - Body parameters:
    - `promptImage` = `{{ $json.data.url }}`
    - `model` = `gen4_turbo`
    - `promptText` = long instruction describing slow elegant 360 product rotation
    - `duration` = `5`
    - `ratio` = `720:1280`
- **Key expressions or variables used:**
  - `$json.data.url` from ImgBB response
- **Input and output connections:**
  - Input ← `Get URL`
  - Output → `60 Seconds`
- **Version-specific requirements:**  
  HTTP Request node version `4.2`; the Runway API version header is mandatory for this configuration.
- **Edge cases or potential failure types:**
  - Invalid Runway bearer token
  - Missing or changed `X-Runway-Version`
  - Invalid public image URL
  - Unsupported model/version combination
  - Runway API task creation failure
- **Sub-workflow reference:** None

#### 10. 60 Seconds
- **Type and technical role:** `n8n-nodes-base.wait`  
  Adds an initial delay before checking task status, giving RunwayML time to begin rendering.
- **Configuration choices:**
  - Wait amount: `60` seconds
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input ← `Create Video`
  - Output → `HTTP Request`
- **Version-specific requirements:**  
  Wait node version `1.1`.
- **Edge cases or potential failure types:**
  - Workflow execution persistence must support waiting/resume
  - If n8n instance restarts without proper execution storage, waits may fail
- **Sub-workflow reference:** None

---

## Block 5 — Polling, Completion Check, and Email Delivery

### Overview
This block repeatedly polls RunwayML for task status. If the task is still running, the workflow waits 30 seconds and checks again. If the task reports success, the workflow sends the final email.

### Nodes Involved
- HTTP Request
- If
- Wait
- Send a message

### Node Details

#### 11. HTTP Request
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Polls the RunwayML task endpoint using the task ID returned from the creation call.
- **Configuration choices:**
  - Method defaults to GET
  - URL: `https://api.dev.runwayml.com/v1/tasks/{{ $('Create Video').item.json.id }}`
  - Authentication: generic HTTP header auth
  - Custom header:
    - `X-Runway-Version: 2024-11-06`
- **Key expressions or variables used:**
  - `$('Create Video').item.json.id`
- **Input and output connections:**
  - Input ← `60 Seconds`
  - Input ← `Wait` (loop path)
  - Output → `If`
- **Version-specific requirements:**  
  HTTP Request node version `4.2`.
- **Edge cases or potential failure types:**
  - Task ID missing or malformed
  - Runway authentication failure
  - Task endpoint returns error status
  - API version header mismatch
- **Sub-workflow reference:** None

#### 12. If
- **Type and technical role:** `n8n-nodes-base.if`  
  Branches based on the Runway task status.
- **Configuration choices:**
  - OR condition:
    - `status == SUCCEEDED`
    - `status == RUNNING`
  - True output is connected to `Send a message`
  - False output is connected to `Wait`
- **Key expressions or variables used:**
  - `{{ $json.status }}`
- **Input and output connections:**
  - Input ← `HTTP Request`
  - True output → `Send a message`
  - False output → `Wait`
- **Version-specific requirements:**  
  If node version `2.2`.
- **Edge cases or potential failure types:**
  - **Important logic issue:** this node treats both `SUCCEEDED` and `RUNNING` as true, so it can send the email even while the video is still rendering.
  - Any status besides `SUCCEEDED` or `RUNNING` goes to the false branch, including `FAILED`, `CANCELLED`, or unknown values, which then loops unnecessarily instead of handling failure cleanly.
- **Sub-workflow reference:** None

#### 13. Wait
- **Type and technical role:** `n8n-nodes-base.wait`  
  Waits 30 seconds between subsequent status checks.
- **Configuration choices:**
  - Wait amount: `30` seconds
- **Key expressions or variables used:** None
- **Input and output connections:**
  - Input ← `If` false branch
  - Output → `HTTP Request`
- **Version-specific requirements:**  
  Wait node version `1.1`.
- **Edge cases or potential failure types:**
  - Same persistence/resume considerations as other Wait nodes
  - Can create an endless polling loop if failures are not explicitly handled
- **Sub-workflow reference:** None

#### 14. Send a message
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the final delivery email to the requester.
- **Configuration choices:**
  - Recipient: form email field
  - Subject: `Marketing Material : {{ Product Image Name }}`
  - Plain-text email body includes:
    - ImgBB viewer link: `$('Get URL').item.json.data.url_viewer`
    - Runway video URL: `$json.output[0]`
  - Attribution disabled
- **Key expressions or variables used:**
  - `$('A.I Stunning Generator Interface').item.json.Email`
  - `$('A.I Stunning Generator Interface').item.json['Product Image Name']`
  - `$('Get URL').item.json.data.url_viewer`
  - `$json.output[0]`
- **Input and output connections:**
  - Input ← `If` true branch
  - No downstream output
- **Version-specific requirements:**  
  Gmail node version `2.1`; requires Gmail OAuth2.
- **Edge cases or potential failure types:**
  - Invalid Gmail OAuth2 credentials
  - Recipient email invalid or rejected
  - `$json.output[0]` may be missing if email is sent while Runway status is still `RUNNING`
  - Gmail sending quotas or account restrictions
- **Sub-workflow reference:** None

---

## Block 6 — Documentation and Embedded Operational Notes

### Overview
These sticky notes are not executable logic, but they are part of the workflow and document purpose, setup requirements, and credential expectations. They are useful for anyone maintaining or cloning the workflow.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3

### Node Details

#### 15. Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  High-level operational documentation for the whole workflow.
- **Configuration choices:**
  - Contains workflow summary
  - Lists credentials to replace
  - Warns not to change `X-Runway-Version`
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 16. Sticky Note1
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents the form trigger block.
- **Configuration choices:**
  - Explains form purpose and accepted file types
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 17. Sticky Note2
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents the image generation block.
- **Configuration choices:**
  - Explains prompt generation and OpenAI image creation
  - Notes Google Drive and OpenAI credential requirements
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### 18. Sticky Note3
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documents the video generation block.
- **Configuration choices:**
  - Explains ImgBB, RunwayML, polling, and Gmail delivery
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| A.I Stunning Generator Interface | n8n-nodes-base.formTrigger | Public form entry point collecting file, prompt, name, and email |  | Upload to Google Drive | ## A.I Stunning Generator Interface<br><br>This form is the entry point. When submitted, it kicks off the entire workflow. It collects the product image, a name for the file, an AI vision prompt, the user's email, and video preferences.<br>→ No credential needed — n8n generates a public URL when the workflow is active<br>→ Share the form URL with whoever needs to submit a job<br>→ File accepts: .jpeg .jpg .png .webm |
| Upload to Google Drive | n8n-nodes-base.googleDrive | Saves original uploaded image to Google Drive | A.I Stunning Generator Interface | AI Agent | ## Image Generation Pipeline<br><br>Upload → AI Agent → Download → OpenAI Create Image → Convert to File<br><br>These nodes take the submitted photo, use GPT-4.1 to craft a professional image prompt from the user's description, then send both the original image and that prompt to OpenAI's image editor (gpt-image-1) to generate a brand-new AI product photo. The result is converted to a usable file.<br>→ Google Drive OAuth2 — connects the Upload and Download nodes. Update the Folder ID to your own Drive folder (copy from the folder URL)<br>→ OpenAI credential (HTTP Header Auth, Bearer token) — connects to Chat GPT 4.1 node AND OpenAi Create Image node<br>→ gpt-image-1 requires image generation access enabled on your OpenAI account (check platform.openai.com) |
| Chat GPT 4.1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | Language model backing the AI Agent |  | AI Agent (AI model connection) | ## Image Generation Pipeline<br><br>Upload → AI Agent → Download → OpenAI Create Image → Convert to File<br><br>These nodes take the submitted photo, use GPT-4.1 to craft a professional image prompt from the user's description, then send both the original image and that prompt to OpenAI's image editor (gpt-image-1) to generate a brand-new AI product photo. The result is converted to a usable file.<br>→ Google Drive OAuth2 — connects the Upload and Download nodes. Update the Folder ID to your own Drive folder (copy from the folder URL)<br>→ OpenAI credential (HTTP Header Auth, Bearer token) — connects to Chat GPT 4.1 node AND OpenAi Create Image node<br>→ gpt-image-1 requires image generation access enabled on your OpenAI account (check platform.openai.com) |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Rewrites the user brief into a high-quality product photo prompt | Upload to Google Drive; Chat GPT 4.1 (AI model) | Download from Google Drive | ## Image Generation Pipeline<br><br>Upload → AI Agent → Download → OpenAI Create Image → Convert to File<br><br>These nodes take the submitted photo, use GPT-4.1 to craft a professional image prompt from the user's description, then send both the original image and that prompt to OpenAI's image editor (gpt-image-1) to generate a brand-new AI product photo. The result is converted to a usable file.<br>→ Google Drive OAuth2 — connects the Upload and Download nodes. Update the Folder ID to your own Drive folder (copy from the folder URL)<br>→ OpenAI credential (HTTP Header Auth, Bearer token) — connects to Chat GPT 4.1 node AND OpenAi Create Image node<br>→ gpt-image-1 requires image generation access enabled on your OpenAI account (check platform.openai.com) |
| Download from Google Drive | n8n-nodes-base.googleDrive | Downloads the stored original image for editing | AI Agent | OpenAi Create Image | ## Image Generation Pipeline<br><br>Upload → AI Agent → Download → OpenAI Create Image → Convert to File<br><br>These nodes take the submitted photo, use GPT-4.1 to craft a professional image prompt from the user's description, then send both the original image and that prompt to OpenAI's image editor (gpt-image-1) to generate a brand-new AI product photo. The result is converted to a usable file.<br>→ Google Drive OAuth2 — connects the Upload and Download nodes. Update the Folder ID to your own Drive folder (copy from the folder URL)<br>→ OpenAI credential (HTTP Header Auth, Bearer token) — connects to Chat GPT 4.1 node AND OpenAi Create Image node<br>→ gpt-image-1 requires image generation access enabled on your OpenAI account (check platform.openai.com) |
| OpenAi Create Image | n8n-nodes-base.httpRequest | Calls OpenAI image edit API with reference image and generated prompt | Download from Google Drive | Convert to File | ## Image Generation Pipeline<br><br>Upload → AI Agent → Download → OpenAI Create Image → Convert to File<br><br>These nodes take the submitted photo, use GPT-4.1 to craft a professional image prompt from the user's description, then send both the original image and that prompt to OpenAI's image editor (gpt-image-1) to generate a brand-new AI product photo. The result is converted to a usable file.<br>→ Google Drive OAuth2 — connects the Upload and Download nodes. Update the Folder ID to your own Drive folder (copy from the folder URL)<br>→ OpenAI credential (HTTP Header Auth, Bearer token) — connects to Chat GPT 4.1 node AND OpenAi Create Image node<br>→ gpt-image-1 requires image generation access enabled on your OpenAI account (check platform.openai.com) |
| Convert to File | n8n-nodes-base.convertToFile | Converts OpenAI base64 image output into binary | OpenAi Create Image | Get URL | ## Image Generation Pipeline<br><br>Upload → AI Agent → Download → OpenAI Create Image → Convert to File<br><br>These nodes take the submitted photo, use GPT-4.1 to craft a professional image prompt from the user's description, then send both the original image and that prompt to OpenAI's image editor (gpt-image-1) to generate a brand-new AI product photo. The result is converted to a usable file.<br>→ Google Drive OAuth2 — connects the Upload and Download nodes. Update the Folder ID to your own Drive folder (copy from the folder URL)<br>→ OpenAI credential (HTTP Header Auth, Bearer token) — connects to Chat GPT 4.1 node AND OpenAi Create Image node<br>→ gpt-image-1 requires image generation access enabled on your OpenAI account (check platform.openai.com) |
| Get URL | n8n-nodes-base.httpRequest | Uploads generated image to ImgBB and returns a public URL | Convert to File | Create Video | ## Video Generation Pipeline<br><br>Get URL → Create Video → 60s Wait → Poll Status → If Done → Send Email<br>These nodes take the AI-generated image, host it publicly via ImgBB, then submit it to RunwayML to render a product marketing video. Because video rendering takes time, the workflow loops every 30 seconds checking if it's done. Once complete, it emails the photo link and video link to the user.<br>→ ImgBB credential (HTTP Query Auth, key = YOUR_IMGBB_API_KEY) — connects to Get URL node. Free at imgbb.com<br>→ RunwayML credential (HTTP Header Auth, Bearer token) — connects to Create Video node AND the HTTP Request polling node<br>→ Gmail OAuth2 — connects to Send a Message node. Sends from whatever Google account you authorize<br>→ Do not change the X-Runway-Version header (2024-11-06) — required by the API |
| Create Video | n8n-nodes-base.httpRequest | Starts RunwayML image-to-video task | Get URL | 60 Seconds | ## Video Generation Pipeline<br><br>Get URL → Create Video → 60s Wait → Poll Status → If Done → Send Email<br>These nodes take the AI-generated image, host it publicly via ImgBB, then submit it to RunwayML to render a product marketing video. Because video rendering takes time, the workflow loops every 30 seconds checking if it's done. Once complete, it emails the photo link and video link to the user.<br>→ ImgBB credential (HTTP Query Auth, key = YOUR_IMGBB_API_KEY) — connects to Get URL node. Free at imgbb.com<br>→ RunwayML credential (HTTP Header Auth, Bearer token) — connects to Create Video node AND the HTTP Request polling node<br>→ Gmail OAuth2 — connects to Send a Message node. Sends from whatever Google account you authorize<br>→ Do not change the X-Runway-Version header (2024-11-06) — required by the API |
| 60 Seconds | n8n-nodes-base.wait | Initial delay before polling Runway task status | Create Video | HTTP Request | ## Video Generation Pipeline<br><br>Get URL → Create Video → 60s Wait → Poll Status → If Done → Send Email<br>These nodes take the AI-generated image, host it publicly via ImgBB, then submit it to RunwayML to render a product marketing video. Because video rendering takes time, the workflow loops every 30 seconds checking if it's done. Once complete, it emails the photo link and video link to the user.<br>→ ImgBB credential (HTTP Query Auth, key = YOUR_IMGBB_API_KEY) — connects to Get URL node. Free at imgbb.com<br>→ RunwayML credential (HTTP Header Auth, Bearer token) — connects to Create Video node AND the HTTP Request polling node<br>→ Gmail OAuth2 — connects to Send a Message node. Sends from whatever Google account you authorize<br>→ Do not change the X-Runway-Version header (2024-11-06) — required by the API |
| HTTP Request | n8n-nodes-base.httpRequest | Polls RunwayML task status | 60 Seconds; Wait | If | ## Video Generation Pipeline<br><br>Get URL → Create Video → 60s Wait → Poll Status → If Done → Send Email<br>These nodes take the AI-generated image, host it publicly via ImgBB, then submit it to RunwayML to render a product marketing video. Because video rendering takes time, the workflow loops every 30 seconds checking if it's done. Once complete, it emails the photo link and video link to the user.<br>→ ImgBB credential (HTTP Query Auth, key = YOUR_IMGBB_API_KEY) — connects to Get URL node. Free at imgbb.com<br>→ RunwayML credential (HTTP Header Auth, Bearer token) — connects to Create Video node AND the HTTP Request polling node<br>→ Gmail OAuth2 — connects to Send a Message node. Sends from whatever Google account you authorize<br>→ Do not change the X-Runway-Version header (2024-11-06) — required by the API |
| If | n8n-nodes-base.if | Branches based on Runway task status | HTTP Request | Send a message; Wait | ## Video Generation Pipeline<br><br>Get URL → Create Video → 60s Wait → Poll Status → If Done → Send Email<br>These nodes take the AI-generated image, host it publicly via ImgBB, then submit it to RunwayML to render a product marketing video. Because video rendering takes time, the workflow loops every 30 seconds checking if it's done. Once complete, it emails the photo link and video link to the user.<br>→ ImgBB credential (HTTP Query Auth, key = YOUR_IMGBB_API_KEY) — connects to Get URL node. Free at imgbb.com<br>→ RunwayML credential (HTTP Header Auth, Bearer token) — connects to Create Video node AND the HTTP Request polling node<br>→ Gmail OAuth2 — connects to Send a Message node. Sends from whatever Google account you authorize<br>→ Do not change the X-Runway-Version header (2024-11-06) — required by the API |
| Wait | n8n-nodes-base.wait | Re-check delay between Runway status polls | If | HTTP Request | ## Video Generation Pipeline<br><br>Get URL → Create Video → 60s Wait → Poll Status → If Done → Send Email<br>These nodes take the AI-generated image, host it publicly via ImgBB, then submit it to RunwayML to render a product marketing video. Because video rendering takes time, the workflow loops every 30 seconds checking if it's done. Once complete, it emails the photo link and video link to the user.<br>→ ImgBB credential (HTTP Query Auth, key = YOUR_IMGBB_API_KEY) — connects to Get URL node. Free at imgbb.com<br>→ RunwayML credential (HTTP Header Auth, Bearer token) — connects to Create Video node AND the HTTP Request polling node<br>→ Gmail OAuth2 — connects to Send a Message node. Sends from whatever Google account you authorize<br>→ Do not change the X-Runway-Version header (2024-11-06) — required by the API |
| Send a message | n8n-nodes-base.gmail | Emails the final image and video links to the user | If |  | ## Video Generation Pipeline<br><br>Get URL → Create Video → 60s Wait → Poll Status → If Done → Send Email<br>These nodes take the AI-generated image, host it publicly via ImgBB, then submit it to RunwayML to render a product marketing video. Because video rendering takes time, the workflow loops every 30 seconds checking if it's done. Once complete, it emails the photo link and video link to the user.<br>→ ImgBB credential (HTTP Query Auth, key = YOUR_IMGBB_API_KEY) — connects to Get URL node. Free at imgbb.com<br>→ RunwayML credential (HTTP Header Auth, Bearer token) — connects to Create Video node AND the HTTP Request polling node<br>→ Gmail OAuth2 — connects to Send a Message node. Sends from whatever Google account you authorize<br>→ Do not change the X-Runway-Version header (2024-11-06) — required by the API |
| Sticky Note | n8n-nodes-base.stickyNote | Embedded documentation for overall workflow |  |  | ## What This Workflow Does<br>A user submits a form with an existing product photo, a name, a vision prompt, and their email. The workflow generates a brand-new AI product image from that photo, turns it into a short marketing video, and emails both back to the user — fully automated after submission.<br><br>## How It Flows<br>The form submission kicks things off. The original photo is saved to Google Drive as a backup, then GPT-4.1 takes the user's vision prompt and rewrites it into a professional image generation prompt. That prompt plus the original photo go to OpenAI's image editor (gpt-image-1) to produce a new AI-generated product image. That image gets uploaded to ImgBB to create a public URL, which is then handed off to RunwayML to render a product marketing video. Because video rendering takes time, the workflow checks back every 30 seconds until it's done. Once complete, Gmail sends the user a link to the photo and the video.<br><br>## Credentials to Swap In<br>-OpenAI — HTTP Header Auth. Header name: Authorization. Value: Bearer YOUR_OPENAI_API_KEY. Connect this to the Chat GPT 4.1 node and the OpenAi Create Image node.<br>-Google Drive — Google Drive OAuth2. Authorize via the OAuth flow in n8n, then open the Upload to Google Drive node and update the Folder ID to your own Drive folder. You can find the folder ID in the URL when you open the folder in Google Drive.<br>-ImgBB — HTTP Query Auth. Parameter name: key. Value: YOUR_IMGBB_API_KEY. Free account at imgbb.com. Connect to the Get URL node.<br>-RunwayML — HTTP Header Auth. Header name: Authorization. Value: Bearer YOUR_RUNWAYML_API_KEY. Connect this to both the Create Video node and the HTTP Request polling node.<br>-Gmail — Gmail OAuth2. Authorize via the OAuth flow in n8n. The email will send from whichever Google account you connect. Connect to the Send a Message node.<br><br>## One Thing to Know<br>Do not change the X-Runway-Version header inside the RunwayML nodes. It must stay set to 2024-11-06 or the API call will fail. Also make sure your OpenAI account has image generation access enabled — gpt-image-1 requires it and you can verify this at platform.openai.com. |
| Sticky Note1 | n8n-nodes-base.stickyNote | Embedded documentation for the form trigger block |  |  | ## A.I Stunning Generator Interface<br><br>This form is the entry point. When submitted, it kicks off the entire workflow. It collects the product image, a name for the file, an AI vision prompt, the user's email, and video preferences.<br>→ No credential needed — n8n generates a public URL when the workflow is active<br>→ Share the form URL with whoever needs to submit a job<br>→ File accepts: .jpeg .jpg .png .webm |
| Sticky Note2 | n8n-nodes-base.stickyNote | Embedded documentation for image generation block |  |  | ## Image Generation Pipeline<br><br>Upload → AI Agent → Download → OpenAI Create Image → Convert to File<br><br>These nodes take the submitted photo, use GPT-4.1 to craft a professional image prompt from the user's description, then send both the original image and that prompt to OpenAI's image editor (gpt-image-1) to generate a brand-new AI product photo. The result is converted to a usable file.<br>→ Google Drive OAuth2 — connects the Upload and Download nodes. Update the Folder ID to your own Drive folder (copy from the folder URL)<br>→ OpenAI credential (HTTP Header Auth, Bearer token) — connects to Chat GPT 4.1 node AND OpenAi Create Image node<br>→ gpt-image-1 requires image generation access enabled on your OpenAI account (check platform.openai.com) |
| Sticky Note3 | n8n-nodes-base.stickyNote | Embedded documentation for video generation block |  |  | ## Video Generation Pipeline<br><br>Get URL → Create Video → 60s Wait → Poll Status → If Done → Send Email<br>These nodes take the AI-generated image, host it publicly via ImgBB, then submit it to RunwayML to render a product marketing video. Because video rendering takes time, the workflow loops every 30 seconds checking if it's done. Once complete, it emails the photo link and video link to the user.<br>→ ImgBB credential (HTTP Query Auth, key = YOUR_IMGBB_API_KEY) — connects to Get URL node. Free at imgbb.com<br>→ RunwayML credential (HTTP Header Auth, Bearer token) — connects to Create Video node AND the HTTP Request polling node<br>→ Gmail OAuth2 — connects to Send a Message node. Sends from whatever Google account you authorize<br>→ Do not change the X-Runway-Version header (2024-11-06) — required by the API |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.

2. **Add a Form Trigger node** named `A.I Stunning Generator Interface`.
   - Set the form title to `Stelo Link Studio`.
   - Disable attribution.
   - Add these fields:
     1. `Image File`  
        - Type: File  
        - Required: Yes  
        - Multiple files: No  
        - Accept: `.jpeg, .jpg, .png, .webm, .JPG`
     2. `Product Image Name`  
        - Type: Text  
        - Required: Yes  
        - Placeholder: `Name of New Image`
     3. `Prompt A.I`  
        - Type: Textarea  
        - Required: Yes  
        - Placeholder: `Explain your Vision. Think [Background, Content and Styles]`
     4. `Email`  
        - Type: Email  
        - Required: Yes  
        - Placeholder: `user@example.com`

3. **Add a Google Drive node** named `Upload to Google Drive`.
   - Connect `A.I Stunning Generator Interface` → `Upload to Google Drive`.
   - Configure Google Drive OAuth2 credentials.
   - Set:
     - Drive: `My Drive`
     - Folder ID: replace `YOUR_GOOGLE_DRIVE_FOLDER_ID` with your target folder
     - Input binary field: `Image_File`
     - File name: `{{ $json['Product Image Name'] }} (original)`

4. **Add an OpenAI Chat Model node** named `Chat GPT 4.1`.
   - Type: LangChain OpenAI Chat Model
   - Model: `gpt-4.1`
   - Configure OpenAI credentials.

5. **Add an AI Agent node** named `AI Agent`.
   - Connect `Upload to Google Drive` → `AI Agent`.
   - Connect `Chat GPT 4.1` to `AI Agent` through the AI language model port.
   - Set prompt type to define manually.
   - In the main text/prompt field, use:
     - `Product : {{ $('A.I Stunning Generator Interface').item.json['Product Image Name'] }}`
     - `Description : {{ $('A.I Stunning Generator Interface').item.json['Prompt A.I'] }}`
   - In the system message, paste the long instruction that tells the model to generate:
     - hyper-realistic
     - premium product photography
     - minimal clean studio-style prompts
     - one final direct output prompt

6. **Add a second Google Drive node** named `Download from Google Drive`.
   - Connect `AI Agent` → `Download from Google Drive`.
   - Set operation to `Download`.
   - File ID expression:
     - `{{ $('Upload to Google Drive').item.json.id }}`

7. **Add an HTTP Request node** named `OpenAi Create Image`.
   - Connect `Download from Google Drive` → `OpenAi Create Image`.
   - Configure generic HTTP Header Auth credential:
     - Header name: `Authorization`
     - Header value: `Bearer YOUR_OPENAI_API_KEY`
   - Set:
     - Method: `POST`
     - URL: `https://api.openai.com/v1/images/edits`
     - Content type: `Multipart Form-Data`
   - Add body parameters:
     1. `image`
        - Type: Form Binary Data
        - Input data field name: `data`
     2. `prompt`
        - Value: `{{ $('AI Agent').item.json.output }}`
     3. `model`
        - Value: `gpt-image-1`

8. **Add a Convert to File node** named `Convert to File`.
   - Connect `OpenAi Create Image` → `Convert to File`.
   - Set operation to `To Binary`.
   - Source property: `data[0].b64_json`

9. **Add an HTTP Request node** named `Get URL`.
   - Connect `Convert to File` → `Get URL`.
   - Configure generic HTTP Query Auth credential for ImgBB:
     - Query parameter name: `key`
     - Value: `YOUR_IMGBB_API_KEY`
   - Set:
     - Method: `POST`
     - URL: `https://api.imgbb.com/1/upload`
     - Content type: `Multipart Form-Data`
   - Add body parameter:
     - `image`
       - Type: Form Binary Data
       - Input data field name: `data`

10. **Add an HTTP Request node** named `Create Video`.
    - Connect `Get URL` → `Create Video`.
    - Configure generic HTTP Header Auth credential for RunwayML:
      - Header name: `Authorization`
      - Value: `Bearer YOUR_RUNWAYML_API_KEY`
    - Set:
      - Method: `POST`
      - URL: `https://api.dev.runwayml.com/v1/image_to_video`
      - Content type: regular body parameters
    - Enable headers and add:
      - `X-Runway-Version: 2024-11-06`
    - Add body parameters:
      1. `promptImage` = `{{ $json.data.url }}`
      2. `model` = `gen4_turbo`
      3. `promptText` = a detailed instruction asking for:
         - professional marketing video
         - smooth 360 rotation
         - realistic turntable effect
         - no cuts, zooms, or flashy transitions
         - full product always visible
      4. `duration` = `5`
      5. `ratio` = `720:1280`

11. **Add a Wait node** named `60 Seconds`.
    - Connect `Create Video` → `60 Seconds`.
    - Set wait amount to `60` seconds.

12. **Add an HTTP Request node** named `HTTP Request`.
    - Connect `60 Seconds` → `HTTP Request`.
    - This same node will also receive the loop-back from a second Wait node later.
    - Configure the same RunwayML HTTP Header Auth credential.
    - Set:
      - Method: `GET`
      - URL: `https://api.dev.runwayml.com/v1/tasks/{{ $('Create Video').item.json.id }}`
    - Add header:
      - `X-Runway-Version: 2024-11-06`

13. **Add an If node** named `If`.
    - Connect `HTTP Request` → `If`.
    - Create OR-based conditions on `{{ $json.status }}`:
      - equals `SUCCEEDED`
      - equals `RUNNING`

14. **Add a Wait node** named `Wait`.
    - Connect the **false** output of `If` → `Wait`.
    - Set wait amount to `30` seconds.

15. **Loop polling back**.
    - Connect `Wait` → `HTTP Request`.

16. **Add a Gmail node** named `Send a message`.
    - Connect the **true** output of `If` → `Send a message`.
    - Configure Gmail OAuth2 credentials.
    - Set:
      - To: `{{ $('A.I Stunning Generator Interface').item.json.Email }}`
      - Subject: `Marketing Material : {{ $('A.I Stunning Generator Interface').item.json['Product Image Name'] }}`
      - Email type: Text
      - Attribution off
      - Message body:
        - `Hey!`
        - `Here is your photo: {{ $('Get URL').item.json.data.url_viewer }}`
        - `and your video: {{ $json.output[0] }}`
        - `Cheers`
        - `Thank you for choosing Stelo Link`

17. **Optionally add sticky notes** to document the workflow:
    - One for the overall workflow
    - One for the form trigger area
    - One for the image pipeline
    - One for the video pipeline

18. **Activate the workflow** so the form URL becomes publicly accessible.

19. **Test with a sample submission**:
    - Upload a valid JPG or PNG
    - Provide a product name and descriptive prompt
    - Confirm Google Drive upload succeeds
    - Confirm OpenAI returns `data[0].b64_json`
    - Confirm ImgBB returns `data.url`
    - Confirm Runway returns a task ID
    - Confirm polling eventually produces video output

20. **Strongly recommended correction before production use**:
    - Change the `If` logic so only `SUCCEEDED` goes to the email branch.
    - Route `RUNNING` to the wait-and-poll loop.
    - Route `FAILED`, `CANCELLED`, or timeout states to a separate error notification path.
    - Without this fix, the workflow may email before the video is ready.

## Required Credentials

### OpenAI
Use either:
- OpenAI credential for the chat model node, and/or
- Generic HTTP Header Auth for the image edit request

For HTTP Header Auth:
- Header name: `Authorization`
- Header value: `Bearer YOUR_OPENAI_API_KEY`

### Google Drive
- Credential type: Google Drive OAuth2
- Must have access to the target folder in My Drive

### ImgBB
- Credential type: Generic HTTP Query Auth
- Query parameter: `key`
- Value: your ImgBB API key

### RunwayML
- Credential type: Generic HTTP Header Auth
- Header name: `Authorization`
- Header value: `Bearer YOUR_RUNWAYML_API_KEY`

### Gmail
- Credential type: Gmail OAuth2
- Sends from the connected Google account

## Sub-workflow Setup
This workflow does **not** invoke any sub-workflow and does not require any separate child workflow configuration.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| OpenAI image generation access is required for `gpt-image-1`. | https://platform.openai.com |
| ImgBB provides the public hosting step used before RunwayML video generation. | https://imgbb.com |
| Do not change the Runway API version header `X-Runway-Version: 2024-11-06`. | Required in both RunwayML HTTP Request nodes |
| Google Drive folder ID must be replaced with your own target folder ID. | Obtain from the Google Drive folder URL |
| Gmail sends the final results from the Google account authorized in n8n. | Gmail OAuth2 credential context |
| The current branching logic contains a delivery risk: `RUNNING` is treated as success. | Recommended to modify the `If` node before production use |