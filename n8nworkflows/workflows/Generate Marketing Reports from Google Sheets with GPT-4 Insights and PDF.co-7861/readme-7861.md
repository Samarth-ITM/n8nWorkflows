Generate Marketing Reports from Google Sheets with GPT-4 Insights and PDF.co

https://n8nworkflows.xyz/workflows/generate-marketing-reports-from-google-sheets-with-gpt-4-insights-and-pdf-co-7861


# Generate Marketing Reports from Google Sheets with GPT-4 Insights and PDF.co

### 1. Workflow Overview

This workflow automates the generation of marketing spend reports by pulling data from Google Sheets, aggregating spend metrics by channel, generating an AI-written summary using GPT-4, and producing a professionally formatted PDF report via PDF.co’s HTML template system. It targets marketing analysts and teams who want to automate daily or periodic reporting without manual data processing or formatting.

The logical blocks are:

- **1.1 Input Trigger & Data Retrieval:** Manual trigger initiates the workflow, which fetches marketing spend data from Google Sheets.
- **1.2 Data Aggregation:** Summarizes total spend and spend by each marketing channel.
- **1.3 AI Summary Generation:** Uses GPT-4 via Langchain nodes to create a concise, structured summary of the marketing spend data.
- **1.4 Data Preparation for PDF:** Combines aggregated data and AI summary, formats it for the PDF template.
- **1.5 PDF Generation:** Sends formatted data to PDF.co to generate a PDF report using a custom Mustache/HTML template.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger & Data Retrieval

**Overview:**  
This block starts the workflow manually and retrieves raw marketing data from a specified Google Sheets document and worksheet.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get Marketing Data (Google Sheets)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to initiate workflow execution manually.  
  - Configuration: No parameters, simply triggered by user action.  
  - Input: None  
  - Output: Triggers the next node(s).  
  - Edge cases: None specific, but workflow won’t start without user action.

- **Get Marketing Data**  
  - Type: Google Sheets  
  - Role: Reads marketing data from Google Sheets.  
  - Configuration:  
    - Spreadsheet ID: `1UDWt0-Z9fHqwnSNfU3vvhSoYCFG6EG3E-ZewJC_CLq4` (sample data template).  
    - Sheet Name: `365710158` (Google Sheet tab ID).  
  - Credentials: Google Sheets OAuth2 credential configured with access to the target spreadsheet.  
  - Input: Trigger from Manual Trigger node.  
  - Output: Raw rows of marketing data (including fields like Channel, Spend ($)).  
  - Edge cases:  
    - Authentication failure if OAuth2 credential expired or revoked.  
    - Empty or malformed spreadsheet data.  
    - Network timeout or API quota exceeded.

---

#### 1.2 Data Aggregation

**Overview:**  
Processes raw data to calculate total marketing spend and break down spend by each channel.

**Nodes Involved:**  
- Sum Spend  
- Sum Spend by Channel  
- Covert to 1 row (Aggregate)  

**Node Details:**

- **Sum Spend**  
  - Type: Summarize  
  - Role: Calculates total sum of the "Spend ($)" column across all data.  
  - Configuration: Aggregation set to sum on the "Spend ($)" field.  
  - Input: Marketing data from Google Sheets node.  
  - Output: Single summarized row with total spend.  
  - Edge cases:  
    - Missing or non-numeric "Spend ($)" values may cause incorrect sums.

- **Sum Spend by Channel**  
  - Type: Summarize  
  - Role: Calculates sum of "Spend ($)" grouped by "Channel".  
  - Configuration: Split by "Channel" and sum on "Spend ($)".  
  - Input: Marketing data from Google Sheets node.  
  - Output: Rows summarizing spend per channel.  
  - Edge cases:  
    - Missing "Channel" field or inconsistent channel names cause grouping errors.

- **Covert to 1 row**  
  - Type: Aggregate  
  - Role: Aggregates all items into a single JSON object to facilitate combined processing downstream.  
  - Configuration: Aggregate all item data into one row.  
  - Input: Output of Sum Spend by Channel.  
  - Output: Single JSON object containing summarized spend by channel.  
  - Edge cases: Empty input array results in empty aggregated data.

---

#### 1.3 AI Summary Generation

**Overview:**  
Generates a structured summary of the marketing spend data using OpenAI GPT-4 implemented via Langchain nodes.

**Nodes Involved:**  
- OpenAI Chat Model  
- Write Summary (Langchain Agent)  
- Structured Output Parser  

**Node Details:**

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Calls GPT-4.1-mini model to generate AI output.  
  - Configuration: Model set to "gpt-4.1-mini", no additional options.  
  - Credentials: OpenAI API key configured.  
  - Input: Prompt text from Write Summary node (via Langchain agent).  
  - Output: AI-generated response JSON.  
  - Edge cases:  
    - API key invalid or quota exceeded.  
    - Model response timeout or errors.  
    - Unexpected response format.

- **Write Summary**  
  - Type: Langchain Agent  
  - Role: Defines the AI prompt and processing logic.  
  - Configuration:  
    - System message instructs GPT-4 to write a 4-sentence daily update summary of marketing data.  
    - Output format enforced as JSON with a "summary" field.  
    - Input text is the aggregated data JSON.  
  - Input: Aggregated data from Covert to 1 row node.  
  - Output: AI response JSON, parsed by Structured Output Parser.  
  - Edge cases:  
    - Expression failures if input data malformed.  
    - Parser failures if AI output is not valid JSON.

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses AI response into structured JSON using provided schema example.  
  - Configuration: JSON schema example expects a "summary" string.  
  - Input: AI response from OpenAI Chat Model.  
  - Output: Parsed summary JSON.  
  - Edge cases: Parsing errors if AI output does not match schema.

---

#### 1.4 Data Preparation for PDF

**Overview:**  
Combines aggregated total spend, channel spend data, and AI summary into a single structured JSON object formatted for the PDF template.

**Nodes Involved:**  
- Combine All (Merge)  
- Convert to PDF Upload (Code Node)

**Node Details:**

- **Combine All**  
  - Type: Merge  
  - Role: Combines three inputs by position into one output array.  
  - Configuration: Combine mode by position, expecting 3 inputs: total spend, channel spend data, and AI summary.  
  - Input:  
    - From Sum Spend (total spend)  
    - From Covert to 1 row (channel spend data)  
    - From Write Summary (AI summary)  
  - Output: Combined data array with all relevant info.  
  - Edge cases: Mismatched input lengths or missing inputs cause incomplete data.

- **Convert to PDF Upload**  
  - Type: Code (JavaScript)  
  - Role: Transforms combined data into the exact JSON structure required by the PDF.co Mustache HTML template.  
  - Configuration:  
    - Formats numeric spend as USD currency strings.  
    - Extracts total spend, channels with formatted spend, and AI summary text.  
    - Adds generation date in ISO format (YYYY-MM-DD).  
  - Input: Combined data from Merge node.  
  - Output: Single JSON object ready for PDF generation.  
  - Edge cases:  
    - Empty input data returns error JSON.  
    - Missing or malformed fields handled gracefully with defaults.

---

#### 1.5 PDF Generation

**Overview:**  
Sends the formatted JSON data to PDF.co to generate a PDF report using a predefined HTML template.

**Nodes Involved:**  
- Create PDF (PDF.co API)

**Node Details:**

- **Create PDF**  
  - Type: PDF.co API  
  - Role: Converts JSON data and HTML/Mustache template into a PDF file.  
  - Configuration:  
    - Operation: "URL/HTML to PDF"  
    - Template ID: "12011" (user must replace with their PDF.co template ID)  
    - Convert Type: "htmlTemplateToPDF"  
    - Template Data: JSON stringified input from previous node.  
  - Credentials: PDF.co API key configured.  
  - Input: JSON data from Convert to PDF Upload node.  
  - Output: PDF file data and metadata.  
  - Edge cases:  
    - API key invalid or quota exceeded errors.  
    - Template ID not found or invalid template.  
    - Network or PDF.co service errors.

---

### 3. Summary Table

| Node Name                    | Node Type                          | Functional Role                         | Input Node(s)                         | Output Node(s)                  | Sticky Note                                                                                                      |
|------------------------------|----------------------------------|---------------------------------------|-------------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Workflow start trigger                 | None                                | Get Marketing Data              |                                                                                                                 |
| Get Marketing Data            | Google Sheets                    | Fetches marketing data from Sheets    | When clicking ‘Execute workflow’    | Sum Spend, Sum Spend by Channel |                                                                                                                 |
| Sum Spend                    | Summarize                       | Calculates total marketing spend      | Get Marketing Data                  | Combine All                    |                                                                                                                 |
| Sum Spend by Channel          | Summarize                       | Calculates spend per channel           | Get Marketing Data                  | Covert to 1 row                |                                                                                                                 |
| Covert to 1 row              | Aggregate                       | Aggregates summarized channel data into one object | Sum Spend by Channel                | Combine All, Write Summary     |                                                                                                                 |
| OpenAI Chat Model             | Langchain OpenAI Chat Model     | Calls GPT-4 to generate summary       | Write Summary                      | Write Summary                  |                                                                                                                 |
| Write Summary                | Langchain Agent                 | Defines AI prompt for summary generation | Covert to 1 row, Structured Output Parser | Combine All                    |                                                                                                                 |
| Structured Output Parser      | Langchain Output Parser         | Parses AI response into structured JSON | OpenAI Chat Model                  | Write Summary                  |                                                                                                                 |
| Combine All                  | Merge                          | Combines total spend, channel data, and AI summary into one dataset | Sum Spend, Covert to 1 row, Write Summary | Convert to PDF Upload          |                                                                                                                 |
| Convert to PDF Upload         | Code (JavaScript)               | Formats data for PDF template         | Combine All                       | Create PDF                    |                                                                                                                 |
| Create PDF                   | PDF.co API                     | Generates PDF report using HTML template | Convert to PDF Upload               | None                          | Sticky Note59 (Instructions for PDF.co setup and template creation)                                            |
| Sticky Note53                | Sticky Note                    | Describes workflow purpose and overview | None                              | None                          | # 📊 Marketing Spend Report → Google Sheets + PDF\n\nThis workflow pulls **marketing data from Google Sheets**, aggregates spend by channel, generates an **AI-written summary**, and outputs a formatted **PDF report** using a custom HTML template on **PDF.co**.  |
| Sticky Note1                 | Sticky Note                    | Setup instructions for Google Sheets and PDF.co credentials | None                              | None                          | ## ⚙️ Setup Instructions\n\n### 1️⃣ Prepare Your Google Sheet ... [includes detailed user instructions with links] |
| Sticky Note59                | Sticky Note                    | PDF.co account and template setup instructions | None                              | None                          | ### 2️⃣ Connect PDF.co\n1. Create a free account at [PDF.co](https://pdf.co/) ... [detailed steps]                |
| Sticky Note60                | Sticky Note                    | Google Sheets setup instructions      | None                              | None                          | ### 1️⃣ Prepare Your Google Sheet  \n- Copy this template into your Google Drive: [Sample Marketing Data](https://docs.google.com/spreadsheets/d/1UDWt0-Z9fHqwnSNfU3vvhSoYCFG6EG3E-ZewJC_CLq4/edit?gid=365710158#gid=365710158)  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node named "When clicking ‘Execute workflow’".  
   - No parameters needed.

2. **Add Google Sheets Node**  
   - Add "Google Sheets" node named "Get Marketing Data".  
   - Configure:  
     - Set Spreadsheet ID to `1UDWt0-Z9fHqwnSNfU3vvhSoYCFG6EG3E-ZewJC_CLq4` (or your own).  
     - Set Sheet name or tab ID to `365710158`.  
   - Set credentials to a Google Sheets OAuth2 credential with access rights.  
   - Connect output of Manual Trigger to this node.

3. **Add Summarize Node for Total Spend**  
   - Add "Summarize" node named "Sum Spend".  
   - Configure to sum the "Spend ($)" field across all rows.  
   - Connect input from "Get Marketing Data".

4. **Add Summarize Node for Spend by Channel**  
   - Add "Summarize" node named "Sum Spend by Channel".  
   - Configure to split by "Channel" and sum "Spend ($)".  
   - Connect input from "Get Marketing Data".

5. **Add Aggregate Node to Convert to Single Row**  
   - Add "Aggregate" node named "Covert to 1 row".  
   - Set to aggregate all item data into a single JSON object.  
   - Connect input from "Sum Spend by Channel".

6. **Add Langchain Agent Node for Summary**  
   - Add "Langchain Agent" node named "Write Summary".  
   - Configure:  
     - Text input expression: `={{ $json.data }}` (from aggregate node).  
     - System message: instruct it to write a 4-sentence summary of marketing data, outputting JSON with a "summary" field.  
     - Enable output parser.  
   - Connect input from "Covert to 1 row".

7. **Add Langchain OpenAI Chat Model Node**  
   - Add "Langchain OpenAI Chat Model" node named "OpenAI Chat Model".  
   - Set model to "gpt-4.1-mini".  
   - Configure OpenAI API credentials.  
   - Connect output from "Write Summary" to this node’s AI language model input.

8. **Add Langchain Structured Output Parser Node**  
   - Add "Langchain Structured Output Parser" node named "Structured Output Parser".  
   - Provide JSON schema example: `{"summary": "summary"}`.  
   - Connect AI output from "OpenAI Chat Model" to this node.  
   - Connect parser output back to "Write Summary" node’s output parser input.

9. **Add Merge Node to Combine Data**  
   - Add "Merge" node named "Combine All".  
   - Configure mode: "Combine" by position with 3 inputs.  
   - Connect:  
     - Input 1 from "Sum Spend" (total spend).  
     - Input 2 from "Covert to 1 row" (channel spend).  
     - Input 3 from "Write Summary" (AI summary).

10. **Add Code Node for PDF Data Preparation**  
    - Add "Code" node named "Convert to PDF Upload".  
    - Paste provided JavaScript code that formats spend as USD, extracts channels and summary, and adds date.  
    - Connect input from "Combine All".

11. **Add PDF.co Node to Create PDF**  
    - Add "PDF.co API" node named "Create PDF".  
    - Configure:  
      - Operation: "URL/HTML to PDF"  
      - Convert Type: "htmlTemplateToPDF"  
      - Template ID: replace `"12011"` with your PDF.co HTML template ID.  
      - Template Data: set expression to `{{ JSON.stringify($json) }}`.  
    - Configure PDF.co API credentials with your API key.  
    - Connect input from "Convert to PDF Upload".

12. **Test the workflow**  
    - Fill your Google Sheet with marketing data in the specified format.  
    - Ensure credentials for Google Sheets, OpenAI, and PDF.co are valid and active.  
    - Execute the workflow manually and verify PDF output is generated.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| This workflow pulls **marketing data from Google Sheets**, aggregates spend by channel, generates an **AI-written summary**, and outputs a formatted **PDF report** using a custom HTML template on **PDF.co**.                       | Workflow overview noted in Sticky Note53                                                                                               |
| Prepare your Google Sheet by copying the sample template: [Sample Marketing Data](https://docs.google.com/spreadsheets/d/1UDWt0-Z9fHqwnSNfU3vvhSoYCFG6EG3E-ZewJC_CLq4/edit?gid=365710158#gid=365710158).                              | Setup instructions in Sticky Note1 and Sticky Note60                                                                                   |
| Create a free PDF.co account and generate an API key: https://pdf.co/                                                                                                                                                               | Setup instructions in Sticky Note1 and Sticky Note59                                                                                   |
| In PDF.co Dashboard, create a new Mustache HTML template for the report and save its Template ID. Replace the `templateId` in the workflow’s PDF node accordingly.                                                                  | Setup instructions in Sticky Note59                                                                                                    |
| For customization assistance (e.g., filtering campaigns, emailing reports), contact Robert Breen at robert@ynteractive.com or via LinkedIn [Robert Breen](https://www.linkedin.com/in/robert-breen-29429625/).                         | Contact info in Sticky Note1                                                                                                           |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, respecting all current content policies. It contains no illegal, offensive, or protected content and processes only legal and public data.