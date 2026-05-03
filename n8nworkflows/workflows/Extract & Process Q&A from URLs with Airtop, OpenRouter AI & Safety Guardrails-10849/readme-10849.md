Extract & Process Q&A from URLs with Airtop, OpenRouter AI & Safety Guardrails

https://n8nworkflows.xyz/workflows/extract---process-q-a-from-urls-with-airtop--openrouter-ai---safety-guardrails-10849


# Extract & Process Q&A from URLs with Airtop, OpenRouter AI & Safety Guardrails

### 1. Workflow Overview

This workflow is designed to safely extract and process question-and-answer (Q&A) content from URLs sent via Telegram messages. It caters primarily to use cases involving automated content extraction from web documents, ensuring that any processed information adheres to privacy and safety guidelines before responding to users.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**  
  Captures incoming messages from Telegram and filters to accept only valid URLs.

- **1.2 Content Extraction**  
  Uses Airtop AI to extract structured Q&A data from the provided URL.

- **1.3 Safety Guardrails Application**  
  Analyzes the extracted content for Personally Identifiable Information (PII), Not Safe For Work (NSFW) material, and keywords, enforcing safety policies.

- **1.4 AI Processing & Response Generation**  
  Employs OpenRouter’s language model alongside optional Tavily web search tools to generate safe, concise responses based on extracted data.

- **1.5 Response Dispatch**  
  Sends the safe, processed response back to the Telegram user or alerts the user if guardrail violations occur.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for Telegram messages, filters incoming messages to accept only those containing pure URLs.

- **Nodes Involved:**  
  - Telegram Trigger1  
  - Filter Only URLs  
  - Note: Trigger2  
  - Note: URL Filter

- **Node Details:**

  - **Telegram Trigger1**  
    - *Type & Role:* Telegram Trigger node; initiates workflow on incoming Telegram messages.  
    - *Configuration:* Listens specifically for "message" updates.  
    - *Expressions:* None.  
    - *Input/Output:* No input; outputs incoming Telegram message JSON.  
    - *Edge Cases:* Workflow inactive means no trigger; malformed messages ignored.  
    - *Sub-workflow:* None.

  - **Filter Only URLs**  
    - *Type & Role:* Filter node; validates that the incoming message text is a valid URL using regex.  
    - *Configuration:* Regex condition `=^(https?:\/\/)[^\s/$.?#].[^\s]*$` applied to `{{$json.message.text}}`.  
    - *Expressions:* Evaluates message text against URL regex.  
    - *Input/Output:* Input from Telegram Trigger1; outputs only messages passing the URL check.  
    - *Edge Cases:* Messages containing extra text besides URL are rejected; strictly enforces URL format.  
    - *Sub-workflow:* None.

  - **Note: Trigger2** & **Note: URL Filter**  
    - Sticky notes providing explanations for the above nodes.

---

#### 2.2 Content Extraction

- **Overview:**  
  Extracts structured Q&A pairs from the validated URL by querying Airtop’s AI-powered extraction resource.

- **Nodes Involved:**  
  - Extract Q&A from URL  
  - Note: Extraction

- **Node Details:**

  - **Extract Q&A from URL**  
    - *Type & Role:* Airtop node; calls Airtop’s extraction API.  
    - *Configuration:*  
      - URL parameter set dynamically to `{{$json.message.text}}`.  
      - Prompt: "Extract all questions and answers from this form or document".  
      - Resource type: "extraction".  
      - Session mode: "new" (starts fresh extraction session).  
    - *Expressions:* URL passed from filtered Telegram message text.  
    - *Input/Output:* Input from Filter Only URLs; outputs extracted Q&A data under `$json.data.modelResponse`.  
    - *Credentials:* Airtop API credentials required.  
    - *Edge Cases:* Extraction may fail on non-text or complex sites; network or API errors possible.  
    - *Sub-workflow:* None.

  - **Note: Extraction**  
    - Sticky note describing the purpose and usage of the node.

---

#### 2.3 Safety Guardrails Application

- **Overview:**  
  Applies safety checks to the extracted content to detect and block PII, NSFW content, and specified keywords before further processing.

- **Nodes Involved:**  
  - Apply Safety Guardrails  
  - Note: Guardrails

- **Node Details:**

  - **Apply Safety Guardrails**  
    - *Type & Role:* Langchain Guardrails node; validates text content against safety policies.  
    - *Configuration:*  
      - Text input: `{{$json.data.modelResponse}}` from Airtop extraction.  
      - Guardrails:  
        - PII check: "all" types.  
        - NSFW threshold: 0.7 (above this triggers violation).  
        - Keywords: none specified (empty).  
    - *Expressions:* Uses extracted data text for evaluation.  
    - *Input/Output:* Input from Extract Q&A from URL; outputs either safe content (to Main agent) or violation flag (to Send Violation Alert).  
    - *Edge Cases:* False positives possible; misconfiguration can block valid content; API or model timeout failures.  
    - *Sub-workflow:* None.

  - **Note: Guardrails**  
    - Sticky note explaining guardrail purpose and configuration.

---

#### 2.4 AI Processing & Response Generation

- **Overview:**  
  Processes safe extracted content with OpenRouter language model and optionally enriches it with Tavily web search results to generate a concise, user-appropriate response.

- **Nodes Involved:**  
  - OpenRouter Model  
  - Tavily Web Search  
  - Main agent  
  - Note: Language Model  
  - Note: Web Search  
  - Note: Agent1

- **Node Details:**

  - **OpenRouter Model**  
    - *Type & Role:* Langchain OpenRouter LM node; provides language model for AI processing and guardrails.  
    - *Configuration:*  
      - Model: "openrouter/sherlock-dash-alpha".  
      - No additional options specified.  
    - *Expressions:* None.  
    - *Input/Output:* Connected as language model input to Apply Safety Guardrails and Main agent nodes.  
    - *Credentials:* Requires OpenRouter API credentials.  
    - *Edge Cases:* API key issues, rate limits, model unavailability.  
    - *Sub-workflow:* None.

  - **Tavily Web Search**  
    - *Type & Role:* Tavily tool node; provides AI-driven web search capabilities to supplement agent knowledge.  
    - *Configuration:*  
      - Query dynamically set via AI override input 'Query' (string).  
      - Options:  
        - Topic: "general".  
        - Max results and search depth dynamically set via AI override inputs 'Max_Results' (number) and 'Search_Depth' (string, default "basic").  
    - *Expressions:* Uses n8n’s fromAI auto-generated inputs for parameters.  
    - *Input/Output:* Tool input to Main agent node.  
    - *Credentials:* Tavily API credentials required.  
    - *Edge Cases:* API failures, empty or irrelevant search results.  
    - *Sub-workflow:* None.

  - **Main agent**  
    - *Type & Role:* Langchain Agent node; orchestrates response generation using OpenRouter LM and Tavily tool results.  
    - *Configuration:*  
      - System message: "Respond to the input question concisely and appropriately."  
      - Prompt type: "guardrails" (integrates safety checks).  
    - *Expressions:* Receives clean extracted content from guardrails; uses OpenRouter LM and Tavily tool as AI language model and knowledge source inputs.  
    - *Input/Output:* Inputs from Apply Safety Guardrails (safe path) and Tavily Web Search; outputs response to Send Safe Response node.  
    - *Edge Cases:* Failures in combining AI outputs; timeouts; prompt failures.  
    - *Sub-workflow:* None.

  - **Note: Language Model**, **Note: Web Search**, **Note: Agent1**  
    - Sticky notes explaining the role of the OpenRouter model, Tavily web search, and the main agent node respectively.

---

#### 2.5 Response Dispatch

- **Overview:**  
  Sends the AI-generated safe response back to the Telegram user or sends an alert message if the content violates safety guardrails.

- **Nodes Involved:**  
  - Send Safe Response  
  - Send Violation Alert

- **Node Details:**

  - **Send Safe Response**  
    - *Type & Role:* Telegram node; sends processed safe text response to original Telegram chat.  
    - *Configuration:*  
      - Text: `{{$json.output}}` (response from Main agent).  
      - Chat ID: Dynamically set to same chat as original message (`{{$('Telegram Trigger1').item.json.message.chat.id}}`).  
      - Additional fields: disables attribution appending.  
    - *Input/Output:* Input from Main agent; output is final Telegram message sent.  
    - *Credentials:* Telegram API credentials required.  
    - *Edge Cases:* Telegram API errors, invalid chat IDs, message length limits.  
    - *Sub-workflow:* None.

  - **Send Violation Alert**  
    - *Type & Role:* Telegram node; sends alert message if guardrail check fails.  
    - *Configuration:*  
      - Text: Static "Failed Guardrail Test".  
      - Chat ID: Same dynamic chat ID as above.  
      - Additional fields: disables attribution appending.  
    - *Input/Output:* Input from Apply Safety Guardrails node’s failure path.  
    - *Credentials:* Telegram API credentials required.  
    - *Edge Cases:* Same Telegram API related issues as above.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name            | Node Type                               | Functional Role                             | Input Node(s)        | Output Node(s)                 | Sticky Note                                                                                                                        |
|----------------------|---------------------------------------|---------------------------------------------|----------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger1     | Telegram Trigger                      | Captures incoming Telegram messages          | None                 | Filter Only URLs               | ## 📥 Telegram Trigger - Listens for incoming messages; configured for message updates. Activate workflow to enable.              |
| Filter Only URLs      | Filter                               | Filters messages to allow only valid URLs    | Telegram Trigger1    | Extract Q&A from URL           | ## 🔍 Filter Only URLs - Validates messages as URLs using regex; ignores messages with extra text; adjust regex for flexibility.  |
| Extract Q&A from URL  | Airtop                              | Extracts Q&A content from URL                 | Filter Only URLs     | Apply Safety Guardrails       | ## 📝 Extract Q&A from URL - Uses Airtop to pull structured Q&A from webpages; best for text-based documents; customizable prompt.|
| Apply Safety Guardrails | Langchain Guardrails               | Checks extracted text for NSFW/PII violations| Extract Q&A from URL | Main agent, Send Violation Alert | ## 🛡️ Apply Safety Guardrails - Checks for NSFW/PII; configurable thresholds; fails send alert.                                   |
| OpenRouter Model      | Langchain LM OpenRouter             | Provides language model for AI and guardrails| None                 | Apply Safety Guardrails, Main agent | ## 🔤 OpenRouter Model - Supplies language model; uses sherlock-dash-alpha; shared across nodes.                                   |
| Tavily Web Search     | Tavily Web Search Tool              | Provides AI-driven web search to agent       | None                 | Main agent                   | ## 🌐 Tavily Web Search - Provides web search capability; uses AI to set query depth and results; general topic default.           |
| Main agent           | Langchain Agent                    | Generates safe responses combining AI and search | Apply Safety Guardrails, Tavily Web Search | Send Safe Response         | ## 🧠 Main Agent - Processes safe content with OpenRouter model and Tavily search.                                                 |
| Send Safe Response    | Telegram Send Message              | Sends safe generated response to Telegram user| Main agent          | None                        |                                                                                                                                   |
| Send Violation Alert  | Telegram Send Message              | Sends alert if guardrail check fails         | Apply Safety Guardrails (fail path) | None                |                                                                                                                                   |
| Note: Trigger2        | Sticky Note                       | Explains Telegram Trigger node                | None                 | None                         |                                                                                                                                   |
| Note: URL Filter      | Sticky Note                       | Explains URL filter purpose                    | None                 | None                         |                                                                                                                                   |
| Note: Extraction      | Sticky Note                       | Explains Airtop extraction                     | None                 | None                         |                                                                                                                                   |
| Note: Guardrails      | Sticky Note                       | Explains safety guardrails                     | None                 | None                         |                                                                                                                                   |
| Note: Language Model  | Sticky Note                       | Explains OpenRouter LM usage                    | None                 | None                         |                                                                                                                                   |
| Note: Web Search      | Sticky Note                       | Explains Tavily web search                      | None                 | None                         |                                                                                                                                   |
| Note: Agent1          | Sticky Note                       | Explains main agent role                        | None                 | None                         |                                                                                                                                   |
| Overview Note9        | Sticky Note                       | Overview and setup instructions                 | None                 | None                         | # 🤖 Secure URL Q&A Extractor for Telegram with AI Safety - Detailed explanation, prerequisites, credentials, use cases, and troubleshooting.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates only.  
   - Credentials: Configure with valid Telegram Bot API credentials.  
   - Position: Place at the start of the workflow.

2. **Add Filter Node "Filter Only URLs"**  
   - Type: Filter  
   - Condition: Apply regex on `{{$json.message.text}}` with pattern `=^(https?:\/\/)[^\s/$.?#].[^\s]*$` to allow only valid URLs.  
   - Connect output of Telegram Trigger to this node.

3. **Add Airtop Node "Extract Q&A from URL"**  
   - Type: Airtop  
   - Parameters:  
     - URL: Set to `{{$json.message.text}}` from previous node.  
     - Prompt: "Extract all questions and answers from this form or document".  
     - Resource: "extraction".  
     - Session Mode: "new".  
   - Credentials: Add Airtop API credentials.  
   - Connect output of Filter Only URLs to this node.

4. **Add Langchain Guardrails Node "Apply Safety Guardrails"**  
   - Type: Langchain Guardrails  
   - Parameters:  
     - Text: Set to `{{$json.data.modelResponse}}` (output from Airtop node).  
     - Guardrails:  
       - PII: enable all types.  
       - NSFW: threshold 0.7.  
       - Keywords: leave empty.  
   - Connect output of Extract Q&A from URL to this node.

5. **Add OpenRouter LM Node "OpenRouter Model"**  
   - Type: Langchain LM Chat OpenRouter  
   - Parameters:  
     - Model: "openrouter/sherlock-dash-alpha".  
   - Credentials: Add OpenRouter API credentials.  
   - Connect this node as AI language model input to Apply Safety Guardrails and to the Main agent node.

6. **Add Tavily Web Search Node "Tavily Web Search"**  
   - Type: Tavily Web Search Tool  
   - Parameters:  
     - Query: Use AI override parameter "Query" (string).  
     - Options:  
       - Topic: "general".  
       - Max Results: AI override parameter "Max_Results" (number).  
       - Search Depth: AI override parameter "Search_Depth" (string), default "basic".  
   - Credentials: Add Tavily API credentials.  
   - Connect output to Main agent node as AI tool input.

7. **Add Langchain Agent Node "Main agent"**  
   - Type: Langchain Agent  
   - Parameters:  
     - System message: "Respond to the input question concisely and appropriately."  
     - Prompt type: "guardrails".  
   - Connect safe output of Apply Safety Guardrails to this node.  
   - Connect Tavily Web Search node as AI tool input.  
   - Connect OpenRouter Model node as AI language model input.

8. **Add Telegram Node "Send Safe Response"**  
   - Type: Telegram  
   - Parameters:  
     - Text: Use `{{$json.output}}` from Main agent node.  
     - Chat ID: Set dynamically to `{{$('Telegram Trigger1').item.json.message.chat.id}}`.  
     - Additional Fields: disable attribution appending.  
   - Credentials: Use Telegram API credentials.  
   - Connect output of Main agent node to this node.

9. **Add Telegram Node "Send Violation Alert"**  
   - Type: Telegram  
   - Parameters:  
     - Text: "Failed Guardrail Test".  
     - Chat ID: Same dynamic chat ID as above.  
     - Additional Fields: disable attribution appending.  
   - Credentials: Use Telegram API credentials.  
   - Connect failure output of Apply Safety Guardrails node to this node.

10. **Add Sticky Notes as Desired**  
    - Add explanatory sticky notes for each block and node to facilitate maintenance and onboarding.

11. **Activate the Workflow**  
    - Ensure all credentials are correctly set and tested.  
    - Enable the workflow and test by sending valid URLs to the Telegram bot.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| # 🤖 Secure URL Q&A Extractor for Telegram with AI Safety - Includes detailed instructions on prerequisites, credentials setup, configuration steps, use cases, and troubleshooting tips.                                                                                                                                                                                                  | Overview Note9 sticky note within the workflow                                                                                               |
| Telegram API setup instructions: Use @BotFather to create a bot and obtain token; add as Telegram API credential in n8n.                                                                                                                                                                                                                                                                    | Overview Note9 sticky note                                                                                                                   |
| Airtop API setup: Register at https://airtop.ai, generate API keys, add to n8n credentials.                                                                                                                                                                                                                                                                                                 | Overview Note9 sticky note                                                                                                                   |
| OpenRouter API setup: Register at https://openrouter.ai, generate API keys, add to n8n credentials.                                                                                                                                                                                                                                                                                         | Overview Note9 sticky note                                                                                                                   |
| Tavily API setup: Register at https://app.tavily.com, generate API keys, add to n8n credentials.                                                                                                                                                                                                                                                                                             | Overview Note9 sticky note                                                                                                                   |
| Adjust guardrail thresholds (NSFW, PII, keywords) depending on use case sensitivity and false positive tolerance.                                                                                                                                                                                                                                                                            | Note: Guardrails sticky note                                                                                                                 |
| Regex filter for URLs is strict; to allow URLs with additional text or parameters, modify the regex pattern accordingly.                                                                                                                                                                                                                                                                     | Note: URL Filter sticky note                                                                                                                 |
| Airtop extraction prompt can be customized to better fit different document structures or extraction targets.                                                                                                                                                                                                                                                                                | Note: Extraction sticky note                                                                                                                 |
| OpenRouter model "sherlock-dash-alpha" selected for balanced performance; alternative models can be configured as needed.                                                                                                                                                                                                                                                                   | Note: Language Model sticky note                                                                                                             |
| Tavily web search supports dynamic query and search depth adjustments via AI override inputs, enabling flexible information retrieval strategies.                                                                                                                                                                                                                                          | Note: Web Search sticky note                                                                                                                 |
| For troubleshooting: If no response is received, verify URL format and guardrail settings; if too much content is blocked, adjust guardrail thresholds; test with simple sites to confirm extraction functionality; monitor API rate limits and quota usage.                                                                                                                                  | Overview Note9 sticky note                                                                                                                   |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies fully with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.