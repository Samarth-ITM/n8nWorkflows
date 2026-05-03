Transform YouTube Videos to LinkedIn Posts with Apify, GPT-4.1 Mini & Google Sheets

https://n8nworkflows.xyz/workflows/transform-youtube-videos-to-linkedin-posts-with-apify--gpt-4-1-mini---google-sheets-9386


# Transform YouTube Videos to LinkedIn Posts with Apify, GPT-4.1 Mini & Google Sheets

### 1. Workflow Overview

This workflow automates the transformation of newly published YouTube videos into LinkedIn posts. It targets content creators, social media managers, and marketing teams seeking to efficiently repurpose video content into professional, engaging LinkedIn posts without manual effort.

The workflow consists of five logical blocks:

- **1.1 RSS Feed Trigger:** Watches a specified YouTube channel’s RSS feed for new video uploads and triggers the workflow.
- **1.2 Transcript Extraction via Apify:** Runs an Apify actor to scrape the full transcript text of the newly published video.
- **1.3 AI Content Generation:** Uses the GPT-4.1 Mini model to analyze the transcript and generate exactly two LinkedIn post drafts as structured JSON.
- **1.4 JSON Parsing and Structuring:** Converts the AI’s raw JSON string output into a native JSON array for downstream processing.
- **1.5 Splitting and Saving Posts:** Splits the array of posts and appends each as a new row in a Google Sheet for review and scheduling.

---

### 2. Block-by-Block Analysis

#### 2.1 RSS Feed Trigger

- **Overview:**  
  This node monitors a specific YouTube channel’s RSS feed for new video uploads, triggering the workflow once per new video detected at a scheduled time.

- **Nodes Involved:**  
  - RSS Feed Trigger

- **Node Details:**

  - **RSS Feed Trigger**  
    - **Type:** RSS Feed Read Trigger  
    - **Role:** Event trigger to start workflow on new YouTube video detection  
    - **Configuration:**  
      - Feed URL set to `https://www.youtube.com/feeds/videos.xml?channel_id=REPLACE_THIS_ID` (requires replacement with actual channel ID)  
      - Poll scheduled daily at 9:00 AM  
    - **Expressions/Variables:** Uses YouTube channel’s RSS feed URL dynamically for each poll  
    - **Input:** None (trigger node)  
    - **Output:** Emits item with video metadata including video URL  
    - **Version-specific:** Uses standard RSS feed trigger node; no special version dependencies  
    - **Potential Failures:**  
      - Invalid or missing channel ID in URL leads to no trigger events  
      - Network issues or RSS feed changes by YouTube could cause trigger failures  
    - **Sub-workflow:** None

#### 2.2 Transcript Extraction via Apify Actor

- **Overview:**  
  Executes an Apify actor that scrapes the full transcript from the YouTube video URL received from the RSS trigger.

- **Nodes Involved:**  
  - Run an Actor and get dataset

- **Node Details:**

  - **Run an Actor and get dataset**  
    - **Type:** Apify node (@apify/n8n-nodes-apify.apify)  
    - **Role:** Runs an actor on Apify platform to scrape YouTube video transcript  
    - **Configuration:**  
      - Operation: “Run actor and get dataset”  
      - Actor ID: `ACirYOwWR3EeEmcqc` (corresponds to scrapingxpert/youtube-video-to-transcript)  
      - Custom JSON body dynamically injects video URL from RSS trigger: `{ "start_urls": [ { "url": "{{ $json.link }}" } ] }`  
      - Authentication via Apify OAuth2 credentials  
    - **Input:** Receives video URL from RSS Feed Trigger node  
    - **Output:** Returns dataset containing the transcript text  
    - **Version-specific:** Requires valid Apify OAuth2 credentials and network access  
    - **Potential Failures:**  
      - API quota exceeded or authentication failures with Apify  
      - Actor execution errors or timeouts  
      - Transcript missing if video has no captions or restrictions  
    - **Sub-workflow:** None

#### 2.3 AI Content Generation (OpenAI GPT-4.1 Mini)

- **Overview:**  
  Sends the full transcript to GPT-4.1 Mini with a detailed prompt to generate exactly two distinct LinkedIn posts in a strict JSON format.

- **Nodes Involved:**  
  - Message a model

- **Node Details:**

  - **Message a model**  
    - **Type:** LangChain OpenAI node (@n8n/n8n-nodes-langchain.openAi)  
    - **Role:** AI text generation using GPT-4.1 Mini  
    - **Configuration:**  
      - Model: `gpt-4.1-mini`  
      - Messages: System message includes persona as a thought leader, context with full transcript, detailed instructions to generate two distinct LinkedIn posts as a JSON array with strict schema  
      - JSON mode disabled (returns raw string)  
      - Input transcript dynamically inserted from Apify output via expression `{{ $json.transcript }}`  
      - Authentication via OpenAI API credentials  
    - **Input:** Transcript dataset from Apify node  
    - **Output:** Raw JSON string containing array of two LinkedIn post objects  
    - **Version-specific:** Requires access to OpenAI API and appropriate model availability  
    - **Potential Failures:**  
      - API quota limits or network errors  
      - Model response format errors or invalid JSON output (handled downstream)  
      - Timeout on long transcripts  
    - **Sub-workflow:** None

#### 2.4 JSON Parsing and Structuring

- **Overview:**  
  Converts the raw JSON string output from the AI node into a structured JSON array to facilitate further processing.

- **Nodes Involved:**  
  - Code in JavaScript

- **Node Details:**

  - **Code in JavaScript**  
    - **Type:** Code Node (JavaScript)  
    - **Role:** Parses AI raw JSON string to a structured JSON array wrapped in an object with key `posts`  
    - **Configuration:**  
      - Runs once per item (mode: runOnceForEachItem)  
      - JavaScript snippet:  
        ```javascript
        const response = $input.item.json.choices[0].message.content;
        const parsedData = JSON.parse(response);
        return { posts: parsedData };
        ```  
      - This enables n8n to handle the output easily as an array  
    - **Input:** Raw AI response string from OpenAI node  
    - **Output:** JSON object containing `posts` array  
    - **Version-specific:** Requires n8n version supporting code node with `runOnceForEachItem` mode  
    - **Potential Failures:**  
      - JSON.parse failure if AI output is malformed or incomplete  
      - Missing expected properties in AI response  
    - **Sub-workflow:** None

#### 2.5 Splitting and Saving Posts to Google Sheets

- **Overview:**  
  Splits the posts array into individual items and appends each as a separate row in a configured Google Sheet for easy review and scheduling.

- **Nodes Involved:**  
  - Split Out  
  - Append row in sheet

- **Node Details:**

  - **Split Out**  
    - **Type:** Split Out node  
    - **Role:** Splits the `posts` array into separate items for iterative processing  
    - **Configuration:**  
      - Field to split: `posts`  
    - **Input:** JSON object containing `posts` array (from Code node)  
    - **Output:** Emits one item per LinkedIn post  
    - **Potential Failures:**  
      - Missing or empty `posts` field causes no output items  
    - **Sub-workflow:** None

  - **Append row in sheet**  
    - **Type:** Google Sheets node  
    - **Role:** Appends each LinkedIn post as a new row in a Google Sheet  
    - **Configuration:**  
      - Operation: Append  
      - Document ID and Sheet Name configured (values expected to be set in credentials or workflow variables)  
      - Columns mapped: Theme, Hook, Body, CTA, Hashtags plus Video Link from RSS feed  
      - Mapping is explicit and uses expressions to map each post’s fields  
    - **Input:** Individual LinkedIn post items from Split Out node  
    - **Output:** Confirmation of row appended  
    - **Version-specific:** Requires Google Sheets OAuth2 credentials with write access  
    - **Potential Failures:**  
      - Authentication errors with Google API  
      - Sheet not found or permission denied  
      - Schema mismatch in columns  
    - **Sub-workflow:** None

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                   | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                              |
|-----------------------------|----------------------------------|-------------------------------------------------|------------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------|
| RSS Feed Trigger            | RSS Feed Read Trigger             | Triggers workflow on new YouTube video           | None                         | Run an Actor and get dataset | ## 1. 📰 RSS Feed (Trigger) The workflow's starting point. Watches YouTube channel RSS for new videos.   |
| Run an Actor and get dataset | Apify Node                       | Scrapes video transcript via Apify actor          | RSS Feed Trigger             | Message a model             | ## 2. 🕷️ Apify Runs Apify actor to extract transcript from video URL.                                   |
| Message a model             | OpenAI Chat Model (LangChain)     | Generates two LinkedIn posts from transcript      | Run an Actor and get dataset | Code in JavaScript          | ## 3. 🤖 OpenAI Chat Model Sends transcript to GPT-4.1 Mini to generate exactly two LinkedIn posts JSON. |
| Code in JavaScript          | Code Node                        | Parses AI JSON string response into JSON array    | Message a model              | Split Out                   | ## 4. 💻 Code Converts AI raw JSON string to structured array wrapped in `posts` object.                 |
| Split Out                  | Split Out                        | Splits posts array into individual post items     | Code in JavaScript           | Append row in sheet          | ## 5. 🔀📊 Split & Save Splits posts and appends each as a row in Google Sheets.                         |
| Append row in sheet         | Google Sheets                    | Saves each LinkedIn post as new row in Google Sheet| Split Out                   | None                        |                                                                                                         |
| Sticky Note                 | Sticky Note                     | Informational note                                | None                         | None                        | ## YouTube to LinkedIn Content Engine overview note                                                   |
| Sticky Note1                | Sticky Note                     | Explains RSS Feed Trigger block                   | None                         | None                        | ## 1. 📰 RSS Feed (Trigger) detailed explanation                                                        |
| Sticky Note2                | Sticky Note                     | Explains Apify actor block                         | None                         | None                        | ## 2. 🕷️ Apify explanation                                                                            |
| Sticky Note3                | Sticky Note                     | Explains OpenAI Chat Model node                    | None                         | None                        | ## 3. 🤖 OpenAI Chat Model explanation                                                                |
| Sticky Note4                | Sticky Note                     | Explains Code node parsing step                    | None                         | None                        | ## 4. 💻 Code explanation                                                                              |
| Sticky Note5                | Sticky Note                     | Explains splitting and saving posts block         | None                         | None                        | ## 5. 🔀📊 Split & Save (Google Sheets) explanation                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "RSS Feed Trigger" node:**  
   - Type: RSS Feed Read Trigger  
   - Set `feedUrl` to `https://www.youtube.com/feeds/videos.xml?channel_id=YOUR_CHANNEL_ID` (replace `YOUR_CHANNEL_ID`)  
   - Schedule polling time at 9:00 AM daily (or your preferred time)  
   - This node triggers on new YouTube video uploads.

2. **Add "Run an Actor and get dataset" node (Apify):**  
   - Type: Apify node  
   - Operation: "Run actor and get dataset"  
   - Actor ID: `ACirYOwWR3EeEmcqc` (scrapingxpert/youtube-video-to-transcript)  
   - Authentication: Configure Apify OAuth2 credentials  
   - Custom Body (JSON):  
     ```json
     {
       "start_urls": [
         { "url": "{{ $json.link }}" }
       ]
     }
     ```  
   - Connect input from "RSS Feed Trigger" main output.

3. **Add "Message a model" node (OpenAI GPT-4.1 Mini):**  
   - Type: LangChain OpenAI node  
   - Model: Select `gpt-4.1-mini`  
   - Authentication: Configure OpenAI API credentials  
   - Messages:  
     - Role: assistant  
     - Content: Use the detailed prompt instructing the AI to generate two distinct LinkedIn posts as a JSON array following the provided schema.  
     - Inject transcript dynamically using `{{ $json.transcript }}` from Apify output.  
   - Connect input from Apify node main output.

4. **Add "Code in JavaScript" node:**  
   - Type: Code node (JavaScript)  
   - Mode: Run once for each item  
   - Code snippet:  
     ```javascript
     const response = $input.item.json.choices[0].message.content;
     const parsedData = JSON.parse(response);
     return { posts: parsedData };
     ```  
   - Connect input from OpenAI node main output.

5. **Add "Split Out" node:**  
   - Type: Split Out  
   - Field to split out: `posts`  
   - Connect input from Code node main output.

6. **Add "Append row in sheet" node (Google Sheets):**  
   - Type: Google Sheets  
   - Operation: Append  
   - Configure Google Sheets OAuth2 credentials with write access  
   - Document ID: Set your target Google Sheet ID  
   - Sheet Name: Set the sheet name or gid (e.g., `gid=0`)  
   - Mapping mode: Define below  
   - Map columns:  
     - Theme: `={{ $json.theme }}`  
     - Hook: `={{ $json.hook }}`  
     - Body: `={{ $json.body }}`  
     - CTA: `={{ $json.cta }}`  
     - Hashtags: `={{ $json.hashtags }}`  
     - Video Link: `={{ $('RSS Feed Trigger').item.json.link }}` (reference original video URL)  
   - Connect input from Split Out node main output.

7. **Connect the nodes in this order:**  
   RSS Feed Trigger → Run an Actor and get dataset → Message a model → Code in JavaScript → Split Out → Append row in sheet

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This automation monitors a YouTube channel for new videos, extracts the transcript, uses AI to create two LinkedIn posts, and saves them for review. | Workflow overview sticky note                                                                            |
| Apify actor used: `scrapingxpert/youtube-video-to-transcript` at https://console.apify.com/actors/ACirYOwWR3EeEmcqc                 | Apify actor reference                                                                                   |
| GPT-4.1 Mini model prompt enforces strict JSON output for two distinct LinkedIn posts with specific schema and tone instructions.   | OpenAI prompt design details                                                                            |
| Use Google Sheets for storage to facilitate easy scheduling and manual review before publishing.                                    | Integration with Google Sheets for content management                                                  |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.