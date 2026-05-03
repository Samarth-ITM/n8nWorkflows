Build your own FileSystem MCP server

https://n8nworkflows.xyz/workflows/build-your-own-filesystem-mcp-server-3630


# Build your own FileSystem MCP server

### 1. Workflow Overview

This workflow implements a simple FileSystem MCP (Model Context Protocol) server designed for self-hosted n8n instances running on Linux. Its primary purpose is to allow MCP clients and agents to remotely interact with the server’s filesystem by listing directories, searching files, creating directories, and reading/writing file contents securely without exposing arbitrary command execution.

The workflow is logically divided into the following blocks:

- **1.1 MCP Server Trigger Setup:** Listens for incoming MCP client requests and routes them to appropriate tools.
- **1.2 Directory Operations via Execute Command Tools:** Handles listing directories, searching files, and creating directories using controlled shell commands.
- **1.3 File Read/Write Operations via Custom Workflow Tools:** Manages reading from and writing to files through dedicated sub-workflows, ensuring safe parameter handling.
- **1.4 Internal Operation Routing:** Uses a Switch node to route file operations (read/write) triggered by sub-workflows.
- **1.5 Execution of File Commands:** Executes shell commands for reading and writing files based on routed operations.

Special emphasis is placed on security by restricting command inputs to parameters like filenames and paths, preventing arbitrary command execution.

---

### 2. Block-by-Block Analysis

#### 2.1 MCP Server Trigger Setup

- **Overview:**  
  This block initializes the MCP server trigger node that listens for incoming MCP client requests and connects them to the various filesystem operation tools.

- **Nodes Involved:**  
  - FileSystem MCP Server (MCP Trigger)  
  - Sticky Note (Documentation)  
  - Sticky Note3 (Authentication Reminder)

- **Node Details:**

  - **FileSystem MCP Server**  
    - Type: MCP Trigger (`@n8n/n8n-nodes-langchain.mcpTrigger`)  
    - Role: Entry point for MCP client requests, exposing an HTTP webhook endpoint.  
    - Configuration:  
      - Webhook path set to a unique ID (`0d93cfd5-2fbf-457e-9535-5bfc9a73ba9e`).  
    - Inputs: None (trigger node)  
    - Outputs: Connects to all filesystem operation tools (Execute Command and Custom Workflow Tools).  
    - Edge Cases:  
      - Requires authentication before production use to prevent unauthorized access (highlighted in Sticky Note3).  
      - Potential webhook connectivity or permission issues on server.  
    - Reference: [MCP Server Trigger Documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger)

  - **Sticky Note**  
    - Type: Sticky Note  
    - Content: Provides a link to MCP Server Trigger documentation.

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Content: Advises enabling authentication on the MCP server trigger before production.

---

#### 2.2 Directory Operations via Execute Command Tools

- **Overview:**  
  This block manages directory-related operations (listing, searching, creating) using Execute Command Tool nodes that run controlled Linux shell commands scoped to the `/home/node/` project root directory.

- **Nodes Involved:**  
  - ListDirectory (Execute Command Tool)  
  - CreateDirectory (Execute Command Tool)  
  - SearchDirectory (Execute Command Tool)  
  - Sticky Note1 (Documentation)

- **Node Details:**

  - **ListDirectory**  
    - Type: Execute Command Tool (`n8n-nodes-base.executeCommandTool`)  
    - Role: Lists directories under `/home/node/` optionally filtered by a path parameter from the MCP client.  
    - Configuration:  
      - Command template: `ls /home/node/{{ $fromAI('path', 'optional, leave blank for project root directory.') }}`  
      - Tool description clarifies the project root directory context.  
    - Inputs: Parameters from MCP client via the MCP trigger node.  
    - Outputs: Directory listing results back to MCP client.  
    - Edge Cases:  
      - Empty or invalid path parameters default to root directory.  
      - Potential permission errors if directories are inaccessible.

  - **CreateDirectory**  
    - Type: Execute Command Tool  
    - Role: Creates directories under `/home/node/` based on filename/path parameters from MCP client.  
    - Configuration:  
      - Command template: `mkdir -p /home/node/{{ $fromAI('filename', 'name of directory. Will be scoped under the /home/node/ project root directory. Optionally use path to create within subdirectories') }}`  
      - Uses `mkdir -p` to create nested directories safely.  
    - Edge Cases:  
      - Invalid directory names or permission issues may cause failures.

  - **SearchDirectory**  
    - Type: Execute Command Tool  
    - Role: Searches for files by name within `/home/node/` using the Linux `find` command.  
    - Configuration:  
      - Command template: `find /home/node/ -name "{{ $fromAI('filename', 'A name search paramter for the linux find tool') }}"`  
    - Edge Cases:  
      - Wildcard or special characters in filename may affect search results.  
      - Large directory trees may cause timeouts.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Content: Provides a link to Execute Command tool documentation for further customization.

---

#### 2.3 File Read/Write Operations via Custom Workflow Tools

- **Overview:**  
  This block handles more complex file operations (reading and writing file contents) by invoking custom sub-workflows. This approach isolates file content handling from direct shell commands, improving security and flexibility.

- **Nodes Involved:**  
  - ReadFiles (Custom Workflow Tool)  
  - WriteFiles (Custom Workflow Tool)  
  - When Executed by Another Workflow (Execute Workflow Trigger)  
  - Operation (Switch)  
  - readOneOrMultipleFiles (Execute Command)  
  - writeOneOrMultipleFiles (Execute Command)

- **Node Details:**

  - **ReadFiles**  
    - Type: Custom Workflow Tool (`@n8n/n8n-nodes-langchain.toolWorkflow`)  
    - Role: Tool to read contents of one or multiple files by calling the internal workflow with parameters.  
    - Configuration:  
      - Points to the current workflow ID (self-referential).  
      - Inputs: `operation` set to `"readOneOrMultipleFiles"`, `filenames` array from MCP client, empty `contents` array.  
      - Description instructs to include file extensions in filenames.  
    - Outputs: File contents returned to MCP client.

  - **WriteFiles**  
    - Type: Custom Workflow Tool  
    - Role: Tool to write contents to one or multiple files by calling the internal workflow.  
    - Configuration:  
      - Points to current workflow ID.  
      - Inputs: `operation` set to `"writeOneOrMultipleFiles"`, `filenames` and `contents` arrays matched by index.  
      - Description explains matching filenames and contents by array index.  
    - Outputs: Confirmation of write operation.

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger (`n8n-nodes-base.executeWorkflowTrigger`)  
    - Role: Entry point for internal calls from the custom workflow tools.  
    - Configuration:  
      - Accepts inputs: `operation` (string), `filenames` (array), `contents` (array).  
    - Outputs: Routes to the Operation Switch node.

  - **Operation**  
    - Type: Switch (`n8n-nodes-base.switch`)  
    - Role: Routes workflow execution based on the `operation` input value.  
    - Configuration:  
      - Two outputs:  
        - `writeOneOrMultipleFiles` if `operation` equals `"writeOneOrMultipleFiles"`  
        - `readOneOrMultipleFiles` if `operation` equals `"readOneOrMultipleFiles"`  
    - Outputs: Connects to respective Execute Command nodes.

  - **readOneOrMultipleFiles**  
    - Type: Execute Command (`n8n-nodes-base.executeCommand`)  
    - Role: Reads contents of one or multiple files using `cat` command.  
    - Configuration:  
      - Command: `cat {{ $json.filenames.join(' ') }}` concatenates filenames with spaces.  
    - Edge Cases:  
      - Non-existent files cause errors.  
      - Large files may cause timeouts or memory issues.

  - **writeOneOrMultipleFiles**  
    - Type: Execute Command  
    - Role: Writes contents to files by echoing content into each file under `/home/node/`.  
    - Configuration:  
      - Dynamically generates multiple echo commands, one per filename/content pair:  
        ```bash
        echo "<content>" > /home/node/<filename>
        ```  
      - Uses JavaScript map to iterate over filenames and contents arrays.  
    - Edge Cases:  
      - Special characters in content or filenames may require escaping.  
      - Permission issues may prevent writing.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                              | Input Node(s)                | Output Node(s)                    | Sticky Note                                                                                      |
|----------------------------|----------------------------------|----------------------------------------------|------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------|
| FileSystem MCP Server       | MCP Trigger                      | Entry point for MCP client requests          | None                         | ListDirectory, CreateDirectory, SearchDirectory, ReadFiles, WriteFiles | See [MCP Server Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger) |
| ListDirectory              | Execute Command Tool             | List directories under project root          | FileSystem MCP Server        | FileSystem MCP Server            |                                                                                                 |
| CreateDirectory            | Execute Command Tool             | Create directories under project root        | FileSystem MCP Server        | FileSystem MCP Server            |                                                                                                 |
| SearchDirectory            | Execute Command Tool             | Search files by name under project root      | FileSystem MCP Server        | FileSystem MCP Server            |                                                                                                 |
| ReadFiles                  | Custom Workflow Tool             | Read file contents via internal workflow     | FileSystem MCP Server        | FileSystem MCP Server            |                                                                                                 |
| WriteFiles                 | Custom Workflow Tool             | Write file contents via internal workflow    | FileSystem MCP Server        | FileSystem MCP Server            |                                                                                                 |
| When Executed by Another Workflow | Execute Workflow Trigger       | Internal entry point for file operations      | None                         | Operation                       |                                                                                                 |
| Operation                  | Switch                          | Routes to read or write file commands        | When Executed by Another Workflow | readOneOrMultipleFiles, writeOneOrMultipleFiles |                                                                                                 |
| readOneOrMultipleFiles     | Execute Command                 | Reads contents of files using `cat`           | Operation                   | None                           |                                                                                                 |
| writeOneOrMultipleFiles    | Execute Command                 | Writes contents to files using `echo`         | Operation                   | None                           |                                                                                                 |
| Sticky Note                | Sticky Note                    | Documentation and links                        | None                         | None                           | "Set up an MCP Server Trigger" with link to MCP Server Trigger docs                             |
| Sticky Note1               | Sticky Note                    | Documentation for Execute Command tool        | None                         | None                           | Link to Execute Command tool docs                                                              |
| Sticky Note3               | Sticky Note                    | Security reminder to enable authentication    | None                         | None                           | "Always Authenticate Your Server!"                                                             |
| Sticky Note2               | Sticky Note                    | Full workflow description and usage notes     | None                         | None                           | Full detailed description and usage instructions                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create MCP Server Trigger Node**  
   - Type: MCP Trigger (`@n8n/n8n-nodes-langchain.mcpTrigger`)  
   - Set webhook path to a unique identifier (e.g., `0d93cfd5-2fbf-457e-9535-5bfc9a73ba9e`).  
   - Ensure authentication is enabled before production use.

2. **Create Execute Command Tool Nodes for Directory Operations**  
   - **ListDirectory:**  
     - Type: Execute Command Tool (`n8n-nodes-base.executeCommandTool`)  
     - Command: `ls /home/node/{{ $fromAI('path', 'optional, leave blank for project root directory.') }}`  
     - Description: Lists directories under `/home/node/`.  
   - **CreateDirectory:**  
     - Type: Execute Command Tool  
     - Command: `mkdir -p /home/node/{{ $fromAI('filename', 'name of directory. Scoped under /home/node/.') }}`  
     - Description: Creates directories under `/home/node/`.  
   - **SearchDirectory:**  
     - Type: Execute Command Tool  
     - Command: `find /home/node/ -name "{{ $fromAI('filename', 'A name search parameter for the linux find tool') }}"`  
     - Description: Searches files by name under `/home/node/`.

3. **Create Custom Workflow Tools for File Read/Write**  
   - **ReadFiles:**  
     - Type: Custom Workflow Tool (`@n8n/n8n-nodes-langchain.toolWorkflow`)  
     - Name: `readFil`  
     - Workflow ID: Set to current workflow ID (self-reference).  
     - Inputs:  
       - `operation`: `"readOneOrMultipleFiles"`  
       - `filenames`: Array of filenames from MCP client  
       - `contents`: Empty array  
     - Description: Reads contents of files.  
   - **WriteFiles:**  
     - Type: Custom Workflow Tool  
     - Name: `write_file`  
     - Workflow ID: Current workflow ID  
     - Inputs:  
       - `operation`: `"writeOneOrMultipleFiles"`  
       - `filenames`: Array of filenames  
       - `contents`: Array of contents matched by index  
     - Description: Writes contents to files.

4. **Create Internal Workflow Trigger for File Operations**  
   - Type: Execute Workflow Trigger (`n8n-nodes-base.executeWorkflowTrigger`)  
   - Inputs: `operation` (string), `filenames` (array), `contents` (array)  
   - Connect output to Operation Switch node.

5. **Create Operation Switch Node**  
   - Type: Switch (`n8n-nodes-base.switch`)  
   - Rules:  
     - If `operation` equals `"writeOneOrMultipleFiles"`, route to write command node.  
     - If `operation` equals `"readOneOrMultipleFiles"`, route to read command node.

6. **Create Execute Command Nodes for File Read/Write**  
   - **readOneOrMultipleFiles:**  
     - Type: Execute Command (`n8n-nodes-base.executeCommand`)  
     - Command: `cat {{ $json.filenames.join(' ') }}`  
   - **writeOneOrMultipleFiles:**  
     - Type: Execute Command  
     - Command: Use JavaScript expression to generate multiple echo commands:  
       ```javascript
       $json.filenames.map((filename, idx) => `echo "${$json.contents[idx] ?? ''}" > /home/node/${filename}`).join('\n')
       ```

7. **Connect Nodes Appropriately**  
   - MCP Server Trigger output connects to:  
     - ListDirectory  
     - CreateDirectory  
     - SearchDirectory  
     - ReadFiles  
     - WriteFiles  
   - ReadFiles and WriteFiles connect to MCP Server Trigger as AI tools (for response).  
   - When Executed by Another Workflow connects to Operation Switch.  
   - Operation Switch connects to readOneOrMultipleFiles and writeOneOrMultipleFiles nodes.

8. **Add Sticky Notes for Documentation and Reminders**  
   - Add notes linking to MCP Server Trigger docs, Execute Command tool docs, and authentication reminders.

9. **Credential Setup**  
   - No external credentials required for this workflow as it runs local shell commands.  
   - Ensure n8n instance has appropriate OS permissions to read/write `/home/node/`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This template is for self-hosted n8n instances only and requires a Linux filesystem. Modify commands if running on Windows.                                                                                                                                                                                                                                                    | Workflow description                                                                                             |
| MCP client integration instructions: Connect your MCP client by following n8n guidelines here - https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger/#integrating-with-claude-desktop                                                                                                                                                             | n8n MCP Trigger integration documentation                                                                       |
| Official MCP reference implementation for filesystem server: https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem                                                                                                                                                                                                                                         | GitHub repository                                                                                                |
| MCP Client example: Claude Desktop - https://claude.ai/download                                                                                                                                                                                                                                                                                                                | External MCP client software                                                                                      |
| Security reminder: Always enable authentication on the MCP server trigger before production to prevent unauthorized access.                                                                                                                                                                                                                                                   | Sticky Note3 content                                                                                            |
| Customizing the workflow: Add more custom workflow tools to support additional filesystem operations like moving or renaming files.                                                                                                                                                                                                                                          | Workflow description                                                                                             |
| Execute Command tool documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.executecommand/                                                                                                                                                                                                                                                        | Sticky Note1 content                                                                                            |
| MCP Server Trigger documentation: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-langchain.mcptrigger                                                                                                                                                                                                                                                          | Sticky Note content                                                                                              |

---

This structured reference document provides a comprehensive understanding of the FileSystem MCP server workflow, enabling advanced users and AI agents to analyze, reproduce, and extend the workflow securely and effectively.