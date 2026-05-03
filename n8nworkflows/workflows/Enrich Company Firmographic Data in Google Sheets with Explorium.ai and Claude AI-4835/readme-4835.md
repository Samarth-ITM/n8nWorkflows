Enrich Company Firmographic Data in Google Sheets with Explorium.ai and Claude AI

https://n8nworkflows.xyz/workflows/enrich-company-firmographic-data-in-google-sheets-with-explorium-ai-and-claude-ai-4835


# Enrich Company Firmographic Data in Google Sheets with Explorium.ai and Claude AI

### 1. Workflow Overview

This workflow automates the enrichment of company firmographic data stored in a Google Sheet by integrating Explorium.ai’s MCP (Machine Comprehension Platform) and Claude AI (Anthropic's language model). It processes newly added or updated rows containing company name and website, enriches them with detailed business metrics, and writes the enhanced data back to the sheet.

The workflow is logically divided into these functional blocks:

- **1.1 Input Reception:** Detects changes or additions in a Google Sheet using a trigger.
- **1.2 Data Validation:** Filters rows to ensure necessary fields are present before processing.
- **1.3 Iterative Processing:** Splits valid rows into individual items for sequential enrichment.
- **1.4 AI Data Enrichment:** Uses a LangChain AI Agent that orchestrates Explorium MCP calls and Claude AI to retrieve and parse firmographic data.
- **1.5 Output Formatting:** Reformats AI response into structured fields.
- **1.6 Data Update:** Appends or updates enriched data back into the Google Sheet row.
- **1.7 Loop Control:** Manages continuation of processing until all rows are handled.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block listens for changes in a specific Google Sheet and triggers the workflow whenever a new row is added or an existing row is updated.
- **Nodes Involved:**  
  - Google Sheets Trigger

- **Node Details:**

  - **Google Sheets Trigger**  
    - *Type:* Trigger node for Google Sheets  
    - *Configuration:* Polls every minute (`mode: everyMinute`) on a specific document (ID `1bAMUxQ2UCU_rd1khEkNZ0RKqmBzal7eKk90-AnVScQE`) and sheet (Sheet1, `gid=0`).  
    - *Inputs:* None (trigger node)  
    - *Outputs:* Emits rows with data changes including fields like `name`, `website`, `row_number`, and others.  
    - *Edge Cases:*  
      - Missing or revoked Google credentials may cause authentication errors.  
      - High frequency polling may lead to rate limits by Google Sheets API.  
      - Sheet or document ID changes require reconfiguration.  
    - *Version Requirements:* n8n Google Sheets Trigger node v1 or later.

#### 2.2 Data Validation

- **Overview:** Filters incoming rows to process only those with non-empty `name` and `website` fields, ensuring meaningful inputs for enrichment.
- **Nodes Involved:**  
  - Filter Valid Rows

- **Node Details:**

  - **Filter Valid Rows**  
    - *Type:* Conditional (If) node  
    - *Configuration:* Checks that both `name` and `website` fields are not empty strings. Uses strict string validation and case sensitivity.  
    - *Inputs:* Data from Google Sheets Trigger  
    - *Outputs:* Passes valid rows to next block; invalid rows are discarded (no output).  
    - *Edge Cases:*  
      - Rows missing either field are skipped silently.  
      - Strings containing whitespace only might need trimming outside this workflow to avoid false positives.  
    - *Version Requirements:* If node v2 or later for strict validation.

#### 2.3 Iterative Processing

- **Overview:** Splits the filtered rows into individual items to be processed one at a time, enabling controlled, sequential API calls and error isolation per row.
- **Nodes Involved:**  
  - Loop Over Items

- **Node Details:**

  - **Loop Over Items**  
    - *Type:* SplitInBatches node  
    - *Configuration:* Splits input items into batches of size 1 (default).  
    - *Inputs:* Valid rows from Filter Valid Rows  
    - *Outputs:* Sends single rows iteratively to AI Agent.  
    - *Edge Cases:*  
      - Large datasets may cause delays due to sequential processing.  
      - Failure processing one item does not block others.  
    - *Version Requirements:* SplitInBatches node v3 or later.

#### 2.4 AI Data Enrichment

- **Overview:** The core logic where firmographic data enrichment occurs by querying Explorium MCP for business matching and enrichment, and parsing output via Claude AI model.
- **Nodes Involved:**  
  - AI Agent  
  - Explorium MCP  
  - Anthropic Chat Model  
  - Output Parser

- **Node Details:**

  - **AI Agent**  
    - *Type:* LangChain AI Agent node  
    - *Configuration:* Receives a prompt template that includes company `name` and `website`. It instructs the agent to:  
      1. Use Explorium’s "match-business" tool to find the business ID.  
      2. Use Explorium’s "enrich-businesses-firmographics" tool to get detailed data.  
      3. Return business metrics in a structured format with fields like `name`, `website`, `business_id`, `naics`, `number_of_employees_range`, and `yearly_revenue_range`.  
    - *Inputs:* Single company row from Loop Over Items  
    - *Outputs:* Raw AI response JSON with enriched firmographic data.  
    - *Sub-nodes:* Connected to Explorium MCP (as `ai_tool`), Anthropic Chat Model (language model), and Output Parser (to parse AI output).  
    - *Edge Cases:*  
      - API rate limiting or downtime on Explorium or Anthropic services.  
      - Unexpected AI output formats causing parse errors.  
      - Missing or incorrect credentials causing auth failures.  
    - *Version Requirements:* LangChain agent v1.6 or later.

  - **Explorium MCP**  
    - *Type:* LangChain MCP Client Tool node  
    - *Configuration:* Connects to Explorium MCP SSE endpoint (`https://mcp.explorium.ai/sse`) using Bearer token authentication.  
    - *Inputs:* Invoked by AI Agent as an external AI tool for business data.  
    - *Outputs:* Business matching and enrichment results streamed to AI Agent.  
    - *Edge Cases:* Network failures, invalid token, or service downtime.  
    - *Version Requirements:* MCP Client Tool v1.

  - **Anthropic Chat Model**  
    - *Type:* LangChain Anthropic chat language model node  
    - *Configuration:* Uses Claude Sonnet 4 model (`claude-sonnet-4-20250514`) with provided Anthropic API credentials.  
    - *Inputs:* Prompt from AI Agent containing company info and instructions.  
    - *Outputs:* Generated text response with enriched data.  
    - *Edge Cases:* API quota exceeded, latency, invalid API key.  
    - *Version Requirements:* Anthropic LM node v1.3.

  - **Output Parser**  
    - *Type:* LangChain Structured Output Parser node  
    - *Configuration:* Parses AI output to comply with a JSON schema example listing fields:  
      ```json
      {
        "name": "Microsoft Corporation",
        "website": "https://www.microsoft.com",
        "business_id": "a34bacf839b923770b2c360eefa26748",
        "naics": "511210",
        "number_of_employees_range": "10001+",
        "yearly_revenue_range": "100B-1T"
      }
      ```  
    - *Inputs:* AI Agent raw output  
    - *Outputs:* Structured JSON data for downstream use.  
    - *Edge Cases:* Parsing errors if AI output deviates from expected format.  
    - *Version Requirements:* Output Parser v1.2.

#### 2.5 Output Formatting

- **Overview:** This block standardizes the parsed AI response, ensuring all expected fields exist and defaults are applied if missing.
- **Nodes Involved:**  
  - Code - Format Output

- **Node Details:**

  - **Code - Format Output**  
    - *Type:* Code node (JavaScript)  
    - *Configuration:* Runs JS code returning an object with keys:  
      - `name`, `website` (fallback to original input if missing)  
      - `business_id`, `naics`, `number_of_employees_range`, `yearly_revenue_range` (empty string fallback)  
    - *Inputs:* Parsed AI output from Output Parser  
    - *Outputs:* Clean, consistent JSON object for update  
    - *Edge Cases:* Missing or undefined fields handled gracefully here.  
    - *Version Requirements:* Code node v2.

#### 2.6 Data Update

- **Overview:** Updates or appends the enriched firmographic data back into the Google Sheet, matching rows based on the `name` column.
- **Nodes Involved:**  
  - Update Company Row

- **Node Details:**

  - **Update Company Row**  
    - *Type:* Google Sheets node (append or update)  
    - *Configuration:*  
      - Document and sheet same as trigger node.  
      - Columns mapped automatically from input data keys.  
      - Matching column: `name` to find existing rows and update them; otherwise, append new row.  
    - *Inputs:* Formatted output data from Code - Format Output  
    - *Outputs:* Confirmation of write operation to Google Sheets  
    - *Edge Cases:*  
      - Name duplicates in sheet may cause unintended updates.  
      - API quota or permission issues may cause write failure.  
      - Sheet structure changes require reconfiguration.  
    - *Version Requirements:* Google Sheets node v4.5 or later.

#### 2.7 Loop Control

- **Overview:** Controls iterative flow by feeding back updated items to Loop Over Items, continuing until all rows are processed.
- **Nodes Involved:**  
  - Loop Over Items (main output connections)

- **Node Details:**

  - The Loop Over Items node receives branches from Filter Valid Rows and outputs processed items one by one to AI Agent.  
  - After Google Sheets update, flow returns to Loop Over Items to fetch next item until completion.

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                   |
|---------------------|----------------------------------|------------------------------|------------------------|-----------------------|----------------------------------------------------------------------------------------------|
| Google Sheets Trigger| Google Sheets Trigger             | Input Reception - trigger     | None                   | Filter Valid Rows      |                                                                                              |
| Filter Valid Rows    | If (Conditional)                  | Data Validation               | Google Sheets Trigger  | Loop Over Items        |                                                                                              |
| Loop Over Items      | SplitInBatches                   | Iterative Processing          | Filter Valid Rows       | AI Agent, Loop Over Items |                                                                                              |
| AI Agent            | LangChain AI Agent                | AI Data Enrichment            | Loop Over Items (batch) | Code - Format Output   |                                                                                              |
| Explorium MCP        | LangChain MCP Client Tool         | External AI Tool for enrichment | AI Agent (ai_tool)      | AI Agent              |                                                                                              |
| Anthropic Chat Model | LangChain Anthropic LM            | Language model for prompt completion | AI Agent (ai_languageModel) | AI Agent              |                                                                                              |
| Output Parser        | LangChain Structured Output Parser| Parses AI response             | AI Agent (ai_outputParser) | AI Agent              |                                                                                              |
| Code - Format Output | Code Node (JavaScript)            | Output Formatting             | AI Agent               | Update Company Row     |                                                                                              |
| Update Company Row   | Google Sheets (appendOrUpdate)    | Updates enriched data in sheet | Code - Format Output    | Loop Over Items        |                                                                                              |
| Sticky Note          | Sticky Note                      | Documentation note            | None                   | None                  | # Enrich Company Data from Google Sheets with Explorium MCP <br> Describes workflow structure and loop behavior. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Type: Google Sheets Trigger  
   - Poll every minute (set `pollTimes` mode to `everyMinute`)  
   - Set Document ID: `1bAMUxQ2UCU_rd1khEkNZ0RKqmBzal7eKk90-AnVScQE`  
   - Set Sheet Name: `gid=0` (Sheet1)  
   - Connect as the workflow’s starting node.

2. **Add Filter Valid Rows Node**  
   - Type: If node  
   - Condition:  
     - `name` field is not empty  
     - `website` field is not empty  
   - Both conditions must be true (AND combinator)  
   - Connect input from Google Sheets Trigger output.

3. **Add Loop Over Items Node**  
   - Type: SplitInBatches  
   - Batch Size: 1 (default)  
   - Connect input from Filter Valid Rows node’s true output.

4. **Set up AI Agent Node**  
   - Type: LangChain AI Agent  
   - Prompt:  
     ```
     You will be provided with company information including:

     Name: {{ $json.name }}
     Domain: {{ $json.website }}

     Your task is to research and extract the following key business metrics for this company:
     Required Data Points:
     - Name (keep the same as provided)
     - Website (keep the same as provided)
     - Annual revenue (as business_id first, then get the range)
     - Number of employees (range)
     - NAICS code

     Use the Explorium tools to:
     1. First use match-business to find the business ID
     2. Then use enrich-businesses-firmographics to get the company details

     Return the data exactly in this format.
     ```
   - Connect input from Loop Over Items output (second/main output) to AI Agent input.

5. **Add Explorium MCP Node**  
   - Type: LangChain MCP Client Tool  
   - SSE Endpoint: `https://mcp.explorium.ai/sse`  
   - Authentication: Bearer Auth with valid token credentials  
   - Link this node as `ai_tool` input for AI Agent node.

6. **Add Anthropic Chat Model Node**  
   - Type: LangChain Anthropic LM Chat  
   - Model: `claude-sonnet-4-20250514`  
   - Set credentials using an Anthropic API key  
   - Link this node as `ai_languageModel` input for AI Agent node.

7. **Add Output Parser Node**  
   - Type: LangChain Structured Output Parser  
   - JSON Schema Example:  
     ```json
     {
       "name": "Microsoft Corporation",
       "website": "https://www.microsoft.com",
       "business_id": "a34bacf839b923770b2c360eefa26748",
       "naics": "511210",
       "number_of_employees_range": "10001+",
       "yearly_revenue_range": "100B-1T"
     }
     ```  
   - Connect as `ai_outputParser` input to AI Agent.

8. **Add Code - Format Output Node**  
   - Type: Code node (JavaScript)  
   - Code:  
     ```javascript
     return {
       name: $json.output.name || $json.name,
       website: $json.output.website || $json.website,
       business_id: $json.output.business_id || '',
       naics: $json.output.naics || '',
       number_of_employees_range: $json.output.number_of_employees_range || '',
       yearly_revenue_range: $json.output.yearly_revenue_range || ''
     };
     ```  
   - Connect input from AI Agent main output.

9. **Add Update Company Row Node**  
   - Type: Google Sheets node (append or update)  
   - Document ID: same as trigger node  
   - Sheet Name: same as trigger node (`gid=0`)  
   - Operation: Append or Update  
   - Matching Column: `name`  
   - Map columns automatically for fields: `name`, `website`, `business_id`, `naics`, `number_of_employees_range`, `yearly_revenue_range`  
   - Connect input from Code - Format Output output.

10. **Connect Update Company Row output back to Loop Over Items**  
    - This ensures the loop continues processing remaining rows sequentially.

11. **Add a Sticky Note to document workflow structure and instructions** (optional)  
    - Content describing the workflow blocks, loop mechanism, and testing steps.

12. **Credential Setup**  
    - Google API OAuth2 credentials for Google Sheets Trigger and Google Sheets nodes.  
    - Bearer token credentials for Explorium MCP node.  
    - Anthropic API credentials for Anthropic Chat Model node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow enriches company data automatically when new rows or updates occur in a Google Sheet, leveraging Explorium.ai for firmographic enrichment and Claude AI for natural language processing.                                        | Workflow purpose summary                                                                           |
| The loop processes rows one at a time to isolate failures and respect API rate limits. Failed rows do not stop processing of others.                                                                                                         | Workflow design principle                                                                          |
| Ensure Google Sheets API quota and Explorium/Anthropic API quotas are sufficient for expected row volumes and frequency.                                                                                                                    | Operational consideration                                                                          |
| The prompt given to the AI Agent uses a two-step enrichment: match business to get ID, then enrich firmographics by ID; this ensures data accuracy and leverages Explorium MCP capabilities.                                                    | AI prompt design                                                                                   |
| Official Explorium MCP documentation: https://docs.explorium.ai/mcp                                                                                                                                                                          | External documentation                                                                             |
| Anthropic Claude AI model details: https://www.anthropic.com/index/claude                                                                                                                                                                    | External documentation                                                                             |

---

**Disclaimer:** The provided text is derived exclusively from an automated n8n workflow integration. It complies strictly with content policies, contains no illegal or offensive content, and only manipulates legal and public data.