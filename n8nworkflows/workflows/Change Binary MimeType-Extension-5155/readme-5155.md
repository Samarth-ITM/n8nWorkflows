Change Binary MimeType/Extension

https://n8nworkflows.xyz/workflows/change-binary-mimetype-extension-5155


# Change Binary MimeType/Extension

### 1. Workflow Overview

This n8n workflow titled **"Change Binary MimeType/Extension"** is designed to modify the MIME type and file extension of a binary file within an n8n automation. The primary use case is when a file's binary data is correct but its MIME type or extension needs to be changed to ensure compatibility with downstream nodes or external services. This can be useful for workflows dealing with file format normalization, data routing, or interfacing with APIs that rely on correct MIME types.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Captures the incoming binary data through a workflow trigger.
- **1.2 File Metadata Configuration:** Sets the target file name and extension that determine the new MIME type.
- **1.3 Binary Data Transformation:** Extracts binary content as Base64, then rebuilds the binary with the new extension and MIME type.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
This block receives the input data in binary form and initiates the workflow. It acts as the entry point.

- **Nodes Involved:**  
  - Change Binary MimeType/Extension (Execute Workflow Trigger)

- **Node Details:**  
  - **Node Name:** Change Binary MimeType/Extension  
  - **Type:** Execute Workflow Trigger  
  - **Role:** Triggers the workflow and passes through all incoming data without modification.  
  - **Configuration:** Set to passthrough mode to forward incoming data as-is.  
  - **Input/Output:** No prior input; outputs data to the next node.  
  - **Version:** 1.1  
  - **Potential Failures:** Misconfiguration of trigger; no data received; execution timeout.  
  - **Sub-workflow:** This node is the main trigger, no further sub-workflows invoked.

#### 1.2 File Metadata Configuration

- **Overview:**  
This block allows the user to specify the desired output file name and extension, which determines the binary's new MIME type.

- **Nodes Involved:**  
  - SET OUTPUT FILE NAME (Set Node)  
  - Sticky Note (Instructional)  

- **Node Details:**  
  - **Node Name:** SET OUTPUT FILE NAME  
  - **Type:** Set  
  - **Role:** Defines two key fields:  
    - `binary_key` (string): dynamically identifies the name of the incoming binary property.  
    - `output_file_name` (string): user-configured output file name including the extension.  
  - **Configuration:**  
    - `binary_key`: Set via expression `={{ Object.keys($binary).first() }}`, automatically fetching the first binary property's key from incoming data.  
    - `output_file_name`: Default placeholder text prompting user to set the desired output file name and extension (e.g., "audio.mp3" or "image.png").  
  - **Input/Output:** Receives data from the trigger node; outputs to "Extract from File".  
  - **Version:** 3.4  
  - **Edge Cases:** User failure to update `output_file_name` leads to incorrect output or workflow errors; no binary data present causes expression to fail.  
  - **Sticky Note:** Attached instructional note explains configuration necessities and cautions not to edit `binary_key`.

  - **Node Name:** Sticky Note  
  - **Type:** Sticky Note  
  - **Role:** Provides user instructions on configuring the `output_file_name` and maintaining `binary_key` as dynamic.  
  - **Content Summary:**  
    - Instructs user to set the output file name and extension correctly.  
    - Warns not to modify the `binary_key`.  
  - **Position:** Visually linked to "SET OUTPUT FILE NAME".

#### 1.3 Binary Data Transformation

- **Overview:**  
This block performs the essential logic to change the binary data's MIME type and extension by re-encoding the binary data based on user configuration.

- **Nodes Involved:**  
  - Extract from File (ExtractFromFile Node)  
  - Change Binary Data Type (Code Node)  
  - Sticky Note1 (Instructional)

- **Node Details:**  
  - **Node Name:** Extract from File  
  - **Type:** ExtractFromFile  
  - **Role:** Converts the binary data into a Base64 encoded string property for manipulation in code.  
  - **Configuration:**  
    - Operation: `binaryToPropery`  
    - Binary property name: dynamically set with expression `={{ Object.keys($binary).first() }}` to grab the first binary input.  
  - **Input/Output:** Receives data from "SET OUTPUT FILE NAME"; outputs to "Change Binary Data Type".  
  - **Version:** 1  
  - **Failure Modes:** No binary data present; expression failure if binary key is missing; file corruption in conversion.  

  - **Node Name:** Change Binary Data Type  
  - **Type:** Code (JavaScript)  
  - **Role:** Rebuilds the binary data buffer using the Base64 string and assigns it a new file name and extension, effectively changing the MIME type.  
  - **Configuration:**  
    - Uses Node.js Buffer to decode Base64 string.  
    - Calls `this.helpers.prepareBinaryData()` with the buffer and the user-defined `output_file_name` from "SET OUTPUT FILE NAME".  
    - Dynamically sets the binary key to the original binary property name.  
  - **Key Expressions:**  
    - `$json.data` (Base64 string from previous node)  
    - `$input.item.binary` assignment with dynamic file name  
  - **Input/Output:** Receives from "Extract from File"; outputs transformed binary data.  
  - **Version:** 2  
  - **Potential Failures:** Buffer decode errors; missing or malformed Base64 data; invalid file extension causing helper failure.  

  - **Node Name:** Sticky Note1  
  - **Type:** Sticky Note  
  - **Role:** Explains the core logic of this block and advises no editing needed in these nodes.  
  - **Content Summary:**  
    - Describes the role of extracting binary to Base64 and reconstructing with new MIME type.  
    - Emphasizes the process changes file format perception for downstream usage.

---

### 3. Summary Table

| Node Name                    | Node Type                   | Functional Role                      | Input Node(s)                         | Output Node(s)         | Sticky Note                                                                                               |
|------------------------------|-----------------------------|------------------------------------|-------------------------------------|-----------------------|----------------------------------------------------------------------------------------------------------|
| Change Binary MimeType/Extension | Execute Workflow Trigger    | Workflow entry trigger              | None                                | SET OUTPUT FILE NAME   |                                                                                                          |
| SET OUTPUT FILE NAME          | Set                         | Define output file name and binary key | Change Binary MimeType/Extension    | Extract from File      | ### ⚙️ CONFIGURE HERE ⚙️ This is the main node you need to edit. 1. Set `output_file_name` (e.g., audio.mp3). 2. Do not edit `binary_key`. |
| Extract from File             | ExtractFromFile              | Convert binary to Base64 property   | SET OUTPUT FILE NAME                 | Change Binary Data Type | ### ⚙️ Core Logic (No Edit Needed) These nodes perform the conversion: Extract from File converts binary to Base64. |
| Change Binary Data Type       | Code                        | Rebuild binary with new file name  | Extract from File                    | None                  | ### ⚙️ Core Logic (No Edit Needed) Code node reconstructs binary with new extension and MIME type.          |
| Sticky Note                  | Sticky Note                 | Instruction for SET OUTPUT FILE NAME | None                                | None                  | See SET OUTPUT FILE NAME sticky note content.                                                             |
| Sticky Note1                 | Sticky Note                 | Instruction for core logic nodes   | None                                | None                  | See Extract from File and Change Binary Data Type sticky note content.                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add an **Execute Workflow Trigger** node.  
   - Set `Input Source` to `passthrough` so incoming data flows unaltered.  
   - Name it **Change Binary MimeType/Extension**.  

2. **Create the Set Node for Output File Name**  
   - Add a **Set** node and name it **SET OUTPUT FILE NAME**.  
   - Add two string fields:  
     - `binary_key` with value set by expression: `={{ Object.keys($binary).first() }}` (this dynamically selects the first binary property name).  
     - `output_file_name` with a placeholder string value like `"SET YOUR OUTPUT FILE NAME AND EXTENSION !! (ex: .mp3 or .png)"`.  
   - Enable `Include Other Fields` to pass all other incoming data through.  
   - Connect **Change Binary MimeType/Extension** node’s output to this node’s input.

3. **Create Extract from File Node**  
   - Add an **ExtractFromFile** node named **Extract from File**.  
   - Set `Operation` to `binaryToPropery`.  
   - For `Binary Property Name`, use expression: `={{ Object.keys($binary).first() }}` to dynamically refer to the binary data.  
   - Connect **SET OUTPUT FILE NAME** node’s output to this node’s input.

4. **Create the Code Node for Binary Transformation**  
   - Add a **Code** node named **Change Binary Data Type**.  
   - Use the following JavaScript code (adapted for n8n environment):  
     ```javascript
     const myBuffer = Buffer.from($json.data, 'base64');

     $input.item.binary = {
       [$('SET OUTPUT FILE NAME').last().json.binary_key]: await this.helpers.prepareBinaryData(myBuffer, $('SET OUTPUT FILE NAME').last().json.output_file_name)
     };

     return $input.item;
     ```  
   - This code decodes the Base64 string back to binary buffer and assigns it a new file name and extension from the set node.  
   - Connect **Extract from File** node’s output to this node.

5. **Add Instructional Sticky Notes (Optional)**  
   - Add a **Sticky Note** near the **SET OUTPUT FILE NAME** node with instructions to edit the `output_file_name` and not to change `binary_key`.  
   - Add another **Sticky Note** near the **Extract from File** and **Change Binary Data Type** nodes explaining the core logic and no edit is needed in these nodes.

6. **Credentials Setup**  
   - No credentials are needed for this workflow as it operates purely on binary data manipulation within n8n.

7. **Final Connections and Testing**  
   - Verify the connection chain:  
     `Change Binary MimeType/Extension` → `SET OUTPUT FILE NAME` → `Extract from File` → `Change Binary Data Type`.  
   - Test by triggering the workflow with a binary file input.  
   - Change `output_file_name` in the Set node to the desired output file name and extension (e.g., `audio.mp3`, `image.png`).  
   - Confirm that downstream nodes or outputs treat the file with the new MIME type and extension.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is specialized for changing MIME types/extensions by reassigning the binary file's name and type. | Useful when file format corrections are needed without altering file content.                   |
| The helper function `prepareBinaryData` from n8n is vital for correctly packaging binary data with new metadata. | n8n documentation on handling binary data: https://docs.n8n.io/nodes/expressions.html#binary     |
| To successfully use this workflow, always ensure the incoming binary data exists and `output_file_name` is set. | Incorrect or missing config can cause silent failures or loss of data integrity.                |

---

**Disclaimer:** The content above is generated exclusively from an n8n workflow JSON export. It respects all content policies and contains no illegal or offensive material. All data processed is legal and publicly accessible.