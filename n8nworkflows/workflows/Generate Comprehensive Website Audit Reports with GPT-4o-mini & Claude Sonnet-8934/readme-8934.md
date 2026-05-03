Generate Comprehensive Website Audit Reports with GPT-4o-mini & Claude Sonnet

https://n8nworkflows.xyz/workflows/generate-comprehensive-website-audit-reports-with-gpt-4o-mini---claude-sonnet-8934


# Generate Comprehensive Website Audit Reports with GPT-4o-mini & Claude Sonnet

---
### 1. Workflow Overview

This workflow automates the generation of a comprehensive website audit report by orchestrating multiple AI agents specialized in distinct audit domains: Technical SEO, Content SEO, and Conversion Rate Optimization (CRO). It accepts a website URL and recipient email via a webhook, scrapes the website content, and sends the content through three specialized AI agents for analysis. Their outputs are merged, aggregated, refined into a professional consulting-style summary report, converted to HTML, and then emailed to the client.

**Target Use Cases:**  
- Digital marketing agencies providing automated website audit reports  
- SEO consultants delivering detailed content and technical analysis  
- Conversion optimization experts offering actionable UX insights to clients  
- Businesses seeking an AI-powered, end-to-end website evaluation and reporting tool

**Logical Blocks:**  
- **1.1 Input Reception & Data Scraping**  
  Receives webhook payload, extracts URL and email, scrapes the website HTML content.  
- **1.2 AI Audit Agents**  
  Three parallel AI agents analyze the website: Technical SEO, SEO Content, and CRO. Each uses OpenAI GPT-4o-mini with specialized prompts.  
- **1.3 Results Aggregation & Editorial Synthesis**  
  Merges AI outputs, aggregates data, and uses an Anthropic Claude Sonnet model to create a polished consulting-style report.  
- **1.4 Report Formatting and Delivery**  
  Converts the report from Markdown to HTML and sends it via Gmail to the specified email address.  
- **1.5 Webhook Response**  
  Sends an empty response to acknowledge webhook receipt.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Scraping

- **Overview:**  
  This block accepts incoming HTTP POST requests containing the target website URL and recipient email. It then scrapes the full HTML content of the website for analysis.

- **Nodes Involved:**  
  - Webhook Webpage Audit AI Agent  
  - Scrape Website

- **Node Details:**

  - **Webhook Webpage Audit AI Agent**  
    - *Type:* Webhook  
    - *Role:* Entry point capturing URL and email from external requests  
    - *Configuration:* HTTP POST on path `/webpage-audit-ai-agent`, raw body enabled  
    - *Key Variables:* Extracts payload data: `.body.payload.data.URL` and `.body.payload.data.Email`  
    - *Connections:* Outputs to Scrape Website node  
    - *Potential Failures:* Invalid JSON payload, missing URL or email fields, request timeout  

  - **Scrape Website**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves raw HTML content of the provided URL  
    - *Configuration:* URL dynamically set from webhook input: `={{ $json.body.payload.data.URL }}`  
    - *Input:* Webhook data  
    - *Output:* Website HTML content as string in `.data` or `.body` field, passed on to AI agents  
    - *Potential Failures:* Network issues, HTTP errors (404, 500), invalid URL, site blocking requests

---

#### 2.2 AI Audit Agents

- **Overview:**  
  This parallel block runs three AI agents, each specialized in auditing a different aspect of the website: technical SEO, content SEO, and CRO/UX. Each agent uses GPT-4o-mini with customized system prompts to restrict and focus their analysis domain.

- **Nodes Involved:**  
  - IT Technical AI Agent  
  - SEO AI Agent  
  - CRO AI Agent  
  - OpenAI Chat Model2 (supports IT Technical AI Agent)  
  - OpenAI Chat Model1 (supports SEO AI Agent)  
  - OpenAI Chat Model (supports CRO AI Agent)

- **Node Details:**

  - **IT Technical AI Agent**  
    - *Type:* Langchain Agent Node  
    - *Role:* Performs on-page technical SEO and security audit on raw HTML  
    - *Configuration:* Receives HTML from Scrape Website, prompt instructs to focus strictly on technical SEO factors only  
    - *Input:* Website HTML (`{{ $json.data }}`)  
    - *Output:* Structured audit report in Markdown + HTML  
    - *AI Model:* OpenAI GPT-4o-mini via OpenAI Chat Model2  
    - *Potential Failures:* Model misinterpretation, input length limits, API errors  

  - **SEO AI Agent**  
    - *Type:* Langchain Agent Node  
    - *Role:* Performs content quality and SEO content strategy audit  
    - *Configuration:* Focused prompt ignoring technical and conversion factors, analyzing keywords, readability, engagement  
    - *Input:* Website HTML content (`{{ $json.data }}`)  
    - *Output:* SEO content audit report in structured Markdown + HTML  
    - *AI Model:* OpenAI GPT-4o-mini via OpenAI Chat Model1  
    - *Potential Failures:* Model hallucinations, input size limits, API failures  

  - **CRO AI Agent**  
    - *Type:* Langchain Agent Node  
    - *Role:* Conversion Rate Optimization and UX audit, focusing on marketing psychology and UX barriers  
    - *Configuration:* Prompt specifically excludes SEO and technical factors  
    - *Input:* Website HTML content (`{{ $json.data }}`)  
    - *Output:* CRO audit report in Markdown format  
    - *AI Model:* OpenAI GPT-4o-mini via OpenAI Chat Model  
    - *Potential Failures:* Similar to above, plus possible ambiguity in UX evaluation  

  - **OpenAI Chat Model, OpenAI Chat Model1, OpenAI Chat Model2**  
    - *Type:* Language Model node (OpenAI GPT-4o-mini)  
    - *Role:* Provide language model inference for each agent with specific prompts  
    - *Configuration:* Use sensitive OpenAI credentials, caching enabled for efficiency  
    - *Connections:* Linked to respective AI Agent nodes via `ai_languageModel` input  
    - *Potential Failures:* API rate limits, credential errors, model unavailability

---

#### 2.3 Results Aggregation & Editorial Synthesis

- **Overview:**  
  Aggregates the three AI audit reports into one combined dataset, invokes an advanced AI editor agent (Anthropic Claude Sonnet) to synthesize a polished consulting-style summary report with business impact framing and actionable next steps.

- **Nodes Involved:**  
  - Merge  
  - Aggregate  
  - Editor in Chef AI Agent  
  - Anthropic Chat Model

- **Node Details:**

  - **Merge**  
    - *Type:* Merge node  
    - *Role:* Combines outputs from the three AI audit agents into a single stream  
    - *Configuration:* Configured for 3 inputs, merging them with `alwaysOutputData` enabled  
    - *Input:* Outputs from IT Technical AI Agent, SEO AI Agent, CRO AI Agent  
    - *Output:* Unified data array passed to Aggregate node  
    - *Potential Failures:* Data schema mismatches, empty inputs  

  - **Aggregate**  
    - *Type:* Aggregate node  
    - *Role:* Concatenates or aggregates the merged audit outputs into one unified field called `output`  
    - *Configuration:* Aggregates the field named `output` from incoming items  
    - *Output:* Single item with combined audit data for editorial processing  
    - *Potential Failures:* Empty or malformed data  

  - **Editor in Chef AI Agent**  
    - *Type:* Langchain Agent Node  
    - *Role:* Crafts a polished, professional, consulting-style newsletter summarizing all audit findings and business implications, with a call-to-action for further engagement  
    - *Input:* Aggregated audit results as `{{ $json.output }}`  
    - *Output:* Final audit report in Markdown + HTML, branded with Sensonar logo and structured into sections (Summary, Strengths, Improvements, Opportunities, Next Step)  
    - *Prompt:* Includes instructions for executive tone, structure, and branding  
    - *AI Model:* Anthropic Claude Sonnet (claude-sonnet-4-20250514) via Anthropic Chat Model node  
    - *Potential Failures:* Prompt compliance, length constraints, API errors  

  - **Anthropic Chat Model**  
    - *Type:* Language Model node (Anthropic Claude Sonnet)  
    - *Role:* Provides advanced language model inference for editorial synthesis  
    - *Credentials:* Anthropic API with valid credentials  
    - *Potential Failures:* API quota exceeded, credential misconfiguration

---

#### 2.4 Report Formatting and Delivery

- **Overview:**  
  Converts the final Markdown report to HTML for email compatibility and sends it to the client's email via Gmail OAuth2 with customized sender details.

- **Nodes Involved:**  
  - Markdown  
  - Gmail

- **Node Details:**

  - **Markdown**  
    - *Type:* Markdown to HTML converter  
    - *Role:* Transforms the Markdown audit report into clean HTML for email formatting  
    - *Input:* Final report text from Editor in Chef AI Agent (`{{ $json.output }}`)  
    - *Output:* HTML formatted report  
    - *Potential Failures:* Markdown syntax issues causing conversion errors  

  - **Gmail**  
    - *Type:* Gmail node (OAuth2)  
    - *Role:* Sends the formatted HTML report via email to the client  
    - *Configuration:*  
      - Recipient email dynamically set from webhook input: `={{ $('Webhook Webpage Audit AI Agent').item.json.body.payload.data.Email }}`  
      - Subject includes audited URL  
      - Reply-To set to sensonar@sensonar.com  
      - Sender name: "Sensonar Audit AI Agent"  
    - *Credentials:* Gmail OAuth2 credentials linked to a specific account  
    - *Potential Failures:* OAuth token expiration, Gmail API quota, invalid recipient email

---

#### 2.5 Webhook Response

- **Overview:**  
  Sends a response to the webhook caller confirming receipt, with no data returned.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**

  - **Respond to Webhook**  
    - *Type:* Respond to Webhook node  
    - *Role:* Sends HTTP 200 response with empty body to acknowledge the webhook trigger  
    - *Input:* After Gmail node completes sending email  
    - *Potential Failures:* Response timeout or failure if downstream nodes error out

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                                  | Input Node(s)                       | Output Node(s)                    | Sticky Note                                                                                                                                        |
|-------------------------------|----------------------------------|-------------------------------------------------|-----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| Webhook Webpage Audit AI Agent | Webhook                          | Receive URL and email input                      | -                                 | Scrape Website                   | ![Sensonar Logo](https://cdn.prod.website-files.com/6540c08f5a4abefc9c0172ad/67cf2fc8095b00fc474866ac_logo_sensonar_black.png) AI AGENT that writes a Digital Audit Report of your website then send it directly to your email… |
| Scrape Website                 | HTTP Request                     | Download website HTML content                     | Webhook Webpage Audit AI Agent     | IT Technical AI Agent, CRO AI Agent, SEO AI Agent |                                                                                                                                                    |
| IT Technical AI Agent          | Langchain Agent (GPT-4o-mini)   | Technical SEO & security audit                    | Scrape Website                    | Merge                            |                                                                                                                                                    |
| OpenAI Chat Model2             | OpenAI GPT-4o-mini Model         | Provides LLM inference for IT Technical AI Agent | IT Technical AI Agent (ai_languageModel) | IT Technical AI Agent            |                                                                                                                                                    |
| CRO AI Agent                  | Langchain Agent (GPT-4o-mini)   | Conversion rate optimization & UX audit           | Scrape Website                    | Merge                            |                                                                                                                                                    |
| OpenAI Chat Model             | OpenAI GPT-4o-mini Model         | Provides LLM inference for CRO AI Agent           | CRO AI Agent (ai_languageModel)  | CRO AI Agent                    |                                                                                                                                                    |
| SEO AI Agent                  | Langchain Agent (GPT-4o-mini)   | Content SEO audit & keyword analysis               | Scrape Website                    | Merge                            |                                                                                                                                                    |
| OpenAI Chat Model1            | OpenAI GPT-4o-mini Model         | Provides LLM inference for SEO AI Agent            | SEO AI Agent (ai_languageModel)  | SEO AI Agent                    |                                                                                                                                                    |
| Merge                        | Merge Node                      | Combine audit outputs                              | IT Technical AI Agent, SEO AI Agent, CRO AI Agent | Aggregate                      |                                                                                                                                                    |
| Aggregate                   | Aggregate Node                   | Aggregate merged outputs into unified structure  | Merge                           | Editor in Chef AI Agent          |                                                                                                                                                    |
| Editor in Chef AI Agent        | Langchain Agent (Claude Sonnet) | Synthesize final consulting-style audit report   | Aggregate                       | Markdown                        |                                                                                                                                                    |
| Anthropic Chat Model          | Anthropic Claude Sonnet Model    | Provides LLM inference for Editor in Chef AI Agent | Editor in Chef AI Agent (ai_languageModel) | Editor in Chef AI Agent          |                                                                                                                                                    |
| Markdown                      | Markdown to HTML converter       | Convert Markdown report to HTML                    | Editor in Chef AI Agent          | Gmail                          |                                                                                                                                                    |
| Gmail                        | Gmail Node (OAuth2)              | Send the final report email                        | Markdown                       | Respond to Webhook              |                                                                                                                                                    |
| Respond to Webhook            | Respond to Webhook Node          | Send HTTP 200 response to webhook caller          | Gmail                          | -                              |                                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node:**  
   - Name: `Webhook Webpage Audit AI Agent`  
   - HTTP Method: POST  
   - Path: `webpage-audit-ai-agent`  
   - Enable raw body capture  
   - This node receives JSON payloads containing URL and recipient email.

2. **Create an HTTP Request node:**  
   - Name: `Scrape Website`  
   - Set URL parameter to dynamic expression: `={{ $json.body.payload.data.URL }}`  
   - No additional options required  
   - Connect webhook output to this node.

3. **Create three Langchain Agent nodes for audits:**  
   - **IT Technical AI Agent:**  
     - Use GPT-4o-mini model  
     - Prompt: Technical SEO + SecOps audit focused on raw HTML input  
     - Input: `={{ $json.data }}` from Scrape Website  
   - **SEO AI Agent:**  
     - Use GPT-4o-mini model  
     - Prompt: SEO content quality, keyword, and readability analysis  
     - Input: `={{ $json.data }}`  
   - **CRO AI Agent:**  
     - Use GPT-4o-mini model  
     - Prompt: Conversion optimization, UX, marketing psychology audit  
     - Input: `={{ $json.data }}`  
   - For each, configure their respective OpenAI credentials and link to OpenAI Chat Model nodes.

4. **Create three OpenAI Chat Model nodes:**  
   - Name accordingly: `OpenAI Chat Model2` (technical), `OpenAI Chat Model1` (SEO), `OpenAI Chat Model` (CRO)  
   - Model: GPT-4o-mini  
   - Connect each to its respective Agent node through `ai_languageModel` input.

5. **Merge AI agents outputs:**  
   - Create a Merge node named `Merge`  
   - Set number of inputs to 3  
   - Connect outputs of IT Technical AI Agent, SEO AI Agent, and CRO AI Agent to this Merge node.

6. **Aggregate merged data:**  
   - Add an Aggregate node named `Aggregate`  
   - Configure it to aggregate the field `output` from all input items  
   - Connect Merge output to Aggregate input.

7. **Create Editor in Chief AI Agent node:**  
   - Name: `Editor in Chef AI Agent`  
   - Use Anthropic Claude Sonnet model (`claude-sonnet-4-20250514`) via an Anthropic Chat Model node  
   - Input: aggregated audit results (`={{ $json.output }}`)  
   - Prompt: instruct to synthesize a polished consulting-style newsletter report with business impact and CTA to engage Improvement Agent  
   - Connect Aggregate output to Editor in Chef AI Agent input.

8. **Create Anthropic Chat Model node:**  
   - Name: `Anthropic Chat Model`  
   - Model: Claude Sonnet 4  
   - Connect to Editor in Chef AI Agent via `ai_languageModel` input  
   - Configure Anthropic API credentials.

9. **Convert Markdown to HTML:**  
   - Add a Markdown node named `Markdown`  
   - Set mode to `markdownToHtml`  
   - Input: output from Editor in Chef AI Agent (`={{ $json.output }}`)  
   - Connect Editor in Chef AI Agent output to this node.

10. **Configure Gmail node to send email:**  
    - Name: `Gmail`  
    - Set recipient: dynamic expression `={{ $('Webhook Webpage Audit AI Agent').item.json.body.payload.data.Email }}`  
    - Set subject: `"Sensonar Audit AI Agent for {{ $('Webhook Webpage Audit AI Agent').item.json.body.payload.data.URL }}"`  
    - Set reply-to: `sensonar@sensonar.com`  
    - Sender name: `Sensonar Audit AI Agent`  
    - Connect credentials for Gmail OAuth2 with valid permissions  
    - Connect Markdown output to Gmail input.

11. **Add Respond to Webhook node:**  
    - Name: `Respond to Webhook`  
    - Configure to send an empty 200 response  
    - Connect Gmail output to this node.

12. **Connections Summary:**  
    - Webhook → Scrape Website → [IT Technical AI Agent, SEO AI Agent, CRO AI Agent] in parallel  
    - Each AI Agent → respective OpenAI Chat Model nodes  
    - AI Agents outputs → Merge → Aggregate → Editor in Chef AI Agent → Anthropic Chat Model  
    - Editor in Chef AI Agent → Markdown → Gmail → Respond to Webhook

13. **Credentials Setup:**  
    - OpenAI API credentials linked to each OpenAI Chat Model node  
    - Anthropic API credentials linked to Anthropic Chat Model node  
    - Gmail OAuth2 credentials linked to Gmail node

14. **Default Values and Constraints:**  
    - Model choice fixed to GPT-4o-mini for OpenAI nodes  
    - Anthropic node uses Claude Sonnet model  
    - Webhook expects JSON with `.body.payload.data.URL` and `.body.payload.data.Email`  
    - Ensure all API keys and OAuth tokens are valid and have sufficient quota.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                  | Context or Link                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| ![Sensonar Logo](https://cdn.prod.website-files.com/6540c08f5a4abefc9c0172ad/67cf2fc8095b00fc474866ac_logo_sensonar_black.png)  \nThis workflow is designed and developed by Sebastian Jakubiak at Sensonar. It generates a complete digital audit report combining Technical SEO, Content SEO, and CRO analysis with AI agents, then sends a polished consulting-style email report. | https://sensonar.com / https://www.linkedin.com/in/sjakubiak     |
| Cost per execution ranges from approximately $0.40 to $0.70 USD depending on website content length; designed to balance speed (GPT-4o-mini) with high-quality editorial output (Claude Sonnet).                                                                | Pricing estimate based on API usage                              |
| The workflow encourages engagement with a follow-up "AI Improvement Agent" to design new digital strategy and website structure, indicated in the final report's call-to-action.                                                                                 | Embedded in Editor in Chef AI Agent prompt                       |

---

**Disclaimer:** The provided text is extracted and fully analyzed from an automated n8n workflow. All data processed are legal and public. The workflow complies strictly with content policies and contains no illegal or offensive material.