Generate real-estate marketing images and videos with OpenAI and Google Drive

https://n8nworkflows.xyz/workflows/generate-real-estate-marketing-images-and-videos-with-openai-and-google-drive-14134


# Generate real-estate marketing images and videos with OpenAI and Google Drive

# 1. Workflow Overview

This workflow captures a real-estate marketing idea from a form, refines it with OpenAI into a higher-quality prompt, generates a marketing image, uploads it to Google Drive, and sends the requester an approval email. If the image is approved, the workflow continues into a second branch where the requester submits a video prompt, that prompt is refined, a video is generated, uploaded, shared, and logged back into Google Sheets. If the image is rejected, the workflow sends a rework email.

The workflow is designed for teams producing real-estate marketing assets such as listing visuals, brochure images, and promotional videos.

## 1.1 Input Reception and Logging
The workflow starts with a public n8n form that captures employee information, the initial creative prompt, and the requester’s email address. The submission is immediately appended to a Google Sheet for traceability.

## 1.2 AI Prompt Refinement for Image Generation
An AI Agent powered by an OpenAI chat model converts the raw idea into a more detailed photorealistic real-estate prompt suitable for DALL·E/OpenAI image generation.

## 1.3 Image Generation, Storage, and Sharing
The improved prompt is saved back to Google Sheets, then used to generate an image. The generated file is uploaded to Google Drive, the sheet is updated with the asset link, and the file is shared.

## 1.4 Approval Routing
The workflow emails the requester with the generated image link and refined prompt, then pauses until an approval/rejection response is received. A Switch node routes approved and rejected outcomes differently.

## 1.5 Video Prompt Collection and Video Generation
If approved, the workflow presents a second form to capture a video-generation prompt. That prompt is refined by another AI Agent, a file from Drive is fetched via HTTP, and the result is used as input for OpenAI video generation using the `sora-2-pro` model.

## 1.6 Video Upload and Final Logging
The generated video is uploaded to Google Drive, shared, and the Google Sheet is updated with the original video prompt, improved video prompt, and generated video link.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Initial Logging

**Overview:**  
This block receives the user’s real-estate idea from an n8n form and stores the raw submission in Google Sheets. It creates the base record used throughout the rest of the workflow.

**Nodes Involved:**  
- On form submission
- Append row in sheet

### Node Details

#### On form submission
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point trigger node that starts the workflow when a form is submitted.
- **Configuration choices:**
  - Form title: `Ideation Prompt Board`
  - Description: `Store and manage ideas for creative prompts.`
  - Required fields:
    - Employee Id
    - Employee Name
    - Initial Prompt
    - Your Mail Id
  - `Your Mail Id` is configured as an email field.
- **Key expressions or variables used:**  
  This node emits fields directly as JSON keys:
  - `$json['Employee Id']`
  - `$json['Employee Name']`
  - `$json['Initial Prompt']`
  - `$json['Your Mail Id']`
- **Input and output connections:**  
  - No input; this is a trigger.
  - Output → `Append row in sheet`
- **Version-specific requirements:**  
  Uses `typeVersion 2.2`, so the workflow expects a relatively recent n8n version with Form Trigger support.
- **Edge cases or potential failure types:**
  - Public form accessibility issues
  - Invalid email format
  - If the workflow is inactive, the production form endpoint may not accept submissions as expected
  - Field-name changes would break downstream expressions
- **Sub-workflow reference:** None

#### Append row in sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends the submitted data as a new row in Google Sheets.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet: `Data`
  - Sheet: `Sheet1` (`gid=0`)
  - Explicit column mapping:
    - `Mail Id` ← `{{ $json['Your Mail Id'] }}`
    - `Employee Id` ← `{{ $json['Employee Id'] }}`
    - `Employee Name` ← `{{ $json['Employee Name'] }}`
    - `Initial Prompt` ← `{{ $json['Initial Prompt'] }}`
- **Key expressions or variables used:**
  - `={{ $json['Your Mail Id'] }}`
  - `={{ $json['Employee Id'] }}`
  - `={{ $json['Employee Name'] }}`
  - `={{ $json['Initial Prompt'] }}`
- **Input and output connections:**  
  - Input ← `On form submission`
  - Output → `AI Agent`
- **Version-specific requirements:**  
  Uses Google Sheets node `typeVersion 4.7`.
- **Edge cases or potential failure types:**
  - OAuth token expiry
  - Missing spreadsheet or changed sheet permissions
  - Column name mismatch between node mapping and sheet header row
  - Duplicate initial prompts are allowed here, which later matters because later update nodes match by `Initial Prompt`
- **Sub-workflow reference:** None

---

## 2.2 Image Prompt Refinement

**Overview:**  
This block transforms the raw user idea into a production-grade photorealistic real-estate image prompt. It uses an AI Agent connected to an OpenAI chat model.

**Nodes Involved:**  
- AI Agent
- OpenAI Chat Model

### Node Details

#### AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain-based agent node that takes the initial prompt and generates a refined output.
- **Configuration choices:**
  - Prompt source: `{{ $json['Initial Prompt'] }}`
  - Prompt type: `define`
  - Strong system message instructing the model to act as `RealEstatePromptRefiner`
  - The system message asks the agent to:
    - infer property type, scene type, style, lighting, perspective, key features
    - apply defaults when details are missing
    - generate a realistic DALL·E-ready prompt
    - also generate `short_caption`
    - also generate `variants` (3 alternatives)
    - avoid unsafe/copyrighted material and unnecessary technical parameters
- **Key expressions or variables used:**
  - `={{ $json['Initial Prompt'] }}`
  - Output is later referenced as `$json.output`
- **Input and output connections:**  
  - Main input ← `Append row in sheet`
  - AI language model input ← `OpenAI Chat Model`
  - Main output → `Append or update row in sheet`
- **Version-specific requirements:**  
  Uses `typeVersion 2.2`. Requires the LangChain nodes package available in the n8n instance.
- **Edge cases or potential failure types:**
  - Model/credential failure
  - Prompt output structure may not be strictly machine-formatted; downstream assumes all useful content is in `output`
  - Very short or vague user prompts may produce generic results
  - Rate limiting from OpenAI
- **Sub-workflow reference:** None

#### OpenAI Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the LLM used by the `AI Agent`.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Responses API disabled
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Output (AI language model) → `AI Agent`
- **Version-specific requirements:**  
  Uses `typeVersion 1.3`.
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model availability changes
  - OpenAI quota exhaustion
- **Sub-workflow reference:** None

---

## 2.3 Store Refined Prompt and Generate Image

**Overview:**  
This block writes the improved prompt back to the Google Sheet, generates the image from that prompt, uploads it to Drive, and logs the Drive view link.

**Nodes Involved:**  
- Append or update row in sheet
- Generate an image2
- Upload file
- Append or update row in sheet1

### Node Details

#### Append or update row in sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Updates the existing row matching the original `Initial Prompt`, or appends if no match is found.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Match column: `Initial Prompt`
  - Mapped fields:
    - `Initial Prompt` ← value from `Append row in sheet`
    - `Improved Prompt` ← `{{ $json.output }}`
- **Key expressions or variables used:**
  - `={{ $('Append row in sheet').item.json['Initial Prompt'] }}`
  - `={{ $json.output }}`
- **Input and output connections:**  
  - Input ← `AI Agent`
  - Output → `Generate an image2`
- **Version-specific requirements:**  
  Google Sheets `typeVersion 4.7`
- **Edge cases or potential failure types:**
  - If multiple rows share the same `Initial Prompt`, update behavior may affect an unintended row
  - If the sheet structure changes, mapping can fail
- **Sub-workflow reference:** None

#### Generate an image2
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  OpenAI image-generation node.
- **Configuration choices:**
  - Resource: `image`
  - Prompt: `{{ $json['Improved Prompt'] }}`
  - Size: `1792x1024`
- **Key expressions or variables used:**
  - `={{ $json['Improved Prompt'] }}`
- **Input and output connections:**  
  - Input ← `Append or update row in sheet`
  - Output → `Upload file`
- **Version-specific requirements:**  
  Uses `typeVersion 2.1`.
- **Edge cases or potential failure types:**
  - The exact returned payload depends on node version and OpenAI API behavior
  - Invalid or excessively long prompt
  - Content policy refusal
  - Binary output expectations for downstream upload must match actual node output
- **Sub-workflow reference:** None

#### Upload file
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the generated image to a Google Drive folder.
- **Configuration choices:**
  - Drive: `My Drive`
  - Folder: `Ritz Media`
  - Operation appears to rely on default upload behavior
- **Key expressions or variables used:** None explicitly shown
- **Input and output connections:**  
  - Input ← `Generate an image2`
  - Output → `Append or update row in sheet1`
- **Version-specific requirements:**  
  Google Drive node `typeVersion 3`
- **Edge cases or potential failure types:**
  - Upload can fail if the previous node does not provide a binary file in the expected property
  - Drive permission or storage quota issues
  - Folder deletion or moved folder ID
- **Sub-workflow reference:** None

#### Append or update row in sheet1
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Logs the generated image’s Google Drive share/view link back to the sheet.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Match column: `Initial Prompt`
  - Mapped fields:
    - `Initial Prompt` ← from initial append node
    - `Image Generated Link` ← `{{ $json.webViewLink }}`
- **Key expressions or variables used:**
  - `={{ $('Append row in sheet').item.json['Initial Prompt'] }}`
  - `={{ $json.webViewLink }}`
- **Input and output connections:**  
  - Input ← `Upload file`
  - Output → `Wait`
- **Version-specific requirements:**  
  Google Sheets `typeVersion 4.7`
- **Edge cases or potential failure types:**
  - `webViewLink` may not exist until sharing/permissions are set depending on Drive behavior
  - Same duplicate-key problem on `Initial Prompt`
- **Sub-workflow reference:** None

---

## 2.4 Delay, Share, and Approval

**Overview:**  
This block introduces a delay, shares the Drive file publicly, sends an approval email, and waits for the requester’s decision.

**Nodes Involved:**  
- Wait
- Share file
- Send message and wait for response
- Switch
- Send a message

### Node Details

#### Wait
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses execution before the sharing step.
- **Configuration choices:**
  - Amount: `10`
  - No unit is explicitly visible in the JSON excerpt; in n8n this usually depends on the node UI default.
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input ← `Append or update row in sheet1`
  - Output → `Share file`
- **Version-specific requirements:**  
  `typeVersion 1.1`
- **Edge cases or potential failure types:**
  - Ambiguity around wait unit if recreated manually
  - Long-running executions may accumulate if many requests arrive
- **Sub-workflow reference:** None

#### Share file
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Sets file-sharing permissions on the uploaded image.
- **Configuration choices:**
  - Operation: `share`
  - File ID: `{{ $('Upload file').item.json.id }}`
  - Permissions:
    - Role: `writer`
    - Type: `anyone`
    - Allow file discovery: `true`
- **Key expressions or variables used:**
  - `={{ $('Upload file').item.json.id }}`
- **Input and output connections:**  
  - Input ← `Wait`
  - Output → `Send message and wait for response`
- **Version-specific requirements:**  
  Google Drive `typeVersion 3`
- **Edge cases or potential failure types:**
  - Permission model may not align with the sticky note’s “view-only” wording; this actually grants broad write-level access
  - Domain policies may block public sharing
  - Incorrect file ID if upload failed
- **Sub-workflow reference:** None

#### Send message and wait for response
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an email and pauses until a recipient approval/rejection action is received.
- **Configuration choices:**
  - Operation: `sendAndWait`
  - Recipient: requester email from form
  - Subject: `Approval for Image generated`
  - Message body includes:
    - improved image prompt from the sheet update node
    - Google Drive link from `Upload file`
  - Approval type: `double`
- **Key expressions or variables used:**
  - `={{ $('On form submission').item.json['Your Mail Id'] }}`
  - `{{ $('Append or update row in sheet').item.json['Improved Prompt'] }}`
  - `{{ $('Upload file').item.json.webViewLink }}`
- **Input and output connections:**  
  - Input ← `Share file`
  - Output → `Switch`
- **Version-specific requirements:**  
  Gmail node `typeVersion 2.1`
- **Edge cases or potential failure types:**
  - Gmail OAuth expiration
  - Approval callback/webhook accessibility
  - Email delivery/spam filtering
  - If `webViewLink` is inaccessible, the reviewer cannot inspect the asset
- **Sub-workflow reference:** None

#### Switch
- **Type and technical role:** `n8n-nodes-base.switch`  
  Routes the workflow depending on whether the reviewer approved or rejected the image.
- **Configuration choices:**
  - Rule 1: if `{{ $json.data.approved }}` is boolean `true` → output 0
  - Rule 2: if `{{ $json.data.approved }}` equals string `false` → output 1
- **Key expressions or variables used:**
  - `={{ $json.data.approved }}`
- **Input and output connections:**  
  - Input ← `Send message and wait for response`
  - Output 0 → `Form`
  - Output 1 → `Send a message`
- **Version-specific requirements:**  
  Switch node `typeVersion 3.2`
- **Edge cases or potential failure types:**
  - Mixed typing is notable: one rule checks boolean `true`, the other checks string `"false"`
  - If the Gmail approval node returns boolean `false` instead of string `"false"`, the rejection path may not match as intended
  - Unmatched values will silently end unless another default path exists
- **Sub-workflow reference:** None

#### Send a message
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends a rejection/rework notification.
- **Configuration choices:**
  - Recipient: requester email
  - Subject: `Failed the Image Test`
  - Message: `Please work on prompt again and generate a new image`
- **Key expressions or variables used:**
  - `={{ $('On form submission').item.json['Your Mail Id'] }}`
- **Input and output connections:**  
  - Input ← `Switch` rejection branch
  - No output connection
- **Version-specific requirements:**  
  Gmail node `typeVersion 2.1`
- **Edge cases or potential failure types:**
  - Gmail credential issues
  - The workflow ends after this message; there is no automatic loopback for regeneration
- **Sub-workflow reference:** None

---

## 2.5 Approved Video Intake and Prompt Refinement

**Overview:**  
If the image is approved, the workflow asks the user for a video-generation prompt, refines that prompt with another AI agent, then downloads the Drive asset for use in the video-generation request.

**Nodes Involved:**  
- Form
- AI Agent1
- OpenAI Chat Model1
- HTTP Request

### Node Details

#### Form
- **Type and technical role:** `n8n-nodes-base.form`  
  Presents a follow-up form during the workflow to collect a video prompt.
- **Configuration choices:**
  - One required field: `Prompt for video generation `
  - Note the trailing space in the field label; downstream expressions depend on that exact text.
- **Key expressions or variables used:**  
  Output key:
  - `$json['Prompt for video generation ']`
- **Input and output connections:**  
  - Input ← `Switch` approved branch
  - Output → `AI Agent1`
- **Version-specific requirements:**  
  `typeVersion 1`
- **Edge cases or potential failure types:**
  - Field label includes a trailing space, which is easy to break during manual recreation
  - If the form is not completed, execution stays paused
- **Sub-workflow reference:** None

#### AI Agent1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Refines the user’s video prompt before sending it to the video generation model.
- **Configuration choices:**
  - Prompt text: enhance and optimize the submitted prompt; return only the enhanced prompt
  - System message: `You are a helpful assistant`
  - Prompt type: `define`
- **Key expressions or variables used:**
  - `{{ $json['Prompt for video generation '] }}`
  - Output is later referenced as `$('AI Agent1').item.json.output`
- **Input and output connections:**  
  - Main input ← `Form`
  - AI language model input ← `OpenAI Chat Model1`
  - Main output → `HTTP Request`
- **Version-specific requirements:**  
  `typeVersion 2.2`
- **Edge cases or potential failure types:**
  - The output is unconstrained free text
  - Prompt refinement is generic, not domain-specialized like the first AI Agent
- **Sub-workflow reference:** None

#### OpenAI Chat Model1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Supplies the model for `AI Agent1`.
- **Configuration choices:**
  - Model: `gpt-4.1-mini`
  - Responses API disabled
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Output (AI language model) → `AI Agent1`
- **Version-specific requirements:**  
  `typeVersion 1.3`
- **Edge cases or potential failure types:**
  - API quota/authentication issues
- **Sub-workflow reference:** None

#### HTTP Request
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Downloads the image file from the Google Drive content link produced earlier.
- **Configuration choices:**
  - URL: `{{ $('Upload file').item.json.webContentLink }}`
- **Key expressions or variables used:**
  - `={{ $('Upload file').item.json.webContentLink }}`
- **Input and output connections:**  
  - Input ← `AI Agent1`
  - Output → `Generate a video`
- **Version-specific requirements:**  
  `typeVersion 4.3`
- **Edge cases or potential failure types:**
  - If the Drive file is not publicly accessible or requires auth, the request may fail
  - If the HTTP Request node is not configured to return binary in the expected property, the video node may not receive usable input
  - Google Drive direct-content links may behave differently depending on file type and permissions
- **Sub-workflow reference:** None

---

## 2.6 Video Generation, Upload, Share, and Logging

**Overview:**  
This block generates a video using OpenAI Sora, uploads the result to Drive, shares it, and writes video metadata back to the same Google Sheet row.

**Nodes Involved:**  
- Generate a video
- Upload file1
- Wait1
- Share file1
- Append or update row in sheet2

### Node Details

#### Generate a video
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`  
  OpenAI video-generation node using the Sora model.
- **Configuration choices:**
  - Resource: `video`
  - Model: `sora-2-pro`
  - Prompt: `{{ $('AI Agent1').item.json.output }}`
  - Size: `1792x1024`
  - Wait time: `7200`
  - Binary input reference property: `data`
- **Key expressions or variables used:**
  - `={{ $('AI Agent1').item.json.output }}`
- **Input and output connections:**  
  - Input ← `HTTP Request`
  - Output → `Upload file1`
- **Version-specific requirements:**  
  `typeVersion 2.1`
- **Edge cases or potential failure types:**
  - Sora/video models may not be available in all OpenAI accounts or regions
  - The `data` binary property must exist and contain the source file if required by the API path used here
  - Video generation can take a long time; timeout and quota issues are realistic
  - Cost can be significant
- **Sub-workflow reference:** None

#### Upload file1
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Uploads the generated video to the same Google Drive folder.
- **Configuration choices:**
  - Drive: `My Drive`
  - Folder: `Ritz Media`
- **Key expressions or variables used:** None explicitly shown
- **Input and output connections:**  
  - Input ← `Generate a video`
  - Output → `Wait1`
- **Version-specific requirements:**  
  Google Drive `typeVersion 3`
- **Edge cases or potential failure types:**
  - Large file uploads may fail or be slow
  - Binary property mismatch from video node
- **Sub-workflow reference:** None

#### Wait1
- **Type and technical role:** `n8n-nodes-base.wait`  
  Pauses before applying sharing permissions to the uploaded video.
- **Configuration choices:**
  - Amount: `10`
- **Key expressions or variables used:** None
- **Input and output connections:**  
  - Input ← `Upload file1`
  - Output → `Share file1`
- **Version-specific requirements:**  
  `typeVersion 1.1`
- **Edge cases or potential failure types:**
  - Same timing ambiguity as the earlier Wait node
- **Sub-workflow reference:** None

#### Share file1
- **Type and technical role:** `n8n-nodes-base.googleDrive`  
  Shares the generated video publicly.
- **Configuration choices:**
  - Operation: `share`
  - File ID: `{{ $('Upload file1').item.json.id }}`
  - Permissions:
    - Role: `writer`
    - Type: `anyone`
    - Allow file discovery: `true`
- **Key expressions or variables used:**
  - `={{ $('Upload file1').item.json.id }}`
- **Input and output connections:**  
  - Input ← `Wait1`
  - Output → `Append or update row in sheet2`
- **Version-specific requirements:**  
  Google Drive `typeVersion 3`
- **Edge cases or potential failure types:**
  - Same security concern: configured as public writer, not view-only
  - Domain policy may forbid public permissions
- **Sub-workflow reference:** None

#### Append or update row in sheet2
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Writes video-related metadata back to the sheet.
- **Configuration choices:**
  - Operation: `appendOrUpdate`
  - Match column: `Initial Prompt`
  - Fields:
    - `Initial Prompt` ← original initial prompt
    - `Video Generated Link` ← `{{ $('Upload file1').item.json.webContentLink }}`
    - `Video Initial Prompt` ← value from follow-up form
    - `Improved Video Prompt` ← AI-refined video prompt
- **Key expressions or variables used:**
  - `={{ $('Append row in sheet').item.json['Initial Prompt'] }}`
  - `={{ $('Upload file1').item.json.webContentLink }}`
  - `={{ $('Form').item.json['Prompt for video generation '] }}`
  - `={{ $('AI Agent1').item.json.output }}`
- **Input and output connections:**  
  - Input ← `Share file1`
  - No downstream connection
- **Version-specific requirements:**  
  Google Sheets `typeVersion 4.7`
- **Edge cases or potential failure types:**
  - Using `webContentLink` instead of `webViewLink` may lead to a download-oriented link rather than a preview link
  - Duplicate `Initial Prompt` values can still cause incorrect row updates
  - Trailing-space field label remains fragile
- **Sub-workflow reference:** None

---

## 2.7 Documentation and Annotation Nodes

**Overview:**  
These nodes are visual documentation only. They do not participate in execution but provide context, quick setup links, and labels for workflow blocks.

**Nodes Involved:**  
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5
- Sticky Note6
- Sticky Note7
- Sticky Note8
- Sticky Note9
- Sticky Note10
- Sticky Note11

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Canvas documentation block covering the full workflow.
- **Configuration choices:**  
  Contains a high-level description of the workflow, steps, and external resource links:
  - Demo & Setup Video: <https://drive.google.com/file/d/1a1n-XZIJkKsmBDjdNsG6Vg_AKv8AJxe5/view?usp=sharing>
  - Sheet Template: <https://docs.google.com/spreadsheets/d/1NEtQvs2hlcqmrFfSC6e1cReC25g2FEvxTiqdXOgtnjM/edit?usp=sharing>
  - Course: <https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC>
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases or potential failure types:** None
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type and technical role:** Sticky note labeling the idea-capture area.
- **Content:** `Idea Capture — Collect user idea + brief fields (style, property type, desired output).`
- **Connections:** None

#### Sticky Note2
- **Type and technical role:** Sticky note labeling the logging area.
- **Content:** `Record / Log — Save raw idea + metadata (user, timestamp) to Google Sheets for audit.`
- **Connections:** None

#### Sticky Note3
- **Type and technical role:** Sticky note labeling AI refinement.
- **Content:** `Prompt Refinement (AI Agent) — LLM converts raw idea → photorealistic prompt + captions + variants.`
- **Connections:** None

#### Sticky Note4
- **Type and technical role:** Sticky note labeling refined-prompt storage.
- **Content:** `Store Refined Prompt — Write back refined prompt and model metadata to the sheet.`
- **Connections:** None

#### Sticky Note5
- **Type and technical role:** Sticky note labeling asset generation.
- **Content:** `Asset Generation — Send refined prompt to image/video generator and receive files/URLs.`
- **Connections:** None

#### Sticky Note6
- **Type and technical role:** Sticky note labeling upload/share for image flow.
- **Content:** `Upload & Share — Upload assets to Google Drive and create view-only share links for reviewers.`
- **Connections:** None

#### Sticky Note7
- **Type and technical role:** Sticky note labeling approval flow.
- **Content:** `Approval Flow — Email the requester with asset + prompt; wait for approval or rejection.`
- **Connections:** None

#### Sticky Note8
- **Type and technical role:** Sticky note labeling decision routing.
- **Content:** `Decision & Routing — Route based on approval — approved → final storage/notifications; rejected → failure email.`
- **Connections:** None

#### Sticky Note9
- **Type and technical role:** Sticky note labeling video branch.
- **Content:** `Video Branch (Parallel) — Faster or separate form flow for 360°/sora-2-pro video generation, then upload & share.`
- **Connections:** None

#### Sticky Note10
- **Type and technical role:** Sticky note labeling upload/share for video flow.
- **Content:** `Upload & Share — Upload assets to Google Drive and create view-only share links for reviewers.`
- **Connections:** None

#### Sticky Note11
- **Type and technical role:** Sticky note labeling final video storage.
- **Content:** `Store Video Data`
- **Connections:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| On form submission | n8n-nodes-base.formTrigger | Entry trigger collecting employee, email, and initial image idea |  | Append row in sheet | Idea Capture — Collect user idea + brief fields (style, property type, desired output). |
| Append row in sheet | n8n-nodes-base.googleSheets | Append raw submission to Google Sheets | On form submission | AI Agent | Record / Log — Save raw idea + metadata (user, timestamp) to Google Sheets for audit. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Refine initial real-estate prompt into a richer image prompt | Append row in sheet; OpenAI Chat Model | Append or update row in sheet | Prompt Refinement (AI Agent) — LLM converts raw idea → photorealistic prompt + captions + variants. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for AI Agent |  | AI Agent | Prompt Refinement (AI Agent) — LLM converts raw idea → photorealistic prompt + captions + variants. |
| Append or update row in sheet | n8n-nodes-base.googleSheets | Save improved image prompt to Google Sheets | AI Agent | Generate an image2 | Store Refined Prompt — Write back refined prompt and model metadata to the sheet. |
| Generate an image2 | @n8n/n8n-nodes-langchain.openAi | Generate image from improved prompt | Append or update row in sheet | Upload file | Asset Generation — Send refined prompt to image/video generator and receive files/URLs. |
| Upload file | n8n-nodes-base.googleDrive | Upload generated image to Google Drive | Generate an image2 | Append or update row in sheet1 | Upload & Share — Upload assets to Google Drive and create view-only share links for reviewers. |
| Append or update row in sheet1 | n8n-nodes-base.googleSheets | Save generated image link to Google Sheets | Upload file | Wait | Upload & Share — Upload assets to Google Drive and create view-only share links for reviewers. |
| Wait | n8n-nodes-base.wait | Pause before sharing image file | Append or update row in sheet1 | Share file | Upload & Share — Upload assets to Google Drive and create view-only share links for reviewers. |
| Share file | n8n-nodes-base.googleDrive | Apply public sharing to image file | Wait | Send message and wait for response | Upload & Share — Upload assets to Google Drive and create view-only share links for reviewers. |
| Send message and wait for response | n8n-nodes-base.gmail | Email image and wait for approval/rejection | Share file | Switch | Approval Flow — Email the requester with asset + prompt; wait for approval or rejection. |
| Switch | n8n-nodes-base.switch | Route approved vs rejected outcomes | Send message and wait for response | Form; Send a message | Decision & Routing — Route based on approval — approved → final storage/notifications; rejected → failure email. |
| Send a message | n8n-nodes-base.gmail | Notify requester that image was rejected and needs rework | Switch |  | Decision & Routing — Route based on approval — approved → final storage/notifications; rejected → failure email. |
| Form | n8n-nodes-base.form | Collect follow-up video prompt after approval | Switch | AI Agent1 | Video Branch (Parallel) — Faster or separate form flow for 360°/sora-2-pro video generation, then upload & share. |
| AI Agent1 | @n8n/n8n-nodes-langchain.agent | Refine video-generation prompt | Form; OpenAI Chat Model1 | HTTP Request | Video Branch (Parallel) — Faster or separate form flow for 360°/sora-2-pro video generation, then upload & share. |
| OpenAI Chat Model1 | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for AI Agent1 |  | AI Agent1 | Video Branch (Parallel) — Faster or separate form flow for 360°/sora-2-pro video generation, then upload & share. |
| HTTP Request | n8n-nodes-base.httpRequest | Download prior asset from Drive for use in video generation | AI Agent1 | Generate a video | Video Branch (Parallel) — Faster or separate form flow for 360°/sora-2-pro video generation, then upload & share. |
| Generate a video | @n8n/n8n-nodes-langchain.openAi | Generate a video using Sora model | HTTP Request | Upload file1 | Video Branch (Parallel) — Faster or separate form flow for 360°/sora-2-pro video generation, then upload & share. |
| Upload file1 | n8n-nodes-base.googleDrive | Upload generated video to Google Drive | Generate a video | Wait1 | Upload & Share — Upload assets to Google Drive and create view-only share links for reviewers. |
| Wait1 | n8n-nodes-base.wait | Pause before sharing video file | Upload file1 | Share file1 | Upload & Share — Upload assets to Google Drive and create view-only share links for reviewers. |
| Share file1 | n8n-nodes-base.googleDrive | Apply public sharing to video file | Wait1 | Append or update row in sheet2 | Upload & Share — Upload assets to Google Drive and create view-only share links for reviewers. |
| Append or update row in sheet2 | n8n-nodes-base.googleSheets | Save video prompt data and video link to sheet | Share file1 |  | Store Video Data |
| Sticky Note | n8n-nodes-base.stickyNote | Overall workflow description and resource links |  |  | # 🏡 Automated Real-Estate Content Generation Workflow  / This n8n workflow automates turning short user ideas into production-ready real-estate marketing assets (photorealistic images and optional 360° videos). A form submission seeds a prompt board → an LLM refines prompts → the refined prompt is sent to an image/video generator → assets are uploaded to Google Drive and an approval email is sent so stakeholders can accept/reject outputs. / --- / ## ⚙️ How It Works (High Level) / 1. Trigger: User submits the ideation form (`On form submission`). / 2. Record: The raw idea is appended to Google Sheets (`Append row in sheet`). / 3. Refine: An AI agent (LangChain/OpenAI) refines the prompt into a photorealistic prompt and produces captions & variants (`AI Agent`). / 4. Store Refined Prompt: Update the sheet with the improved prompt (`Append or update row in sheet`). / 5. Generate Asset: Send improved prompt to OpenAI image/video generation nodes (`Generate an image2` / `Generate a video`). / 6. Upload & Share: Upload resulting files to Google Drive (`Upload file / Upload file1`) and set sharing permissions (`Share file / Share file1`). / 7. Approval Flow: Email the requester with the asset + improved prompt and wait for approval (`Send message and wait for response`). / 8. Decision: A `Switch` node routes approved outputs to final storage/notifications and rejected ones back for rework (or sends a failure email). / 9. Video Branch: Separate flow → refine → HTTP → video generator → upload → share → update sheet (parallel video pipeline). / Demo & Setup Video: <https://drive.google.com/file/d/1a1n-XZIJkKsmBDjdNsG6Vg_AKv8AJxe5/view?usp=sharing> / Sheet Template: <https://docs.google.com/spreadsheets/d/1NEtQvs2hlcqmrFfSC6e1cReC25g2FEvxTiqdXOgtnjM/edit?usp=sharing> / Course: <https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC> |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual label for idea capture section |  |  | Idea Capture — Collect user idea + brief fields (style, property type, desired output). |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual label for record/log section |  |  | Record / Log — Save raw idea + metadata (user, timestamp) to Google Sheets for audit. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual label for AI refinement section |  |  | Prompt Refinement (AI Agent) — LLM converts raw idea → photorealistic prompt + captions + variants. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Visual label for refined-prompt storage |  |  | Store Refined Prompt — Write back refined prompt and model metadata to the sheet. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Visual label for asset generation |  |  | Asset Generation — Send refined prompt to image/video generator and receive files/URLs. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Visual label for image upload/share |  |  | Upload & Share — Upload assets to Google Drive and create view-only share links for reviewers. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Visual label for approval flow |  |  | Approval Flow — Email the requester with asset + prompt; wait for approval or rejection. |
| Sticky Note8 | n8n-nodes-base.stickyNote | Visual label for decision routing |  |  | Decision & Routing — Route based on approval — approved → final storage/notifications; rejected → failure email. |
| Sticky Note9 | n8n-nodes-base.stickyNote | Visual label for video branch |  |  | Video Branch (Parallel) — Faster or separate form flow for 360°/sora-2-pro video generation, then upload & share. |
| Sticky Note10 | n8n-nodes-base.stickyNote | Visual label for video upload/share |  |  | Upload & Share — Upload assets to Google Drive and create view-only share links for reviewers. |
| Sticky Note11 | n8n-nodes-base.stickyNote | Visual label for video storage |  |  | Store Video Data |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like `Automated-Real-Estate-Content-Generation-Workflow`.
   - Ensure your n8n instance supports:
     - Form Trigger and Form nodes
     - Gmail OAuth2
     - Google Sheets OAuth2
     - Google Drive OAuth2
     - LangChain AI Agent nodes
     - OpenAI image/video generation nodes

2. **Prepare the external services**
   - Create or reuse a Google Sheet with headers at minimum:
     - `Employee Id`
     - `Employee Name`
     - `Initial Prompt`
     - `Improved Prompt`
     - `Image Generated Link`
     - `Mail Id`
     - `Video Initial Prompt`
     - `Improved Video Prompt`
     - `Video Generated Link`
   - Create a Google Drive folder for generated assets.
   - Configure credentials in n8n:
     - Google Sheets OAuth2
     - Google Drive OAuth2
     - OpenAI API
     - Gmail OAuth2

3. **Add the trigger node**
   - Add a **Form Trigger** node named `On form submission`.
   - Set:
     - Form title: `Ideation Prompt Board`
     - Description: `Store and manage ideas for creative prompts.`
   - Add required form fields:
     1. `Employee Id`
     2. `Employee Name`
     3. `Initial Prompt`
     4. `Your Mail Id` as email type

4. **Add the initial Google Sheets logging node**
   - Add a **Google Sheets** node named `Append row in sheet`.
   - Operation: `Append`
   - Select the target spreadsheet and `Sheet1`.
   - Map:
     - `Mail Id` → `{{ $json['Your Mail Id'] }}`
     - `Employee Id` → `{{ $json['Employee Id'] }}`
     - `Employee Name` → `{{ $json['Employee Name'] }}`
     - `Initial Prompt` → `{{ $json['Initial Prompt'] }}`
   - Connect `On form submission` → `Append row in sheet`.

5. **Add the first OpenAI chat model**
   - Add an **OpenAI Chat Model** node named `OpenAI Chat Model`.
   - Choose model `gpt-4.1-mini`.

6. **Add the image prompt refinement agent**
   - Add an **AI Agent** node named `AI Agent`.
   - Set prompt text to:
     - `{{ $json['Initial Prompt'] }}`
   - Use prompt type `Define`.
   - Paste the long system prompt that instructs the model to:
     - act as `RealEstatePromptRefiner`
     - create photorealistic real-estate prompts
     - include caption and variants
     - avoid unsafe/copyrighted elements
     - avoid technical generation flags
   - Connect:
     - `Append row in sheet` → `AI Agent`
     - `OpenAI Chat Model` → `AI Agent` via AI language model connection

7. **Store the improved image prompt**
   - Add a **Google Sheets** node named `Append or update row in sheet`.
   - Operation: `Append or Update`
   - Match column: `Initial Prompt`
   - Map:
     - `Initial Prompt` → `{{ $('Append row in sheet').item.json['Initial Prompt'] }}`
     - `Improved Prompt` → `{{ $json.output }}`
   - Connect `AI Agent` → `Append or update row in sheet`.

8. **Generate the image**
   - Add an **OpenAI** node named `Generate an image2`.
   - Resource: `Image`
   - Prompt: `{{ $json['Improved Prompt'] }}`
   - Image size: `1792x1024`
   - Connect `Append or update row in sheet` → `Generate an image2`.

9. **Upload the image to Google Drive**
   - Add a **Google Drive** node named `Upload file`.
   - Select your Drive and the destination folder, e.g. `Ritz Media`.
   - Configure it to upload the binary returned by the image-generation node.
   - Connect `Generate an image2` → `Upload file`.
   - Important: verify the binary property name produced by the image node and make sure the Drive node reads that same property.

10. **Write the image link back to the sheet**
    - Add a **Google Sheets** node named `Append or update row in sheet1`.
    - Operation: `Append or Update`
    - Match column: `Initial Prompt`
    - Map:
      - `Initial Prompt` → `{{ $('Append row in sheet').item.json['Initial Prompt'] }}`
      - `Image Generated Link` → `{{ $json.webViewLink }}`
    - Connect `Upload file` → `Append or update row in sheet1`.

11. **Insert a delay before sharing**
    - Add a **Wait** node named `Wait`.
    - Set amount to `10`.
    - Choose and document the unit explicitly when building manually, since the JSON excerpt does not show it clearly.
    - Connect `Append or update row in sheet1` → `Wait`.

12. **Share the image file**
    - Add a **Google Drive** node named `Share file`.
    - Operation: `Share`
    - File ID: `{{ $('Upload file').item.json.id }}`
    - Set permissions to:
      - Type: `Anyone`
      - Role: currently configured as `Writer`
      - File discovery: enabled
    - Connect `Wait` → `Share file`.
    - Recommended improvement: use `Reader` instead of `Writer` if you want actual reviewer-safe sharing.

13. **Send approval email and wait for response**
    - Add a **Gmail** node named `Send message and wait for response`.
    - Operation: `Send and Wait`
    - To: `{{ $('On form submission').item.json['Your Mail Id'] }}`
    - Subject: `Approval for Image generated`
    - Message:
      - `Image prompt: {{ $('Append or update row in sheet').item.json['Improved Prompt'] }}`
      - `Link: {{ $('Upload file').item.json.webViewLink }}`
    - Approval type: `double`
    - Connect `Share file` → `Send message and wait for response`.

14. **Route approval and rejection**
    - Add a **Switch** node named `Switch`.
    - Create two rules:
      - Rule 1: `{{ $json.data.approved }}` is boolean `true`
      - Rule 2: `{{ $json.data.approved }}` equals string `false`
    - Connect `Send message and wait for response` → `Switch`.
    - Recommended improvement: normalize the type and use boolean checks consistently.

15. **Add rejection email**
    - Add a **Gmail** node named `Send a message`.
    - To: `{{ $('On form submission').item.json['Your Mail Id'] }}`
    - Subject: `Failed the Image Test`
    - Message: `Please work on prompt again and generate a new image`
    - Connect the rejection output of `Switch` → `Send a message`.

16. **Add the approved-branch follow-up form**
    - Add a **Form** node named `Form`.
    - Add one required field exactly named:
      - `Prompt for video generation `
    - Keep the trailing space if you want exact compatibility with the existing expressions.
    - Connect the approved output of `Switch` → `Form`.

17. **Add the second OpenAI chat model**
    - Add an **OpenAI Chat Model** node named `OpenAI Chat Model1`.
    - Model: `gpt-4.1-mini`

18. **Add the video prompt refinement agent**
    - Add an **AI Agent** node named `AI Agent1`.
    - Prompt text:
      - `Enhance and optimize the following prompt for clarity, engagement, and effectiveness: {{ $json['Prompt for video generation '] }} . Just provide the enhanced prompt`
    - System message: `You are a helpful assistant`
    - Connect:
      - `Form` → `AI Agent1`
      - `OpenAI Chat Model1` → `AI Agent1` via AI language model input

19. **Download the previously generated image**
    - Add an **HTTP Request** node named `HTTP Request`.
    - URL: `{{ $('Upload file').item.json.webContentLink }}`
    - Configure it to download binary data if the video node expects binary input.
    - Connect `AI Agent1` → `HTTP Request`.

20. **Generate the video**
    - Add an **OpenAI** node named `Generate a video`.
    - Resource: `Video`
    - Model: `sora-2-pro`
    - Prompt: `{{ $('AI Agent1').item.json.output }}`
    - Size: `1792x1024`
    - Wait time: `7200`
    - Binary property name reference: `data`
    - Connect `HTTP Request` → `Generate a video`.
    - Important:
      - confirm your OpenAI account has access to the selected video model
      - ensure the HTTP Request node outputs binary in property `data`

21. **Upload the video to Google Drive**
    - Add a **Google Drive** node named `Upload file1`.
    - Select the same Drive and folder.
    - Configure binary upload from the video node.
    - Connect `Generate a video` → `Upload file1`.

22. **Add a second delay**
    - Add a **Wait** node named `Wait1`.
    - Set amount to `10`.
    - Connect `Upload file1` → `Wait1`.

23. **Share the video file**
    - Add a **Google Drive** node named `Share file1`.
    - Operation: `Share`
    - File ID: `{{ $('Upload file1').item.json.id }}`
    - Permissions:
      - Type: `Anyone`
      - Role: `Writer`
      - File discovery: enabled
    - Connect `Wait1` → `Share file1`.
    - Again, `Reader` would usually be safer than `Writer`.

24. **Log the video results**
    - Add a **Google Sheets** node named `Append or update row in sheet2`.
    - Operation: `Append or Update`
    - Match column: `Initial Prompt`
    - Map:
      - `Initial Prompt` → `{{ $('Append row in sheet').item.json['Initial Prompt'] }}`
      - `Video Generated Link` → `{{ $('Upload file1').item.json.webContentLink }}`
      - `Video Initial Prompt` → `{{ $('Form').item.json['Prompt for video generation '] }}`
      - `Improved Video Prompt` → `{{ $('AI Agent1').item.json.output }}`
    - Connect `Share file1` → `Append or update row in sheet2`.

25. **Add optional visual documentation**
    - Add sticky notes describing:
      - idea capture
      - record/log
      - prompt refinement
      - refined prompt storage
      - asset generation
      - upload/share
      - approval flow
      - decision/routing
      - video branch
      - video data storage
    - Add the broader project note with the external links if desired.

26. **Test the image path first**
    - Submit the initial form.
    - Confirm:
      - a sheet row is created
      - the AI prompt is stored
      - an image is generated
      - a Drive upload occurs
      - the approval email is sent

27. **Test the approval path**
    - Approve the email.
    - Confirm:
      - the video form opens
      - the refined video prompt is created
      - the HTTP download works
      - the video generation starts and completes
      - the video is uploaded/shared
      - the video fields update in the sheet

28. **Test the rejection path**
    - Reject the approval email.
    - Confirm `Send a message` dispatches the rework notice.

29. **Recommended hardening changes before production**
    - Replace `Initial Prompt` as a unique match key with a generated request ID
    - Change Drive sharing role from `writer` to `reader`
    - Normalize approval data typing in `Switch`
    - Make binary-property configuration explicit on image upload, HTTP download, and video upload nodes
    - Add error branches for OpenAI, Google Sheets, Drive, Gmail, and timeout failures

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Demo & Setup Video | <https://drive.google.com/file/d/1a1n-XZIJkKsmBDjdNsG6Vg_AKv8AJxe5/view?usp=sharing> |
| Sheet Template | <https://docs.google.com/spreadsheets/d/1NEtQvs2hlcqmrFfSC6e1cReC25g2FEvxTiqdXOgtnjM/edit?usp=sharing> |
| Course | <https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC> |
| Workflow branding / purpose | Automated real-estate content generation for photorealistic images and optional videos using OpenAI, Google Sheets, Google Drive, and Gmail |
| Important implementation caveat | Sticky notes describe shared links as “view-only”, but both Drive share nodes are configured with role `writer` for `anyone` |
| Important data-model caveat | Multiple update nodes use `Initial Prompt` as the match key; duplicate prompts may overwrite or update the wrong row |
| Important expression caveat | The field label `Prompt for video generation ` includes a trailing space and must be reproduced exactly unless all dependent expressions are updated |