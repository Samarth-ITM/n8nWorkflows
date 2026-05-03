Financial reporting AI: concise SEC 10-K/10-Q briefs via OpenRouter + Perplexity

https://n8nworkflows.xyz/workflows/financial-reporting-ai--concise-sec-10-k-10-q-briefs-via-openrouter---perplexity-8313


# Financial reporting AI: concise SEC 10-K/10-Q briefs via OpenRouter + Perplexity

---
### 1. Workflow Overview

This workflow, titled **Financial reporting AI: concise SEC 10-K/10-Q briefs via OpenRouter + Perplexity**, is designed to automate the generation of concise financial report summaries based on SEC filings such as 10-K and 10-Q documents. It leverages AI language models and external knowledge tools to process chat input messages, generate briefings, and output formatted documents.

**Target Use Cases:**  
- Financial analysts seeking quick, AI-generated summaries of SEC filings.  
- Automating the transformation of raw financial documents into digestible briefs.  
- Integrating conversational AI with external knowledge databases for enhanced context.

**Logical Blocks:**

- **1.1 Input Reception:** Listens for incoming chat messages to trigger the workflow.  
- **1.2 AI Processing:** Uses a LangChain AI Agent that integrates a language model (OpenRouter Chat Model) and an external knowledge tool (Perplexity) to process inputs and generate text outputs.  
- **1.3 Post-Processing and Formatting:** Converts AI-generated text to markdown, edits fields to prepare content, transforms markdown into HTML, and formats it for document creation.  
- **1.4 Document Creation:** Sends the formatted content to create a Google Document via an HTTP request node.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block waits for a chat message input to start the workflow, serving as the entry point.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - *Type:* LangChain Chat Trigger  
    - *Role:* Webhook listener for incoming chat messages that initiate the workflow.  
    - *Configuration:* Uses webhook with ID `9cfe5875-f425-4003-bd5e-c75a07c758ba`. No custom parameters, default behavior to listen for messages.  
    - *Inputs:* External chat message via webhook.  
    - *Outputs:* Triggers the AI Agent node.  
    - *Edge Cases:* Potential failure if webhook is unreachable or message format invalid. Also dependent on LangChain integration availability.

#### 2.2 AI Processing

- **Overview:**  
  This block uses an AI Agent node that combines a language model (OpenRouter Chat Model) and an external knowledge tool (Perplexity) to generate concise, contextually enriched responses to chat inputs.

- **Nodes Involved:**  
  - AI Agent  
  - OpenRouter Chat Model  
  - Message a model in Perplexity

- **Node Details:**  
  - **AI Agent**  
    - *Type:* LangChain Agent  
    - *Role:* Orchestrates AI model and external tool to process input messages and generate responses.  
    - *Configuration:* No explicit parameters set, relies on linked AI language model and AI tool connections.  
    - *Inputs:* Receives from "When chat message received" node and AI tools (OpenRouter, Perplexity).  
    - *Outputs:* Passes generated text to Markdown node.  
    - *Edge Cases:* Failures can occur due to API errors from attached AI models/tools, timeouts, or invalid responses.  
  - **OpenRouter Chat Model**  
    - *Type:* LangChain OpenRouter Chat Language Model  
    - *Role:* Provides the core language model capabilities for the AI Agent.  
    - *Configuration:* Default settings; connected as `ai_languageModel` input to AI Agent.  
    - *Inputs:* None (standalone model node).  
    - *Outputs:* Feeds into AI Agent.  
    - *Edge Cases:* API authentication errors, rate limits, or connectivity issues.  
  - **Message a model in Perplexity**  
    - *Type:* Perplexity Tool Node  
    - *Role:* Provides an external knowledge tool integration for AI Agent to enhance responses with real-time data or factual information.  
    - *Configuration:* No parameters set explicitly here; configured as `ai_tool` input to AI Agent.  
    - *Inputs:* None (standalone tool node).  
    - *Outputs:* Feeds into AI Agent.  
    - *Edge Cases:* API failures, timeout, or unexpected tool responses.

#### 2.3 Post-Processing and Formatting

- **Overview:**  
  Processes the AI-generated text output by converting it into markdown, editing fields as needed, and converting markdown into formatted HTML suitable for document creation.

- **Nodes Involved:**  
  - Markdown  
  - Edit Fields  
  - Format_HTML

- **Node Details:**  
  - **Markdown**  
    - *Type:* Markdown Node  
    - *Role:* Converts raw AI text output into markdown format for better structuring and formatting.  
    - *Configuration:* Default behavior, no custom parameters.  
    - *Inputs:* From AI Agent node.  
    - *Outputs:* To Edit Fields node.  
    - *Edge Cases:* Malformed markdown input may cause conversion issues.  
  - **Edit Fields**  
    - *Type:* Set Node  
    - *Role:* Edits or adds fields to the data stream, possibly preparing data structure or cleaning content before formatting.  
    - *Configuration:* No explicit parameter details; assumed to set or rename fields.  
    - *Inputs:* From Markdown node.  
    - *Outputs:* To Format_HTML node.  
    - *Edge Cases:* Incorrect field operations may cause missing or malformed data.  
  - **Format_HTML**  
    - *Type:* Code Node  
    - *Role:* Converts markdown content into HTML, likely applying custom formatting rules for document readiness.  
    - *Configuration:* Custom JavaScript or code logic implemented (details not provided).  
    - *Inputs:* From Edit Fields node.  
    - *Outputs:* To CreateGoogleDoc node.  
    - *Edge Cases:* Syntax errors in code, edge cases in markdown content causing broken HTML.

#### 2.4 Document Creation

- **Overview:**  
  Sends the final formatted HTML content to an HTTP endpoint to create or update a Google Document, completing the automation of report generation.

- **Nodes Involved:**  
  - CreateGoogleDoc

- **Node Details:**  
  - **CreateGoogleDoc**  
    - *Type:* HTTP Request Node  
    - *Role:* Performs an HTTP request to create a Google Doc with the formatted content.  
    - *Configuration:* URL, method, headers, and body parameters configured to interact with Google Docs API or custom service. Specific parameters are not detailed.  
    - *Inputs:* Receives HTML formatted content from Format_HTML node.  
    - *Outputs:* None further in this workflow (terminal node).  
    - *Edge Cases:* Authentication errors with Google API, request failures, quota limits, or malformed request body.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                         | Input Node(s)                  | Output Node(s)                | Sticky Note |
|----------------------------|----------------------------------|---------------------------------------|-------------------------------|------------------------------|-------------|
| When chat message received | LangChain Chat Trigger            | Entry point; receives chat messages   | (External webhook)             | AI Agent                     |             |
| AI Agent                   | LangChain Agent                  | Core AI processing combining LM & tool| When chat message received, OpenRouter Chat Model, Message a model in Perplexity | Markdown                     |             |
| OpenRouter Chat Model      | LangChain OpenRouter Chat Model   | AI language model for generating text | None                          | AI Agent (ai_languageModel)  |             |
| Message a model in Perplexity | Perplexity Tool Node             | External knowledge tool integration    | None                          | AI Agent (ai_tool)            |             |
| Markdown                   | Markdown Node                    | Converts AI text to markdown           | AI Agent                      | Edit Fields                  |             |
| Edit Fields                | Set Node                        | Edits/sets fields for formatting prep | Markdown                      | Format_HTML                  |             |
| Format_HTML                | Code Node                      | Converts markdown to HTML format       | Edit Fields                   | CreateGoogleDoc              |             |
| CreateGoogleDoc            | HTTP Request Node                | Creates Google Doc with formatted HTML | Format_HTML                   | None                        |             |
| Sticky Note                | Sticky Note Node                 | Documentation or comments              | None                          | None                        |             |
| Sticky Note1               | Sticky Note Node                 | Documentation or comments              | None                          | None                        |             |
| Sticky Note2               | Sticky Note Node                 | Documentation or comments              | None                          | None                        |             |
| Sticky Note3               | Sticky Note Node                 | Documentation or comments              | None                          | None                        |             |
| Sticky Note4               | Sticky Note Node                 | Documentation or comments              | None                          | None                        |             |
| Sticky Note5               | Sticky Note Node                 | Documentation or comments              | None                          | None                        |             |
| Sticky Note6               | Sticky Note Node                 | Documentation or comments              | None                          | None                        |             |
| Sticky Note7               | Sticky Note Node                 | Documentation or comments              | None                          | None                        |             |
| Sticky Note8               | Sticky Note Node                 | Documentation or comments              | None                          | None                        |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node:**
   - Type: LangChain Chat Trigger  
   - Configure webhook with default parameters to listen for incoming chat messages.

2. **Create "OpenRouter Chat Model" node:**
   - Type: LangChain OpenRouter Chat Model  
   - Use default parameters, connect to valid OpenRouter API credentials.

3. **Create "Message a model in Perplexity" node:**
   - Type: Perplexity Tool Node  
   - Configure with valid Perplexity API credentials (if needed).

4. **Create "AI Agent" node:**
   - Type: LangChain Agent  
   - No explicit parameters needed.  
   - Connect inputs:  
     - Main input from "When chat message received" node.  
     - `ai_languageModel` input from "OpenRouter Chat Model" node.  
     - `ai_tool` input from "Message a model in Perplexity" node.

5. **Create "Markdown" node:**
   - Type: Markdown Node  
   - Connect main input from "AI Agent" node output.

6. **Create "Edit Fields" node:**
   - Type: Set Node  
   - Connect main input from "Markdown" node output.  
   - Configure to edit or add necessary fields to prepare data for HTML formatting (e.g., rename fields, clear unnecessary ones).

7. **Create "Format_HTML" node:**
   - Type: Code Node  
   - Connect main input from "Edit Fields" node output.  
   - Write custom JavaScript to transform markdown text into HTML formatted content suitable for document creation.

8. **Create "CreateGoogleDoc" node:**
   - Type: HTTP Request Node  
   - Connect main input from "Format_HTML" node output.  
   - Configure HTTP request to Google Docs API or custom endpoint:  
     - Set method (POST/PUT) and URL for document creation.  
     - Include authentication headers (OAuth2 credentials for Google).  
     - Pass formatted HTML content in the request body.

9. **Connect nodes in this sequence:**
   - When chat message received → AI Agent → Markdown → Edit Fields → Format_HTML → CreateGoogleDoc

10. **Credential Setup:**
    - Setup OpenRouter API credentials for the OpenRouter Chat Model node.  
    - Setup Perplexity API credentials for the Perplexity node.  
    - Configure OAuth2 credentials for Google API in the HTTP Request node.

11. **Test the workflow end-to-end:**
    - Send a chat message via the webhook to trigger the process.  
    - Confirm AI Agent generates a response.  
    - Verify markdown conversion and HTML formatting.  
    - Ensure the Google Doc is created with the expected content.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| This workflow integrates LangChain nodes and external AI tools for enhanced financial document summarization.        | n8n LangChain Nodes Documentation: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain/                          |
| OpenRouter provides flexible AI model routing, useful for scalable language model usage.                             | OpenRouter info: https://openrouter.ai/                                                                                         |
| Perplexity tool node enriches AI agent responses with real-time knowledge retrieval capabilities.                     | Perplexity AI: https://www.perplexity.ai/                                                                                       |
| Google Docs creation requires OAuth2 credentials with appropriate scopes (e.g., https://www.googleapis.com/auth/documents) | Google Docs API Docs: https://developers.google.com/docs/api/quickstart/js                                                       |
| Edge cases may include API rate limits, authentication failures, and malformed inputs; proper error handling is advised.|                                                                                                                                |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is lawful and publicly available.