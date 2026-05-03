Monitor and analyze competitor Facebook ads with Apify, GPT-4o, Gemini, and Google Sheets

https://n8nworkflows.xyz/workflows/monitor-and-analyze-competitor-facebook-ads-with-apify--gpt-4o--gemini--and-google-sheets-13096


# Monitor and analyze competitor Facebook ads with Apify, GPT-4o, Gemini, and Google Sheets

This reference document provides a technical breakdown of the n8n workflow: **Monitor and analyze competitor Facebook ads with Apify, GPT-4o, Gemini, and Google Sheets**.

---

### 1. Workflow Overview
The primary purpose of this workflow is to automate competitive intelligence by scraping active advertisements from the Facebook Ad Library, filtering for significant advertisers, and using Multi-modal AI to reverse-engineer creative strategies. The workflow categorizes ads into three formats—Video, Image, and Text—and applies specific AI models (Google Gemini for video, GPT-4o for images) to generate summaries and rewritten ad copy.

#### Logical Blocks:
*   **1.1 Data Ingestion:** Scrapes raw ad data via Apify.
*   **1.2 Signal Filtering:** Removes low-impact ads based on page popularity.
*   **1.3 Creative Routing:** Identifies the media type and routes data to the appropriate processing pipeline.
*   **1.4 AI Analysis Pipelines:** format-specific blocks for extracting intelligence from video, image, or text content.
*   **1.5 Structured Storage:** Records results into Google Sheets with rate-limit protection.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Ingestion & Filtering
This block handles the retrieval of raw data and ensures only "high-signal" advertisers are analyzed.
*   **Nodes Involved:** `When clicking ‘Execute workflow’`, `Run an Actor and get dataset`, `Filter High-Signal Pages (>5k Likes)`.
*   **Node Details:**
    *   **Run an Actor and get dataset (Apify):** Executes a scraper (ID: `XtaWFhbtfxyzqrFmd`) targeting specific Facebook URLs. It is configured to retrieve up to 100 ads.
    *   **Filter High-Signal Pages:** A logic node that checks if `snapshot.page_like_count` is greater than 5,000. This ensures the workflow ignores small, unproven advertisers.

#### 2.2 Creative Routing
Categorizes the filtered ads to ensure the correct AI model is used for the specific media type.
*   **Nodes Involved:** `Detect Ad Creative Type`.
*   **Node Details:**
    *   **Detect Ad Creative Type (Switch):** Uses conditional logic to route items. Output 0 (Video) triggers if `video_sd_url` exists. Output 1 (Image) triggers if `original_image_url` exists. Output 2 (Text) is the fallback.

#### 2.3 Video Processing Pipeline
Analyzes video content using Google Gemini's native video understanding capabilities.
*   **Nodes Involved:** `Loop Over Video Ads`, `Download Video`, `Upload a file`, `Download a file`, `Analyze video`, `Add as Type = Video`, `Rate Limit Guard (Video)`.
*   **Node Details:**
    *   **Loop/Batching:** Uses `Split In Batches` to process ads one by one.
    *   **Media Handling:** `Download Video` fetches the MP4. `Upload a file` (Dropbox) and subsequent `Download a file` are used as an intermediary step to provide a binary buffer for the AI.
    *   **Analyze video (Google Gemini):** Uses `gemini-2.5-flash` to "analyze" the video binary.
    *   **Add as Type = Video (Google Sheets):** Performs an `appendOrUpdate` operation using `ad_archive_id` as the unique identifier.

#### 2.4 Image Processing Pipeline
Uses GPT-4o Vision to describe imagery and GPT-4 to rewrite the copy.
*   **Nodes Involved:** `Loop Over Image Ads`, `Scrape Facebook Ad Library (Apify)`, `Analyze Image`, `Output Image Summary`, `Add as Type = Image`, `Rate Limit Guard (Image)`.
*   **Node Details:**
    *   **Analyze Image (OpenAI):** Uses `gpt-4o` to comprehensively describe the visual content of the ad.
    *   **Output Image Summary (OpenAI):** A text-based prompt that combines the ad's metadata and the image description. It uses a specific "classic style" persona to summarize the strategy and rewrite the ad copy.
    *   **Google Sheets:** Appends structured data including the `image_prompt`.

#### 2.5 Text Processing Pipeline
For ads without media, focusing purely on copy strategy.
*   **Nodes Involved:** `Loop Over Text Ads`, `Output Text Summary`, `Add as Type = Text`, `Rate Limit Guard (Text)`.
*   **Node Details:**
    *   **Output Text Summary (OpenAI):** Uses GPT-4 to analyze the scraped Javascript object of the text ad, generating an analytical summary and a casual, human-like rewrite.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking ‘Execute workflow’ | Manual Trigger | Entry Point | - | Run an Actor... | - |
| Run an Actor and get dataset | Apify | Data Ingestion | When clicking... | Filter High-Signal... | Data Ingestion: Apify scrape and raw ad snapshot retrieval |
| Filter High-Signal Pages (>5k Likes) | Filter | Data Validation | Run an Actor... | Detect Ad Creative... | Signal Filtering: Remove low-quality advertisers early |
| Detect Ad Creative Type | Switch | Routing | Filter High-Signal... | Loops (Video, Image, Text) | Creative Routing: Split ads into video, image, text pipelines |
| Loop Over Video Ads | Split In Batches | Looping | Detect Ad Creative... | Download Video | - |
| Download Video | HTTP Request | Media Retrieval | Loop Over Video Ads | Upload a file | - |
| Upload a file | Dropbox | Storage | Download Video | Download a file | - |
| Download a file | Dropbox | Retrieval | Upload a file | Analyze video | - |
| Analyze video | Google Gemini | AI Analysis | Download a file | Add as Type = Video | AI Video Analysis: Format-specific analysis and rewriting |
| Add as Type = Video | Google Sheets | Data Storage | Analyze video | Rate Limit Guard (Video) | - |
| Rate Limit Guard (Video) | Wait | Throttle | Add as Type = Video | Loop Over Video Ads | - |
| Loop Over Image Ads | Split In Batches | Looping | Detect Ad Creative... | Scrape FB Ad Library | - |
| Scrape FB Ad Library (Apify) | HTTP Request | Data Retrieval | Loop Over Image Ads | Analyze Image | - |
| Analyze Image | OpenAI | Vision Analysis | Scrape FB Ad Library | Output Image Summary | AI Image Analysis: Format-specific analysis and rewriting |
| Output Image Summary | OpenAI | Copy Generation | Analyze Image | Add as Type = Image | AI Image Analysis: Format-specific analysis and rewriting |
| Add as Type = Image | Google Sheets | Data Storage | Output Image Summary | Rate Limit Guard (Image) | - |
| Rate Limit Guard (Image) | Wait | Throttle | Add as Type = Image | Loop Over Image Ads | - |
| Loop Over Text Ads | Split In Batches | Looping | Detect Ad Creative... | Output Text Summary | - |
| Output Text Summary | OpenAI | Text Analysis | Loop Over Text Ads | Add as Type = Text | AI Text Analysis: Format-specific analysis and rewriting |
| Add as Type = Text | Google Sheets | Data Storage | Output Text Summary | Rate Limit Guard (Text) | - |
| Rate Limit Guard (Text) | Wait | Throttle | Add as Type = Text | Loop Over Text Ads | - |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:** Ensure credentials for Apify, OpenAI, Google Gemini, Dropbox, and Google Sheets are configured in n8n.
2.  **Ingestion:** Create a **Manual Trigger** node. Connect it to an **Apify** node. Set the operation to "Run actor and get dataset". Input the actor ID for the Facebook Ads Scraper and provide a JSON body containing the target Facebook Page URLs.
3.  **Logic Filtering:** Connect a **Filter** node. Set the condition to check if `{{ $json.snapshot.page_like_count }}` is greater than `5000`.
4.  **Routing:** Connect a **Switch** node. Add three outputs:
    *   **Video:** Condition `{{ $json.snapshot.cards[0].video_sd_url }}` exists.
    *   **Image:** Condition `{{ $json.snapshot.cards[0].original_image_url }}` exists.
    *   **Text:** Fallback/Extra.
5.  **Processing (Image Path example):**
    *   Add **Split In Batches** node.
    *   Add **OpenAI** node (gpt-4o) with "Analyze Image" resource. Set input to the image URL from the Apify data.
    *   Add a second **OpenAI** node to summarize the ad copy based on the image analysis.
    *   Add **Google Sheets** node (Append operation). Use `ad_archive_id` for deduplication.
    *   Add **Wait** node (1 second) to act as a rate limit guard before looping back to the batch node.
6.  **Processing (Video Path):** Follow a similar loop but include **Dropbox** upload/download nodes to pipe the video binary into **Google Gemini**.
7.  **Processing (Text Path):** Directly use **OpenAI** to analyze `{{ $json.snapshot.body.text }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **How it works** | Scrapes live ads, filters for proven advertisers, detects type, applies AI analysis, and stores deduped data in G-Sheets. |
| **Setup Steps** | Connect Apify, OpenAI, Gemini, G-Sheets. Adjust like threshold if needed. |
| **Disclaimer** | This workflow uses legally public and automated data. [n8n.io](https://n8n.io) |
| **Google Sheet Schema** | Ensure columns: `ad_archive_id`, `page_id`, `type`, `date_added`, `page_name`, `page_url`, `summary`, `rewritten_ad_copy`, `image_prompt`, `video_prompt`. |