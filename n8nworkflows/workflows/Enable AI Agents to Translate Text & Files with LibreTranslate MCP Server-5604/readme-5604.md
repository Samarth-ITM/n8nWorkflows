Enable AI Agents to Translate Text & Files with LibreTranslate MCP Server

https://n8nworkflows.xyz/workflows/enable-ai-agents-to-translate-text---files-with-libretranslate-mcp-server-5604


# Enable AI Agents to Translate Text & Files with LibreTranslate MCP Server

### 1. Workflow Overview

This workflow exposes the LibreTranslate API as a Modular Chat Protocol (MCP) server endpoint, enabling AI agents to translate text and files seamlessly via six distinct API operations. It is designed primarily for integration scenarios where AI agents require translation services through an MCP interface, without needing authentication. The workflow logically divides into three main functional blocks:

- **1.1 MCP Server Setup and Configuration:** Sets up the MCP server webhook endpoint and provides setup instructions and workflow overview notes.
- **1.2 Translation API Operations:** Implements four core translation-related API endpoints — detecting text language, listing supported languages, translating text, and translating files.
- **1.3 Frontend Settings & Feedback:** Provides access to frontend settings and allows submitting translation suggestions.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server Setup and Configuration

**Overview:**  
This block initializes the MCP trigger node which acts as the server endpoint handling incoming requests from AI agents. It also contains detailed sticky notes with setup instructions and a high-level overview describing the workflow’s purpose, usage, and customization tips.

**Nodes Involved:**  
- Setup Instructions (Sticky Note)  
- Workflow Overview (Sticky Note)  
- LibreTranslate MCP Server (MCP Trigger Node)

**Node Details:**

- **Setup Instructions**  
  - Type: Sticky Note  
  - Role: Provides comprehensive textual setup and usage instructions including how to import, authenticate (none required), activate, and connect AI agents to the MCP server webhook.  
  - Configuration: Color-coded (color 4), large size to contain detailed instructions, including links to Discord and n8n documentation for support.  
  - Inputs/Outputs: None (informational only).  
  - Edge Cases: None.

- **Workflow Overview**  
  - Type: Sticky Note  
  - Role: Summarizes the workflow purpose, outlining available API endpoints and their grouping (Translate, Frontend, Feedback).  
  - Configuration: Medium size with markdown content, no inputs/outputs.  
  - Edge Cases: None.

- **LibreTranslate MCP Server**  
  - Type: MCP Trigger Node (from n8n Langchain integration)  
  - Role: Listens on a webhook path `/libretranslate-mcp` for AI agent requests, serving as the entry point for all translation-related operations.  
  - Configuration:  
    - `path`: "libretranslate-mcp" (defines the webhook URL suffix)  
    - No authentication required.  
  - Inputs: External HTTP requests from AI agents via MCP protocol.  
  - Outputs: Routes requests to downstream HTTP Request Tool nodes depending on requested operation.  
  - Edge Cases: Potential webhook configuration errors, network issues, malformed MCP requests.  
  - Version: Requires n8n with Langchain MCP Trigger node support.

---

#### 2.2 Translation API Operations

**Overview:**  
This block contains nodes that connect to LibreTranslate API endpoints to perform translation-related functions. Each node corresponds to one API endpoint and processes AI agent requests routed from the MCP trigger.

**Nodes Involved:**  
- Detect Text Language (HTTP Request Tool)  
- List Supported Languages (HTTP Request Tool)  
- Translate Text (HTTP Request Tool)  
- Translate File (HTTP Request Tool)  
- Sticky Note (Translate)

**Node Details:**

- **Sticky Note (Translate)**  
  - Type: Sticky Note  
  - Role: Provides a header label grouping the four translation endpoints visually.  
  - Configuration: Color 2, medium size.  
  - Inputs/Outputs: None.

- **Detect Text Language**  
  - Type: HTTP Request Tool  
  - Role: Calls `POST http://libretranslate.local/detect` to detect the language of provided text.  
  - Configuration:  
    - Method: POST  
    - URL: `http://libretranslate.local/detect` (evaluated as expression)  
    - Tool Description: "Detect the language of a single text"  
    - Parameters are auto-populated dynamically via `$fromAI()` expressions by AI agents.  
  - Inputs: Requests from MCP Trigger routed through AI Tool connection.  
  - Outputs: Returns JSON detection response keeping original API structure.  
  - Edge Cases: Request timeouts, invalid text input, server unavailability.

- **List Supported Languages**  
  - Type: HTTP Request Tool  
  - Role: Calls `GET http://libretranslate.local/languages` to retrieve supported language list.  
  - Configuration:  
    - Method: GET (default)  
    - URL: `http://libretranslate.local/languages`  
    - Tool Description: "Retrieve list of supported languages"  
  - Inputs: MCP Trigger routed requests.  
  - Outputs: Returns supported languages JSON array.  
  - Edge Cases: Network issues, empty or malformed response.

- **Translate Text**  
  - Type: HTTP Request Tool  
  - Role: Calls `POST http://libretranslate.local/translate` to translate provided text between languages.  
  - Configuration:  
    - Method: POST  
    - URL: `http://libretranslate.local/translate`  
    - Tool Description: "Translate text from a language to another"  
    - Parameters dynamically provided by AI agent.  
  - Inputs: MCP Trigger routed requests.  
  - Outputs: JSON translation result.  
  - Edge Cases: Oversized text, invalid language codes, API errors.

- **Translate File**  
  - Type: HTTP Request Tool  
  - Role: Calls `POST http://libretranslate.local/translate_file` to translate files between languages.  
  - Configuration:  
    - Method: POST  
    - URL: `http://libretranslate.local/translate_file`  
    - Tool Description: "Translate file from a language to another"  
    - Parameters and file data dynamically injected.  
  - Inputs: MCP Trigger routed requests.  
  - Outputs: Translated file response.  
  - Edge Cases: File format unsupported, large file sizes, timeouts.

---

#### 2.3 Frontend Settings & Feedback

**Overview:**  
This block provides nodes interfacing with frontend-related settings and a feedback endpoint for submitting translation improvement suggestions.

**Nodes Involved:**  
- Sticky Note (Frontend)  
- Retrieve Frontend Settings (HTTP Request Tool)  
- Sticky Note (Feedback)  
- Submit Translation Suggestion (HTTP Request Tool)

**Node Details:**

- **Sticky Note (Frontend)**  
  - Type: Sticky Note  
  - Role: Header for frontend-related API operation.  
  - Configuration: Color 3, medium size.  
  - Inputs/Outputs: None.

- **Retrieve Frontend Settings**  
  - Type: HTTP Request Tool  
  - Role: Calls `GET http://libretranslate.local/frontend/settings` to retrieve frontend configuration.  
  - Configuration:  
    - Method: GET (default)  
    - URL: `http://libretranslate.local/frontend/settings`  
    - Tool Description: "Retrieve frontend specific settings"  
  - Inputs: MCP Trigger routed requests.  
  - Outputs: JSON frontend settings.  
  - Edge Cases: Server errors, malformed configuration data.

- **Sticky Note (Feedback)**  
  - Type: Sticky Note  
  - Role: Header for feedback submission endpoint.  
  - Configuration: Color 4, medium size.  
  - Inputs/Outputs: None.

- **Submit Translation Suggestion**  
  - Type: HTTP Request Tool  
  - Role: Calls `POST http://libretranslate.local/suggest` to submit translation improvement suggestions.  
  - Configuration:  
    - Method: POST  
    - URL: `http://libretranslate.local/suggest`  
    - Tool Description: "Submit a suggestion to improve a translation"  
    - Parameters dynamically set by AI agents.  
  - Inputs: MCP Trigger routed requests.  
  - Outputs: Confirmation or result of suggestion submission.  
  - Edge Cases: Validation errors, server rejection.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                        | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                           |
|-----------------------------|-------------------------|--------------------------------------|----------------------|----------------------|-----------------------------------------------------------------------------------------------------|
| Setup Instructions          | Sticky Note             | Setup and usage instructions          | None                 | None                 | Detailed setup instructions with links to Discord and n8n docs.                                    |
| Workflow Overview           | Sticky Note             | Workflow purpose and API endpoint summary | None                 | None                 | Overview of workflow features and operations.                                                      |
| LibreTranslate MCP Server   | MCP Trigger             | MCP webhook endpoint for AI agent requests | External HTTP requests | Detect Text Language, List Supported Languages, Translate Text, Translate File, Retrieve Frontend Settings, Submit Translation Suggestion |                                                                                                     |
| Sticky Note (Translate)     | Sticky Note             | Visual header for translation nodes   | None                 | None                 |                                                                                                     |
| Detect Text Language        | HTTP Request Tool       | Detect language of a given text       | LibreTranslate MCP Server |                      |                                                                                                     |
| List Supported Languages    | HTTP Request Tool       | Retrieve supported languages list     | LibreTranslate MCP Server |                      |                                                                                                     |
| Translate Text              | HTTP Request Tool       | Translate text between languages      | LibreTranslate MCP Server |                      |                                                                                                     |
| Translate File              | HTTP Request Tool       | Translate file content                 | LibreTranslate MCP Server |                      |                                                                                                     |
| Sticky Note (Frontend)      | Sticky Note             | Visual header for frontend settings   | None                 | None                 |                                                                                                     |
| Retrieve Frontend Settings  | HTTP Request Tool       | Get frontend configuration settings   | LibreTranslate MCP Server |                      |                                                                                                     |
| Sticky Note (Feedback)      | Sticky Note             | Visual header for feedback submission | None                 | None                 |                                                                                                     |
| Submit Translation Suggestion | HTTP Request Tool     | Submit translation improvement suggestions | LibreTranslate MCP Server |                      |                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note - Setup Instructions**  
   - Add a Sticky Note node.  
   - Set color to blue (#4).  
   - Enter detailed setup instructions covering workflow import, activation, getting webhook URL, AI agent connection, usage notes, customization tips, and links to Discord and n8n docs.

2. **Create Sticky Note - Workflow Overview**  
   - Add another Sticky Note node.  
   - Set width ~420, height ~820.  
   - Enter markdown summarizing the workflow, MCP server role, available endpoints grouped by function: Translate (4), Frontend (1), Feedback (1).

3. **Add MCP Trigger Node - LibreTranslate MCP Server**  
   - Type: Langchain MCP Trigger (requires Langchain package).  
   - Configure `path` parameter to "libretranslate-mcp".  
   - No authentication required.  
   - Position near the sticky notes.

4. **Add Sticky Note - Translate Section**  
   - Add a Sticky Note node with color yellow (#2).  
   - Title it "Translate" to group translation nodes visually.

5. **Create HTTP Request Tool - Detect Text Language**  
   - Type: HTTP Request Tool.  
   - Set method to POST.  
   - URL: `http://libretranslate.local/detect` (use expression if needed).  
   - Description: "Detect the language of a single text".  
   - Ensure parameters will be injected dynamically by AI agent expressions.  
   - Connect the MCP Trigger node output `ai_tool` to this node input.

6. **Create HTTP Request Tool - List Supported Languages**  
   - Type: HTTP Request Tool.  
   - Method: GET (default).  
   - URL: `http://libretranslate.local/languages`.  
   - Description: "Retrieve list of supported languages".  
   - Connect MCP Trigger `ai_tool` output to this node input.

7. **Create HTTP Request Tool - Translate Text**  
   - Type: HTTP Request Tool.  
   - Method: POST.  
   - URL: `http://libretranslate.local/translate`.  
   - Description: "Translate text from a language to another".  
   - Parameter injection via AI expressions.  
   - Connect MCP Trigger `ai_tool` output to this node input.

8. **Create HTTP Request Tool - Translate File**  
   - Type: HTTP Request Tool.  
   - Method: POST.  
   - URL: `http://libretranslate.local/translate_file`.  
   - Description: "Translate file from a language to another".  
   - Connect MCP Trigger `ai_tool` output to this node input.

9. **Add Sticky Note - Frontend Section**  
   - Add a Sticky Note node with color green (#3).  
   - Label it "Frontend" as a section header.

10. **Create HTTP Request Tool - Retrieve Frontend Settings**  
    - Type: HTTP Request Tool.  
    - Method: GET (default).  
    - URL: `http://libretranslate.local/frontend/settings`.  
    - Description: "Retrieve frontend specific settings".  
    - Connect MCP Trigger `ai_tool` output to this node input.

11. **Add Sticky Note - Feedback Section**  
    - Add a Sticky Note node with color blue (#4).  
    - Label it "Feedback" as a section header.

12. **Create HTTP Request Tool - Submit Translation Suggestion**  
    - Type: HTTP Request Tool.  
    - Method: POST.  
    - URL: `http://libretranslate.local/suggest`.  
    - Description: "Submit a suggestion to improve a translation".  
    - Parameters dynamically injected.  
    - Connect MCP Trigger `ai_tool` output to this node input.

13. **Finalize Connections**  
    - Ensure the MCP Trigger node output labeled `ai_tool` is connected as input to each HTTP Request Tool node corresponding to the six API operations.

14. **Activate Workflow**  
    - Save and activate the workflow.  
    - Copy the webhook URL from the MCP Trigger node (e.g., `https://your-n8n-instance/webhook/libretranslate-mcp`).  
    - Use this URL in AI agent configurations to send MCP requests.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| The workflow requires no authentication, simplifying integration with AI agents.                               | Setup Instructions Sticky Note                                                                               |
| For guidance or custom automation help, join the Discord community at https://discord.me/cfomodz               | Setup Instructions Sticky Note                                                                               |
| Official n8n documentation for Langchain MCP Trigger: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Setup Instructions Sticky Note                                                                               |
| Parameters for HTTP Request nodes are automatically populated by AI using `$fromAI()` expressions.             | Workflow Overview Sticky Note                                                                                 |
| The MCP Trigger node requires n8n version supporting Langchain MCP Trigger nodes (usually n8n v1.95+).         | MCP Trigger Node details                                                                                      |
| The LibreTranslate API is expected to be reachable at `http://libretranslate.local` within the same network.    | All HTTP Request Tool nodes                                                                                   |
| Consider adding logging or error handling nodes for production use.                                            | Setup Instructions Sticky Note                                                                               |

---

**Disclaimer:**  
The provided content is generated exclusively from an n8n workflow automation and complies with applicable content policies. It contains no illegal or offensive material. All data handled is public and legal.