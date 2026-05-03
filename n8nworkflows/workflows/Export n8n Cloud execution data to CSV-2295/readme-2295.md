Export n8n Cloud execution data to CSV

https://n8nworkflows.xyz/workflows/export-n8n-cloud-execution-data-to-csv-2295


# Export n8n Cloud execution data to CSV

### 1. Workflow Overview

This workflow is designed to assist n8n Cloud users in exporting all execution data from their n8n instance into a CSV file format. The primary use case is to enable easier data analysis, reporting, and monitoring of workflow executions for optimization and auditing purposes.

The workflow logically divides into three blocks:

- **1.1 Input Reception:** Manual trigger to start the export process.
- **1.2 Data Retrieval:** Fetching all execution records from the n8n Cloud API.
- **1.3 Data Conversion and Output:** Converting the retrieved execution data into a CSV file and preparing it for download or further use.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually when a user clicks the "Test Workflow" button in the n8n editor.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  

  - **Node Name:** When clicking ‘Test workflow’  
    - **Type & Technical Role:** Manual Trigger Node - starts workflow execution manually.  
    - **Configuration:** Default manual trigger with no parameters configured.  
    - **Expressions/Variables:** None.  
    - **Input/Output:** No input; output connects to "n8n | Get all executions".  
    - **Version-specific Requirements:** Compatible with n8n version supporting manual trigger (standard).  
    - **Edge Cases:** Workflow will not run unless manually triggered. No automatic scheduling. No failure expected unless n8n internal error occurs.  
    - **Sub-workflow:** None.

#### 1.2 Data Retrieval

- **Overview:**  
  This block fetches all workflow execution records from the connected n8n Cloud instance via the API, providing raw execution data for export.

- **Nodes Involved:**  
  - n8n | Get all executions

- **Node Details:**  

  - **Node Name:** n8n | Get all executions  
    - **Type & Technical Role:** n8n API Node - requests execution data from the n8n instance.  
    - **Configuration:**  
      - Resource: Execution  
      - Return All: True (fetches all executions without pagination limits)  
      - Filters: None set by default, but can be configured for workflow or status filters.  
    - **Credentials:** Uses n8n API credential with access rights to fetch executions.  
    - **Expressions/Variables:** None explicitly, but filters could be configured with expressions if customized.  
    - **Input/Output:** Input from Manual Trigger; output to "Convert to CSV".  
    - **Version-specific Requirements:** Requires n8n Cloud API access and credentials set up.  
    - **Edge Cases:**  
      - Authentication failure if credentials are invalid.  
      - Timeout or API rate limits if large data sets exist.  
      - Empty results if no executions recorded or filters exclude all data.  
    - **Sub-workflow:** None.

- **Sticky Note Content:**  
  "**Get all executions**  
  Workflow and Status Filters can be applied here"

#### 1.3 Data Conversion and Output

- **Overview:**  
  This block converts the JSON array of execution data to a CSV file format and prepares the binary data for download or further cloud storage.

- **Nodes Involved:**  
  - Convert to CSV  
  - No Operation, do nothing

- **Node Details:**  

  - **Node Name:** Convert to CSV  
    - **Type & Technical Role:** Convert to File Node - converts JSON input to CSV binary output.  
    - **Configuration:** Default CSV conversion options; no special delimiter or encoding specified.  
    - **Expressions/Variables:** None.  
    - **Input/Output:** Input from "n8n | Get all executions"; output to "No Operation, do nothing".  
    - **Version-specific Requirements:** Uses version 1.1 which supports improved conversion features.  
    - **Edge Cases:**  
      - Conversion may fail if input JSON is malformed or empty.  
      - Large datasets may cause performance delays.  
    - **Sub-workflow:** None.

  - **Node Name:** No Operation, do nothing  
    - **Type & Technical Role:** NoOp Node - placeholder node that does nothing with input.  
    - **Configuration:** None; default settings.  
    - **Expressions/Variables:** None.  
    - **Input/Output:** Input from "Convert to CSV"; no output connections.  
    - **Version-specific Requirements:** Standard.  
    - **Edge Cases:** None; node is a placeholder.  
    - **Usage:** Recommended to be replaced by a node that uploads the CSV file to a cloud storage destination or sends it via email.

- **Sticky Note Content:**  
  "**Convert to CSV**  
  CSV for easy parsing"  
  and  
  "**Replace this node**  
  Replace this node with any cloud storage destination"

---

### 3. Summary Table

| Node Name                  | Node Type                   | Functional Role                      | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                   |
|----------------------------|-----------------------------|------------------------------------|---------------------------|----------------------------|-----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger               | Initiates workflow manually         | None                      | n8n | Get all executions          |                                                                                               |
| n8n | Get all executions         | n8n API Node                  | Fetches all executions from n8n API | When clicking ‘Test workflow’ | Convert to CSV              | Get all executions; Workflow and Status Filters can be applied here                           |
| Convert to CSV             | Convert to File              | Converts JSON execution data to CSV | n8n | Get all executions          | No Operation, do nothing       | Convert to CSV; CSV for easy parsing                                                          |
| No Operation, do nothing   | NoOp                        | Placeholder for output handling     | Convert to CSV             | None                       | Replace this node; Replace this node with any cloud storage destination                        |
| Sticky Note                | Sticky Note                 | Annotation/Comment                  | None                      | None                       |                                                                                               |
| Sticky Note1               | Sticky Note                 | Annotation/Comment                  | None                      | None                       |                                                                                               |
| Sticky Note2               | Sticky Note                 | Annotation/Comment                  | None                      | None                       |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’".  
   - Leave default settings (no parameters).  
   - This node will serve as the entry point to manually start the workflow.

2. **Create n8n API Node to Fetch Executions**  
   - Add an **n8n** node named "n8n | Get all executions".  
   - Set **Resource** to `Execution`.  
   - Enable **Return All** to fetch all execution records.  
   - Leave **Filters** empty or configure as desired to limit workflows or statuses.  
   - Connect the output of the Manual Trigger node to this node’s input.  
   - Configure credentials: set up and select an **n8n API credential** with access to your n8n Cloud instance.

3. **Create Convert to File Node**  
   - Add a **Convert to File** node named "Convert to CSV".  
   - Set the conversion to produce a CSV file (default for JSON to CSV conversion).  
   - Connect the output of "n8n | Get all executions" to this node's input.  
   - Use default conversion options; no special parameters required.

4. **Add No Operation Node**  
   - Add a **NoOp** node named "No Operation, do nothing".  
   - Connect the output of the "Convert to CSV" node to this node's input.  
   - This node acts as a placeholder and should be replaced with a node that handles the CSV binary output—such as an upload to cloud storage or an email send node.

5. **Optional: Add Sticky Notes**  
   - Add sticky notes for documentation and clarity as per the original workflow:  
     - Over "n8n | Get all executions" explaining filters.  
     - Over "Convert to CSV" explaining conversion purpose.  
     - Over "No Operation" node recommending replacement with cloud storage or other destinations.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This template is intended for n8n Cloud plan users to simplify exporting execution data to CSV for analysis and optimization.                                                                                                                     | Workflow description                                                                                                 |
| Click ["Test Workflow"](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.manualworkflowtrigger/) to manually execute the workflow.                                                                                             | Manual trigger documentation                                                                                        |
| Instructions to import workflows: [Import the workflow](https://docs.n8n.io/workflows/export-import/)                                                                                                                                               | n8n Docs                                                                                                            |
| Exported CSV files enable comprehensive insights, custom reporting, historical data review, error tracking, and help maintain audit trails for compliance.                                                                                       | Benefits section                                                                                                    |
| Recommended to replace the No Operation node with a cloud storage node (e.g., Google Drive, AWS S3) or email node to save or distribute the CSV file.                                                                                             | Sticky Note2 content                                                                                                 |
| Screenshot showing access to CSV binary data after execution available inside "Convert to CSV" node output.                                                                                                                                         | Image reference in description (fileId:811)                                                                         |

---

This structured reference should enable users and automation agents to fully understand, reproduce, and extend the workflow for exporting n8n Cloud execution data to CSV files.