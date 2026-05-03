Generate Instagram posts with OpenAI, RSS news, and auto image posting

https://n8nworkflows.xyz/workflows/generate-instagram-posts-with-openai--rss-news--and-auto-image-posting-15033


# Generate Instagram posts with OpenAI, RSS news, and auto image posting

# Technical Documentation: Generate Instagram Posts with OpenAI, RSS, and Auto-Posting

### 1. Workflow Overview

This workflow automates the entire pipeline of social media content creation: from news discovery to final publishing on Instagram. It leverages a rotating thematic strategy to ensure content variety, uses AI to synthesize news into engaging posts, generates a custom image via API, and implements a "human-in-the-loop" approval process via Telegram before the final post goes live.

The logic is divided into the following functional blocks:
*   **1.1 Trigger & Theme Selection:** Determines which news category to target based on the date.
*   **1.2 News Acquisition:** Fetches the most recent article from a specific RSS feed.
*   **1.3 AI Content Strategy:** Transforms the news article into an image prompt and a Spanish caption.
*   **1.4 Visual Asset Generation:** Generates an image via HTTP API, processes the binary data, and resizes it.
*   **1.5 Hosting & Review:** Uploads the image to cloud storage (S3) and sends it to Telegram for manual approval.
*   **1.6 Automated Publishing:** Posts the approved content to Instagram.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Theme Selection
**Overview:** Initiates the workflow and dynamically selects a content theme and RSS feed to prevent repetitive content.
*   **Nodes Involved:** `Schedule Trigger1`, `CODE TEMAS`
*   **Node Details:**
    *   `Schedule Trigger1`: Triggers every 2 days at 09:00.
    *   `CODE TEMAS` (Code Node): Uses JavaScript to rotate through an array of 5 themes (e.g., "AI for entrepreneurs", "AI risks"). It calculates the current day of the year to select the index.
    *   **Outputs:** Provides `feed_url`, `categoria`, `enfoque`, `audience`, `brand_style`, and `visual_direction`.

#### 2.2 News Acquisition
**Overview:** Retrieves and filters the latest news item based on the selected theme.
*   **Nodes Involved:** `RSS Noticias`, `CODE RSS`
*   **Node Details:**
    *   `RSS Noticias` (RSS Feed Read): Fetches data from the `feed_url` provided by the previous block.
    *   `CODE RSS` (Code Node): Sorts the incoming RSS items by date (`isoDate` or `pubDate`) in descending order and returns only the single most recent item.

#### 2.3 AI Content Strategy
**Overview:** Uses a Large Language Model to draft the creative elements of the post.
*   **Nodes Involved:** `AI Agent1`, `OpenAI Chat Model`, `Structured Output Parser1`
*   **Node Details:**
    *   `AI Agent1` (AI Agent): Acts as a social media strategist. It takes the news title, summary, and brand direction to create a JSON response.
    *   `OpenAI Chat Model`: Provides the intelligence (configured as `gpt-5-mini`).
    *   `Structured Output Parser1`: Ensures the AI returns a valid JSON with exactly two keys: `prompt_image` and `caption`.
    *   **Edge Cases:** If the AI fails to produce valid JSON, the output parser will throw an error.

#### 2.4 Visual Asset Generation
**Overview:** Converts the AI-generated prompt into a physical image file.
*   **Nodes Involved:** `HTTP NANOBANANA 3.1`, `CODE BINARY`, `Edit Image1`
*   **Node Details:**
    *   `HTTP NANOBANANA 3.1` (HTTP Request): Sends a POST request to an image generation API (Gemini/Google) using the `prompt_imagen`.
    *   `CODE BINARY` (Code Node): Extracts the base64 image data from the API response and converts it into an n8n binary object (`image`).
    *   `Edit Image1` (Edit Image): Resizes the generated image and converts it to JPEG format for Instagram compatibility.

#### 2.5 Hosting & Review
**Overview:** Handles the technical requirement of public image hosting and the human approval gate.
*   **Nodes Involved:** `Upload a file`, `Send a photo message1`, `Send message and wait for response1`, `Merge`
*   **Node Details:**
    *   `Upload a file` (S3): Uploads the binary image to an S3 bucket. This is critical because Instagram requires a public URL.
    *   `Send a photo message1` (Telegram): Sends the generated image to the user for review.
    *   `Send message and wait for response1` (Telegram): Sends the caption and waits for a "Double Approval" (Yes/No) response.
    *   `Merge`: Combines the S3 upload result (URL) and the Telegram approval status.

#### 2.6 Automated Publishing
**Overview:** The final execution step based on the approval result.
*   **Nodes Involved:** `If`, `Publish1`
*   **Node Details:**
    *   `If`: Checks if the Telegram response `approved` is `true`.
    *   `Publish1` (Instagram): Uses the Instagram API to post the `public_image_url` and the `caption`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger1 | Schedule Trigger | Workflow Entry | - | CODE TEMAS | Trigger: Runs automatically on a schedule. |
| CODE TEMAS | Code | Theme Selection | Schedule Trigger1 | RSS Noticias | News sources: Rotating list of RSS feeds. / Content direction: Defines brand style. |
| RSS Noticias | RSS Feed Read | News Fetching | CODE TEMAS | CODE RSS | News sources: Extracts latest news item. |
| CODE RSS | Code | Latest Item Filter | RSS Noticias | AI Agent1 | News sources: Returns the most recent item. |
| AI Agent1 | AI Agent | Content Creation | CODE RSS | HTTP NANOBANANA 3.1 | AI prompt generation: Transforms news into social content. |
| OpenAI Chat Model | LLM Chat | AI Brain | - | AI Agent1 | - |
| Structured Output Parser1| Output Parser | JSON Formatting | - | AI Agent1 | - |
| HTTP NANOBANANA 3.1 | HTTP Request | Image Generation | AI Agent1 | CODE BINARY | Image generation: Creates the final visual asset. |
| CODE BINARY | Code | B64 to Binary | HTTP NANOBANANA 3.1 | Edit Image1 | Image generation: Converts API response to binary. |
| Edit Image1 | Edit Image | Formatting | CODE BINARY | Send photo / Upload | Image generation: Resizes to final format. |
| Upload a file | S3 | Cloud Hosting | Edit Image1 | Merge | Image hosting: Creates public URL required by Instagram. |
| Send a photo message1 | Telegram | Review Part 1 | Edit Image1 | Send msg & wait | Approval step: Validates image quality. |
| Send message and wait...| Telegram | Review Part 2 | Send photo msg | Merge | Approval step: Validates caption and brand alignment. |
| Merge | Merge | Data Aggregation | Upload / Send msg | If | - |
| If | If | Approval Gate | Merge | Publish1 | Approval step: Control before going live. |
| Publish1 | Instagram | Final Posting | If | - | Publish: Automatically posts approved content. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Schedule Trigger** set to run every 2 days at 09:00.
2.  **Theme Selection:** Add a **Code Node** (`CODE TEMAS`) containing the JavaScript logic to rotate through the 5 provided RSS feeds based on the day of the year.
3.  **News Retrieval:**
    *   Connect an **RSS Feed Read** node using the expression `{{ $json.feed_url }}`.
    *   Connect a **Code Node** (`CODE RSS`) to sort the results and return only the first item `[items[0]]`.
4.  **AI Integration:**
    *   Create an **AI Agent** node. 
    *   Attach an **OpenAI Chat Model** (GPT-4o or GPT-5-mini).
    *   Attach a **Structured Output Parser** with the schema: `{"prompt_image": "string", "caption": "string"}`.
    *   Paste the detailed system prompt into the agent, mapping variables from `CODE TEMAS` and `CODE RSS`.
5.  **Image Pipeline:**
    *   Use an **HTTP Request** node (POST) to call the image generation API. Use the `prompt_image` from the AI Agent in the body.
    *   Add a **Code Node** (`CODE BINARY`) to map the API response `inlineData.data` to an n8n binary property named `image`.
    *   Add an **Edit Image** node to resize the image to 1080x1350 (JPEG).
6.  **Hosting & Human Review:**
    *   Connect an **S3 Node** (Upload) to store the image and output a `public_image_url`.
    *   Connect a **Telegram Node** (Send Photo) to send the binary image.
    *   Connect a **Telegram Node** (Send and Wait) to send the caption and wait for a boolean approval.
    *   Use a **Merge Node** (Combine all) to bring the S3 URL and Telegram approval together.
7.  **Final Execution:**
    *   Add an **If Node** to check if the approved status is `true`.
    *   Connect the `true` branch to an **Instagram Node** (Publish). Map the `caption` from the AI Agent and the `imageUrl` from the S3 node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Setup requires: OpenAI API Key, Google Gemini/Nanobanana API Key, Telegram Bot Token, S3 Credentials, and Instagram Business Account. | Setup Requirements |
| The image MUST be hosted on a public URL (S3/Cloudinary) because the Instagram API does not accept direct binary uploads. | Technical Constraint |
| Estimated setup time: 10–20 minutes. | Implementation Time |