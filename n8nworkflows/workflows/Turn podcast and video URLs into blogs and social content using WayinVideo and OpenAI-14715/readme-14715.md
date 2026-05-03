Turn podcast and video URLs into blogs and social content using WayinVideo and OpenAI

https://n8nworkflows.xyz/workflows/turn-podcast-and-video-urls-into-blogs-and-social-content-using-wayinvideo-and-openai-14715


# Turn podcast and video URLs into blogs and social content using WayinVideo and OpenAI

# Workflow Technical Documentation: Podcast to Blog & Social Content Generator

## 1. Workflow Overview
This workflow automates the transformation of audio/video content (via URLs) into high-quality, structured blog posts and social content. It leverages **WayinVideo** for AI-driven transcription and summarization, **OpenAI** for creative writing, and **Google Sheets** as a content database.

The logic is divided into five primary functional blocks:
- **1.1 Input Reception:** Collects source URLs and branding via an n8n Form.
- **1.2 Video Summarization:** Submits the video to WayinVideo and retrieves AI-generated highlights.
- **1.3 Readiness Check:** A recursive loop that ensures the video processing is complete before proceeding.
- **1.4 AI Blog Writing:** A LangChain agent transforms raw summaries into a human-sounding blog post.
- **1.5 Output Storage:** Archives the final content and metadata into a Google Sheet.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception
**Overview:** Captures the necessary entry data from the user to initiate the process.
- **Nodes Involved:** `1. Form — Podcast URL + Brand Name1`
- **Node Details:**
    - **Type:** Form Trigger
    - **Configuration:** Defines two required fields: "Podcast / Video URL" and "Brand / Show Name".
    - **Input/Output:** Starts the workflow; outputs the form responses.
    - **Edge Cases:** Empty submissions are prevented by the "Required Field" setting.

### 2.2 Video Summarization
**Overview:** Interfaces with the WayinVideo API to request and retrieve content analysis.
- **Nodes Involved:** `2. WayinVideo — Submit Summary Request1`, `3. Wait — 40 Seconds1`, `4. WayinVideo — Get Summary Result1`
- **Node Details:**
    - **Submit Summary Request (HTTP Request):** 
        - **Role:** Sends a POST request to `/summaries` with the video URL and target language ("en").
        - **Auth:** Uses a Bearer Token in the header.
        - **Output:** Returns a unique ID for the summary request.
    - **Wait (Wait):**
        - **Role:** Pauses execution for 40 seconds to allow the external API to process the video.
    - **Get Summary Result (HTTP Request):**
        - **Role:** Performs a GET request to retrieve results using the ID from the submission node.
        - **Expressions:** Dynamically injects the ID: `{{ $('2. WayinVideo — Submit Summary Request1').item.json.data.id }}`.

### 2.3 Readiness Check
**Overview:** Implements a polling mechanism to ensure the AI has finished generating highlights.
- **Nodes Involved:** `5. IF — Highlights Ready?1`
- **Node Details:**
    - **Type:** IF Node
    - **Logic:** Checks if the `data.highlights` array is "Not Empty".
    - **Connections:** 
        - **True:** Proceeds to the AI Agent.
        - **False:** Loops back to the `3. Wait — 40 Seconds1` node.
    - **Edge Case:** **Infinite Loop Risk.** If the API fails or the URL is invalid, the workflow will loop indefinitely.

### 2.4 AI Blog Writing
**Overview:** Uses a LLM agent to convert structured highlights into a professional blog post.
- **Nodes Involved:** `6. AI Agent — Generate Blog Post1`, `7. OpenAI — GPT Chat Model1`, `9. Parser — Structured Blog Output1`
- **Node Details:**
    - **AI Agent (Agent):** 
        - **Role:** Orchestrates the prompt using data from WayinVideo.
        - **Prompt Logic:** Instructs the AI to write 550–700 words, avoid robotic tones, and use a specific structure (Title $\rightarrow$ Hook $\rightarrow$ Subheadings $\rightarrow$ Conclusion).
        - **Key Expression:** Maps highlights using `.map(h => h.desc + ': ' + h.events.map(e => e.desc).join(' | '))`.
    - **GPT Chat Model (Language Model):** 
        - **Role:** Provides the "brain" (GPT-4o-mini).
    - **Structured Blog Output (Output Parser):** 
        - **Role:** Forces the AI to return a valid JSON object containing `blog_title` and `blog_content`.

### 2.5 Output Storage
**Overview:** Finalizes the process by saving all generated data to a spreadsheet.
- **Nodes Involved:** `8. Google Sheets — Save Blog Content1`
- **Node Details:**
    - **Type:** Google Sheets (Append operation)
    - **Configuration:** Maps the following fields:
        - `Blog` $\leftarrow$ AI output content.
        - `Blog Title` $\leftarrow$ AI output title.
        - `Video URL` $\leftarrow$ Original form input.
        - `Key Summary` & `Highlights` & `Tags` $\leftarrow$ Data from WayinVideo.
    - **Auth:** Requires Google OAuth2 credentials.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1. Form — Podcast URL + Brand Name1 | Form Trigger | Data Entry | - | 2. WayinVideo... | User submits a podcast or YouTube video URL... |
| 2. WayinVideo — Submit Summary Request1 | HTTP Request | Request Analysis | 1. Form... | 3. Wait... | Sends the video URL to WayinVideo. |
| 3. Wait — 40 Seconds1 | Wait | Processing Delay | 2. WayinVideo... / 5. IF... | 4. WayinVideo... | Pauses to allow WayinVideo time to process. |
| 4. WayinVideo — Get Summary Result1 | HTTP Request | Fetch Analysis | 3. Wait... | 5. IF... | Fetches the AI-generated summary, highlights, and tags. |
| 5. IF — Highlights Ready?1 | IF | Logic Gate / Polling | 4. WayinVideo... | 6. AI Agent... / 3. Wait... | Checks if highlights were returned. |
| 6. AI Agent — Generate Blog Post1 | AI Agent | Content Creation | 5. IF... | 8. Google Sheets... | AI Agent reads summary and writes a structured blog. |
| 7. OpenAI — GPT Chat Model1 | LLM Chat | Intelligence Provider | - | 6. AI Agent... | (Connected as Model) |
| 8. Google Sheets — Save Blog Content1 | Google Sheets | Data Persistence | 6. AI Agent... | - | Saves the blog title, content, summary, highlights, and tags. |
| 9. Parser — Structured Blog Output1 | Output Parser | Format Enforcement | - | 6. AI Agent... | (Connected as Parser) |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Input Setup
1. Create a **Form Trigger** node.
2. Add two fields: `Podcast / Video URL` (String, Required) and `Brand / Show Name` (String, Required).

### Step 2: WayinVideo Integration
1. Create an **HTTP Request** node (`Submit Summary Request`).
    - **Method:** POST
    - **URL:** `https://wayinvideo-api.wayin.ai/api/v2/summaries`
    - **Headers:** `Authorization: Bearer [YOUR_TOKEN]`, `x-wayinvideo-api-version: v2`, `Content-Type: application/json`.
    - **Body:** JSON containing `video_url` (from Form) and `target_lang: "en"`.
2. Create a **Wait** node. Set the amount to `40` seconds.
3. Create a second **HTTP Request** node (`Get Summary Result`).
    - **Method:** GET
    - **URL:** `https://wayinvideo-api.wayin.ai/api/v2/summaries/results/{{ $json.data.id }}` (Reference the ID from the first HTTP node).
    - **Headers:** Same as the first request.

### Step 3: The Polling Loop
1. Create an **IF** node.
2. Condition: Check if `{{ $json.data.highlights }}` is **Not Empty**.
3. **True Path:** Connect to the AI Agent (Step 4).
4. **False Path:** Connect back to the **Wait** node.

### Step 4: AI Content Generation
1. Create an **AI Agent** node.
2. Connect a **Chat OpenAI** model node (Select `gpt-4o-mini`).
3. Connect a **Structured Output Parser** node. Define a JSON schema with `blog_title` and `blog_content`.
4. In the Agent's prompt, paste the professional writer instructions. Use expressions to map:
    - Title $\rightarrow$ `$('4. WayinVideo...').item.json.data.title`
    - Summary $\rightarrow$ `$('4. WayinVideo...').item.json.data.summary`
    - Highlights $\rightarrow$ Use the `.map()` function to join descriptions and events.

### Step 5: Data Storage
1. Create a **Google Sheets** node.
2. **Operation:** Append.
3. **Document:** Provide the URL of your target sheet.
4. **Mapping:** Map the AI output (`blog_title`, `blog_content`) and WayinVideo output (`summary`, `tags`, `highlights`) to the corresponding sheet columns.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Infinite Loop Warning** | If WayinVideo never returns highlights (invalid URL/private video), the loop between node 3 and 5 will continue indefinitely. Recommendation: Implement a retry counter. |
| **API Authentication** | Both WayinVideo nodes require a Bearer Token in the Authorization header. |
| **Output Length** | The AI is specifically constrained to write between 550 and 700 words. |