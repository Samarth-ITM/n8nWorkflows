Competitor Intelligence Agent: SERP Monitoring + Summary with Thordata + OpenAI

https://n8nworkflows.xyz/workflows/competitor-intelligence-agent--serp-monitoring---summary-with-thordata---openai-10252


# Competitor Intelligence Agent: SERP Monitoring + Summary with Thordata + OpenAI

---

## 1. Workflow Overview

This workflow automates competitor intelligence gathering by performing SERP (Search Engine Results Page) monitoring and detailed SEO analysis using multiple search engines via Thordata API and advanced AI processing with OpenAI GPT models. Its main goal is to extract competitor insights, keyword opportunities, and content gap analysis for strategic SEO planning.

The workflow is logically structured into these main functional blocks:

- **1.1 Input Reception and Initialization**  
  Receives and sets the search query for SEO analysis.

- **1.2 Multi-Engine SERP Data Retrieval**  
  Queries multiple search engines (Bing, Google, Yandex, DuckDuckGo) through Thordata SERP API for comprehensive search results.

- **1.3 AI Agent Processing**  
  Uses LangChain AI Agent with OpenAI GPT-4.1-mini to process the search query and orchestrate the analysis.

- **1.4 Data Enrichment and Analysis**  
  Performs summarization of SERP content, keyword and topic extraction, and SEO competitor analysis via structured AI information extractors.

- **1.5 Data Merging and Export**  
  Consolidates AI outputs and exports the insights into a Google Sheets document for tracking and reporting.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and Initialization

**Overview:**  
This block sets the initial search query that drives the entire SERP analysis process. It can be triggered manually.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Set the Input Fields

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual trigger  
  - Role: Initiates workflow execution on user command  
  - Config: No parameters; simple manual start  
  - Input: None  
  - Output: Triggers next node Set the Input Fields  
  - Edge cases: User-trigger only; no fail modes expected

- **Set the Input Fields**  
  - Type: Set node  
  - Role: Defines input parameter `search_query` with a fixed string value  
  - Config: Sets `search_query` to `"Google Search for Top SEO strategies for e-commerce in 2025"`  
  - Input: Trigger from manual node  
  - Output: Passes JSON with `search_query` field  
  - Edge cases: Static input; can be modified for dynamic queries

---

### 2.2 Multi-Engine SERP Data Retrieval

**Overview:**  
This block queries four different search engines via the Thordata API to collect broad SERP data for the input query.

**Nodes Involved:**  
- Bing Search  
- Google Search  
- Yandex Search  
- DuckDuckGo Search

**Node Details (all similar):**

- **Node Type:** HTTP Request Tool  
- **Role:** Sends POST request to `https://scraperapi.thordata.com/request` with form-urlencoded body to retrieve SERP JSON data  
- **Configuration Highlights:**  
  - Authenticated using a predefined HTTP Bearer credential with Thordata API token  
  - Body parameters include:  
    - `engine`: search engine name (`bing`, `google`, `yandex`, `duckduckgo`)  
    - `q`: query string dynamically overridden by AI or static input (`$fromAI` expression fallback)  
    - `json`: set to `"1"` for JSON response  
- **Input:** Receives trigger from AI Agent's ai_tool input connection  
- **Output:** JSON SERP data to AI Agent  
- **Edge cases:**  
  - API authentication failure (invalid/missing token)  
  - Thordata rate limits or downtime  
  - Empty or malformed query  
  - Network timeout or HTTP errors

---

### 2.3 AI Agent Processing

**Overview:**  
Serves as the orchestrator AI component that receives the search query, queries the search engines, and triggers downstream AI analyses.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent node (version 2.2)  
  - Role: Central AI orchestrator; receives the `search_query` and coordinates search engine requests and downstream processing  
  - Configuration:  
    - Uses `text` input bound to `$json.search_query`  
    - Prompt type: `define` (structured prompt)  
    - Retry enabled on failure  
  - Inputs: From Set the Input Fields node  
  - Outputs: Connects to Bing, Google, Yandex, DuckDuckGo Search nodes via ai_tool; connects to Summarize content, Keyword and Topic Analysis, SEO Analyst nodes for AI processing  
  - Edge cases: Expression failures if `search_query` missing; failure to route calls; AI processing errors  
  - Requires OpenAI credentials

- **OpenAI Chat Model**  
  - Type: LangChain LLM Chat OpenAI node (version 1.2)  
  - Role: Provides GPT-4.1-mini language model for the AI Agent  
  - Configuration: Model set to `gpt-4.1-mini` with default options  
  - Credentials: OpenAI API credentials required  
  - Input: Connected from AI Agent node as `ai_languageModel`  
  - Output: Feeds responses back to AI Agent  
  - Edge cases: API quota, authentication, rate limits, timeout

---

### 2.4 Data Enrichment and Analysis

**Overview:**  
Processes raw SERP data to generate summaries, keyword/topic analyses, and competitor SEO insights using OpenAI-powered LangChain nodes with structured output schemas.

**Nodes Involved:**  
- Summarize the content  
- OpenAI Chat Model For Summarization  
- Keyword and Topic Analysis  
- OpenAI Chat Model for Keyword and Topic Analysis  
- SEO Analyst  
- OpenAI Chat Model for SEO Analyst  
- Structured Output Parser

**Node Details:**

- **Summarize the content**  
  - Type: LangChain Chain LLM (v1.7)  
  - Role: Summarizes the SERP content output into a concise summary without advice or suggestions  
  - Configuration: Uses prompt `"Summarize the following content {{ $json.output }}. Output just the summary."`  
  - Input: From AI Agent  
  - Output: To Merge node  
  - Edge cases: Failures in text parsing or empty input

- **OpenAI Chat Model For Summarization**  
  - Type: LangChain LLM Chat OpenAI (v1.2)  
  - Role: GPT-4.1-mini model to generate the summary  
  - Credentials: OpenAI API required  
  - Input: Connected from Summarize the content node as language model  
  - Output: To Summarize the content for final output

- **Keyword and Topic Analysis**  
  - Type: LangChain Information Extractor (v1.2)  
  - Role: Extracts structured keyword and topic data from summarized content using a detailed JSON schema  
  - Configuration:  
    - Prompt requests keyword and topic analysis with a complex JSON schema defining keywords, metrics, topic clusters, competitor insights, and AI summary  
  - Input: From AI Agent  
  - Output: To Merge node  
  - Edge cases: Parsing errors, schema validation failures

- **OpenAI Chat Model for Keyword and Topic Analysis**  
  - Type: LangChain LLM Chat OpenAI (v1.2)  
  - Role: GPT-4.1-mini model for keyword/topic extraction  
  - Credentials: OpenAI API required  
  - Input: Connected to Keyword and Topic Analysis node as language model

- **SEO Analyst**  
  - Type: LangChain Information Extractor (v1.2)  
  - Role: Analyzes SERP results to extract competitor insights, keyword opportunities, SEO metrics, and content gaps using a comprehensive JSON schema  
  - Configuration:  
    - System prompt defines role as SEO analyst  
    - JSON schema includes competitor details, keyword opportunities, topic clusters, content gaps, SEO scores, and AI summary  
  - Input: From AI Agent node  
  - Output: To Merge node  
  - Edge cases: Schema validation, AI response correctness

- **OpenAI Chat Model for SEO Analyst**  
  - Type: LangChain LLM Chat OpenAI (v1.2)  
  - Role: GPT-4.1-mini model providing NLP for SEO Analyst node  
  - Credentials: OpenAI API required  
  - Input: Connected to SEO Analyst node as language model

- **Structured Output Parser**  
  - Type: LangChain Output Parser Structured (v1.3)  
  - Role: Parses AI output into structured JSON with two fields: `comprehensive_summary` and `abstract_summary`  
  - Configuration: Manual schema with two string properties  
  - Input: From Summarize the content node's AI output parser connection  
  - Output: Feeds parsed summary back into Summarize the content node

---

### 2.5 Data Merging and Export

**Overview:**  
Consolidates outputs from summary, keyword/topic analysis, and SEO analyst into a single data stream and exports the results to a Google Sheets document for record keeping and reporting.

**Nodes Involved:**  
- Merge  
- Append or update row in sheet

**Node Details:**

- **Merge**  
  - Type: Merge node (v3.2)  
  - Role: Combines three input streams (summary, keyword analysis, SEO analyst) into one output data set  
  - Configuration: Set to handle 3 inputs  
  - Inputs: From Summarize the content, Keyword and Topic Analysis, SEO Analyst  
  - Output: To Google Sheets node  
  - Edge cases: Missing inputs can cause incomplete merges

- **Append or update row in sheet**  
  - Type: Google Sheets node (v4.7)  
  - Role: Appends or updates a row in a specified Google Sheet for tracking SEO analysis data  
  - Configuration:  
    - Operation: appendOrUpdate  
    - Sheet name: `gid=0` (Sheet1)  
    - Document ID: Google Sheet ID provided  
    - Mapping Mode: Auto map input data fields to columns  
  - Credentials: Google Sheets OAuth2 required  
  - Input: From Merge node  
  - Edge cases: Google API auth errors, spreadsheet access permissions, rate limits

---

## 3. Summary Table

| Node Name                            | Node Type                                  | Functional Role                                    | Input Node(s)                    | Output Node(s)                         | Sticky Note                                                                                                    |
|------------------------------------|--------------------------------------------|---------------------------------------------------|---------------------------------|--------------------------------------|----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’   | Manual Trigger                             | Starts the workflow manually                       | -                               | Set the Input Fields                  |                                                                                                                |
| Set the Input Fields                | Set                                        | Defines initial search query                        | When clicking ‘Execute workflow’ | AI Agent                            |                                                                                                                |
| AI Agent                           | LangChain Agent                            | Orchestrates AI processing and search queries     | Set the Input Fields             | Bing Search, Google Search, Yandex Search, DuckDuckGo Search, Summarize the content, Keyword and Topic Analysis, SEO Analyst |                                                                                                                |
| Bing Search                       | HTTP Request Tool                           | Queries Bing SERP via Thordata API                  | AI Agent (ai_tool)               | AI Agent (ai_tool)                   |                                                                                                                |
| Google Search                     | HTTP Request Tool                           | Queries Google SERP via Thordata API                | AI Agent (ai_tool)               | AI Agent (ai_tool)                   |                                                                                                                |
| Yandex Search                    | HTTP Request Tool                           | Queries Yandex SERP via Thordata API                | AI Agent (ai_tool)               | AI Agent (ai_tool)                   |                                                                                                                |
| DuckDuckGo Search               | HTTP Request Tool                           | Queries DuckDuckGo SERP via Thordata API            | AI Agent (ai_tool)               | AI Agent (ai_tool)                   |                                                                                                                |
| OpenAI Chat Model                | LangChain LLM Chat OpenAI                   | Provides GPT-4.1-mini model for AI Agent           | AI Agent (ai_languageModel)     | AI Agent                            |                                                                                                                |
| Summarize the content            | LangChain Chain LLM                         | Summarizes SERP content                             | AI Agent                        | Merge                              |                                                                                                                |
| OpenAI Chat Model For Summarization | LangChain LLM Chat OpenAI                   | GPT model for summarization                         | Summarize the content (ai_languageModel) | Summarize the content         |                                                                                                                |
| Keyword and Topic Analysis      | LangChain Information Extractor             | Extracts keywords and topic insights               | AI Agent                        | Merge                              |                                                                                                                |
| OpenAI Chat Model for Keyword and Topic Analysis | LangChain LLM Chat OpenAI                   | GPT model for keyword and topic analysis           | Keyword and Topic Analysis (ai_languageModel) | Keyword and Topic Analysis    |                                                                                                                |
| SEO Analyst                    | LangChain Information Extractor             | Extracts competitor insights and SEO opportunities | AI Agent                        | Merge                              |                                                                                                                |
| OpenAI Chat Model for SEO Analyst | LangChain LLM Chat OpenAI                   | GPT model for SEO analysis                          | SEO Analyst (ai_languageModel)  | SEO Analyst                        |                                                                                                                |
| Structured Output Parser         | LangChain Output Parser Structured           | Parses AI summary outputs                           | Summarize the content (ai_outputParser) | Summarize the content        |                                                                                                                |
| Merge                           | Merge                                       | Combines multiple AI analysis outputs              | Summarize the content, Keyword and Topic Analysis, SEO Analyst | Append or update row in sheet |                                                                                                                |
| Append or update row in sheet    | Google Sheets                               | Exports consolidated data to Google Sheets         | Merge                          | -                                  |                                                                                                                |
| Sticky Note1                    | Sticky Note                                 | Logo display                                       | -                               | -                                  | ![Logo](https://consumersiteimages.trustpilot.net/business-units/67b212598525b99cf90a59cc-198x149-1x.jpg)      |
| Sticky Note                     | Sticky Note                                 | Describes SERP AI Agent block                       | -                               | -                                  | ## SERP AI Agent - Performs the SERP search via Thordata SERP API                                              |
| Sticky Note2                    | Sticky Note                                 | Describes Data Enrichment block                      | -                               | -                                  | ## Data Enrichment - Perform Comprehensive and Abstract Summaries and Keyword/Topic analysis                   |
| Sticky Note3                    | Sticky Note                                 | Describes SEO Analyst block                          | -                               | -                                  | ## SEO Analyst - Analyzes the SERP results for competitor insights and keyword opportunities                    |
| Sticky Note4                    | Sticky Note                                 | Describes Export Data Handling block                 | -                               | -                                  | ## Export Data Handling - Exports the output to the Google Spreadsheet                                         |
| Sticky Note5                    | Sticky Note                                 | Workflow purpose and overview                         | -                               | -                                  | ## **Purpose:** Automate SERP analysis for competitor and keyword insights using Thordata and OpenAI. Workflow steps and use case detailed. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node: Manual Trigger  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters.

2. **Create Set Node for Input Query**  
   - Node: Set  
   - Name: `Set the Input Fields`  
   - Add field: `search_query` (string) with value: `"Google Search for Top SEO strategies for e-commerce in 2025"`  
   - Connect Manual Trigger to this Set node.

3. **Create LangChain Agent Node**  
   - Node: LangChain Agent (`@n8n/n8n-nodes-langchain.agent`)  
   - Name: `AI Agent`  
   - Parameters:  
     - `text`: Expression `{{$json.search_query}}`  
     - Prompt type: `define`  
     - Enable retry on failure  
   - Connect Set node output to AI Agent input.

4. **Setup HTTP Request Nodes for SERP Engines**  
   For each search engine (Bing, Google, Yandex, DuckDuckGo):

   - Node: HTTP Request Tool  
   - Name: `{Engine} Search` (e.g., `Bing Search`)  
   - Parameters:  
     - URL: `https://scraperapi.thordata.com/request`  
     - Method: POST  
     - Content-Type: `application/x-www-form-urlencoded`  
     - Authentication: Use HTTP Bearer Auth Credential with your Thordata API token  
     - Body Parameters:  
       - `engine`: engine name (e.g., `bing`)  
       - `q`: Expression: `{{$fromAI('parameters1_Value', '', 'string')}}` (to get override from AI Agent)  
       - `json`: `1`  
     - Send Headers: true  
   - Connect AI Agent node to each HTTP Request node via the `ai_tool` connection.

5. **Create OpenAI Chat Model for AI Agent**  
   - Node: LangChain LLM Chat OpenAI  
   - Name: `OpenAI Chat Model`  
   - Model: `gpt-4.1-mini`  
   - Credentials: Configure OpenAI API credentials  
   - Connect to AI Agent node via `ai_languageModel` input.

6. **Create Summarization Nodes**  
   - LangChain Chain LLM node:  
     - Name: `Summarize the content`  
     - Text: `"Summarize the following content {{ $json.output }}. Output just the summary. Do not provide your own suggestions or recommendations"`  
     - Prompt type: `define`  
     - Retry on fail: true  
   - LangChain LLM Chat OpenAI node:  
     - Name: `OpenAI Chat Model For Summarization`  
     - Model: `gpt-4.1-mini`  
     - Credentials: OpenAI API  
   - Connect OpenAI Chat Model For Summarization to Summarize the content as language model.  
   - Connect AI Agent to Summarize the content node.

7. **Create Keyword and Topic Analysis Nodes**  
   - LangChain Information Extractor node:  
     - Name: `Keyword and Topic Analysis`  
     - Text: `"Perform Keyword and Topic Analysis of the following {{ $json.output }}"`  
     - Schema: Use provided detailed JSON schema for keyword, topic, competitor insights, and AI summary.  
   - LangChain LLM Chat OpenAI node:  
     - Name: `OpenAI Chat Model for Keyword and Topic Analysis`  
     - Model: `gpt-4.1-mini`  
     - Credentials: OpenAI API  
   - Connect OpenAI Chat Model for Keyword and Topic Analysis to Keyword and Topic Analysis node as language model input.  
   - Connect AI Agent to Keyword and Topic Analysis node.

8. **Create SEO Analyst Nodes**  
   - LangChain Information Extractor node:  
     - Name: `SEO Analyst`  
     - Text: `"Analyze the following SERP results for competitor insights and keyword opportunities: {{ $json.output }}"`  
     - System prompt template: `"You are an SEO analyst. Extract primary keywords, competitor names, content tone, and summarize SEO strength from SERP data."`  
     - Schema: Use the detailed JSON schema for competitor insights, keyword opportunities, topic clusters, content gaps, SEO scores, and AI summary.  
   - LangChain LLM Chat OpenAI node:  
     - Name: `OpenAI Chat Model for SEO Analyst`  
     - Model: `gpt-4.1-mini`  
     - Credentials: OpenAI API  
   - Connect OpenAI Chat Model for SEO Analyst to SEO Analyst node as language model input.  
   - Connect AI Agent to SEO Analyst node.

9. **Create Structured Output Parser Node**  
   - Node: LangChain Output Parser Structured  
   - Name: `Structured Output Parser`  
   - Schema: Manual JSON schema with properties `comprehensive_summary` and `abstract_summary` both of type string  
   - Connect Summarize the content node’s AI output parser to this node.

10. **Create Merge Node**  
    - Node: Merge  
    - Name: `Merge`  
    - Parameters: Number of inputs set to 3  
    - Connect outputs of Summarize the content, Keyword and Topic Analysis, and SEO Analyst nodes to the three inputs of the Merge node.

11. **Create Google Sheets Node for Export**  
    - Node: Google Sheets  
    - Name: `Append or update row in sheet`  
    - Operation: `appendOrUpdate`  
    - Document ID: (set your Google Sheet ID)  
    - Sheet Name: `gid=0` (Sheet1)  
    - Mapping Mode: Auto map input data to columns  
    - Credentials: Google Sheets OAuth2 credentials configured  
    - Connect Merge node output to this Google Sheets node.

12. **Add Optional Sticky Notes**  
    - Add sticky notes for documentation near each block for clarity (e.g., "SERP AI Agent", "Data Enrichment", "SEO Analyst", "Export Data Handling", and workflow purpose).

---

## 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                                            |
|------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| ![Logo](https://consumersiteimages.trustpilot.net/business-units/67b212598525b99cf90a59cc-198x149-1x.jpg)                         | Branding displayed in Sticky Note1                                                                                         |
| Workflow automates SERP analysis for competitor insights and SEO keyword opportunities using Thordata API and OpenAI GPT models.   | Sticky Note5 contains purpose and detailed workflow step summary                                                            |
| Thordata SERP API requires Bearer token authentication for search engine requests.                                                  | Credential setup required for HTTP Bearer Auth nodes (Bing, Google, Yandex, DuckDuckGo Search nodes)                        |
| OpenAI GPT-4.1-mini model used throughout for summarization and analysis tasks.                                                    | OpenAI API credentials needed for all LangChain LLM Chat OpenAI nodes                                                       |
| Google Sheets node appends or updates rows for historical tracking and reporting of SEO insights.                                 | Google Sheets OAuth2 credentials required                                                                                   |
| Recommended use case: Weekly competitor SEO monitoring for marketing teams and agencies.                                           | Workflow design supports manual trigger but can be modified to scheduled trigger for automation                              |
| Detailed JSON schemas ensure structured and consistent AI output enabling easier parsing and integration downstream.              | Schemas defined in Keyword and Topic Analysis and SEO Analyst nodes                                                          |

---

**Disclaimer:**  
The provided text comes exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.

---