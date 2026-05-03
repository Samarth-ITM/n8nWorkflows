Generate AI Media with ComfyUI: Images, Video, 3D & Audio Bridge

https://n8nworkflows.xyz/workflows/generate-ai-media-with-comfyui--images--video--3d---audio-bridge-4468


# Generate AI Media with ComfyUI: Images, Video, 3D & Audio Bridge

---
### 1. Workflow Overview

This workflow, titled **Generate AI Media with ComfyUI: Images, Video, 3D & Audio Bridge**, orchestrates AI-driven media generation using ComfyUI workflows exported in API format. It supports both **Text-to-Image (Txt2Img)** and **Image-to-Image (Img2Img)** generation modes, handling local ComfyUI execution, managing API calls, and processing outputs including error handling and logging.

The workflow is structured into logical blocks reflecting its lifecycle:

- **1.1 Trigger & Initialization:** Entry points for manual or external workflow execution, initial configuration setup.
- **1.2 Mode Selection & Input Preparation:** Decision logic to branch between Img2Img and Txt2Img modes, preparing workflow inputs accordingly.
- **1.3 ComfyUI Workflow Handling:** Reading, extracting, and editing ComfyUI workflows exported in API format from disk.
- **1.4 Local ComfyUI Execution:** Executing the prepared ComfyUI workflow locally and awaiting results.
- **1.5 Result Retrieval & Post-Processing:** Retrieving generated images, converting to files, editing fields, and optionally merging or logging results.
- **1.6 Error Handling & Logging:** Managing failures at various stages with custom code nodes, retry logic, and error logging.
- **1.7 Auxiliary / Testing:** Includes nodes for manual trigger and test workflow execution to facilitate debugging and development.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Initialization

**Overview:**  
This block manages workflow invocation either by manual trigger or by being called from another workflow, and sets up initial connection configuration and client identification.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Client ID  
- Connection Config Duplicate  
- Wait For Test Type Select  
- If Img2Img  
- When Executed by Another Workflow  
- Connection Config  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual testing  
  - Connections: Outputs to Client ID  
  - Edge Cases: None specific, manual start only

- **Client ID**  
  - Type: Crypto  
  - Role: Generates or processes a client identifier (likely for session or request correlation)  
  - Input: From manual trigger  
  - Output: To Connection Config Duplicate  
  - Edge Cases: Crypto failure or missing data - low risk

- **Connection Config Duplicate**  
  - Type: Set  
  - Role: Stores or duplicates connection parameters for test mode  
  - Input: Client ID  
  - Output: Wait For Test Type Select  
  - Edge Cases: Misconfiguration of credentials can cause downstream failures

- **Wait For Test Type Select**  
  - Type: Wait  
  - Role: Waits for external input or selection to determine mode (Img2Img or Txt2Img)  
  - Input: Connection Config Duplicate  
  - Output: If Img2Img  
  - Edge Cases: Timeout if no input received

- **If Img2Img**  
  - Type: If  
  - Role: Branches workflow based on whether image-to-image mode is selected  
  - Input: Wait For Test Type Select  
  - Outputs: Wait1 (true branch), Read API Exported Txt2Img ComfyUI Workflow from Disk (false branch)  
  - Edge Cases: Expression errors in condition evaluation

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point for external workflow calls  
  - Output: Connection Config  
  - Edge Cases: Missing data from external caller

- **Connection Config**  
  - Type: Set  
  - Role: Sets connection parameters for external execution  
  - Output: Trigger LOCAL Workflow  
  - Edge Cases: Credential misconfiguration

---

#### 1.2 Mode Selection & Input Preparation

**Overview:**  
This block reads the appropriate ComfyUI API-exported workflow from disk depending on the selected mode, extracts the workflow content, and prepares the input parameters such as prompts and seeds.

**Nodes Involved:**  
- Read API Exported Img2Img ComfyUI Workflow from Disk  
- Extract Img2Img Comfy Workflow  
- Edit Img2Img Inputs  
- Read API Exported Txt2Img ComfyUI Workflow from Disk  
- Extract Txt2Img Comfy Workflow  
- Edit Txt2Img Inputs  
- Fallback Img2Img SDXL Turbo  
- Fallback Txt2Img SDXL Turbo  

**Node Details:**

- **Read API Exported Img2Img ComfyUI Workflow from Disk**  
  - Type: Read/Write File  
  - Role: Reads Img2Img ComfyUI workflow JSON exported via ComfyUI API export  
  - Notes: User must export workflow from ComfyUI via Workflow > Export(API)  
  - Outputs: Extract Img2Img Comfy Workflow (main), Fallback Img2Img SDXL Turbo (error or alternate path)  
  - Edge Cases: File not found, parsing error

- **Extract Img2Img Comfy Workflow**  
  - Type: Extract From File  
  - Role: Parses the file content to extract the workflow JSON  
  - Input: File content from previous node  
  - Output: Edit Img2Img Inputs  
  - Edge Cases: Extraction failure if file format unexpected

- **Edit Img2Img Inputs**  
  - Type: Set  
  - Role: Prepares inputs such as positive & negative prompts, seeds for Img2Img workflow  
  - Input: Extracted workflow data  
  - Output: 🎨🏠 Run local ComfyUI workflow  
  - Edge Cases: Missing or invalid input parameters

- **Fallback Img2Img SDXL Turbo**  
  - Type: Set  
  - Role: Provides fallback default inputs for Img2Img if main workflow fails or missing  
  - Output: Edit Img2Img Inputs (alternative path)  

- **Read API Exported Txt2Img ComfyUI Workflow from Disk**  
  - Type: Read/Write File  
  - Role: Reads Txt2Img ComfyUI workflow JSON exported via ComfyUI API export  
  - Notes: Same export instructions as Img2Img  
  - Outputs: Extract Txt2Img Comfy Workflow (main), Fallback Txt2Img SDXL Turbo (alternate)  
  - Edge Cases: File missing or unreadable

- **Extract Txt2Img Comfy Workflow**  
  - Type: Extract From File  
  - Role: Parses Txt2Img workflow JSON from file content  
  - Output: Edit Txt2Img Inputs  
  - Edge Cases: Extraction/parsing errors

- **Edit Txt2Img Inputs**  
  - Type: Set  
  - Role: Prepares prompt, seed, and other inputs for Txt2Img workflow execution  
  - Output: 🎨🏠 Run local ComfyUI workflow  
  - Edge Cases: Input validation failures

- **Fallback Txt2Img SDXL Turbo**  
  - Type: Set  
  - Role: Provides fallback default inputs for Txt2Img if main workflow unavailable  
  - Output: Edit Txt2Img Inputs (alternative path)

---

#### 1.3 Local ComfyUI Execution

**Overview:**  
Executes the prepared ComfyUI workflow locally via the Execute Workflow node and manages waiting for results and error handling.

**Nodes Involved:**  
- 🎨🏠 Run local ComfyUI workflow  
- Wait1  
- Upload Attachments LOCAL  
- Fail Upload  
- Link This To Error Handling  

**Node Details:**

- **🎨🏠 Run local ComfyUI workflow**  
  - Type: Execute Workflow  
  - Role: Executes the edited ComfyUI workflow (either Img2Img or Txt2Img) locally  
  - Input: Edited inputs from previous block  
  - Output: Upload Attachments LOCAL  
  - Edge Cases: Execution failure, connection issues, timeouts

- **Wait1**  
  - Type: Wait  
  - Role: Pauses to allow Img2Img workflow execution or resource availability  
  - Output: Upload Attachments LOCAL  
  - Edge Cases: Timeout if execution takes too long

- **Upload Attachments LOCAL**  
  - Type: HTTP Request  
  - Role: Uploads generated media or attachments to a local server or storage endpoint  
  - On error: Continues to Fail Upload  
  - Output: Read API Exported Img2Img ComfyUI Workflow from Disk (loop for img2img), or Fail Upload  
  - Edge Cases: Network errors, upload failures

- **Fail Upload**  
  - Type: Code  
  - Role: Custom error handling logic for upload failures  
  - Output: Link This To Error Handling  
  - Edge Cases: Code exceptions

- **Link This To Error Handling**  
  - Type: No Operation (NoOp)  
  - Role: Placeholder node to unify error flow for logging or alerts  
  - Edge Cases: None

---

#### 1.4 Result Retrieval & Post-Processing

**Overview:**  
Handles retrieval of generated images, conditional branching, conversion to file format, editing metadata, merging results, and logging.

**Nodes Involved:**  
- Trigger LOCAL Workflow  
- HTTP Request  
- If  
- Get Generated Image  
- Return The Output JSON Instead  
- Edit Fields  
- Convert to File  
- Write to error log  
- Merge  
- Fail Trigger  
- Fail Get History  

**Node Details:**

- **Trigger LOCAL Workflow**  
  - Type: HTTP Request  
  - Role: Initiates a local workflow call to generate media  
  - On error: Continue to Fail Trigger  
  - Output: HTTP Request, Fail Trigger  
  - Edge Cases: Network failure, authentication errors

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Requests status or result of generation  
  - On error: Continue to Fail Get History  
  - Output: If, Fail Get History  
  - Edge Cases: Timeout, invalid response

- **If**  
  - Type: If  
  - Role: Conditional check on response to decide next step, e.g., if generation completed  
  - Output: Get Generated Image (true), Wait (false)  
  - Edge Cases: Expression evaluation failure

- **Get Generated Image**  
  - Type: HTTP Request  
  - Role: Fetches the generated image file or data  
  - Output: None (main), Return The Output JSON Instead (error branch)  
  - Edge Cases: File not found, network errors

- **Return The Output JSON Instead**  
  - Type: Set  
  - Role: Prepares the raw JSON output instead of image files if needed  
  - Output: None  
  - Edge Cases: Data formatting issues

- **Edit Fields**  
  - Type: Set  
  - Role: Cleans or modifies data fields in the output before saving or forwarding  
  - Output: Convert to File  
  - Edge Cases: Expression errors

- **Convert to File**  
  - Type: Convert to File  
  - Role: Converts image or media data into file format suitable for storage or transmission  
  - Output: Write to error log  
  - Edge Cases: Conversion failures

- **Write to error log**  
  - Type: Read/Write File  
  - Role: Logs errors or output data to disk for auditing or debugging  
  - Edge Cases: Disk write errors

- **Merge**  
  - Type: Merge  
  - Role: Combines multiple data streams or results, e.g., merging success and error outputs  
  - Output: Edit Fields  
  - Edge Cases: Data mismatch or merge conflicts

- **Fail Trigger**  
  - Type: Code  
  - Role: Handles failure from Trigger LOCAL Workflow node  
  - Output: Merge (error path)  
  - Edge Cases: Code exceptions

- **Fail Get History**  
  - Type: Code  
  - Role: Handles failure from HTTP Request node fetching status/history  
  - Output: Merge  
  - Edge Cases: Code exceptions

---

#### 1.5 Error Handling & Logging

**Overview:**  
Dedicated nodes for capturing failures at various stages and consolidating error messages for logging or alerting.

**Nodes Involved:**  
- Fail Trigger  
- Fail Get History  
- Fail Upload  
- Write to error log  
- Link This To Error Handling  

**Node Details:**

- **Fail Trigger**  
  - See above under Result Retrieval

- **Fail Get History**  
  - See above under Result Retrieval

- **Fail Upload**  
  - See above under Local ComfyUI Execution

- **Write to error log**  
  - See above under Result Retrieval

- **Link This To Error Handling**  
  - See above under Local ComfyUI Execution

---

#### 1.6 Auxiliary / Testing

**Overview:**  
Nodes to facilitate manual testing and debugging with triggers and client ID generation.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Client ID  
- Sticky Notes (various)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - See above under Trigger & Initialization

- **Client ID**  
  - See above under Trigger & Initialization

- **Sticky Notes**  
  - Type: Sticky Note  
  - Role: Visual comments for users; currently empty content  
  - Notes: Scattered throughout the workflow for annotation purposes

---

### 3. Summary Table

| Node Name                                   | Node Type                  | Functional Role                             | Input Node(s)                         | Output Node(s)                           | Sticky Note               |
|---------------------------------------------|----------------------------|--------------------------------------------|-------------------------------------|-----------------------------------------|---------------------------|
| When Executed by Another Workflow            | ExecuteWorkflowTrigger     | External workflow trigger                   | —                                   | Connection Config                       |                           |
| Connection Config                            | Set                        | Setup connection parameters                 | When Executed by Another Workflow   | Trigger LOCAL Workflow                   |                           |
| Trigger LOCAL Workflow                       | HTTP Request               | Initiate local ComfyUI workflow             | Connection Config                   | HTTP Request, Fail Trigger              |                           |
| Fail Trigger                                | Code                       | Handle trigger failure                       | Trigger LOCAL Workflow              | Merge                                  |                           |
| HTTP Request                                | HTTP Request               | Query generation status                      | Trigger LOCAL Workflow              | If, Fail Get History                    |                           |
| If                                          | If                         | Check if generation result ready            | HTTP Request                      | Get Generated Image, Wait               |                           |
| Wait                                        | Wait                       | Wait for generation completion               | If                                 | HTTP Request                           |                           |
| Get Generated Image                         | HTTP Request               | Retrieve generated image                     | If                                 | Return The Output JSON Instead (error) |                           |
| Return The Output JSON Instead              | Set                        | Output JSON instead of image                 | Get Generated Image                 | —                                     |                           |
| Edit Fields                                 | Set                        | Clean or modify output fields                | Merge                             | Convert to File                        |                           |
| Convert to File                             | ConvertToFile              | Convert data to file format                   | Edit Fields                      | Write to error log                     |                           |
| Write to error log                          | ReadWriteFile              | Log errors or outputs to file                 | Convert to File                  | —                                     |                           |
| Merge                                       | Merge                      | Merge success and error outputs              | Fail Trigger, Fail Get History     | Edit Fields                          |                           |
| Fail Get History                            | Code                       | Handle failure in getting generation status | HTTP Request                      | Merge                                |                           |
| Connection Config Duplicate                 | Set                        | Duplicate connection config for test         | Client ID                        | Wait For Test Type Select              |                           |
| Client ID                                   | Crypto                     | Generate or process client identifier        | When clicking ‘Test workflow’     | Connection Config Duplicate            |                           |
| When clicking ‘Test workflow’               | ManualTrigger              | Manual start trigger                          | —                               | Client ID                            |                           |
| Wait For Test Type Select                   | Wait                       | Wait for user selection of Img2Img or Txt2Img | Connection Config Duplicate     | If Img2Img                          |                           |
| If Img2Img                                  | If                         | Branch on Img2Img or Txt2Img mode             | Wait For Test Type Select         | Wait1, Read API Exported Txt2Img ComfyUI Workflow from Disk |                           |
| Wait1                                       | Wait                       | Wait for Img2Img processing                   | If Img2Img                      | Upload Attachments LOCAL              |                           |
| Upload Attachments LOCAL                    | HTTP Request               | Upload generated media attachments            | Wait1                          | Read API Exported Img2Img ComfyUI Workflow from Disk, Fail Upload |                           |
| Fail Upload                                | Code                       | Handle upload failure                          | Upload Attachments LOCAL          | Link This To Error Handling            |                           |
| Link This To Error Handling                 | NoOp                       | Consolidate error handling                    | Fail Upload                    | —                                     |                           |
| Read API Exported Img2Img ComfyUI Workflow from Disk | ReadWriteFile              | Read Img2Img workflow JSON from disk         | Upload Attachments LOCAL          | Extract Img2Img Comfy Workflow, Fallback Img2Img SDXL Turbo |                           |
| Extract Img2Img Comfy Workflow              | ExtractFromFile            | Extract Img2Img workflow JSON content         | Read API Exported Img2Img ComfyUI Workflow from Disk | Edit Img2Img Inputs               |                           |
| Edit Img2Img Inputs                         | Set                        | Prepare Img2Img inputs (prompts, seeds)       | Extract Img2Img Comfy Workflow    | 🎨🏠 Run local ComfyUI workflow        |                           |
| Fallback Img2Img SDXL Turbo                 | Set                        | Provide fallback Img2Img inputs                | Read API Exported Img2Img ComfyUI Workflow from Disk | Edit Img2Img Inputs               |                           |
| Read API Exported Txt2Img ComfyUI Workflow from Disk | ReadWriteFile              | Read Txt2Img workflow JSON from disk          | If Img2Img (false branch)         | Extract Txt2Img Comfy Workflow, Fallback Txt2Img SDXL Turbo |                           |
| Extract Txt2Img Comfy Workflow              | ExtractFromFile            | Extract Txt2Img workflow JSON content          | Read API Exported Txt2Img ComfyUI Workflow from Disk | Edit Txt2Img Inputs               |                           |
| Edit Txt2Img Inputs                         | Set                        | Prepare Txt2Img inputs (prompts, seeds)       | Extract Txt2Img Comfy Workflow    | 🎨🏠 Run local ComfyUI workflow        |                           |
| Fallback Txt2Img SDXL Turbo                 | Set                        | Provide fallback Txt2Img inputs                | Read API Exported Txt2Img ComfyUI Workflow from Disk | Edit Txt2Img Inputs               |                           |
| 🎨🏠 Run local ComfyUI workflow              | ExecuteWorkflow            | Run prepared ComfyUI workflow locally          | Edit Img2Img Inputs, Edit Txt2Img Inputs | Upload Attachments LOCAL          |                           |
| Sticky Note(s)                              | Sticky Note                | User annotations / comments                     | —                               | —                                     | Empty content in all notes |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`.
   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow`.

2. **Client ID Generation:**
   - Add a **Crypto** node named `Client ID` connected from the manual trigger.
   - Configure it to generate a unique client identifier or hash as needed.

3. **Connection Config Setup:**
   - Add two **Set** nodes:
     - `Connection Config Duplicate` connected from `Client ID`.
     - `Connection Config` connected from `When Executed by Another Workflow`.
   - Configure these to set all required credentials and parameters (e.g., API URLs, tokens, local server endpoints).

4. **Wait for Mode Selection:**
   - Add a **Wait** node named `Wait For Test Type Select` connected from `Connection Config Duplicate`.
   - Configure it with a webhook or timeout as required to receive mode selection input.

5. **Conditional Branching for Img2Img or Txt2Img:**
   - Add an **If** node named `If Img2Img` connected from `Wait For Test Type Select`.
   - Configure the condition to check if the mode is Img2Img (boolean or string condition).

6. **Txt2Img Workflow File Handling:**
   - Add a **Read/Write File** node named `Read API Exported Txt2Img ComfyUI Workflow from Disk` connected from the false output of `If Img2Img`.
   - Configure the file path to the exported Txt2Img workflow JSON.
   - Add an **Extract From File** node named `Extract Txt2Img Comfy Workflow` connected from the previous node.
   - Add a **Set** node named `Edit Txt2Img Inputs` connected from extraction.
   - Configure these nodes to read, parse, and prepare Txt2Img inputs (prompts, negative prompts, seeds).

7. **Img2Img Workflow File Handling:**
   - Add a **Wait** node named `Wait1` connected from the true output of `If Img2Img`.
   - Add a **Read/Write File** node named `Read API Exported Img2Img ComfyUI Workflow from Disk` connected from `Upload Attachments LOCAL` (see below).
   - Add an **Extract From File** node named `Extract Img2Img Comfy Workflow` connected from the previous node.
   - Add a **Set** node named `Edit Img2Img Inputs` connected from extraction.
   - Configure these nodes as with Txt2Img but for Img2Img.

8. **Fallback Input Preparation:**
   - For both Txt2Img and Img2Img, add **Set** nodes named `Fallback Txt2Img SDXL Turbo` and `Fallback Img2Img SDXL Turbo`.
   - Connect these from the Read/Write File nodes as alternate outputs.
   - Configure default or safe input parameters for fallback scenarios.

9. **Run Local ComfyUI Workflow:**
   - Add an **Execute Workflow** node named `🎨🏠 Run local ComfyUI workflow`.
   - Connect from `Edit Txt2Img Inputs` and `Edit Img2Img Inputs`.
   - Configure it to execute the ComfyUI workflow locally with the prepared inputs.

10. **Upload Attachments and Wait:**
    - Add a **Wait** node named `Wait1` connected from `If Img2Img`.
    - Add an **HTTP Request** node named `Upload Attachments LOCAL` connected from the execute workflow node and wait.
    - Configure the HTTP request to upload generated media to the local storage or service endpoint.

11. **Error Handling for Upload:**
    - Add a **Code** node named `Fail Upload` connected from the error output of `Upload Attachments LOCAL`.
    - Add a **NoOp** node named `Link This To Error Handling` connected from `Fail Upload`.
    - Configure custom error logic inside the code node.

12. **Trigger and Result Retrieval:**
    - Add an **HTTP Request** node named `Trigger LOCAL Workflow` connected from `Connection Config`.
    - Configure to call the local ComfyUI API to start generation.
    - Add a **Code** node named `Fail Trigger` connected from error output.
    - Add an **HTTP Request** node named `HTTP Request` connected from `Trigger LOCAL Workflow`.
    - Configure to poll or request generation status.
    - Add a **Code** node named `Fail Get History` connected from error output.
    - Add an **If** node named `If` connected from `HTTP Request` to check if generation is ready.
    - Add a **Wait** node named `Wait` connected from false output.
    - Add an **HTTP Request** node named `Get Generated Image` connected from true output.
    - Configure to fetch generated image data.

13. **Output Handling and Logging:**
    - Add a **Set** node named `Return The Output JSON Instead` connected from error output of `Get Generated Image`.
    - Add a **Set** node named `Edit Fields` connected from `Merge`.
    - Add a **Convert To File** node connected from `Edit Fields`.
    - Add a **Read/Write File** node named `Write to error log` connected from `Convert to File`.
    - Add a **Merge** node connected from `Fail Trigger` and `Fail Get History` to unify error and success branches.

14. **Sticky Notes:**
    - Optionally add **Sticky Note** nodes for documentation or annotation at relevant points.

15. **Credentials Setup:**
    - Ensure all HTTP Request nodes are configured with proper credentials or API keys if required.
    - Local access to ComfyUI endpoints should have authentication or be on trusted network.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                           |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------|
| Export your workflow in API format from ComfyUI file menu: Workflow > Export(API)             | Applies to `Read API Exported ... Workflow from Disk` nodes |
| This workflow manages both Txt2Img and Img2Img AI media generation modes                       | Multi-modal AI media generation                           |
| Error handling nodes use custom JavaScript code for more flexible error management            | `Fail Upload`, `Fail Trigger`, `Fail Get History` nodes  |
| Local ComfyUI execution requires a running local instance accessible via HTTP                 | `🎨🏠 Run local ComfyUI workflow` and HTTP Request nodes  |
| Sticky notes are placeholders for user documentation or comments; currently empty             | Can be used to add contextual info for maintainers       |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.