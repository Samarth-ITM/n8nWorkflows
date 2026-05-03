Summarize News Articles & Auto-Post to Social Media with GPT-4 and GetLate

https://n8nworkflows.xyz/workflows/summarize-news-articles---auto-post-to-social-media-with-gpt-4-and-getlate-8694


# Summarize News Articles & Auto-Post to Social Media with GPT-4 and GetLate

### 1. Workflow Overview

This workflow automates the process of summarizing news articles from multiple URLs and publishing ready-to-post content on social media platforms using AI and the GetLate multipublishing API. It is designed for content creators, news aggregators, or social media managers who want to efficiently convert raw news articles into engaging, concise, and structured social media posts targeted at young audiences (18-25 years). The workflow includes the following logical blocks:

- **1.1 Input Reception & URL Management:** Accepts and manages multiple news article URLs for processing.
- **1.2 Article Retrieval & Extraction:** Fetches full HTML content from each URL and extracts clean text (title, article body, main image).
- **1.3 Article Summarization (Fact-Based):** Uses OpenAI GPT to generate factual summaries of each article.
- **1.4 Aggregation & Structured Summarization:** Aggregates individual summaries and creates structured summaries tailored for young readers.
- **1.5 Social Media Content Generation:** Produces concise, impactful social media post descriptions optimized for multiple platforms.
- **1.6 Image Handling & Upload:** Extracts main images from articles and uploads them to Google Drive.
- **1.7 Multi-Platform Publishing:** Posts the generated content and images to social media platforms via the GetLate API.
- **1.8 Manual Trigger & Notes:** Workflow entry point and contextual notes explaining usage and requirements.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & URL Management

- **Overview:** Defines the list of news article URLs to be processed and splits them into individual items for parallel processing.
- **Nodes Involved:**  
  - Edit Fields  
  - Split Out

- **Node Details:**

  - **Edit Fields**  
    - Type: Set node  
    - Role: Hardcodes the URLs array containing news article URLs to process.  
    - Configuration: Raw JSON mode with a field `urls` containing an array of URLs (e.g. `URL1`, `URL2`, `URL3`).  
    - Inputs: Manual Trigger node  
    - Outputs: Split Out node  
    - Edge Cases: Empty or malformed URLs can cause downstream HTTP errors.

  - **Split Out**  
    - Type: Split Out node  
    - Role: Splits the array of URLs from the `urls` field into individual items to process each URL independently.  
    - Configuration: Splits on the field `urls`.  
    - Inputs: Edit Fields  
    - Outputs: HTTP Request node  
    - Edge Cases: If `urls` is missing or empty, no items are generated.

#### 2.2 Article Retrieval & Extraction

- **Overview:** Fetches the raw HTML content of each article URL and parses it to extract the article's title, clean textual content, and main image URL.
- **Nodes Involved:**  
  - HTTP Request  
  - Code  
  - HTML1

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Retrieves the full HTML content of each article URL.  
    - Configuration: URL dynamically set from the split URL items (`={{ $json.urls }}`). On error, it continues without halting the workflow.  
    - Inputs: Split Out  
    - Outputs: Code node, HTML1 node (parallel outputs)  
    - Edge Cases: HTTP errors (404, 500), timeouts, invalid URLs.

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Parses HTML to extract:  
      - Article title via meta tags or `<title>`.  
      - Main text content by prioritizing `<article>`, `<main>`, or common article body containers.  
      - Cleans “noise” blocks such as share buttons, newsletters, and scripts.  
      - Extracts paragraphs, filters short lines, truncates to max 20 paragraphs or 8000 characters.  
      - Removes trailing “parasite” blocks triggered by keywords like "Partager", "newsletter", etc.  
      - Locates main image URL via `og:image`, `twitter:image`, or `<figure>` tags.  
      - Decodes HTML entities.  
    - Configuration: Runs once per item to process the HTML content from HTTP Request.  
    - Inputs: HTTP Request  
    - Outputs: OpenAI (fact-based summary)  
    - Edge Cases: Unusual HTML structure, missing meta tags, relative URLs not resolvable, malformed HTML.

  - **HTML1**  
    - Type: HTML Extract  
    - Role: Extracts the main article image URL from meta property `og:image` in the HTML for image downloading.  
    - Configuration: CSS selector targeting `meta[property="og:image"]`, returning the `content` attribute.  
    - Inputs: HTTP Request  
    - Outputs: HTTP Request1 (image download)  
    - Edge Cases: Missing or incorrect meta tags, broken image URLs.

#### 2.3 Article Summarization (Fact-Based)

- **Overview:** Generates a factual, concise summary of each article based solely on extracted title and content without adding any external information.
- **Nodes Involved:**  
  - OpenAI

- **Node Details:**

  - **OpenAI**  
    - Type: OpenAI (Langchain)  
    - Role: Uses GPT-5 model to create fact-based summaries from the extracted article content.  
    - Configuration: System prompt instructs to avoid invention and keep summaries factual and hierarchical. User prompt includes article title and content dynamically.  
    - Inputs: Code node output  
    - Outputs: Aggregate  
    - Credentials: Requires OpenAI API key credential.  
    - Edge Cases: API rate limits, network errors, content too long for prompt, malformed input.

#### 2.4 Aggregation & Structured Summarization

- **Overview:** Aggregates multiple fact-based summaries into a single combined content block and generates a structured summary formatted in JSON for young readers.
- **Nodes Involved:**  
  - Aggregate  
  - Message a model

- **Node Details:**

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Combines the individual messages (summary contents) from the OpenAI node into a single array for consolidated processing.  
    - Configuration: Aggregates the field `message.content`.  
    - Inputs: OpenAI  
    - Outputs: Message a model  
    - Edge Cases: Empty inputs, large aggregated content.

  - **Message a model**  
    - Type: OpenAI (Langchain)  
    - Role: Generates a structured JSON summary targeted at young readers with fields: `titre`, `sous_titre`, and an array of `parties` containing titles and descriptions.  
    - Configuration: GPT-4o model with system instructions to strictly respect JSON format, journalistic tone, no invention. User prompt includes the aggregated content.  
    - Inputs: Aggregate  
    - Outputs: Message a model1  
    - Credentials: OpenAI API key required.  
    - Edge Cases: JSON formatting errors, generation inconsistencies, API errors.

#### 2.5 Social Media Content Generation

- **Overview:** Converts the structured summary into a social media post description optimized for young adults, including a hook, summary, call-to-action, and relevant hashtags.
- **Nodes Involved:**  
  - Message a model1

- **Node Details:**

  - **Message a model1**  
    - Type: OpenAI (Langchain)  
    - Role: Creates social media-ready post descriptions formatted with: hook, summary, call-to-action, and hashtags without emojis at the start and no invented content.  
    - Configuration: GPT-4o-latest model with strict editorial instructions. Input is the JSON summary from Message a model.  
    - Inputs: Message a model  
    - Outputs: GetLate multipublishing  
    - Credentials: OpenAI API key required.  
    - Edge Cases: Content length limits, missing data in input JSON, API failures.

#### 2.6 Image Handling & Upload

- **Overview:** Downloads the main article image and uploads it to a specified Google Drive folder for use in social media posts.
- **Nodes Involved:**  
  - HTTP Request1  
  - Upload file

- **Node Details:**

  - **HTTP Request1**  
    - Type: HTTP Request  
    - Role: Downloads the image from the extracted URL.  
    - Configuration: URL set dynamically from `HTML1` node’s `figure_img` field.  
    - Inputs: HTML1  
    - Outputs: Upload file  
    - Edge Cases: Broken image URLs, large image files, download timeouts.

  - **Upload file**  
    - Type: Google Drive  
    - Role: Uploads the downloaded image file to a specific Google Drive folder named "Pikor - Images articles".  
    - Configuration: File name generated dynamically with timestamp, source, and item index. Folder ID and Drive ID configured from credentials.  
    - Inputs: HTTP Request1  
    - Outputs: GetLate multipublishing  
    - Credentials: Google Drive OAuth2 credentials required.  
    - Edge Cases: Google Drive quota issues, permission errors, upload failures.

#### 2.7 Multi-Platform Publishing

- **Overview:** Posts the social media description and associated image to multiple platforms (Twitter, Instagram, LinkedIn, Threads, YouTube) using the GetLate API.
- **Nodes Involved:**  
  - GetLate multipublishing

- **Node Details:**

  - **GetLate multipublishing**  
    - Type: HTTP Request  
    - Role: Sends a POST request to GetLate API with the social media post content, scheduled publishing parameters, tags, and media items including the uploaded image URL.  
    - Configuration:  
      - URL: `https://getlate.dev/api/v1/post`  
      - Method: POST  
      - Headers: Authorization contains JSON payload with content from `Message a model1`, platform account IDs, scheduling info, tags, and media URLs from `HTTP Request1`.  
      - Content-Type: application/json  
    - Inputs: Upload file and Message a model1 (parallel inputs merged)  
    - Outputs: None  
    - Edge Cases: API auth errors, invalid platform IDs, network errors, malformed payload.

#### 2.8 Manual Trigger & Notes

- **Overview:** Facilitates manual execution and provides explanatory notes about the workflow’s purpose, usage, and requirements.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Sticky Notes (various)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution.  
    - Outputs: Edit Fields  
    - Edge Cases: None.

  - **Sticky Notes**  
    - Type: Sticky Note  
    - Role: Provide explanations, instructions, requirements, and context for users.  
    - Content Highlights:  
      - Test mode indicator  
      - Overall workflow description and use cases  
      - How the workflow operates step-by-step  
      - Required credentials and accounts (OpenAI, Google Drive, GetLate)  
    - Position: Strategically placed for user clarity.

---

### 3. Summary Table

| Node Name                  | Node Type                    | Functional Role                                   | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                      |
|----------------------------|------------------------------|-------------------------------------------------|------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger               | Workflow entry point                             |                              | Edit Fields                 |                                                                                                 |
| Edit Fields                | Set                           | Define URLs array to process                     | When clicking ‘Execute workflow’ | Split Out                  |                                                                                                 |
| Split Out                  | Split Out                     | Split URLs array into individual items           | Edit Fields                  | HTTP Request                |                                                                                                 |
| HTTP Request               | HTTP Request                  | Fetch raw HTML content for each URL              | Split Out                   | Code, HTML1                 |                                                                                                 |
| Code                       | Code                          | Extract title, content, and main image URL       | HTTP Request                | OpenAI                      |                                                                                                 |
| HTML1                      | HTML Extract                  | Extract main image URL from meta tags            | HTTP Request                | HTTP Request1               |                                                                                                 |
| HTTP Request1              | HTTP Request                  | Download main image                               | HTML1                       | Upload file                 |                                                                                                 |
| Upload file                | Google Drive                  | Upload the downloaded image to Google Drive      | HTTP Request1               | GetLate multipublishing     |                                                                                                 |
| OpenAI                     | OpenAI (Langchain)            | Generate factual summary of article              | Code                        | Aggregate                   |                                                                                                 |
| Aggregate                  | Aggregate                    | Combine summaries into one content block          | OpenAI                      | Message a model             |                                                                                                 |
| Message a model            | OpenAI (Langchain)            | Create structured JSON summary for young readers | Aggregate                   | Message a model1            |                                                                                                 |
| Message a model1           | OpenAI (Langchain)            | Generate social media post description            | Message a model             | GetLate multipublishing     |                                                                                                 |
| GetLate multipublishing    | HTTP Request                  | Post content + image to multiple social platforms | Upload file, Message a model1 |                             |                                                                                                 |
| Sticky Note                | Sticky Note                   | Test mode indicator                              |                              |                             | ## Test mode                                                                                     |
| Sticky Note1               | Sticky Note                   | Workflow overview and use cases                   |                              |                             | ## Different Articles Summarizer & Social Media Auto-Poster\n*Automates summarization and posting.* |
| Sticky Note2               | Sticky Note                   | Image downloading explanation                     |                              |                             | ## Images downloading                                                                            |
| Sticky Note3               | Sticky Note                   | Social networks posting explanation               |                              |                             | ## Social Networks posting                                                                       |
| Sticky Note4               | Sticky Note                   | How it works step-by-step                          |                              |                             | ## How it works\n*Detailed workflow steps*                                                       |
| Sticky Note5               | Sticky Note                   | Requirements and credentials                       |                              |                             | ## Requirements\n*OpenAI, Google Drive, Late API keys, valid URLs*                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: *When clicking ‘Execute workflow’*  
   - Purpose: Manual start of the workflow.

2. **Add Set Node for URLs:**  
   - Name: *Edit Fields*  
   - Type: Set  
   - Configuration: Raw JSON mode, field `urls` with array of news article URLs to process.

3. **Add Split Out Node:**  
   - Name: *Split Out*  
   - Configuration: Split field `urls` into individual items.

4. **Add HTTP Request Node for Article HTML:**  
   - Name: *HTTP Request*  
   - Configuration: URL set dynamically using expression `={{ $json.urls }}`.  
   - On Error: Set to continue on error to avoid stopping workflow on failed requests.

5. **Add Code Node to Parse HTML:**  
   - Name: *Code*  
   - Configuration: JavaScript code to extract article title, clean text content, and main image URL. Use functions for decoding HTML entities, stripping noise blocks, extracting paragraphs, and resolving relative URLs.  
   - Input: from HTTP Request node.

6. **Add OpenAI Node for Fact-Based Summary:**  
   - Name: *OpenAI*  
   - Type: Langchain OpenAI node  
   - Model: GPT-5 (or latest available with similar capabilities)  
   - System prompt: Instruct to summarize factually with no invention.  
   - User prompt: Provide article title and content dynamically.  
   - Credentials: Connect OpenAI API key.  
   - Input: Code node output.

7. **Add Aggregate Node:**  
   - Name: *Aggregate*  
   - Configuration: Aggregate field `message.content` from OpenAI outputs.  
   - Input: OpenAI node.

8. **Add OpenAI Node for Structured Summary:**  
   - Name: *Message a model*  
   - Model: GPT-4o (or equivalent)  
   - System prompt: Strict JSON output with structure for young readers.  
   - User prompt: Provide aggregated content.  
   - Credentials: OpenAI API key.  
   - Input: Aggregate node output.

9. **Add OpenAI Node for Social Media Post Generation:**  
   - Name: *Message a model1*  
   - Model: GPT-4o-latest  
   - System prompt: Editorial instructions for short, impactful social media posts targeted at ages 18-25 with hook, summary, CTA, hashtags.  
   - User prompt: Input structured JSON summary.  
   - Credentials: OpenAI API key.  
   - Input: Message a model node output.

10. **Add HTML Extract Node for Image URL:**  
    - Name: *HTML1*  
    - Operation: Extract HTML content  
    - Extraction: CSS selector `meta[property="og:image"]` attribute `content` for main image URL.  
    - Input: HTTP Request node output.

11. **Add HTTP Request Node to Download Image:**  
    - Name: *HTTP Request1*  
    - URL: Dynamically set to extracted image URL from HTML1 node.  
    - Input: HTML1 node output.

12. **Add Google Drive Upload Node:**  
    - Name: *Upload file*  
    - Configuration:  
      - Upload downloaded image file.  
      - File name: `{{$now.format('f')}}_{{$json["source"] || "image"}}_{{$itemIndex}}.jpg`  
      - Folder ID: Configure with your Google Drive folder for article images.  
      - Drive ID: Select your Drive (e.g. “My Drive”).  
    - Credentials: Google Drive OAuth2 credentials.  
    - Input: HTTP Request1.

13. **Add HTTP Request Node for GetLate Publishing:**  
    - Name: *GetLate multipublishing*  
    - Method: POST  
    - URL: `https://getlate.dev/api/v1/post`  
    - Headers:  
      - `Authorization`: JSON string with:  
        - `content`: social media post content from *Message a model1*  
        - `platforms`: array with platform names and account IDs (replace placeholders with actual IDs)  
        - `scheduledFor`, `timezone`, `publishNow`, `isDraft`, `visibility`, `tags`  
        - `mediaItems`: image URL from *HTTP Request1* (or Google Drive URL if available)  
      - `Content-Type`: application/json  
    - Inputs: Connect outputs from *Upload file* and *Message a model1* as needed.

14. **Connect all nodes accordingly:**  
    - Manual Trigger → Edit Fields → Split Out → HTTP Request  
    - HTTP Request → Code → OpenAI → Aggregate → Message a model → Message a model1 → GetLate multipublishing  
    - HTTP Request → HTML1 → HTTP Request1 → Upload file → GetLate multipublishing

15. **Add Sticky Notes for documentation:**  
    - Add notes describing test mode, workflow overview, image downloading, social posting, how it works, and requirements.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                           | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow automates summarizing news articles and posting on social media for young audiences using GPT and GetLate API.                          | Overview sticky note                                                                                             |
| Requires OpenAI API key connected in n8n for all GPT nodes.                                                                                            | Requirements sticky note                                                                                         |
| Requires Google Drive OAuth2 credentials for image storage.                                                                                           | Requirements sticky note                                                                                         |
| Requires GetLate API credentials and valid platform account IDs for posting to Twitter, Instagram, LinkedIn, Threads, and YouTube.                   | Requirements sticky note                                                                                         |
| Workflow can be started manually or integrated with other triggers.                                                                                   | How it works sticky note                                                                                         |
| The image extraction is robust, preferring meta tags but falling back to figures and main/article images if needed.                                  | Image downloading sticky note                                                                                    |
| Social media posts are strictly formatted with hook, summary, call-to-action, hashtags, without emojis at the start, and no invented content.         | Social networks posting sticky note                                                                              |
| The workflow is designed to handle multiple article URLs in parallel and aggregate results into a single post.                                        | Different Articles Summarizer & Social Media Auto-Poster sticky note                                            |
| All OpenAI prompts emphasize factual accuracy, no invention, and strict output formatting to prevent errors and maintain content integrity.           | Various OpenAI nodes prompts                                                                                     |
| Replace placeholder account IDs in GetLate node with actual IDs to enable publishing.                                                                 | GetLate multipublishing node configuration                                                                       |

---

**Disclaimer:** The provided content is extracted from an n8n workflow automation, respecting content policies strictly without any illegal or offensive material. All processed data is public and legal.