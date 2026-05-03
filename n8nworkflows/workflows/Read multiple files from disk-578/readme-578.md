Read multiple files from disk

https://n8nworkflows.xyz/workflows/read-multiple-files-from-disk-578


# Read multiple files from disk

### 1. Workflow Overview

This workflow is designed to read multiple binary files from disk, specifically targeting all JPEG image files located in the `/data/lol/` directory. It serves as a companion example for the "Read Binary Files" node documentation, demonstrating how to trigger and process multiple files matching a given file path pattern.

Logical blocks included:  
- **1.1 Manual Trigger**: Initiates the workflow execution on demand.  
- **1.2 Binary File Reading**: Reads multiple files from a specified directory path pattern and outputs their binary content.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  This block provides a manual execution starting point for the workflow, allowing the user to trigger reading files at any time without external input.

- **Nodes Involved:**  
  - *On clicking 'execute'*

- **Node Details:**  
  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger  
  - **Technical Role:** Entry point node that waits for a manual trigger to start the workflow.  
  - **Configuration Choices:** No parameters configured; default manual trigger behavior.  
  - **Key Expressions/Variables:** None.  
  - **Input Connections:** None (start node).  
  - **Output Connections:** Connected to *Read Binary Files* node.  
  - **Version Requirements:** Compatible with n8n version 0.100.0 and later where manual trigger is supported.  
  - **Potential Failures:** None typical; user must manually trigger.  
  - **Sub-workflow Reference:** None.

#### 1.2 Binary File Reading

- **Overview:**  
  This block reads all files matching the pattern `/data/lol/*.jpg` from the local filesystem, loading their binary content into the workflow for further processing.

- **Nodes Involved:**  
  - *Read Binary Files*

- **Node Details:**  
  - **Node Name:** Read Binary Files  
  - **Type:** Read Binary Files  
  - **Technical Role:** Reads multiple binary files from disk based on glob file selector input.  
  - **Configuration Choices:**  
    - *File Selector:* `/data/lol/*.jpg` - uses a glob pattern to select all JPG files in the specified directory.  
  - **Key Expressions/Variables:** None beyond the static file selector string.  
  - **Input Connections:** Receives trigger from the manual trigger node.  
  - **Output Connections:** None; terminal node in this workflow.  
  - **Version Requirements:** Requires n8n version 0.150.0 or later for glob pattern support in the Read Binary Files node.  
  - **Potential Failures:**  
    - Path does not exist or is inaccessible (permissions).  
    - No files matching pattern found (empty output).  
    - File read errors (locked files, corrupt files).  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role        | Input Node(s)        | Output Node(s)      | Sticky Note                                |
|---------------------|----------------------|-----------------------|----------------------|---------------------|--------------------------------------------|
| On clicking 'execute'| Manual Trigger       | Workflow start trigger | None                 | Read Binary Files   |                                            |
| Read Binary Files    | Read Binary Files    | Reads multiple JPG files from disk | On clicking 'execute' | None                | Companion workflow for Read Binary Files node docs |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `On clicking 'execute'`  
   - Type: `Manual Trigger`  
   - No parameters need to be set. This node will act as the starting point for manual execution.

2. **Create Read Binary Files Node**  
   - Name: `Read Binary Files`  
   - Type: `Read Binary Files`  
   - Configuration:  
     - Set *File Selector* to `/data/lol/*.jpg` to select all JPEG files in `/data/lol/` directory using a glob pattern.  
   - No credentials required since reading local disk files.

3. **Connect Nodes**  
   - Connect the output of `On clicking 'execute'` node to the input of the `Read Binary Files` node.

4. **Save and Execute Workflow**  
   - Save the workflow.  
   - Click "Execute Workflow" manually to trigger the file reading process.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow is a companion example specifically for demonstrating the "Read Binary Files" node usage.   | See official n8n documentation on the Read Binary Files node for more details. |
| Ensure the directory `/data/lol/` exists and contains `.jpg` files for this workflow to produce output.   | Local disk prerequisite                                |
| Reading files requires proper file system permissions for the n8n process user.                          | Operating system file permissions                       |

---

This documentation enables advanced users and AI agents to understand, reproduce, and extend the workflow while anticipating typical failure modes and configuration needs.