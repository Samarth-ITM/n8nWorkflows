Smart Nested Folder Creation in OneDrive with Existence Checking

https://n8nworkflows.xyz/workflows/smart-nested-folder-creation-in-onedrive-with-existence-checking-7415


# Smart Nested Folder Creation in OneDrive with Existence Checking

---

### 1. Workflow Overview

This workflow automates the creation of nested folders in Microsoft OneDrive, ensuring that each folder in the nested path exists before creating the next. It is designed to handle paths like `Foobar/Barfur/Furbar`, splitting them into individual folder levels and creating each folder only if it doesn’t already exist. This approach circumvents OneDrive API limitations, which do not support creating nested folders in a single call.

**Target use cases:**  
- Automating nested folder structure creation in OneDrive.  
- Integrating with other workflows or systems that require dynamic folder management.  
- Avoiding duplicate folder creation by checking for existence before creation.

**Logical blocks:**  
- **1.1 Input Reception:** Receives nested folder path as input from another workflow.  
- **1.2 Path Parsing and Preparation:** Splits the nested folder path into components with metadata.  
- **1.3 Parent Folder Clearing:** Clears any previously stored parent folder ID before processing new folder list.  
- **1.4 Folder Existence Checking:** Searches OneDrive for each folder name to check existence under the expected parent.  
- **1.5 Conditional Folder Creation:** Creates folder if it does not exist, otherwise reuses existing folder ID.  
- **1.6 Parent Folder ID Management:** Uses datastore nodes to store and retrieve current parent folder ID dynamically during iteration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow when invoked by another workflow and accepts the nested folder path input.

**Nodes Involved:**  
- When Executed by Another Workflow

**Node Details:**

- **When Executed by Another Workflow**  
  - Type: `ExecuteWorkflowTrigger` (trigger node)  
  - Role: Starts workflow execution and accepts input parameters.  
  - Configuration: Expects a workflow input parameter named `folder`.  
  - Inputs: External workflow trigger with input JSON containing `folder` string.  
  - Outputs: Passes input data downstream to "Paths" node.  
  - Edge Cases: Missing or malformed `folder` input may lead to default path usage downstream.  
  - Version: 1.1

---

#### 2.2 Path Parsing and Preparation

**Overview:**  
Splits the nested folder path into individual folder components, preparing metadata for each folder such as its name, full path, and parent path.

**Nodes Involved:**  
- Paths  
- clearParent  
- Split Out  
- Loop Over Items

**Node Details:**

- **Paths**  
  - Type: `Code` (JavaScript)  
  - Role: Parses the input folder string into an array of objects representing each folder level.  
  - Configuration: Uses JavaScript to split folder string by `/`, maps each folder to an object with:  
    - `name`: folder name  
    - `path`: full OneDrive path to this folder  
    - `parent`: OneDrive path to parent folder  
    - `parentId`: initially null  
  - Key Expression:  
    ```js
    const folder = $input.first().json.folder || "Foobar/Barfur/Furbar";
    return {
      paths: folder.split('/').map((folder, index, arr) => ({
        name: folder,
        path: `/drive/root:/${arr.slice(0, index + 1).join('/')}`,
        parentId: null,
        parent: index === 0 ? "/drive/root:" : `/drive/root:/${arr.slice(0, index).join('/')}`
      }))
    };
    ```  
  - Input: Output from "When Executed by Another Workflow"  
  - Output: JSON with field `paths` containing array of folder objects  
  - Edge Cases: If input folder is empty or missing, defaults to `"Foobar/Barfur/Furbar"`.  
  - Version: 2

- **clearParent**  
  - Type: `Datastore` (clear operation)  
  - Role: Clears stored `parentID` key before processing new folder list to reset state.  
  - Configuration: Clears key `"parentID"` in datastore.  
  - Input: From "Paths" node  
  - Output: To "Split Out" node  
  - Edge Cases: If datastore unavailable or key does not exist, operation is idempotent.  
  - Version: 1

- **Split Out**  
  - Type: `SplitOut`  
  - Role: Splits the array of folder objects (`paths`) into separate items for iteration.  
  - Configuration: Splits by field `"paths"`.  
  - Input: From "clearParent" node  
  - Output: To "Loop Over Items" node  
  - Edge Cases: Empty input array results in no iterations.  
  - Version: 1

- **Loop Over Items**  
  - Type: `SplitInBatches`  
  - Role: Iterates over each folder object individually for processing.  
  - Configuration: Default batch size (process sequentially).  
  - Input: From "Split Out" node (individual folder JSONs)  
  - Output: Two outputs:  
    - Main output 0: continues loop for folder creation  
    - Main output 1: triggers search for folder existence  
  - Edge Cases: Empty batches stop processing.  
  - Version: 3

---

#### 2.3 Folder Existence Checking

**Overview:**  
Searches OneDrive for the folder by name, then checks if it exists under the expected parent folder path.

**Nodes Involved:**  
- Search  
- Exists  
- setParentFound

**Node Details:**

- **Search**  
  - Type: `Microsoft OneDrive` node  
  - Role: Searches OneDrive folders by name.  
  - Configuration:  
    - Resource: `folder`  
    - Operation: `search`  
    - Query: Expression `={{ $json.name }}` (folder name from current item)  
  - Credentials: Microsoft OneDrive OAuth2  
  - Input: From Loop Over Items (output 1)  
  - Output: Search results including folder metadata if found  
  - Edge Cases:  
    - API errors (auth failure, rate limits)  
    - No search results returns empty list  
  - Version: 1

- **Exists**  
  - Type: `IF` node  
  - Role: Checks if searched folders include one with matching parent path, confirming existence in correct place.  
  - Configuration:  
    - Condition 1: `$json.id` exists (folder found)  
    - Condition 2: `$json.parentReference.path` equals current loop's `parent` path  
    - Logical AND of both  
  - Input: From Search node  
  - Output:  
    - True: Folder exists  
    - False: Folder does not exist  
  - Edge Cases: If search result empty or parent path mismatched, goes to create folder path.  
  - Version: 2.2

- **setParentFound**  
  - Type: `Datastore` (set operation)  
  - Role: Stores existing folder ID as current parent folder ID for next iteration.  
  - Configuration:  
    - Key: `"parentID"`  
    - Value: `={{ $('Search').item.json.id }}` (folder ID of found folder)  
  - Input: True branch from Exists node  
  - Output: To Loop Over Items to continue iteration  
  - Edge Cases: Datastore failure may cause loss of state.  
  - Version: 1

---

#### 2.4 Conditional Folder Creation and Parent ID Management

**Overview:**  
If folder does not exist, create it under the current parent folder ID, then update parent ID for subsequent folder creation.

**Nodes Involved:**  
- getParent  
- Create a folder  
- setParentCreated

**Node Details:**

- **getParent**  
  - Type: `Datastore` (get operation)  
  - Role: Retrieves the current parent folder ID to create new folder under it.  
  - Configuration:  
    - Key: `"parentID"`  
    - Operation: get  
  - Input: False branch from Exists node (folder does not exist)  
  - Output: Passes parentID downstream for folder creation  
  - Edge Cases: Missing parentID means root or null parent.  
  - Version: 1

- **Create a folder**  
  - Type: `Microsoft OneDrive` node  
  - Role: Creates a new folder in OneDrive under the specified parent folder ID.  
  - Configuration:  
    - Resource: `folder`  
    - Operation: `create`  
    - Name: `={{ $('Split Out').item.json.name }}` (current folder name)  
    - Parent Folder ID: `={{ $json.value || null }}` (from getParent node; null means root)  
  - Credentials: Microsoft OneDrive OAuth2  
  - Input: From getParent node  
  - Output: New folder metadata including new folder ID  
  - Edge Cases:  
    - API errors (auth, quota)  
    - Name conflicts if folder was created in parallel externally  
  - Version: 1

- **setParentCreated**  
  - Type: `Datastore` (set operation)  
  - Role: Stores newly created folder ID as current parent folder ID for next iteration.  
  - Configuration:  
    - Key: `"parentID"`  
    - Value: `={{ $json.id }}` (ID of newly created folder)  
  - Input: From Create a folder node  
  - Output: Loops back to Loop Over Items to continue processing next folder  
  - Edge Cases: Datastore failure may cause loss of state.  
  - Version: 1

---

#### 2.5 Sticky Notes and Annotations

**Nodes Involved:**  
- Sticky Note1  
- Sticky Note

**Node Details:**

- **Sticky Note1**  
  - Purpose: Overall workflow description and usage notes.  
  - Content Highlights:  
    - Explains purpose: create nested folders in Microsoft OneDrive by looping and creating if they don't exist.  
    - Notes default test path usage in "Paths" node.  
    - Advises usage as a Sub-workflow with `$folder` input variable.  
  - Position: Top-left, visually covering input and path parsing nodes.

- **Sticky Note**  
  - Purpose: Describes folder creation step and storing folder ID.  
  - Content Highlights:  
    - Notes that folder creation only happens if folder does not exist.  
    - Explains passing and storing created folder ID for next iteration.  
  - Position: Near folder creation nodes.

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                                | Input Node(s)                   | Output Node(s)               | Sticky Note                              |
|----------------------------|----------------------------|-----------------------------------------------|--------------------------------|-----------------------------|-----------------------------------------|
| When Executed by Another Workflow | ExecuteWorkflowTrigger       | Trigger workflow and receive `folder` input  | External workflow trigger       | Paths                       | See Sticky Note1 for overall workflow info |
| Paths                      | Code                       | Parse folder string into folder objects array | When Executed by Another Workflow | clearParent                 | See Sticky Note1                        |
| clearParent                | Datastore (clear)          | Clear stored parentID before processing       | Paths                          | Split Out                   | See Sticky Note1                        |
| Split Out                  | SplitOut                   | Split array of folder objects into individual items | clearParent                   | Loop Over Items             | See Sticky Note1                        |
| Loop Over Items            | SplitInBatches             | Iterate over each folder item                  | Split Out                     | Search (output 1), (output 0 unused) | See Sticky Note                        |
| Search                     | Microsoft OneDrive         | Search OneDrive for folder by name             | Loop Over Items (output 1)      | Exists                      |                                         |
| Exists                     | IF                         | Check if folder exists under expected parent  | Search                        | setParentFound (true), getParent (false) |                                         |
| setParentFound             | Datastore (set)            | Store existing folder ID as parent ID          | Exists (true branch)           | Loop Over Items             | See Sticky Note                         |
| getParent                  | Datastore (get)            | Retrieve current parent folder ID              | Exists (false branch)          | Create a folder             | See Sticky Note                         |
| Create a folder            | Microsoft OneDrive         | Create folder under current parent ID          | getParent                     | setParentCreated            | See Sticky Note                         |
| setParentCreated           | Datastore (set)            | Store newly created folder ID as parent ID     | Create a folder               | Loop Over Items             | See Sticky Note                         |
| Sticky Note1               | Sticky Note                | Workflow purpose, usage, and testing notes     | N/A                          | N/A                         | Overall workflow description           |
| Sticky Note                | Sticky Note                | Explains folder creation and parentID storing  | N/A                          | N/A                         | Folder creation explanation             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add `ExecuteWorkflowTrigger` node named "When Executed by Another Workflow".  
   - Configure input parameter: `folder` (string).  
   - This node starts the workflow and receives the nested folder path.

2. **Parse Folder Path:**  
   - Add a `Code` node named "Paths".  
   - Use JavaScript to split the input folder path string by `/` into an array of folder objects with:  
     - `name` (folder name)  
     - `path` (full OneDrive path for this folder)  
     - `parent` (OneDrive path of parent folder)  
     - `parentId` null initially  
   - Example JS code:  
     ```js
     const folder = $input.first().json.folder || "Foobar/Barfur/Furbar";
     return {
       paths: folder.split('/').map((folder, index, arr) => ({
         name: folder,
         path: `/drive/root:/${arr.slice(0, index + 1).join('/')}`,
         parentId: null,
         parent: index === 0 ? "/drive/root:" : `/drive/root:/${arr.slice(0, index).join('/')}`
       }))
     };
     ```
   - Connect output of trigger node to this node.

3. **Clear Previous Parent ID:**  
   - Add a `Datastore` node named "clearParent".  
   - Configure operation: `clear` key `"parentID"`.  
   - Connect "Paths" node output to this node.

4. **Split Folder Paths into Items:**  
   - Add `SplitOut` node named "Split Out".  
   - Configure to split by the field `"paths"`.  
   - Connect "clearParent" node output to this node.

5. **Loop Over Each Folder:**  
   - Add `SplitInBatches` node named "Loop Over Items".  
   - Default batch size (process sequentially).  
   - Connect "Split Out" node output to this node.

6. **Search for Folder Existence:**  
   - Add Microsoft OneDrive node named "Search".  
   - Configure:  
     - Resource: `folder`  
     - Operation: `search`  
     - Query: expression `={{ $json.name }}` to search current folder name.  
   - Connect "Loop Over Items" node output 1 (second output) to "Search" node.

7. **Check Folder Existence Condition:**  
   - Add `IF` node named "Exists".  
   - Configure conditions (AND):  
     - Check if `$json.id` exists (folder found).  
     - Check if `$json.parentReference.path` equals current folder's `parent` path from "Loop Over Items".  
   - Connect "Search" output to "Exists" input.

8. **If Folder Exists - Store Parent ID:**  
   - Add `Datastore` node named "setParentFound".  
   - Configure operation: `set`  
   - Key: `"parentID"`  
   - Value: expression `={{ $('Search').item.json.id }}` (ID of found folder)  
   - Connect "Exists" true output to this node.  
   - Connect this node back to "Loop Over Items" to continue.

9. **If Folder Does Not Exist - Retrieve Parent ID:**  
   - Add `Datastore` node named "getParent".  
   - Configure operation: `get`  
   - Key: `"parentID"`  
   - Connect "Exists" false output to this node.

10. **Create Folder in OneDrive:**  
    - Add Microsoft OneDrive node named "Create a folder".  
    - Configure:  
      - Resource: `folder`  
      - Operation: `create`  
      - Name: expression `={{ $('Split Out').item.json.name }}` (current folder name)  
      - Parent Folder ID: expression `={{ $json.value || null }}` (from "getParent")  
    - Connect "getParent" output to this node.

11. **Store Created Folder ID:**  
    - Add `Datastore` node named "setParentCreated".  
    - Configure operation: `set`  
    - Key: `"parentID"`  
    - Value: expression `={{ $json.id }}` (new folder ID)  
    - Connect "Create a folder" output to this node.  
    - Connect this node back to "Loop Over Items" to continue.

12. **Add Sticky Notes:**  
    - Add sticky note near input and parsing nodes explaining workflow purpose and usage (similar to Sticky Note1).  
    - Add sticky note near creation nodes explaining folder creation only if not exists and storing parent ID (similar to Sticky Note).

13. **Credentials Setup:**  
    - Configure Microsoft OneDrive OAuth2 credentials for all OneDrive nodes ("Search" and "Create a folder").  
    - Ensure credential is authorized with sufficient scopes to read and create folders.

14. **Save and Activate Workflow:**  
    - Save all nodes and activate the workflow.  
    - Test by triggering it from another workflow passing a nested folder path as input.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                             |
|------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------|
| The workflow is designed as a sub-workflow; invoke it with the nested folder path as the `$folder` input variable in another workflow.   | Usage instructions in Sticky Note1         |
| OneDrive API requires creating nested folders step-by-step; this workflow automates that process efficiently.                            | Workflow Overview                           |
| Datastore nodes are used to maintain state (`parentID`) between iterations without external storage.                                     | Parent ID management analysis               |
| API error handling is minimal; consider adding error nodes or retry logic for production use.                                            | Edge cases in OneDrive nodes                |
| Default test path in the "Paths" node is `Foobar/Barfur/Furbar`. Pass custom path to override.                                          | Paths node JS code and notes                 |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---