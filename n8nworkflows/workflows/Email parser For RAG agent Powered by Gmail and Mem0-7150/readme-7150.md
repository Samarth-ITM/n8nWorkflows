Email parser For RAG agent Powered by Gmail and Mem0

https://n8nworkflows.xyz/workflows/email-parser-for-rag-agent-powered-by-gmail-and-mem0-7150


# Email parser For RAG agent Powered by Gmail and Mem0

### 1. Workflow Overview

This workflow is an automated email parsing and analysis engine designed for use with Gmail and mem0.ai, aimed at transforming unstructured email communications into structured, actionable data for Retrieval-Augmented Generation (RAG) agents or other AI-driven applications. It continuously monitors a Gmail inbox for new emails, extracts essential content, analyzes the sentiment and key elements of the message, and stores the results in a persistent memory system (mem0).

The workflow’s logical blocks are:

- **1.1 Input Reception:** Capture new emails from Gmail and prepare email data for processing.
- **1.2 Contextual Memory Setup:** Manage conversational context by retrieving recent messages within the same thread.
- **1.3 AI Analysis Agent:** Use a configurable AI language model to parse and analyze email content, extracting core messages, sentiment, red flags, and keywords.
- **1.4 Output Parsing and Validation:** Ensure AI output is structured, valid JSON; apply auto-fixing and secondary parsing if necessary.
- **1.5 Memory Storage:** Store parsed email data persistently in mem0 for long-term client intelligence.
- **1.6 Auxiliary & Informational:** Sticky note node provides detailed explanation and instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow on new incoming Gmail messages and prepares a clean, standardized set of email properties for downstream processing.

**Nodes Involved:**  
- Full_Email (Gmail Trigger)  
- Set Target Email (Set Node)

**Node Details:**

- **Full_Email**  
  - Type: Gmail Trigger  
  - Role: Watches Gmail inbox in real-time, triggering workflow on each new email.  
  - Configuration: Polls every minute; does not filter emails; retrieves full email data including HTML and text bodies.  
  - Inputs: None (trigger node)  
  - Outputs: Raw email JSON with metadata, headers, content, sender info, etc.  
  - Failures: Possible OAuth token expiration; Gmail API rate limiting.

- **Set Target Email**  
  - Type: Set  
  - Role: Extracts and normalizes key email properties for easier handling downstream.  
  - Configuration: Extracts fields such as id, threadId, labelIds, textAsHtml, plain text, html, subject, date, sender address, and parses sender email from headers. Uses fallback defaults to avoid null errors.  
  - Inputs: Output of Full_Email  
  - Outputs: JSON object with clean, flattened email data.  
  - Failures: Expression failures if input data is missing or malformed.

---

#### 2.2 Contextual Memory Setup

**Overview:**  
This block maintains conversational context by loading recent messages from the same email thread to provide the AI agent with relevant history, improving analysis quality.

**Nodes Involved:**  
- Window Buffer Memory

**Node Details:**

- **Window Buffer Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Retrieves last 10 messages in the same thread identified by threadId for context.  
  - Configuration: Uses custom session key based on threadId from Set Target Email node; context window length set to 10 messages.  
  - Inputs: Set Target Email (to get threadId)  
  - Outputs: Contextual memory passed to AI agent.  
  - Failures: Missing or empty threadId; context retrieval issues.

---

#### 2.3 AI Analysis Agent

**Overview:**  
This core block uses a configurable AI language model to parse the cleaned email text, extracting the core message, sentiment, red flags, and keywords according to defined prompt constraints and instructions.

**Nodes Involved:**  
- llm of your choice (OpenAI GPT-4.1-nano)  
- Parse_Email Agent (LangChain Agent Node)

**Node Details:**

- **llm of your choice**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Primary LLM for text understanding and analysis  
  - Configuration: GPT-4.1-nano model selected; temperature set to 0.7 for balanced creativity and determinism; linked to OpenAI API credentials.  
  - Inputs: Not directly connected; likely referenced as a resource by Parse_Email Agent.  
  - Outputs: AI-generated completions.  
  - Failures: API quota limits, network timeouts, auth errors.

- **Parse_Email Agent**  
  - Type: LangChain Agent (Chat)  
  - Role: Central AI agent performing multi-step email parsing, sentiment analysis, red flag detection, and keyword extraction.  
  - Configuration: Receives plain text email content from Set Target Email; system prompt defines strict extraction and output formatting rules including multi-language support, error handling, and output structure (CRITICS structure). Includes tools like HTML Parser, Sentiment Analysis, Keyword Spotting, and NLP Keyword Extraction. Output parser enabled.  
  - Inputs: Email text from Set Target Email; memory context from Window Buffer Memory; LLM resource.  
  - Outputs: Structured JSON containing parsed_email, sentiment, potential red flags, keywords, and NLP keywords.  
  - Failures: Parsing errors, model timeouts, malformed input, multi-language ambiguities.

---

#### 2.4 Output Parsing and Validation

**Overview:**  
To guarantee data integrity, the workflow validates and auto-corrects the AI output JSON, using a two-step parsing and auto-fixing mechanism.

**Nodes Involved:**  
- Structured Output Parser  
- Auto-fixing Output Parser  
- Parsing LLM (Mistral)

**Node Details:**

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Validates that the AI output matches a defined JSON schema.  
  - Configuration: Manual JSON schema requiring fields for parsed_email, sentiment, potential_red_flags, keywords, nlp_keywords.  
  - Inputs: Output from AI agent (Parse_Email Agent)  
  - Outputs: Parsed and validated JSON object.  
  - Failures: Schema mismatch, missing fields.

- **Auto-fixing Output Parser**  
  - Type: LangChain Output Parser Autofixing  
  - Role: If Structured Output Parser fails, attempts to repair the output by re-prompting with error details.  
  - Configuration: Uses a prompt template including instructions, completion, and error to retry parse correction.  
  - Inputs: Output of Structured Output Parser or Parsing LLM  
  - Outputs: Corrected JSON output for downstream use.  
  - Failures: Persistent parsing failures or model errors.

- **Parsing LLM**  
  - Type: LangChain Chat Model (Mistral)  
  - Role: Secondary LLM to assist in parsing and fixing malformed outputs.  
  - Configuration: Smaller Mistral model with temperature 0.7 for creative corrections.  
  - Inputs: Invoked by Auto-fixing Output Parser when needed.  
  - Outputs: Corrected structured output.  
  - Failures: Parsing failure, auth issues.

---

#### 2.5 Memory Storage

**Overview:**  
Stores the finalized parsed email data into mem0, creating a persistent, queryable memory record keyed by sender email address.

**Nodes Involved:**  
- Add_Parsed email to memory (MCP Client)  
- email to mem0 (HTTP Request)

**Node Details:**

- **Add_Parsed email to memory**  
  - Type: MCP Client (Community Node)  
  - Role: Adds parsed output as memory content linked to the sender’s email in mem0.  
  - Configuration: Uses HTTP connection type; sends JSON stringified AI output in `content` field; userId is sender email address.  
  - Inputs: Output from Parse_Email Agent (validated and parsed JSON)  
  - Outputs: Confirmation response from mem0 API.  
  - Failures: Network errors, auth failures, malformed data.

- **email to mem0**  
  - Type: HTTP Request  
  - Role: Alternative or complementary method to send parsed email data to mem0 API.  
  - Configuration: POST request to mem0 memories API endpoint; payload includes messages content, user_id, agent_id (sentiment), metadata (keywords), infer flag, output format and version; uses HTTP Header Authentication credentials.  
  - Inputs: Output of Parse_Email Agent  
  - Outputs: API response  
  - Failures: HTTP errors, auth errors, rate limits.  
  - Special: On error, continues workflow without stopping.

---

#### 2.6 Auxiliary & Informational

**Overview:**  
Provides detailed documentation and explanation within the workflow for maintainers and users.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Contains detailed problem description, workflow purpose, component explanations, setup requirements, and optional usage notes.  
  - Inputs: None  
  - Outputs: None

---

### 3. Summary Table

| Node Name               | Node Type                            | Functional Role                          | Input Node(s)          | Output Node(s)                  | Sticky Note                                                                                                                   |
|-------------------------|------------------------------------|----------------------------------------|------------------------|--------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Full_Email              | Gmail Trigger                      | Trigger on new incoming Gmail emails   | None                   | Set Target Email               | Your inbox is unstructured; this workflow automates parsing and analysis to generate structured data for growth.             |
| Set Target Email        | Set                                | Normalize and extract key email fields | Full_Email             | Parse_Email Agent              | Maintains data cleanliness for AI processing.                                                                                 |
| Window Buffer Memory    | LangChain Memory Buffer Window     | Retrieve last 10 messages in thread    | Set Target Email       | Parse_Email Agent              | Provides context for smarter AI analysis.                                                                                    |
| llm of your choice      | LangChain OpenAI Chat Model        | Primary LLM for email content analysis | None (referenced)      | Parse_Email Agent (indirect)   | Configured with GPT-4.1-nano, balanced temperature for creative yet consistent responses.                                     |
| Parse_Email Agent       | LangChain Agent                    | Parse email, extract message & metadata | Set Target Email, Window Buffer Memory, llm of your choice | Add_Parsed email to memory, email to mem0 | Core AI engine performing multi-step analysis and extraction.                                                                |
| Structured Output Parser| LangChain Structured Output Parser | Validate AI JSON output                 | Parsing LLM             | Auto-fixing Output Parser      | Ensures output matches strict schema.                                                                                        |
| Auto-fixing Output Parser| LangChain Output Parser Autofixing | Auto-corrects malformed AI output      | Structured Output Parser | Parse_Email Agent              | Retries with error context for reliable parsing.                                                                              |
| Parsing LLM             | LangChain Chat Model (Mistral)     | Secondary LLM for output fix            | Auto-fixing Output Parser | Auto-fixing Output Parser      | Provides fallback parsing corrections.                                                                                       |
| Add_Parsed email to memory| MCP Client (Community Node)       | Store parsed email data in mem0         | Parse_Email Agent       | None                         | Logs client intelligence by sender email address.                                                                            |
| email to mem0           | HTTP Request                      | Alternative mem0 data ingestion          | Parse_Email Agent       | None                         | Sends structured data to mem0 REST API; continues on errors.                                                                 |
| Sticky Note             | Sticky Note                       | Documentation and explanation            | None                   | None                         | Detailed problem statement, workflow overview, and setup instructions.                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Configuration:  
     - Poll interval: Every minute  
     - Simple mode: Off (retrieve full email data)  
     - Filters: None  
   - Credentials: Connect your Gmail OAuth2 credentials.

2. **Create Set Node (Set Target Email)**  
   - Purpose: Extract key email fields for simpler downstream processing.  
   - Add assignments for:  
     - id: `={{ $json.id || null }}`  
     - threadId: `={{ $json.threadId || null }}`  
     - labelIds: `={{ $json.labelIds || [] }}`  
     - textAsHtml: `={{ $json.textAsHtml || '' }}`  
     - text: `={{ $json.text || '' }}`  
     - html: `={{ $json.html || '' }}`  
     - subject: `={{ $json.subject || '' }}`  
     - date: `={{ $json.date || null }}`  
     - from.value[0].address: `={{ $json.from?.value?.[0]?.address || null }}`  
     - headers.from: `={{ $json.headers.from.extractEmail()}}`  
   - Connect Gmail Trigger node output to this node’s input.

3. **Create Window Buffer Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Parameters:  
     - sessionKey: `={{ $('Set Target Email').item.json.threadId }}`  
     - sessionIdType: `customKey`  
     - contextWindowLength: 10  
   - Connect Set Target Email to this node.

4. **Create AI Language Model Node (llm of your choice)**  
   - Type: LangChain OpenAI Chat Model  
   - Parameters:  
     - Model: Select GPT-4.1-nano  
     - Temperature: 0.7  
   - Credentials: Use OpenAI API key.

5. **Create LangChain Agent Node (Parse_Email Agent)**  
   - Parameters:  
     - Text input: `={{ $json.text }}` from Set Target Email  
     - System message: Use detailed prompt defining parsing rules, constraints, tools, and error handling as in original workflow.  
     - Enable output parser.  
   - Connect inputs:  
     - Email text from Set Target Email  
     - AI model from llm of your choice  
     - Memory context from Window Buffer Memory

6. **Create Structured Output Parser Node**  
   - Type: LangChain Structured Output Parser  
   - Parameters:  
     - Schema type: Manual  
     - Input schema: JSON schema with fields - parsed_email (string), sentiment (string), potential_red_flags (array), keywords (array), nlp_keywords (array)  
   - Connect output of AI parsing node (Parse_Email Agent) to this node.

7. **Create Auto-fixing Output Parser Node**  
   - Type: LangChain Output Parser Autofixing  
   - Parameters: Prompt template with placeholders for instructions, completion, and error as in original.  
   - Connect Structured Output Parser output to this node.

8. **Create Secondary Parsing LLM Node (Parsing LLM)**  
   - Type: LangChain Chat Model  
   - Model: Mistral-small-2506 or similar lightweight model  
   - Temperature: 0.7  
   - Connect Auto-fixing Output Parser node’s AI languageModel output to this node.

9. **Connect Auto-fixing Output Parser output back to Parse_Email Agent output parser**  
   - This closes the loop for auto-correcting AI output.

10. **Create MCP Client Node (Add_Parsed email to memory)**  
    - Type: MCP Client (Community Node)  
    - Parameters:  
      - Tool name: add-memory  
      - Operation: executeTool  
      - Connection type: HTTP  
      - Tool parameters:  
        ```json
        {
          "content": "={{ JSON.stringify($json.output) }}",
          "userId": "={{ $('Set Target Email').item.json.from.value[0].address }}"
        }
        ```  
    - Credentials: Configure mem0 HTTP API credentials.  
    - Connect output of Parse_Email Agent to this node.

11. **Create HTTP Request Node (email to mem0)**  
    - Type: HTTP Request  
    - Parameters:  
      - URL: `https://api.mem0.ai/v1/memories/`  
      - Method: POST  
      - JSON Body:  
        ```json
        {
          "messages": [
            {
              "role": "user",
              "content": "={{ $json.output.core_message ?? \"\" }}"
            }
          ],
          "user_id": "={{ $('Set Target Email').item.json.from.value[0].address }}",
          "agent_id": "={{ $json.output.sentiment ?? \"unknown\" }}",
          "metadata": "={{ JSON.stringify($json.output.keywords ?? {}) }}",
          "infer": true,
          "output_format": "v1.1",
          "version": "v2"
        }
        ```  
      - Authentication: HTTP Header Authentication with mem0 credentials  
      - On Error: Continue workflow (do not stop on error)  
    - Connect output of Parse_Email Agent.

12. **Create Sticky Note Node**  
    - Add detailed explanation, problem solved, workflow steps, and setup requirements as content.

13. **Connect Nodes in Sequence:**  
    - Full_Email → Set Target Email → Window Buffer Memory → Parse_Email Agent  
    - Parse_Email Agent → Structured Output Parser → Auto-fixing Output Parser → Parsing LLM → Auto-fixing Output Parser (loop back)  
    - Parse_Email Agent → Add_Parsed email to memory  
    - Parse_Email Agent → email to mem0

14. **Test workflow with live emails to verify parsing accuracy and mem0 integration.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                                                |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow is designed to enable an “always-on” autonomous engine that converts raw email communication into structured, actionable client intelligence for scaling customer understanding and automation. It leverages advanced AI models including GPT-4 and Mistral, integrated with mem0.ai for long-term memory storage.                                                                                                                                                                                                                                                                                                                                                                                                              | Workflow Description                                                                                                                          |
| Optional usage: For historical email backfilling, disable the Gmail Trigger and use a Gmail "Get Many" node connected to Set Target Email to process batches of older emails.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Usage Note                                                                                                                                   |
| Required credentials include Gmail OAuth2 credentials, OpenAI API key, and mem0.ai API credentials. The MCP Client node requires community node installation or can be replaced by the HTTP request node alternative.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Setup Requirements                                                                                                                           |
| Sticky note in workflow contains comprehensive problem statement, architectural rationale, and instructions to support maintainers and users.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Sticky Note Content                                                                                                                          |
| Official mem0 API documentation: https://docs.mem0.ai                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | External Resource                                                                                                                            |
| Gmail API documentation: https://developers.google.com/gmail/api                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | External Resource                                                                                                                            |
| OpenAI GPT-4 API documentation: https://platform.openai.com/docs/models/gpt-4                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | External Resource                                                                                                                            |

---

**Disclaimer:** The text above is extracted and interpreted exclusively from an n8n workflow automation and complies fully with applicable content policies. No illegal, offensive, or protected content is included. All data processed is legal and publicly permissible.