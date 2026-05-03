Generate cinematic videos with OpenAI Sora and uploadtourl hosting

https://n8nworkflows.xyz/workflows/generate-cinematic-videos-with-openai-sora-and-uploadtourl-hosting-13752


# Generate cinematic videos with OpenAI Sora and uploadtourl hosting

# 1. Workflow Overview

This workflow automates the creation of high-quality cinematic videos using the OpenAI Sora API and handles the subsequent hosting of the generated files. It serves as a middle-layer service that transforms simple text or image inputs into professional video assets for two specific use cases: E-commerce product reveals and Social Media "Remixes."

The workflow operates asynchronously to accommodate Sora's rendering time. It manages the full lifecycle of a video job: receiving the request, crafting optimized prompts, submitting the job, polling for completion, downloading the binary file, and uploading it to a CDN/Storage service.

**Logical Blocks:**
*   **1.1 Entry & Routing:** Receives incoming Webhook data and directs the flow based on the `jobType`.
*   **1.2 Cinematic E-commerce Pipeline:** Logic specialized for 4K product showcases and 360° rotations.
*   **1.3 Dynamic Social Media Remix Pipeline:** Logic specialized for vertical (9:16) or square (1:1) content with specific stylistic transformations.
*   **1.4 Async Polling Loop:** A recurring logic gate that checks Sora's API until the video generation is successful.
*   **1.5 Storage & Delivery:** Downloads the generated MP4 and pushes it to a remote URL via a PUT request.

---

# 2. Block-by-Step Analysis

### 2.1 Entry & Routing
This block acts as the API gateway for the workflow.

*   **Webhook – Receive Video Job**
    *   **Type:** Webhook
    *   **Role:** Entry point for POST requests at `/sora-video-job`.
    *   **Input/Output:** Receives JSON body (product info, styles); outputs data to the Switch.
*   **Route by Job Type**
    *   **Type:** Switch
    *   **Configuration:** Evaluates `{{ $json.body.jobType.toLowerCase() }}`.
    *   **Outputs:** Routes to "E-commerce" branch, "Remix" branch, or an Error branch if the type is unknown.
*   **Respond – Error**
    *   **Type:** Respond to Webhook
    *   **Role:** Returns a 400-level error response if the job type is invalid.

### 2.2 Cinematic E-commerce Pipeline
Handles high-end product video generation.

*   **Build E-commerce Prompt**
    *   **Type:** Code
    *   **Role:** Uses JavaScript to construct a detailed Sora prompt emphasizing studio lighting and 360° rotation.
    *   **Variables:** `productName`, `duration`, `style`.
*   **Sora – Submit E-commerce Job**
    *   **Type:** HTTP Request
    *   **Role:** `POST` to `https://api.openai.com/v1/video/generations`.
    *   **Auth:** Generic Header Auth (`Authorization: Bearer YOUR_KEY`).
*   **Store E-commerce Job ID**
    *   **Type:** Code
    *   **Role:** Normalizes the `generation_id` from the API response to ensure it can be used in the polling loop.

### 2.3 Dynamic Social Media Remix Pipeline
Handles vertical short-form content generation.

*   **Build Remix Prompt**
    *   **Type:** Code
    *   **Role:** Logic to determine aspect ratios (9:16 for Reels/TikTok, 1:1 for Feed) and applies stylistic "remix" effects.
*   **Sora – Submit Remix Job**
    *   **Type:** HTTP Request
    *   **Role:** Similar to the E-commerce submit node, but sends remix-specific parameters.
*   **Store Remix Job ID**
    *   **Type:** Code
    *   **Role:** Extracts the ID for the Remix loop.

### 2.4 Async Polling & Binary Handling
Since Sora is asynchronous, this block loops until the file is ready.

*   **Wait 20s (E-commerce/Remix)**
    *   **Type:** Wait
    *   **Role:** Pauses the workflow for 20 seconds to prevent API rate-limiting and allow rendering time.
*   **Sora – Poll Status (E-commerce/Remix)**
    *   **Type:** HTTP Request
    *   **Role:** `GET` request to `.../v1/video/generations/{generationId}`.
*   **Check Done (IF Nodes)**
    *   **Type:** IF
    *   **Condition:** Checks if `status` equals `succeeded`. If false, it loops back to the Wait node.
*   **Fetch Video (E-commerce/Remix)**
    *   **Type:** HTTP Request
    *   **Role:** Downloads the actual MP4 binary from the `video_url` provided in the successful Sora response.

### 2.5 Storage & Delivery
*   **Upload to URL (1 & 2)**
    *   **Type:** `n8n-nodes-uploadtourl.uploadToUrl`
    *   **Role:** Takes the binary MP4 and performs a `PUT` or `POST` to a pre-defined CDN or storage URL.
*   **Build Response (E-commerce/Remix)**
    *   **Type:** Code
    *   **Role:** Constructs the final JSON payload containing the `publicUrl` and metadata.
*   **Respond to Webhook**
    *   **Type:** Respond to Webhook
    *   **Role:** Sends the final successful JSON back to the original caller.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook – Receive Video Job | Webhook | API Entry Point | (None) | Route by Job Type | 1️⃣ Webhook Entry & Job Router |
| Route by Job Type | Switch | Logic Routing | Webhook | Build Prompt (Ecom/Remix), Error | 1️⃣ Webhook Entry & Job Router |
| Respond – Error | Respond to Webhook | Error Handling | Route by Job Type | (None) | 1️⃣ Webhook Entry & Job Router |
| Build E-commerce Prompt | Code | Prompt Engineering | Route by Job Type | Sora – Submit E-commerce Job | 2️⃣ Cinematic E-commerce Walkthrough |
| Sora – Submit E-commerce Job | HTTP Request | API Trigger | Build E-commerce Prompt | Store E-commerce Job ID | 2️⃣ Cinematic E-commerce Walkthrough |
| Store E-commerce Job ID | Code | Data Normalization | Sora – Submit E-commerce Job | Wait 20s – E-commerce | 2️⃣ Cinematic E-commerce Walkthrough |
| Wait 20s – E-commerce | Wait | Loop Delay | Store E-commerce Job ID, Check E-commerce Done | Sora – Poll E-commerce Status | 2️⃣ Cinematic E-commerce Walkthrough |
| Sora – Poll E-commerce Status | HTTP Request | API Polling | Wait 20s – E-commerce | Check E-commerce Done | 2️⃣ Cinematic E-commerce Walkthrough |
| Check E-commerce Done | IF | Status Check | Sora – Poll E-commerce Status | Fetch E-commerce Video, Wait 20s | 2️⃣ Cinematic E-commerce Walkthrough |
| Fetch E-commerce Video | HTTP Request | Binary Download | Check E-commerce Done | Upload to URL | 2️⃣ Cinematic E-commerce Walkthrough |
| Upload to URL | Upload to URL | Cloud Storage | Fetch E-commerce Video | Build E-commerce Response | 2️⃣ Cinematic E-commerce Walkthrough |
| Build E-commerce Response | Code | Result Mapping | Upload to URL | Respond to Webhook – E-commerce | 2️⃣ Cinematic E-commerce Walkthrough |
| Respond to Webhook – E-commerce | Respond to Webhook | API Response | Build E-commerce Response | (None) | 2️⃣ Cinematic E-commerce Walkthrough |
| Build Remix Prompt | Code | Prompt Engineering | Route by Job Type | Sora – Submit Remix Job | 3️⃣ Dynamic Social Media Remix |
| Sora – Submit Remix Job | HTTP Request | API Trigger | Build Remix Prompt | Store Remix Job ID | 3️⃣ Dynamic Social Media Remix |
| Store Remix Job ID | Code | Data Normalization | Sora – Submit Remix Job | Wait 20s – Remix | 3️⃣ Dynamic Social Media Remix |
| Wait 20s – Remix | Wait | Loop Delay | Store Remix Job ID, Check Remix Done | Sora – Poll Remix Status | 3️⃣ Dynamic Social Media Remix |
| Sora – Poll Remix Status | HTTP Request | API Polling | Wait 20s – Remix | Check Remix Done | 3️⃣ Dynamic Social Media Remix |
| Check Remix Done | IF | Status Check | Sora – Poll Remix Status | Fetch Remix Video, Wait 20s | 3️⃣ Dynamic Social Media Remix |
| Fetch Remix Video | HTTP Request | Binary Download | Check Remix Done | Upload to URL1 | 3️⃣ Dynamic Social Media Remix |
| Upload to URL1 | Upload to URL | Cloud Storage | Fetch Remix Video | Build Remix Response | 3️⃣ Dynamic Social Media Remix |
| Build Remix Response | Code | Result Mapping | Upload to URL1 | Respond to Webhook – Remix | 3️⃣ Dynamic Social Media Remix |
| Respond to Webhook – Remix | Respond to Webhook | API Response | Build Remix Response | (None) | 3️⃣ Dynamic Social Media Remix |

---

# 4. Reproducing the Workflow from Scratch

1.  **Entry Point:** Create a **Webhook** node set to `POST` with the path `sora-video-job`. Set Response Mode to "When Last Node Finishes" (or via Respond to Webhook nodes).
2.  **Routing:** Add a **Switch** node. Create two rules using String comparison:
    *   If `jobType` equals `ecommerce` -> Output 0.
    *   If `jobType` equals `remix` -> Output 1.
3.  **Prompt Engineering (Branch 1):** Add a **Code** node. Use JS to take the webhook body and return a template string formatted for high-end product videos.
4.  **Sora Integration:** Add an **HTTP Request** node.
    *   Method: `POST`.
    *   URL: `https://api.openai.com/v1/video/generations`.
    *   Authentication: Header Auth (`Authorization: Bearer [KEY]`).
    *   Body: Pass the prompt from the Code node.
5.  **The Polling Loop:**
    *   Add a **Wait** node set to 20 seconds.
    *   Add an **HTTP Request** node (`GET`) calling the Sora status endpoint using the `id` from the previous step.
    *   Add an **IF** node to check if `status` is `succeeded`.
    *   Connect the **False** output of the IF node back to the **Wait** node.
6.  **Binary Handling:** Connect the **True** output of the IF node to a new **HTTP Request** node. Use the `video_url` from the IF node's data. Set the response type to "File" (Binary).
7.  **Cloud Storage:** Add the **Upload to URL** node. Configure your credentials for your CDN/storage provider. This node will automatically pick up the binary file from the previous node.
8.  **Final Response:** Add a **Code** node to format the final JSON output (including the new public URL) and connect it to a **Respond to Webhook** node.
9.  **Branch 2:** Repeat steps 3–8 for the "Remix" branch, adjusting the prompt logic and aspect ratios in the Code node.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **OpenAI Sora API Access** | Requires Tier 4/5 or Enterprise access at [platform.openai.com](https://platform.openai.com) |
| **API Endpoints** | Submit: `/v1/video/generations` \| Poll: `/v1/video/generations/{id}` |
| **Credential Setup** | Use "Header Auth" with name `Authorization` and value `Bearer YOUR_OPENAI_KEY` |
| **E-commerce Use Case** | Best for Shopify, Meesho, or Amazon product listings. |
| **Remix Use Case** | Optimized for TikTok, Reels, and YouTube Shorts (9:16 aspect ratio). |