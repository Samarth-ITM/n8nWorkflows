Generate Fact-Checked Research Reports with Llama AI and Web Search

https://n8nworkflows.xyz/workflows/generate-fact-checked-research-reports-with-llama-ai-and-web-search-11116


# Generate Fact-Checked Research Reports with Llama AI and Web Search

---

### 1. Workflow Overview

This workflow titled **"Generate Fact-Checked Research Reports with Llama AI and Web Search"** orchestrates an autonomous AI-powered research system. It accepts a user-submitted research topic and parameters, then leverages multiple AI agents and web search to generate a comprehensive, well-structured, and fact-checked research report. The output includes rich citations and is formatted in multiple user-selectable styles.

**Target Use Cases:**  
- Automated research report generation for academic, business, or technical topics  
- Content creation with verified, up-to-date sources  
- Streamlining team research processes with AI collaboration  

**Logical Blocks:**

- **1.1 Input Reception**  
  Collect user input through a web form and parse the data for downstream use.

- **1.2 Research Query Planning**  
  Use an AI research agent to generate targeted search queries based on the topic and optional context.

- **1.3 Web Search and Data Aggregation**  
  Perform web searches with SerpAPI for each generated query, then merge and aggregate results.

- **1.4 AI Content Generation and Verification**  
  Deploy a Writer Agent to draft content, a Fact-Checker Agent to verify facts, and an Editor Agent to improve text quality.

- **1.5 Final Review and Report Consolidation**  
  Use a Project Manager Agent to combine all outputs into a final, citation-backed report.

- **1.6 Output Formatting and Delivery**  
  Convert the final research content into well-formatted HTML and respond to the user request.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block captures user input via a form submission and parses it into structured data for the workflow.

**Nodes Involved:**  
- Form Trigger  
- Parse Form Input  

**Node Details:**  

- **Form Trigger**  
  - Type: Form Trigger (Webhook-based)  
  - Role: Entry point, exposes a form with fields: Research Topic, Research Depth, Output Format, Additional Context  
  - Config: Path unique webhook ID, form with required and optional fields, response mode set to "responseNode"  
  - Inputs: None (triggered by form submission)  
  - Outputs: Parsed form data JSON  
  - Potential Failures: Webhook not reachable, invalid form data submission  

- **Parse Form Input**  
  - Type: Code Node  
  - Role: Transforms raw form data into structured JSON with fields: query, depth, format, context, timestamp, sessionId  
  - Key Expressions: Uses `$input.item.json` to extract form fields; generates timestamp and unique sessionId combining workflow ID and timestamp  
  - Inputs: Output of Form Trigger  
  - Outputs: Structured object for downstream nodes  
  - Potential Failures: Missing expected form fields, expression errors  

---

#### 2.2 Research Query Planning

**Overview:**  
The AI agent formulates a detailed research plan by generating 5-7 specific web search queries tailored to the user's topic and context.

**Nodes Involved:**  
- Research Agent - Plan  
- Extract Search Queries  

**Node Details:**  

- **Research Agent - Plan**  
  - Type: HTTP Request (OpenAI-compatible API via Groq)  
  - Role: Sends system and user prompts to Llama 3.3 model to generate a JSON array of search queries  
  - Config: POST request to Groq API endpoint with JSON body containing model, messages, temperature, max tokens; uses Groq API credentials  
  - Inputs: Parsed form data (query, context)  
  - Outputs: AI-generated research plan text  
  - Potential Failures: API authentication errors, rate limiting, malformed responses, timeout  

- **Extract Search Queries**  
  - Type: Code Node  
  - Role: Parses response text from Research Agent into a JSON array of queries; supports JSON parsing fallback to text extraction  
  - Key Expressions: Attempts JSON.parse; fallback parses lines and extracts query strings  
  - Inputs: Research Agent output  
  - Outputs: Array of individual search query JSON objects, each with query text, original topic, sessionId  
  - Potential Failures: Parsing errors, empty or invalid response content  

---

#### 2.3 Web Search and Data Aggregation

**Overview:**  
Performs real-time web searches for each query, merges results, and aggregates relevant source data for later AI processing.

**Nodes Involved:**  
- SERP Search  
- Merge Research  
- Aggregate Research  

**Node Details:**  

- **SERP Search**  
  - Type: HTTP Request  
  - Role: Queries SerpAPI with search query string, limiting to 5 results per query  
  - Config: Uses SerpAPI credentials; query parameter from extracted queries; content-type application/json  
  - Inputs: Extract Search Queries output (each query processed individually)  
  - Outputs: JSON search results for each query  
  - Potential Failures: API key invalid/expired, network errors, quota exceeded  

- **Merge Research**  
  - Type: Merge Node  
  - Role: Combines parallel search results into one stream for aggregation  
  - Config: Multiplex mode to collect all incoming items  
  - Inputs: SERP Search outputs  
  - Outputs: Combined search results array  
  - Potential Failures: Merge conflicts if inputs malformed  

- **Aggregate Research**  
  - Type: Code Node  
  - Role: Collates all search results into a structured research object including topic, context, and array of source entries (title, snippet, link)  
  - Inputs: Merged search results, form input data  
  - Outputs: Aggregated research data JSON with sourceCount  
  - Potential Failures: Missing organic results, inconsistent data structure  

---

#### 2.4 AI Content Generation and Verification

**Overview:**  
Sequentially generates written content, fact-checks it against sources, and refines the text for quality.

**Nodes Involved:**  
- Writer Agent  
- Fact-Checker Agent  
- Editor Agent  
- Merge All Agents  

**Node Details:**  

- **Writer Agent**  
  - Type: LangChain Chain LLM node  
  - Role: Creates initial comprehensive content based on aggregated research sources and user format selection (e.g., Executive Summary, Blog Article)  
  - Config: Uses Llama 3.3 model via Groq; prompt includes research topic, sources, and formatting instructions  
  - Inputs: Aggregate Research output  
  - Outputs: Draft text content  
  - Potential Failures: API errors, incomplete source data  

- **Fact-Checker Agent**  
  - Type: LangChain Chain LLM node  
  - Role: Verifies statements in written content against source material; outputs fact-check report with corrections  
  - Config: Prompt instructs to highlight inaccuracies and unsupported claims using sources from Aggregate Research  
  - Inputs: Writer Agent content and Aggregate Research data  
  - Outputs: Fact-check report text  
  - Potential Failures: Fact-check inaccuracies, API errors  

- **Editor Agent**  
  - Type: LangChain Chain LLM node  
  - Role: Improves draft content clarity, grammar, readability, and tone  
  - Config: Prompt to enhance professional tone and structure  
  - Inputs: Writer Agent content  
  - Outputs: Edited improved text  
  - Potential Failures: Over-editing or distortion of original meaning  

- **Merge All Agents**  
  - Type: Merge Node  
  - Role: Combines outputs of Fact-Checker and Editor agents for final review  
  - Config: Multiplex mode  
  - Inputs: Fact-Checker Agent, Editor Agent outputs  
  - Outputs: Combined agent outputs for final consolidation  
  - Potential Failures: Merging delays or conflicts  

---

#### 2.5 Final Review and Report Consolidation

**Overview:**  
A Project Manager AI agent synthesizes all prior outputs into a single polished, citation-backed final research report.

**Nodes Involved:**  
- PM Agent - Final Review1  

**Node Details:**  

- **PM Agent - Final Review1**  
  - Type: LangChain Chain LLM node  
  - Role: Acts as project manager to merge original topic, written content, fact-check report, edited version, and source citations into one cohesive final deliverable  
  - Config: Uses Llama 3.3 model; prompt emphasizes consolidation and citation inclusion  
  - Inputs: Aggregate Research, Writer Agent, Fact-Checker Agent, Editor Agent outputs  
  - Outputs: Final consolidated report text  
  - Potential Failures: Over-summarization, missing citations, API errors  

---

#### 2.6 Output Formatting and Delivery

**Overview:**  
Prepares the final report for user delivery by converting markdown-like text to clean HTML and responding to the original form submission.

**Nodes Involved:**  
- Return Results  
- Code  

**Node Details:**  

- **Return Results**  
  - Type: Respond to Webhook  
  - Role: Sends the final HTML report back to the user with appropriate headers (Content-Type: text/html)  
  - Inputs: Output of PM Agent - Final Review1  
  - Outputs: HTTP response with formatted report  
  - Potential Failures: Client disconnect, response timeouts  

- **Code**  
  - Type: Code Node  
  - Role: Converts markdown-ish text from final report into sanitized HTML wrapped in full HTML document structure, ready for PDF conversion or display  
  - Key Functions:  
    - Replaces simple markdown bold (**text**) with `<strong>`  
    - Converts paragraphs and line breaks  
    - Wraps in standard HTML5 with CSS styling and footer timestamp  
    - Encodes HTML as base64 binary for further processing  
  - Inputs: Final report text JSON  
  - Outputs: JSON with `.html` and binary base64 of HTML file  
  - Potential Failures: Malformed input text, encoding issues  

---

### 3. Summary Table

| Node Name             | Node Type                         | Functional Role                         | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                   |
|-----------------------|----------------------------------|---------------------------------------|--------------------------|----------------------------|-----------------------------------------------------------------------------------------------|
| Form Trigger          | Form Trigger                     | Receive user research input            | None                     | Parse Form Input            | ## INPUT                                                                                        |
| Parse Form Input       | Code                            | Parse and structure form data          | Form Trigger             | Merge Research, Research Agent - Plan | ## INPUT                                                                                        |
| Research Agent - Plan  | HTTP Request (Groq API)          | Generate targeted search queries       | Parse Form Input         | Extract Search Queries      | ## RESEARCH AGENT                                                                              |
| Extract Search Queries | Code                            | Parse AI response into search queries  | Research Agent - Plan    | SERP Search                | ## RESEARCH AGENT                                                                              |
| SERP Search            | HTTP Request (SerpAPI)           | Perform live web searches               | Extract Search Queries   | Merge Research              | ## FINALIZING RESEARCH                                                                         |
| Merge Research         | Merge Node                      | Combine search results                  | SERP Search, Parse Form Input | Aggregate Research          | ## FINALIZING RESEARCH                                                                         |
| Aggregate Research     | Code                            | Collect and structure all source data  | Merge Research           | Writer Agent               | ## FINALIZING RESEARCH                                                                         |
| Writer Agent           | LangChain Chain LLM             | Draft initial content from research    | Aggregate Research       | Fact-Checker Agent, Editor Agent | ## AI AGENTS                                                                                   |
| Fact-Checker Agent     | LangChain Chain LLM             | Verify facts against sources            | Writer Agent             | Merge All Agents            | ## AI AGENTS                                                                                   |
| Editor Agent           | LangChain Chain LLM             | Improve clarity and style               | Writer Agent             | Merge All Agents            | ## AI AGENTS                                                                                   |
| Merge All Agents       | Merge Node                      | Combine fact-check and editing outputs | Fact-Checker Agent, Editor Agent | PM Agent - Final Review1    | ## AI AGENTS                                                                                   |
| PM Agent - Final Review1 | LangChain Chain LLM           | Synthesize final report with citations | Merge All Agents         | Return Results              | ## OUTPUT                                                                                     |
| Return Results         | Respond to Webhook              | Return final HTML report to user       | PM Agent - Final Review1 | Code                       | ## OUTPUT                                                                                     |
| Code                   | Code                            | Convert markdown text to HTML           | Return Results           | None                       | ## OUTPUT                                                                                     |
| Sticky Note            | Sticky Note                    | Visual notes for workflow sections     | None                     | None                       | # AGENTIC AI (global title)                                                                   |
| Sticky Note1           | Sticky Note                    | Section header: RESEARCH AGENT          | None                     | None                       | ## RESEARCH AGENT                                                                             |
| Sticky Note2           | Sticky Note                    | Section header: FINALIZING RESEARCH     | None                     | None                       | ## FINALIZING RESEARCH                                                                        |
| Sticky Note3           | Sticky Note                    | Section header: INPUT                    | None                     | None                       | ## INPUT                                                                                     |
| Sticky Note4           | Sticky Note                    | Section header: OUTPUT                   | None                     | None                       | ## OUTPUT                                                                                    |
| Sticky Note5           | Sticky Note                    | Section header: AI AGENTS                | None                     | None                       | ## AI AGENTS                                                                                |
| Sticky Note6           | Sticky Note                    | Detailed project description, tutorial link, credits | None                     | None                       | # 🧠 Agentic AI Research System  \n\n🎥 Watch the Tutorial: https://youtu.be/GUfUzls_9yI?si=XgjiMw9tysyNb6TQ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Form Trigger" Node**  
   - Type: Form Trigger  
   - Configure webhook path (unique)  
   - Form Title: "AI Research Team"  
   - Fields:  
     - Research Topic (text, required)  
     - Research Depth (dropdown: Quick (5 min), Standard (10 min), Deep (15 min), required)  
     - Output Format (dropdown: Executive Summary, Detailed Report, Blog Article, required)  
     - Additional Context (textarea, optional)  
   - Response Mode: responseNode  

2. **Create "Parse Form Input" Node**  
   - Type: Code Node  
   - JavaScript to extract fields from form input JSON: `query`, `depth`, `format`, `context` (default empty), `timestamp` (ISO), `sessionId` (workflow ID + timestamp)  
   - Connect output of Form Trigger to this node  

3. **Create "Research Agent - Plan" Node**  
   - Type: HTTP Request (POST)  
   - URL: `https://api.groq.com/openai/v1/chat/completions`  
   - Authentication: Groq API credentials (set up with API key)  
   - Body (JSON): Specify model "llama-3.3-70b-versatile", system prompt instructing to generate 5-7 search queries in JSON array format, user prompt includes research topic and optional context  
   - Headers: Content-Type application/json  
   - Connect output of Parse Form Input to this node  

4. **Create "Extract Search Queries" Node**  
   - Type: Code Node  
   - JavaScript to parse AI response text into array of queries, fallback to line extraction if JSON parsing fails  
   - Each output item: JSON with `searchQuery`, `originalTopic`, `sessionId`  
   - Connect output of Research Agent - Plan to this node  

5. **Create "SERP Search" Node**  
   - Type: HTTP Request  
   - URL: `https://serpapi.com/search.json`  
   - Authentication: SerpAPI credentials (API key)  
   - Parameters: `q` = `={{ $json.searchQuery }}`, `num` = 5  
   - Headers: Content-Type application/json  
   - Connect output of Extract Search Queries to this node  

6. **Create "Merge Research" Node**  
   - Type: Merge Node  
   - Mode: Combine  
   - Combination Mode: Multiplex  
   - Connect outputs of SERP Search and Parse Form Input (for form data pass-through) to this node  

7. **Create "Aggregate Research" Node**  
   - Type: Code Node  
   - JavaScript to aggregate all SERP results into a research object with topic, context, and array of sources (title, snippet, link)  
   - Connect output of Merge Research to this node  

8. **Create "Writer Agent" Node**  
   - Type: LangChain Chain LLM  
   - Model: Llama 3.3 (Groq API credentials)  
   - Prompt: Input the research topic, sources, and request comprehensive content in user-selected format (from aggregated research)  
   - Connect output of Aggregate Research to this node  

9. **Create "Fact-Checker Agent" Node**  
   - Type: LangChain Chain LLM  
   - Prompt: Verify claims in written content against source material, provide fact-check report  
   - Inputs: Writer Agent output and Aggregate Research output (for sources)  
   - Connect output of Writer Agent to this node  

10. **Create "Editor Agent" Node**  
    - Type: LangChain Chain LLM  
    - Prompt: Improve clarity, grammar, readability, tone of written content  
    - Input: Writer Agent output  
    - Connect output of Writer Agent to this node  

11. **Create "Merge All Agents" Node**  
    - Type: Merge Node  
    - Mode: Combine  
    - Combination Mode: Multiplex  
    - Connect outputs of Fact-Checker Agent and Editor Agent to this node  

12. **Create "PM Agent - Final Review1" Node**  
    - Type: LangChain Chain LLM  
    - Prompt: Merge original topic, written content, fact-check report, edited version, and sources into final consolidated output with citations  
    - Inputs: Merge All Agents output, Aggregate Research output, Writer Agent output, Fact-Checker Agent output, Editor Agent output  
    - Connect output of Merge All Agents to this node  

13. **Create "Return Results" Node**  
    - Type: Respond to Webhook  
    - Configure response headers: Content-Type = text/html  
    - Connect output of PM Agent - Final Review1 to this node  

14. **Create "Code" Node**  
    - Type: Code Node  
    - JavaScript to convert markdown-like final report text into sanitized HTML wrapped in full HTML5 document with styling and footer timestamp  
    - Outputs JSON with `.html` field and binary base64 for HTML file attachment  
    - Connect output of Return Results to this node  

15. **Set up Credentials**  
    - Groq API credentials: API key for Llama 3.3 model access  
    - SerpAPI credentials: API key for web search  

16. **Connect all nodes according to the flow described above.** Ensure proper data passing and error handling where possible.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Build a complete autonomous research engine inside n8n that behaves like a small AI research team. Uses Groq (Llama 3.3), LangChain Agents, and SerpAPI to collect information, analyze data, fact-check, and generate reports. | Project description in Sticky Note6                              |
| Tutorial video for setup and usage available: [Watch on YouTube](https://youtu.be/GUfUzls_9yI?si=XgjiMw9tysyNb6TQ)                                                                                                              | YouTube tutorial linked in Sticky Note6                          |
| Created by Muhammad Shaheer. Contact: shaheerawan001@gmail.com. LinkedIn: www.linkedin.com/in/muhammad-shaheer-898513192                                                                                                        | Credits from Sticky Note6                                        |
| Use cases: research automation for teams and freelancers, long-form content creation with verified sources, knowledge gathering for business, academic, and technical projects.                                                  | Project description in Sticky Note6                              |
| Setup requires less than 10 minutes after adding Groq API key and SerpAPI key.                                                                                                                                                    | Project description in Sticky Note6                              |

---

**Disclaimer:**  
This document is derived exclusively from an automated n8n workflow. It complies fully with applicable content policies and contains no illegal or protected material. All data processed is legal and public.

---