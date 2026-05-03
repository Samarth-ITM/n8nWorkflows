Extract & Summarize B2B Leads from Crunchbase with Bright Data, GPT-4o & Google Sheets

https://n8nworkflows.xyz/workflows/extract---summarize-b2b-leads-from-crunchbase-with-bright-data--gpt-4o---google-sheets-4250


# Extract & Summarize B2B Leads from Crunchbase with Bright Data, GPT-4o & Google Sheets

---
### 1. Workflow Overview

This n8n workflow, titled **"Extract & Summarize B2B Leads from Crunchbase with Bright Data, GPT-4o & Google Sheets"**, is designed to automate the extraction, processing, summarization, and storage of B2B lead data from Crunchbase. It leverages Bright Data’s web unlocking proxy service to reliably scrape Crunchbase data, uses OpenAI's GPT-4o model for advanced language processing, and stores structured and summarized output in Google Sheets and local disk files. Additionally, it sends webhook notifications to external services.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception and Setup**: Manual trigger and setting of Crunchbase URL and Bright Data zone parameters.
- **1.2 Data Acquisition via Bright Data**: Making authenticated POST requests to Bright Data API to scrape Crunchbase pages.
- **1.3 Markdown to Text Conversion**: Using GPT-4o to convert the raw markdown content from Bright Data into clean textual data.
- **1.4 Textual Data Summarization**: Summarizing the extracted textual data into concise insights with GPT-4o.
- **1.5 Structured Data Extraction**: Extracting detailed structured data from the textual content using GPT-4o and parsing it via a JSON schema.
- **1.6 Data Storage and Notification**: Writing both summarized and structured data to disk and Google Sheets, and sending webhook notifications with extracted data.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Setup

- **Overview:**  
  This block initiates the workflow manually and configures the Crunchbase target URL and Bright Data proxy zone to be used for web scraping.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set URL and Bright Data Zone (Set Node)  
  - Sticky Note (Instructional)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type & Role:* Manual Trigger; entry point to start workflow execution manually.  
    - *Configuration:* No parameters; triggers workflow on user action.  
    - *Connections:* Output → Set URL and Bright Data Zone.  
    - *Failures:* None expected; user must manually trigger.

  - **Set URL and Bright Data Zone**  
    - *Type & Role:* Set node; assigns static variables `url` and `zone`.  
    - *Config:*  
      - `url`: Crunchbase organization URL (default: `https://www.crunchbase.com/organization/stripe`).  
      - `zone`: Bright Data proxy zone name (default: `web_unlocker1`).  
    - *Connections:* Input from Manual Trigger; output → Perform Bright Data Web Request.  
    - *Failures:* Incorrect URL or zone name will cause downstream failures.

  - **Sticky Note (block comment)**  
    - *Content:* Instructions to update Crunchbase URL, Bright Data zone, Google Sheets credentials, and webhook URL.  
    - *Purpose:* User guidance to prevent misconfiguration.

#### 1.2 Data Acquisition via Bright Data

- **Overview:**  
  Executes a POST request to the Bright Data API to retrieve raw markdown data from the specified Crunchbase URL using the configured proxy zone.

- **Nodes Involved:**  
  - Perform Bright Data Web Request (HTTP Request)  
  - Sticky Note1 (LLM Usage Explanation)

- **Node Details:**

  - **Perform Bright Data Web Request**  
    - *Type & Role:* HTTP Request node; sends POST request to Bright Data API.  
    - *Config:*  
      - URL: `https://api.brightdata.com/request`  
      - Method: POST  
      - Body Parameters: `zone` and `url` taken from previous Set node, with `format=raw` and `data_format=markdown` to get markdown content.  
      - Authentication: HTTP Header Auth via stored credentials (Bright Data header token).  
    - *Connections:* Input from Set URL and Bright Data Zone; output → Markdown to Textual Data Extractor.  
    - *Failures:*  
      - Authentication errors if credentials invalid.  
      - Timeout or rate limiting from Bright Data API.  
      - Incorrect zone or URL causing empty or error responses.

  - **Sticky Note1**  
    - *Content:* Explanation of LLM usage - GPT-4o is used for markdown conversion and structured extraction.  
    - *Purpose:* Inform users about AI model usage and workflow logic.

#### 1.3 Markdown to Text Conversion

- **Overview:**  
  Converts raw markdown data obtained from Bright Data into clean, link-free textual data using GPT-4o.

- **Nodes Involved:**  
  - Markdown to Textual Data Extractor (LangChain LLM Chain)  
  - OpenAI Chat Model (GPT-4o-mini)

- **Node Details:**

  - **Markdown to Textual Data Extractor**  
    - *Type & Role:* LangChain Chain LLM node; processes markdown input with a custom prompt to produce plain text.  
    - *Config:*  
      - Prompt instructs the model to analyze markdown and output textual data only, excluding links, scripts, CSS, etc.  
      - Message context: "You are a markdown expert".  
    - *Connections:* Input from Bright Data HTTP Request; output → Summarize the content and Structured Data Extractor nodes.  
    - *Failures:*  
      - LLM API errors (rate limit, auth).  
      - Prompt misinterpretation causing incomplete text extraction.

  - **OpenAI Chat Model**  
    - *Type & Role:* GPT-4o-mini model node; underlying AI model for markdown conversion.  
    - *Config:* Model set to `gpt-4o-mini`, no special options.  
    - *Connections:* Input from Markdown to Textual Data Extractor; outputs to Summarization and Structured Data Extraction nodes.  
    - *Failures:* API quota or connectivity issues.

#### 1.4 Textual Data Summarization

- **Overview:**  
  Generates a concise summary of the cleaned textual content extracted from Crunchbase data.

- **Nodes Involved:**  
  - Summarize the content (LangChain Chain Summarization)  
  - OpenAI Chat Model1 (GPT-4o-mini)  
  - Create a binary data for Summarization (Function)  
  - Write the summarized content to disk (Read/Write File)  
  - Sticky Note3 (Summarization block comment)

- **Node Details:**

  - **Summarize the content**  
    - *Type & Role:* LangChain summarization chain; creates summaries from input text.  
    - *Config:* Uses advanced chunking mode to handle large input text.  
    - *Connections:* Input from Markdown to Textual Data Extractor; output → Create binary data for Summarization.  
    - *Failures:* LLM API errors; chunking errors if input too large or malformed.

  - **OpenAI Chat Model1**  
    - *Type & Role:* GPT-4o-mini for summarization tasks.  
    - *Config:* Same as previous models.  
    - *Connections:* Underlying model for Summarize the content node.  
    - *Failures:* Same as other LLM nodes.

  - **Create a binary data for Summarization**  
    - *Type & Role:* Function node; converts JSON summary output into base64-encoded binary format for file writing.  
    - *Config:* Uses Buffer to encode JSON string.  
    - *Connections:* Input from Summarize the content; output → Write the summarized content to disk.  
    - *Failures:* Code errors if input JSON is malformed.

  - **Write the summarized content to disk**  
    - *Type & Role:* Read/Write File node; writes the summarized JSON content to local disk.  
    - *Config:*  
      - Operation: Write  
      - Filename: `d:\Crunchbase-Summary.json` (Windows path)  
    - *Connections:* Input from Create binary data for Summarization.  
    - *Failures:*  
      - File system permission errors.  
      - Invalid path or disk full.

  - **Sticky Note3**  
    - *Content:* Title “Summarization” to visually group this block.

#### 1.5 Structured Data Extraction

- **Overview:**  
  Extracts detailed structured data from the textual content using GPT-4o and parses it into a defined JSON schema for downstream use.

- **Nodes Involved:**  
  - Structured Data Extractor (LangChain Chain LLM)  
  - Structured Output Parser (LangChain Output Parser Structured)  
  - OpenAI Chat Model2 (GPT-4o-mini)  
  - Create a binary data for Structured Data Extract (Function)  
  - Write the structured content to disk (Read/Write File)  
  - Initiate a Webhook Notification for the extracted data (HTTP Request)  
  - Google Sheets (Google Sheets node)  
  - Sticky Note2 (Structured Data Extract block comment)

- **Node Details:**

  - **Structured Data Extractor**  
    - *Type & Role:* LangChain Chain LLM node; extracts structured information based on prompt and uses output parser.  
    - *Config:*  
      - Prompt instructs extraction of structured info from provided text content.  
      - Has Output Parser enabled with a JSON schema example defining multiple company fields including name, funding, contacts, products, news, etc.  
    - *Connections:* Input from Markdown to Textual Data Extractor; output → Google Sheets, Create binary data for Structured Data Extract, and Webhook Notification.  
    - *Failures:*  
      - LLM parsing errors or mismatches to JSON schema.  
      - Invalid or incomplete input data.

  - **Structured Output Parser**  
    - *Type & Role:* LangChain output parser; validates and parses LLM output into structured JSON according to schema.  
    - *Config:* Uses the provided JSON schema example.  
    - *Connections:* Input from Structured Data Extractor; output back to Structured Data Extractor node.  
    - *Failures:* Parsing failures if LLM output doesn't conform to schema.

  - **OpenAI Chat Model2**  
    - *Type & Role:* GPT-4o-mini model for structured extraction tasks.  
    - *Config:* Standard GPT-4o-mini.  
    - *Connections:* Used internally by Structured Data Extractor.  
    - *Failures:* API errors or quota limits.

  - **Create a binary data for Structured Data Extract**  
    - *Type & Role:* Function node; encodes extracted structured JSON data to base64 for file writing.  
    - *Config:* Similar to summarization binary data creator.  
    - *Connections:* Input from Structured Data Extractor; output → Write the structured content to disk.  
    - *Failures:* JSON encoding errors.

  - **Write the structured content to disk**  
    - *Type & Role:* Read/Write File node; writes structured data JSON to local disk.  
    - *Config:*  
      - Operation: Write  
      - Filename: `d:\Crunchbase-Summary.json` (same as summary, which may cause overwrite)  
    - *Connections:* Input from Create binary data for Structured Data Extract.  
    - *Failures:* File system errors; possible data overwrite issues due to same filename.

  - **Initiate a Webhook Notification for the extracted data**  
    - *Type & Role:* HTTP Request node; sends extracted structured data as JSON payload to an externally configurable webhook URL.  
    - *Config:*  
      - Method: POST (default)  
      - URL: `https://webhook.site/7b5380a0-0544-48dc-be43-0116cb2d52c2` (placeholder)  
      - Body Parameters: `summary` field populated with structured output.  
    - *Connections:* Input from Structured Data Extractor.  
    - *Failures:* Network errors; invalid webhook URL; data format mismatches.

  - **Google Sheets**  
    - *Type & Role:* Google Sheets node; appends or updates extracted structured data into a specified Google Sheet.  
    - *Config:*  
      - Document ID: specific Google Sheet ID  
      - Sheet name: `gid=0` (Sheet1)  
      - Operation: AppendOrUpdate  
      - Columns: Automatically mapped with schema expecting `Structured Data` column.  
      - Credentials: Google OAuth2 stored credentials.  
    - *Connections:* Input from Structured Data Extractor.  
    - *Failures:* Auth errors, API quota, sheet permission issues, schema mismatches.

  - **Sticky Note2**  
    - *Content:* Title “Structured Data Extract” to visually group this block.

#### 1.6 Workflow Metadata and Branding

- **Overview:**  
  Provides branding and additional workflow comments for user orientation.

- **Nodes Involved:**  
  - Sticky Note4 (Logo and branding)

- **Node Details:**

  - **Sticky Note4**  
    - *Content:* Contains Bright Data logo image link for branding.  
    - *Purpose:* Visual identification of Bright Data usage.

---

### 3. Summary Table

| Node Name                             | Node Type                             | Functional Role                                | Input Node(s)                        | Output Node(s)                                      | Sticky Note                                                                                 |
|-------------------------------------|-------------------------------------|-----------------------------------------------|------------------------------------|-----------------------------------------------------|---------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’        | Manual Trigger                      | Entry point to start workflow                   |                                    | Set URL and Bright Data Zone                         | Please set Crunchbase URL, Bright Data zone, webhook URL, and Google Sheets credentials.   |
| Set URL and Bright Data Zone         | Set                                 | Sets Crunchbase URL and Bright Data proxy zone | When clicking ‘Test workflow’       | Perform Bright Data Web Request                      |                                                                                             |
| Perform Bright Data Web Request       | HTTP Request                       | Retrieves raw markdown data from Crunchbase via Bright Data proxy | Set URL and Bright Data Zone        | Markdown to Textual Data Extractor                   |                                                                                             |
| Markdown to Textual Data Extractor    | LangChain Chain LLM                | Converts markdown to clean textual content      | Perform Bright Data Web Request     | Summarize the content, Structured Data Extractor    | GPT-4o model used for markdown conversion and structured extraction.                        |
| OpenAI Chat Model                    | LangChain LLM OpenAI GPT-4o-mini    | Underlying GPT-4o model for markdown processing | Markdown to Textual Data Extractor | Summarize the content, Structured Data Extractor    | GPT-4o-mini LLM for text conversion.                                                       |
| Summarize the content               | LangChain Chain Summarization      | Summarizes textual content                       | Markdown to Textual Data Extractor | Create a binary data for Summarization               | Summarization chain with advanced chunking.                                                |
| OpenAI Chat Model1                  | LangChain LLM OpenAI GPT-4o-mini    | GPT-4o model for summarization                   | Summarize the content              | Create a binary data for Summarization               | GPT-4o-mini LLM for summarization.                                                        |
| Create a binary data for Summarization | Function                          | Encodes summary JSON to base64 binary            | Summarize the content              | Write the summarized content to disk                 |                                                                                             |
| Write the summarized content to disk | Read/Write File                   | Writes summary JSON to disk                       | Create a binary data for Summarization |                                     |                                                                                             |
| Structured Data Extractor            | LangChain Chain LLM                | Extracts detailed structured data from text     | Markdown to Textual Data Extractor | Google Sheets, Create binary data for Structured Data Extract, Initiate Webhook Notification | GPT-4o-mini used with structured output parser.                                          |
| Structured Output Parser             | LangChain Output Parser Structured | Parses LLM output into JSON schema                | Structured Data Extractor          | Structured Data Extractor                             | JSON schema validation.                                                                    |
| OpenAI Chat Model2                  | LangChain LLM OpenAI GPT-4o-mini    | GPT-4o model for structured data extraction      | Structured Data Extractor          | Structured Output Parser                              | GPT-4o-mini LLM for structured extraction.                                                |
| Create a binary data for Structured Data Extract | Function               | Encodes structured JSON to base64 binary          | Structured Data Extractor          | Write the structured content to disk                 |                                                                                             |
| Write the structured content to disk | Read/Write File                   | Writes structured JSON to disk                     | Create a binary data for Structured Data Extract |                                     |                                                                                             |
| Initiate a Webhook Notification for the extracted data | HTTP Request          | Sends extracted structured data to external webhook | Structured Data Extractor          |                                                     | Webhook URL must be updated by user.                                                      |
| Google Sheets                      | Google Sheets                      | Stores structured data in Google Sheets           | Structured Data Extractor          |                                                     | Requires user OAuth2 credentials.                                                           |
| Sticky Note                        | Sticky Note                       | User instructions and notes                        |                                    |                                                     | Please set Crunchbase URL, Bright Data zone, webhook URL, and Google Sheets credentials.   |
| Sticky Note1                       | Sticky Note                       | Explains AI model usage                            |                                    |                                                     | GPT-4o model used for markdown conversion and structured extraction.                        |
| Sticky Note2                       | Sticky Note                       | Visual label for Structured Data Extract block    |                                    |                                                     | Structured Data Extract block.                                                             |
| Sticky Note3                       | Sticky Note                       | Visual label for Summarization block               |                                    |                                                     | Summarization block.                                                                        |
| Sticky Note4                       | Sticky Note                       | Branding with Bright Data logo                      |                                    |                                                     | Contains Bright Data logo.                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To manually start the workflow.

2. **Add a Set Node:**  
   - Name: `Set URL and Bright Data Zone`  
   - Parameters:  
     - `url`: Set to a Crunchbase organization URL (e.g., `https://www.crunchbase.com/organization/stripe`)  
     - `zone`: Set to your Bright Data proxy zone name (e.g., `web_unlocker1`)  
   - Connect output of Manual Trigger to this node.

3. **Add HTTP Request Node for Bright Data:**  
   - Name: `Perform Bright Data Web Request`  
   - URL: `https://api.brightdata.com/request`  
   - Method: POST  
   - Body Parameters:  
     - `zone`: from previous Set node (`={{ $json.zone }}`)  
     - `url`: from previous Set node (`={{ $json.url }}`)  
     - `format`: `raw`  
     - `data_format`: `markdown`  
   - Authentication: HTTP Header Auth with Bright Data API token credentials.  
   - Connect output of Set node to this node.

4. **Add LangChain Chain LLM Node for Markdown to Text Conversion:**  
   - Name: `Markdown to Textual Data Extractor`  
   - Prompt: "You need to analyze the below markdown and convert to textual data. Please do not output with your own thoughts. Make sure to output with textual data only with no links, scripts, css etc.\n\n{{ $json.data }}"  
   - Message: "You are a markdown expert"  
   - Model: Use an OpenAI GPT-4o-mini model node connected internally.  
   - Connect output of HTTP Request node to this node.

5. **Add LangChain Chain Summarization Node:**  
   - Name: `Summarize the content`  
   - Chunking mode: `advanced`  
   - Connect output of Markdown to Textual Data Extractor to this node.

6. **Add OpenAI GPT-4o-mini Chat Model Node for Summarization:**  
   - Name: `OpenAI Chat Model1`  
   - Model: `gpt-4o-mini`  
   - Credentials: OpenAI API key  
   - Connect as AI Language Model node internally to the Summarize node.

7. **Add Function Node to Encode Summary as Binary:**  
   - Name: `Create a binary data for Summarization`  
   - Function code: Encode JSON summary output to base64 binary (see Node 1.4).  
   - Connect output of Summarize node to this node.

8. **Add Read/Write File Node to Write Summary:**  
   - Name: `Write the summarized content to disk`  
   - Operation: Write  
   - Filename: `d:\Crunchbase-Summary.json` (or appropriate path)  
   - Connect output of Function node to this node.

9. **Add LangChain Chain LLM Node for Structured Data Extraction:**  
   - Name: `Structured Data Extractor`  
   - Prompt: "Extract the structured info from the below content.\n\nHere's the Content:  {{ $json.text }}"  
   - Enable Output Parser with JSON schema defining organization data fields (name, funding, website, contacts, etc.).  
   - Connect output of Markdown to Textual Data Extractor to this node.

10. **Add LangChain Output Parser Structured Node:**  
    - Name: `Structured Output Parser`  
    - JSON Schema: Use the detailed company schema provided.  
    - Connect output parser port back to Structured Data Extractor.

11. **Add OpenAI GPT-4o-mini Chat Model Node for Structured Extraction:**  
    - Name: `OpenAI Chat Model2`  
    - Model: `gpt-4o-mini`  
    - Credentials: OpenAI API key  
    - Connect internally to Structured Data Extractor node.

12. **Add Function Node to Encode Structured Data as Binary:**  
    - Name: `Create a binary data for Structured Data Extract`  
    - Use same base64 encoding approach as summarization function.  
    - Connect output of Structured Data Extractor to this node.

13. **Add Read/Write File Node to Write Structured Data:**  
    - Name: `Write the structured content to disk`  
    - Operation: Write  
    - Filename: Same or different path as summary (recommend different to avoid overwrite).  
    - Connect output of Function node to this node.

14. **Add HTTP Request Node for Webhook Notification:**  
    - Name: `Initiate a Webhook Notification for the extracted data`  
    - URL: User-configurable webhook URL (default placeholder `https://webhook.site/...`)  
    - Method: POST  
    - Body Parameter: `summary` with extracted structured JSON data.  
    - Connect output of Structured Data Extractor to this node.

15. **Add Google Sheets Node:**  
    - Name: `Google Sheets`  
    - Operation: AppendOrUpdate  
    - Document ID: Set to your Google Sheet ID  
    - Sheet Name: `gid=0`  
    - Columns: Auto map input data with `Structured Data` field  
    - Credentials: Google Sheets OAuth2 credentials  
    - Connect output of Structured Data Extractor to this node.

16. **Add Sticky Note Nodes for Documentation and Branding:**  
    - Add notes for user instructions on URLs, credentials, and AI model usage.  
    - Add branding image for Bright Data logo.

17. **Connect nodes following the described flow:**  
    - Manual Trigger → Set URL and Bright Data Zone → Bright Data HTTP Request → Markdown to Textual Data Extractor → both Summarize the content and Structured Data Extractor → respective downstream nodes as per above.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Deals with Crunchbase B2B Lead Discovery using Bright Data Web Unlocker. Update URLs and credentials accordingly. | Workflow instructions sticky note.                                                              |
| OpenAI GPT-4o model (gpt-4o-mini variant) is used for markdown conversion, summarization, and structured extraction. | Explanation of AI usage sticky note.                                                            |
| Bright Data Logo included for branding: ![Bright Data Logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png) | Branding sticky note.                                                                            |
| Webhook URL placeholder `https://webhook.site/7b5380a0-0544-48dc-be43-0116cb2d52c2` should be replaced with a user endpoint. | Webhook notification node configuration note.                                                  |
| Google Sheets node requires OAuth2 credentials with access to target spreadsheet.                   | Google Sheets integration requirements.                                                        |
| The same filename `d:\Crunchbase-Summary.json` is used for both summary and structured data storage; consider using distinct filenames to avoid overwriting. | File storage consideration.                                                                     |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated workflow created with n8n, adhering strictly to current content policies without any illegal or protected elements. All processed data is legal and public.

---