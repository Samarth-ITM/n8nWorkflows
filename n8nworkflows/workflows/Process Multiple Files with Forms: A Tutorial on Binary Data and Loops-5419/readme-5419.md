Process Multiple Files with Forms: A Tutorial on Binary Data and Loops

https://n8nworkflows.xyz/workflows/process-multiple-files-with-forms--a-tutorial-on-binary-data-and-loops-5419


# Process Multiple Files with Forms: A Tutorial on Binary Data and Loops

### 1. Workflow Overview

This workflow demonstrates how to process multiple files uploaded simultaneously via an n8n form, focusing on handling binary data and looping through files one-by-one. It is designed for use cases where a user uploads multiple JSON files in a single form submission, but the workflow needs to process each file individually rather than as a bulk array. The workflow shows how to:

- Receive multiple files via a form trigger
- Split the aggregated file array into individual file events, preserving binary data
- Loop through each file sequentially
- Save each file to disk with a dynamic filename
- Control downstream processing to proceed only once after all files are handled

Logical blocks include:

- **1.1 Input Reception (Form Trigger):** Receives multiple files via a form submission.
- **1.2 Splitting Files:** Splits the incoming array of binary files into individual events.
- **1.3 Looping and File Processing:** Loops through each file to save it individually.
- **1.4 Flow Control:** Ensures downstream nodes run only once after the loop completes.
- **1.5 Documentation and Guidance:** Sticky notes providing contextual explanations.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception (Form Trigger)

- **Overview:** This block collects multiple JSON files uploaded via a form. It acts as the workflow's entry point, capturing all files in a single event.
- **Nodes Involved:**  
  - Form - Load Multiple Files

- **Node Details:**

  - **Form - Load Multiple Files**  
    - **Type:** Form Trigger  
    - **Role:** Entry point to receive user-submitted files via a web form.  
    - **Configuration:**  
      - Form Title: "LOAD MULTIPLE FILES"  
      - One form field of type "file" labeled "files"  
      - Accepts only files with extension `.json`  
      - Field is required  
      - Webhook ID assigned for external triggering  
    - **Key expressions/variables:** None specific at this node.  
    - **Input/Output:** No input; outputs a single event containing an array of files under the field `files`.  
    - **Edge cases:**  
      - User submits no files → blocked by required field setting.  
      - Large file uploads or multiple large files may cause timeouts or memory constraints.  
      - Supported file types restricted to `.json`.  
    - **Sub-workflows:** None  

#### 1.2 Splitting Files

- **Overview:** This block splits the aggregated array of multiple files into individual events, each carrying one file with its binary data intact, enabling subsequent per-file processing.
- **Nodes Involved:**  
  - Split Out Files

- **Node Details:**

  - **Split Out Files**  
    - **Type:** Split Out  
    - **Role:** Converts one event containing an array of files into multiple events, one per file.  
    - **Configuration:**  
      - Field to Split Out: `files` (the array holding uploaded files)  
      - Include Binary Data: enabled (critical to preserve file contents)  
      - Destination Field Name: `files` (each output event will have a single file in this field)  
    - **Input:** The single event from the form trigger containing all files.  
    - **Output:** Multiple events, each with one file and its binary data under the `files` field.  
    - **Edge cases:**  
      - Failure to preserve binary data would cause file corruption downstream.  
      - If the `files` array is empty or malformed, no events will be emitted.  
    - **Sub-workflows:** None  

#### 1.3 Looping and File Processing

- **Overview:** This block loops over each individual file event to process files sequentially. Each file is saved to disk with a dynamically generated filename based on the file metadata.
- **Nodes Involved:**  
  - Loop Over Items  
  - Save Each File

- **Node Details:**

  - **Loop Over Items**  
    - **Type:** Split In Batches (Loop)  
    - **Role:** Processes each item (file) one-by-one rather than all at once.  
    - **Configuration:**  
      - Default batching options (no special options set)  
      - Receives multiple file events from Split Out Files  
    - **Input:** Multiple events from Split Out Files, each with one file in `files`.  
    - **Output:** For each loop iteration, sends one file downstream.  
    - **Key expressions:**  
      - Uses `{{$runIndex}}` to track loop iteration index for dynamic naming downstream.  
    - **Edge cases:**  
      - Timeout or delay if processing many files.  
      - If an iteration fails, subsequent iterations may be skipped depending on error settings.  
    - **Sub-workflows:** None  

  - **Save Each File**  
    - **Type:** Read/Write File  
    - **Role:** Writes each file to disk.  
    - **Configuration:**  
      - Operation: "write"  
      - Filename: dynamically set as `out_{{ $json.files.filename }}`, i.e., prefix `out_` plus original filename.  
      - Append: false (overwrite if file exists)  
      - Data Source: Reads the binary file from property named dynamically as `files_{{$runIndex}}`  
    - **Input:** Receives one file per loop iteration from Loop Over Items.  
    - **Output:** None (writes file to disk).  
    - **Key expressions:**  
      - Filename expression based on input file's metadata.  
      - Binary data accessed via dynamic property `files_{{$runIndex}}`.  
    - **Edge cases:**  
      - Write permission errors on disk or invalid path.  
      - Missing binary data if loop index does not match binary field names.  
    - **Sub-workflows:** None  

#### 1.4 Flow Control

- **Overview:** This block ensures that after looping through all files, a single event continues downstream, preventing multiple parallel flows from triggering.
- **Nodes Involved:**  
  - Continue Once

- **Node Details:**

  - **Continue Once**  
    - **Type:** No Operation (NoOp)  
    - **Role:** Acts as a gate that lets only one event pass downstream regardless of the number of loop iterations.  
    - **Configuration:**  
      - Set to "Execute Once" — only the first incoming event proceeds; others are ignored.  
    - **Input:** Receives events from Loop Over Items (one per iteration).  
    - **Output:** Emits a single event downstream after the first iteration completes.  
    - **Edge cases:**  
      - Subsequent events after the first are dropped silently; ensure downstream logic expects a single event.  
    - **Sub-workflows:** None  

#### 1.5 Documentation and Guidance (Sticky Notes)

- **Overview:** Several sticky notes provide learning context and explanations.
- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note4

- **Node Details:**

  - **Sticky Note**  
    - Content: Overview of learning about loops, multiple binary files, and use of `{{$runIndex}}`  
  - **Sticky Note1**  
    - Content: Explanation that form submission sends all files in one event, requiring splitting to process individually  
  - **Sticky Note2**  
    - Content: Explains the Split Out node’s role in firing one event per file, emphasizing preserving binary data  
  - **Sticky Note3**  
    - Content: Discusses usage of loop and `{{$runIndex}}` to reference binary data fields passed downstream  
  - **Sticky Note4**  
    - Content: Explains the NoOp node configured to run once to ensure single downstream event regardless of file count  

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                                   | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                                   |
|---------------------|---------------------|-------------------------------------------------|---------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------|
| Form - Load Multiple Files | Form Trigger         | Receive multiple JSON files via form submission | None                      | Split Out Files          | Use a Form to load multiple files. This is a case where n8n will NOT fire an event per file loaded, but instead a single event with all files nested into 1 array. However, we want to process the files one at a time... |
| Split Out Files      | Split Out           | Split array of files into individual file events | Form - Load Multiple Files | Loop Over Items          | Split Out will fix this, firing one event for each file. *We must make sure to pass binary data along.*        |
| Loop Over Items      | Split In Batches (Loop) | Loop over each file individually                  | Split Out Files            | Continue Once, Save Each File | Now we can use a Loop to handle each file, one at a time. With this setup, the special {{$runIndex}} can be used to get our loop counter. We use it to get the correct binary file, which has been inconveniently passed downstream as files_0, files_1, files_2... etc. |
| Save Each File       | Read/Write File     | Save each file to disk with dynamic filename      | Loop Over Items            | None                    |                                                                                                              |
| Continue Once        | No Operation (NoOp) | Allow a single event to continue downstream after loop | Loop Over Items            | None                    | We typically only want to proceed downstream with a single event, so this NOP set to "run once" will let us do that regardless of the file count. |
| Sticky Note          | Sticky Note         | Learning context on loops and binary files        | None                      | None                    | ## Learning n8n\n### Loops, Multiple Binary Files, {{$runIndex}}\n\nLoops are rarely needed in n8n, but here is one example of how to effectively use one to process multiple binary files. |
| Sticky Note1         | Sticky Note         | Explains form behavior with multiple files        | None                      | None                    | Use a Form to load multiple files. \n\nThis is a case where n8n will NOT fire an event per file loaded, but instead a single event with all files nested into 1 array. \nHowever, we want to process the files one at a time... |
| Sticky Note2         | Sticky Note         | Explains Split Out role and binary preservation    | None                      | None                    | Split Out will fix this, firing one event for each file.\n\n*We must make sure to pass binary data along.*    |
| Sticky Note3         | Sticky Note         | Explains loop usage and `{{$runIndex}}` usage      | None                      | None                    | Now we can use a Loop to handle each file, one at a time.\nWith this setup, the special {{$runIndex}} can be used to get our loop counter.\n\nWe use it to get the correct binary file, which has been inconveniently passed downstream as files_0, files_1, files_2... etc. |
| Sticky Note4         | Sticky Note         | Explains NoOp run once to allow single downstream event | None                      | None                    | We typically only want to proceed downstream with a single event, so this NOP set to "run once" will let us do that regardless of the file count. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to recreate the workflow in n8n:

1. **Create Form Trigger Node**  
   - Name: "Form - Load Multiple Files"  
   - Type: Form Trigger  
   - Configure form:  
     - Title: "LOAD MULTIPLE FILES"  
     - Add one form field:  
       - Type: File  
       - Label: "files"  
       - Required: Yes  
       - Accept file types: `*.json`  
   - Save and note webhook URL for triggering externally.

2. **Create Split Out Node**  
   - Name: "Split Out Files"  
   - Type: Split Out  
   - Connect input from "Form - Load Multiple Files" node output.  
   - Configure:  
     - Field to Split Out: `files`  
     - Include Binary Data: Enabled (checked)  
     - Destination Field Name: `files`

3. **Create Split In Batches (Loop) Node**  
   - Name: "Loop Over Items"  
   - Type: Split In Batches  
   - Connect input from "Split Out Files" output.  
   - Leave default batching options (empty).  
   - This node will emit one file event per batch iteration.

4. **Create Read/Write File Node**  
   - Name: "Save Each File"  
   - Type: Read/Write File  
   - Connect input from second output of "Loop Over Items" (index 1)  
   - Configure:  
     - Operation: Write  
     - File Name: Use expression: `out_{{ $json.files.filename }}`  
     - Append: false (overwrite if exists)  
     - Data Property Name: Use expression: `files_{{$runIndex}}`  
       (This dynamically references the binary file of the current iteration)  

5. **Create No Operation (NoOp) Node**  
   - Name: "Continue Once"  
   - Type: No Operation  
   - Connect input from first output of "Loop Over Items" (index 0)  
   - Configure:  
     - Enable "Execute Once" option (only allow one event to proceed).  

6. **Add Sticky Notes** (Optional but recommended for documentation)  
   - Place sticky notes near the nodes to explain the concepts:  
     - Explain form submission behavior with multiple files.  
     - Explain necessity of Split Out node with binary preservation.  
     - Explain loop usage and `{{$runIndex}}` for dynamic binary access.  
     - Explain NoOp node usage to control downstream event count.  

7. **Credential Setup**  
   - No external credentials are needed for this workflow since no external API calls are involved.  
   - Ensure the n8n instance has proper file system permissions for the Read/Write File node.  

8. **Testing the Workflow**  
   - Activate the workflow.  
   - Use the form webhook URL to upload multiple `.json` files in one submission.  
   - Confirm that files are saved individually with the prefix `out_` in the configured directory.  
   - Verify that only one downstream event proceeds past the loop (if extended downstream).  

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                        |
|--------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| Loops in n8n are rarely needed but are useful for processing multiple binary files sequentially.                         | Sticky Note content explains this concept in the workflow.            |
| Form submissions with multiple files send a single event with files nested in an array, not multiple events.             | Sticky Note1 clarifies this important behavior of n8n form triggers.  |
| The Split Out node must preserve binary data explicitly to avoid file corruption or data loss when splitting arrays.     | Sticky Note2 highlights this critical requirement.                    |
| The special variable `{{$runIndex}}` is used inside loops to dynamically access the current loop iteration index.       | Sticky Note3 provides context on how to use this variable effectively.|
| The No Operation node configured to "Execute Once" is a pattern to ensure only one event continues downstream after loops.| Sticky Note4 details this flow control technique.                      |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.