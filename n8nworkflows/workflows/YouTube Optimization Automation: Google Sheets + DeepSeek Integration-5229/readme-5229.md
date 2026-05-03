YouTube Optimization Automation: Google Sheets + DeepSeek Integration

https://n8nworkflows.xyz/workflows/youtube-optimization-automation--google-sheets---deepseek-integration-5229


# YouTube Optimization Automation: Google Sheets + DeepSeek Integration

### 1. Workflow Overview

This workflow automates the optimization of YouTube video metadata using data from Google Sheets and AI-powered content generation via DeepSeek. It targets YouTube content creators or marketers seeking to enhance video titles and descriptions for better SEO and viewer engagement.

The logical blocks are:

- **1.1 Input Reception**: Manual trigger initiates the process and retrieves pending video URLs and keywords from a Google Sheet.
- **1.2 YouTube Data Extraction**: Fetches the raw HTML of each YouTube video page and extracts the current title and description using a JavaScript code node.
- **1.3 AI-Powered Title and Description Generation**: Sends the extracted data and target keyword to DeepSeek AI models to generate optimized titles and descriptions.
- **1.4 Data Aggregation and Update**: Merges original and new data, aggregates it, and updates the corresponding rows in the Google Sheet with the enhanced metadata and status.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Starts the workflow manually and retrieves rows marked as "Pending" from a specific Google Sheet.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get row(s) in sheet (Google Sheets)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Config: No parameters; triggers workflow execution.  
    - Input: None  
    - Output: Starts the flow to "Get row(s) in sheet" node.  
    - Edge cases: None; manual trigger ensures controlled execution.

  - **Get row(s) in sheet**  
    - Type: Google Sheets node, read operation  
    - Role: Retrieves rows where the "Status" column is "Pending" from a specified sheet.  
    - Config:  
      - Document ID: Google Sheet ID `"1zoIIfA03no9Q_huenS6qYrDFKdhrf1kFijDEHoiBZdk"`  
      - Sheet Name: Sheet ID `46645040`  
      - Filter: Lookup rows where `Status` equals `"Pending"`  
    - Credentials: Google Sheets OAuth2  
    - Input: Trigger from manual node  
    - Output: Passes rows to HTTP Request node  
    - Edge cases:  
      - No rows found (empty output)  
      - Google API auth errors or rate limits  
      - Sheet or document ID changes or access revoked

---

#### 2.2 YouTube Data Extraction

- **Overview:**  
  Fetches YouTube video pages for each URL obtained and extracts the original video title and description from the HTML content.

- **Nodes Involved:**  
  - HTTP Request  
  - Code

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Downloads the HTML content of each YouTube video URL.  
    - Config:  
      - URL: Dynamic, from `Url` field in each row JSON  
      - Headers: Sets `User-Agent` to `"Mozilla/5.0"` to mimic browser  
      - Sends headers as JSON  
    - Input: Rows from "Get row(s) in sheet"  
    - Output: Raw HTML to "Code" node  
    - Edge cases:  
      - 404 or unavailable video URLs  
      - Network timeouts  
      - YouTube blocking or bot detection

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Parses the HTML to extract YouTube's initial data JSON, then extracts video title and description.  
    - Config:  
      - Uses regex to find `ytInitialData` JavaScript variable  
      - Parses JSON safely  
      - Searches nested structures for `title` and `attributedDescription`  
    - Input: HTML from HTTP Request  
    - Output: JSON with keys `title` and `description`  
    - Edge cases:  
      - Missing or malformed `ytInitialData` leading to parse errors  
      - Unexpected YouTube page structure changes  
      - Null or empty title/description fallback

---

#### 2.3 AI-Powered Title and Description Generation

- **Overview:**  
  Uses DeepSeek AI language models to generate optimized video titles and descriptions based on extracted metadata and target keywords.

- **Nodes Involved:**  
  - DeepSeek Chat Model  
  - New Title Generating (LangChain LLM Chain)  
  - New Description Generating (LangChain LLM Chain)  
  - Code1  
  - Code2

- **Node Details:**

  - **DeepSeek Chat Model**  
    - Type: DeepSeek AI Chat Model  
    - Role: Acts as the AI engine backend for subsequent title and description generation nodes.  
    - Config: Empty options, uses DeepSeek API credentials  
    - Input: Feeds downstream AI nodes  
    - Output: Feeds "New Title Generating" and "New Description Generating"  
    - Edge cases:  
      - API authorization errors  
      - Rate limits or service downtime

  - **New Title Generating**  
    - Type: LangChain LLM Chain (OpenAI or DeepSeek powered)  
    - Role: Generates 3 optimized YouTube video titles based on old title and keyword.  
    - Config:  
      - Prompt instructs to create clickable, curiosity-driven titles including keywords naturally  
      - Outputs a single-line JSON with `newTitle` key only  
      - Uses expressions to inject old title and keyword from Google Sheet row  
    - Input: Output from "DeepSeek Chat Model"  
    - Output: Raw JSON text (string) with new titles  
    - Edge cases:  
      - AI output not compliant with single-line JSON format  
      - Missing or malformed inputs causing nonsensical output

  - **New Description Generating**  
    - Type: LangChain LLM Chain  
    - Role: Rewrites YouTube video description for engagement and SEO, preserving timeline sections.  
    - Config:  
      - Prompt instructs to keep timeline intact, improve engagement, include keyword, max 1500 chars  
      - Outputs JSON with `newDescription` key only  
      - Injects original description and keyword dynamically  
    - Input: Output from "DeepSeek Chat Model"  
    - Output: Raw JSON text (string) with new description  
    - Edge cases:  
      - Same as above for AI compliance and input validity

  - **Code1**  
    - Type: Code (JavaScript)  
    - Role: Parses the JSON string from "New Title Generating" into JSON object  
    - Config: `JSON.parse` on `$json.text`  
    - Input: Raw JSON text from "New Title Generating"  
    - Output: Parsed JSON with `newTitle` field  
    - Edge cases: JSON parse errors if AI output malformed

  - **Code2**  
    - Type: Code (JavaScript)  
    - Role: Parses the JSON string from "New Description Generating" into JSON object  
    - Config: Same as Code1 but for description  
    - Input: Raw JSON text from "New Description Generating"  
    - Output: Parsed JSON with `newDescription` field  
    - Edge cases: Same JSON parse error risks

---

#### 2.4 Data Aggregation and Update

- **Overview:**  
  Merges outputs from AI nodes and original data, aggregates them, and updates the Google Sheet row with new title, description, and status.

- **Nodes Involved:**  
  - Merge  
  - Aggregate  
  - Update row in sheet1 (Google Sheets)

- **Node Details:**

  - **Merge**  
    - Type: Merge node  
    - Role: Combines output from 3 sources:  
      - Parsed new title (Code1)  
      - Parsed new description (Code2)  
      - Original extracted data (Code)  
    - Config: Number of inputs set to 3, merging by index/order  
    - Input: Three separate JSON inputs as above  
    - Output: Single merged item with combined data  
    - Edge cases: Different array lengths or asynchronous results causing merge mismatch

  - **Aggregate**  
    - Type: Aggregate node  
    - Role: Aggregates all merged items into one array for batch updating  
    - Config: Aggregate all item data  
    - Input: Merged items stream  
    - Output: Single aggregated array  
    - Edge cases: Large data causing memory issues or slow processing

  - **Update row in sheet1**  
    - Type: Google Sheets node, update operation  
    - Role: Updates the original Google Sheet row with new values and marks status as "Done"  
    - Config:  
      - Matching column: `Url` to identify row  
      - Columns updated:  
        - `Etap`: set to `"Done"`  
        - `New Title`: from AI output  
        - `Old Title`: from original extraction  
        - `New Description`: from AI output  
        - `Old Description`: from original extraction  
        - `Url`: copied from original  
      - Same spreadsheet and sheet as input node  
    - Credentials: Google Sheets OAuth2  
    - Input: Aggregated data array  
    - Output: Final update confirmation  
    - Edge cases:  
      - Row not found due to URL mismatch  
      - Google Sheets permission or quota errors

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                      | Input Node(s)            | Output Node(s)                  | Sticky Note                             |
|-----------------------------|----------------------------------|------------------------------------|--------------------------|--------------------------------|---------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Starts workflow manually            | -                        | Get row(s) in sheet            |                                       |
| Get row(s) in sheet          | Google Sheets (Read)              | Reads rows with Status "Pending"    | When clicking ‘Execute workflow’ | HTTP Request                  |                                       |
| HTTP Request                | HTTP Request                     | Fetches YouTube video page HTML    | Get row(s) in sheet       | Code                          |                                       |
| Code                        | Code (JS)                       | Extracts title and description from HTML | HTTP Request             | Merge                         |                                       |
| DeepSeek Chat Model          | DeepSeek AI Chat Model            | AI engine for content generation    | -                        | New Title Generating, New Description Generating |                                       |
| New Title Generating         | LangChain LLM Chain               | Generates optimized titles          | DeepSeek Chat Model       | Code1                         |                                       |
| New Description Generating   | LangChain LLM Chain               | Generates optimized descriptions    | DeepSeek Chat Model       | Code2                         |                                       |
| Code1                       | Code (JS)                       | Parses AI JSON string for title     | New Title Generating      | Merge                         |                                       |
| Code2                       | Code (JS)                       | Parses AI JSON string for description | New Description Generating | Merge                         |                                       |
| Merge                       | Merge                           | Combines original and AI data       | Code1, Code2, Code        | Aggregate                     |                                       |
| Aggregate                   | Aggregate                       | Aggregates all merged items         | Merge                    | Update row in sheet1           |                                       |
| Update row in sheet1         | Google Sheets (Update)            | Updates Google Sheet with new SEO data and status | Aggregate                 | -                              |                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Name: "When clicking ‘Execute workflow’"  
   - No parameters; this triggers workflow execution manually.

2. **Add Google Sheets node (Read operation):**  
   - Name: "Get row(s) in sheet"  
   - Operation: Read rows  
   - Document ID: `"1zoIIfA03no9Q_huenS6qYrDFKdhrf1kFijDEHoiBZdk"`  
   - Sheet Name/ID: `46645040`  
   - Filter: Retrieve rows where column `Status` equals `"Pending"`  
   - Connect input from Manual Trigger node.  
   - Use Google Sheets OAuth2 credentials.

3. **Add HTTP Request node:**  
   - Name: "HTTP Request"  
   - HTTP Method: GET  
   - URL: Expression `{{$json.Url}}` (dynamic URL from each row)  
   - Headers: Set `"User-Agent": "Mozilla/5.0"` as JSON header  
   - Connect input from "Get row(s) in sheet".

4. **Add Code node to extract YouTube data:**  
   - Name: "Code"  
   - Language: JavaScript  
   - Paste the provided code to parse `ytInitialData` JSON from HTML, extract `title` and `description`.  
   - Connect input from "HTTP Request".

5. **Add DeepSeek Chat Model node:**  
   - Name: "DeepSeek Chat Model"  
   - Credentials: Provide DeepSeek API credentials  
   - No additional parameters.  
   - This node acts as the AI backend for next nodes.

6. **Add LangChain LLM Chain node for title generation:**  
   - Name: "New Title Generating"  
   - Use DeepSeek Chat Model as AI backend (set in credentials)  
   - Prompt: Include instructions to create 3 attention-grabbing, keyword-rich titles based on old title and keyword (use expressions to pass data).  
   - Output: Must be a single-line JSON with key `newTitle`.  
   - Connect input from "DeepSeek Chat Model".

7. **Add LangChain LLM Chain node for description generation:**  
   - Name: "New Description Generating"  
   - Same AI backend as above  
   - Prompt: Rewrite description for engagement and SEO, keeping timeline intact, max 1500 chars, output JSON with `newDescription`.  
   - Connect input from "DeepSeek Chat Model".

8. **Add Code node "Code1" to parse new title JSON:**  
   - Parse the JSON string from "New Title Generating" using `JSON.parse($json.text)`.  
   - Connect input from "New Title Generating".

9. **Add Code node "Code2" to parse new description JSON:**  
   - Same parsing logic as Code1 but for description.  
   - Connect input from "New Description Generating".

10. **Add Merge node:**  
    - Name: "Merge"  
    - Set number of inputs to 3.  
    - Connect inputs:  
      - Input 1: "Code1"  
      - Input 2: "Code2"  
      - Input 3: "Code" (original extracted data)  
    - Merge method: Default (merge by index/order).

11. **Add Aggregate node:**  
    - Name: "Aggregate"  
    - Operation: Aggregate all item data into one array.  
    - Connect input from "Merge".

12. **Add Google Sheets node (Update operation):**  
    - Name: "Update row in sheet1"  
    - Operation: Update row by matching `Url` column.  
    - Document ID and Sheet Name same as in step 2.  
    - Columns to update:  
      - `Etap`: set to `"Done"`  
      - `New Title`: from AI output `newTitle` in merged data  
      - `Old Title`: from original extracted title  
      - `New Description`: from AI output `newDescription`  
      - `Old Description`: from original extracted description  
      - `Url`: from original row  
    - Connect input from "Aggregate".  
    - Use Google Sheets OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow uses DeepSeek AI for NLP tasks including title and description rewriting, requiring API credentials. | DeepSeek API documentation or credentials setup is necessary.                                   |
| The Google Sheet must include columns: `Url`, `keyword`, `Status`, `Old Title`, `New Title`, `Old Description`, `New Description` for correct mapping. | Ensure Sheet schema matches these fields for proper operation.                                  |
| The workflow assumes YouTube page structure does not change drastically; modifications to the parsing code may be needed if YouTube updates their HTML/JS layout. | YouTube page parsing is fragile and can break with site changes.                                |
| User-Agent header in HTTP Request is used to avoid bot detection by YouTube.                            | Using `"Mozilla/5.0"` mimics browser request.                                                   |
| The AI nodes expect strict JSON output without markdown or explanations; otherwise JSON parsing will fail. | Prompt engineering is critical to maintain correct output format.                               |

---

**Disclaimer:**  
The text provided derives solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected content. All data handled is legal and publicly available.