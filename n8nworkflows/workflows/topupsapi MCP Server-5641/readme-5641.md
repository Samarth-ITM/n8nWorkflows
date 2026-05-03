topupsapi MCP Server

https://n8nworkflows.xyz/workflows/topupsapi-mcp-server-5641


# topupsapi MCP Server

### 1. Workflow Overview

The **topupsapi MCP Server** workflow serves as an MCP (Multi-Channel Platform) server interface that exposes two key operations of the topupsapi API (which is a simple polling API) to AI agents. It enables AI agents to interact with the polls API by listing all poll questions and creating new poll questions through an MCP-compatible webhook endpoint.

The workflow is logically grouped into these blocks:

- **1.1 Setup and Documentation**: Provides setup instructions and workflow overview notes for users and maintainers.
- **1.2 MCP Server Trigger**: The main entry point that listens for incoming AI agent requests via an MCP webhook.
- **1.3 API Operations**: Contains two HTTP Request Tool nodes that implement the two API endpoints:
  - List all poll questions
  - Create a new poll question

The workflow is designed to automatically populate API request parameters using AI expressions (`$fromAI()`), returning the responses with the original structure directly to the AI agent.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup and Documentation

- **Overview:**  
  This block provides detailed usage instructions, setup steps, and a high-level explanation of the workflow’s purpose and capabilities. It helps users understand how to import, activate, and connect the MCP server for AI integration.

- **Nodes Involved:**  
  - `Setup Instructions` (Sticky Note)  
  - `Workflow Overview` (Sticky Note)  
  - `Sticky Note` (label for "Questions" section)

- **Node Details:**

  - **Setup Instructions**  
    - Type: Sticky Note  
    - Role: Documentation of setup steps, usage notes, customization tips, and support links.  
    - Configuration Highlights: Contains markdown-formatted instructions covering import, activation, authentication (none required), AI parameter auto-population, and links to discord support and n8n docs.  
    - Inputs/Outputs: None (informational only)  
    - Edge Cases: None

  - **Workflow Overview**  
    - Type: Sticky Note  
    - Role: Describes the workflow’s architecture, API supported, and how it operates as an MCP server.  
    - Configuration Highlights: Markdown content summarizing the API, MCP trigger usage, HTTP request nodes, AI expressions, and available endpoints.  
    - Inputs/Outputs: None (informational only)  
    - Edge Cases: None

  - **Sticky Note ("Questions")**  
    - Type: Sticky Note  
    - Role: Visual label for the API operations block (grouping the question-related nodes).  
    - Inputs/Outputs: None  

---

#### 2.2 MCP Server Trigger

- **Overview:**  
  This block contains the MCP trigger node that acts as the server endpoint for AI agents to send requests. It listens on a webhook path and initiates the workflow when triggered.

- **Nodes Involved:**  
  - `topupsapi MCP Server` (MCP Trigger node)

- **Node Details:**

  - **topupsapi MCP Server**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger` (MCP trigger node)  
    - Role: Entry point for AI agents’ API requests into the workflow.  
    - Configuration:  
      - Webhook path: `"topupsapi-mcp"` (exposed endpoint)  
      - No authentication configured (open endpoint)  
    - Inputs: External HTTP requests from AI agents  
    - Outputs: Triggers subsequent nodes handling API operations  
    - Version-specific: Requires n8n version supporting MCP nodes and Langchain integration  
    - Edge Cases:  
      - Network errors or webhook misconfiguration could prevent triggering  
      - No auth means open access; security considerations apply  
      - Payload validation errors if AI agent sends malformed data

---

#### 2.3 API Operations

- **Overview:**  
  Implements two core HTTP API calls to the polls API: listing all questions and creating a new question. Parameters for the create operation are dynamically populated from AI input using `$fromAI()` expressions.

- **Nodes Involved:**  
  - `Get Questions 1` (HTTP Request Tool)  
  - `Create Question 1` (HTTP Request Tool)

- **Node Details:**

  - **Get Questions 1**  
    - Type: `httpRequestTool`  
    - Role: Retrieves the list of all poll questions via GET request to the polls API.  
    - Configuration:  
      - URL: `https://polls.apiblueprint.org/questions`  
      - Method: GET (default)  
      - No body or additional parameters  
      - Tool description: "List All Questions"  
    - Inputs: Triggered by MCP Server node requests for listing questions  
    - Outputs: Returns API response directly to the MCP trigger (AI agent)  
    - Edge Cases:  
      - API downtime or network issues  
      - Unexpected response format  
      - Rate limiting by polls API

  - **Create Question 1**  
    - Type: `httpRequestTool`  
    - Role: Creates a new poll question by POSTing to the polls API.  
    - Configuration:  
      - URL: `https://polls.apiblueprint.org/questions`  
      - Method: POST  
      - Body parameters:  
        - `choices`: Populated from AI input as JSON (`$fromAI('choices', 'Choices', 'json')`)  
        - `question`: Populated from AI input as string (`$fromAI('question', 'Question', 'string')`)  
      - Sends body as JSON payload  
      - Tool description includes parameter explanations  
    - Inputs: Triggered by MCP Server node requests for question creation  
    - Outputs: Returns API response directly to the MCP trigger (AI agent)  
    - Edge Cases:  
      - Missing or invalid `choices` or `question` parameters causing API errors  
      - Payload serialization errors from AI expressions  
      - API errors such as 400 Bad Request or 500 Server Error

---

### 3. Summary Table

| Node Name           | Node Type                       | Functional Role                      | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                           |
|---------------------|--------------------------------|------------------------------------|------------------------|-----------------------|-----------------------------------------------------------------------------------------------------|
| Setup Instructions   | Sticky Note                    | Setup and usage documentation      | None                   | None                  | Contains detailed setup instructions, usage notes, customization tips, and support links.           |
| Workflow Overview    | Sticky Note                    | Workflow purpose and architecture  | None                   | None                  | Describes workflow functionality, API usage, and MCP server integration.                            |
| Sticky Note (Questions) | Sticky Note                    | Visual label for API operations    | None                   | None                  |                                                                                                     |
| topupsapi MCP Server | MCP Trigger                   | Entry point webhook for AI agents  | External HTTP requests | Get Questions 1, Create Question 1 | No authentication required; exposed MCP webhook endpoint "topupsapi-mcp".                          |
| Get Questions 1      | HTTP Request Tool             | List all poll questions via API    | topupsapi MCP Server    | MCP Server (response)  | Performs GET request to polls API to list questions.                                                |
| Create Question 1    | HTTP Request Tool             | Create a new poll question          | topupsapi MCP Server    | MCP Server (response)  | Performs POST request with AI-populated parameters to create a question.                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in your n8n instance and name it "topupsapi MCP Server".

2. **Add Sticky Note nodes** for documentation (optional but recommended):  
   - Create a Sticky Note named `Setup Instructions` containing setup and usage instructions.  
   - Create a Sticky Note named `Workflow Overview` summarizing the workflow purpose, API info, and MCP integration.  
   - Create a Sticky Note labeled `Questions` as a visual grouping label.

3. **Add an MCP Trigger node**:  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Name: `topupsapi MCP Server`  
   - Set the webhook path to `"topupsapi-mcp"`.  
   - No authentication is required (leave auth settings empty).  
   - This node listens for incoming requests from AI agents.

4. **Add an HTTP Request Tool node for listing questions**:  
   - Name: `Get Questions 1`  
   - URL: `https://polls.apiblueprint.org/questions`  
   - HTTP Method: GET (default)  
   - No additional parameters or body required.  
   - Description: "List All Questions"

5. **Add an HTTP Request Tool node for creating questions**:  
   - Name: `Create Question 1`  
   - URL: `https://polls.apiblueprint.org/questions`  
   - HTTP Method: POST  
   - Enable "Send Body" with JSON parameters:  
     - `choices`: Set value expression to `{{$fromAI('choices', 'Choices', 'json')}}`  
     - `question`: Set value expression to `{{$fromAI('question', 'Question', 'string')}}`  
   - Description: "Create a New Question with parameters choices (optional) and question (optional)"

6. **Connect the MCP Trigger node outputs** to both HTTP Request Tool nodes:  
   - The MCP trigger should route incoming AI requests to either `Get Questions 1` or `Create Question 1` based on the requested operation (this routing logic may be implicit or controlled by the MCP system).

7. **Ensure that parameter auto-population expressions (`$fromAI()`) are correctly recognized** by your n8n version and Langchain MCP integration.

8. **Activate the workflow** to make it live and start accepting requests.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                           | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Ping me on Discord for integration guidance and custom automations.                                                                                    | https://discord.me/cfomodz                                                                                       |
| Check the n8n documentation for more on MCP and Langchain integration.                                                                                 | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                     |
| Parameters in HTTP requests are auto-populated by AI using `$fromAI()` expressions, which requires n8n with Langchain MCP integration support.         | Related to AI-driven parameter handling in HTTP Request Tool nodes                                              |
| No authentication is configured on the MCP webhook, so secure access should be considered if deploying in production environments.                     | Security note regarding open endpoint                                                                            |
| The workflow exposes two API endpoints of the polls API: list questions (GET) and create question (POST).                                              | Functional summary                                                                                               |

---

**Disclaimer:** The provided text is derived exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.