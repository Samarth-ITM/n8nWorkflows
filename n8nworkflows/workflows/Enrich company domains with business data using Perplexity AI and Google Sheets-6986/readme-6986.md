Enrich company domains with business data using Perplexity AI and Google Sheets

https://n8nworkflows.xyz/workflows/enrich-company-domains-with-business-data-using-perplexity-ai-and-google-sheets-6986


# Enrich company domains with business data using Perplexity AI and Google Sheets

### 1. Workflow Overview

This workflow automates the enrichment of company domain data by fetching detailed business information using Perplexity AI and updating a Google Sheets document accordingly. Its primary use case is to enhance datasets containing company domains with verified German business addresses and additional corporate data, facilitating better analysis or CRM enrichment.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggering the workflow manually and retrieving unprocessed company domains from Google Sheets.
- **1.2 Batch Processing:** Splitting domains into manageable batches (default size: 10) to optimize API usage and cost.
- **1.3 AI Data Enrichment:** Sending batch requests to Perplexity AI to fetch structured company business data focused on German addresses.
- **1.4 Data Parsing:** Extracting and validating the JSON-formatted response from the AI.
- **1.5 Data Persistence:** Updating Google Sheets with the enriched data and marking entries as processed to prevent duplication.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow manually and reads unprocessed company domains from a Google Sheets spreadsheet to prepare them for enrichment.

**Nodes Involved:**  
- Manual Trigger  
- Fetch Unprocessed Domains

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node  
  - Role: Starts the workflow on-demand  
  - Configuration: No parameters; simple manual activation  
  - Connections: Outputs to "Fetch Unprocessed Domains"  
  - Failures: None expected unless manual activation is missed  

- **Fetch Unprocessed Domains**  
  - Type: Google Sheets node  
  - Role: Reads rows where the "processed" column is empty (unprocessed domains)  
  - Configuration:  
    - Sheet name: "Data" tab  
    - Document ID: Google Sheets file containing company domains  
    - Filter: Rows where "processed" equals "" (empty)  
    - Credentials: Google Sheets OAuth2 account  
  - Connections: Outputs to "Batch Process Domains"  
  - Failures/Potential Issues:  
    - Authentication errors if OAuth2 token is invalid or expired  
    - Empty result sets if all domains processed or filter misconfigured  
    - API quota limits from Google Sheets  

#### 2.2 Batch Processing

**Overview:**  
Splits the domain list into smaller batches of 10 to optimize API calls to Perplexity AI and control costs.

**Nodes Involved:**  
- Batch Process Domains

**Node Details:**

- **Batch Process Domains**  
  - Type: SplitInBatches node  
  - Role: Processes domains in batches of 10  
  - Configuration: Batch size set to 10 (adjustable)  
  - Input: Receives all unprocessed domains from "Fetch Unprocessed Domains"  
  - Outputs:  
    - Main output: Continues processing with the current batch  
    - Empty output: Ends batch processing cycle  
  - Connections:  
    - Main output (batch data) to "Perplexity AI Research"  
    - Empty output back to "Fetch Unprocessed Domains" for next batch  
  - Failures/Potential Issues:  
    - Improper batch size leading to API rate limits or excessive cost  
    - Handling last batch with fewer than 10 items correctly  
    - Workflow resumption logic depends on processed flag  

#### 2.3 AI Data Enrichment

**Overview:**  
Sends a batch of domains to Perplexity AI for enrichment, requesting structured business data focused on German addresses.

**Nodes Involved:**  
- Perplexity AI Research

**Node Details:**

- **Perplexity AI Research**  
  - Type: HTTP Request node  
  - Role: Calls Perplexity AI chat completion API with a prompt to fetch company details  
  - Configuration:  
    - URL: `https://api.perplexity.ai/chat/completions`  
    - Method: POST  
    - Request Body (JSON):  
      - Model: "sonar"  
      - Messages: User prompt requesting German addresses and business info for 10 companies, dynamically injecting domain names from the batch  
      - Response format: JSON Schema specifying expected structured fields including domain, company name, address components, phone, employees, revenue, industry, LinkedIn URL, source URL  
    - Authentication: Uses stored Perplexity API credentials  
  - Input: Receives batch of domains from "Batch Process Domains"  
  - Output: AI response JSON to "Parse AI Response"  
  - Failures/Potential Issues:  
    - API authentication errors (invalid or missing API key)  
    - Rate limiting or quota exceeded errors from Perplexity AI  
    - Response format changes or schema mismatches  
    - Network timeouts or connectivity issues  

#### 2.4 Data Parsing

**Overview:**  
Parses the AI response text, extracts JSON data, validates it, and splits it into individual company records for further processing.

**Nodes Involved:**  
- Parse AI Response

**Node Details:**

- **Parse AI Response**  
  - Type: Code node (JavaScript)  
  - Role: Extracts JSON from AI response, parses it, and outputs array of company objects  
  - Key Logic:  
    - Extracts JSON block from ```json ... ``` markdown if present  
    - Attempts multiple JSON.parse calls to handle nested or stringified JSON  
    - Validates that result is an array of company objects  
    - On failure, outputs detailed error or warning for diagnostics  
  - Input: Raw AI response from "Perplexity AI Research"  
  - Output: Array of JSON objects, one per company, to "Save Enriched Data"  
  - Failures/Potential Issues:  
    - Parsing errors due to malformed or unexpected AI responses  
    - Missing expected keys or structure changes in AI output  
    - Outputting error info for monitoring and troubleshooting  

#### 2.5 Data Persistence

**Overview:**  
Updates the original Google Sheets document with the enriched company data and marks domains as processed to avoid reprocessing.

**Nodes Involved:**  
- Save Enriched Data  
- Batch Process Domains (for looping)

**Node Details:**

- **Save Enriched Data**  
  - Type: Google Sheets node  
  - Role: Updates rows matching company domain with enriched fields and sets "processed" to "true"  
  - Configuration:  
    - Sheet name: "Data" tab  
    - Document ID: Same Google Sheets file as input  
    - Matching columns: "domain" used to match existing rows for update  
    - Fields updated: company, address, city, state, postal_code, country, phone, employees, revenue, industry, LinkedIn URL, source URL, processed flag  
    - Phone number sanitized to prefix '+' with "'" to preserve format in Sheets  
    - Credentials: Google Sheets OAuth2 account  
  - Input: Individual company records from "Parse AI Response"  
  - Output: Loops back to "Batch Process Domains" to trigger next batch processing  
  - Failures/Potential Issues:  
    - Update conflicts if sheet modified concurrently  
    - Authentication errors for Google Sheets OAuth2  
    - Data type mismatches or invalid values in fields  
    - Partial updates if some records fail  

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                     | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                            |
|-------------------------|-------------------------|-----------------------------------|------------------------|-------------------------|------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger          | Trigger                 | Manual workflow start              | —                      | Fetch Unprocessed Domains |                                                                                                                        |
| Fetch Unprocessed Domains| Google Sheets           | Reads unprocessed domains from sheet| Manual Trigger          | Batch Process Domains    |                                                                                                                        |
| Batch Process Domains   | SplitInBatches          | Splits domains into batches of 10  | Fetch Unprocessed Domains, Save Enriched Data | Perplexity AI Research (batch), (empty output) |                                                                                                                        |
| Perplexity AI Research  | HTTP Request            | Calls Perplexity AI for enrichment | Batch Process Domains   | Parse AI Response        |                                                                                                                        |
| Parse AI Response       | Code                    | Parses AI JSON response             | Perplexity AI Research  | Save Enriched Data       |                                                                                                                        |
| Save Enriched Data      | Google Sheets           | Updates sheet with enriched data   | Parse AI Response       | Batch Process Domains    |                                                                                                                        |
| Sticky Note             | Sticky Note             | Describes workflow overview        | —                      | —                       | 🏢 COMPANY DATA ENRICHMENT SYSTEM - Automatically enriches company domains with detailed business data using Perplexity AI |
| Sticky Note1            | Sticky Note             | Setup checklist                    | —                      | —                       | 🚀 SETUP CHECKLIST - Google Sheets template, credentials, and batch size info                                          |
| Sticky Note2            | Sticky Note             | Output data & error handling notes | —                      | —                       | 📊 OUTPUT DATA FIELDS - Enriched data fields, error handling, and monitoring instructions                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add "Manual Trigger" node to start workflow manually.

2. **Add Google Sheets Node to Fetch Domains**  
   - Node Type: Google Sheets  
   - Operation: Read rows  
   - Spreadsheet: Use your Google Sheets document containing the company domains  
   - Sheet Name: "Data"  
   - Filter: Set filter to retrieve rows where the "processed" column is empty (`processed = ""`)  
   - Credentials: Configure Google Sheets OAuth2 credentials  
   - Connect Manual Trigger output to this node.

3. **Add SplitInBatches Node**  
   - Node Type: SplitInBatches  
   - Batch Size: Set to 10 (adjustable based on API limits and cost)  
   - Connect output of Google Sheets fetch node to SplitInBatches input.

4. **Add HTTP Request Node for Perplexity AI**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.perplexity.ai/chat/completions`  
   - Body Content Type: JSON  
   - Body: Configure JSON body with:  
     - model: "sonar"  
     - messages: User prompt requesting German addresses and business info for the 10 domains from batch, using an expression to inject domain list dynamically  
     - response_format: JSON Schema specifying the expected structured response (see section 2.3 for schema details)  
   - Authentication: Use Perplexity AI API credentials (API key configured in n8n)  
   - Connect SplitInBatches output to this node.

5. **Add Code Node to Parse AI Response**  
   - Node Type: Code (JavaScript)  
   - Paste the JavaScript code that:  
     - Extracts JSON inside code blocks in AI response  
     - Parses JSON safely, handling nested stringified JSON  
     - Returns an array of company objects  
     - Returns error info if parsing fails  
   - Connect HTTP Request output to this node.

6. **Add Google Sheets Node to Save Enriched Data**  
   - Node Type: Google Sheets  
   - Operation: Update rows  
   - Spreadsheet: Same as fetch node  
   - Sheet Name: "Data"  
   - Matching Columns: "domain" to identify rows to update  
   - Map columns: Map each enriched field (company, address, city, state, postal_code, country, phone, employees, revenue, industry, linkedin URL, source URL) to corresponding JSON fields from code node  
   - Set "processed" column to "true" to mark processed  
   - Phone field: Replace + with "'+" to preserve international format in Sheets  
   - Credentials: Google Sheets OAuth2 credentials  
   - Connect Code node output to this node.

7. **Connect Save Node Back to SplitInBatches Node**  
   - Connect output of Google Sheets update node to the empty output of SplitInBatches node to trigger processing of next batch.

8. **Add Sticky Notes for Documentation (Optional)**  
   - Add sticky notes describing workflow overview, setup checklist, and output data fields for clarity.

9. **Configure Credentials**  
   - Google Sheets OAuth2: Set up OAuth2 credentials with read/write access to the spreadsheet.  
   - Perplexity API: Add API key credential for Perplexity AI.

10. **Test and Run Workflow**  
    - Trigger workflow manually.  
    - Monitor logs for errors, check Google Sheets for updated enriched data.  
    - Adjust batch size or prompts if needed for performance or cost optimization.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                                            |
|--------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| 🏢 COMPANY DATA ENRICHMENT SYSTEM - Automatically enriches company domains with detailed business data using Perplexity AI | Workflow overview sticky note                                                                                              |
| 🚀 SETUP CHECKLIST - Make a copy of the Google Sheets template before use. Required columns: domain, processed. Update sheet ID in nodes. | [Google Sheets Template](https://docs.google.com/spreadsheets/d/1bdK8xskt-qfLlDwdzolM0zFyo9KxZ-HHpTVxcEw3ZMY/edit?usp=sharing) |
| 📊 OUTPUT DATA FIELDS - Enriched data includes company name, address, phone, employees, revenue, industry, LinkedIn URL, source URL, and processed flag. Error handling and monitoring instructions included. |                                                                                                                            |
| Cost per 10 domains is approximately $0.005 using Perplexity AI. Runtime ~2-3 minutes per batch.  | Monitoring recommended for cost and performance                                                                              |
| Focus is on German addresses but prompt can be modified for other geographies or HQ addresses.    | Customization point                                                                                                         |

---

This documentation provides a complete, structured understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.