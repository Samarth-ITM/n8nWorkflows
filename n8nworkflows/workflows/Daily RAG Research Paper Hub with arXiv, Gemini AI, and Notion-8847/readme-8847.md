Daily RAG Research Paper Hub with arXiv, Gemini AI, and Notion

https://n8nworkflows.xyz/workflows/daily-rag-research-paper-hub-with-arxiv--gemini-ai--and-notion-8847


# Daily RAG Research Paper Hub with arXiv, Gemini AI, and Notion

---

### 1. Workflow Overview

This workflow automates the daily retrieval, analysis, categorization, and dissemination of research papers related to Retrieval-Augmented Generation (RAG) from arXiv. It integrates AI-powered analysis with Google Gemini and LangChain, stores enriched data in Notion databases, and distributes daily summaries via Gmail and Feishu messaging platforms.

Logical blocks include:

- **1.1 Scheduled Data Retrieval:** Daily triggering and querying arXiv API for papers submitted on the previous day with RAG relevance.
- **1.2 XML Data Extraction and Transformation:** Parsing and cleaning the arXiv Atom XML feed into structured JSON objects.
- **1.3 AI Analysis and Enrichment:** Applying AI models to analyze paper summaries, extract RAG relevance, categorize papers, and enrich metadata.
- **1.4 Data Storage:** Inserting enriched paper entries and daily summaries into Notion databases.
- **1.5 Message Construction and Delivery:** Composing daily summary messages in HTML and text formats, then sending via Gmail and Feishu IM.
- **1.6 Conditional Routing and Error Handling:** Conditional flows to handle cases such as zero papers or API errors.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Data Retrieval

**Overview:**  
This block schedules the workflow to run daily at 6:00 AM, calculates the previous day’s date range, and queries the arXiv API for research papers related to RAG submitted within that time frame.

**Nodes Involved:**  
- Schedule Trigger  
- submittedDate:T-1 (Code)  
- arXiv API (HTTP Request)  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow every day at 06:00 AM.  
  - Config: Hourly trigger set to 6:00 AM daily.  
  - Inputs: None  
  - Outputs: Triggers next node.  
  - Edge Cases: Missed triggers if n8n instance is offline or overloaded.

- **submittedDate:T-1**  
  - Type: Code (JavaScript)  
  - Role: Computes date range for query (previous day 00:00 to 23:59).  
  - Config: Uses current date, shifts back two days (likely to adjust for arXiv delay), formats strings as `YYYYMMDD0000` and `YYYYMMDD2359`.  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: JSON with `from` and `to` fields for API query.  
  - Edge Cases: Timezone assumptions; date shifting logic may introduce off-by-one errors if arXiv update rules change.

- **arXiv API**  
  - Type: HTTP Request  
  - Role: Queries arXiv API with search term `RAG` and submittedDate between calculated `from` and `to`.  
  - Config: GET request to `https://export.arxiv.org/api/query?search_query=all:RAG+AND+submittedDate:[from+TO+to]` with dynamic parameters.  
  - Inputs: Date range from previous node.  
  - Outputs: Atom XML response containing papers metadata.  
  - Edge Cases: API rate limits, network errors, malformed responses, empty results.

---

#### 2.2 XML Data Extraction and Transformation

**Overview:**  
This block parses the arXiv Atom XML feed, extracts relevant fields for each paper entry, cleans and structures the data into JSON objects suitable for downstream processing.

**Nodes Involved:**  
- Data Extraction (Code)  
- Basic LLM Chain (LangChain LLM Chain Node)  

**Node Details:**

- **Data Extraction**  
  - Type: Code (JavaScript)  
  - Role: Parses XML to extract paper entries `<entry>`, then extracts fields: id, updated, published, title, summary, authors (array), html_url, pdf_url, primary_category, categories (array), arxiv_comment (ignored), github (empty), huggingface (empty).  
  - Config: Regex-based XML parsing, date formatting to `YYYY-MM-DD HH:mm:ss` UTC, array extraction for authors and categories for Notion compatibility.  
  - Inputs: XML data from arXiv API.  
  - Outputs: JSON array with cleaned paper metadata.  
  - Edge Cases: XML parsing failure, missing tags, malformed XML, empty `<entry>` nodes.

- **Basic LLM Chain**  
  - Type: LangChain LLM Chain Node  
  - Role: AI-powered analysis of each paper’s summary to:  
    1. Determine RAG relevance and assign `RAG_TF`, `RAG_REASON`, `RAG_Category`.  
    2. Extract RAG method name into `RAG_NAME`.  
    3. Extract GitHub and HuggingFace URLs from summary text, populating corresponding fields.  
  - Config: Uses Google Gemini model via LangChain, prompt designed to analyze JSON input field `summary`. Outputs enriched JSON per paper.  
  - Inputs: Array of paper JSON objects from Data Extraction.  
  - Outputs: Enriched paper JSON with RAG tagging and extracted metadata.  
  - Edge Cases: AI model errors, incomplete or ambiguous summaries, missing URLs, rate limits, malformed JSON parsing.

---

#### 2.3 Daily Summary Generation and Formatting

**Overview:**  
Generates a daily summary of all retrieved papers, including multilingual (Chinese and English) summaries, paper count, and date. The output is structured JSON for storage and message generation.

**Nodes Involved:**  
- Message a model (Google Gemini Chat Model)  
- JSON FORMAT (Code)  
- RAG Daily Paper Summary (Notion)  
- If (Conditional)  
- Message Construction (Code)  

**Node Details:**

- **Message a model**  
  - Type: LangChain Google Gemini Chat Model  
  - Role: Summarizes all papers’ XML data into a daily report JSON containing title, date, number of papers, Chinese summary (`SUMMARY_CN`), and English summary (`SUMMARY_EN`).  
  - Config: Uses model `models/gemini-2.5-flash-lite`, prompt instructs extraction of total papers, date, and bilingual summaries.  
  - Inputs: Raw XML data from arXiv API.  
  - Outputs: AI-generated JSON text embedded inside a larger response structure.  
  - Edge Cases: Model timeout, incomplete summaries, malformed output.

- **JSON FORMAT**  
  - Type: Code (JavaScript)  
  - Role: Extracts AI response text, parses JSON inside code blocks, handles duplicated keys manually, and structures fields: title, date, paperCount, summaryCN, summaryEN.  
  - Inputs: AI model response.  
  - Outputs: Clean JSON for downstream use.  
  - Edge Cases: Parsing errors if AI output is malformed; missing fields.

- **RAG Daily Paper Summary**  
  - Type: Notion (Create Database Page)  
  - Role: Stores daily summary data into a Notion database with fields: Title, Date, Number of papers, SUMMARY_EN, SUMMARY_CN.  
  - Config: Maps JSON fields to Notion properties, uses Notion API credentials.  
  - Inputs: Parsed daily summary JSON.  
  - Outputs: Confirmation of page creation.  
  - Edge Cases: Notion API rate limits, missing required fields, invalid data types.

- **If**  
  - Type: Conditional  
  - Role: Checks if `paperCount` is not zero to conditionally proceed with message construction and delivery.  
  - Config: Condition: `paperCount !== 0` (strict numeric check).  
  - Inputs: Daily summary JSON.  
  - Outputs: Routes flow to messaging nodes or terminates early.  
  - Edge Cases: Zero papers, missing count field.

- **Message Construction**  
  - Type: Code (JavaScript)  
  - Role: Builds message payloads for Gmail (HTML email) and Feishu (text message) using daily summary data.  
  - Config:  
    - Gmail message: HTML template with embedded CSS, bilingual summaries, paper count, and date.  
    - Feishu message: Plain text with paper count and summaries.  
  - Inputs: Daily summary JSON.  
  - Outputs: Two JSON objects, one for Gmail, one for Feishu, each tagged with a `type` field.  
  - Edge Cases: Missing summary fields, malformed HTML, encoding issues.

---

#### 2.4 Data Storage for Individual Papers

**Overview:**  
Stores each enriched paper’s data as a separate page in a dedicated Notion database, including metadata, RAG labels, categories, and extracted URLs.

**Nodes Involved:**  
- JSON Format (Code)  
- RAG Daily papers (Notion)  

**Node Details:**

- **JSON Format**  
  - Type: Code (JavaScript)  
  - Role: Similar to Data Extraction node; possibly a duplicate or fallback transformer to ensure data is well-formatted before Notion insertion.  
  - Inputs: Enriched paper JSON from Basic LLM Chain.  
  - Outputs: Cleaned JSON entries for each paper.  
  - Edge Cases: Data inconsistency between nodes; failure to parse.

- **RAG Daily papers**  
  - Type: Notion (Create Database Page)  
  - Role: Inserts each paper as a new page with mapped properties including IDs, URLs, timestamps, authors as multi-select, categories as multi-select, RAG labels, and extracted links.  
  - Config: Maps multiple fields, including array fields for multi-select Notion properties.  
  - Inputs: JSON array of enriched papers.  
  - Outputs: Confirmation of page creation in Notion.  
  - Edge Cases: Notion API errors, invalid data types (nulls), multi-select formatting errors.

---

#### 2.5 Message Delivery

**Overview:**  
Routes constructed messages to appropriate delivery channels: Gmail for email and Feishu for instant messaging.

**Nodes Involved:**  
- gmail (Switch)  
- Send a message (Gmail Node)  
- FEISHU (Switch)  
- FEISHU POST (HTTP Request)  

**Node Details:**

- **gmail (Switch)**  
  - Type: Switch  
  - Role: Routes messages with `"type" == "gmail"` to Gmail node.  
  - Inputs: Message Construction output.  
  - Outputs: Routes matching messages to Send a message node.  
  - Edge Cases: Misrouting if type field missing.

- **Send a message**  
  - Type: Gmail Node  
  - Role: Sends email with OAuth2 credentials configured, message content is HTML email.  
  - Config: Uses Gmail OAuth2 credentials; sendTo is hardcoded to `xing.adam@gmail.com`.  
  - Inputs: Message from switch.  
  - Outputs: Confirmation or error of email sent.  
  - Edge Cases: OAuth token expiration, Gmail API limits, invalid email addresses.

- **FEISHU (Switch)**  
  - Type: Switch  
  - Role: Routes messages with `"type" == "feishu"` to Feishu POST node.  
  - Inputs: Message Construction output.  
  - Outputs: Routes matching messages to HTTP request node.  
  - Edge Cases: Misrouting if type field missing.

- **FEISHU POST**  
  - Type: HTTP Request  
  - Role: Sends POST request to Feishu API to post text message.  
  - Config: URL is dynamic; payload contains `msg_type` and `content` as per Feishu bot API.  
  - Inputs: JSON message from switch.  
  - Outputs: API response.  
  - Edge Cases: Invalid token or URL, network errors, API limits.

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                           | Input Node(s)              | Output Node(s)                       | Sticky Note                                                                                  |
|------------------------|----------------------------------|-----------------------------------------|----------------------------|------------------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger                  | Initiate daily workflow at 6:00 AM      | None                       | submittedDate:T-1                  | See Sticky Note3: Data Retrieval, arXiv API details                                           |
| submittedDate:T-1      | Code                             | Compute previous day's date range       | Schedule Trigger           | arXiv API                        | See Sticky Note3: arXiv API query specifics                                                  |
| arXiv API             | HTTP Request                     | Query arXiv for RAG papers by date      | submittedDate:T-1          | Message a model                   | See Sticky Note3: arXiv API response format                                                  |
| Message a model        | LangChain Google Gemini Model    | Generate daily summary and bilingual overview | arXiv API                | JSON FORMAT                      | See Sticky Note1: Data Processing - summarization and translation                            |
| JSON FORMAT            | Code                             | Parse AI response JSON for daily summary | Message a model            | RAG Daily Paper Summary, If       |                                                                                              |
| RAG Daily Paper Summary | Notion (Create Database Page)    | Store daily summary in Notion database  | JSON FORMAT                | If                              | See Sticky Note2: Notion database setup and field mapping                                    |
| If                    | Conditional                     | Route flow if papers exist (paperCount != 0) | RAG Daily Paper Summary    | Message Construction, end flow    |                                                                                              |
| Message Construction   | Code                             | Compose Gmail HTML and Feishu messages  | If                        | gmail, FEISHU                    | See Sticky Note5: Message Push instructions, Gmail OAuth2 setup, Feishu Bot usage            |
| gmail                  | Switch                          | Route Gmail messages                     | Message Construction       | Send a message                   |                                                                                              |
| Send a message         | Gmail Node                      | Send email via Gmail OAuth2              | gmail                      | End                             |                                                                                              |
| FEISHU                 | Switch                          | Route Feishu messages                    | Message Construction       | FEISHU POST                     |                                                                                              |
| FEISHU POST            | HTTP Request                   | Send message to Feishu API               | FEISHU                     | End                             |                                                                                              |
| Data Extraction        | Code                             | Parse arXiv XML entries into JSON array | If                        | Basic LLM Chain                 | See Sticky Note4: Data extraction and cleaning rules from arXiv Atom XML                     |
| Basic LLM Chain        | LangChain LLM Chain             | Analyze paper summaries, tag RAG relevance, extract URLs | Data Extraction        | JSON Format                    | See Sticky Note1: AI-assisted content analysis and enrichment                                |
| JSON Format            | Code                             | Format enriched paper JSON for Notion   | Basic LLM Chain            | RAG Daily papers                |                                                                                              |
| RAG Daily papers       | Notion (Create Database Page)    | Store each paper as a Notion page       | JSON Format                | End                             | See Sticky Note2: Notion storage details and field mapping                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 6:00 AM.

2. **Create Code Node "submittedDate:T-1"**  
   - JavaScript to calculate previous day's date range (`from` and `to` in `YYYYMMDD0000` and `YYYYMMDD2359` format).  
   - Connect Schedule Trigger → submittedDate:T-1.

3. **Create HTTP Request Node "arXiv API"**  
   - Method: GET  
   - URL: `https://export.arxiv.org/api/query?search_query=all:RAG+AND+submittedDate:[{{$json["from"]}}+TO+{{$json["to"]}}]`  
   - Use query parameters as dynamic inputs from previous node.  
   - Connect submittedDate:T-1 → arXiv API.

4. **Create LangChain Chat Model Node "Message a model"**  
   - Model: Google Gemini `models/gemini-2.5-flash-lite`  
   - Prompt: Instructions to parse XML and generate daily summary JSON (title, date, paper count, Chinese and English summaries).  
   - Credentials: Google Palm API account.  
   - Connect arXiv API → Message a model.

5. **Create Code Node "JSON FORMAT" (Daily Summary)**  
   - JavaScript to extract and parse JSON from AI model output (handle embedded code block).  
   - Connect Message a model → JSON FORMAT.

6. **Create Notion Node "RAG Daily Paper Summary"**  
   - Operation: Create Database Page  
   - Database: Daily summary database ID in Notion  
   - Map fields: title, date, number of papers, SUMMARY_EN, SUMMARY_CN.  
   - Credentials: Notion API integration.  
   - Connect JSON FORMAT → RAG Daily Paper Summary.

7. **Create Conditional Node "If"**  
   - Condition: paperCount (or Number of papers) not equal to 0.  
   - Connect RAG Daily Paper Summary → If.

8. **Create Code Node "Message Construction"**  
   - Compose Gmail HTML email and Feishu text message using daily summary data.  
   - Connect If (true) → Message Construction.

9. **Create Switch Node "gmail"**  
   - Condition: `$json.type === 'gmail'`  
   - Connect Message Construction → gmail.

10. **Create Gmail Node "Send a message"**  
    - Configure Gmail OAuth2 credentials.  
    - Recipient hardcoded or configurable email address.  
    - Connect gmail → Send a message.

11. **Create Switch Node "FEISHU"**  
    - Condition: `$json.type === 'feishu'`  
    - Connect Message Construction → FEISHU.

12. **Create HTTP Request Node "FEISHU POST"**  
    - POST to Feishu bot webhook URL (set dynamic or fixed).  
    - Payload includes `msg_type` and `content` fields.  
    - Connect FEISHU → FEISHU POST.

13. **Create Code Node "Data Extraction"**  
    - JavaScript to parse arXiv XML `<entry>` blocks into JSON array.  
    - Extract fields: id, updated, published, title, summary, authors (array), html_url, pdf_url, primary_category, categories (array), empty github and huggingface fields.  
    - Connect If (false branch or arXiv API) → Data Extraction.

14. **Create LangChain LLM Chain Node "Basic LLM Chain"**  
    - Model: Google Gemini via LangChain chain LLM node.  
    - Prompt instructs to analyze paper summaries to tag RAG relevance, extract method names, and external URLs.  
    - Credentials: Google Palm API account.  
    - Connect Data Extraction → Basic LLM Chain.

15. **Create Code Node "JSON Format" (Paper Data)**  
    - Similar XML parsing or data formatting to ensure clean JSON for Notion.  
    - Connect Basic LLM Chain → JSON Format.

16. **Create Notion Node "RAG Daily papers"**  
    - Operation: Create Database Page  
    - Database: Paper details database ID in Notion  
    - Map all paper fields including arrays for multi-select fields.  
    - Credentials: Notion API integration.  
    - Connect JSON Format → RAG Daily papers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| **arXiv API User Manual:** Covers API usage, response format, update frequency, and query parameters.                                                                                                                                                                                                                                                                                   | [https://info.arxiv.org/help/api/user-manual.html#arxiv-api-users-manual](https://info.arxiv.org/help/api/user-manual.html#arxiv-api-users-manual) |
| **Scheduled Task Setup:** Daily execution at 6:00 AM to fetch previous day's papers due to arXiv's daily update cycle at midnight EST.                                                                                                                                                                                                                                                  | Sticky Note3 in workflow                                                                                                                     |
| **Data Extraction Rules:** Detailed XML tag extraction, date formatting, array handling for authors and categories, ignoring comments, and adding empty github/huggingface fields for later AI enrichment.                                                                                                                                                                              | Sticky Note4 in workflow                                                                                                                     |
| **Notion API Integration:** Create database pages only (no update), multi-select fields require arrays, avoid null values to prevent 400 errors.                                                                                                                                                                                                                                         | Sticky Note2 in workflow                                                                                                                     |
| **Message Push Channels:** Gmail uses OAuth2 with HTML email templates; Feishu uses bot API with text messages. Instructions included for OAuth consent screen configuration and Feishu bot usage in groups.                                                                                                                                                                             | Sticky Note5 in workflow                                                                                                                     |
| **AI Models Used:** Google Gemini (PaLM) models `gemini-2.5-flash` and `gemini-2.5-flash-lite` via LangChain nodes for paper summary and enrichment tasks.                                                                                                                                                                                                                                | Workflow node credentials and configurations                                                                                                |

---

**Disclaimer:** The provided text is derived solely from an automated workflow created with n8n, a no-code automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---