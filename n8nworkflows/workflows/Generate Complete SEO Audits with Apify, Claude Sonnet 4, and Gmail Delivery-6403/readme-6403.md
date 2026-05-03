Generate Complete SEO Audits with Apify, Claude Sonnet 4, and Gmail Delivery

https://n8nworkflows.xyz/workflows/generate-complete-seo-audits-with-apify--claude-sonnet-4--and-gmail-delivery-6403


# Generate Complete SEO Audits with Apify, Claude Sonnet 4, and Gmail Delivery

### 1. Workflow Overview

This workflow automates the generation of comprehensive SEO audits by integrating web crawling with Apify, advanced AI-driven content and technical analyses via Claude Sonnet 4 models, and final delivery through Gmail. It is designed for SEO professionals and digital marketers who require detailed, multi-dimensional SEO reports compiled automatically and sent by email.

The workflow logically breaks down into these main blocks:

- **1.1 Input Reception & Initialization:** Triggering the workflow manually and setting initial variables.
- **1.2 Web Crawling (Apify):** Collecting SEO-relevant data from target websites.
- **1.3 AI-Powered SEO Analyses:** Running three specialized AI agents for content audit, technical audit, and strategic SEO analysis, each supported by dedicated language models.
- **1.4 Results Aggregation & Summarization:** Merging individual AI outputs, aggregating them, and generating an executive summary.
- **1.5 Report Generation:** Converting textual summaries into HTML and PDF formats, preparing multiple report components.
- **1.6 Email Delivery:** Merging report files and sending the compiled audit via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initialization

- **Overview:** This block starts the workflow manually and sets up initial variables needed downstream.
- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’` (Manual Trigger)  
  - `Variables` (Set node to initialize variables)  

- **Node Details:**  
  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Function:* Starts the workflow on user command without external input  
    - *Connections:* Outputs to `Variables`  
    - *Failures:* None typical; manual start  

  - **Variables**  
    - *Type:* Set  
    - *Function:* Defines initial parameters such as URLs or configuration flags for crawl and AI steps  
    - *Connections:* Outputs to `Apify Crawl Request`  
    - *Failures:* Expression errors if variables are misconfigured  

#### 1.2 Web Crawling (Apify)

- **Overview:** This block uses Apify’s HTTP Request node to crawl the target website and gather raw SEO data.
- **Nodes Involved:**  
  - `Apify Crawl Request` (HTTP Request)  

- **Node Details:**  
  - **Apify Crawl Request**  
    - *Type:* HTTP Request  
    - *Function:* Sends crawl request to Apify API, retrieves SEO crawl data  
    - *Configuration:* Custom HTTP method, URL, headers, and body as per Apify API specifications (not detailed in JSON)  
    - *Connections:* Outputs to three AI audit nodes (`Enhanced Content Audit`, `Enhanced Technical Audit`, `Strategic SEO Analysis`)  
    - *Error Handling:* Configured to continue on errors (e.g., timeouts, API limits)  
    - *Failures:* HTTP errors, authentication failures, API limits, malformed responses  

#### 1.3 AI-Powered SEO Analyses

- **Overview:** Executes three parallel AI agents, each performing a specific SEO audit: content, technical, and strategic analysis. Each agent runs on Claude Sonnet 4 language models.
- **Nodes Involved:**  
  - `Enhanced Content Audit` (Langchain Agent)  
  - `Enhanced Technical Audit` (Langchain Agent)  
  - `Strategic SEO Analysis` (Langchain Agent)  
  - `Content Audit Model` (Langchain LM Chat Anthropic)  
  - `Technical Audit Model` (Langchain LM Chat Anthropic)  
  - `Strategic Analysis Model` (Langchain LM Chat Anthropic)  

- **Node Details:**  
  - **Enhanced Content Audit**  
    - *Type:* Langchain Agent  
    - *Function:* Uses AI to analyze site content quality and SEO factors  
    - *Configuration:* Interfaces with `Content Audit Model` as language model backend  
    - *Inputs:* Data from `Apify Crawl Request`  
    - *Outputs:* Audit results to `Merge Results` node  
    - *Failures:* AI timeout, model errors, invalid data inputs  

  - **Enhanced Technical Audit**  
    - *Type:* Langchain Agent  
    - *Function:* Performs technical SEO audit (site structure, speed, crawlability)  
    - *Configuration:* Uses `Technical Audit Model`  
    - *Inputs:* Crawl data  
    - *Outputs:* Audit to `Merge Results`  
    - *Failures:* Same as content audit  

  - **Strategic SEO Analysis**  
    - *Type:* Langchain Agent  
    - *Function:* Provides strategic recommendations for SEO improvements  
    - *Configuration:* Uses `Strategic Analysis Model`  
    - *Inputs:* Crawl data  
    - *Outputs:* Audit to `Merge Results`  
    - *Failures:* Same AI-related failure modes  

  - **Content Audit Model, Technical Audit Model, Strategic Analysis Model**  
    - *Type:* Langchain Language Model Chat (Anthropic Claude Sonnet 4)  
    - *Function:* Provide AI natural language processing capabilities for corresponding audits  
    - *Connections:* Each linked as language model backend to respective Langchain Agent nodes  
    - *Failures:* Authentication errors, API rate limits, model unavailability  

#### 1.4 Results Aggregation & Summarization

- **Overview:** Combines individual audit outputs, aggregates data, and generates a high-level executive summary.
- **Nodes Involved:**  
  - `Merge Results` (Merge node)  
  - `Aggregate Reports` (Aggregate node)  
  - `Executive Summary Generator` (Langchain Agent)  
  - `Summary Model` (Langchain LM Chat Anthropic)  

- **Node Details:**  
  - **Merge Results**  
    - *Type:* Merge  
    - *Function:* Combines three audit outputs into a single data stream  
    - *Inputs:* From Enhanced Content Audit, Enhanced Technical Audit, Strategic SEO Analysis  
    - *Outputs:* To `Aggregate Reports`  
    - *Failures:* Data mismatch, empty inputs  

  - **Aggregate Reports**  
    - *Type:* Aggregate  
    - *Function:* Consolidates merged data into summarized form  
    - *Outputs:* To `Executive Summary Generator`  
    - *Failures:* Empty data, aggregation errors  

  - **Executive Summary Generator**  
    - *Type:* Langchain Agent  
    - *Function:* Creates a concise executive summary from aggregated audit data  
    - *Uses:* `Summary Model` as language model backend  
    - *Outputs:* To markdown and HTML conversion nodes  
    - *Failures:* AI model errors, input formatting errors  

  - **Summary Model**  
    - *Type:* Langchain Language Model Chat (Anthropic Claude Sonnet 4)  
    - *Function:* Provides AI summarization capabilities  
    - *Connections:* Backend for `Executive Summary Generator`  
    - *Failures:* API errors, rate limits  

#### 1.5 Report Generation

- **Overview:** Converts AI outputs into multiple report formats (Markdown, HTML, PDF) and prepares them for email delivery.
- **Nodes Involved:**  
  - `Summary to HTML`, `Summary to HTML1` (Markdown and HTML nodes)  
  - `Technical Audit to PDF` (Markdown node)  
  - `Content Audit to HTML` (Markdown node)  
  - `SEO Analysis to HTML` (Markdown node)  
  - `Generate HTML template1`, `Generate HTML template2`, `Generate HTML template3` (HTML nodes)  
  - `Convert to HTML`, `Convert to HTML1`, `Convert to HTML2`, `Convert to HTML3` (ConvertToFile nodes)  
  - `Merge` (Merge node)  

- **Node Details:**  
  - **Markdown Nodes** (`Technical Audit to PDF`, `Content Audit to HTML`, `SEO Analysis to HTML`, `Summary to HTML`)  
    - *Type:* Markdown  
    - *Function:* Converts AI textual data into Markdown format ready for further processing  
    - *Inputs:* From `Executive Summary Generator` and audit outputs  
    - *Outputs:* To HTML template generators or direct HTML conversion  

  - **HTML Nodes** (`Generate HTML template1`, `Generate HTML template2`, `Generate HTML template3`)  
    - *Type:* HTML  
    - *Function:* Applies HTML templates to Markdown content, formatting reports for readability and visual appeal  
    - *Outputs:* To `Convert to HTML` nodes  

  - **Convert to HTML Nodes** (`Convert to HTML`, `Convert to HTML1`, `Convert to HTML2`, `Convert to HTML3`)  
    - *Type:* ConvertToFile  
    - *Function:* Converts HTML content to file format (e.g., PDF or HTML files) for attachment or delivery  
    - *Outputs:* Feed into final `Merge` node  

  - **Merge**  
    - *Type:* Merge  
    - *Function:* Combines all report files into a single consolidated output for emailing  
    - *Outputs:* To `Aggregate` node  

  - *Failures Common to this Block:* Conversion errors, file size limits, template syntax issues  

#### 1.6 Email Delivery

- **Overview:** Aggregates all report files and sends the complete SEO audit report via Gmail.
- **Nodes Involved:**  
  - `Aggregate` (Aggregate node)  
  - `Gmail` (Gmail node)  

- **Node Details:**  
  - **Aggregate**  
    - *Type:* Aggregate  
    - *Function:* Collects all file outputs into one payload  
    - *Outputs:* To `Gmail` node  
    - *Failures:* Missing files, aggregation errors  

  - **Gmail**  
    - *Type:* Gmail node  
    - *Function:* Sends email with SEO audit report attached  
    - *Configuration:* Requires Gmail OAuth2 credentials properly set up  
    - *Failures:* Authentication errors, quota exceeded, invalid email addresses  

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                     | Input Node(s)                                   | Output Node(s)                            | Sticky Note                  |
|----------------------------|----------------------------------|-----------------------------------|------------------------------------------------|-------------------------------------------|------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Workflow start                    | —                                              | Variables                                  |                              |
| Variables                  | Set                              | Initialize variables              | When clicking ‘Execute workflow’                | Apify Crawl Request                        |                              |
| Apify Crawl Request        | HTTP Request                     | Crawl site data                  | Variables                                       | Enhanced Content Audit, Enhanced Technical Audit, Strategic SEO Analysis |                              |
| Enhanced Content Audit     | Langchain Agent                  | Content SEO audit                | Apify Crawl Request                             | Merge Results                              |                              |
| Enhanced Technical Audit   | Langchain Agent                  | Technical SEO audit              | Apify Crawl Request                             | Merge Results                              |                              |
| Strategic SEO Analysis     | Langchain Agent                  | Strategic SEO analysis           | Apify Crawl Request                             | Merge Results                              |                              |
| Content Audit Model        | Langchain LM Chat Anthropic      | AI model backend for content audit | —                                              | Enhanced Content Audit                     |                              |
| Technical Audit Model      | Langchain LM Chat Anthropic      | AI model backend for technical audit | —                                              | Enhanced Technical Audit                   |                              |
| Strategic Analysis Model   | Langchain LM Chat Anthropic      | AI model backend for strategic analysis | —                                              | Strategic SEO Analysis                     |                              |
| Merge Results              | Merge                            | Combine audit outputs            | Enhanced Content Audit, Enhanced Technical Audit, Strategic SEO Analysis | Aggregate Reports                          |                              |
| Aggregate Reports          | Aggregate                       | Summarize merged audits          | Merge Results                                   | Executive Summary Generator                |                              |
| Executive Summary Generator | Langchain Agent                  | Generate executive summary       | Aggregate Reports                               | Summary to HTML1, Technical Audit to PDF, Content Audit to HTML, SEO Analysis to HTML |                              |
| Summary Model              | Langchain LM Chat Anthropic      | AI model backend for summary     | —                                              | Executive Summary Generator                |                              |
| Summary to HTML            | Markdown                        | Convert summary to Markdown      | Executive Summary Generator                      | Convert to HTML                            |                              |
| Summary to HTML1           | Markdown                        | Pass-through/formatting step     | Summary to HTML                                 | —                                         |                              |
| Technical Audit to PDF     | Markdown                        | Prepare technical audit report   | Executive Summary Generator                      | Generate HTML template1                    |                              |
| Content Audit to HTML      | Markdown                        | Prepare content audit report     | Executive Summary Generator                      | Generate HTML template2                    |                              |
| SEO Analysis to HTML       | Markdown                        | Prepare strategic audit report   | Executive Summary Generator                      | Generate HTML template3                    |                              |
| Generate HTML template1    | HTML                            | Apply HTML template to technical audit | Technical Audit to PDF                          | Convert to HTML1                           |                              |
| Generate HTML template2    | HTML                            | Apply HTML template to content audit | Content Audit to HTML                           | Convert to HTML2                           |                              |
| Generate HTML template3    | HTML                            | Apply HTML template to SEO analysis | SEO Analysis to HTML                            | Convert to HTML3                           |                              |
| Convert to HTML            | ConvertToFile                   | Convert summary HTML to file     | Summary to HTML1                                | Merge                                     |                              |
| Convert to HTML1           | ConvertToFile                   | Convert technical audit HTML     | Generate HTML template1                         | Merge                                     |                              |
| Convert to HTML2           | ConvertToFile                   | Convert content audit HTML       | Generate HTML template2                         | Merge                                     |                              |
| Convert to HTML3           | ConvertToFile                   | Convert SEO analysis HTML        | Generate HTML template3                         | Merge                                     |                              |
| Merge                      | Merge                            | Combine all report files         | Convert to HTML, Convert to HTML1, Convert to HTML2, Convert to HTML3 | Aggregate                                  |                              |
| Aggregate                  | Aggregate                       | Aggregate files for sending      | Merge                                           | Gmail                                     |                              |
| Gmail                      | Gmail                           | Send audit report by email       | Aggregate                                        | —                                         |                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: Start workflow manually.

2. **Create Set Node for Variables:**  
   - Name: `Variables`  
   - Configure variables such as target URLs, API keys, or crawl parameters.  
   - Connect output from manual trigger to this node.

3. **Create HTTP Request Node for Apify Crawl:**  
   - Name: `Apify Crawl Request`  
   - Configure to invoke Apify's SEO crawling API with proper method, URL, headers, and body.  
   - Set error handling to continue on error.  
   - Connect output from `Variables` to this node.

4. **Create Langchain LM Chat Anthropic Nodes for AI Models:**  
   - Names: `Content Audit Model`, `Technical Audit Model`, `Strategic Analysis Model`, `Summary Model`  
   - Configure each with Claude Sonnet 4 credentials (Anthropic API key).  
   - No direct inputs; they serve as backends for agent nodes.

5. **Create Langchain Agent Nodes:**  
   - Names: `Enhanced Content Audit`, `Enhanced Technical Audit`, `Strategic SEO Analysis`, `Executive Summary Generator`  
   - Configure each to use the corresponding Langchain LM Chat Anthropic node as the language model.  
   - Connect output of `Apify Crawl Request` to the first three agent nodes.  
   - Connect output of `Aggregate Reports` to `Executive Summary Generator`.

6. **Create Merge Node for Audit Results:**  
   - Name: `Merge Results`  
   - Configure to merge three inputs (content, technical, and strategic audit results).  
   - Connect outputs of the three audit agents to this node.

7. **Create Aggregate Node:**  
   - Name: `Aggregate Reports`  
   - Connect output from `Merge Results` to `Aggregate Reports`.  
   - Connect output from `Aggregate Reports` to `Executive Summary Generator`.

8. **Create Markdown Nodes for Report Preparation:**  
   - Names: `Summary to HTML`, `Summary to HTML1`, `Technical Audit to PDF`, `Content Audit to HTML`, `SEO Analysis to HTML`  
   - Connect outputs from `Executive Summary Generator` accordingly.

9. **Create HTML Nodes for Templating:**  
   - Names: `Generate HTML template1`, `Generate HTML template2`, `Generate HTML template3`  
   - Connect respective Markdown nodes to these.

10. **Create ConvertToFile Nodes:**  
    - Names: `Convert to HTML`, `Convert to HTML1`, `Convert to HTML2`, `Convert to HTML3`  
    - Connect each HTML node to corresponding ConvertToFile node.

11. **Create Merge Node for Report Files:**  
    - Name: `Merge`  
    - Connect all ConvertToFile nodes to this Merge node.

12. **Create Aggregate Node for Email:**  
    - Name: `Aggregate`  
    - Connect output of `Merge` node to `Aggregate`.

13. **Create Gmail Node:**  
    - Name: `Gmail`  
    - Configure with OAuth2 credentials for Gmail.  
    - Compose email with attachments from `Aggregate` node.  
    - Connect output from `Aggregate` to Gmail node.

14. **Connect the Flow:**  
    - Manual Trigger → Variables → Apify Crawl Request → (Enhanced Content Audit, Enhanced Technical Audit, Strategic SEO Analysis) → Merge Results → Aggregate Reports → Executive Summary Generator → Markdown nodes → HTML templates → ConvertToFile nodes → Merge → Aggregate → Gmail.

15. **Credentials Setup:**  
    - Anthropic Claude Sonnet 4 API key for all Langchain LM Chat Anthropic nodes.  
    - Apify API Key for HTTP Request node if required.  
    - Gmail OAuth2 for Gmail node.

16. **Defaults and Constraints:**  
    - Set timeouts and error handling especially on API calls.  
    - Ensure correct MIME types for file conversions and email attachments.  
    - Validate URLs and email addresses in variables.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                   | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow integrates Apify crawling with Anthropic Claude Sonnet 4 via n8n’s Langchain nodes for advanced SEO auditing automation.                                            | Main design principle of the workflow                                                             |
| Gmail node requires OAuth2 and appropriate Gmail API scopes enabled.                                                                                                          | Gmail Deliverability                                                                               |
| Anthropic Claude Sonnet 4 is a powerful LLM; rate limits and token costs should be monitored.                                                                                  | AI model usage considerations                                                                      |
| Apify HTTP Request node is configured to continue on errors to prevent workflow interruption in case of crawl failures or API limits.                                        | Robustness in data crawling                                                                        |
| Markdown to HTML and PDF conversion nodes rely on well-formed Markdown input; malformed AI outputs might cause rendering issues.                                              | Data formatting best practices                                                                     |
| For optimal output, customize the HTML templates (`Generate HTML template1/2/3`) to match branding and report style.                                                          | Report presentation customization                                                                 |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.