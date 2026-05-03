Post blog articles to LinkedIn and Twitter with OpenAI captions and UploadToURL

https://n8nworkflows.xyz/workflows/post-blog-articles-to-linkedin-and-twitter-with-openai-captions-and-uploadtourl-14970


# Post blog articles to LinkedIn and Twitter with OpenAI captions and UploadToURL

# Technical Documentation: Auto-Post Blog Content to LinkedIn & Twitter/X

This document provides a comprehensive technical analysis of the n8n workflow designed to automate the distribution of blog articles to social media platforms using AI-generated captions and image hosting.

---

### 1. Workflow Overview

The primary purpose of this workflow is to monitor a blog's RSS feed and automatically transform new entries into optimized social media posts for LinkedIn and Twitter/X. The workflow ensures that images are hosted on a public URL (required by social APIs) and that captions are tailored to the specific audience of each platform using OpenAI.

**Logical Blocks:**
- **1.1 Input Reception & Filtering:** Monitors the RSS feed and validates that the post contains necessary metadata (title and image).
- **1.2 Data Processing & Image Hosting:** Cleans the blog summary, downloads the cover image, and uploads it to a public hosting service.
- **1.3 AI Content Generation:** Uses OpenAI to create platform-specific copy based on the blog's content.
- **1.4 Social Distribution & Logging:** Publishes the content to LinkedIn and Twitter/X in parallel and logs the success.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering
**Overview:** This block acts as the gateway, triggering the workflow when new content is detected and ensuring the data is complete enough to proceed.

- **Nodes Involved:** `RSS Feed — Blog Monitor`, `IF — Has Title & Cover Image`.
- **Node Details:**
    - **RSS Feed — Blog Monitor** (RSS Feed Read Trigger): 
        - **Role:** Polling trigger.
        - **Configuration:** Set to poll every minute to detect new items.
        - **Output:** RSS item objects.
    - **IF — Has Title & Cover Image** (IF Node):
        - **Role:** Quality gate/Filter.
        - **Logic:** Checks if `title` is not empty AND `enclosure.url` is not empty.
        - **Failure Type:** If either is missing, the workflow stops (False path), preventing "broken" posts without images.

#### 2.2 Data Processing & Image Hosting
**Overview:** Extracts raw data from the RSS feed, cleans HTML noise, and ensures the image is accessible via a permanent public URL.

- **Nodes Involved:** `Code — Extract & Clean Blog Data`, `HTTP — Fetch Cover Image Binary`, `Upload a File`, `Code — Merge Hosted URL with Blog Data`.
- **Node Details:**
    - **Code — Extract & Clean Blog Data** (Code Node):
        - **Role:** Data sanitization.
        - **Logic:** Uses JavaScript to strip HTML tags from the summary, normalizes whitespace, and creates a URL slug.
    - **HTTP — Fetch Cover Image Binary** (HTTP Request):
        - **Role:** Image acquisition.
        - **Configuration:** Requests the `imageUrl` as a "File" (binary) response.
    - **Upload a File** (UploadToURL):
        - **Role:** Image hosting.
        - **Configuration:** Uploads the binary image to a public endpoint.
    - **Code — Merge Hosted URL with Blog Data** (Code Node):
        - **Role:** Data reconciliation.
        - **Logic:** Merges the original cleaned blog data with the new public URL returned by the upload service.

#### 2.3 AI Content Generation
**Overview:** Transforms the blog summary into high-engagement social media captions.

- **Nodes Involved:** `OpenAI — Generate Platform Captions`, `Code — Parse AI Captions Safely`.
- **Node Details:**
    - **OpenAI — Generate Platform Captions** (OpenAI Node):
        - **Role:** Text generation.
        - **Configuration:** Uses a Chat model to generate platform-specific captions.
    - **Code — Parse AI Captions Safely** (Code Node):
        - **Role:** JSON parsing and error handling.
        - **Logic:** Attempts to parse the OpenAI response as JSON. If the AI fails or returns malformed JSON, it applies a hard-coded "Fallback" template to ensure the post is still sent.

#### 2.4 Social Distribution & Logging
**Overview:** The final stage where the content is distributed to the target platforms.

- **Nodes Involved:** `LinkedIn — Publish Post with Image`, `Twitter/X — Publish Tweet`, `Code — Success Logger`.
- **Node Details:**
    - **LinkedIn — Publish Post with Image** (LinkedIn Node):
        - **Role:** Distribution.
        - **Configuration:** Uses the `linkedinCaption` and `hostedImageUrl`.
    - **Twitter/X — Publish Tweet** (Twitter Node):
        - **Role:** Distribution.
        - **Configuration:** Uses the `twitterCaption`.
    - **Code — Success Logger** (Code Node):
        - **Role:** Audit trail.
        - **Logic:** Prints a summary of the published post to the console for monitoring.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| RSS Feed — Blog Monitor | RSS Feed Trigger | Polls for new blog posts | (Start) | IF — Has Title & Cover Image | 🔁 Step 1 — RSS Trigger & Filter |
| IF — Has Title & Cover Image | IF | Validates image/title presence | RSS Feed — Blog Monitor | Code — Extract & Clean Blog Data | 🔁 Step 1 — RSS Trigger & Filter |
| Code — Extract & Clean Blog Data | Code | Sanitizes HTML and text | IF — Has Title & Cover Image | HTTP — Fetch Cover Image Binary | |
| HTTP — Fetch Cover Image Binary | HTTP Request | Downloads image binary | Code — Extract & Clean Blog Data | Upload a File | ☁️ Step 2 — Fetch & Upload Image |
| Upload a File | UploadToURL | Hosts image on public URL | HTTP — Fetch Cover Image Binary | Code — Merge Hosted URL with Blog Data | ☁️ Step 2 — Fetch & Upload Image |
| Code — Merge Hosted URL with Blog Data | Code | Combines binary URL with text | Upload a File | OpenAI — Generate Platform Captions | |
| OpenAI — Generate Platform Captions | OpenAI | Creates AI captions | Code — Merge Hosted URL with Blog Data | Code — Parse AI Captions Safely | 🤖 Step 3 — AI Caption + Posting |
| Code — Parse AI Captions Safely | Code | Parses JSON or provides fallback | OpenAI — Generate Platform Captions | LinkedIn / Twitter Nodes | 🤖 Step 3 — AI Caption + Posting |
| LinkedIn — Publish Post with Image | LinkedIn | Posts image and text | Code — Parse AI Captions Safely | Code — Success Logger | 🤖 Step 3 — AI Caption + Posting |
| Twitter/X — Publish Tweet | Twitter | Posts text/link | Code — Parse AI Captions Safely | Code — Success Logger | 🤖 Step 3 — AI Caption + Posting |
| Code — Success Logger | Code | Logs execution results | LinkedIn / Twitter Nodes | (End) | |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    - Create an **RSS Feed Read Trigger**.
    - Set the polling interval (e.g., every 15 minutes).
    - Enter your blog's RSS Feed URL.

2.  **Validation:**
    - Add an **IF Node**.
    - Condition 1: `title` is not empty.
    - Condition 2: `enclosure.url` (or equivalent image field) is not empty.
    - Connect the "True" output to the next step.

3.  **Data Cleaning:**
    - Add a **Code Node** (JavaScript).
    - Implement logic to remove HTML tags from the summary using a Regex like `/<[^>]+>/g`.
    - Map output fields: `title`, `link`, `imageUrl`, `summary`.

4.  **Image Hosting Pipeline:**
    - Add an **HTTP Request Node**. Set URL to `{{ $json.imageUrl }}` and response format to **File**.
    - Add an **UploadToURL Node**. Link your account credentials and map the binary input.
    - Add a **Code Node** to merge the resulting `hostedUrl` back into the main data object.

5.  **AI Integration:**
    - Add an **OpenAI Node**. Use the "Chat" resource.
    - Prompt the AI to return a JSON object containing two keys: `linkedin` (professional/long) and `twitter` (short/punchy).
    - Add a **Code Node** to `JSON.parse()` the result. Implement a `try-catch` block to provide default text if the AI fails.

6.  **Social Publishing:**
    - Add a **LinkedIn Node**. Configure the "Person" URN, set the text to the AI-generated LinkedIn caption, and set the media category to "Image" using the hosted URL.
    - Add a **Twitter Node**. Set the text to the AI-generated Twitter caption.
    - Connect both nodes in parallel from the parsing code node.

7.  **Finalization:**
    - Add a **Code Node** at the end to log the final output of the published posts for debugging.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Deduplication logic is built-in via n8n's item uniqueness check to avoid re-posting old articles. | Workflow Logic |
| Setup requires OpenAI, LinkedIn OAuth2, and Twitter/X OAuth1 credentials. | Prerequisites |
| Image hosting is mandatory because most social APIs require a publicly accessible URL rather than a binary upload for certain post types. | Technical Requirement |