Proxmox AI Agent with n8n and Generative AI Integration

https://n8nworkflows.xyz/workflows/proxmox-ai-agent-with-n8n-and-generative-ai-integration-2749


# Proxmox AI Agent with n8n and Generative AI Integration

### 1. Workflow Overview

This workflow, titled **Proxmox AI Agent with n8n and Generative AI Integration**, automates management tasks on a Proxmox Virtual Environment (VE) by interpreting natural language commands via AI and converting them into Proxmox API calls. It supports multi-channel input (chat, Telegram, email, webhook), uses generative AI (Google Gemini) to parse user intents into structured API requests, executes these requests against Proxmox, and formats responses for user-friendly output.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Accepts user commands from various triggers (chat, Telegram, Gmail, webhook).
- **1.2 AI Intent Parsing and Command Generation**: Uses AI models and Proxmox API references to convert natural language into structured API calls.
- **1.3 API Request Routing and Execution**: Routes requests based on HTTP method and executes corresponding Proxmox API calls.
- **1.4 Response Processing and Formatting**: Processes raw API responses, extracts meaningful information, and formats it for user consumption.
- **1.5 Documentation and Reference Tools**: Provides embedded access to Proxmox API documentation and wiki for AI context.
- **1.6 Error Handling and Output Validation**: Ensures AI output correctness and handles missing parameters or invalid requests.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block captures user input from multiple channels, enabling flexible interaction with the AI agent.

**Nodes Involved:**  
- Telegram Trigger  
- Gmail Trigger  
- Webhook  
- When chat message received

**Node Details:**

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens for incoming Telegram messages to trigger the workflow.  
  - Configuration: Listens to "message" updates; uses Telegram API credentials.  
  - Inputs: External Telegram messages  
  - Outputs: Passes message text to AI Agent node  
  - Edge Cases: Telegram API downtime, invalid credentials, message format issues.

- **Gmail Trigger**  
  - Type: Gmail Trigger  
  - Role: Polls Gmail inbox every minute for new emails to trigger workflow.  
  - Configuration: No filters applied; uses Gmail OAuth2 credentials.  
  - Inputs: New emails  
  - Outputs: Passes email content to AI Agent node  
  - Edge Cases: Gmail API rate limits, OAuth token expiration, email parsing errors.

- **Webhook**  
  - Type: HTTP Webhook  
  - Role: Accepts HTTP requests with authentication header to trigger workflow.  
  - Configuration: Path set to a unique webhook ID; header authentication with Proxmox credentials.  
  - Inputs: HTTP requests from external systems  
  - Outputs: Passes payload to AI Agent node  
  - Edge Cases: Unauthorized access, malformed requests, network issues.

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Listens for chat messages from n8n’s built-in chat interface.  
  - Configuration: Default options; no special filters.  
  - Inputs: Chat messages  
  - Outputs: Passes chat input to AI Agent node  
  - Edge Cases: Chat service unavailability, malformed messages.

---

#### 1.2 AI Intent Parsing and Command Generation

**Overview:**  
This block uses generative AI (Google Gemini) combined with Proxmox API references to parse user input into structured API commands, ensuring strict JSON output for API calls.

**Nodes Involved:**  
- AI Agent  
- Google Gemini Chat Model  
- Proxmox API Documentation (tool)  
- Proxmox API Wiki (tool)  
- Proxmox (tool)  
- Structured Output Parser  
- Auto-fixing Output Parser

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Core AI logic that interprets user input, references Proxmox API docs and wiki, and generates structured API commands.  
  - Configuration: Uses "reActAgent" prompt type with detailed instructions on required JSON output format, validation rules, default behaviors, and examples.  
  - Inputs: User input from triggers, API documentation tools, and AI language model outputs.  
  - Outputs: JSON with fields: `response_type`, `url`, `details`, or error messages.  
  - Edge Cases: Ambiguous user input, missing required parameters, unrelated queries (returns `response_type: Invalid`).

- **Google Gemini Chat Model**  
  - Type: LangChain LM Chat Google Gemini  
  - Role: Language model used by AI Agent to process natural language and generate responses.  
  - Configuration: Model name "models/gemini-2.0-flash-exp"; uses Google PaLM API credentials.  
  - Inputs: Prompts from AI Agent  
  - Outputs: Text completions passed to output parsers  
  - Edge Cases: API rate limits, model unavailability, network errors.

- **Proxmox API Documentation (toolHttpRequest)**  
  - Type: LangChain Tool HTTP Request  
  - Role: Provides AI Agent with live access to Proxmox API documentation URL for reference.  
  - Configuration: URL set to official Proxmox API docs.  
  - Inputs: AI Agent queries  
  - Outputs: Documentation content for AI context.

- **Proxmox API Wiki (toolHttpRequest)**  
  - Type: LangChain Tool HTTP Request  
  - Role: Supplies AI Agent with Proxmox API wiki content to enhance understanding.  
  - Configuration: URL set to Proxmox VE API wiki page.  
  - Inputs: AI Agent queries  
  - Outputs: Wiki content for AI context.

- **Proxmox (toolHttpRequest)**  
  - Type: LangChain Tool HTTP Request  
  - Role: Allows AI Agent to query live Proxmox cluster status and node info to validate or fetch data dynamically.  
  - Configuration: URL points to Proxmox cluster status API endpoint; uses Proxmox API credentials with header auth.  
  - Inputs: AI Agent queries  
  - Outputs: Live Proxmox data for AI context.

- **Structured Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses AI model output to enforce strict JSON structure as per schema example (response_type, url, details, message).  
  - Configuration: JSON schema example provided to guide parsing.  
  - Inputs: Raw AI model text output  
  - Outputs: Parsed JSON object for downstream use  
  - Edge Cases: Parsing failures, malformed AI output.

- **Auto-fixing Output Parser**  
  - Type: LangChain Output Parser Autofixing  
  - Role: Automatically attempts to fix AI output parsing errors by re-prompting AI to correct output format.  
  - Configuration: Custom prompt instructing AI to fix errors without extraneous text.  
  - Inputs: Output from Structured Output Parser  
  - Outputs: Corrected JSON output or error message  
  - Edge Cases: Persistent parsing errors, infinite loops if AI fails to fix.

---

#### 1.3 API Request Routing and Execution

**Overview:**  
This block routes the structured API commands based on HTTP method and executes corresponding HTTP requests to Proxmox API endpoints.

**Nodes Involved:**  
- Switch  
- HTTP Request (GET)  
- HTTP Request1 (POST)  
- HTTP Request2 (POST alternative)  
- HTTP Request3 (DELETE)  
- HTTP Request4 (DELETE alternative)  
- If  
- If1  
- Merge  
- Merge1

**Node Details:**

- **Switch**  
  - Type: Switch  
  - Role: Routes workflow execution based on `response_type` field from AI output (GET, POST, PUT, DELETE, OPTIONS, Invalid).  
  - Configuration: Conditions check exact match of `response_type`.  
  - Inputs: AI Agent output JSON  
  - Outputs: Directs to appropriate HTTP Request node or error handling.

- **HTTP Request (GET)**  
  - Type: HTTP Request  
  - Role: Executes GET requests to Proxmox API using URL from AI output.  
  - Configuration: URL constructed by concatenating base Proxmox API URL with AI-generated endpoint; uses header auth credentials; allows unauthorized certs.  
  - Inputs: Switch node (GET path)  
  - Outputs: Raw API response JSON  
  - Edge Cases: Network errors, invalid URLs, auth failures.

- **HTTP Request1 (POST)**  
  - Type: HTTP Request  
  - Role: Executes POST requests with JSON body payload to Proxmox API.  
  - Configuration: URL and JSON body from AI output; header auth; allows unauthorized certs.  
  - Inputs: Switch node (POST path), If node (checks if details exist)  
  - Outputs: API response JSON  
  - Edge Cases: Missing required fields, server errors.

- **HTTP Request2 (POST alternative)**  
  - Type: HTTP Request  
  - Role: Alternative POST request node used for certain POST operations.  
  - Configuration: Similar to HTTP Request1 but with different connection path.  
  - Inputs: If node (alternative POST path)  
  - Outputs: API response JSON

- **HTTP Request3 (DELETE)**  
  - Type: HTTP Request  
  - Role: Executes DELETE requests to Proxmox API.  
  - Configuration: URL from AI output; header auth; allows unauthorized certs.  
  - Inputs: If1 node (checks if details missing or empty)  
  - Outputs: API response JSON

- **HTTP Request4 (DELETE alternative)**  
  - Type: HTTP Request  
  - Role: Alternative DELETE request node.  
  - Configuration: Similar to HTTP Request3.  
  - Inputs: If1 node (alternative DELETE path)  
  - Outputs: API response JSON

- **If**  
  - Type: If  
  - Role: Checks if `details` field in AI output is present and non-empty to decide POST request path.  
  - Inputs: Switch node (POST path)  
  - Outputs: Routes to HTTP Request1 or HTTP Request2

- **If1**  
  - Type: If  
  - Role: Checks if `details` field is empty or missing to decide DELETE request path.  
  - Inputs: Switch node (DELETE path)  
  - Outputs: Routes to HTTP Request3 or HTTP Request4

- **Merge**  
  - Type: Merge  
  - Role: Combines outputs from POST request nodes for unified downstream processing.  
  - Inputs: HTTP Request1 and HTTP Request2  
  - Outputs: Structured Response node

- **Merge1**  
  - Type: Merge  
  - Role: Combines outputs from DELETE request nodes for unified downstream processing.  
  - Inputs: HTTP Request3 and HTTP Request4  
  - Outputs: Structured Response node

---

#### 1.4 Response Processing and Formatting

**Overview:**  
This block transforms raw Proxmox API responses into concise, user-friendly summaries, hiding sensitive data and extracting key operation details.

**Nodes Involved:**  
- Structure Response  
- AI Agent1  
- Structgure Response from Proxmox (code)  
- Format Response and Hide Sensitive Data (code)  
- Sticky Notes (contextual)

**Node Details:**

- **Structure Response**  
  - Type: Code  
  - Role: Combines all fields from API response items into a single string for summary generation.  
  - Configuration: JavaScript code concatenates key-value pairs into a formatted string.  
  - Inputs: Merged API response data  
  - Outputs: Combined string field for AI Agent1

- **AI Agent1**  
  - Type: LangChain Agent  
  - Role: Acts as a Proxmox Information Output Expert, generating natural language summaries from combined API data.  
  - Configuration: Uses Google Gemini Chat Model2; prompt instructs to summarize Proxmox info meaningfully.  
  - Inputs: Structured combined data from Structure Response  
  - Outputs: User-friendly summary message

- **Structgure Response from Proxmox**  
  - Type: Code  
  - Role: Parses Proxmox task UPID string into discrete fields (upid, node, processID, taskID, timestamp, operation, user).  
  - Configuration: JavaScript splits raw string by colon delimiter and maps parts to named fields.  
  - Inputs: Raw API response data field  
  - Outputs: Parsed object with detailed fields

- **Format Response and Hide Sensitive Data**  
  - Type: Code  
  - Role: Formats parsed Proxmox task data into a readable message, converts timestamp from hex to human-readable date, and hides sensitive info.  
  - Configuration: JavaScript constructs a message string with operation, node, user, and timestamp details.  
  - Inputs: Parsed Proxmox response object  
  - Outputs: Final formatted message JSON

- **Sticky Notes**  
  - Provide contextual information about response formatting and API key usage.

---

#### 1.5 Documentation and Reference Tools

**Overview:**  
This block provides AI with access to official Proxmox API documentation and wiki pages to improve command generation accuracy.

**Nodes Involved:**  
- Proxmox API Documentation  
- Proxmox API Wiki  
- Proxmox (live API tool)

**Node Details:**

- **Proxmox API Documentation**  
  - Type: LangChain Tool HTTP Request  
  - Role: Supplies AI with official API documentation URL for reference.  
  - Configuration: URL points to Proxmox API viewer.  
  - Inputs: AI Agent queries  
  - Outputs: Documentation content

- **Proxmox API Wiki**  
  - Type: LangChain Tool HTTP Request  
  - Role: Supplies AI with Proxmox API wiki content.  
  - Configuration: URL points to Proxmox VE API wiki page.  
  - Inputs: AI Agent queries  
  - Outputs: Wiki content

- **Proxmox**  
  - Type: LangChain Tool HTTP Request  
  - Role: Provides live Proxmox cluster status and node info for AI validation.  
  - Configuration: Uses Proxmox API credentials and header auth.  
  - Inputs: AI Agent queries  
  - Outputs: Live data

---

#### 1.6 Error Handling and Output Validation

**Overview:**  
This block ensures AI output correctness, fixes parsing errors automatically, and handles invalid or incomplete user inputs gracefully.

**Nodes Involved:**  
- Structured Output Parser  
- Auto-fixing Output Parser  
- Switch (routes Invalid response_type)  
- Sticky Notes (HTTP methods, API key instructions)

**Node Details:**

- **Structured Output Parser**  
  - Parses AI output strictly according to JSON schema.

- **Auto-fixing Output Parser**  
  - Attempts to correct AI output if parsing fails by re-prompting AI.

- **Switch**  
  - Routes invalid AI outputs to appropriate handling nodes or terminates workflow.

- **Sticky Notes**  
  - Provide instructions on API key creation, HTTP methods explanation, and trigger usage.

---

### 3. Summary Table

| Node Name                      | Node Type                           | Functional Role                                | Input Node(s)                         | Output Node(s)                      | Sticky Note                                                                                         |
|--------------------------------|-----------------------------------|------------------------------------------------|-------------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------|
| Telegram Trigger               | Telegram Trigger                  | Receives Telegram messages to trigger workflow | -                                   | AI Agent                          |                                                                                                   |
| Gmail Trigger                 | Gmail Trigger                    | Polls Gmail inbox for new emails                | -                                   | AI Agent                          |                                                                                                   |
| Webhook                      | HTTP Webhook                     | Receives authenticated HTTP requests           | -                                   | AI Agent                          |                                                                                                   |
| When chat message received    | LangChain Chat Trigger           | Receives chat messages from n8n chat            | -                                   | AI Agent                          | "You can use any trigger as input, a chat, telegram, email etc"                                   |
| AI Agent                     | LangChain Agent                 | Parses user input into structured Proxmox API commands | Telegram Trigger, Gmail Trigger, Webhook, When chat message received, Proxmox API Docs, Wiki, Proxmox tool, Google Gemini Chat Model | Switch                            | Detailed prompt with instructions for generating valid Proxmox API commands                       |
| Google Gemini Chat Model      | LangChain LM Chat Google Gemini | AI language model for natural language processing | AI Agent                            | Structured Output Parser          |                                                                                                   |
| Proxmox API Documentation    | LangChain Tool HTTP Request      | Provides Proxmox API docs to AI Agent           | AI Agent                            | AI Agent                         | "This is Proxmox API Documentation ensure to read the details from here"                          |
| Proxmox API Wiki             | LangChain Tool HTTP Request      | Provides Proxmox API wiki content to AI Agent   | AI Agent                            | AI Agent                         | "Get the proxmox API details from Proxmox Wiki"                                                  |
| Proxmox                     | LangChain Tool HTTP Request      | Provides live Proxmox cluster status to AI Agent | AI Agent                            | AI Agent                         | "This is Proxmox which will help you to get the details of existing Proxmox installations..."     |
| Structured Output Parser     | LangChain Output Parser Structured | Parses AI output into strict JSON format         | Google Gemini Chat Model            | Auto-fixing Output Parser        |                                                                                                   |
| Auto-fixing Output Parser    | LangChain Output Parser Autofixing | Fixes AI output parsing errors                   | Structured Output Parser            | AI Agent                        |                                                                                                   |
| Switch                      | Switch                          | Routes API calls based on HTTP method            | AI Agent                           | HTTP Request, If, If1             |                                                                                                   |
| If                         | If                              | Checks if `details` field exists for POST routing | Switch (POST output)                | HTTP Request1, HTTP Request2      |                                                                                                   |
| If1                        | If                              | Checks if `details` field missing/empty for DELETE routing | Switch (DELETE output)              | HTTP Request3, HTTP Request4      |                                                                                                   |
| HTTP Request                | HTTP Request                    | Executes GET requests to Proxmox API             | Switch (GET output)                 | Structure Response               |                                                                                                   |
| HTTP Request1               | HTTP Request                    | Executes POST requests with payload               | If (details present)                | Merge                           |                                                                                                   |
| HTTP Request2               | HTTP Request                    | Alternative POST request node                      | If (details present)                | Merge                           |                                                                                                   |
| HTTP Request3               | HTTP Request                    | Executes DELETE requests                           | If1 (details missing)               | Merge1                          |                                                                                                   |
| HTTP Request4               | HTTP Request                    | Alternative DELETE request node                    | If1 (details missing)               | Merge1                          |                                                                                                   |
| Merge                      | Merge                          | Combines POST request outputs                      | HTTP Request1, HTTP Request2       | Structure Response               |                                                                                                   |
| Merge1                     | Merge                          | Combines DELETE request outputs                    | HTTP Request3, HTTP Request4       | Structure Response               |                                                                                                   |
| Structure Response          | Code                           | Combines API response fields into a single string | Merge, Merge1                     | AI Agent1                       |                                                                                                   |
| AI Agent1                  | LangChain Agent                 | Generates user-friendly summary of Proxmox info  | Structure Response, Google Gemini Chat Model2 | -                             | "This agent will convert the response from proxmox to meaningful explanation"                     |
| Structgure Response from Proxmox | Code                           | Parses Proxmox UPID string into detailed fields   | Merge, Merge1                     | Format Response and Hide Sensitive Data |                                                                                                   |
| Format Response and Hide Sensitive Data | Code                           | Formats parsed data into readable message, hides sensitive info | Structgure Response from Proxmox | -                             |                                                                                                   |
| Sticky Note                 | Sticky Note                    | Instructions on API key creation and usage         | -                                 | -                               | "Create Credentials in Proxmox Data Center as API Key..."                                        |
| Sticky Note1                | Sticky Note                    | Notes on trigger usage                              | -                                 | -                               | "You can use any trigger as input, a chat, telegram, email etc"                                  |
| Sticky Note2                | Sticky Note                    | Overview of Proxmox Custom AI Agent                 | -                                 | -                               | "It uses the intelligence provided including Proxmox API Wiki, Cluster Linked and API Documentation" |
| Sticky Note3                | Sticky Note                    | HTTP methods explanation                            | -                                 | -                               | "GET Retrieve resources, POST Create or trigger actions, PUT Update, DELETE Delete, OPTIONS, PATCH" |
| Sticky Note4                | Sticky Note                    | Notes on Proxmox Custom AI Agent (Get)              | -                                 | -                               | "This agent will convert the response from proxmox to meaningful explanation"                    |
| Sticky Note5                | Sticky Note                    | Notes on server action response                      | -                                 | -                               | "Created or triggered an action on the server. Response will come back here"                     |
| Sticky Note6                | Sticky Note                    | Trigger usage notes                                  | -                                 | -                               | "You can use any trigger as input, a chat, telegram, email etc. Possibilities are limitless."    |
| Sticky Note7                | Sticky Note                    | Developer credits and support links                  | -                                 | -                               | "Developed by Amjid Ali. Support via PayPal: http://paypal.me/pmptraining"                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Input Triggers**  
   - Add a **Telegram Trigger** node: Configure with your Telegram API credentials; listen to "message" updates.  
   - Add a **Gmail Trigger** node: Configure with Gmail OAuth2 credentials; set to poll every minute.  
   - Add an **HTTP Webhook** node: Set a unique webhook path; enable header authentication with Proxmox API credentials.  
   - Add a **When chat message received** node: Use default settings for n8n chat interface.

2. **Add AI Language Model Node**  
   - Add a **Google Gemini Chat Model** node: Configure with Google PaLM API credentials; set model to "models/gemini-2.0-flash-exp".

3. **Add Proxmox Reference Tools**  
   - Add **Proxmox API Documentation** node (LangChain Tool HTTP Request): URL `https://pve.proxmox.com/pve-docs/api-viewer/index.html`.  
   - Add **Proxmox API Wiki** node (LangChain Tool HTTP Request): URL `https://pve.proxmox.com/wiki/Proxmox_VE_API`.  
   - Add **Proxmox** node (LangChain Tool HTTP Request): URL `https://10.11.12.101:8006/api2/json/cluster/status`; use Proxmox API credentials with header auth.

4. **Create AI Agent Node**  
   - Add **AI Agent** node (LangChain Agent):  
     - Use "reActAgent" prompt type.  
     - Paste detailed prompt instructions for parsing user input into Proxmox API commands (including validation, defaults, and examples).  
     - Connect input triggers and Proxmox reference tools as AI tools.  
     - Connect Google Gemini Chat Model as AI language model.  
     - Connect Structured Output Parser as output parser.

5. **Add Output Parsers**  
   - Add **Structured Output Parser** node: Use JSON schema example for expected AI output format.  
   - Add **Auto-fixing Output Parser** node: Configure with prompt to fix parsing errors automatically.

6. **Add Switch Node for HTTP Method Routing**  
   - Add **Switch** node: Configure rules to route based on `response_type` field (`GET`, `POST`, `PUT`, `DELETE`, `OPTIONS`, `Invalid`).

7. **Add HTTP Request Nodes for API Calls**  
   - **HTTP Request (GET)**: Configure with URL `https://10.11.12.101:8006/api2/json{{ $json.output.url }}`, method GET, header auth credentials, allow unauthorized certs.  
   - **HTTP Request1 (POST)**: Same base URL, method POST, JSON body from `details`, header auth.  
   - **HTTP Request2 (POST alternative)**: Same as HTTP Request1, used conditionally.  
   - **HTTP Request3 (DELETE)**: Same base URL, method DELETE, header auth.  
   - **HTTP Request4 (DELETE alternative)**: Same as HTTP Request3.

8. **Add If Nodes to Check Payload Presence**  
   - **If** node: Checks if `details` field exists and is not empty for POST requests; routes to HTTP Request1 or HTTP Request2.  
   - **If1** node: Checks if `details` field is empty or missing for DELETE requests; routes to HTTP Request3 or HTTP Request4.

9. **Add Merge Nodes**  
   - **Merge** node: Combines outputs from POST requests.  
   - **Merge1** node: Combines outputs from DELETE requests.

10. **Add Response Processing Nodes**  
    - **Structure Response** (Code): Combine API response fields into a single string for summary.  
    - **AI Agent1** (LangChain Agent): Use Google Gemini Chat Model2; prompt to generate user-friendly summaries from combined data.  
    - **Structgure Response from Proxmox** (Code): Parse Proxmox UPID string into detailed fields.  
    - **Format Response and Hide Sensitive Data** (Code): Format parsed data into readable message, convert timestamp, hide sensitive info.

11. **Connect Nodes**  
    - Connect input triggers to AI Agent.  
    - AI Agent outputs to Switch.  
    - Switch routes to HTTP Request nodes or If nodes.  
    - If nodes route to appropriate HTTP Request nodes.  
    - HTTP Request nodes connect to Merge nodes.  
    - Merge nodes connect to Structure Response.  
    - Structure Response connects to AI Agent1.  
    - AI Agent1 outputs final formatted message.

12. **Credential Setup**  
    - Create Proxmox API credentials in n8n as Header Auth with header:  
      - Name: `Authorization`  
      - Value: `PVEAPIToken=<user>@<realm>!<token-id>=<token-value>`  
    - Create Google PaLM API credentials for Google Gemini nodes.  
    - Configure Telegram and Gmail credentials as needed.

13. **Add Sticky Notes**  
    - Add notes for API key instructions, HTTP methods, trigger usage, AI agent overview, and developer credits for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Watch Video on Youtube: https://www.youtube.com/watch?v=XykvpCj9wDA                                                               | Official video demonstrating the workflow                                                       |
| API Key for Proxmox: Create credentials in Proxmox Data Center as API Key; add to n8n as Header Auth with Authorization header. | Sticky Note in workflow explaining API key setup                                                |
| HTTP Methods Explanation: GET, POST, PUT, DELETE, OPTIONS, PATCH with their roles in Proxmox API                                | Sticky Note describing HTTP methods                                                             |
| Trigger Flexibility: Any input trigger can be used (chat, Telegram, email, webhook, cloud platform, web app)                     | Sticky Note emphasizing limitless input possibilities                                           |
| Developed by Amjid Ali; Support via PayPal: http://paypal.me/pmptraining; Contact: amjid@amjidali.com                             | Developer credits and support information                                                       |
| Workflow designed for Proxmox 7.x and above; test in non-production environment first                                            | Important deployment note from description                                                      |
| AI Models Supported: Google Gemini, OpenAI, Claude, Ollama                                                                        | Extensibility note for AI model integration                                                     |
| Proxmox API Documentation: https://pve.proxmox.com/pve-docs/api-viewer/index.html                                                | Official API documentation used as AI reference                                                |
| Proxmox API Wiki: https://pve.proxmox.com/wiki/Proxmox_VE_API                                                                     | Wiki page used as AI knowledge source                                                          |

---

This comprehensive documentation enables advanced users and AI agents to understand, reproduce, and extend the Proxmox AI Agent workflow in n8n with confidence, anticipating integration challenges and ensuring robust operation.