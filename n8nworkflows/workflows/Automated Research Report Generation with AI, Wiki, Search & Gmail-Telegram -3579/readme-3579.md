Automated Research Report Generation with AI, Wiki, Search & Gmail/Telegram 

https://n8nworkflows.xyz/workflows/automated-research-report-generation-with-ai--wiki--search---gmail-telegram--3579


# Automated Research Report Generation with AI, Wiki, Search & Gmail/Telegram 

### 1. Workflow Overview

This workflow automates the generation of professional research reports by aggregating data from multiple sources, refining queries, generating structured content with AI, converting it into a PDF report, and delivering it via Gmail or Telegram. It targets researchers, students, educators, and professionals who require quick, formatted research reports without manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Validation**: Receives and validates the user’s research query.
- **1.2 Query Refinement and Expansion**: Refines the input query and generates related search queries to broaden research scope.
- **1.3 Data Aggregation and Research AI Agent**: Uses multiple tools (Wikipedia, Google Search, SerpApi, News API) coordinated by an AI agent to gather comprehensive research data.
- **1.4 Data Parsing and Merging**: Parses AI outputs and merges data from different sources into a structured JSON.
- **1.5 Report Generation (HTML and PDF)**: Creates a styled HTML report and converts it into a PDF using an external API.
- **1.6 PDF Download and Delivery**: Downloads the generated PDF and sends it via Gmail and Telegram.
- **1.7 Metadata Storage**: Stores research metadata in Google Sheets for record-keeping.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

**Overview:**  
This block receives the initial research query and validates it to ensure it meets minimum requirements before processing.

**Nodes Involved:**  
- `When clicking ‘Test workflow’` (Manual Trigger)  
- `Input Validation` (Code)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual testing, triggers the workflow with a sample input.  
  - Configuration: No parameters; triggers manually.  
  - Inputs: None  
  - Outputs: Connects to `Input Validation`  
  - Edge Cases: None (manual trigger)  

- **Input Validation**  
  - Type: Code Node  
  - Role: Validates the input query string to ensure it is at least 3 characters long and trims/normalizes it.  
  - Configuration: JavaScript code checks `$input.all()[0].json.query` for length and trims it; throws error if invalid.  
  - Expressions: Uses `$input.all()[0].json.query` to access input query.  
  - Inputs: From `When clicking ‘Test workflow’`  
  - Outputs: JSON with `originalQuery`, `cleanedQuery` (lowercase, trimmed), and `timestamp`.  
  - Edge Cases: Throws error if query is missing or too short, preventing further execution.

---

#### 2.2 Query Refinement and Expansion

**Overview:**  
Refines the user’s query for better readability and generates 5 related search queries to broaden the research scope.

**Nodes Involved:**  
- `OpenAI Chat Model1` (OpenAI Language Model)  
- `Structured Output Parser` (AI Output Parser)  
- `Query Refiner` (AI Agent)  

**Node Details:**

- **OpenAI Chat Model1**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides language model capabilities to the `Query Refiner`.  
  - Configuration: Uses model `gpt-4o-mini` with OpenAI credentials.  
  - Inputs: None directly; used as language model by `Query Refiner`.  
  - Outputs: Language model responses.  
  - Edge Cases: API key issues, rate limits, or model unavailability.

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses the JSON output from the `Query Refiner` to ensure structured data extraction.  
  - Configuration: Uses a JSON schema example defining expected output with `topic` and `searchQueries` array.  
  - Inputs: Output from `Query Refiner` (AI Agent).  
  - Outputs: Parsed JSON object with refined topic and search queries.  
  - Edge Cases: Parsing errors if AI output deviates from schema.

- **Query Refiner**  
  - Type: LangChain Agent  
  - Role: Refines the input query and generates 5 related search queries focusing on different aspects of the topic.  
  - Configuration: Custom prompt instructing generation of 5 queries in JSON format; uses `OpenAI Chat Model1` and `Structured Output Parser`.  
  - Expressions: Uses `{{ $json.cleanedQuery }}` as input refined query.  
  - Inputs: From `Input Validation` node.  
  - Outputs: JSON with `topic` and `searchQueries` array.  
  - Edge Cases: AI model errors, prompt misinterpretation, or incomplete output.

---

#### 2.3 Data Aggregation and Research AI Agent

**Overview:**  
Performs comprehensive research using multiple data sources coordinated by an AI agent, including Wikipedia, Google Search, SerpApi (Google Scholar), and News API.

**Nodes Involved:**  
- `Simple Memory` (Memory Buffer)  
- `Research AI Agent` (LangChain Agent)  
- `OpenAI Chat Model` (OpenAI Language Model)  
- `Wikipedia` (HTTP Request Tool)  
- `Google Search Web` (HTTP Request Tool)  
- `SerpApi` (HTTP Request Tool with SerpApi credentials)  
- `Search News` (HTTP Request Tool)  

**Node Details:**

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains session context for the AI agent using a custom session key based on search queries.  
  - Configuration: Session key set to `={{ $json.output.searchQueries }}` to maintain context per query set.  
  - Inputs: From `Query Refiner` output.  
  - Outputs: Memory context to `Research AI Agent`.  
  - Edge Cases: Memory overflow or session key misconfiguration.

- **Research AI Agent**  
  - Type: LangChain Agent  
  - Role: Orchestrates research by querying multiple tools and aggregating results into structured JSON.  
  - Configuration: Custom system message instructing research steps, expected JSON output schema with fields like introduction, summary, key findings, news highlights, scholarly insights, Wikipedia summary, and sources.  
  - Expressions: Uses `{{ $json.output.topic }}` and `{{ $json.output.searchQueries.join(",") }}` for dynamic input.  
  - Inputs: Receives refined topic and search queries from `Query Refiner` and memory from `Simple Memory`.  
  - Outputs: Raw JSON string with research data.  
  - Edge Cases: API failures, incomplete data, or malformed JSON output.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides language model capabilities to `Research AI Agent`.  
  - Configuration: Model `gpt-4o-mini` with OpenAI credentials.  
  - Inputs: Used internally by `Research AI Agent`.  
  - Outputs: AI-generated content.  
  - Edge Cases: Same as other OpenAI nodes.

- **Wikipedia**  
  - Type: LangChain HTTP Request Tool  
  - Role: Fetches introductory extracts from Wikipedia for foundational knowledge.  
  - Configuration: Uses Wikipedia API with query parameter from refined query or input query.  
  - Inputs: Invoked by `Research AI Agent`.  
  - Outputs: Wikipedia extract JSON.  
  - Edge Cases: API rate limits, missing pages, or malformed queries.

- **Google Search Web**  
  - Type: LangChain HTTP Request Tool  
  - Role: Performs web search using Google Custom Search API for general information.  
  - Configuration: API key placeholder `"YOURAPIKEY"`, query encoded from input.  
  - Inputs: Invoked by `Research AI Agent`.  
  - Outputs: Search results JSON.  
  - Edge Cases: API key issues, quota limits, or empty results.

- **SerpApi**  
  - Type: LangChain HTTP Request Tool with predefined credentials  
  - Role: Searches Google Scholar for academic papers.  
  - Configuration: Uses SerpApi with API key from credentials, query encoded from refined query.  
  - Inputs: Invoked by `Research AI Agent`.  
  - Outputs: Academic paper search results.  
  - Edge Cases: Credential errors, API limits, or no results.

- **Search News**  
  - Type: LangChain HTTP Request Tool  
  - Role: Fetches recent news articles related to the topic using NewsAPI.  
  - Configuration: URL with query parameter and API key placeholder `"YOURAPIKEY"`.  
  - Inputs: Invoked by `Research AI Agent`.  
  - Outputs: News articles JSON.  
  - Edge Cases: API key issues, rate limits, or no recent news.

---

#### 2.4 Data Parsing and Merging

**Overview:**  
Parses the raw JSON output from the AI agent and merges data from multiple sources into a single structured JSON object for report generation.

**Nodes Involved:**  
- `Parse Research Output` (Code)  
- `Split Out` (Split Out)  
- `Merge Split Items` (Code)  
- `Aggregate` (Aggregate)

**Node Details:**

- **Parse Research Output**  
  - Type: Code Node  
  - Role: Parses the raw JSON string output from `Research AI Agent` into a JSON object.  
  - Configuration: Uses `JSON.parse()` on the `output` string field.  
  - Inputs: From `Research AI Agent`.  
  - Outputs: Parsed JSON object with research data fields.  
  - Edge Cases: JSON parse errors if AI output is malformed.

- **Split Out**  
  - Type: Split Out Node  
  - Role: Splits the parsed JSON into separate items for each research section (introduction, summary, key findings, etc.).  
  - Configuration: Splits fields: `introduction, summary, key_findings, news_highlights, scholarly_insights, wikipedia_summary, sources`.  
  - Inputs: From `Parse Research Output`.  
  - Outputs: Multiple items, each containing one field.  
  - Edge Cases: Missing fields result in empty splits.

- **Merge Split Items**  
  - Type: Code Node  
  - Role: Merges all split items back into a single JSON object, concatenating arrays and preserving scalar fields.  
  - Configuration: Concatenates arrays like `key_findings`, `news_highlights`, `scholarly_insights`, and `sources`.  
  - Inputs: From `Split Out`.  
  - Outputs: Single merged JSON object with all research data combined.  
  - Edge Cases: Non-array fields handled gracefully; missing data results in empty arrays.

- **Aggregate**  
  - Type: Aggregate Node  
  - Role: Aggregates all item data into one item for downstream processing.  
  - Configuration: Uses `aggregateAllItemData` option.  
  - Inputs: From `Merge Split Items`.  
  - Outputs: Single aggregated JSON object.  
  - Edge Cases: None significant.

---

#### 2.5 Report Generation (HTML and PDF)

**Overview:**  
Generates a professional HTML report from the aggregated data and converts it into a PDF using the PDFShift API.

**Nodes Involved:**  
- `Generate PDF HTML` (Code)  
- `Convert HTML to PDF` (HTTP Request)

**Node Details:**

- **Generate PDF HTML**  
  - Type: Code Node  
  - Role: Creates a styled HTML document for the research report, including cover page, sections, and sources.  
  - Configuration:  
    - Escapes HTML special characters to prevent injection/errors.  
    - Formats topic with capitalization and "AI" uppercase.  
    - Uses research data fields (introduction, summary, key findings, etc.) from merged data.  
    - Defines file name based on topic and current date.  
    - Includes CSS styling for professional appearance (fonts, colors, layout).  
  - Inputs: From `Aggregate` (research data) and `Query Refiner` (topic).  
  - Outputs: JSON with `htmlContent`, `fileName`, `topic`, `rawTimestamp`, and `formattedDate`.  
  - Edge Cases: Missing data fields default to placeholders; invalid timestamps fallback to current date.

- **Convert HTML to PDF**  
  - Type: HTTP Request Node  
  - Role: Sends the HTML content to PDFShift API to convert it into a PDF document.  
  - Configuration:  
    - POST request to `https://api.pdfshift.io/v3/convert/pdf`.  
    - Sends HTML content as `source` parameter.  
    - Includes filename and options for PDF generation.  
    - Uses Basic Auth header with encoded API key.  
  - Inputs: From `Generate PDF HTML`.  
  - Outputs: JSON response containing URL to the generated PDF.  
  - Edge Cases: API key invalid, network errors, or API limits; returns URL, not binary data.

---

#### 2.6 PDF Download and Delivery

**Overview:**  
Downloads the generated PDF from the URL and sends it as an attachment via Gmail and Telegram.

**Nodes Involved:**  
- `Download PDF` (HTTP Request)  
- `Send Research to Gmail` (Gmail)  
- `Send PDF` (Telegram)  
- `Search Folder` (Google Drive)  
- `If` (Conditional)

**Node Details:**

- **Download PDF**  
  - Type: HTTP Request Node  
  - Role: Downloads the PDF file from the URL provided by PDFShift.  
  - Configuration: Uses URL from previous node’s JSON (`$json.url`).  
  - Outputs: Binary PDF data with MIME type `application/pdf`.  
  - Edge Cases: Download failures, invalid URL.

- **Send Research to Gmail**  
  - Type: Gmail Node  
  - Role: Sends the PDF as an email attachment with a styled HTML message.  
  - Configuration:  
    - Recipient email hardcoded as `emaikuri@gmail.com` (should be parameterized).  
    - Subject includes capitalized topic.  
    - HTML body includes greeting, report summary, and signature.  
    - Attaches PDF binary from `Download PDF`.  
    - Uses Gmail OAuth2 credentials.  
  - Inputs: PDF binary from `Download PDF`.  
  - Outputs: Email sent confirmation.  
  - Edge Cases: Credential issues, email delivery failures.

- **Send PDF**  
  - Type: Telegram Node  
  - Role: Sends the PDF as a document to a Telegram chat.  
  - Configuration:  
    - Chat ID hardcoded as `1274041539` (should be parameterized).  
    - Sends binary PDF data.  
    - Uses Telegram Bot credentials.  
  - Inputs: PDF binary from `Download PDF`.  
  - Outputs: Message sent confirmation.  
  - Edge Cases: Bot token invalid, chat ID incorrect, Telegram API errors.

- **Search Folder**  
  - Type: Google Drive Node  
  - Role: Searches for a folder named "Research Reports" in Google Drive (not connected to main flow).  
  - Configuration: Query string to find folder by name.  
  - Inputs: None connected in main flow.  
  - Outputs: Folder metadata.  
  - Edge Cases: Folder not found.

- **If**  
  - Type: Conditional Node  
  - Role: Checks if Google Drive search returned any folders (not connected in main flow).  
  - Configuration: Condition checks if length of folder list > 0.  
  - Inputs: From `Search Folder`.  
  - Outputs: Branches based on folder existence.  
  - Edge Cases: None.

---

#### 2.7 Metadata Storage

**Overview:**  
Stores metadata about the research report (topic, sources, timestamp, search queries) in a Google Sheets spreadsheet for record-keeping.

**Nodes Involved:**  
- `Store Research Metadata` (Google Sheets)

**Node Details:**

- **Store Research Metadata**  
  - Type: Google Sheets Node  
  - Role: Appends a new row with metadata about the research report.  
  - Configuration:  
    - Spreadsheet ID and sheet name specified.  
    - Columns: Topic, Sources, Timestamp, Search Queries.  
    - Uses OAuth2 credentials for Google Sheets.  
  - Inputs: From `Aggregate` node (aggregated research data).  
  - Outputs: Confirmation of row append.  
  - Edge Cases: Credential issues, API limits, or spreadsheet access errors.

---

### 3. Summary Table

| Node Name               | Node Type                               | Functional Role                          | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                      |
|-------------------------|---------------------------------------|----------------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                       | Entry point for manual testing          | None                          | Input Validation              |                                                                                                |
| Input Validation        | Code                                  | Validates and cleans input query        | When clicking ‘Test workflow’ | Query Refiner                 |                                                                                                |
| OpenAI Chat Model1      | LangChain OpenAI Chat Model            | Provides language model for Query Refiner | None                          | Query Refiner (via agent)     |                                                                                                |
| Structured Output Parser| LangChain Output Parser Structured     | Parses JSON output from Query Refiner   | Query Refiner                 | Query Refiner                 |                                                                                                |
| Query Refiner           | LangChain Agent                       | Refines query and generates related queries | Input Validation, OpenAI Chat Model1, Structured Output Parser | Research AI Agent             |                                                                                                |
| Simple Memory           | LangChain Memory Buffer Window         | Maintains session context for AI agent  | Query Refiner                 | Research AI Agent             |                                                                                                |
| Research AI Agent       | LangChain Agent                       | Coordinates research using multiple tools | Query Refiner, Simple Memory  | Parse Research Output         |                                                                                                |
| OpenAI Chat Model       | LangChain OpenAI Chat Model            | Provides language model for Research AI Agent | None                          | Research AI Agent (via agent) |                                                                                                |
| Wikipedia               | LangChain HTTP Request Tool            | Fetches Wikipedia data                   | Research AI Agent             | Research AI Agent             |                                                                                                |
| Google Search Web       | LangChain HTTP Request Tool            | Performs Google Custom Search            | Research AI Agent             | Research AI Agent             |                                                                                                |
| SerpApi                 | LangChain HTTP Request Tool            | Searches Google Scholar via SerpApi     | Research AI Agent             | Research AI Agent             |                                                                                                |
| Search News             | LangChain HTTP Request Tool            | Fetches recent news articles             | Research AI Agent             | Research AI Agent             |                                                                                                |
| Parse Research Output   | Code                                  | Parses raw JSON string from AI agent    | Research AI Agent             | Split Out                    |                                                                                                |
| Split Out               | Split Out                             | Splits research data into sections      | Parse Research Output         | Merge Split Items            |                                                                                                |
| Merge Split Items       | Code                                  | Merges split data into single JSON      | Split Out                    | Aggregate                    |                                                                                                |
| Aggregate               | Aggregate                            | Aggregates all item data into one item  | Merge Split Items            | Store Research Metadata, Generate PDF HTML |                                                                                                |
| Store Research Metadata | Google Sheets                        | Stores research metadata in spreadsheet | Aggregate                    | None                        |                                                                                                |
| Generate PDF HTML       | Code                                  | Generates styled HTML report             | Aggregate                    | Convert HTML to PDF          |                                                                                                |
| Convert HTML to PDF     | HTTP Request                         | Converts HTML to PDF via PDFShift API   | Generate PDF HTML            | Download PDF                 |                                                                                                |
| Download PDF            | HTTP Request                         | Downloads PDF binary from PDFShift URL  | Convert HTML to PDF          | Send Research to Gmail, Send PDF |                                                                                                |
| Send Research to Gmail  | Gmail                                | Sends PDF report via email               | Download PDF                 | None                        |                                                                                                |
| Send PDF                | Telegram                             | Sends PDF report via Telegram            | Download PDF                 | None                        |                                                                                                |
| Search Folder           | Google Drive                        | Searches for "Research Reports" folder  | None                        | If                          |                                                                                                |
| If                      | If                                   | Checks if folder exists                   | Search Folder                | None                        |                                                                                                |
| Sticky Note             | Sticky Note                         | Documentation and workflow explanation   | None                        | None                        | # 📋 Research Report Workflow 🧠💻 ... (full content as provided)                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - Purpose: Entry point for manual testing.

2. **Create Code Node for Input Validation**  
   - Name: `Input Validation`  
   - Type: Code  
   - JavaScript:  
     ```js
     const query = $input.all()[0].json.query;
     if (!query || query.trim().length < 3) {
       throw new Error('Research query must be at least 3 characters long');
     }
     return {
       json: {
         originalQuery: query,
         cleanedQuery: query.trim().toLowerCase(),
         timestamp: new Date().toISOString()
       }
     };
     ```  
   - Connect `When clicking ‘Test workflow’` → `Input Validation`.

3. **Create OpenAI Chat Model Node for Query Refinement**  
   - Name: `OpenAI Chat Model1`  
   - Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4o-mini` (or preferred GPT model)  
   - Credentials: OpenAI API key stored in n8n credentials.

4. **Create Structured Output Parser Node**  
   - Name: `Structured Output Parser`  
   - Type: LangChain Output Parser Structured  
   - JSON Schema Example:  
     ```json
     {
       "output": {
         "topic": "the best ai models 2025",
         "searchQueries": [
           "best AI models 2025 natural language processing",
           "top AI models 2025 computer vision",
           "best AI models 2025 generative AI",
           "recent advancements in AI models 2025 news",
           "scholarly research on AI models 2020-2025",
           "ethical concerns in AI models 2025",
           "AI models 2025 applications in healthcare",
           "AI models 2025 trends in automation"
         ]
       }
     }
     ```
   - Connect `Input Validation` → `Query Refiner`.

5. **Create LangChain Agent Node for Query Refiner**  
   - Name: `Query Refiner`  
   - Type: LangChain Agent  
   - Prompt: Custom prompt instructing generation of 5 related queries based on cleaned query.  
   - Use `OpenAI Chat Model1` as language model.  
   - Use `Structured Output Parser` as output parser.  
   - Connect `Input Validation` → `Query Refiner`.

6. **Create Simple Memory Node**  
   - Name: `Simple Memory`  
   - Type: LangChain Memory Buffer Window  
   - Session Key: `={{ $json.output.searchQueries }}`  
   - Connect `Query Refiner` → `Simple Memory`.

7. **Create OpenAI Chat Model Node for Research AI Agent**  
   - Name: `OpenAI Chat Model`  
   - Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4o-mini`  
   - Credentials: OpenAI API key.

8. **Create Research AI Agent Node**  
   - Name: `Research AI Agent`  
   - Type: LangChain Agent  
   - Prompt: Detailed system message instructing research steps and expected JSON output structure.  
   - Inputs: Refined topic and search queries from `Query Refiner`.  
   - Use `OpenAI Chat Model` as language model.  
   - Use `Simple Memory` as memory.  
   - Connect `Query Refiner` → `Research AI Agent`.  
   - Connect `Simple Memory` → `Research AI Agent` (memory input).

9. **Create HTTP Request Nodes for Data Sources**  
   - `Wikipedia`: Wikipedia API request with query parameter from refined query.  
   - `Google Search Web`: Google Custom Search API request with query parameter.  
   - `SerpApi`: Google Scholar search via SerpApi with API key credential.  
   - `Search News`: NewsAPI request with query parameter and API key.  
   - Connect all these nodes as tools to `Research AI Agent`.

10. **Create Code Node to Parse Research Output**  
    - Name: `Parse Research Output`  
    - JavaScript:  
      ```js
      const outputString = $input.first().json.output;
      const parsedOutput = JSON.parse(outputString);
      return [{ json: parsedOutput }];
      ```  
    - Connect `Research AI Agent` → `Parse Research Output`.

11. **Create Split Out Node**  
    - Name: `Split Out`  
    - Type: Split Out  
    - Field to split: `introduction, summary, key_findings, news_highlights, scholarly_insights, wikipedia_summary, sources`  
    - Connect `Parse Research Output` → `Split Out`.

12. **Create Code Node to Merge Split Items**  
    - Name: `Merge Split Items`  
    - JavaScript:  
      ```js
      const mergedItem = {
        key_findings: [],
        news_highlights: [],
        scholarly_insights: [],
        sources: []
      };
      $input.all().forEach(item => {
        const data = item.json;
        if (data.introduction) mergedItem.introduction = data.introduction;
        if (data.summary) mergedItem.summary = data.summary;
        if (data.wikipedia_summary) mergedItem.wikipedia_summary = data.wikipedia_summary;
        if (data.key_findings) mergedItem.key_findings = mergedItem.key_findings.concat(Array.isArray(data.key_findings) ? data.key_findings : [data.key_findings]);
        if (data.news_highlights) mergedItem.news_highlights = mergedItem.news_highlights.concat(Array.isArray(data.news_highlights) ? data.news_highlights : [data.news_highlights]);
        if (data.scholarly_insights) mergedItem.scholarly_insights = mergedItem.scholarly_insights.concat(Array.isArray(data.scholarly_insights) ? data.scholarly_insights : [data.scholarly_insights]);
        if (data.sources) mergedItem.sources = mergedItem.sources.concat(Array.isArray(data.sources) ? data.sources : [data.sources]);
      });
      return [{ json: mergedItem }];
      ```  
    - Connect `Split Out` → `Merge Split Items`.

13. **Create Aggregate Node**  
    - Name: `Aggregate`  
    - Operation: Aggregate all item data into one item.  
    - Connect `Merge Split Items` → `Aggregate`.

14. **Create Google Sheets Node to Store Metadata**  
    - Name: `Store Research Metadata`  
    - Operation: Append row  
    - Spreadsheet ID and Sheet Name configured.  
    - Columns mapped: Topic, Sources, Timestamp, Search Queries from aggregated JSON.  
    - Credentials: Google Sheets OAuth2.  
    - Connect `Aggregate` → `Store Research Metadata`.

15. **Create Code Node to Generate PDF HTML**  
    - Name: `Generate PDF HTML`  
    - JavaScript: Use provided code to generate styled HTML report with research data and formatted date.  
    - Connect `Aggregate` → `Generate PDF HTML`.

16. **Create HTTP Request Node to Convert HTML to PDF**  
    - Name: `Convert HTML to PDF`  
    - Method: POST  
    - URL: `https://api.pdfshift.io/v3/convert/pdf`  
    - Body Parameters: `source` (HTML content), `filename`, and options.  
    - Headers: Authorization with Basic Auth and PDFShift API key.  
    - Connect `Generate PDF HTML` → `Convert HTML to PDF`.

17. **Create HTTP Request Node to Download PDF**  
    - Name: `Download PDF`  
    - Method: GET  
    - URL: From `Convert HTML to PDF` response JSON field `url`.  
    - Connect `Convert HTML to PDF` → `Download PDF`.

18. **Create Gmail Node to Send PDF**  
    - Name: `Send Research to Gmail`  
    - Operation: Send email with attachment  
    - Recipient: Configured email address (e.g., `emaikuri@gmail.com`)  
    - Subject and HTML body use expressions to include topic and date.  
    - Attach PDF binary from `Download PDF`.  
    - Credentials: Gmail OAuth2.  
    - Connect `Download PDF` → `Send Research to Gmail`.

19. **Create Telegram Node to Send PDF**  
    - Name: `Send PDF`  
    - Operation: Send Document  
    - Chat ID: Configured Telegram chat ID  
    - Attach PDF binary from `Download PDF`.  
    - Credentials: Telegram Bot token.  
    - Connect `Download PDF` → `Send PDF`.

20. **Optional: Create Google Drive Search and Conditional Nodes**  
    - `Search Folder`: Search for "Research Reports" folder.  
    - `If`: Check if folder exists.  
    - These nodes are not connected in main flow but can be integrated for file storage.

---

This completes the detailed reconstruction of the workflow, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.