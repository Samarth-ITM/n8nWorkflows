Enrich Company Profiles with Explorium MCP and Claude AI for GTM Research

https://n8nworkflows.xyz/workflows/enrich-company-profiles-with-explorium-mcp-and-claude-ai-for-gtm-research-6064


# Enrich Company Profiles with Explorium MCP and Claude AI for GTM Research

---

### 1. Workflow Overview

This workflow, titled **"Enrich Company Profiles with Explorium MCP and Claude AI for GTM Research"**, automates the enrichment of company profiles using AI-powered web research. It targets users who need to augment company datasets (e.g., from Google Sheets) with detailed market and product information for go-to-market (GTM) strategies.

The workflow logically divides into these main blocks:

- **1.1 Input Acquisition:** Reads rows from a Google Sheet containing company identifiers (domain or name) to enrich.
- **1.2 Data Preparation and Batching:** Structures input data and processes it in manageable batches.
- **1.3 AI Research Execution:** Uses Explorium MCP and Anthropic's Claude AI to research company details, including website content retrieval and AI-based information extraction.
- **1.4 Output Parsing and Structuring:** Parses AI output into a structured JSON object with predefined fields.
- **1.5 Data Merging and Sheet Update:** Combines enriched data with original input and updates the Google Sheet rows accordingly.
- **1.6 Supporting Documentation:** Sticky notes provide usage instructions, prompts, and references.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Acquisition

- **Overview:**  
  This block fetches company data rows from Google Sheets to be processed.

- **Nodes Involved:**  
  - Get rows to enrich  
  - Input  

- **Node Details:**  

  - **Get rows to enrich**  
    - *Type:* Google Sheets  
    - *Role:* Reads data rows from a specified Google Sheet (document ID and sheet name set to "input" tab) containing company inputs.  
    - *Configuration:* No filters; reads full sheet. Uses OAuth2 credentials for authentication.  
    - *Inputs:* Triggered manually or from the workflow start node.  
    - *Outputs:* JSON objects including fields `input` (company identifier) and `row_number`.  
    - *Edge Cases:* Possible Google Sheets API errors (auth failures, rate limiting, invalid document ID). Empty or malformed rows may cause downstream issues.

  - **Input**  
    - *Type:* Set  
    - *Role:* Extracts and assigns key variables (`company_input`, `row_number`) from the incoming data for further use.  
    - *Configuration:* Sets `company_input` from `$json.input` and `row_number` from `$json.row_number`.  
    - *Inputs:* Connected from "Get rows to enrich" node.  
    - *Outputs:* Structured input JSON for batching and AI processing.  
    - *Edge Cases:* Empty or null inputs may propagate empty values.

---

#### 2.2 Data Preparation and Batching

- **Overview:**  
  Splits the input data into batches to control processing load and manage API rate limits.

- **Nodes Involved:**  
  - Loop Over Items  

- **Node Details:**  

  - **Loop Over Items**  
    - *Type:* SplitInBatches  
    - *Role:* Processes input data in batches (default batch size if not specified).  
    - *Configuration:* No specific batch size configured, uses defaults.  
    - *Inputs:* Receives structured inputs from "Input" node.  
    - *Outputs:* Outputs batches of company input for AI research.  
    - *Edge Cases:* If batch size is too large, may cause timeouts or API limits. Empty batches halt processing.

---

#### 2.3 AI Research Execution

- **Overview:**  
  This core block sends company inputs to Explorium MCP and Claude AI agents to research and gather company information, including web content retrieval.

- **Nodes Involved:**  
  - MCP Client  
  - Anthropic Chat Model  
  - Get website content (sub-workflow)  
  - AI company researcher  
  - Structured Output Parser  

- **Node Details:**  

  - **MCP Client**  
    - *Type:* LangChain MCP Client Tool  
    - *Role:* Connects to Explorium MCP server to perform web research and data retrieval.  
    - *Configuration:* SSE endpoint set to `mcp.explorium.ai/sse` with header authentication using configured credentials.  
    - *Inputs:* Receives company input from previous batch node.  
    - *Outputs:* Sends results to "AI company researcher" node as AI tool input.  
    - *Edge Cases:* Connection errors, authentication failures, or SSE stream interruptions possible.  
    - *Credentials:* HTTP Header Auth credential named "Explorium."

  - **Anthropic Chat Model**  
    - *Type:* LangChain AI Chat (Anthropic Claude 3.7 Sonnet)  
    - *Role:* Provides natural language processing and reasoning to analyze company data.  
    - *Configuration:* Model set to `claude-3-7-sonnet-20250219`. No additional options.  
    - *Inputs:* Receives prompts from "AI company researcher".  
    - *Outputs:* Processes AI language model response for the researcher node.  
    - *Edge Cases:* API quota limits, latency, or model errors.  
    - *Credentials:* Anthropic API key named "Anthropic account."

  - **Get website content (sub-workflow)**  
    - *Type:* LangChain Tool Workflow (sub-workflow)  
    - *Role:* Visits company website and extracts clean textual content for AI analysis.  
    - *Configuration:* Uses HTTP Request node to fetch URL, HTML node to extract `<html>` content excluding `<head>`. Cleans up text for use.  
    - *Inputs:* URL parameter from AI research flow.  
    - *Outputs:* Returns body text content of website.  
    - *Edge Cases:* Website unreachable, HTTP errors, or HTML structure changes.  
    - *Sub-workflow:* Defined inline with nodes "Execute Workflow Trigger", "Visit Website", "HTML".

  - **AI company researcher**  
    - *Type:* LangChain Agent (AI Researcher)  
    - *Role:* Orchestrates the AI prompt to fetch company details: domain, LinkedIn URL, market type, pricing plans, case study links, API availability, enterprise plans, free trial status, and integrations.  
    - *Configuration:* Custom prompt with instructions to return structured fields. Max 20 iterations. Uses output parser.  
    - *Inputs:* Receives company name/domain from batching node or AI tools (MCP, Anthropic).  
    - *Outputs:* Raw AI output JSON passed to "Structured Output Parser".  
    - *Edge Cases:* AI hallucination, incomplete or inaccurate data, prompt misinterpretation.

  - **Structured Output Parser**  
    - *Type:* LangChain Structured Output Parser  
    - *Role:* Validates and parses AI output JSON into a strict schema with fields: `case_study_link`, `domain`, `linkedinUrl`, `market`, `cheapest_plan`, `has_enterprise_plan`, `has_API`, `has_free_trial`, `integrations`.  
    - *Configuration:* Manual JSON schema defines accepted types and nullable fields.  
    - *Inputs:* Raw AI output from "AI company researcher".  
    - *Outputs:* Clean structured JSON for downstream processing.  
    - *Edge Cases:* Parsing errors if AI output deviates from schema, null or missing values.

---

#### 2.4 Output Structuring and Data Merging

- **Overview:**  
  This block restructures AI parsed output into individual fields, merges it with original input data, and prepares it for Google Sheets update.

- **Nodes Involved:**  
  - AI Researcher Output Data  
  - Merge data  
  - Google Sheets - Update Row with data  

- **Node Details:**  

  - **AI Researcher Output Data**  
    - *Type:* Set  
    - *Role:* Maps structured output JSON fields into individual variables for clarity and mapping.  
    - *Configuration:* Assigns variables like `domain`, `linkedinUrl`, `market`, `cheapest_plan`, `has_enterprise_plan`, `has_API`, `has_free_trial`, `integrations`, `case_study_link` from `$json.output`.  
    - *Inputs:* From "AI company researcher" node after parsing.  
    - *Outputs:* Passes structured data for merging.  
    - *Edge Cases:* Missing or null fields are assigned as-is; downstream nodes must handle nulls gracefully.

  - **Merge data**  
    - *Type:* Merge  
    - *Role:* Combines original input data (row number) with enriched AI output by position to maintain row integrity.  
    - *Configuration:* Mode set to "combine" with "mergeByPosition".  
    - *Inputs:* Two inputs: original data and AI enriched data.  
    - *Outputs:* Merged JSON for updating Google Sheets.  
    - *Edge Cases:* Position mismatch causing data misalignment.

  - **Google Sheets - Update Row with data**  
    - *Type:* Google Sheets  
    - *Role:* Updates the corresponding row in the Google Sheet with enriched company data and sets enrichment status to "done".  
    - *Configuration:*  
      - Document ID set to the target sheet (template provided).  
      - Sheet name set to "input".  
      - Matching rows by `row_number`.  
      - Updates columns: domain, market, linkedinUrl, integrations, cheapest_plan, has_free_trial, enrichment_status, has_entreprise_plan, last_case_study_link.  
    - *Credentials:* OAuth2 Google Sheets credential "Google Sheets account 2".  
    - *Inputs:* Merged data from "Merge data".  
    - *Outputs:* None (terminal output).  
    - *Edge Cases:* Google API errors, row not found, write conflicts, data type mismatches.

---

#### 2.5 Supporting Documentation and Notes

- **Overview:**  
  Provides user guidance, prompt explanations, instructions, and reference links through sticky notes.

- **Nodes Involved:**  
  - Sticky Note (Read Me)  
  - Sticky Note2 (Output Format Guidance)  
  - Sticky Note3 (AI Inquiry Prompt)  
  - Sticky Note4 (Explorium MCP Info)  
  - Sticky Note7 (Google Sheets Template Instructions)  

- **Node Details:**  

  - **Sticky Note**  
    - Contains detailed description of workflow purpose, AI capabilities (Google search and website content extraction), expected output fields, and link to an external video guide with instructions.  
    - Link: https://lempire.notion.site/AI-Web-research-with-n8n-a25aae3258d0423481a08bd102f16906  

  - **Sticky Note2**  
    - Reminds user to specify the output data format in the "Structured Output Parser" node.

  - **Sticky Note3**  
    - Explains that the AI prompt should specify what information is sought about the company.

  - **Sticky Note4**  
    - Notes that Explorium MCP server handles the research workload.

  - **Sticky Note7**  
    - Provides instructions for using the Google Sheets template, including a link to copy the template and guidance on updating node configurations for URL and sheet access.  
    - Link: https://docs.google.com/spreadsheets/d/1vR6s2nlTwu01v3GP7wvSRWS5W49FJIh20ZF7AUkmMDo/edit?usp=sharing

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                          | Input Node(s)                   | Output Node(s)                         | Sticky Note                                                                                                                        |
|--------------------------------|----------------------------------|----------------------------------------|--------------------------------|--------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| When clicking "Test workflow"  | Manual Trigger                   | Initiates the workflow manually        | -                              | Get rows to enrich                   |                                                                                                                                   |
| Get rows to enrich             | Google Sheets                   | Reads raw company input rows            | When clicking "Test workflow"  | Input                               | See Sticky Note7 for Google Sheets template and setup instructions.                                                               |
| Input                         | Set                            | Assigns input variables                 | Get rows to enrich             | Loop Over Items                     |                                                                                                                                   |
| Loop Over Items               | SplitInBatches                 | Processes input in batches              | Input                         | AI company researcher, Merge data   |                                                                                                                                   |
| MCP Client                   | LangChain MCP Client Tool       | Connects to Explorium MCP for research | Loop Over Items (AI tool)      | AI company researcher (ai_tool)     | See Sticky Note4 about Explorium MCP server handling research.                                                                    |
| Anthropic Chat Model         | LangChain AI Chat Model         | Provides AI language model (Claude)    | AI company researcher (ai_languageModel) | AI company researcher             |                                                                                                                                   |
| Get website content          | LangChain Tool Workflow          | Retrieves and cleans website content   | AI company researcher (ai_tool) | AI company researcher (ai_tool)     |                                                                                                                                   |
| AI company researcher        | LangChain Agent                 | Orchestrates AI prompt and research    | Loop Over Items, MCP Client, Anthropic Chat Model, Get website content | Structured Output Parser          | See Sticky Note3 for prompt guidance.                                                                                            |
| Structured Output Parser     | LangChain Output Parser Structured | Parses AI output into structured JSON  | AI company researcher          | AI Researcher Output Data           | See Sticky Note2 about specifying output data format.                                                                             |
| AI Researcher Output Data    | Set                            | Maps structured output fields          | Structured Output Parser       | Merge data                        |                                                                                                                                   |
| Merge data                  | Merge                          | Merges original input with AI output   | AI Researcher Output Data, Loop Over Items | Google Sheets - Update Row with data |                                                                                                                                   |
| Google Sheets - Update Row with data | Google Sheets               | Updates sheet rows with enriched data  | Merge data                    | Loop Over Items                    | See Sticky Note7 for Google Sheets template and setup instructions.                                                               |
| Sticky Note                 | Sticky Note                    | Workflow overview and instructions     | -                             | -                                  | Contains workflow description and video guide link: https://lempire.notion.site/AI-Web-research-with-n8n-a25aae3258d0423481a08bd102f16906 |
| Sticky Note2                | Sticky Note                    | Output format reminder                  | -                             | -                                  |                                                                                                                                   |
| Sticky Note3                | Sticky Note                    | AI prompt guidance                      | -                             | -                                  |                                                                                                                                   |
| Sticky Note4                | Sticky Note                    | Explorium MCP server info               | -                             | -                                  |                                                                                                                                   |
| Sticky Note7                | Sticky Note                    | Google Sheets template instructions    | -                             | -                                  | Link to Google Sheets template: https://docs.google.com/spreadsheets/d/1vR6s2nlTwu01v3GP7wvSRWS5W49FJIh20ZF7AUkmMDo/edit?usp=sharing  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger:**  
   - Add **Manual Trigger** node named "When clicking \"Test workflow\"". No parameters required.

2. **Add Google Sheets Node to Read Input Rows:**  
   - Add **Google Sheets** node named "Get rows to enrich".  
   - Set operation to read rows from your spreadsheet.  
   - Configure `documentId` and `sheetName` (e.g., "input").  
   - Set up OAuth2 credentials for Google Sheets API.  
   - Connect "When clicking \"Test workflow\"" → "Get rows to enrich".

3. **Add Set Node for Input Assignment:**  
   - Add **Set** node named "Input".  
   - Assign two variables:  
     - `company_input` → `{{$json.input}}`  
     - `row_number` → `{{$json.row_number}}`  
   - Connect "Get rows to enrich" → "Input".

4. **Add SplitInBatches Node:**  
   - Add **SplitInBatches** node named "Loop Over Items".  
   - No special batch size needed (default).  
   - Connect "Input" → "Loop Over Items".

5. **Add MCP Client Node:**  
   - Add **LangChain MCP Client Tool** node named "MCP Client".  
   - Configure SSE endpoint: `mcp.explorium.ai/sse`.  
   - Use HTTP Header Authentication credential for Explorium.  
   - Connect "Loop Over Items" (ai_tool output) → "MCP Client" (ai_tool input).

6. **Add Anthropic Chat Model Node:**  
   - Add **LangChain AI Chat Anthropic** node named "Anthropic Chat Model".  
   - Set model to `claude-3-7-sonnet-20250219`.  
   - Use Anthropic API credential.  
   - Connect "MCP Client" (ai_languageModel output) → "Anthropic Chat Model" (ai_languageModel input).

7. **Add Sub-Workflow Node for Website Content Retrieval:**  
   - Create sub-workflow "get_website_content":  
     - Nodes:  
       - Execute Workflow Trigger (start)  
       - HTTP Request node to visit URL (`{{$json.query.url}}`)  
       - HTML Extract node configured to extract `<html>` excluding `<head>` and clean text.  
     - This sub-workflow accepts a URL input and returns page text content as `body`.  
   - Add **LangChain Tool Workflow** node named "Get website content".  
   - Configure to use the above sub-workflow.  
   - Connect "Loop Over Items" (ai_tool output) → "Get website content" (ai_tool input).

8. **Add LangChain Agent Node for AI Research:**  
   - Add **LangChain Agent** node named "AI company researcher".  
   - Configure prompt with instructions to research company data including domain, LinkedIn URL, market type (B2B/B2C), pricing plans, case study URLs, API presence, enterprise plan, free trial, integrations.  
   - Set max iterations to 20.  
   - Enable output parser.  
   - Connect inputs:  
     - From "Loop Over Items" (main output)  
     - From "MCP Client" (ai_tool output)  
     - From "Anthropic Chat Model" (ai_languageModel output)  
     - From "Get website content" (ai_tool output)

9. **Add Structured Output Parser Node:**  
   - Add **LangChain Output Parser Structured** node named "Structured Output Parser".  
   - Define a manual JSON schema for output structure with fields:  
     `case_study_link` (string|null), `domain` (string|null), `linkedinUrl` (string|null), `market` (string|null), `cheapest_plan` (number|null), `has_enterprise_plan` (boolean|null), `has_API` (boolean|null), `has_free_trial` (boolean|null), `integrations` (array|null).  
   - Connect "AI company researcher" (ai_outputParser output) → "Structured Output Parser".

10. **Add Set Node for Output Mapping:**  
    - Add **Set** node named "AI Researcher Output Data".  
    - Assign variables from structured output JSON: domain, linkedinUrl, market, cheapest_plan, has_enterprise_plan, has_API, has_free_trial, integrations, case_study_link.  
    - Connect "Structured Output Parser" → "AI Researcher Output Data".

11. **Add Merge Node:**  
    - Add **Merge** node named "Merge data".  
    - Set mode to "combine" and combination mode to "mergeByPosition".  
    - Connect "AI Researcher Output Data" → input 1, and "Loop Over Items" → input 2.  
    - This merges original input with AI enriched data.

12. **Add Google Sheets Node to Update Rows:**  
    - Add **Google Sheets** node named "Google Sheets - Update Row with data".  
    - Set operation to "update".  
    - Configure `documentId` and sheet name same as input sheet.  
    - Map columns: domain, market, linkedinUrl, integrations, cheapest_plan, has_free_trial, enrichment_status (set to "done"), has_entreprise_plan, last_case_study_link, row_number (used as matching column).  
    - Use OAuth2 Google Sheets credential.  
    - Connect "Merge data" → "Google Sheets - Update Row with data".

13. **Connect Loop Back for Batch Processing:**  
    - Connect "Google Sheets - Update Row with data" → "Loop Over Items" (to continue batch processing).

14. **Add Sticky Notes:**  
    - Add sticky notes with content as per documentation for user guidance, prompt instructions, output format notes, and Google Sheets template link.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| This workflow allows you to do account research with the web using AI, combining Google Search (via SerpAPI) and website content extraction. The output fields and prompt can be customized in the Structured Output Parser node. | Sticky Note (Read Me) with detailed workflow description                                                                         |
| Specify the exact output data format in the "Structured Output Parser" node to match your data requirements.                                                                                                                   | Sticky Note2                                                                                                                    |
| The AI prompt instructs what information to research about the company, such as domain, LinkedIn, plans, API, integrations, etc.                                                                                               | Sticky Note3                                                                                                                    |
| Explorium MCP server executes the bulk of the web research, streamlining data retrieval.                                                                                                                                       | Sticky Note4                                                                                                                    |
| Use the provided Google Sheets template for faster setup. Copy it, then update the `documentId` in the Google Sheets nodes accordingly.                                                                                      | Sticky Note7 with link: https://docs.google.com/spreadsheets/d/1vR6s2nlTwu01v3GP7wvSRWS5W49FJIh20ZF7AUkmMDo/edit?usp=sharing     |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---