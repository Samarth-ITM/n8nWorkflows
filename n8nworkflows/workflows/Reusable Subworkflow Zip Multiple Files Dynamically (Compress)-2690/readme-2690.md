Reusable Subworkflow Zip Multiple Files Dynamically (Compress)

https://n8nworkflows.xyz/workflows/reusable-subworkflow-zip-multiple-files-dynamically--compress--2690


# Reusable Subworkflow Zip Multiple Files Dynamically (Compress)

### 1. Workflow Overview

This workflow is a reusable **Subworkflow** designed to dynamically compress multiple binary files into a single ZIP archive. It is intended to be called from other workflows, enabling modular and consistent file archiving without duplicating compression logic.

**Target Use Cases:**  
- Automating file archiving tasks in business processes  
- Developers managing multiple files programmatically  
- Any automation requiring bundling of diverse file types (images, PDFs, spreadsheets, etc.) into a ZIP file  

**Logical Blocks:**  
- **1.1 Input Reception:** Receives multiple binary files dynamically via the Subworkflow trigger.  
- **1.2 Binary Data Preparation:** Aggregates and organizes incoming binary files into a format suitable for compression.  
- **1.3 Compression:** Compresses the aggregated binary files into a single ZIP archive with a timestamped filename.  
- **1.4 Output Preparation:** Cleans and formats the output ZIP file metadata for downstream use.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the Subworkflow execution and receives multiple binary files as input from the parent workflow.

- **Nodes Involved:**  
  - Execute Workflow Trigger

- **Node Details:**  
  - **Execute Workflow Trigger**  
    - *Type:* Trigger node for Subworkflow execution  
    - *Configuration:* No parameters; acts as entry point for incoming data  
    - *Input/Output:* No input connections; outputs all incoming files to next node  
    - *Edge Cases:* Failure if no input files are provided; ensure parent workflow passes binary data correctly  
    - *Version:* Compatible with n8n v1+  
    - *Sub-workflow:* This node is the entry point of this Subworkflow  

#### 1.2 Binary Data Preparation

- **Overview:**  
  This block collects all incoming binary files, assigns them unique keys, and prepares a list of these keys for the compression node.

- **Nodes Involved:**  
  - Code Magic

- **Node Details:**  
  - **Code Magic**  
    - *Type:* Code node (JavaScript)  
    - *Configuration:*  
      - Iterates over all input items, extracts their binary data under the property `data`, and dynamically assigns keys like `data_0`, `data_1`, etc.  
      - Creates a comma-separated string of these keys in `binary_keys` JSON property.  
      - Returns one output item containing all binaries and the `binary_keys` string.  
    - *Key Expressions:*  
      - Uses `$input.all()` to access all input items  
      - Constructs `binaries` object and `binary_keys` array dynamically  
    - *Input/Output:*  
      - Input: Multiple binary files from Execute Workflow Trigger  
      - Output: Single item with combined binaries and `binary_keys` string  
    - *Edge Cases:*  
      - Fails if input items lack binary data under `data` property  
      - Expression errors if input structure changes  
    - *Version:* Uses n8n v2 code node syntax  
    - *Sub-workflow:* Internal processing step  

#### 1.3 Compression

- **Overview:**  
  Compresses the dynamically collected binary files into a ZIP archive. The ZIP filename includes the current date and time.

- **Nodes Involved:**  
  - Compression

- **Node Details:**  
  - **Compression**  
    - *Type:* Compression node  
    - *Configuration:*  
      - Operation: `compress`  
      - Binary Property Name: Uses expression `{{$json.binary_keys}}` to specify which binary keys to compress  
      - File Name: Uses expression `=data{{$now.format('yyyy-MM-dd-tt')}}.zip` to generate a timestamped ZIP filename (e.g., data2024-06-15-AM.zip)  
    - *Input/Output:*  
      - Input: Single item with multiple binaries and `binary_keys` string from Code Magic  
      - Output: Single binary ZIP file under property `data`  
    - *Edge Cases:*  
      - Compression failure if binary keys are invalid or missing  
      - Filename expression errors if `$now` is not available or misformatted  
    - *Version:* Requires n8n v1.1+ for compression node with dynamic binary keys  
    - *Sub-workflow:* Core compression step  

#### 1.4 Output Preparation

- **Overview:**  
  Cleans the output ZIP file metadata by removing spaces from the filename and ensures the output is ready for downstream consumption.

- **Nodes Involved:**  
  - Prepare Output

- **Node Details:**  
  - **Prepare Output**  
    - *Type:* Set node  
    - *Configuration:*  
      - Assigns a new JSON property `fileName` by removing all spaces from the binary file’s `fileName` property (`{{$binary.data.fileName.replaceAll(" ","")}}`)  
      - Includes all other fields unchanged (`includeOtherFields: true`)  
    - *Input/Output:*  
      - Input: ZIP file binary from Compression node  
      - Output: ZIP file binary with cleaned filename property  
    - *Edge Cases:*  
      - Fails if binary data or filename is missing  
      - Expression failure if `fileName` property is undefined  
    - *Version:* Compatible with n8n v3+  
    - *Sub-workflow:* Final output formatting  

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                  |
|-------------------------|--------------------------------|-------------------------------|--------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------|
| Execute Workflow Trigger | Execute Workflow Trigger        | Subworkflow entry point        | —                        | Code Magic              |                                                                                                              |
| Code Magic              | Code                           | Prepare binaries for compression | Execute Workflow Trigger | Compression             |                                                                                                              |
| Compression             | Compression                    | Compress multiple binaries into ZIP | Code Magic               | Prepare Output          |                                                                                                              |
| Prepare Output          | Set                            | Clean and format output filename | Compression              | —                       |                                                                                                              |
| Sticky Note             | Sticky Note                    | Documentation note             | —                        | —                       | ## About Use me as modular workflow. Instead of building me fixed in your workflow. Just call me when you need me. Input can be multiple files - images, pdfs, xlsx, csv.... Output Single zip file |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new Subworkflow** in n8n and name it "Zip multiple files".

2. **Add an "Execute Workflow Trigger" node**  
   - This node will serve as the entry point for the Subworkflow.  
   - No special configuration needed.  
   - Position it at the start (e.g., coordinates [0,0]).

3. **Add a "Code" node named "Code Magic"**  
   - Connect the output of "Execute Workflow Trigger" to this node.  
   - Configure the code as follows (JavaScript):  
     ```javascript
     let binaries = {}, binary_keys = [];
     
     for (const [index, inputItem] of Object.entries($input.all())) {
       binaries[`data_${index}`] = inputItem.binary.data;
       binary_keys.push(`data_${index}`);
     }
     
     return [{
         json: {
             binary_keys: binary_keys.join(',')
         },
         binary: binaries
     }];
     ```  
   - This code dynamically collects all incoming binary files and prepares them for compression.

4. **Add a "Compression" node named "Compression"**  
   - Connect the output of "Code Magic" to this node.  
   - Set the operation to `compress`.  
   - Set the **Binary Property Name** to the expression: `{{$json.binary_keys}}`  
   - Set the **File Name** to the expression: `=data{{$now.format('yyyy-MM-dd-tt')}}.zip`  
     - This generates a ZIP filename with the current date and AM/PM indicator.  
   - Ensure the node version supports dynamic binary keys (n8n v1.1+).

5. **Add a "Set" node named "Prepare Output"**  
   - Connect the output of "Compression" to this node.  
   - Configure it to:  
     - Assign a new string field `fileName` with the expression: `{{$binary.data.fileName.replaceAll(" ","")}}`  
     - Enable "Keep Only Set" to false (or include other fields) so the binary data passes through unchanged.

6. **Connect the output of "Prepare Output" as the final output of the Subworkflow.**

7. **Add a "Sticky Note" node** (optional, for documentation)  
   - Content:  
     ```
     ## About
     Use me as modular workflow. Instead of building me fixed in your workflow. Just call me when you need me.

     ## Input
     Input can be multiple files 
     - images
     - pdfs
     - xlsx, csv....

     ## Output
     Single zip file
     ```
   - Position it near the start for visibility.

8. **Save and activate the Subworkflow.**

9. **Usage in parent workflows:**  
   - Pass multiple binary files as input to this Subworkflow node.  
   - The Subworkflow will output a single ZIP file with a timestamped name.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Code source and inspiration by Tom (mutedjam)                                                             | https://n8n.io/creators/mutedjam/                                                                 |
| This Subworkflow is designed for modular reuse to avoid duplicating ZIP logic in multiple workflows        |                                                                                                   |
| Filename timestamp format uses `yyyy-MM-dd-tt` where `tt` is AM/PM indicator                                | Adjust the format string in Compression node if different naming is desired                         |
| Input binary property assumed to be named `data` in incoming files                                         | Parent workflows must ensure binary files are under `binary.data` property for compatibility       |
| Compatible with n8n versions supporting Compression node v1.1+ and Code node v2+                            |                                                                                                   |

---

This documentation provides a complete, detailed reference for understanding, reproducing, and modifying the "Zip multiple files" Subworkflow in n8n. It ensures clarity on node roles, configurations, and integration points for both human users and automation agents.