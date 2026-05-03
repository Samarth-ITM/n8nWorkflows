🛠️ Airtop Tool MCP Server 💪 all 20 operations

https://n8nworkflows.xyz/workflows/----airtop-tool-mcp-server----all-20-operations-5334


# 🛠️ Airtop Tool MCP Server 💪 all 20 operations

### 1. Workflow Overview

The **Airtop Tool MCP Server** workflow is designed to provide a comprehensive server-side automation interface for controlling Airtop Tool’s multi-capability platform (MCP). It exposes 20 distinct operations related to web scraping, file management, session and window control, and web interaction tasks through a single MCP trigger entry point.

**Target Use Cases:**  
- Automating complex browser and file operations programmatically via webhooks  
- Managing sessions and windows for web automation tasks  
- Performing advanced page queries, scraping, and interaction with page elements  
- Handling file operations including upload, download, deletion, and loading from Airtop’s environment  

**Logical Blocks:**  
- **1.1 Trigger and Input Reception:** Entry point waiting for external calls to initiate any of the 20 operations  
- **1.2 Page Query and Scraping Operations:** Nodes handling page querying, smart scraping, and pagination  
- **1.3 File Management Operations:** Nodes managing file upload, download, deletion, and batch retrieval  
- **1.4 Web Interaction Operations:** Nodes for clicking, filling forms, typing, scrolling, hovering on elements  
- **1.5 Session Lifecycle Management:** Nodes controlling session creation, termination, and profile saving  
- **1.6 Window Lifecycle Management:** Nodes for creating, loading pages into, taking screenshots from, and closing browser windows  
- **1.7 Sticky Notes:** Non-functional nodes used for visual grouping and notes in the editor  

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Input Reception

**Overview:**  
This block acts as the sole entry point for the workflow, receiving external requests specifying which Airtop Tool MCP operation to perform.

**Nodes Involved:**  
- Airtop Tool MCP Server (MCP Trigger)

**Node Details:**  
- **Name:** Airtop Tool MCP Server  
- **Type:** MCP Trigger (LangChain n8n node)  
- **Role:** Listens for incoming webhook requests to trigger the workflow dynamically for any of the 20 supported operations  
- **Configuration:** Default MCP Trigger setup with a unique webhook ID  
- **Input:** External HTTP request (webhook)  
- **Output:** Triggers downstream Airtop Tool operation nodes based on input parameters  
- **Edge Cases:**  
  - Invalid or missing webhook payload could lead to no operation being triggered  
  - Authentication or permission issues if webhook security is enabled outside the workflow  
  - Timeout if the trigger is not invoked within expected time  
- **Sub-workflow:** None  

---

#### 1.2 Page Query and Scraping Operations

**Overview:**  
Handles reading and scraping web pages with options for pagination and smart extraction.

**Nodes Involved:**  
- Query page  
- Query page with pagination  
- Smart scrape page  

**Node Details:**  

- **Query page**  
  - **Type:** Airtop Tool node  
  - **Role:** Executes a basic page query to retrieve content or data from a given URL or page context  
  - **Configuration:** Parameters to specify URL/query selectors (not shown in JSON)  
  - **Input:** Triggered by MCP Server node  
  - **Output:** Passes page data downstream if needed  
  - **Edge Cases:** Page load failures, invalid selectors, timeout on slow pages  

- **Query page with pagination**  
  - Similar to Query page, but supports iterating over paginated content  
  - Additional parameters for pagination controls (next page selectors, max pages)  
  - Edge cases include infinite pagination loops or missing next page links  

- **Smart scrape page**  
  - Advanced scraping using heuristics or AI-based extraction  
  - Flexible selector input or data pattern recognition  
  - Edge cases include unexpected page layout changes or anti-bot protections  

---

#### 1.3 File Management Operations

**Overview:**  
Manages file lifecycle within Airtop environment including listing, uploading, downloading, and deletion.

**Nodes Involved:**  
- Delete a file  
- Get a file  
- Get many files  
- Load a file  
- Upload a file  

**Node Details:**  

- **Delete a file**  
  - Deletes a specified file from Airtop storage  
  - Requires file identifier/path  
  - Edge cases: file not found, permission denied  

- **Get a file**  
  - Downloads or retrieves a single file’s content or metadata  
  - Edge cases: file missing, large file timeout  

- **Get many files**  
  - Retrieves multiple files based on criteria or folder  
  - Edge cases: high volume leading to performance issues  

- **Load a file**  
  - Loads a file into the environment, possibly for use in subsequent operations  
  - Edge cases: unsupported file type, corrupted file  

- **Upload a file**  
  - Uploads a local or external file into Airtop’s storage  
  - Edge cases: upload failures, file size limits  

---

#### 1.4 Web Interaction Operations

**Overview:**  
Simulates user actions on web pages such as clicking, typing, scrolling, and form filling.

**Nodes Involved:**  
- Click an element  
- Fill form  
- Hover on an element  
- Scroll on page  
- Type text  

**Node Details:**  

- **Click an element**  
  - Simulates mouse click on a page element identified by selector  
  - Edge cases: element not found, element not clickable  

- **Fill form**  
  - Inputs data into form fields  
  - Edge cases: field validation errors, missing fields  

- **Hover on an element**  
  - Moves mouse pointer over an element to trigger hover states  
  - Edge cases: element obscured or off-screen  

- **Scroll on page**  
  - Scrolls viewport by specified amount or to element  
  - Edge cases: scroll limits reached, dynamic content loading issues  

- **Type text**  
  - Types text input into focused element or specified field  
  - Edge cases: input blocked, typing speed mismatches  

---

#### 1.5 Session Lifecycle Management

**Overview:**  
Controls broader session state including creation, termination, and profile saving.

**Nodes Involved:**  
- Create a session  
- Save a profile on session termination  
- Terminate a session  

**Node Details:**  

- **Create a session**  
  - Initializes a new session context for browsing/automation  
  - Edge cases: session creation failures due to limits or resource constraints  

- **Save a profile on session termination**  
  - Persists session state or profile data when session ends  
  - Edge cases: data corruption, failed write operations  

- **Terminate a session**  
  - Gracefully ends session and cleans resources  
  - Edge cases: session already terminated, partial cleanup failures  

---

#### 1.6 Window Lifecycle Management

**Overview:**  
Manages browser window lifecycle and visual capture.

**Nodes Involved:**  
- Create a window  
- Load a page  
- Take screenshot  
- Close a window  

**Node Details:**  

- **Create a window**  
  - Opens new browser window or tab within the session  
  - Edge cases: resource limits, popup blockers  

- **Load a page**  
  - Loads a specified URL in the created window  
  - Edge cases: navigation timeouts, invalid URLs  

- **Take screenshot**  
  - Captures visible viewport or full page screenshot  
  - Edge cases: rendering delays, permission issues  

- **Close a window**  
  - Closes the window/tab cleanly  
  - Edge cases: window already closed, errors on close  

---

#### 1.7 Sticky Notes

**Overview:**  
Visual notes in the workflow editor for grouping or reminders; no execution role.

**Nodes Involved:**  
- Workflow Overview 0  
- Sticky Note 1  
- Sticky Note 2  
- Sticky Note 3  
- Sticky Note 4  
- Sticky Note 5  

**Node Details:**  
- All are n8n stickyNote nodes with empty content. They serve to visually separate and organize the node groups above. No runtime impact.

---

### 3. Summary Table

| Node Name                        | Node Type                     | Functional Role                  | Input Node(s)              | Output Node(s)               | Sticky Note         |
|---------------------------------|-------------------------------|--------------------------------|----------------------------|-----------------------------|---------------------|
| Workflow Overview 0             | stickyNote                    | Visual grouping                 |                            |                             |                     |
| Airtop Tool MCP Server          | MCP Trigger                  | Main trigger for all operations|                            | All Airtop Tool nodes        |                     |
| Query page                     | Airtop Tool                  | Basic page query                | Airtop Tool MCP Server      |                             |                     |
| Query page with pagination     | Airtop Tool                  | Page query with pagination      | Airtop Tool MCP Server      |                             |                     |
| Smart scrape page              | Airtop Tool                  | Advanced scraping               | Airtop Tool MCP Server      |                             |                     |
| Sticky Note 1                 | stickyNote                    | Visual grouping                 |                            |                             |                     |
| Delete a file                 | Airtop Tool                  | Deletes a file                  | Airtop Tool MCP Server      |                             |                     |
| Get a file                   | Airtop Tool                  | Retrieves a single file         | Airtop Tool MCP Server      |                             |                     |
| Get many files              | Airtop Tool                  | Retrieves multiple files        | Airtop Tool MCP Server      |                             |                     |
| Load a file                 | Airtop Tool                  | Loads a file into environment   | Airtop Tool MCP Server      |                             |                     |
| Upload a file               | Airtop Tool                  | Uploads a file                  | Airtop Tool MCP Server      |                             |                     |
| Sticky Note 2               | stickyNote                    | Visual grouping                 |                            |                             |                     |
| Click an element            | Airtop Tool                  | Simulates click on element      | Airtop Tool MCP Server      |                             |                     |
| Fill form                  | Airtop Tool                  | Fills out form fields           | Airtop Tool MCP Server      |                             |                     |
| Hover on an element        | Airtop Tool                  | Simulates hover over element    | Airtop Tool MCP Server      |                             |                     |
| Scroll on page             | Airtop Tool                  | Scrolls page                   | Airtop Tool MCP Server      |                             |                     |
| Type text                | Airtop Tool                  | Types text into element         | Airtop Tool MCP Server      |                             |                     |
| Sticky Note 3             | stickyNote                    | Visual grouping                 |                            |                             |                     |
| Create a session          | Airtop Tool                  | Creates session context         | Airtop Tool MCP Server      |                             |                     |
| Save a profile on session termination | Airtop Tool          | Saves session profile on end    | Airtop Tool MCP Server      |                             |                     |
| Terminate a session       | Airtop Tool                  | Ends session                   | Airtop Tool MCP Server      |                             |                     |
| Sticky Note 4             | stickyNote                    | Visual grouping                 |                            |                             |                     |
| Create a window           | Airtop Tool                  | Opens new window/tab            | Airtop Tool MCP Server      |                             |                     |
| Load a page              | Airtop Tool                  | Loads URL in window             | Airtop Tool MCP Server      |                             |                     |
| Take screenshot          | Airtop Tool                  | Captures screenshot             | Airtop Tool MCP Server      |                             |                     |
| Close a window           | Airtop Tool                  | Closes browser window           | Airtop Tool MCP Server      |                             |                     |
| Sticky Note 5             | stickyNote                    | Visual grouping                 |                            |                             |                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Trigger Node:**  
   - Add node of type `MCP Trigger` (LangChain node)  
   - Set webhookId to unique value or leave default for auto-generation  
   - This node serves as the sole entry point to trigger all Airtop Tool operations  

2. **Add Airtop Tool Nodes for Query & Scraping:**  
   - Add `Query page` node, connect input from MCP Trigger via ai_tool output  
   - Add `Query page with pagination` node, connect from MCP Trigger  
   - Add `Smart scrape page` node, connect from MCP Trigger  
   - Configure each with necessary parameters (URLs, selectors, pagination controls), credentials if required  

3. **Add Airtop Tool Nodes for File Management:**  
   - Add nodes `Delete a file`, `Get a file`, `Get many files`, `Load a file`, `Upload a file`  
   - Connect each from MCP Trigger ai_tool output  
   - Configure file identifiers, paths, upload sources as needed  

4. **Add Airtop Tool Nodes for Web Interaction:**  
   - Add nodes `Click an element`, `Fill form`, `Hover on an element`, `Scroll on page`, `Type text`  
   - Connect each from MCP Trigger ai_tool output  
   - Configure selectors, form data, text input parameters accordingly  

5. **Add Airtop Tool Nodes for Session Management:**  
   - Add nodes `Create a session`, `Save a profile on session termination`, `Terminate a session`  
   - Connect each from MCP Trigger ai_tool output  
   - Configure session parameters if any  

6. **Add Airtop Tool Nodes for Window Management:**  
   - Add nodes `Create a window`, `Load a page`, `Take screenshot`, `Close a window`  
   - Connect each from MCP Trigger ai_tool output  
   - Configure URLs, screenshot options as needed  

7. **Add Sticky Note Nodes for Visual Organization:**  
   - Insert sticky notes to separate logical groups as desired (optional)  
   - No configuration needed  

8. **Credential Configuration:**  
   - Ensure Airtop Tool credentials are set up in n8n for all Airtop Tool nodes to function  
   - MCP Trigger webhook security and authentication as required  

9. **Connection Setup:**  
   - Each Airtop Tool node’s `ai_tool` input connects directly from `Airtop Tool MCP Server` node’s `ai_tool` output  
   - This allows the MCP Trigger to dynamically dispatch the correct node based on incoming request  

10. **Save and Activate Workflow:**  
    - Test each operation individually by invoking the webhook with appropriate payloads specifying operation name and parameters  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                         |
|-------------------------------------------------------------------------------------------------|---------------------------------------|
| The workflow integrates all current 20 Airtop Tool operations for centralized server control.   | Workflow Title                         |
| Sticky notes are used extensively for visual clarity but contain no runtime logic.               | Workflow Visual Organization          |
| MCP Trigger node enables dynamic dispatching of operations via a single webhook interface.      | n8n LangChain MCP Trigger Documentation |
| Airtop Tool nodes require proper credentials setup within n8n to connect to the Airtop platform. | Airtop Tool Credential Setup Guide    |

---

**Disclaimer:** The text provided derives exclusively from an automated workflow built with n8n, a tool for integration and automation. This process strictly respects current content policies and contains no illegal, offensive, or protected material. All data manipulated is legal and public.