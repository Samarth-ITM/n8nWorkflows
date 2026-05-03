Generate Comprehensive Research Reports with Gemini AI and Tavily Search for Japanese Users

https://n8nworkflows.xyz/workflows/generate-comprehensive-research-reports-with-gemini-ai-and-tavily-search-for-japanese-users-5897


# Generate Comprehensive Research Reports with Gemini AI and Tavily Search for Japanese Users

### 1. Workflow Overview

This workflow automates comprehensive research report generation tailored for Japanese users. It starts from a user question, automatically generates optimized search queries using Google Gemini AI, performs advanced multi-query searches via the Tavily search tool, integrates findings into a structured HTML report, and finally emails the report. The workflow is designed to facilitate deep research with minimal manual input, ideal for knowledge workers, researchers, and content creators focusing on Japanese-language topics.

Logical blocks:

- **1.1 Input Reception:** Accepts or sets the user question.
- **1.2 AI Query Generation:** Uses Google Gemini AI to create 3 optimized search queries.
- **1.3 Multi-Query Research:** Executes searches on Tavily with the generated queries, gathering detailed information.
- **1.4 Report Generation:** Integrates and structures the collected data into an easy-to-read HTML report.
- **1.5 Email Delivery:** Sends the final report via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives and sets the initial user question to be researched.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- query (Set)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually.  
  - Configuration: No parameters; triggers workflow execution on manual command.  
  - Input: None  
  - Output: Connects to `query` node.  
  - Edge cases: None; user must manually trigger workflow.  

- **query**  
  - Type: Set node  
  - Role: Sets the user question for research.  
  - Configuration: Assigns a string value `"n8nとdifyの違い"` (meaning "Difference between n8n and dify") to the field `query`.  
  - Key variables: `query` field used downstream.  
  - Input: Triggered from Manual Trigger node.  
  - Output: Connects to `Query Generator`.  
  - Edge cases: User must edit this node to input a different question; empty or irrelevant queries may reduce output quality.  

---

#### 1.2 AI Query Generation

**Overview:**  
Analyzes the user’s question and generates three effective search queries optimized for deep research.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Structured Output Parser  
- Query Generator

**Node Details:**  

- **Google Gemini Chat Model**  
  - Type: Google Gemini Chat (PaLM) Language Model node  
  - Role: Processes the input question text to begin query generation.  
  - Configuration: Model `models/gemini-2.5-flash` without extra options.  
  - Input: Receives input from `query` node (implicitly via connection to `Query Generator`).  
  - Output: Sends result to `Structured Output Parser`.  
  - Edge cases: API failures, rate limiting, or malformed inputs could disrupt processing. Requires valid Google Gemini API credentials.  

- **Structured Output Parser**  
  - Type: Structured Output Parser (Langchain)  
  - Role: Parses AI output into a structured JSON object containing three string queries: `query1`, `query2`, `query3`.  
  - Configuration: JSON schema specifying three string properties.  
  - Input: Output from the Google Gemini Chat Model.  
  - Output: Passes parsed queries to `Query Generator` node.  
  - Edge cases: Parsing errors if AI output deviates from expected schema.  

- **Query Generator**  
  - Type: Langchain Agent  
  - Role: Coordinates AI prompt and output parsing to generate three optimized search queries based on the user question.  
  - Configuration:  
    - Prompt: Instructions in Japanese to extract three potent search queries from the question, outputting only queries without explanation.  
    - System message details the role and requirements, including English query output if effective.  
    - Uses structured output parser to enforce output format.  
  - Input: Receives raw question from `query` node and AI model output from parsed Gemini model response.  
  - Output: Sends structured queries to `Research Agent`.  
  - Edge cases: If input question is ambiguous or poorly formed, generated queries may be irrelevant or weak.  

---

#### 1.3 Multi-Query Research

**Overview:**  
Executes advanced searches on Tavily using each of the three AI-generated queries, retrieves detailed information, and consolidates findings.

**Nodes Involved:**  
- Google Gemini Chat Model1  
- Tavily_Search_Tool  
- Research Agent

**Node Details:**  

- **Google Gemini Chat Model1**  
  - Type: Google Gemini Chat model node  
  - Role: Supports the Research Agent with AI language model capabilities during research integration.  
  - Configuration: Model `models/gemini-2.5-flash`, with credentials for Google Palm API.  
  - Input: Invoked by `Research Agent`.  
  - Output: Supports `Research Agent` processing.  
  - Edge cases: API authentication errors, rate limits.  

- **Tavily_Search_Tool**  
  - Type: Tavily search tool node  
  - Role: Conducts search queries on Tavily with advanced options.  
  - Configuration:  
    - Query input dynamically set from AI-generated queries (`$fromAI('Query', ...)`).  
    - Parameters: `max_results`=10, `search_depth`="advanced", `include_answer`="advanced".  
  - Input: Receives queries from `Research Agent`.  
  - Output: Returns detailed search results back to `Research Agent`.  
  - Edge cases: API quota limits, no results found, network errors. Requires valid Tavily API credentials.  

- **Research Agent**  
  - Type: Langchain Agent  
  - Role: Uses the three queries to perform research via Tavily and produces a detailed output combining queries and results.  
  - Configuration:  
    - Prompt in Japanese instructing the agent to call Tavily_Search_Tool with each query and output full search queries and uncensored results.  
    - System message emphasizes inclusion of AI-generated answers and detailed analysis.  
  - Input: Receives structured queries from `Query Generator`.  
  - Output: Sends research results to `Report Agent`.  
  - Edge cases: Failure in any Tavily search call could impact completeness; partial results might be returned.  

---

#### 1.4 Report Generation

**Overview:**  
Integrates and analyzes the collected research data, generating a comprehensive, structured HTML report tailored to the original user question.

**Nodes Involved:**  
- Google Gemini Chat Model2  
- Report Agent

**Node Details:**  

- **Google Gemini Chat Model2**  
  - Type: Google Gemini Chat model node  
  - Role: Provides advanced AI language model processing to synthesize and structure the final report.  
  - Configuration:  
    - Model: `models/gemini-2.5-pro` (higher quality, max output tokens 6000).  
    - Uses Google Palm API credentials.  
  - Input: Invoked by `Report Agent`.  
  - Output: Supports `Report Agent` in report generation.  
  - Edge cases: API rate limits, token limits exceeded.  

- **Report Agent**  
  - Type: Langchain Agent  
  - Role: Combines the multiple research results into a single coherent HTML report answering the user’s question.  
  - Configuration:  
    - Prompt instructs to create a structured, comprehensive, easy-to-understand HTML report, eliminating redundancy and contradictions.  
    - Inputs: User question and integrated research results.  
    - Output: HTML formatted report.  
  - Input: Receives research output from `Research Agent`.  
  - Output: Sends HTML report to `Send a message` node.  
  - Edge cases: Insufficient data may lead to less complete reports; malformed HTML output could affect email rendering.  

---

#### 1.5 Email Delivery

**Overview:**  
Sends the final HTML research report by email using Gmail.

**Nodes Involved:**  
- Send a message (Gmail)

**Node Details:**  

- **Send a message**  
  - Type: Gmail node  
  - Role: Sends the finalized HTML report as an email.  
  - Configuration:  
    - Subject line dynamically set to the original user question (`$('query').item.json.query`).  
    - Email body contains the HTML report (`$json.output`).  
    - Requires Gmail OAuth2 credentials.  
  - Input: Receives report from `Report Agent`.  
  - Output: None (ends workflow).  
  - Edge cases: Email sending failures (authentication, quota, invalid recipient). User must configure recipient address.  

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                  | Input Node(s)               | Output Node(s)          | Sticky Note                                                                                                                             |
|---------------------------|----------------------------------|--------------------------------|----------------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Start workflow manually         | None                       | query                   |                                                                                                                                         |
| query                     | Set                              | Sets user question              | When clicking ‘Execute workflow’ | Query Generator          | ## 🎯 Step 1: Query Input Setup<br>Set user question here; default is "n8nとdifyの違い".                                                  |
| Google Gemini Chat Model  | Google Gemini Chat Model (PaLM)  | Initial AI processing           | Query Generator (indirect) | Structured Output Parser | ## 🧠 Step 2: AI Query Generation<br>Uses Google Gemini 2.5-flash to generate 3 optimized search queries from the input question.<br>Requires Google Gemini API credentials. |
| Structured Output Parser  | Structured Output Parser (Langchain) | Parses AI output into structured queries | Google Gemini Chat Model   | Query Generator          |                                                                                                                                         |
| Query Generator           | Langchain Agent                  | Generates 3 optimized queries   | query                      | Research Agent           |                                                                                                                                         |
| Google Gemini Chat Model1 | Google Gemini Chat Model (PaLM)  | AI support for research agent   | Research Agent             | Research Agent           | ## 🔍 Step 3: Multi-Query Research<br>Uses Tavily to perform advanced searches with generated queries.<br>Needs Tavily API credentials. |
| Tavily_Search_Tool        | Tavily search tool               | Executes advanced web searches  | Research Agent             | Research Agent           |                                                                                                                                         |
| Research Agent            | Langchain Agent                  | Performs research with queries  | Query Generator            | Report Agent             |                                                                                                                                         |
| Google Gemini Chat Model2 | Google Gemini Chat Model (PaLM)  | AI support for report generation | Report Agent               | Report Agent             | ## 📊 Step 4: Report Generation<br>Creates comprehensive HTML report.<br>Uses higher-quality Gemini 2.5-pro model.                      |
| Report Agent              | Langchain Agent                  | Creates final HTML report       | Research Agent             | Send a message           |                                                                                                                                         |
| Send a message            | Gmail                           | Sends final report email        | Report Agent               | None                    | ## 📧 Step 5: Email Delivery<br>Sends the report via Gmail.<br>Configure recipient email address appropriately.                         |
| Sticky Note               | Sticky Note                     | Documentation note              | None                       | None                    | ## 📋 Simple Deep Research for Japanese Users<br>Workflow overview and usage instructions in Japanese.                                   |
| Sticky Note1              | Sticky Note                     | Documentation note              | None                       | None                    | ## 🎯 Step 1: Query Input Setup<br>Details about setting the user question.                                                              |
| Sticky Note2              | Sticky Note                     | Documentation note              | None                       | None                    | ## 🧠 Step 2: AI Query Generation<br>Explains query generation using Google Gemini 2.5-flash.                                             |
| Sticky Note3              | Sticky Note                     | Documentation note              | None                       | None                    | ## 🔍 Step 3: Multi-Query Research<br>Details on Tavily search setup and usage.                                                          |
| Sticky Note4              | Sticky Note                     | Documentation note              | None                       | None                    | ## 📊 Step 4: Report Generation<br>Information on report creation using Gemini 2.5-pro.                                                  |
| Sticky Note5              | Sticky Note                     | Documentation note              | None                       | None                    | ## 📧 Step 5: Email Delivery<br>Instructions for sending the report by Gmail.                                                            |
| Sticky Note6              | Sticky Note                     | Documentation note              | None                       | None                    | ## 💡 Customization Tips<br>Tips for query optimization, report quality improvement, and email configuration.                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger (n8n built-in)  
   - No special parameters.

2. **Create Set Node for Query Input:**  
   - Name: `query`  
   - Type: Set  
   - Assign a string field named `query` with default value `"n8nとdifyの違い"` (modifiable by user).  
   - Connect output of Manual Trigger to input of this node.

3. **Create Google Gemini Chat Model Node for Query Generation:**  
   - Name: `Google Gemini Chat Model`  
   - Type: Langchain Google Gemini Chat model  
   - Model name: `models/gemini-2.5-flash`  
   - No special options.  
   - Connect output of `query` node to this node as an AI language model input.  
   - Configure Google Palm API credentials with valid account.

4. **Create Structured Output Parser Node:**  
   - Name: `Structured Output Parser`  
   - Type: Langchain Structured Output Parser  
   - Schema: JSON object with properties: `query1` (string), `query2` (string), `query3` (string).  
   - Connect output of `Google Gemini Chat Model` to this parser node.

5. **Create Langchain Agent Node for Query Generation:**  
   - Name: `Query Generator`  
   - Type: Langchain Agent  
   - Prompt: Japanese text instructing to generate three potent search queries from the user question, output queries only.  
   - System message includes detailed instructions and example.  
   - Use the structured output parser above.  
   - Connect parser output to this node.  
   - Connect output of `query` node as input to this agent for raw question.  
   - Connect this node’s output to the Research Agent node (created next).

6. **Create Langchain Agent Node for Research Execution:**  
   - Name: `Research Agent`  
   - Type: Langchain Agent  
   - Prompt: Japanese instructions to use the three queries (`query1`, `query2`, `query3`) to call the Tavily search tool with `search_depth` = "advanced", `max_results`=10, `include_answer`=true, and output full queries plus results.  
   - System message includes stepwise instructions and focus areas for analysis.  
   - Connect output of `Query Generator` to this node.  

7. **Create Google Gemini Chat Model Node for Research Agent:**  
   - Name: `Google Gemini Chat Model1`  
   - Type: Langchain Google Gemini Chat model  
   - Model name: `models/gemini-2.5-flash`  
   - Provide Google Palm API credentials.  
   - Connect as AI language model node inside the Research Agent.

8. **Create Tavily Search Tool Node:**  
   - Name: `Tavily_Search_Tool`  
   - Type: Tavily search tool node  
   - Query parameter set dynamically from AI overrides with the three queries passed from Research Agent.  
   - Options:  
     - `max_results`: 10  
     - `search_depth`: advanced  
     - `include_answer`: advanced  
   - Provide valid Tavily API credentials.  
   - Connect as AI tool node inside Research Agent.

9. **Create Langchain Agent Node for Report Generation:**  
   - Name: `Report Agent`  
   - Type: Langchain Agent  
   - Prompt: Japanese instructions to integrate the research results into a structured, comprehensive HTML report answering the user’s question, removing duplicates and contradictions.  
   - System message includes user question and research results placeholders.  
   - Connect output of `Research Agent` to this node.

10. **Create Google Gemini Chat Model Node for Report Agent:**  
    - Name: `Google Gemini Chat Model2`  
    - Type: Langchain Google Gemini Chat model  
    - Model name: `models/gemini-2.5-pro` (higher token limit and quality)  
    - Provide Google Palm API credentials.  
    - Connect as AI language model node inside Report Agent.

11. **Create Gmail Send Node:**  
    - Name: `Send a message`  
    - Type: Gmail  
    - Parameters:  
      - Subject: expression `={{ $('query').item.json.query }}` (use the original question)  
      - Message: expression `={{ $json.output }}` (HTML report)  
      - Set recipient email address in the “To” field.  
    - Connect output of `Report Agent` to this node.  
    - Configure Gmail OAuth2 credentials.

12. **Connect Workflow:**  
    - Manual Trigger → `query` → `Query Generator` → `Research Agent` → `Report Agent` → `Send a message`.

13. **Optional: Add Sticky Notes** for documentation as per user preference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                             |
|------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------|
| This workflow is designed specifically for Japanese users with Japanese prompts and instructions.                                  | Workflow localization                       |
| Google Gemini API credentials (Google Palm API) and Tavily API credentials are mandatory for the AI and search nodes to function. | API credential setup guides                  |
| Gmail OAuth2 credentials must be configured for sending emails.                                                                     | Gmail credential configuration               |
| For customization, increase `max_results` in Tavily or number of queries for deeper research.                                       | Sticky Note6 customization tips             |
| The workflow leverages advanced AI models Gemini 2.5-flash and Gemini 2.5-pro for different processing steps for quality balance.  | AI model usage notes                         |
| See the sticky notes in the workflow for stepwise guidance and best practices in Japanese.                                          | Inside workflow sticky notes                 |

---

**Disclaimer:** This documentation is generated exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal or protected content. All data processed is legal and publicly available.