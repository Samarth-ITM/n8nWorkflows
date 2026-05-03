Turn YouTube Comments into Content Ideas with GPT-4.1-mini, Tavily & Apify

https://n8nworkflows.xyz/workflows/turn-youtube-comments-into-content-ideas-with-gpt-4-1-mini--tavily---apify-7479


# Turn YouTube Comments into Content Ideas with GPT-4.1-mini, Tavily & Apify

### 1. Workflow Overview

This n8n workflow automates the process of transforming YouTube comments into well-researched, engaging content ideas for video creation. It is targeted at content creators, marketers, and social media managers who want to repurpose audience interactions into actionable video hooks and outlines, saving time on ideation and research.

The workflow consists of five main logical blocks:

- **1.1 Input Reception:** Receives a YouTube video URL via chat input to trigger the workflow.
- **1.2 Comment Scraping:** Uses Apify’s YouTube Comments Scraper API to fetch raw comments from the video.
- **1.3 Comment Analysis & Filtering:** Uses OpenAI GPT-4.1-mini to analyze comments for viable content ideas and filters those marked “Yes.”
- **1.4 Research & Idea Enrichment:** For each valid comment, combines Tavily web research with OpenAI to generate a researched topic overview and then crafts a compelling hook and outline.
- **1.5 Data Storage:** Saves raw comments and enriched content ideas back into Google Sheets, updating rows to keep all related data unified.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block starts the workflow upon receiving a YouTube URL pasted into a chat input, exposing a webhook for interaction.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: LangChain Chat Trigger (webhook-based)  
    - Configuration: Waits for chat input, capturing the YouTube video URL as `{{ $json.chatInput }}`.  
    - Input: External chat message input  
    - Output: JSON object containing the pasted YouTube URL  
    - Failure/Edge Cases: Missing or invalid URL input could cause downstream failures; no built-in validation here.  
    - Version: 1.1  

---

#### 2.2 Comment Scraping

- **Overview:**  
  Scrapes YouTube comments from the provided video URL using Apify’s YouTube Comments Scraper API.

- **Nodes Involved:**  
  - Fetch YouTube Comments (Apify)

- **Node Details:**  
  - **Fetch YouTube Comments (Apify)**  
    - Type: HTTP Request (POST)  
    - Configuration:  
      - URL: Apify’s YouTube Comments Scraper API endpoint  
      - Body: JSON with max 100 comments, sorting by top, startUrls containing the YouTube URL from input  
      - Headers: Authorization token for Apify, Accept application/json  
    - Input: YouTube URL from chat trigger  
    - Output: Array of comment objects with fields like `id`, `text`, `author`, `likeCount`, `replyCount`, `publishedTime`  
    - Failure/Edge Cases: API rate limits, invalid token, network errors, empty comments dataset  
    - Version: 4.2  

---

#### 2.3 Comment Analysis & Filtering

- **Overview:**  
  Uses GPT-4.1-mini to analyze each comment and decide if it contains a viable content idea (“Yes” or “No”). Only comments marked “Yes” move forward.

- **Nodes Involved:**  
  - Analyzing YouTube comments  
  - Comment Idea Filter  
  - Store Raw Comments  
  - Loop Over Items

- **Node Details:**  
  - **Analyzing YouTube comments**  
    - Type: OpenAI node (GPT-4.1-mini)  
    - Configuration:  
      - Prompt: Evaluates comment text for content idea viability  
      - Output: “Yes” or “No” string  
    - Input: Comments from Apify node  
    - Output: Content idea flag per comment  
    - Failure/Edge Cases: API errors, prompt output parsing errors, ambiguous comments  
    - Version: 1.8  

  - **Comment Idea Filter**  
    - Type: Filter node  
    - Configuration: Passes only items where AI output equals "Yes"  
    - Input: Output from Analyzing YouTube comments node  
    - Output: Filtered comments  
    - Failure/Edge Cases: None significant; strict equality check  

  - **Store Raw Comments**  
    - Type: Google Sheets Append or Update  
    - Configuration:  
      - Stores each comment’s `id`, `text`, `author`, `likeCount`, `replyCount`, `publishedTime`, and the “Yes”/“No” flag  
      - Uses `id` as key to prevent duplicates  
    - Input: Filtered comments from Comment Idea Filter  
    - Output: Data saved in Google Sheets  
    - Failure/Edge Cases: Google Sheets API quota, authentication errors, schema mismatches  
    - Version: 4.5  

  - **Loop Over Items**  
    - Type: SplitInBatches node  
    - Configuration: Processes comments one by one for sequential enrichment to avoid race conditions  
    - Input: Stored raw comments flagged “Yes”  
    - Output: Single comment item per iteration  
    - Failure/Edge Cases: Batch size defaults to 1 (implicit), can slow workflow for large datasets  

---

#### 2.4 Research & Idea Enrichment

- **Overview:**  
  For each valid comment, extracts a content topic and conducts real-time web research with Tavily, then uses OpenAI to generate a compelling hook and structured outline.

- **Nodes Involved:**  
  - Extracting content topics from YouTube comments  
  - Research Context (Tavily)  
  - Transforming a researched YouTube video topic into a compelling video concept

- **Node Details:**  
  - **Extracting content topics from YouTube comments**  
    - Type: OpenAI node (GPT-4.1-mini)  
    - Configuration:  
      - Prompt: Extracts a content-worthy video topic and instructs Tavily to research it  
      - Output: JSON with `topic` and `research_overview` (300–500 word research summary)  
    - Input: Single comment text from Loop Over Items  
    - Output: Topic + research overview JSON  
    - Failure/Edge Cases: API errors, incomplete research results, malformed JSON output  
    - Version: 1.8  

  - **Research Context (Tavily)**  
    - Type: LangChain Tool HTTP Request node  
    - Configuration:  
      - Calls Tavily web search API with parameters for advanced research depth, 1 max result, last 7 days timeframe  
      - Authorization header with Tavily API token  
    - Input: Topic extracted by the previous OpenAI node (passed internally)  
    - Output: Research data used by OpenAI prompt  
    - Failure/Edge Cases: API rate limits, network issues, empty search results  
    - Version: 1.1  

  - **Transforming a researched YouTube video topic into a compelling video concept**  
    - Type: OpenAI node (GPT-4.1-mini)  
    - Configuration:  
      - Prompt: Using `topic` and `research_overview`, generates a curiosity-driven hook and 3–6 bullet point outline  
      - Output: JSON with keys `hook` and `outline` (array)  
    - Input: Output JSON from Extracting content topics node  
    - Output: Hook and outline JSON  
    - Failure/Edge Cases: API errors, output parsing errors, insufficient or generic hooks/outlines  
    - Version: 1.8  

---

#### 2.5 Data Storage

- **Overview:**  
  Updates the Google Sheets row corresponding to each comment with enriched data including topic, research overview, hook, and outline.

- **Nodes Involved:**  
  - Update Idea with Enriched Data

- **Node Details:**  
  - **Update Idea with Enriched Data**  
    - Type: Google Sheets Update  
    - Configuration:  
      - Matches rows by comment `id`  
      - Adds columns for `topic`, `research`, `hook`, and `outline`  
      - Keeps all original comment metadata intact  
    - Input: Enriched content from Transforming node and comment metadata from Loop Over Items  
    - Output: Updated Google Sheet row  
    - Failure/Edge Cases: Google Sheets API limits, credential issues, schema mismatch  
    - Version: 4.5  

---

### 3. Summary Table

| Node Name                                             | Node Type                              | Functional Role                           | Input Node(s)                      | Output Node(s)                                  | Sticky Note                                                                                                                                            |
|-------------------------------------------------------|--------------------------------------|-----------------------------------------|----------------------------------|------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| When chat message received                             | LangChain Chat Trigger                | Entry point; receives YouTube URL       | -                                | Fetch YouTube Comments (Apify)                  | ## 1. Chat Trigger – Paste YouTube URL<br>Starts workflow on YouTube URL pasted in chat input.                                                         |
| Fetch YouTube Comments (Apify)                         | HTTP Request                         | Scrapes YouTube comments                 | When chat message received        | Analyzing YouTube comments                      | ## 2. Scrape YouTube Comments (Apify)<br>Uses Apify API to collect raw comment data from video.                                                       |
| Analyzing YouTube comments                             | OpenAI (GPT-4.1-mini)                 | Analyzes comments for content potential | Fetch YouTube Comments (Apify)    | Comment Idea Filter                             | ## 3. Analyze Comments with OpenAI<br>Outputs "Yes" or "No" to flag viable content ideas.                                                             |
| Comment Idea Filter                                   | Filter                              | Filters comments flagged "Yes"          | Analyzing YouTube comments        | Store Raw Comments                             | ## 4. Filter “Yes” Comments<br>Only comments marked “Yes” continue further.                                                                            |
| Store Raw Comments                                    | Google Sheets AppendOrUpdate         | Saves raw comments and metadata         | Comment Idea Filter               | Loop Over Items                                | ## 5. Save Raw Comments to Google Sheets<br>Prevents duplicates using `id` as key. Stores comment details and content flag.                           |
| Loop Over Items                                      | SplitInBatches                      | Processes comments individually         | Store Raw Comments                | Extracting content topics from YouTube comments | ## 6. Split in Batches (Loop)<br>Processes comments one by one for sequential enrichment.                                                             |
| Extracting content topics from YouTube comments       | OpenAI (GPT-4.1-mini)                 | Extracts topic and initiates research   | Loop Over Items                   | Transforming a researched YouTube video topic  | ## 7. Research Topics with Tavily + OpenAI<br>Generates topic and 300-500 word research overview using Tavily for real-time data.                      |
| Research Context (Tavily)                             | LangChain Tool HTTP Request           | Performs web research                    | Extracting content topics from YouTube comments (as ai_tool) | Extracting content topics from YouTube comments | Part of 7. Research block                                                                                                                               |
| Transforming a researched YouTube video topic into a compelling video concept | OpenAI (GPT-4.1-mini)                 | Generates hook and outline from research | Extracting content topics from YouTube comments | Update Idea with Enriched Data                 | ## 8. Generate Hook & Outline (OpenAI)<br>Transforms researched topic into engaging hook and structured outline.                                        |
| Update Idea with Enriched Data                         | Google Sheets Update                  | Updates Google Sheet with enriched data | Transforming a researched YouTube video topic into a compelling video concept | -                                              | ## 9. Update Google Sheets with Enriched Data<br>Keeps each comment’s enriched output in one row for easy reference.                                   |
| Sticky Note, Sticky Note8, Sticky Note10, Sticky Note13, Sticky Note14, Sticky Note17, Sticky Note18, Sticky Note19, Sticky Note20 | Sticky Note                         | Documentation and instructions          | -                                | -                                              | Various sticky notes provide detailed explanations, usage instructions, and external links for nodes or workflow sections.                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Type: LangChain Chat Trigger  
   - Purpose: Listen for YouTube URL input via chat  
   - Configuration: Default webhook, output JSON contains `chatInput` with URL  

2. **Add HTTP Request Node for Comment Scraping**  
   - Name: Fetch YouTube Comments (Apify)  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/mExYO4A2k9976zMfA/run-sync-get-dataset-items`  
   - Headers:  
     - `Authorization: Bearer <Apify_Token>` (replace with your token)  
     - `Accept: application/json`  
   - Body (JSON):  
     ```json
     {
       "customMapFunction": "(object) => { return {...object} }",
       "maxItems": 100,
       "sort": "top",
       "startUrls": ["{{$json.chatInput}}"]
     }
     ```  
   - Connect: Chat Trigger → HTTP Request  

3. **Add OpenAI Node to Analyze Comments**  
   - Name: Analyzing YouTube comments  
   - Model: GPT-4.1-mini  
   - Prompt Template: Analyzes comment text (`{{ $json.text }}`) to output “Yes” or “No”  
   - Credentials: OpenAI API Key  
   - Connect: HTTP Request → OpenAI Analyze  

4. **Add Filter Node to Pass Only “Yes” Comments**  
   - Name: Comment Idea Filter  
   - Condition: `$json.choices[0].message.content.output == "Yes"`  
   - Connect: OpenAI Analyze → Filter  

5. **Add Google Sheets Node to Save Raw Comments**  
   - Name: Store Raw Comments  
   - Operation: Append or Update  
   - Document: Your Google Sheet with comment columns (`id`, `text`, `author`, `likeCount`, `replyCount`, `publishedTime`, `contentIdea`)  
   - Key Column: `id` to prevent duplicates  
   - Map fields from Apify comment data and AI flag  
   - Credentials: Google Sheets OAuth2  
   - Connect: Filter → Google Sheets Append  

6. **Add SplitInBatches Node**  
   - Name: Loop Over Items  
   - Batch Size: 1 (default)  
   - Connect: Google Sheets Append → SplitInBatches  

7. **Add OpenAI Node to Extract Topic & Research Instruction**  
   - Name: Extracting content topics from YouTube comments  
   - Model: GPT-4.1-mini  
   - Prompt: Extracts topic from comment text and instructs Tavily research  
   - Output: JSON with `topic`, `research_overview`  
   - Credentials: OpenAI API Key  
   - Connect: Loop Over Items → OpenAI Extraction  

8. **Add HTTP Request Node (LangChain Tool) for Tavily Research**  
   - Name: Research Context (Tavily)  
   - Method: POST  
   - URL: `https://api.tavily.com/search`  
   - Headers:  
     - `Authorization: Bearer <Tavily_Token>`  
     - `Content-Type: application/json`  
   - Body (JSON):  
     ```json
     {
       "query": "{searchTerm}",
       "topic": "general",
       "search_depth": "advanced",
       "chunks_per_source": 3,
       "max_results": 1,
       "time_range": null,
       "days": 7,
       "include_answer": true,
       "include_raw_content": false,
       "include_images": false,
       "include_image_descriptions": false,
       "include_domains": [],
       "exclude_domains": []
     }
     ```  
   - Connect as AI Tool to OpenAI Extraction node  

9. **Add OpenAI Node to Generate Hook & Outline**  
   - Name: Transforming a researched YouTube video topic into a compelling video concept  
   - Model: GPT-4.1-mini  
   - Prompt: Takes `topic` and `research_overview` to output JSON with `hook` and `outline`  
   - Credentials: OpenAI API Key  
   - Connect: OpenAI Extraction → OpenAI Transform  

10. **Add Google Sheets Node to Update Rows with Enriched Data**  
    - Name: Update Idea with Enriched Data  
    - Operation: Update  
    - Document: Same Google Sheet as raw comments  
    - Key Column: `id`  
    - Map fields: `topic`, `research` (research_overview), `hook`, and `outline` from AI outputs  
    - Credentials: Google Sheets OAuth2  
    - Connect: OpenAI Transform → Google Sheets Update  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This n8n template demonstrates how to turn **YouTube comments into researched content ideas with hooks and outlines**. Use cases include repurposing YouTube discussions into videos, blogs, or social media content while storing ideas in Google Sheets.                                                                                                                                       | Linked sticky note at start of workflow                                                                           |
| Join the [n8n Discord Community](https://discord.com/invite/XPKeKXeB7d) or the [n8n Forum](https://community.n8n.io/) for support and questions.                                                                                                                                                                                                                                              | Community support links                                                                                          |
| Google Sheets nodes require OAuth2 credentials with edit access to the specific spreadsheet used for comment storage and enrichment.                                                                                                                                                                                                                                                         | Credential setup note                                                                                            |
| Apify API requires an API token with access to the YouTube Comments Scraper act.                                                                                                                                                                                                                                                                                                               | Apify API token setup                                                                                           |
| OpenAI nodes require an API key with access to GPT-4.1-mini or similar model.                                                                                                                                                                                                                                                                                                                 | OpenAI API key setup                                                                                            |
| Tavily web research API requires an authorization token. This node is used for real-time, advanced research to enrich content ideas with factual and current data.                                                                                                                                                                                                                          | Tavily API token setup                                                                                           |
| The workflow is designed to avoid duplicates in Google Sheets using the comment `id` as key. Make sure your sheet schema matches the expected columns.                                                                                                                                                                                                                                      | Google Sheets schema note                                                                                        |
| Handling API rate limits, network failures, or empty results gracefully may require adding error handling or retry logic outside this basic workflow.                                                                                                                                                                                                                                         | Recommended enhancements                                                                                         |

---

**Disclaimer:**  
The provided documentation is based exclusively on an n8n automation workflow. It complies with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.