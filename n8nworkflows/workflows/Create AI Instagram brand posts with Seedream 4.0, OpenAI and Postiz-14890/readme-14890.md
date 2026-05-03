Create AI Instagram brand posts with Seedream 4.0, OpenAI and Postiz

https://n8nworkflows.xyz/workflows/create-ai-instagram-brand-posts-with-seedream-4-0--openai-and-postiz-14890


# Create AI Instagram brand posts with Seedream 4.0, OpenAI and Postiz

# Workflow Analysis: Brand Content Automation for Instagram using Seedream 4.0

### 1. Workflow Overview
This workflow automates the end-to-end process of creating branded marketing content for Instagram. It transforms a text prompt and a reference image (such as a logo) into a professional social media post consisting of an AI-generated visual and a high-engagement caption.

The logic is divided into four primary functional blocks:
- **1.1 Input & Data Normalization:** Defines the creative prompt and reference images, ensuring data formats are compatible with subsequent APIs.
- **1.2 AI Copywriting (Parallel Branch A):** Uses a Large Language Model (LLM) to generate a platform-optimized Instagram caption.
- **1.3 AI Visual Generation (Parallel Branch B):** Interfaces with Seedream 4.0 via an asynchronous process to generate a branded image based on the input.
- **1.4 Media Integration & Publishing:** Merges the text and image assets, uploads the visual to Postiz, and schedules the final post to Instagram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Data Normalization
**Overview:** Sets the initial creative direction and prepares image URLs for the AI generator.
- **Nodes Involved:** `When clicking ‘Execute workflow’`, `Set params`, `Normalize`.
- **Node Details:**
    - `When clicking ‘Execute workflow’`: Manual trigger to initiate the process.
    - `Set params`: A **Set** node defining two global variables: `PROMPT` (the creative brief) and `IMAGES` (the URL of the reference logo/image).
    - `Normalize`: A **Code** node that converts the comma-separated string of images into a JSON array string, which is required by the Seedream API.
    - **Potential Failures:** Invalid image URLs in the `Set params` node will cause the image generation block to fail.

#### 2.2 AI Copywriting
**Overview:** Generates a professional, emoji-rich Instagram caption tailored for tech professionals.
- **Nodes Involved:** `Social Media Manager`, `OpenAI Chat Model`, `Get caption`.
- **Node Details:**
    - `Social Media Manager`: A **Chain LLM** node. It uses a system prompt to act as a social media expert, enforcing a specific JSON output format containing only the `instagram` caption.
    - `OpenAI Chat Model`: Provides the intelligence (GPT-4o-mini/gpt-5-mini) to the chain.
    - `Get caption`: A **Code** node that parses the LLM's JSON string output into an n8n object for downstream use.
    - **Edge Cases:** If the LLM returns markdown (e.g., ```json ... ```) instead of raw JSON, the `Get caption` node may throw a parsing error.

#### 2.3 AI Visual Generation
**Overview:** Generates a high-quality branded image using an asynchronous API call.
- **Nodes Involved:** `Seedream 4.0 Edit`, `Wait`, `Get ImageUrl`, `Get Image`.
- **Node Details:**
    - `Seedream 4.0 Edit`: An **HTTP Request** node calling the Kie AI API (`bytedance/seedream-v4-edit`). It sends the prompt and reference images.
    - `Wait`: A critical **Wait** node configured to "Resume on Webhook." Since image generation is asynchronous, n8n pauses here until Kie AI sends a POST request to the webhook URL to signal completion.
    - `Get ImageUrl`: A **Code** node that extracts the resulting image URL from the webhook's JSON response.
    - `Get Image`: An **HTTP Request** node that downloads the actual image binary from the generated URL.
    - **Potential Failures:** Network timeouts or the `Wait` node's webhook being unreachable (e.g., if n8n is behind a firewall without a public tunnel).

#### 2.4 Media Integration & Publishing
**Overview:** Combines the caption and image and publishes them via Postiz.
- **Nodes Involved:** `Merge`, `Upload IG Image`, `Instagram`.
- **Node Details:**
    - `Merge`: Combines the binary image data from the visual branch and the text caption from the copywriting branch.
    - `Upload IG Image`: An **HTTP Request** node that performs a `multipart-form-data` POST to Postiz's upload endpoint to host the image.
    - `Instagram`: A **Postiz** node that creates the final scheduled post, linking the uploaded image ID and the generated caption.
    - **Edge Cases:** Authentication expiration for Postiz or Instagram API limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking... | Manual Trigger | Start workflow | - | Set params | - |
| Set params | Set | Define prompt & image | Manual Trigger | Normalize | STEP 1 - Set vars |
| Normalize | Code | Format image URLs | Set params | Seedream, Social Media Mgr | - |
| Social Media Mgr | Chain LLM | Generate IG caption | Normalize | Get caption | STEP 2 - Social Media Manager |
| OpenAI Chat Model | LLM Chat | AI Intelligence | - | Social Media Mgr | - |
| Get caption | Code | Parse LLM output | Social Media Mgr | Merge | - |
| Seedream 4.0 Edit | HTTP Request | Trigger image creation | Normalize | Wait | STEP 3 - Generate image |
| Wait | Wait | Async webhook pause | Seedream 4.0 Edit | Get ImageUrl | - |
| Get ImageUrl | Code | Extract image link | Wait | Get Image | - |
| Get Image | HTTP Request | Download image binary | Get ImageUrl | Merge | - |
| Merge | Merge | Join image & caption | Get caption, Get Image | Upload IG Image | - |
| Upload IG Image | HTTP Request | Upload binary to Postiz | Merge | Instagram | - |
| Instagram | Postiz | Publish/Schedule post | Upload IG Image | - | STEP 4 - Postiz |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Setup Inputs
1. Create a **Manual Trigger** node.
2. Add a **Set** node named `Set params`. Define two string values:
   - `PROMPT`: Your descriptive image prompt.
   - `IMAGES`: A comma-separated list of image URLs.
3. Add a **Code** node (`Normalize`) to split the `IMAGES` string into a JSON array using `.split(',').map(url => url.trim())`.

#### Step 2: Build the Text Branch
1. Add a **Chain LLM** node (`Social Media Manager`). Set the prompt to define the persona (Instagram Expert) and require a JSON array output: `[{"instagram": "caption"}]`.
2. Connect an **OpenAI Chat Model** node to the Chain LLM (Model: `gpt-4o-mini` or similar).
3. Add a **Code** node (`Get caption`) to `JSON.parse()` the output of the LLM.

#### Step 3: Build the Visual Branch
1. Add an **HTTP Request** node (`Seedream 4.0 Edit`).
   - **Method:** POST
   - **URL:** `https://api.kie.ai/api/v1/jobs/createTask`
   - **Auth:** Header Auth (Bearer Token).
   - **Body:** JSON containing `model: "bytedance/seedream-v4-edit"`, `prompt`, `image_urls`, and `callBackUrl: "{{ $execution.resumeUrl }}"`.
2. Add a **Wait** node. Set it to **Resume: On Webhook Call**.
3. Add a **Code** node (`Get ImageUrl`) to parse the `resultUrls[0]` from the webhook response.
4. Add an **HTTP Request** node (`Get Image`) to GET the image binary from the `imageUrl`.

#### Step 4: Final Integration
1. Add a **Merge** node. Connect the `Get caption` node to Input 1 and the `Get Image` node to Input 2. Set mode to **Combine**.
2. Add an **HTTP Request** node (`Upload IG Image`).
   - **Method:** POST
   - **URL:** `https://api.postiz.com/public/v1/upload`
   - **Body:** `multipart-form-data` with the binary file from the Merge node.
3. Add a **Postiz** node (`Instagram`). Configure the post content using the caption from the Merge node and the image ID returned from the upload step.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Kie AI API Key for image generation | [kie.ai](https://kie.ai?ref=188b79f5cb949c9e875357ac098e1ff5) |
| Postiz API Key for scheduling | [postiz.pro/n3witalia](https://postiz.pro/n3witalia) |
| Workflow logic allows for parallel processing of text and visuals to reduce total execution time. | Architectural Note |