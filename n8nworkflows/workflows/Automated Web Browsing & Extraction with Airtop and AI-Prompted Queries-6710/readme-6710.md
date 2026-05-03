Automated Web Browsing & Extraction with Airtop and AI-Prompted Queries

https://n8nworkflows.xyz/workflows/automated-web-browsing---extraction-with-airtop-and-ai-prompted-queries-6710


# Automated Web Browsing & Extraction with Airtop and AI-Prompted Queries

### 1. Workflow Overview

This workflow automates web browsing and data extraction using Airtop’s browser automation capabilities, triggered by an AI-prompted interface via LangChain’s MCP Server Trigger. Its primary use case is to perform complex browsing tasks such as loading pages, interacting with page elements, filling forms, scrolling, extracting data (including pagination support), waiting for downloads, taking screenshots, and managing browser sessions/windows, all orchestrated by AI-generated dynamic input parameters.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and AI Integration:** The entry point accepting AI-prompted commands.
- **1.2 Airtop Session Management:** Creating and terminating browsing sessions.
- **1.3 Browser Window Management:** Opening, closing, and loading pages in browser windows.
- **1.4 Browser Interaction:** Actions like clicking, scrolling, filling forms.
- **1.5 Data Extraction:** Querying page data with or without pagination.
- **1.6 Auxiliary Operations:** Waiting for downloads, taking screenshots.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and AI Integration

- **Overview:**  
  This block is the workflow’s trigger, receiving AI-driven prompts and parameters from LangChain’s MCP server, which then routes commands to the appropriate Airtop nodes.

- **Nodes Involved:**  
  - MCP Server Trigger

- **Node Details:**

  - **MCP Server Trigger**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger` (Trigger)  
    - Role: Listens for incoming webhook requests on path `/airtop` containing AI prompts and parameters.  
    - Configuration: Webhook path set to `"airtop"`.  
    - Inputs: None (trigger node).  
    - Outputs: Connected as `ai_tool` output to all Airtop operation nodes.  
    - Version: 2  
    - Edge Cases: Webhook authentication/authorization failures; malformed input data; network timeouts.  
    - Notes: This node centralizes AI input and dynamically routes parameters to downstream nodes.

#### 2.2 Airtop Session Management

- **Overview:**  
  Handles lifecycle of Airtop sessions, including creating and terminating sessions, which are required for all browser interactions.

- **Nodes Involved:**  
  - Create a session in Airtop  
  - Terminate a session in Airtop

- **Node Details:**

  - **Create a session in Airtop**  
    - Type: `n8n-nodes-base.airtopTool` (Session Management)  
    - Role: Initiates a new Airtop browsing session with configurable profile and timeout.  
    - Configuration:  
      - `profileName` set dynamically via AI input parameter `Profile_Name`.  
      - `timeoutMinutes` fixed to 10 minutes.  
      - `saveProfileOnTermination` optionally set by AI input `Save_Profile`.  
    - Inputs: Receives AI parameters from MCP Server Trigger.  
    - Outputs: Session ID passed downstream for further operations.  
    - Credentials: Uses configured Airtop API credentials.  
    - Version: 1  
    - Edge Cases: API authentication errors, session creation timeouts, invalid profile names.

  - **Terminate a session in Airtop**  
    - Type: `n8n-nodes-base.airtopTool` (Session Management)  
    - Role: Ends an existing Airtop session by ID.  
    - Configuration: Uses AI input parameter `Session_ID` to identify session.  
    - Inputs: AI parameters from MCP Server Trigger.  
    - Outputs: Confirmation of session termination.  
    - Credentials: Airtop API credentials.  
    - Version: 1  
    - Edge Cases: Invalid session ID, session already terminated, API errors.

#### 2.3 Browser Window Management

- **Overview:**  
  Manages browser windows within Airtop sessions, including creating new windows, loading URLs, closing windows, and taking screenshots.

- **Nodes Involved:**  
  - Create a window in Airtop  
  - Load a page in Airtop  
  - Close a window in Airtop  
  - Take screenshot in Airtop

- **Node Details:**

  - **Create a window in Airtop**  
    - Type: `n8n-nodes-base.airtopTool` (Window Management)  
    - Role: Opens a new browser window within a session, optionally navigates to a URL, and can enable live view.  
    - Configuration:  
      - `url` configured dynamically via AI input `URL`.  
      - `getLiveView` boolean flag from AI input `Get_Live_View`.  
      - `sessionId` from AI input.  
    - Inputs: AI parameters.  
    - Outputs: Window ID for subsequent interactions.  
    - Credentials: Airtop API credentials.  
    - Version: 1  
    - Edge Cases: Invalid URL, session expired, API rate limits.

  - **Load a page in Airtop**  
    - Type: `n8n-nodes-base.airtopTool`  
    - Role: Loads or reloads a specific URL in an existing window.  
    - Configuration:  
      - Uses AI inputs: `URL`, `Window_ID`, `Session_ID`.  
    - Inputs: AI parameters.  
    - Outputs: Page load status.  
    - Credentials: Airtop API credentials.  
    - Version: 1  
    - Edge Cases: Page load failures, invalid window ID, session timeout.

  - **Close a window in Airtop**  
    - Type: `n8n-nodes-base.airtopTool`  
    - Role: Closes a browser window identified by `Window_ID` within a session.  
    - Configuration: Uses AI inputs `Window_ID` and `Session_ID`.  
    - Inputs: AI parameters.  
    - Outputs: Confirmation of closure.  
    - Credentials: Airtop API credentials.  
    - Version: 1  
    - Edge Cases: Window already closed, invalid window ID.

  - **Take screenshot in Airtop**  
    - Type: `n8n-nodes-base.airtopTool`  
    - Role: Captures screenshot of the given window, optionally returning binary image data.  
    - Configuration:  
      - Uses AI inputs: `Window_ID`, `Session_ID`, and `Output_Binary_Image` boolean flag.  
    - Inputs: AI parameters.  
    - Outputs: Screenshot data (binary or otherwise).  
    - Credentials: Airtop API credentials.  
    - Version: 1  
    - Edge Cases: Large image size causing timeout, window not found.

#### 2.4 Browser Interaction

- **Overview:**  
  Performs interactions on the loaded web page such as clicking elements, scrolling, and filling out forms.

- **Nodes Involved:**  
  - Click an element in Airtop  
  - Scroll on page in Airtop  
  - Fill form in Airtop

- **Node Details:**

  - **Click an element in Airtop**  
    - Type: `n8n-nodes-base.airtopTool` (Interaction)  
    - Role: Simulates a click on a specified element within the window.  
    - Configuration:  
      - Uses AI inputs: `Window_ID`, `Session_ID`, and `Element_Description` (selector or description).  
    - Inputs: AI parameters.  
    - Outputs: Interaction result confirmation.  
    - Credentials: Airtop API credentials.  
    - Version: 1  
    - Edge Cases: Element not found, stale element reference, timing issues.

  - **Scroll on page in Airtop**  
    - Type: `n8n-nodes-base.airtopTool` (Interaction)  
    - Role: Scrolls the page or a scrollable region to a specific element or position.  
    - Configuration:  
      - AI inputs: `Window_ID`, `Session_ID`, `Scrollable_Area` (container selector), `Element_Description` (target element).  
    - Inputs: AI parameters.  
    - Outputs: Scroll completion confirmation.  
    - Credentials: Airtop API credentials.  
    - Version: 1  
    - Edge Cases: Non-scrollable area, element outside viewport, slow rendering.

  - **Fill form in Airtop**  
    - Type: `n8n-nodes-base.airtopTool` (Interaction)  
    - Role: Fills form fields with provided data on the page.  
    - Configuration:  
      - AI inputs: `Window_ID`, `Session_ID`, `Form_Data` (JSON or string describing form inputs).  
    - Inputs: AI parameters.  
    - Outputs: Form fill confirmation.  
    - Credentials: Airtop API credentials.  
    - Version: 1  
    - Edge Cases: Form fields missing, validation errors, dynamic forms.

#### 2.5 Data Extraction

- **Overview:**  
  Extracts data from web pages using AI-prompted queries, supports both single page and paginated extraction.

- **Nodes Involved:**  
  - Query page in Airtop  
  - Query page with pagination in Airtop

- **Node Details:**

  - **Query page in Airtop**  
    - Type: `n8n-nodes-base.airtopTool` (Extraction)  
    - Role: Sends an AI prompt to extract structured data from the current page.  
    - Configuration:  
      - AI inputs: `Prompt` (query), `Window_ID`, `Session_ID`.  
    - Inputs: AI parameters.  
    - Outputs: Extracted data as structured output.  
    - Credentials: Airtop API credentials.  
    - Version: 1  
    - Edge Cases: Ambiguous or incomplete prompt, extraction failures.

  - **Query page with pagination in Airtop**  
    - Type: `n8n-nodes-base.airtopTool` (Extraction)  
    - Role: Similar to above but supports automatic pagination handling to extract data across multiple pages.  
    - Configuration: Same AI inputs as above.  
    - Inputs: AI parameters.  
    - Outputs: Aggregated data from multiple pages.  
    - Credentials: Airtop API credentials.  
    - Version: 1  
    - Edge Cases: Pagination structure changes, infinite loops, inconsistent data.

#### 2.6 Auxiliary Operations

- **Overview:**  
  Handles waiting for file downloads, an operation often required after form submission or clicking download links.

- **Nodes Involved:**  
  - Wait for a download in Airtop

- **Node Details:**

  - **Wait for a download in Airtop**  
    - Type: `n8n-nodes-base.airtopTool` (Download Management)  
    - Role: Pauses workflow until a download triggered by the session completes.  
    - Configuration: Uses AI input `Session_ID`.  
    - Inputs: AI parameters.  
    - Outputs: Download confirmation or file metadata.  
    - Credentials: Airtop API credentials.  
    - Version: 1  
    - Edge Cases: Download timeout, no download triggered, network errors.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role               | Input Node(s)      | Output Node(s)     | Sticky Note                      |
|-------------------------------|----------------------------------|------------------------------|--------------------|--------------------|---------------------------------|
| MCP Server Trigger             | @n8n/n8n-nodes-langchain.mcpTrigger | AI Prompt Reception & Routing | None               | All Airtop nodes   | Central AI input trigger         |
| Create a session in Airtop     | n8n-nodes-base.airtopTool        | Start Airtop browsing session | MCP Server Trigger | Others (session users) | Uses profileName and timeout; credentials required |
| Terminate a session in Airtop  | n8n-nodes-base.airtopTool        | End Airtop browsing session   | MCP Server Trigger | None               | Requires session ID             |
| Create a window in Airtop      | n8n-nodes-base.airtopTool        | Open new browser window       | MCP Server Trigger | Load page, interactions | Supports live view option       |
| Load a page in Airtop          | n8n-nodes-base.airtopTool        | Load URL in a window          | MCP Server Trigger | Interactions, extraction | Uses session & window IDs       |
| Close a window in Airtop       | n8n-nodes-base.airtopTool        | Close browser window          | MCP Server Trigger | None               | Requires window ID              |
| Take screenshot in Airtop      | n8n-nodes-base.airtopTool        | Capture screenshot            | MCP Server Trigger | None               | Optionally outputs binary image |
| Click an element in Airtop     | n8n-nodes-base.airtopTool        | Click page element            | MCP Server Trigger | None               | Uses element description        |
| Scroll on page in Airtop       | n8n-nodes-base.airtopTool        | Scroll page or scrollable area | MCP Server Trigger | None               | Scroll target configurable      |
| Fill form in Airtop            | n8n-nodes-base.airtopTool        | Fill form fields              | MCP Server Trigger | None               | Uses JSON form data             |
| Query page in Airtop           | n8n-nodes-base.airtopTool        | Extract data from page        | MCP Server Trigger | None               | AI prompt driven extraction     |
| Query page with pagination in Airtop | n8n-nodes-base.airtopTool  | Extract paginated data        | MCP Server Trigger | None               | Handles multi-page queries      |
| Wait for a download in Airtop  | n8n-nodes-base.airtopTool        | Await file download completion | MCP Server Trigger | None               | Download monitoring             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger node**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters: Set webhook path to `"airtop"`  
   - This node will receive AI prompts and parameters as input.

2. **Create “Create a session in Airtop” node**  
   - Type: `n8n-nodes-base.airtopTool`  
   - Operation: Default (create session)  
   - Parameters:  
     - `profileName`: Use expression to get from AI input parameter `Profile_Name`  
     - `timeoutMinutes`: Set to 10  
     - `saveProfileOnTermination`: Boolean from AI input `Save_Profile`  
   - Credentials: Assign Airtop API credentials  
   - Connect output of MCP Server Trigger to this node’s `ai_tool` input.

3. **Create “Terminate a session in Airtop” node**  
   - Type: `n8n-nodes-base.airtopTool`  
   - Operation: `terminate`  
   - Parameters:  
     - `sessionId`: Get from AI input `Session_ID`  
   - Credentials: Airtop API  
   - Connect MCP Server Trigger `ai_tool` output to this node.

4. **Create “Create a window in Airtop” node**  
   - Type: `n8n-nodes-base.airtopTool`  
   - Operation: `create` window (default)  
   - Parameters:  
     - `url`: AI input `URL`  
     - `sessionId`: AI input `Session_ID`  
     - `getLiveView`: Boolean from AI input `Get_Live_View`  
   - Credentials: Airtop API  
   - Connect MCP Server Trigger `ai_tool` output.

5. **Create “Load a page in Airtop” node**  
   - Type: `n8n-nodes-base.airtopTool`  
   - Operation: `load` (window resource)  
   - Parameters:  
     - `url`: AI input `URL`  
     - `windowId`: AI input `Window_ID`  
     - `sessionId`: AI input `Session_ID`  
   - Credentials: Airtop API  
   - Connect MCP Server Trigger `ai_tool` output.

6. **Create “Close a window in Airtop” node**  
   - Type: `n8n-nodes-base.airtopTool`  
   - Operation: `close` (window resource)  
   - Parameters:  
     - `windowId`: AI input `Window_ID`  
     - `sessionId`: AI input `Session_ID`  
   - Credentials: Airtop API  
   - Connect MCP Server Trigger `ai_tool` output.

7. **Create “Take screenshot in Airtop” node**  
   - Type: `n8n-nodes-base.airtopTool`  
   - Operation: `takeScreenshot` (window resource)  
   - Parameters:  
     - `windowId`: AI input `Window_ID`  
     - `sessionId`: AI input `Session_ID`  
     - `outputImageAsBinary`: Boolean AI input `Output_Binary_Image`  
   - Credentials: Airtop API  
   - Connect MCP Server Trigger `ai_tool` output.

8. **Create “Click an element in Airtop” node**  
   - Type: `n8n-nodes-base.airtopTool`  
   - Operation: `interaction` > Click  
   - Parameters:  
     - `windowId`: AI input `Window_ID`  
     - `sessionId`: AI input `Session_ID`  
     - `elementDescription`: AI input `Element_Description`  
   - Credentials: Airtop API  
   - Connect MCP Server Trigger `ai_tool` output.

9. **Create “Scroll on page in Airtop” node**  
   - Type: `n8n-nodes-base.airtopTool`  
   - Operation: `scroll` under `interaction` resource  
   - Parameters:  
     - `windowId`: AI input `Window_ID`  
     - `sessionId`: AI input `Session_ID`  
     - `scrollWithin`: AI input `Scrollable_Area`  
     - `scrollToElement`: AI input `Element_Description`  
   - Credentials: Airtop API  
   - Connect MCP Server Trigger `ai_tool` output.

10. **Create “Fill form in Airtop” node**  
    - Type: `n8n-nodes-base.airtopTool`  
    - Operation: `fill` under `interaction` resource  
    - Parameters:  
      - `windowId`: AI input `Window_ID`  
      - `sessionId`: AI input `Session_ID`  
      - `formData`: AI input `Form_Data` (JSON/string describing form fields and values)  
    - Credentials: Airtop API  
    - Connect MCP Server Trigger `ai_tool` output.

11. **Create “Query page in Airtop” node**  
    - Type: `n8n-nodes-base.airtopTool`  
    - Operation: `query` under `extraction` resource  
    - Parameters:  
      - `windowId`: AI input `Window_ID`  
      - `sessionId`: AI input `Session_ID`  
      - `prompt`: AI input `Prompt` (query for data extraction)  
    - Credentials: Airtop API  
    - Connect MCP Server Trigger `ai_tool` output.

12. **Create “Query page with pagination in Airtop” node**  
    - Type: `n8n-nodes-base.airtopTool`  
    - Operation: `query` under `extraction` resource with pagination support  
    - Parameters:  
      - `windowId`: AI input `Window_ID`  
      - `sessionId`: AI input `Session_ID`  
      - `prompt`: AI input `Prompt`  
    - Credentials: Airtop API  
    - Connect MCP Server Trigger `ai_tool` output.

13. **Create “Wait for a download in Airtop” node**  
    - Type: `n8n-nodes-base.airtopTool`  
    - Operation: `waitForDownload`  
    - Parameters:  
      - `sessionId`: AI input `Session_ID`  
    - Credentials: Airtop API  
    - Connect MCP Server Trigger `ai_tool` output.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                 |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| Workflow designed to be fully driven by AI input parameters via MCP Server Trigger for dynamic web automation.       | n8n + Airtop + LangChain integration           |
| Airtop API credentials must be configured in n8n credentials with valid permissions.                                  | n8n credential manager                          |
| AI-driven parameters use the `$fromAI()` expression for dynamic parameter assignment, enabling flexible operations.  | n8n expression syntax documentation             |
| Airtop nodes require stable API connectivity; network issues may affect session management and interactions.         | Airtop API docs (not public, internal use)      |
| Consider adding error handling (e.g., catch nodes) for production resilience in case of session failures or timeouts.| Recommended best practice for robust workflows |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.