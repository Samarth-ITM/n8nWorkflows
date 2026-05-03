Publish NASA images with watermarks to WordPress, Facebook, Instagram, Telegram and LinkedIn

https://n8nworkflows.xyz/workflows/publish-nasa-images-with-watermarks-to-wordpress--facebook--instagram--telegram-and-linkedin-13767


# Publish NASA images with watermarks to WordPress, Facebook, Instagram, Telegram and LinkedIn

# Reference Documentation: Automated NASA Content Publisher

This document provides a technical breakdown of an n8n workflow designed to monitor NASA news, process associated imagery with custom watermarking, and publish content across a multi-platform ecosystem including WordPress, Facebook, Instagram, Telegram, and LinkedIn.

---

### 1. Workflow Overview

The workflow automates the content lifecycle from discovery to multi-channel distribution. It monitors NASA's RSS feeds, uses AI to extract high-quality image URLs, applies a brand watermark to those images, and distributes the final media to various social networks and a WordPress site (including featured image setup).

#### Logical Blocks:
*   **1.1 Ingestion & Triggering:** Monitors NASA RSS feeds for new articles.
*   **1.2 AI Data Extraction:** Uses an AI Agent with tool access to identify and extract image links from the RSS feed content.
*   **1.3 Image Processing:** Downloads the discovered image and applies a digital watermark.
*   **1.4 NASA API Fallback:** If RSS imagery is missing, it fetches the "NASA Picture of the Day" as a high-quality alternative.
*   **1.5 Multi-Channel Distribution:** Parallel execution of posting tasks to WordPress (including media library upload), Facebook, Instagram (via Meta Graph API), Telegram, and LinkedIn (Profile & Page).

---

### 2. Block-by-Block Analysis

#### 2.1 Ingestion & Primary AI Analysis
This block initiates the process by checking the RSS feed and attempting to find a featured image URL.
*   **Nodes Involved:** `Checks the NASA news RSS feed every minute`, `Loop Over Items`, `AI Agent (get only featured image)`, `OpenAI Chat Model`, `RSS Read Tool`.
*   **Node Details:**
    *   **Trigger (RSS):** Polls NASA's Breaking News feed.
    *   **AI Agent:** Configured as a "Tool Agent". It uses the `RSS Read Tool` to parse the content and the `OpenAI Chat Model` to identify the most relevant image URL.
    *   **Failure Modes:** RSS feed downtime, AI failing to find a valid URL (addressed by the "check link" logic).

#### 2.2 Fallback Discovery Logic
If the first AI pass fails to find a link, this block manually reads the feed again and attempts a secondary AI extraction or reverts to a static NASA image.
*   **Nodes Involved:** `If (check link)`, `Reads NASA Breaking News RSS manually`, `AI Agent (Again tries to extract an image link)`, `Get NASA Picture`.
*   **Node Details:**
    *   **Condition Node:** Checks if the AI returned a valid URL.
    *   **Fallback (NASA API):** Uses the dedicated NASA node to fetch the "Picture of the Day" if no article-specific image is found.
    *   **Secondary AI Agent:** Similar to the primary agent but focused on the manual RSS read output.

#### 2.3 Image Transformation
Before publishing, the workflow ensures the image contains the user's branding.
*   **Nodes Involved:** `HTTP Request (link to binary file)`, `Edits Image (Add Your Brand Watermark)`.
*   **Node Details:**
    *   **HTTP Request:** Downloads the image from the URL into a binary buffer.
    *   **Edit Image:** Applies a watermark. *Note: Requires a secondary image file or text overlay configuration.*
    *   **Failure Modes:** 404 on image link, unsupported image format (WebP/SVG issues).

#### 2.4 Distribution: Social Media & WordPress
The final block sends the processed image and article description to all connected platforms.
*   **Nodes Involved:** `Create WordPress Post`, `upload media to wp`, `Facebook media post`, `Publish Post to Instagram`, `Post on telegram Channel`, `Post on LinkedIn Profile/Page`.
*   **Node Details:**
    *   **WordPress:** A three-step process—creates the post, uploads the watermarked image to the media library, and then links that media ID as the "Featured Image" via HTTP Request (WP REST API).
    *   **Instagram:** Uses a two-step process—`Get Image ID` (uploading to Meta) followed by `Publish Post` (finalizing the container).
    *   **Telegram/LinkedIn/Facebook:** Directly posts the watermarked binary data with the article text.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Checks the NASA news RSS feed... | RSS Feed Trigger | Trigger | - | Loop Over Items | - |
| AI Agent (get only featured image) | AI Agent | URL Extraction | Loop Over Items | If (check link) | - |
| HTTP Request (link to binary...) | HTTP Request | Download Media | If (check link) | Edits Image | - |
| Edits Image (Add Watermark) | Edit Image | Branding | HTTP Request | WordPress, FB, IG, etc. | - |
| Create WordPress Post | WordPress | CMS Posting | Edits Image | download image | - |
| upload media to wp | HTTP Request | WP Media Upload | Add Watermark | upload image to meta | - |
| Get NASA Picture | NASA | Fallback Data | HTTP Request | HTTP Request | - |
| Publish Post to Instagram | Facebook Graph | IG Publishing | Get Image ID | - | - |
| Post on telegram Channel | Telegram | Messaging | Edits Image | - | - |
| Post on LinkedN page | LinkedIn | Professional Post | Edits Image | - | - |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:**
    *   Create an **RSS Feed Read Trigger**. Set the URL to NASA's news feed (e.g., `https://www.nasa.gov/rss/dyn/breaking_news.rss`).
    *   Add a **Split in Batches** (Loop Over Items) to handle multiple new articles.

2.  **AI Extraction Logic:**
    *   Create an **AI Agent** node. Attach an **OpenAI Chat Model** (e.g., GPT-4o) and an **RSS Read Tool**.
    *   Prompt the agent: "Identify the main image URL from this article content."

3.  **Conditional Routing:**
    *   Add an **If** node to check if `{{ $node["AI Agent"].json["output"] }}` contains "http".
    *   **True Path:** Use an **HTTP Request** node (Method: GET, Response: File) to download the URL.
    *   **False Path:** Add a **NASA** node (Operation: Get APOD) to fetch an alternative image.

4.  **Watermarking:**
    *   Add an **Edit Image** node. Select "Overlay". Upload your logo/watermark file and set the position (e.g., Bottom Right).

5.  **Platform Integration (Repeated for each loop branch):**
    *   **WordPress:** 
        1. `WordPress` Node: Create Post (Title and Content from RSS).
        2. `HTTP Request`: POST to `/wp-json/wp/v2/media` (Send watermarked binary).
        3. `HTTP Request`: POST to `/wp-json/wp/v2/posts/{{post_id}}` to set `featured_media`.
    *   **Meta (FB/IG):** Use the **Facebook Graph API** node. For Instagram, you must first create a media container and then publish it.
    *   **LinkedIn/Telegram:** Use the respective nodes, mapping the watermarked binary file to the "Attachments" or "Media" fields.

6.  **Credentials:**
    *   Required: NASA API Key, OpenAI API Key, WordPress (Application Password), Meta OAuth2 (Manage Pages/IG), Telegram Bot Token, LinkedIn OAuth2.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Ensure the WordPress user has "Editor" or "Administrator" permissions for REST API uploads. | Technical Requirement |
| Facebook/Instagram requires a Business Account and the "instagram_graph_user_media" permission. | Integration Note |
| NASA APOD API may have rate limits for the "DEMO_KEY"; use a dedicated API key for production. | [NASA API Portal](https://api.nasa.gov/) |
| Watermarking logic requires the n8n instance to have sufficient memory for image manipulation. | Infrastructure Note |