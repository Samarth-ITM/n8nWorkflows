TrustPilot SaaS Product Review Tracker with Bright Data & OpenAI

https://n8nworkflows.xyz/workflows/trustpilot-saas-product-review-tracker-with-bright-data---openai-4280


# TrustPilot SaaS Product Review Tracker with Bright Data & OpenAI

### 1. Workflow Overview

This workflow automates the extraction, transformation, and storage of product reviews for a SaaS company (Bright Data) from TrustPilot. It leverages web scraping via Bright Data's Web Unlocker, processes the retrieved markdown content into structured textual data using OpenAI models, extracts structured JSON review data, summarizes it, and finally pushes the results to multiple outbound destinations including Google Sheets, disk storage, and a webhook notification.

**Logical Blocks:**

- **1.1 Input Reception and Configuration:** Entry trigger and setting URLs and scraping zone.
- **1.2 Data Retrieval:** Performing the web request to scrape TrustPilot data through Bright Data.
- **1.3 Data Transformation:** Converting markdown scraped data into clean textual content.
- **1.4 Structured Data Extraction:** Extracting structured JSON data from the textual content.
- **1.5 Summarization:** Creating a concise summary of the extracted information.
- **1.6 Outbound Data Handling:** Merging, aggregating, saving, and distributing the processed data to external systems.
- **1.7 Supporting Notes and Credentials:** Documentation notes and API credential usage.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Configuration

**Overview:**  
Starts the workflow manually and sets the necessary input parameters including the TrustPilot URL to scrape and the Bright Data zone name.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Set URL and Bright Data Zone

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution of this workflow.  
  - Config: No parameters; triggers workflow on manual user action.  
  - Connections: Output → Set URL and Bright Data Zone  
  - Failures: None typical; manual start only.

- **Set URL and Bright Data Zone**  
  - Type: Set  
  - Role: Defines key variables — the TrustPilot review URL and Bright Data scraping zone.  
  - Config: Sets two string parameters:  
    - `url`: "https://www.trustpilot.com/review/brightdata.com"  
    - `zone`: "web_unlocker1" (Bright Data zone for scraping)  
  - Connections: Input → Manual Trigger; Output → Perform Bright Data Web Request  
  - Failures: Misconfiguration of URL or zone will cause data retrieval issues.

---

#### 1.2 Data Retrieval

**Overview:**  
Executes the web scraping request via Bright Data API to fetch the raw markdown content of the TrustPilot page.

**Nodes Involved:**  
- Perform Bright Data Web Request

**Node Details:**

- **Perform Bright Data Web Request**  
  - Type: HTTP Request  
  - Role: Sends POST request to Bright Data API to retrieve TrustPilot page content.  
  - Config:  
    - URL: https://api.brightdata.com/request  
    - Method: POST  
    - Body Parameters: zone (from input), url (from input), format=raw, data_format=markdown  
    - Auth: HTTP Header Auth with credentials named "Header Auth account"  
  - Connections: Input → Set URL and Bright Data Zone; Output → Markdown to Textual Data Extractor  
  - Failures: HTTP errors, auth failures, Bright Data API quota limits, malformed request body.

---

#### 1.3 Data Transformation

**Overview:**  
Converts the raw markdown content obtained from Bright Data into clean, textual data suitable for structured extraction and summarization.

**Nodes Involved:**  
- Markdown to Textual Data Extractor  
- OpenAI Chat Model (used internally by Markdown to Textual Data Extractor)

**Node Details:**

- **Markdown to Textual Data Extractor**  
  - Type: LangChain LLM Chain Node (chainLlm)  
  - Role: Processes markdown input using OpenAI GPT-4o-mini to output plain textual content without links, scripts, or CSS.  
  - Config:  
    - Prompt: "You need to analyze the below markdown and convert to textual data. Please do not output with your own thoughts. Make sure to output with textual data only with no links, scripts, css etc."  
    - System message: "You are a markdown expert"  
    - Input: `data` field from Bright Data response  
  - Connections: Input → Perform Bright Data Web Request; Output → Structured JSON Data Extractor, Summarization Chain  
  - Failures: API timeouts, prompt interpretation errors, malformed markdown input.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides GPT-4o-mini model used internally by the Markdown to Textual Data Extractor.  
  - Config: Model set to "gpt-4o-mini" with OpenAI credentials "OpenAi account".  
  - Connections: Invoked by Markdown to Textual Data Extractor node.  
  - Failures: API key issues, rate limits.

---

#### 1.4 Structured Data Extraction

**Overview:**  
Extracts detailed structured JSON data representing the reviews from the textual data output.

**Nodes Involved:**  
- Structured JSON Data Extractor  
- OpenAI Chat Model for Structured Data  
- Code to extract the first element

**Node Details:**

- **Structured JSON Data Extractor**  
  - Type: LangChain Information Extractor  
  - Role: Uses OpenAI to parse the textual data and extract structured review information formatted as JSON-LD schema.org.  
  - Config:  
    - Input text: `"Extract the review mentioned below\n\n {{ $json.text }}"`  
    - JSON schema example provided to guide extraction including organization info, ratings, reviews, etc.  
  - Connections: Input → Markdown to Textual Data Extractor; Output → Code to extract the first element  
  - Failures: Extraction inaccuracies, schema mismatches, API failures.

- **OpenAI Chat Model for Structured Data**  
  - Type: LangChain OpenAI Chat Model  
  - Role: GPT-4o-mini used internally for the structured data extraction.  
  - Config: Same OpenAI credentials as above.  
  - Connections: Invoked by Structured JSON Data Extractor node.  
  - Failures: Same as other OpenAI nodes.

- **Code to extract the first element**  
  - Type: Code (JavaScript)  
  - Role: Simplifies the output by extracting the first element from the array output of the extractor to flatten data flow.  
  - Config: `return $input.first().json.output`  
  - Connections: Input → Structured JSON Data Extractor; Output → Merge the responses (second input)  
  - Failures: Empty input arrays, unexpected data structures.

---

#### 1.5 Summarization

**Overview:**  
Creates a concise summary of the textual content derived from the markdown data.

**Nodes Involved:**  
- Summarization Chain  
- OpenAI Chat Model for Summarization  
- Merge the responses  
- Aggregate the responses

**Node Details:**

- **Summarization Chain**  
  - Type: LangChain ChainSummarization  
  - Role: Summarizes the textual content using an advanced chunking mode and a defined prompt.  
  - Config: Prompt - "Write a concise summary of the following:\n\n\"{text}\""  
  - Connections: Input → Markdown to Textual Data Extractor; Output → Merge the responses (first input)  
  - Failures: API limits, chunking errors.

- **OpenAI Chat Model for Summarization**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides GPT-4o-mini for summarization.  
  - Config: OpenAI credentials as above.  
  - Connections: Invoked by Summarization Chain.  
  - Failures: Same as other OpenAI nodes.

- **Merge the responses**  
  - Type: Merge  
  - Role: Merges outputs from the summarization chain and the code node extracting structured data's first element for combined downstream processing.  
  - Connections: Inputs - Summarization Chain (index 0), Code to extract the first element (index 1); Output → Aggregate the responses  
  - Failures: Mismatched input counts.

- **Aggregate the responses**  
  - Type: Aggregate  
  - Role: Aggregates all item data from merge for final unified output.  
  - Connections: Input → Merge the responses; Outputs → Google Sheets, Webhook Notification, Create a binary data for Structured Data Extract  
  - Failures: Large data size, aggregation errors.

---

#### 1.6 Outbound Data Handling

**Overview:**  
Saves and pushes the processed structured data and summaries to external systems for storage and notification.

**Nodes Involved:**  
- Google Sheets  
- Initiate a Webhook Notification for the Structured Data  
- Create a binary data for Structured Data Extract  
- Write the structured content to disk

**Node Details:**

- **Google Sheets**  
  - Type: Google Sheets  
  - Role: Appends or updates the processed data into a Google Sheet (SaaS Product Review spreadsheet).  
  - Config:  
    - Operation: Append or Update  
    - Document ID: Google Sheet with ID "1wRJ1tInpPq35WJuKwWaD_nPvcsXmsXJxWgcTjhAinEQ"  
    - Sheet Name: "gid=0" (Sheet1)  
    - Mapping columns for field "data" auto-mapped from input  
    - Credentials: Google Sheets OAuth2  
  - Connections: Input → Aggregate the responses  
  - Failures: Auth token expiration, API quota, sheet access restrictions.

- **Initiate a Webhook Notification for the Structured Data**  
  - Type: HTTP Request  
  - Role: Sends a webhook POST with the summary data to a configured webhook URL for downstream notification/processing.  
  - Config:  
    - URL: "https://webhook.site/7b5380a0-0544-48dc-be43-0116cb2d52c2" (replaceable)  
    - Sends body parameter "summary" with JSON data  
  - Connections: Input → Aggregate the responses  
  - Failures: Webhook endpoint unavailability, network errors.

- **Create a binary data for Structured Data Extract**  
  - Type: Function  
  - Role: Converts structured JSON data into base64-encoded binary format for file writing.  
  - Config: Uses Node.js Buffer to encode JSON stringified data.  
  - Connections: Input → Aggregate the responses; Output → Write the structured content to disk  
  - Failures: Large data size, encoding errors.

- **Write the structured content to disk**  
  - Type: Read/Write File  
  - Role: Writes the binary JSON structured data to a local disk file named "d:\TrustPilot-StructuredData.json".  
  - Config: Write operation with fixed path.  
  - Connections: Input → Create a binary data for Structured Data Extract  
  - Failures: File system permissions, path existence, disk space.

---

#### 1.7 Supporting Notes and Credentials

**Overview:**  
Contains sticky notes for user guidance and documentation, as well as usage of credentials for Bright Data and OpenAI.

**Nodes Involved:**  
- Sticky Note (general workflow note)  
- Sticky Note1 (LLM usages)  
- Sticky Note2 (Summarization)  
- Sticky Note3 (Structured Data Extract)  
- Sticky Note4 (Outbound Data Push)  
- Sticky Note5 (Logo display)  
- Credentials: OpenAI API, Bright Data HTTP Header Auth, Google Sheets OAuth2

**Node Details:**

- Sticky Notes provide contextual information on workflow sections, usage instructions, and branding.  
- Credentials used securely for API authentication.  
- Failures: Missing or invalid credentials cause API call failures.

---

### 3. Summary Table

| Node Name                             | Node Type                             | Functional Role                            | Input Node(s)                      | Output Node(s)                                     | Sticky Note                                                                                       |
|-------------------------------------|-------------------------------------|-------------------------------------------|----------------------------------|---------------------------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’        | Manual Trigger                      | Workflow start                            | -                                | Set URL and Bright Data Zone                       | ## Note Deals with the Trust Pilot data extraction and summarization by utilizing the Bright Data Web Unlocker Product. Please make sure to set the TrustPilot URL with the Bright Data zone name. Also update the Webhook Notification URL of your interest |
| Set URL and Bright Data Zone         | Set                                 | Define TrustPilot URL and Bright Data zone | When clicking ‘Test workflow’    | Perform Bright Data Web Request                    |                                                                                                 |
| Perform Bright Data Web Request      | HTTP Request                       | Scrape TrustPilot page via Bright Data API | Set URL and Bright Data Zone     | Markdown to Textual Data Extractor                 |                                                                                                 |
| Markdown to Textual Data Extractor   | LangChain LLM Chain (chainLlm)     | Convert markdown to clean textual content | Perform Bright Data Web Request  | Structured JSON Data Extractor, Summarization Chain | ## LLM Usages OpenAI GPT 4o model is being used. Basic LLM Chain for markdown to text conversion. Information Extractor node transforms markdown to text. Summarization Chain creates summaries. |
| OpenAI Chat Model                   | LangChain LM Chat OpenAI            | Provides GPT-4o-mini model for markdown conversion | Invoked by Markdown to Textual Data Extractor | Markdown to Textual Data Extractor (internal)      |                                                                                                 |
| Structured JSON Data Extractor       | LangChain Information Extractor     | Extract detailed structured JSON review data | Markdown to Textual Data Extractor | Code to extract the first element                 | ## Structured Data Extract                                                                         |
| OpenAI Chat Model for Structured Data | LangChain LM Chat OpenAI            | Provides GPT-4o-mini model for structured data extraction | Invoked by Structured JSON Data Extractor | Structured JSON Data Extractor (internal)          |                                                                                                 |
| Code to extract the first element    | Code                               | Simplify output by extracting first element | Structured JSON Data Extractor   | Merge the responses (input index 1)                |                                                                                                 |
| Summarization Chain                  | LangChain ChainSummarization        | Summarizes textual content                | Markdown to Textual Data Extractor | Merge the responses (input index 0)                | ## Summarization                                                                                 |
| OpenAI Chat Model for Summarization  | LangChain LM Chat OpenAI            | Provides GPT-4o-mini model for summarization | Invoked by Summarization Chain   | Summarization Chain (internal)                      |                                                                                                 |
| Merge the responses                  | Merge                              | Merge structured data and summary outputs | Summarization Chain, Code to extract the first element | Aggregate the responses                             | ## Outbound Data Push Outbound data handling by merging, aggregating and pushing to multiple sources such as Google Sheets, disk, webhook |
| Aggregate the responses             | Aggregate                          | Aggregate all merged data                 | Merge the responses              | Google Sheets, Initiate a Webhook Notification, Create a binary data for Structured Data Extract |                                                                                                 |
| Google Sheets                      | Google Sheets                      | Append or update data into Google Sheet   | Aggregate the responses          | -                                                 |                                                                                                 |
| Initiate a Webhook Notification for the Structured Data | HTTP Request                       | Send summary data via webhook POST        | Aggregate the responses          | -                                                 |                                                                                                 |
| Create a binary data for Structured Data Extract | Function                           | Encode structured JSON data to base64 binary | Aggregate the responses          | Write the structured content to disk               |                                                                                                 |
| Write the structured content to disk | Read/Write File                   | Write binary encoded JSON to disk          | Create a binary data for Structured Data Extract | -                                                 |                                                                                                 |
| Sticky Note                       | Sticky Note                       | Documentation note                         | -                              | -                                                 | See notes in section 1.7                                                                           |
| Sticky Note1                      | Sticky Note                       | Documentation note on LLM usage           | -                              | -                                                 | See notes in section 1.7                                                                           |
| Sticky Note2                      | Sticky Note                       | Summarization block note                   | -                              | -                                                 | See notes in section 1.7                                                                           |
| Sticky Note3                      | Sticky Note                       | Structured data extraction block note     | -                              | -                                                 | See notes in section 1.7                                                                           |
| Sticky Note4                      | Sticky Note                       | Outbound data handling block note         | -                              | -                                                 | See notes in section 1.7                                                                           |
| Sticky Note5                      | Sticky Note                       | Logo display                               | -                              | -                                                 | See notes in section 1.7                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Test workflow’"  
   - No parameters.

2. **Create Set Node**  
   - Type: Set  
   - Name: "Set URL and Bright Data Zone"  
   - Add two string fields:  
     - `url`: "https://www.trustpilot.com/review/brightdata.com"  
     - `zone`: "web_unlocker1"  
   - Connect manual trigger output to this node.

3. **Create HTTP Request Node for Bright Data**  
   - Type: HTTP Request  
   - Name: "Perform Bright Data Web Request"  
   - HTTP Method: POST  
   - URL: https://api.brightdata.com/request  
   - Body Parameters:  
     - `zone`: reference from Set node (`{{$json.zone}}`)  
     - `url`: reference from Set node (`{{$json.url}}`)  
     - `format`: "raw"  
     - `data_format`: "markdown"  
   - Authentication: HTTP Header Auth; configure credentials for Bright Data API key.  
   - Connect Set node output to this node.

4. **Create LangChain Chain LLM Node to convert Markdown to Text**  
   - Type: LangChain Chain LLM (chainLlm)  
   - Name: "Markdown to Textual Data Extractor"  
   - Prompt:  
     ```
     You need to analyze the below markdown and convert to textual data. Please do not output with your own thoughts. Make sure to output with textual data only with no links, scripts, css etc.

     {{ $json.data }}
     ```  
   - System message: "You are a markdown expert"  
   - Use OpenAI GPT-4o-mini as language model with your OpenAI API credentials.  
   - Connect HTTP Request node output to this node.

5. **Create LangChain Information Extractor Node**  
   - Type: LangChain Information Extractor  
   - Name: "Structured JSON Data Extractor"  
   - Text parameter:  
     ```
     Extract the review mentioned below

     {{ $json.text }}
     ```  
   - Provide JSON schema example describing organization and review structure (as per workflow).  
   - Use OpenAI GPT-4o-mini with credentials.  
   - Connect Markdown to Textual Data Extractor output to this node.

6. **Create Code Node to extract first element**  
   - Type: Code (JavaScript)  
   - Name: "Code to extract the first element"  
   - Code: `return $input.first().json.output`  
   - Connect Structured JSON Data Extractor output to this node.

7. **Create LangChain Chain Summarization Node**  
   - Type: LangChain Chain Summarization  
   - Name: "Summarization Chain"  
   - Prompt:  
     ```
     Write a concise summary of the following:

     "{text}"
     ```  
   - Chunking Mode: Advanced  
   - Use OpenAI GPT-4o-mini with credentials.  
   - Connect Markdown to Textual Data Extractor output to this node.

8. **Create Merge Node**  
   - Type: Merge  
   - Name: "Merge the responses"  
   - Connect Summarization Chain output to input 0.  
   - Connect Code node output (first element) to input 1.

9. **Create Aggregate Node**  
   - Type: Aggregate  
   - Name: "Aggregate the responses"  
   - Aggregate all item data.  
   - Connect Merge node output to this node.

10. **Create Google Sheets Node**  
    - Type: Google Sheets  
    - Name: "Google Sheets"  
    - Operation: Append or Update  
    - Document ID: Your Google Sheets ID (e.g., "1wRJ1tInpPq35WJuKwWaD_nPvcsXmsXJxWgcTjhAinEQ")  
    - Sheet Name: "gid=0"  
    - Map field "data" to input.  
    - Use OAuth2 credentials for Google Sheets.  
    - Connect Aggregate node output to this node.

11. **Create HTTP Request Node for Webhook Notification**  
    - Type: HTTP Request  
    - Name: "Initiate a Webhook Notification for the Structured Data"  
    - Method: POST  
    - URL: Set to your webhook URL or test URL (e.g., "https://webhook.site/your-unique-url")  
    - Body Parameters: pass field `summary` with data from aggregate node (`{{$json.data}}`)  
    - Connect Aggregate node output to this node.

12. **Create Function Node to encode JSON to binary**  
    - Type: Function  
    - Name: "Create a binary data for Structured Data Extract"  
    - Code:  
      ```javascript
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
        }
      };
      return items;
      ```  
    - Connect Aggregate node output to this node.

13. **Create Read/Write File Node**  
    - Type: Read/Write File  
    - Name: "Write the structured content to disk"  
    - Operation: Write  
    - File Name: "d:\\TrustPilot-StructuredData.json" (adjust path as needed)  
    - Connect Function node output to this node.

14. **Add Sticky Note Nodes**  
    - Add notes with content as per documentation to guide users on configuration, LLM usage, summarization, structured data extraction, outbound data push, and branding/logo display.

15. **Verify all credentials**  
    - Bright Data HTTP Header Auth  
    - OpenAI API key  
    - Google Sheets OAuth2 credentials  

16. **Connect all nodes according to dependencies:**  
    - Manual Trigger → Set URL and Bright Data Zone → Perform Bright Data Web Request → Markdown to Textual Data Extractor  
    - Markdown to Textual Data Extractor → Structured JSON Data Extractor → Code to extract first element → Merge the responses (input 1)  
    - Markdown to Textual Data Extractor → Summarization Chain → Merge the responses (input 0)  
    - Merge the responses → Aggregate the responses → Google Sheets, Initiate Webhook Notification, Create binary data node → Write file node  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Deals with the Trust Pilot data extraction and summarization by utilizing the Bright Data Web Unlocker Product. Please make sure to set the TrustPilot URL with the Bright Data zone name. Also update the Webhook Notification URL of your interest | Sticky Note near workflow start                                                                                         |
| OpenAI GPT 4o model is being used. Basic LLM Chain converts markdown to textual content. Information Extractor node transforms markdown to text. Summarization Chain creates summaries from extracted content.                    | Sticky Note1 about LLM Usages                                                                                           |
| Summarization block handles concise summary creation of extracted textual data.                                                                                                                                                   | Sticky Note2                                                                                                            |
| Structured Data Extract block is responsible for extracting detailed review data into JSON schema format.                                                                                                                        | Sticky Note3                                                                                                            |
| Outbound Data Push block merges, aggregates, and pushes data to Google Sheets, disk, and webhook endpoints.                                                                                                                       | Sticky Note4                                                                                                            |
| ![Bright Data Logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png)                                                                                                                            | Sticky Note5 (branding/logo)                                                                                            |
| Use up-to-date API keys and manage quotas for Bright Data, OpenAI, and Google Sheets to avoid failures.                                                                                                                           | General operational advice                                                                                              |
| Replace the webhook.site URL with your own webhook endpoint for production usage.                                                                                                                                                  | Webhook node configuration                                                                                                |

---

This documentation provides a complete and detailed reference enabling reproduction, modification, and troubleshooting of the "TrustPilot SaaS Product Review Tracker with Bright Data & OpenAI" workflow.