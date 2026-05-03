Transform YouTube Transcripts to Content Assets with AI and Google Sheets

https://n8nworkflows.xyz/workflows/transform-youtube-transcripts-to-content-assets-with-ai-and-google-sheets-9213


# Transform YouTube Transcripts to Content Assets with AI and Google Sheets

### 1. Workflow Overview

This workflow automates the extraction, processing, rewriting, and saving of YouTube video transcripts into Google Sheets, leveraging AI language models for content enhancement. It supports two main input modes: monitoring a Google Sheet for new YouTube URLs and receiving direct webhook inputs. The workflow parses the video ID, fetches the transcript via HTTP requests, rewrites the transcript with AI assistance, formats the output into structured headings, and saves the results back into Google Sheets. The workflow is logically divided into two parallel pipelines corresponding to these input modes, sharing similar processing steps but triggered differently.

Logical blocks are:

- **1.1 Input Reception**
  - Google Sheets trigger (new URLs added)
  - Webhook trigger (direct requests)

- **1.2 YouTube Transcript Extraction**
  - Extract video ID
  - HTTP request to fetch transcript data
  - Parse raw transcript text

- **1.3 AI Transcript Rewriting**
  - Invoke OpenRouter Chat Model for rewriting
  - Parse AI output into structured headings

- **1.4 Data Persistence**
  - Save rewritten transcript scripts to Google Sheets
  - Save raw transcripts to Google Sheets

- **1.5 Response Handling**
  - Return processed transcript data via webhook response

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures input URLs either by monitoring a Google Sheet or via direct webhook calls, triggering the downstream processing of YouTube transcripts.

**Nodes Involved:**  
- Monitor Google Sheet for URLs  
- Webhook Trigger (Direct Input)

**Node Details:**

- **Monitor Google Sheet for URLs**  
  - *Type:* Google Sheets Trigger  
  - *Role:* Watches a specific Google Sheet for new rows or changes containing YouTube URLs to process.  
  - *Configuration:* Set to trigger on sheet changes, likely filtered to URL columns.  
  - *Input/Output:* No input; outputs new row data.  
  - *Edge cases:* Missed triggers if sheet API rate limited or offline; invalid URLs may propagate downstream.

- **Webhook Trigger (Direct Input)**  
  - *Type:* Webhook  
  - *Role:* Listens for HTTP requests with YouTube URLs or video IDs for immediate processing.  
  - *Configuration:* Webhook ID "extract-youtube-transcript" for external calls.  
  - *Input/Output:* No input; outputs request body data.  
  - *Edge cases:* Unauthorized calls, malformed payloads, timeouts on slow clients.

---

#### 2.2 YouTube Transcript Extraction

**Overview:**  
Extracts the YouTube video ID from input, fetches the transcript data via HTTP, and parses the raw transcript text for further processing.

**Nodes Involved:**  
- Extract YouTube Video ID (Sheets)  
- Extract YouTube Video ID (Sheets)2  
- Fetch Video Transcript Data (Sheets)  
- Fetch Video Transcript Data (Webhook)  
- Parse Transcript Text (Sheets)  
- Parse Transcript Text (Webhook)

**Node Details:**

- **Extract YouTube Video ID (Sheets)** and **Extract YouTube Video ID (Sheets)2**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses input URLs or text to extract the YouTube video ID.  
  - *Configuration:* Custom code to handle various YouTube URL formats.  
  - *Input:* URL strings from Google Sheet rows or webhook data.  
  - *Output:* Video ID string.  
  - *Edge cases:* Invalid URLs, missing ID, malformed inputs cause extraction failure.

- **Fetch Video Transcript Data (Sheets)** and **Fetch Video Transcript Data (Webhook)**  
  - *Type:* HTTP Request  
  - *Role:* Calls an external API or service to retrieve the transcript for the given video ID.  
  - *Configuration:* Likely configured with URL templates containing the extracted video ID; possibly includes API keys or tokens.  
  - *Input:* Video ID from prior node.  
  - *Output:* Raw transcript data (likely JSON or text).  
  - *Edge cases:* Network errors, API rate limits, transcript not available, invalid video ID.

- **Parse Transcript Text (Sheets)** and **Parse Transcript Text (Webhook)**  
  - *Type:* Code (JavaScript)  
  - *Role:* Processes the raw transcript data to extract clean text, removing metadata or formatting noise.  
  - *Input:* Raw transcript data.  
  - *Output:* Cleaned transcript text string.  
  - *Edge cases:* Unexpected transcript format, empty data, parsing exceptions.

---

#### 2.3 AI Transcript Rewriting

**Overview:**  
Uses OpenRouter’s chat model to rewrite the transcript, making it more suitable as a content asset. Then parses the AI output into structured headings for better organization.

**Nodes Involved:**  
- OpenRouter Chat Model  
- OpenRouter Chat Model1  
- Rewrite The Transcript (sheets)  
- Rewrite The Transcript (webhook)  
- Parse AI Output Into Headings(sheets)  
- Parse AI Output Into Headings (webhook)

**Node Details:**

- **OpenRouter Chat Model** and **OpenRouter Chat Model1**  
  - *Type:* Langchain OpenRouter LLM Chat Nodes  
  - *Role:* Sends prompt with transcript text to AI model for rewriting.  
  - *Configuration:* Connected with appropriate OpenRouter credentials; prompt templates likely used inside chain nodes.  
  - *Input:* Clean transcript text.  
  - *Output:* AI rewritten transcript text.  
  - *Edge cases:* API auth failures, timeouts, unexpected AI output format.

- **Rewrite The Transcript (sheets)** and **Rewrite The Transcript (webhook)**  
  - *Type:* Langchain Chain LLM  
  - *Role:* Defines the AI prompt and processing chain to rewrite the transcript.  
  - *Configuration:* Contains prompt templates and possibly parameters for tone, style, or length.  
  - *Input:* Cleaned transcript text.  
  - *Output:* Rewritten transcript text.  
  - *Edge cases:* Prompt errors, chain execution failures, malformed outputs.

- **Parse AI Output Into Headings(sheets)** and **Parse AI Output Into Headings (webhook)**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses the AI output text into a structured format with headings/subheadings for better usability.  
  - *Input:* AI rewritten transcript text.  
  - *Output:* Structured data ready for sheet insertion.  
  - *Edge cases:* Unexpected AI text structure, parsing errors.

---

#### 2.4 Data Persistence

**Overview:**  
Saves the rewritten transcripts and raw transcripts into designated Google Sheets for record-keeping and further use.

**Nodes Involved:**  
- Save New Script to Sheet  
- Save New Script to Sheet(webhook)  
- Save Transcript to Sheet  
- Save Transcript to Sheet(webhook)

**Node Details:**

- **Save New Script to Sheet** and **Save New Script to Sheet(webhook)**  
  - *Type:* Google Sheets  
  - *Role:* Inserts the rewritten, structured transcript content into a Google Sheet.  
  - *Configuration:* Configured with target spreadsheet ID, worksheet, and mapping fields.  
  - *Input:* Structured rewritten transcript data.  
  - *Output:* Confirmation of row insertion.  
  - *Edge cases:* Google API quota limits, incorrect sheet access permissions, data mapping errors.

- **Save Transcript to Sheet** and **Save Transcript to Sheet(webhook)**  
  - *Type:* Google Sheets  
  - *Role:* Stores raw or initial transcript text into Google Sheets.  
  - *Configuration:* Target spreadsheet and fields configured accordingly.  
  - *Input:* Clean transcript text or raw transcript data.  
  - *Output:* Row insertion confirmation.  
  - *Edge cases:* Same as above.

---

#### 2.5 Response Handling

**Overview:**  
For the webhook-triggered pipeline, responds back to the caller with the processed transcript data to complete the request-response cycle.

**Nodes Involved:**  
- Return Transcript Response

**Node Details:**

- **Return Transcript Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends the final processed transcript data back to the HTTP request originator.  
  - *Configuration:* Configured to output the last node’s data as HTTP response.  
  - *Input:* Processed transcript data from prior Google Sheets save node.  
  - *Output:* HTTP response to webhook caller.  
  - *Edge cases:* Timeout if response delayed, malformed response data, client disconnects.

---

### 3. Summary Table

| Node Name                         | Node Type                             | Functional Role                 | Input Node(s)                      | Output Node(s)                      | Sticky Note                                               |
|----------------------------------|-------------------------------------|--------------------------------|----------------------------------|----------------------------------|-----------------------------------------------------------|
| Main Template Explanation         | Sticky Note                         | Documentation placeholder      |                                  |                                  |                                                           |
| Sheets Workflow Note              | Sticky Note                         | Documentation placeholder      |                                  |                                  |                                                           |
| Webhook Workflow Note             | Sticky Note                         | Documentation placeholder      |                                  |                                  |                                                           |
| Extract YouTube Video ID (Sheets) | Code                               | Extract video ID from URLs     | Monitor Google Sheet for URLs    | Fetch Video Transcript Data (Sheets) |                                                           |
| Fetch Video Transcript Data (Sheets) | HTTP Request                     | Retrieve transcript data       | Extract YouTube Video ID (Sheets) | Parse Transcript Text (Sheets)    |                                                           |
| Parse Transcript Text (Sheets)    | Code                               | Clean raw transcript text      | Fetch Video Transcript Data (Sheets) | Rewrite The Transcript (sheets)  |                                                           |
| Save Transcript to Sheet          | Google Sheets                      | Save raw transcript text       | Save New Script to Sheet         |                                  |                                                           |
| Monitor Google Sheet for URLs     | Google Sheets Trigger              | Trigger on new URLs            |                                  | Extract YouTube Video ID (Sheets) |                                                           |
| OpenRouter Chat Model             | Langchain OpenRouter LLM Chat Node | AI rewriting model             | Rewrite The Transcript (sheets) | Rewrite The Transcript (sheets)  |                                                           |
| Return Transcript Response        | Respond to Webhook                 | Send webhook response          | Save Transcript to Sheet(webhook) |                                  |                                                           |
| Webhook Trigger (Direct Input)    | Webhook                           | Receive direct input requests  |                                  | Extract YouTube Video ID (Sheets)2 |                                                           |
| Save New Script to Sheet          | Google Sheets                     | Save AI rewritten transcript  | Parse AI Output Into Headings(sheets) | Save Transcript to Sheet          |                                                           |
| OpenRouter Chat Model1            | Langchain OpenRouter LLM Chat Node | AI rewriting model (webhook)  | Rewrite The Transcript (webhook) | Rewrite The Transcript (webhook) |                                                           |
| Rewrite The Transcript (webhook)  | Langchain Chain LLM               | AI rewriting chain (webhook)   | Parse Transcript Text (Webhook)  | Parse AI Output Into Headings (webhook) |                                                           |
| Rewrite The Transcript (sheets)   | Langchain Chain LLM               | AI rewriting chain (sheets)    | Parse Transcript Text (Sheets)   | Parse AI Output Into Headings(sheets) |                                                           |
| Extract YouTube Video ID (Sheets)2 | Code                             | Extract video ID (webhook)     | Webhook Trigger (Direct Input)  | Fetch Video Transcript Data (Webhook) |                                                           |
| Fetch Video Transcript Data (Webhook) | HTTP Request                   | Retrieve transcript data       | Extract YouTube Video ID (Sheets)2 | Parse Transcript Text (Webhook)  |                                                           |
| Parse Transcript Text (Webhook)   | Code                             | Clean raw transcript text      | Fetch Video Transcript Data (Webhook) | Rewrite The Transcript (webhook) |                                                           |
| Parse AI Output Into Headings(sheets) | Code                         | Format AI output to headings   | Rewrite The Transcript (sheets) | Save New Script to Sheet          |                                                           |
| Parse AI Output Into Headings (webhook) | Code                         | Format AI output to headings   | Rewrite The Transcript (webhook) | Save New Script to Sheet(webhook) |                                                           |
| Save New Script to Sheet(webhook) | Google Sheets                   | Save AI rewritten transcript  | Parse AI Output Into Headings (webhook) | Save Transcript to Sheet(webhook) |                                                           |
| Save Transcript to Sheet(webhook) | Google Sheets                   | Save raw transcript text       | Save New Script to Sheet(webhook) | Return Transcript Response        |                                                           |
| Sticky Note1                     | Sticky Note                       | Documentation placeholder      |                                  |                                  |                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Type: Google Sheets Trigger  
   - Configure to watch a specific spreadsheet and worksheet for new YouTube URL entries.

2. **Add Code Node: Extract YouTube Video ID (Sheets)**  
   - Purpose: Parse the URL from Google Sheets to extract the video ID.  
   - Code: Implement regex or string parsing to handle various YouTube URL formats.

3. **Add HTTP Request Node: Fetch Video Transcript Data (Sheets)**  
   - Configure the request URL using the extracted video ID.  
   - Set method to GET and define any required headers or authentication.

4. **Add Code Node: Parse Transcript Text (Sheets)**  
   - Extracts clean transcript text from the raw API response.  
   - Handle JSON parsing and data cleanup.

5. **Add Langchain Chain LLM Node: Rewrite The Transcript (sheets)**  
   - Configure AI prompt templates focused on rewriting transcripts for content assets.  
   - Connect OpenRouter Chat Model node with required API credentials.

6. **Add Langchain OpenRouter Chat Model Node**  
   - Enter OpenRouter API credentials.  
   - Connect as language model node for the chain.

7. **Add Code Node: Parse AI Output Into Headings(sheets)**  
   - Parses AI output into structured headings/subheadings.  
   - Write code to split text into sections.

8. **Add Google Sheets Node: Save New Script to Sheet**  
   - Configure to write structured AI output into target spreadsheet.  
   - Map fields accordingly.

9. **Add Google Sheets Node: Save Transcript to Sheet**  
   - Configure to save raw or cleaned transcript text into a sheet.

10. **Connect the above nodes in the order:**  
    Monitor Google Sheet for URLs → Extract YouTube Video ID (Sheets) → Fetch Video Transcript Data (Sheets) → Parse Transcript Text (Sheets) → Rewrite The Transcript (sheets) → OpenRouter Chat Model → Parse AI Output Into Headings(sheets) → Save New Script to Sheet → Save Transcript to Sheet

---

11. **Create Webhook Trigger Node (Direct Input)**  
    - Configure webhook with unique ID (e.g., "extract-youtube-transcript") to accept direct HTTP requests with YouTube URLs.

12. **Add Code Node: Extract YouTube Video ID (Sheets)2**  
    - Similar extraction logic as node 2 but for webhook input.

13. **Add HTTP Request Node: Fetch Video Transcript Data (Webhook)**  
    - Same as node 3 but triggered by webhook input.

14. **Add Code Node: Parse Transcript Text (Webhook)**  
    - Same logic as node 4 but for webhook.

15. **Add Langchain Chain LLM Node: Rewrite The Transcript (webhook)**  
    - Same as node 5 but handling webhook input.

16. **Add Langchain OpenRouter Chat Model Node (OpenRouter Chat Model1)**  
    - Same as node 6 but for webhook branch.

17. **Add Code Node: Parse AI Output Into Headings (webhook)**  
    - Same as node 7 but for webhook branch.

18. **Add Google Sheets Node: Save New Script to Sheet(webhook)**  
    - Save rewritten transcript for webhook inputs.

19. **Add Google Sheets Node: Save Transcript to Sheet(webhook)**  
    - Save raw transcript for webhook inputs.

20. **Add Respond to Webhook Node: Return Transcript Response**  
    - Send final processed transcript data back to the webhook caller.

21. **Connect the webhook branch nodes in order:**  
    Webhook Trigger (Direct Input) → Extract YouTube Video ID (Sheets)2 → Fetch Video Transcript Data (Webhook) → Parse Transcript Text (Webhook) → Rewrite The Transcript (webhook) → OpenRouter Chat Model1 → Parse AI Output Into Headings (webhook) → Save New Script to Sheet(webhook) → Save Transcript to Sheet(webhook) → Return Transcript Response

---

22. **Credentials Setup:**  
    - OpenRouter API credentials for AI nodes.  
    - Google OAuth2 credentials with access to required Google Sheets.  
    - Ensure HTTP Request nodes have any required API keys or tokens if external transcript API is protected.

23. **Default Values and Constraints:**  
    - URL parsing should handle various YouTube URL formats robustly.  
    - AI prompt templates should be tested for output consistency.  
    - Google Sheets nodes should have correct spreadsheet IDs and correct worksheet names.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                      |
|----------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The workflow integrates AI rewriting with YouTube transcript extraction for content creation. | Workflow purpose                                     |
| OpenRouter Chat Model nodes require valid API keys and network connectivity.                 | OpenRouter API info                                  |
| Google Sheets nodes require OAuth2 credentials with edit access to target spreadsheets.      | Google Sheets API docs: https://developers.google.com/sheets/api                           |
| Webhook Trigger enables external HTTP clients to invoke transcript extraction on demand.     | n8n Webhook docs: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/                        |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow export. It strictly adheres to content policies, containing no illegal, offensive, or protected material. All data processed is legal and public.