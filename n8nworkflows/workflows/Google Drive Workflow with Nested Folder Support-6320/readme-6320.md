Google Drive Workflow with Nested Folder Support

https://n8nworkflows.xyz/workflows/google-drive-workflow-with-nested-folder-support-6320


# Google Drive Workflow with Nested Folder Support

---

### 1. Workflow Overview

This workflow is designed to retrieve all files inside a specified Google Drive folder, including support for nested folders. It can handle cases where the parent folder contains subfolders or only files, iteratively collecting all files recursively in nested folder structures.

Logical blocks:

- **1.1 Input Reception and Initial Folder Listing:** Starts the workflow and lists children folders inside the given Google Drive folder.
- **1.2 Folder Existence Decision:** Determines if the parent folder contains nested folders or not.
- **1.3 Recursive Folder ID Collection:** When nested folders exist, recursively collects all folder IDs including nested ones.
- **1.4 File Retrieval Loop:** Iterates over collected folder IDs to get all files inside each folder.
- **1.5 Output Preparation:** Sets and returns the list of all file IDs found.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initial Folder Listing

- **Overview:** This block starts the workflow execution and fetches the immediate children folders of the specified parent folder ID.
- **Nodes Involved:** `Start Execution`, `Get children folders from folder id`, `If parent folder has nested folders or not`
  
##### Nodes:

- **Start Execution**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Entry point triggering the workflow execution.  
  - *Configuration:* Default trigger node, no parameters.  
  - *Connections:* Output to `Get children folders from folder id`.  
  - *Failure modes:* None typical; workflow trigger errors are rare unless misconfigured.

- **Get children folders from folder id**  
  - *Type:* Google Drive node  
  - *Role:* Lists children folders under the given parent folder ID.  
  - *Configuration:* Configured to fetch folders; uses Google Drive OAuth2 credentials.  
  - *Key variables:* Uses input parameter for parent folder ID (likely from trigger or prior node).  
  - *Connections:* Output to `If parent folder has nested folders or not`.  
  - *Failure modes:* Authentication errors, API rate limits, invalid folder ID errors, network timeouts.

- **If parent folder has nested folders or not**  
  - *Type:* If node  
  - *Role:* Conditional split based on whether the folder contains nested folders.  
  - *Configuration:* Evaluates if the output of `Get children folders` contains any folders.  
  - *Connections:* True branch to `Start sub-execution` (recursive folder collection), False branch to `If parent folder has no folder`.  
  - *Failure modes:* Expression evaluation errors if input data malformed.

---

#### 1.2 Folder Existence Decision and Simple Return

- **Overview:** Handles the case where no nested folders exist by returning the parent folder ID directly or passing it to the sub-workflow.
- **Nodes Involved:** `If parent folder has no folder`, `Return parent folder id`, `Return parent folder id to sub-workflow`

##### Nodes:

- **If parent folder has no folder**  
  - *Type:* If node  
  - *Role:* Distinguishes between two return flows for non-nested folder scenarios.  
  - *Configuration:* Checks if the folder truly has no nested folders to decide the return path.  
  - *Connections:* True branch to `Return parent folder id`, False branch to `Return parent folder id to sub-workflow`.  
  - *Failure modes:* Expression errors, unexpected input formats.

- **Return parent folder id**  
  - *Type:* Set node  
  - *Role:* Prepares the parent folder ID data to be sent downstream for folders without nested folders.  
  - *Configuration:* Sets output data with the parent folder ID.  
  - *Connections:* Output to `Loop Over folder ids`.  
  - *Failure modes:* Data formatting issues.

- **Return parent folder id to sub-workflow**  
  - *Type:* Set node  
  - *Role:* Prepares parent folder ID to return to sub-workflow call if needed.  
  - *Configuration:* Sets data for output to sub-workflow caller.  
  - *Connections:* No output connections (end of this path).  
  - *Failure modes:* Data formatting issues.

---

#### 1.3 Recursive Folder ID Collection

- **Overview:** When nested folders exist, this block triggers a sub-workflow recursively to collect all nested folder IDs.
- **Nodes Involved:** `Start sub-execution`, `Get all folder ids`, `Loop Over folder ids`

##### Nodes:

- **Start sub-execution**  
  - *Type:* Execute Workflow  
  - *Role:* Calls the same or related workflow recursively to gather folder IDs from nested folders.  
  - *Configuration:* Invoked with parent folder ID parameter.  
  - *Connections:* Output to `Get all folder ids`.  
  - *Failure modes:* Workflow call failure, parameter passing errors.

- **Get all folder ids**  
  - *Type:* Code node  
  - *Role:* Processes input data to extract or compile a list of folder IDs from recursion results.  
  - *Configuration:* Contains JavaScript code to gather folder IDs in an array.  
  - *Connections:* Output to `Loop Over folder ids`.  
  - *Failure modes:* Code errors, empty or malformed input data.

- **Loop Over folder ids**  
  - *Type:* SplitInBatches  
  - *Role:* Iterates over each folder ID to process files or nested folders in batches.  
  - *Configuration:* Batch size not specified, defaults to 1 or configured as needed.  
  - *Connections:* Multiple outputs—one to `Return all file ids`, another to `Get all files inside a folder`.  
  - *Failure modes:* Batch processing errors, empty input array.

---

#### 1.4 File Retrieval Loop

- **Overview:** This block loops over all collected folder IDs to retrieve all files inside each folder.
- **Nodes Involved:** `Get all files inside a folder`, `Return all file ids`

##### Nodes:

- **Get all files inside a folder**  
  - *Type:* Google Drive node  
  - *Role:* Retrieves files inside each folder ID passed from the batch loop.  
  - *Configuration:* Configured to list files (not folders), using Google Drive OAuth2 credentials.  
  - *Connections:* Output loops back to `Loop Over folder ids` for further iterations.  
  - *Failure modes:* API errors, authentication issues, invalid folder IDs.

- **Return all file ids**  
  - *Type:* Set node  
  - *Role:* Aggregates and formats the list of all file IDs found for output.  
  - *Configuration:* Sets output data containing file IDs.  
  - *Connections:* No further output connections (end of workflow).  
  - *Failure modes:* Data aggregation issues.

---

#### 1.5 Sticky Notes

- **Overview:** Sticky notes are placed for documentation or annotation purposes, but contain no content. They do not affect workflow logic.
- **Nodes Involved:** `Sticky Note`, `Sticky Note1`, `Sticky Note2`
- **Details:** Located near the start and decision nodes, likely placeholders for future comments.

---

### 3. Summary Table

| Node Name                         | Node Type               | Functional Role                            | Input Node(s)                     | Output Node(s)                                | Sticky Note |
|----------------------------------|-------------------------|--------------------------------------------|----------------------------------|-----------------------------------------------|-------------|
| Start Execution                  | Execute Workflow Trigger| Entry point to start workflow execution    |                                  | Get children folders from folder id           |             |
| Get children folders from folder id | Google Drive           | Lists child folders of given folder ID     | Start Execution                  | If parent folder has nested folders or not    |             |
| If parent folder has nested folders or not | If                     | Checks if folder has nested folders        | Get children folders from folder id | Start sub-execution, If parent folder has no folder |             |
| Start sub-execution              | Execute Workflow        | Recursively get all nested folder IDs      | If parent folder has nested folders or not | Get all folder ids                             |             |
| Get all folder ids               | Code                    | Extracts all folder IDs from recursion     | Start sub-execution              | Loop Over folder ids                           |             |
| Loop Over folder ids             | SplitInBatches          | Iterates over folder IDs for file retrieval| Get all folder ids, Return parent folder id, Get all files inside a folder | Return all file ids, Get all files inside a folder |             |
| Get all files inside a folder    | Google Drive            | Retrieves files inside a folder             | Loop Over folder ids             | Loop Over folder ids                           |             |
| Return all file ids              | Set                     | Aggregates all file IDs for output          | Loop Over folder ids             |                                               |             |
| If parent folder has no folder   | If                      | Branches when no nested folders exist       | If parent folder has nested folders or not | Return parent folder id, Return parent folder id to sub-workflow |             |
| Return parent folder id          | Set                     | Outputs parent folder ID for non-nested case| If parent folder has no folder   | Loop Over folder ids                           |             |
| Return parent folder id to sub-workflow | Set               | Returns folder ID to sub-workflow caller    | If parent folder has no folder   |                                               |             |
| Sticky Note                     | Sticky Note             | Annotation (empty)                          |                                  |                                               |             |
| Sticky Note1                    | Sticky Note             | Annotation (empty)                          |                                  |                                               |             |
| Sticky Note2                    | Sticky Note             | Annotation (empty)                          |                                  |                                               |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it appropriately (e.g., "Google Drive Workflow with Nested Folder Support").

2. **Add a `Execute Workflow Trigger` node** named `Start Execution`. This is your workflow entry point.

3. **Add a `Google Drive` node** named `Get children folders from folder id`.  
   - Set operation to "List" and configure it to list folders only (filter by MIME type `application/vnd.google-apps.folder`).  
   - Set the parent folder ID dynamically from the trigger input (expression referencing execution input).  
   - Connect `Start Execution` output to this node.

4. **Add an `If` node** named `If parent folder has nested folders or not`.  
   - Condition: Check if output of `Get children folders from folder id` contains any folders.  
   - Connect `Get children folders from folder id` output to this node.

5. **Add an `Execute Workflow` node** named `Start sub-execution`.  
   - Configure it to call the same workflow or a sub-workflow that recursively retrieves folder IDs.  
   - Pass the parent folder ID as input parameter.  
   - Connect the **true** branch of `If parent folder has nested folders or not` to this node.

6. **Add a `Set` node** named `Return parent folder id to sub-workflow`.  
   - Configure it to set data returning the current parent folder ID to the caller.  
   - Connect the **false** branch of `If parent folder has nested folders or not` to this node.

7. **Add another `If` node** named `If parent folder has no folder`.  
   - Configure it to distinguish between returning the parent folder ID directly or passing it to the sub-workflow (details depend on your exact needs).  
   - Connect the output from the false branch of `If parent folder has nested folders or not` to this node.

8. **Add a `Set` node** named `Return parent folder id`.  
   - Sets the parent folder ID as data for further processing.  
   - Connect the true branch of `If parent folder has no folder` to this node.

9. **Add a `Code` node** named `Get all folder ids`.  
   - Write JavaScript code to process and collect all folder IDs from the recursive workflow execution results.  
   - Connect the output of `Start sub-execution` to this node.

10. **Add a `SplitInBatches` node** named `Loop Over folder ids`.  
    - Configure batch size as desired (default 1 or more).  
    - Connect outputs from `Return parent folder id` and `Get all folder ids` to this node.

11. **Add a `Google Drive` node** named `Get all files inside a folder`.  
    - Configure it to list files (exclude folders) inside the folder ID passed from batch node.  
    - Connect one output of `Loop Over folder ids` to this node.

12. **Add a `Set` node** named `Return all file ids`.  
    - Configure it to aggregate and output all file IDs collected from the Google Drive query.  
    - Connect the other output of `Loop Over folder ids` to this node.

13. **Connect `Get all files inside a folder` output back to `Loop Over folder ids`** for iterative processing.

14. **Ensure Google Drive credentials** are properly configured with OAuth2 authentication in all Google Drive nodes.

15. Optionally, **add sticky notes** near key nodes to document purpose or caveats.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow demonstrates recursive folder traversal in Google Drive via n8n Google Drive nodes.| Useful for automating complex folder structures and bulk file retrieval in Google Drive.      |
| Google Drive API limits and permissions may affect runtime; ensure OAuth2 credentials have proper scopes.| https://developers.google.com/drive/api/v3/about-auth                                      |
| Recursive sub-workflow calls must be carefully managed to avoid infinite loops or excessive API calls.| Best practice: limit recursion depth or batch sizes.                                         |
| Sticky notes currently empty; recommended to add contextual explanations for maintainability.   | n8n Sticky Notes feature                                                                        |

---

**Disclaimer:** The text provided is derived solely from an automated workflow created in n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.

---