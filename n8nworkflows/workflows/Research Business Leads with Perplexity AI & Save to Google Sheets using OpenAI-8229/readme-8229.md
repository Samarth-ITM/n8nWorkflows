Research Business Leads with Perplexity AI & Save to Google Sheets using OpenAI

https://n8nworkflows.xyz/workflows/research-business-leads-with-perplexity-ai---save-to-google-sheets-using-openai-8229


# Research Business Leads with Perplexity AI & Save to Google Sheets using OpenAI

### 1. Workflow Overview

This n8n workflow automates the research and logging of new business leads specifically for coffee shops near a given geographical area. It integrates Perplexity AI for initial lead research, OpenAI for structured data parsing and formatting, and Google Sheets for storage and updating of lead data.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Sets the target research area and retrieves existing leads from Google Sheets.
- **1.2 Lead Research via Perplexity AI:** Queries Perplexity AI to find new coffee shop leads in the target area that are not already scraped.
- **1.3 Data Structuring with OpenAI:** Uses OpenAI’s GPT model to convert the raw lead data into a structured JSON format.
- **1.4 Data Splitting and Merging:** Splits the structured data into separate company and email records and merges them back into combined rows.
- **1.5 Data Storage in Google Sheets:** Appends new or updates existing leads in a Google Sheets spreadsheet, keyed by email.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:** This block defines the target area for research and fetches existing leads from Google Sheets filtered by that area. It prepares the data context for the lead research step.
- **Nodes Involved:**
  - Start Workflow
  - Set Location
  - Get Current Leads
  - Combine Into One
  - Set as text
- **Node Details:**

  - **Start Workflow**
    - Type: Manual Trigger
    - Role: Entry point to manually start the workflow.
    - Configuration: No parameters.
    - Inputs: None.
    - Outputs: Connects to "Set Location".
    - Edge cases: None.

  - **Set Location**
    - Type: Set
    - Role: Defines the research area variable "Area" (string).
    - Configuration: Sets Area = "Hershey PA" (default example).
    - Inputs: From Start Workflow.
    - Outputs: Connects to "Get Current Leads".
    - Edge cases: Hardcoded area; parameterizing would improve flexibility.

  - **Get Current Leads**
    - Type: Google Sheets
    - Role: Retrieves existing leads filtered by Area from a specified Google Sheet.
    - Configuration:
      - Spreadsheet ID: "1MnaU8hSi8PleDNVcNnyJ5CgmDYJSUTsr7X5HIwa-MLk"
      - Worksheet: Sheet1 (gid=0)
      - Filter: Lookup rows where "Area" equals the Area variable from "Set Location".
    - Credentials: Google Sheets OAuth2.
    - Inputs: From "Set Location".
    - Outputs: Connects to "Combine Into One".
    - Edge cases:
      - Possible auth errors if credentials expire.
      - Empty or missing sheet data.

  - **Combine Into One**
    - Type: Aggregate
    - Role: Combines all retrieved rows into a single aggregated data object to simplify downstream usage.
    - Configuration: Aggregates all item data including binaries.
    - Inputs: From "Get Current Leads".
    - Outputs: Connects to "Set as text".
    - Edge cases: Empty input could produce null or empty aggregation.

  - **Set as text**
    - Type: Set
    - Role: Prepares text variables for downstream nodes, passing Area and aggregated data.
    - Configuration:
      - Sets "Search Area" to Area value.
      - Sets "data" to aggregated lead data.
    - Inputs: From "Combine Into One".
    - Outputs: Connects to "Research Leads".
    - Edge cases: Data may be empty or malformed.

#### 2.2 Lead Research via Perplexity AI

- **Overview:** This block uses the Perplexity API to research 20 coffee shops near the specified area. It excludes businesses already scraped, passing existing leads as context.
- **Nodes Involved:**
  - Research Leads
- **Node Details:**

  - **Research Leads**
    - Type: Perplexity API node
    - Role: Queries Perplexity AI using a "sonar" model to get new leads.
    - Configuration:
      - Model: "sonar"
      - Messages:
        - System: Instruction to research 20 coffee shops and only output company name and email.
        - User: Template including:
          - Area from "Set as text" node.
          - Already scraped data (existing leads) from "Set as text" node.
      - Simplify: true (for simplified output).
    - Credentials: Perplexity API key.
    - Inputs: From "Set as text".
    - Outputs: Connects to "Write JSON".
    - Edge cases:
      - API key invalid or rate limits.
      - Unexpected or incomplete AI response.
      - Empty or irrelevant results.

#### 2.3 Data Structuring with OpenAI

- **Overview:** Uses OpenAI GPT model to convert the raw lead data from Perplexity into a structured JSON format with arrays of companies and emails.
- **Nodes Involved:**
  - OpenAI Chat Model
  - Structured Output Parser
  - Write JSON
- **Node Details:**

  - **OpenAI Chat Model**
    - Type: Langchain OpenAI Chat Model
    - Role: Processes input with GPT-4.1 mini model.
    - Configuration:
      - Model: "gpt-4.1-mini"
      - Options: default
    - Credentials: OpenAI API key.
    - Inputs: From "Research Leads" (via AI pipeline).
    - Outputs: Connects to "Write JSON" as AI language model.
    - Edge cases:
      - API auth failure.
      - Model latency or errors.
      - Model output unexpected format.

  - **Structured Output Parser**
    - Type: Langchain Structured Output Parser
    - Role: Parses the OpenAI raw output to enforce a JSON schema.
    - Configuration:
      - JSON schema example with "Company" and "Email" arrays.
    - Inputs: From "Write JSON" AI output parser.
    - Outputs: Connects to "Write JSON".
    - Edge cases:
      - Parsing failures if output does not match schema.
      - Schema mismatch.

  - **Write JSON**
    - Type: Langchain Agent
    - Role: Converts AI chat message into JSON format with company and email arrays.
    - Configuration:
      - Prompt: Convert to JSON with a specific example format.
      - Has output parser enabled.
    - Inputs: From "Research Leads" and AI model/parser.
    - Outputs: Connects to "Split Companies" and "Split Emails".
    - Edge cases:
      - Parsing errors.
      - Missing fields or null values.

#### 2.4 Data Splitting and Merging

- **Overview:** Splits the structured JSON arrays of companies and emails into individual items, then merges them back in positional order for synchronized records.
- **Nodes Involved:**
  - Split Companies
  - Split Emails
  - Merge Rows
- **Node Details:**

  - **Split Companies**
    - Type: Split Out
    - Role: Splits the "output.Company" JSON array into individual items.
    - Configuration: Field to split out is "output.Company".
    - Inputs: From "Write JSON".
    - Outputs: Connects to "Merge Rows".
    - Edge cases: Empty or null array.

  - **Split Emails**
    - Type: Split Out
    - Role: Splits the "output.Email" JSON array into individual items.
    - Configuration: Field to split out is "output.Email".
    - Inputs: From "Write JSON".
    - Outputs: Connects to "Merge Rows".
    - Edge cases: Mismatch in array lengths compared to companies.

  - **Merge Rows**
    - Type: Merge
    - Role: Combines company and email items by position into single records.
    - Configuration:
      - Mode: Combine
      - Combine by: Position
    - Inputs: From "Split Companies" (main input 0) and "Split Emails" (main input 1).
    - Outputs: Connects to "Send Leads to Google Sheets".
    - Edge cases:
      - Arrays of different lengths causing misaligned merging.

#### 2.5 Data Storage in Google Sheets

- **Overview:** Appends or updates leads into a Google Sheet, keyed by email, including area, company, and email fields.
- **Nodes Involved:**
  - Send Leads to Google Sheets
- **Node Details:**

  - **Send Leads to Google Sheets**
    - Type: Google Sheets
    - Role: Appends new leads or updates existing leads in Google Sheets.
    - Configuration:
      - Spreadsheet ID and Worksheet as before.
      - Columns mapped: Area, Email, Company.
      - Matching column: Email (to update existing rows).
      - Operation: Append or update.
    - Credentials: Google Sheets OAuth2.
    - Inputs: From "Merge Rows".
    - Outputs: None (end node).
    - Edge cases:
      - Auth errors.
      - Conflicts if multiple leads share email.
      - Sheet quota limits.

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                            | Input Node(s)             | Output Node(s)                   | Sticky Note                                                                                                         |
|---------------------------|--------------------------------------|--------------------------------------------|---------------------------|---------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Start Workflow            | Manual Trigger                       | Entry point to start workflow               | None                      | Set Location                   |                                                                                                                     |
| Set Location              | Set                                 | Sets target research area                   | Start Workflow            | Get Current Leads               |                                                                                                                     |
| Get Current Leads         | Google Sheets                       | Retrieves existing leads filtered by area  | Set Location              | Combine Into One                |                                                                                                                     |
| Combine Into One          | Aggregate                           | Aggregates retrieved leads into one object | Get Current Leads         | Set as text                    |                                                                                                                     |
| Set as text               | Set                                 | Prepares variables for downstream nodes     | Combine Into One          | Research Leads                 |                                                                                                                     |
| Research Leads            | Perplexity API                      | Queries Perplexity AI for new leads         | Set as text               | Write JSON                    |                                                                                                                     |
| OpenAI Chat Model         | Langchain OpenAI Chat Model         | Processes raw lead data with GPT model      | Research Leads            | Write JSON (AI languageModel)  |                                                                                                                     |
| Structured Output Parser  | Langchain Output Parser Structured  | Parses GPT output into JSON format           | Write JSON (AI outputParser) | Write JSON                    |                                                                                                                     |
| Write JSON                | Langchain Agent                    | Converts chat output into structured JSON   | Research Leads, OpenAI Chat Model, Structured Output Parser | Split Companies, Split Emails |                                                                                                                     |
| Split Companies           | Split Out                          | Splits company array to individual items    | Write JSON                | Merge Rows                    |                                                                                                                     |
| Split Emails              | Split Out                          | Splits email array to individual items      | Write JSON                | Merge Rows                    |                                                                                                                     |
| Merge Rows                | Merge                             | Combines company and email items by position | Split Companies, Split Emails | Send Leads to Google Sheets  |                                                                                                                     |
| Send Leads to Google Sheets | Google Sheets                     | Appends or updates leads in Google Sheets   | Merge Rows                | None                          |                                                                                                                     |
| Sticky Note59             | Sticky Note                       | Workflow purpose and summary                 | None                      | None                          | Research and log new business leads into Google Sheets (Perplexity + OpenAI + Google Sheets).                       |
| Sticky Note8              | Sticky Note                       | Setup instructions overview                   | None                      | None                          | Setup instructions including Google Sheets OAuth2, Perplexity API key, OpenAI API key, and example sheet link.      |
| Sticky Note71             | Sticky Note                       | Detailed Perplexity API credential setup     | None                      | None                          | Step-by-step instructions for setting up Perplexity API credentials in n8n.                                        |
| Sticky Note72             | Sticky Note                       | Detailed Google Sheets OAuth2 setup          | None                      | None                          | Steps to connect Google Sheets OAuth2 and select spreadsheet/worksheet.                                            |
| Sticky Note69             | Sticky Note                       | OpenAI API credential setup instructions     | None                      | None                          | Instructions for configuring OpenAI API credentials and selecting vision-capable model in OpenAI Chat Model node. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Name: Start Workflow
   - Purpose: To manually start the workflow.
   - No parameters.

2. **Create Set Node**
   - Name: Set Location
   - Purpose: Define the target research area.
   - Parameters:
     - Add field: Name = "Area", Type = String, Value = "Hershey PA" (or desired location).
   - Connect input from Start Workflow.

3. **Create Google Sheets Node**
   - Name: Get Current Leads
   - Purpose: Retrieve existing leads filtered by Area.
   - Parameters:
     - Operation: Read rows.
     - Spreadsheet ID: Provide your Google Sheet ID (example: "1MnaU8hSi8PleDNVcNnyJ5CgmDYJSUTsr7X5HIwa-MLk").
     - Sheet Name: "Sheet1" (or your worksheet).
     - Filters: Lookup rows where "Area" equals `={{ $json.Area }}` from Set Location.
   - Credential: Google Sheets OAuth2 (set up with your Google account).
   - Connect input from Set Location.

4. **Create Aggregate Node**
   - Name: Combine Into One
   - Purpose: Aggregate all rows into one object.
   - Parameters:
     - Aggregate: aggregateAllItemData.
     - Include binaries: true.
   - Connect input from Get Current Leads.

5. **Create Set Node**
   - Name: Set as text
   - Purpose: Prepare variables for Perplexity query.
   - Parameters:
     - Add field: "Search Area" = `={{ $('Set Location').item.json.Area }}`
     - Add field: "data" = `={{ $json.data }}`
   - Connect input from Combine Into One.

6. **Create Perplexity Node**
   - Name: Research Leads
   - Purpose: Query Perplexity AI for new leads.
   - Parameters:
     - Model: "sonar"
     - Messages:
       - System: "Research 20 coffee shops near the area. Output only the company name and the email address for each. Do not include businesses already scraped."
       - User: 
         ```
         Area:  {{ $json['Search Area'] }}
         Already Scraped: {{ $json.data }}
         ```
     - Simplify: true.
   - Credential: Perplexity API (configured with your API key).
   - Connect input from Set as text.

7. **Create Langchain OpenAI Chat Model Node**
   - Name: OpenAI Chat Model
   - Purpose: Use GPT (gpt-4.1-mini) to process lead data.
   - Parameters:
     - Model: "gpt-4.1-mini"
     - Options: defaults.
   - Credential: OpenAI API.
   - Connect input from Research Leads (as AI languageModel).

8. **Create Langchain Structured Output Parser Node**
   - Name: Structured Output Parser
   - Purpose: Parse OpenAI output into structured JSON.
   - Parameters:
     - JSON schema example:
       ```json
       {
         "Company": ["company1", "company2"],
         "Email": ["email1", "email2"]
       }
       ```
   - Connect input from Write JSON AI outputParser.

9. **Create Langchain Agent Node**
   - Name: Write JSON
   - Purpose: Convert AI message to JSON with companies and emails.
   - Parameters:
     - Text: `={{ $json.message }}`
     - System Message: Instructions to convert to JSON with company and email arrays.
     - Prompt Type: define.
     - Enable output parser.
   - Connect inputs from Research Leads (main), OpenAI Chat Model (ai_languageModel), Structured Output Parser (ai_outputParser).

10. **Create Split Out Node**
    - Name: Split Companies
    - Purpose: Split company array into individual items.
    - Parameters:
      - Field to split out: "output.Company"
    - Connect input from Write JSON.

11. **Create Split Out Node**
    - Name: Split Emails
    - Purpose: Split email array into individual items.
    - Parameters:
      - Field to split out: "output.Email"
    - Connect input from Write JSON.

12. **Create Merge Node**
    - Name: Merge Rows
    - Purpose: Combine company and email items by position.
    - Parameters:
      - Mode: combine.
      - Combine by: position.
    - Connect inputs from Split Companies (main input 0) and Split Emails (main input 1).

13. **Create Google Sheets Node**
    - Name: Send Leads to Google Sheets
    - Purpose: Append or update leads in Google Sheets.
    - Parameters:
      - Operation: Append or update.
      - Spreadsheet ID: Same as Get Current Leads.
      - Sheet Name: Same worksheet.
      - Columns:
        - Area: `={{ $('Set as text').item.json['Search Area'] }}`
        - Email: `={{ $json['output.Email'] }}`
        - Company: `={{ $json['output.Company'] }}`
      - Matching columns: Email (to prevent duplicates).
    - Credential: Google Sheets OAuth2.
    - Connect input from Merge Rows.

14. **Verify Credentials Setup**
    - Google Sheets OAuth2 configured with access to target spreadsheet.
    - Perplexity API key credential configured in n8n.
    - OpenAI API key credential configured in n8n.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                      | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Research and log new business leads into Google Sheets (Perplexity + OpenAI + Google Sheets). Automatically research new leads in your target area, structure the results with AI, and append them into Google Sheets — all orchestrated in n8n. | Workflow purpose summary (Sticky Note59)                                                          |
| Setup instructions for Google Sheets OAuth2, Perplexity API key, and OpenAI API key, including example Google Sheet link.                                                                                                                      | Setup instructions (Sticky Note8) https://docs.google.com/spreadsheets/d/1MnaU8hSi8PleDNVcNnyJ5CgmDYJSUTsr7X5HIwa-MLk/edit#gid=0 |
| Perplexity API credential setup instructions, including links to account and getting started guide.                                                                                                                                                | Sticky Note71: https://www.perplexity.ai/account and https://docs.perplexity.ai/guides/getting-started |
| Google Sheets OAuth2 detailed connection steps and example spreadsheet link.                                                                                                                                                                       | Sticky Note72: https://docs.google.com/spreadsheets/d/1MnaU8hSi8PleDNVcNnyJ5CgmDYJSUTsr7X5HIwa-MLk/edit#gid=0 |
| OpenAI API credential setup instructions highlighting the need for a vision-capable chat model such as gpt-4o-mini or gpt-4o.                                                                                                                    | Sticky Note69                                                                                     |
| Contact and author info: Robert Breen - rbreen@ynteractive.com - https://www.linkedin.com/in/robert-breen-29429625/ - https://ynteractive.com                                                                                                   | From Sticky Note8                                                                                  |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.