IPQualityScore API MCP Server

https://n8nworkflows.xyz/workflows/ipqualityscore-api-mcp-server-5584


# IPQualityScore API MCP Server

### 1. Workflow Overview

This workflow, **IPQualityScore API MCP Server**, exposes three distinct IPQualityScore API operations as a unified MCP (Modular Chatbot Protocol) server interface for AI agents. It allows AI agents to query email validation, phone number validation, and URL malware scanning through a single webhook endpoint.

The workflow is structured into the following logical blocks:

- **1.1 Setup and Documentation**: Provides setup instructions and workflow overview notes for users.
- **1.2 MCP Server Trigger**: The main entry point via an MCP trigger node that listens for AI agent requests.
- **1.3 API Operation Blocks**: Three parallel HTTP Request nodes each handling a specific IPQualityScore API endpoint:
  - Email validation
  - Phone number validation
  - Malicious URL scanning

Parameters for API calls are dynamically populated using `$fromAI()` expressions, allowing AI agents to pass required inputs seamlessly. The responses from IPQualityScore are forwarded directly back to the AI agent in their original structure.


---

### 2. Block-by-Block Analysis

#### 2.1 Setup and Documentation

**Overview:**  
This block includes sticky notes that provide setup instructions, usage notes, and a high-level overview of the workflow's function. These nodes do not interact programmatically but serve as documentation for users and maintainers.

**Nodes Involved:**  
- Setup Instructions (Sticky Note)  
- Workflow Overview (Sticky Note)

**Node Details:**

- **Setup Instructions**  
  - Type: Sticky Note  
  - Role: Provides detailed step-by-step instructions on how to import, authenticate, activate, and connect the workflow with AI agents. Also includes customization tips and a Discord support link.  
  - Inputs/Outputs: None (informational only)  
  - Key info:  
    - Use of API Key auth  
    - MCP webhook URL usage  
    - AI agent parameter population using `$fromAI()`  
    - Suggests adding error handling, logging, or data transformations if needed  
  - Edge cases: None (non-executable)

- **Workflow Overview**  
  - Type: Sticky Note  
  - Role: Summarizes the workflow operation, emphasizing the three API operations exposed and how the MCP trigger and HTTP request nodes work together.  
  - Inputs/Outputs: None  
  - Edge cases: None

---

#### 2.2 MCP Server Trigger

**Overview:**  
This node is the main trigger that listens for incoming MCP requests from AI agents. It serves as the central server endpoint that routes requests to the appropriate API operation nodes.

**Nodes Involved:**  
- IPQualityScore MCP Server (MCP Trigger)

**Node Details:**

- **IPQualityScore MCP Server**  
  - Type: MCP Trigger (from `@n8n/n8n-nodes-langchain`)  
  - Role: Entry point webhook that accepts AI agent requests at the path `/ipqualityscore-mcp`.  
  - Configuration:  
    - Webhook path: `ipqualityscore-mcp`  
  - Inputs: Incoming MCP requests with parameters populated by AI agent via `$fromAI()` expressions  
  - Outputs: Routes data to one of the three HTTP request nodes depending on the AI agent's tool selection  
  - Version specifics: Requires n8n version supporting MCP trigger node  
  - Edge cases:  
    - Webhook URL must be publicly accessible  
    - Authentication and rate limiting must be handled externally if needed  
    - Failure if MCP trigger misconfigured or offline

---

#### 2.3 API Operation Blocks

Each operation block consists of one HTTP Request Tool node that calls a specific IPQualityScore API endpoint. These nodes use dynamic URL construction with `$fromAI()` to inject API key and user input parameters.

---

##### 2.3.1 Email Validation

**Overview:**  
This block validates an email address via IPQualityScore's email validation API.

**Nodes Involved:**  
- Validate Email Address (HTTP Request Tool)  
- Grid Note 1 (Sticky Note)

**Node Details:**

- **Validate Email Address**  
  - Type: HTTP Request Tool  
  - Role: Calls `https://ipqualityscore.com/api/json/email/{API_KEY}/{EMAIL}` to validate the email address.  
  - Configuration:  
    - URL uses expressions to dynamically insert:  
      - API Key: `$fromAI('YOUR_API_KEY_HERE')`  
      - User Email: `$fromAI('USER_EMAIL_HERE')`  
    - HTTP method: GET (default for HTTP Request Tool)  
    - No additional query parameters  
  - Inputs: Data from MCP Trigger node (input passed from AI agent)  
  - Outputs: API JSON response forwarded back to MCP for AI agent consumption  
  - Edge cases:  
    - Missing or invalid API key leads to auth errors  
    - Invalid email format or empty parameter leads to API errors  
    - Network or timeout errors  
  - Version: Requires HTTP Request Tool v4.2 or higher for expression support

- **Grid Note 1**  
  - Type: Sticky Note  
  - Role: Labels the block as "Email Validation" for clarity  
  - Inputs/Outputs: None

---

##### 2.3.2 Phone Number Validation

**Overview:**  
This block validates a phone number using IPQualityScore's phone validation API.

**Nodes Involved:**  
- Validate Phone Number (HTTP Request Tool)  
- Grid Note 2 (Sticky Note)

**Node Details:**

- **Validate Phone Number**  
  - Type: HTTP Request Tool  
  - Role: Calls `https://ipqualityscore.com/api/json/phone/{API_KEY}/{PHONE}` with optional country query parameter.  
  - Configuration:  
    - URL path parameters:  
      - API Key: `$fromAI('YOUR_API_KEY_HERE')`  
      - User Phone: `$fromAI('USER_PHONE_HERE')`  
    - Query parameter:  
      - `country`: optional, passed as `$fromAI('country')`  
    - HTTP method: GET  
  - Inputs: MCP Trigger node output  
  - Outputs: API JSON response to MCP server output  
  - Edge cases:  
    - Missing API key or phone number causes errors  
    - Invalid country code may affect validation accuracy  
    - Network/timeout or API limits  
  - Version: Requires HTTP Request Tool v4.2+

- **Grid Note 2**  
  - Type: Sticky Note  
  - Role: Marks this block as "Phone Validation"  
  - Inputs/Outputs: None

---

##### 2.3.3 Malicious URL Scanner

**Overview:**  
This block scans a URL for malware or malicious content via IPQualityScore's URL scanning API.

**Nodes Involved:**  
- Scan URL for Malware (HTTP Request Tool)  
- Grid Note 3 (Sticky Note)

**Node Details:**

- **Scan URL for Malware**  
  - Type: HTTP Request Tool  
  - Role: Calls `https://ipqualityscore.com/api/json/url/{API_KEY}/{URL}` to scan for malicious content.  
  - Configuration:  
    - URL parameters:  
      - API Key: `$fromAI('YOUR_API_KEY_HERE')`  
      - URL to scan: `$fromAI('URL_HERE')`  
    - HTTP method: GET  
  - Inputs: MCP Trigger output  
  - Outputs: API JSON response back to MCP server  
  - Edge cases:  
    - Missing API key or URL parameter leads to errors  
    - Malformed or unreachable URLs may cause API failures  
    - Timeout or network errors  
  - Version: HTTP Request Tool v4.2+

- **Grid Note 3**  
  - Type: Sticky Note  
  - Role: Labels the block as "Malicious Url Scanner"  
  - Inputs/Outputs: None

---

### 3. Summary Table

| Node Name              | Node Type                 | Functional Role                       | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                          |
|------------------------|---------------------------|-------------------------------------|------------------------|-------------------------|----------------------------------------------------------------------------------------------------|
| Setup Instructions     | Sticky Note               | Setup and usage documentation       | None                   | None                    | Provides detailed setup instructions, API key usage, MCP URL info, customization tips, support link|
| Workflow Overview      | Sticky Note               | Workflow summary and explanation    | None                   | None                    | Explains workflow purpose, MCP trigger role, API operations, and AI parameter usage                |
| IPQualityScore MCP Server | MCP Trigger               | Main webhook entrypoint for AI requests | None                   | Validate Email Address, Validate Phone Number, Scan URL for Malware |                                                                                                    |
| Validate Email Address | HTTP Request Tool         | Calls email validation API           | IPQualityScore MCP Server | MCP Server output       |                                                                                                    |
| Grid Note 1            | Sticky Note               | Labels "Email Validation" block     | None                   | None                    |                                                                                                    |
| Validate Phone Number  | HTTP Request Tool         | Calls phone number validation API    | IPQualityScore MCP Server | MCP Server output       |                                                                                                    |
| Grid Note 2            | Sticky Note               | Labels "Phone Validation" block     | None                   | None                    |                                                                                                    |
| Scan URL for Malware   | HTTP Request Tool         | Calls malicious URL scanner API      | IPQualityScore MCP Server | MCP Server output       |                                                                                                    |
| Grid Note 3            | Sticky Note               | Labels "Malicious Url Scanner" block | None                   | None                    |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and set the timezone to America/New_York (optional).**

2. **Add a Sticky Note named "Setup Instructions":**  
   - Paste the detailed setup instructions from section 2.1.  
   - Set color to 4, height to about 1060px.

3. **Add a Sticky Note named "Workflow Overview":**  
   - Paste the workflow summary content from section 2.1.  
   - Set width to 320px and height to 920px.

4. **Add an MCP Trigger node named "IPQualityScore MCP Server":**  
   - Set the webhook path to `ipqualityscore-mcp`.  
   - No authentication required here but ensure the webhook URL is publicly accessible.  
   - This node will serve as the main entrypoint for AI agent requests.

5. **Add a Sticky Note "Grid Note 1" near the Email Validation block:**  
   - Content: "## Email Validation"  
   - Color: 7, Height: ~220px

6. **Add an HTTP Request Tool node named "Validate Email Address":**  
   - Set URL to:  
     `https://ipqualityscore.com/api/json/email/{{ $fromAI('YOUR_API_KEY_HERE', '(Required) YOUR_API_KEY_HERE', 'string') }}/{{ $fromAI('USER_EMAIL_HERE', '(Required) USER_EMAIL_HERE', 'string') }}`  
   - HTTP method: GET  
   - Leave other options default.  
   - Tool Description: "Email Validation" with parameters explained for API key and user email.  
   - Connect the MCP Trigger node output to this node's input with the role `ai_tool`.

7. **Add a Sticky Note "Grid Note 2" near the Phone Validation block:**  
   - Content: "## Phone Validation"  
   - Color: 7, Height: ~220px

8. **Add an HTTP Request Tool node named "Validate Phone Number":**  
   - Set URL to:  
     `https://ipqualityscore.com/api/json/phone/{{ $fromAI('YOUR_API_KEY_HERE', '(Required) YOUR_API_KEY_HERE', 'string') }}/{{ $fromAI('USER_PHONE_HERE', '(Required) USER_PHONE_HERE', 'string') }}`  
   - HTTP method: GET  
   - Add Query Parameter:  
     - Name: `country`  
     - Value: `={{ $fromAI('country', 'country', 'string') }}`  
   - Tool Description: "Phone Validation" with parameters for API key, phone, and optional country.  
   - Connect MCP Trigger output to this node's input with the role `ai_tool`.

9. **Add a Sticky Note "Grid Note 3" near the URL scanner block:**  
   - Content: "## Malicious Url Scanner"  
   - Color: 7, Height: ~220px, Width: ~280px

10. **Add an HTTP Request Tool node named "Scan URL for Malware":**  
    - Set URL to:  
      `https://ipqualityscore.com/api/json/url/{{ $fromAI('YOUR_API_KEY_HERE', '(Required) YOUR_API_KEY_HERE', 'string') }}/{{ $fromAI('URL_HERE', '(Required) URL_HERE', 'string') }}`  
    - HTTP method: GET  
    - Tool Description: "Malicious URL Scanner" with parameters for API key and URL.  
    - Connect MCP Trigger output to this node's input with the role `ai_tool`.

11. **Ensure all HTTP Request Tool nodes are connected as `ai_tool` outputs from the MCP Trigger.**

12. **Activate the workflow.**

13. **Provide the MCP webhook URL to your AI agent for integrations.**

14. **API Key Management:**  
    - The API key is passed dynamically via `$fromAI()` expressions in each HTTP Request node.  
    - Your AI agent must supply the API key as part of the request parameters.

15. **Testing:**  
    - Test each API endpoint by sending appropriate parameters through the MCP webhook and verify the JSON response matches IPQualityScore's API response.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                               | Context or Link                                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| For integration guidance and custom automations, you can join the Discord community at: https://discord.me/cfomodz                                         | Support link included in Setup Instructions sticky note                                                                         |
| Further documentation on MCP and n8n Langchain tools can be found here: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Official n8n documentation for MCP trigger and related tool nodes                                                               |
| The workflow maintains the original API response structure, easing integration into AI agents without needing custom data transformations.                  | Workflow design note                                                                                                             |
| To customize error handling, logging, or data transformation, consider adding Function or Set nodes after HTTP requests before returning results.          | Customization advice from Setup Instructions sticky note                                                                          |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow created with n8n, an integration and automation tool. It fully complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.