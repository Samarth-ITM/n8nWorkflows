Repurpose YouTube videos into multi-platform content with OpenAI and Anthropic

https://n8nworkflows.xyz/workflows/repurpose-youtube-videos-into-multi-platform-content-with-openai-and-anthropic-15014


# Repurpose YouTube videos into multi-platform content with OpenAI and Anthropic

# Technical Documentation: YouTube Content Repurposing Engine

## 1. Workflow Overview
The **YouTube Content Repurposing Engine** is an advanced automation system designed to monitor a specific YouTube channel for new uploads and automatically transform the video transcripts into a multi-platform content ecosystem. By leveraging AI (OpenAI and Anthropic), the workflow converts a single long-form video into SEO-optimized blog posts, viral Twitter threads, and professional LinkedIn updates.

The logic is divided into four primary functional blocks:
- **1.1 Video Discovery & Transcript Extraction:** Monitors the channel and prepares the raw text.
- **1.2 AI Content Generation:** Transforms the transcript into various formats using different LLMs.
- **1.3 Content Formatting & Optimization:** Cleans and structures AI outputs for specific platform requirements.
- **1.4 Multi-Platform Publishing:** Distributes the final content to WordPress, X (Twitter), and LinkedIn.

---

## 2. Block-by-Block Analysis

### 2.1 Video Discovery & Transcript Extraction
**Overview:** This block identifies new content on YouTube and retrieves the textual data required for AI processing.

- **Nodes Involved:** `Schedule Daily Check`, `Fetch Latest Videos`, `Filter New Videos`, `Get Transcript`, `Clean Text`.
- **Node Details:**
    - **Schedule Daily Check:** Trigger node that executes the workflow every 6 hours.
    - **Fetch Latest Videos:** HTTP Request to YouTube v3 API. Fetches the 5 most recent videos for a specific `channelId`.
    - **Filter New Videos:** Filter node that checks if the `publishedAt` date is within the last 24 hours.
    - **Get Transcript:** HTTP Request to YouTube v3 API to retrieve captions associated with the `videoId`.
    - **Clean Text:** Code node (JavaScript) that removes timestamps (e.g., `[00:00:00]`), collapses whitespace, and calculates word count to ensure the AI receives a clean string.
- **Potential Failures:** YouTube API quota exhaustion (10k units/day) or videos without available transcripts.

### 2.2 AI Content Generation
**Overview:** This block sends the cleaned transcript to different AI models to generate various content formats, incorporating a wait period to prevent API rate limiting.

- **Nodes Involved:** `Generate Short Scripts`, `Generate Blog Post`, `Generate Thread`, `Wait - Rate Limit`, `Merge1`.
- **Node Details:**
    - **Generate Short Scripts:** HTTP Request (OpenAI GPT-4). Prompts the AI to create three 60-second scripts for short-form video.
    - **Generate Blog Post:** HTTP Request (OpenAI GPT-4 Turbo). Prompts for a 1500-word SEO-optimized article including H2/H3 tags and meta descriptions.
    - **Generate Thread:** HTTP Request (Anthropic Claude Sonnet). Prompts for a 10-12 tweet viral thread with hooks.
    - **Wait - Rate Limit:** Wait node (60 seconds) positioned after the blog post generation to avoid hitting OpenAI's tokens-per-minute (TPM) limits.
    - **Merge1:** Merge node that consolidates the outputs from the three AI parallel paths.
- **Potential Failures:** API Authentication errors or timeout due to the length of the transcript.

### 2.3 Content Formatting & Optimization
**Overview:** This block transforms raw AI text into structured data (JSON/HTML) suitable for API publication.

- **Nodes Involved:** `Format Blog`, `Format Thread`, `Format LinkedIn`, `Merge2`.
- **Node Details:**
    - **Format Blog:** Code node that uses Regex to extract the Title and Meta Description and converts Markdown-style headers into HTML tags.
    - **Format Thread:** Code node that splits the Claude output into an array of individual tweets, ensuring each stays under 280 characters and includes numbering (e.g., 1/12).
    - **Format LinkedIn:** Code node that adds a professional hook, a list of key takeaways, and industry hashtags to the AI content.
    - **Merge2:** Combines the formatted objects into a single data stream for the publishing stage.

### 2.4 Multi-Platform Publishing
**Overview:** The final stage where the formatted content is pushed to external platforms and backed up.

- **Nodes Involved:** `Wait - Publishing`, `WordPress`, `Split Tweets`, `Post Tweet`, `LinkedIn`, `Backup to Drive`.
- **Node Details:**
    - **Wait - Publishing:** Wait node (minute-based) to stagger posts and avoid being flagged as spam.
    - **WordPress:** HTTP Request via Basic Auth to the WP-JSON API. Publishes the blog content as a "publish" status post.
    - **Split Tweets:** Split-in-batches node that iterates through the array of tweets generated in the formatting stage.
    - **Post Tweet:** HTTP Request (Twitter OAuth2) that posts each individual tweet from the batch.
    - **LinkedIn:** HTTP Request (LinkedIn OAuth2) that posts the professional content to a specific person's URN.
    - **Backup to Drive:** Google Drive node that saves the entire final JSON payload as a timestamped file for archiving and review.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Daily Check | Schedule Trigger | Workflow Entry | - | Fetch Latest Videos | 🎬 YouTube Content Repurposing Engine... |
| Fetch Latest Videos | HTTP Request | API Data Fetch | Schedule Daily Check | Filter New Videos | 📡 Stage 1: Video Discovery... |
| Filter New Videos | Filter | Date Validation | Fetch Latest Videos | Get Transcript | 📡 Stage 1: Video Discovery... |
| Get Transcript | HTTP Request | Transcript Retrieval | Filter New Videos | Clean Text | 📡 Stage 1: Video Discovery... |
| Clean Text | Code | Text Sanitization | Get Transcript | Gen. Scripts, Gen. Blog, Gen. Thread | 📡 Stage 1: Video Discovery... |
| Generate Short Scripts | HTTP Request | AI Content Gen | Clean Text | Merge1 | 🤖 Stage 2: AI Content Generation... |
| Generate Blog Post | HTTP Request | AI Content Gen | Clean Text | Wait - Rate Limit | 🤖 Stage 2: AI Content Generation... |
| Generate Thread | HTTP Request | AI Content Gen | Clean Text | Merge1 | 🤖 Stage 2: AI Content Generation... |
| Wait - Rate Limit | Wait | API Throttling | Generate Blog Post | Merge1 | 🤖 Stage 2: AI Content Generation... |
| Merge1 | Merge | Data Consolidation | Scripts, Wait, Thread | Formatter Nodes | 🤖 Stage 2: AI Content Generation... |
| Format Blog | Code | HTML Formatting | Merge1 | Merge2 | ✨ Stage 3: Content Formatting... |
| Format Thread | Code | Tweet Splitting | Merge1 | Merge2 | ✨ Stage 3: Content Formatting... |
| Format LinkedIn | Code | Professional Formatting | Merge1 | Merge2 | ✨ Stage 3: Content Formatting... |
| Merge2 | Merge | Data Consolidation | Blog, Thread, LinkedIn | Wait - Publishing | ✨ Stage 3: Content Formatting... |
| Wait - Publishing | Wait | Spam Prevention | Merge2 | WP, Split Tweets, LinkedIn | 🚀 Stage 4: Multi-Platform Publishing |
| WordPress | HTTP Request | CMS Publishing | Wait - Publishing | Backup to Drive | 🚀 Stage 4: Multi-Platform Publishing |
| Split Tweets | Split In Batches | Iteration | Wait - Publishing | Post Tweet | 🚀 Stage 4: Multi-Platform Publishing |
| Post Tweet | HTTP Request | Social Publishing | Split Tweets | Backup to Drive | 🚀 Stage 4: Multi-Platform Publishing |
| LinkedIn | HTTP Request | Social Publishing | Wait - Publishing | Backup to Drive | 🚀 Stage 4: Multi-Platform Publishing |
| Backup to Drive | Google Drive | Archiving | WP, Post Tweet, LinkedIn | - | 🚀 Stage 4: Multi-Platform Publishing |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Video Discovery Setup
1. Create a **Schedule Trigger** set to every 6 hours.
2. Add an **HTTP Request** node (`Fetch Latest Videos`):
    - URL: `https://www.googleapis.com/youtube/v3/search`
    - Parameters: `part=snippet`, `channelId=[YOUR_ID]`, `order=date`, `maxResults=5`, `key=[YOUR_API_KEY]`.
3. Add a **Filter** node: Compare `{{ $json.snippet.publishedAt }}` to be "after" `{{ $now.minus({ hours: 24 }).toISO() }}`.
4. Add an **HTTP Request** node (`Get Transcript`):
    - URL: `https://www.googleapis.com/youtube/v3/captions`
    - Parameter: `videoId={{ $json.id.videoId }}`.
5. Add a **Code** node: Implement the JS regex to remove `[00:00:00]` patterns and trim whitespace.

### Step 2: AI Generation Pipeline
1. Create three parallel **HTTP Request** nodes:
    - **OpenAI (Scripts):** POST to `/v1/chat/completions` using GPT-4.
    - **OpenAI (Blog):** POST to `/v1/chat/completions` using GPT-4-Turbo.
    - **Anthropic (Thread):** POST to `/v1/messages` using Claude-Sonnet.
2. Place a **Wait** node (60s) immediately after the "Generate Blog Post" node.
3. Connect all three paths (Scripts, Wait, and Thread) into a **Merge** node (Mode: Combine).

### Step 3: Formatting Logic
1. Create three **Code** nodes stemming from the Merge node:
    - **Blog Formatter:** Extract Title/Meta via Regex and replace `##` with `<h2>`.
    - **Thread Formatter:** Split text by digit markers (`1.`, `2.`) into an array of objects.
    - **LinkedIn Formatter:** Prepend a hook and append a standard set of hashtags.
2. Connect all three into a second **Merge** node (Mode: Combine).

### Step 4: Publishing & Archive
1. Add a **Wait** node (Unit: Minutes) to stagger the delivery.
2. Create three publishing paths:
    - **WordPress:** HTTP Request POST to `/wp-json/wp/v2/posts` (Basic Auth).
    - **X (Twitter):** **Split In Batches** $\rightarrow$ **HTTP Request** POST to `/2/tweets` (OAuth2).
    - **LinkedIn:** HTTP Request POST to `/v2/ugcPosts` (OAuth2).
3. Connect all publishing outputs to a **Google Drive** node to upload a JSON backup named `content_backup_{{ $now }}.json`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| YouTube API Quota | Search requests cost 100 units; daily limit is 10,000. |
| Twitter API v2 | Ensure the App has "Write" permissions enabled in the Developer Portal. |
| Rate Limiting | Wait nodes are critical; adjust times based on your OpenAI/Anthropic tier. |
| Content Review | All generated content is backed up to Drive for human quality review before/after posting. |