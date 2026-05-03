Package & Deploy Multiple n8n Workflows with Automatic Credential Mapping

https://n8nworkflows.xyz/workflows/package---deploy-multiple-n8n-workflows-with-automatic-credential-mapping-7028


# Package & Deploy Multiple n8n Workflows with Automatic Credential Mapping

---

### 1. Workflow Overview

This workflow automates the packaging and deployment of multiple n8n workflows with automatic credential mapping. Its core purpose is to take a compressed package of workflows, decompress and split them, fix and map required credentials, create the workflows in a target n8n environment, and optionally move them into a specified project. This is highly useful for automating multi-workflow deployments, ensuring credentials align properly, and managing workflows in bulk.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Credential Setup:** Manual initiation and validation of required credentials.
- **1.2 Input Preparation and Decompression:** Receiving the packaged workflows, converting to file format, and extracting workflows.
- **1.3 Workflow Splitting and Credential Fixing:** Splitting decompressed workflows into individual units and fixing credential references automatically.
- **1.4 Workflow Creation and Project Assignment:** Creating workflows in the target environment and optionally assigning them to a project.
- **1.5 Loop Control and Completion:** Looping over workflows for sequential processing and signaling completion or errors.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Credential Setup

**Overview:**  
This block starts the workflow manually and ensures that all necessary credentials (OpenAI, GitHub, and n8n) are configured before proceeding.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- OpenAi Credentials  
- Github Credentials  
- n8n Credentials  
- Credential Dictionary  
- Extract Project Details  
- If 3 Credentials  
- Merge  
- Stop and Error

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Start node activated manually by user  
  - Configuration: Default manual trigger  
  - Inputs: None  
  - Outputs: Connects to OpenAi Credentials  
  - Edge cases: None typical, manual start requires user interaction

- **OpenAi Credentials**  
  - Type: HTTP Request (used here as a credential validation node)  
  - Role: Validates/configures OpenAI credentials  
  - Configuration: User must set authentication and endpoint info  
  - Inputs: From trigger  
  - Outputs: To Github Credentials  
  - Edge cases: Authentication errors, network timeouts

- **Github Credentials**  
  - Type: GitHub node  
  - Role: Validates/configures GitHub credentials  
  - Configuration: Requires OAuth or token setup  
  - Inputs: From OpenAi Credentials  
  - Outputs: To n8n Credentials  
  - Edge cases: Permission issues, invalid tokens

- **n8n Credentials**  
  - Type: n8n node (likely to validate access to target n8n instance)  
  - Role: Validates/configures n8n credentials to target instance  
  - Configuration: Requires OAuth2 or API key for n8n instance  
  - Inputs: From Github Credentials  
  - Outputs: To Credential Dictionary and Extract Project Details nodes  
  - Edge cases: Unauthorized access, invalid credentials

- **Credential Dictionary**  
  - Type: Set node  
  - Role: Creates an internal dictionary mapping credential names or IDs for use later in credential fixing  
  - Configuration: Static or dynamic mapping of credential identifiers  
  - Inputs: From n8n Credentials  
  - Outputs: To If 3 Credentials  
  - Edge cases: Missing keys or mismatches in dictionary entries

- **Extract Project Details**  
  - Type: Set node  
  - Role: Extracts or sets parameters related to project assignment (e.g., project ID)  
  - Configuration: Depends on input data or static assignment  
  - Inputs: From n8n Credentials  
  - Outputs: To Merge node  
  - Edge cases: Missing or invalid project info

- **If 3 Credentials**  
  - Type: If node  
  - Role: Checks if all three required credentials are present and valid  
  - Configuration: Conditional check on presence/validity of OpenAI, GitHub, and n8n credentials  
  - Inputs: From Credential Dictionary  
  - Outputs:  
    - True branch: To Merge node  
    - False branch: To Stop and Error node  
  - Edge cases: Failed credential validation triggers error stop

- **Merge**  
  - Type: Merge node  
  - Role: Combines outputs from Extract Project Details and If 3 Credentials for downstream processing  
  - Configuration: Default merge mode (likely merge by index)  
  - Inputs: From Extract Project Details and If 3 Credentials  
  - Outputs: To Installer Data  
  - Edge cases: Mismatched data length or missing inputs

- **Stop and Error**  
  - Type: Stop and Error node  
  - Role: Stops workflow execution and reports error if credential checks fail  
  - Configuration: Default error stop  
  - Inputs: From If 3 Credentials (false branch)  
  - Outputs: None (workflow terminates here)  
  - Edge cases: Triggered on missing/invalid credentials

---

#### 1.2 Input Preparation and Decompression

**Overview:**  
This block receives the packaged workflows, converts them to a file format, decompresses the base64 encoded data, and prepares the workflows for splitting.

**Nodes Involved:**  
- Installer Data  
- Convert to File  
- Decompress Workflows  
- Split Workflows

**Node Details:**

- **Installer Data**  
  - Type: Set node  
  - Role: Sets or prepares the incoming data package (likely a compressed base64 string of workflows)  
  - Configuration: Static or dynamic data input (not detailed)  
  - Inputs: From Merge  
  - Outputs: To Convert to File  
  - Edge cases: Missing or malformed input data

- **Convert to File**  
  - Type: Convert to File node  
  - Role: Converts raw data into a file object for extraction  
  - Configuration: Default conversion settings  
  - Inputs: From Installer Data  
  - Outputs: To Decompress Workflows  
  - Edge cases: Conversion failures if data is not valid for file conversion

- **Decompress Workflows**  
  - Type: Extract From File node  
  - Role: Extracts and decompresses JSON from base64 encoded file data  
  - Configuration: Default extraction, expects base64-encoded JSON content  
  - Inputs: From Convert to File  
  - Outputs: To Split Workflows  
  - Edge cases: Corrupt or invalid base64 data, extraction failures

- **Split Workflows**  
  - Type: Split Out node  
  - Role: Splits the decompressed workflows JSON array into individual workflow items  
  - Configuration: Default split settings  
  - Inputs: From Decompress Workflows  
  - Outputs: To Fix Credentials  
  - Edge cases: Empty or malformed workflow array

---

#### 1.3 Workflow Splitting and Credential Fixing

**Overview:**  
This block processes each individual workflow, fixes or maps credentials according to the earlier dictionary, and prepares for creation.

**Nodes Involved:**  
- Fix Credentials  
- Loop Over Workflows  
- Done

**Node Details:**

- **Fix Credentials**  
  - Type: Code node  
  - Role: Custom JavaScript code that inspects and replaces credential references in each workflow with the mapped credentials from the dictionary  
  - Configuration: Contains code logic to iterate over workflow nodes and update credentials accordingly  
  - Inputs: From Split Workflows  
  - Outputs: To Loop Over Workflows  
  - Edge cases: Code errors, missing expected credential fields, unexpected workflow structure

- **Loop Over Workflows**  
  - Type: Split In Batches node  
  - Role: Processes workflows one at a time or in batches to manage load and sequential creation  
  - Configuration: Batch size likely set to 1 for sequential processing  
  - Inputs: From Fix Credentials  
  - Outputs:  
    - Main output: To Extract Workflow for processing individual workflow creation  
    - Secondary output: To Done node indicating completion of batch processing  
  - Edge cases: Batch processing errors, timeouts

- **Done**  
  - Type: No Operation (NoOp) node  
  - Role: Marks completion of loop/batch processing  
  - Inputs: From Loop Over Workflows  
  - Outputs: To Install Success1 node  
  - Edge cases: None

---

#### 1.4 Workflow Creation and Project Assignment

**Overview:**  
This block creates each workflow in the target n8n instance, checks if a project assignment is specified, and moves the workflow into the project if applicable.

**Nodes Involved:**  
- Extract Workflow  
- Create Workflow  
- If Project  
- Move to Project  
- No Op  
- Install Success1

**Node Details:**

- **Extract Workflow**  
  - Type: Set node  
  - Role: Extracts or prepares the workflow data for creation from the batch input  
  - Configuration: Extracts relevant JSON workflow object  
  - Inputs: From Loop Over Workflows  
  - Outputs: To Create Workflow  
  - Edge cases: Missing workflow data

- **Create Workflow**  
  - Type: n8n node (invokes n8n API to create workflow)  
  - Role: Creates the workflow on the target n8n instance using API or internal node  
  - Configuration: Uses credentials from earlier nodes, passes workflow JSON  
  - Inputs: From Extract Workflow  
  - Outputs: To If Project  
  - Edge cases: API errors, workflow creation failures, invalid JSON

- **If Project**  
  - Type: If node  
  - Role: Checks if project details are present to decide if the workflow should be moved to a project  
  - Configuration: Condition checks project ID or flag presence  
  - Inputs: From Create Workflow  
  - Outputs:  
    - True branch: To Move to Project  
    - False branch: To No Op  
  - Edge cases: Missing project info, logic errors

- **Move to Project**  
  - Type: HTTP Request node  
  - Role: Calls n8n API to move the newly created workflow into the specified project  
  - Configuration: Requires project ID, workflow ID, and valid n8n credentials  
  - Inputs: From If Project (true branch)  
  - Outputs: To No Op  
  - Edge cases: API authorization errors, invalid IDs

- **No Op**  
  - Type: No Operation node  
  - Role: Acts as a pass-through or placeholder to continue workflow execution  
  - Inputs: From Move to Project or If Project (false branch)  
  - Outputs: To Loop Over Workflows for next iteration  
  - Edge cases: None

- **Install Success1**  
  - Type: No Operation node  
  - Role: Marks successful completion of all workflow installations  
  - Inputs: From Done  
  - Outputs: None (terminal node)  
  - Edge cases: None

---

#### 1.5 Loop Control and Completion

**Overview:**  
Manages the iterative processing of workflows and signals the end of the deployment process.

**Nodes Involved:**  
- Loop Over Workflows  
- Done  
- Install Success1

**Node Details:**

- Already detailed above in 1.3 and 1.4, these nodes control batch processing, loop iteration, and final success signaling.

---

### 3. Summary Table

| Node Name                  | Node Type               | Functional Role                          | Input Node(s)                                | Output Node(s)                        | Sticky Note                                                                                          |
|----------------------------|-------------------------|----------------------------------------|----------------------------------------------|-------------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger          | Manual start trigger                    | None                                         | OpenAi Credentials                   |                                                                                                    |
| OpenAi Credentials          | HTTP Request            | Validate/configure OpenAI credentials  | When clicking ‘Execute workflow’             | Github Credentials                  | Configure this.                                                                                     |
| Github Credentials          | GitHub                  | Validate/configure GitHub credentials  | OpenAi Credentials                           | n8n Credentials                    | Configure this.                                                                                     |
| n8n Credentials             | n8n                     | Validate/configure n8n target credentials | Github Credentials                          | Credential Dictionary, Extract Project Details | Configure this.                                                                                     |
| Credential Dictionary       | Set                     | Build internal credential mapping      | n8n Credentials                             | If 3 Credentials                   | Create Dictionary                                                                                   |
| Extract Project Details     | Set                     | Extract project assignment details     | n8n Credentials                             | Merge                              |                                                                                                    |
| If 3 Credentials            | If                      | Check all 3 credentials presence       | Credential Dictionary                       | Merge (true branch), Stop and Error (false branch) |                                                                                                    |
| Stop and Error              | Stop and Error          | Halt execution on missing credentials  | If 3 Credentials (false)                    | None                              |                                                                                                    |
| Merge                      | Merge                   | Combine project details and credential checks | Extract Project Details, If 3 Credentials | Installer Data                    |                                                                                                    |
| Installer Data             | Set                     | Prepare packaged workflow data         | Merge                                       | Convert to File                   |                                                                                                    |
| Convert to File            | Convert to File          | Convert raw data to file                | Installer Data                              | Decompress Workflows               |                                                                                                    |
| Decompress Workflows       | Extract From File        | Decompress base64 workflow JSON         | Convert to File                             | Split Workflows                   | Extract JSON from base64                                                                             |
| Split Workflows            | Split Out                | Split workflows array into individual workflows | Decompress Workflows                      | Fix Credentials                   |                                                                                                    |
| Fix Credentials            | Code                     | Replace credential references dynamically | Split Workflows                           | Loop Over Workflows               |                                                                                                    |
| Loop Over Workflows        | Split In Batches         | Process workflows sequentially          | Fix Credentials                            | Done (batch complete), Extract Workflow (processing) |                                                                                                    |
| Done                      | NoOp                     | Marks completion of batch processing    | Loop Over Workflows                         | Install Success1                  |                                                                                                    |
| Extract Workflow           | Set                      | Extract single workflow data             | Loop Over Workflows                         | Create Workflow                  |                                                                                                    |
| Create Workflow            | n8n                      | Create workflow in target environment   | Extract Workflow                           | If Project                      |                                                                                                    |
| If Project                 | If                       | Check if project assignment is needed   | Create Workflow                            | Move to Project (true), No Op (false) |                                                                                                    |
| Move to Project            | HTTP Request             | Move workflow into specified project    | If Project (true)                          | No Op                           |                                                                                                    |
| No Op                     | NoOp                     | Pass-through node                        | Move to Project, If Project (false)         | Loop Over Workflows               |                                                                                                    |
| Install Success1           | NoOp                     | Marks successful installation completion | Done                                      | None                           |                                                                                                    |
| Sticky Note                | Sticky Note              | Comment/Instructional notes              | None                                        | None                              |                                                                                                    |
| Sticky Note2               | Sticky Note              | Comment/Instructional notes              | None                                        | None                              |                                                                                                    |
| Sticky Note3               | Sticky Note              | Comment/Instructional notes              | None                                        | None                              |                                                                                                    |
| Sticky Note4               | Sticky Note              | Comment/Instructional notes              | None                                        | None                              |                                                                                                    |
| Sticky Note5               | Sticky Note              | Comment/Instructional notes              | None                                        | None                              |                                                                                                    |
| Sticky Note6               | Sticky Note              | Comment/Instructional notes              | None                                        | None                              |                                                                                                    |
| Sticky Note7               | Sticky Note              | Comment/Instructional notes              | None                                        | None                              |                                                                                                    |
| Sticky Note8               | Sticky Note              | Comment/Instructional notes              | None                                        | None                              |                                                                                                    |
| Sticky Note10              | Sticky Note              | Comment/Instructional notes              | None                                        | None                              |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Position: Start node  
   - Purpose: To manually start the workflow.

2. **Add OpenAi Credentials Node**  
   - Type: HTTP Request  
   - Configure with OpenAI API credentials (API key, endpoint)  
   - Connect Manual Trigger → OpenAi Credentials.

3. **Add Github Credentials Node**  
   - Type: GitHub node  
   - Configure with OAuth/token for GitHub access  
   - Connect OpenAi Credentials → Github Credentials.

4. **Add n8n Credentials Node**  
   - Type: n8n node (API access to target n8n)  
   - Configure with OAuth2/API key for target n8n instance  
   - Connect Github Credentials → n8n Credentials.

5. **Add Credential Dictionary Node**  
   - Type: Set node  
   - Configure to build a dictionary object mapping required credential names to target credential IDs/names  
   - Connect n8n Credentials → Credential Dictionary.

6. **Add Extract Project Details Node**  
   - Type: Set node  
   - Configure to extract or set project ID or assignment flag if applicable  
   - Connect n8n Credentials → Extract Project Details.

7. **Add If 3 Credentials Node**  
   - Type: If node  
   - Configure condition to check presence/validity of OpenAI, GitHub, and n8n credentials  
   - Connect Credential Dictionary → If 3 Credentials.

8. **Add Stop and Error Node**  
   - Type: Stop and Error node  
   - Connect If 3 Credentials (false branch) → Stop and Error.

9. **Add Merge Node**  
   - Type: Merge node  
   - Connect Extract Project Details → Merge (input 1)  
   - Connect If 3 Credentials (true branch) → Merge (input 2).

10. **Add Installer Data Node**  
    - Type: Set node  
    - Configure to accept the packaged workflows data input (compressed base64 string)  
    - Connect Merge → Installer Data.

11. **Add Convert to File Node**  
    - Type: Convert to File node  
    - Connect Installer Data → Convert to File.

12. **Add Decompress Workflows Node**  
    - Type: Extract From File node  
    - Configure to extract JSON from base64 file data  
    - Connect Convert to File → Decompress Workflows.

13. **Add Split Workflows Node**  
    - Type: Split Out node  
    - Connect Decompress Workflows → Split Workflows.

14. **Add Fix Credentials Node**  
    - Type: Code node  
    - Write JavaScript to iterate through each workflow’s nodes and replace credentials using the dictionary set earlier  
    - Connect Split Workflows → Fix Credentials.

15. **Add Loop Over Workflows Node**  
    - Type: Split In Batches node  
    - Set batch size to 1 for sequential processing  
    - Connect Fix Credentials → Loop Over Workflows.

16. **Add Extract Workflow Node**  
    - Type: Set node  
    - Configure to extract a single workflow JSON from batch item  
    - Connect Loop Over Workflows → Extract Workflow.

17. **Add Create Workflow Node**  
    - Type: n8n node  
    - Configure to create a workflow in the target n8n instance using API and credentials  
    - Connect Extract Workflow → Create Workflow.

18. **Add If Project Node**  
    - Type: If node  
    - Configure to check if project details exist for assignment  
    - Connect Create Workflow → If Project.

19. **Add Move to Project Node**  
    - Type: HTTP Request node  
    - Configure to call n8n API to move created workflow into the specified project (requires project ID and workflow ID)  
    - Connect If Project (true branch) → Move to Project.

20. **Add No Op Node**  
    - Type: No Operation node  
    - Connect Move to Project → No Op  
    - Connect If Project (false branch) → No Op.

21. **Connect No Op → Loop Over Workflows**  
    - To continue processing next workflow.

22. **Add Done Node**  
    - Type: No Operation node  
    - Connect Loop Over Workflows (completion output) → Done.

23. **Add Install Success1 Node**  
    - Type: No Operation node  
    - Connect Done → Install Success1.

24. **Add Sticky Notes as needed**  
    - Use for documentation and comments within workflow editor.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                |
|------------------------------------------------------------------------------|------------------------------------------------|
| The workflow enables bulk deployment of workflows with automatic credential mapping to avoid manual errors. | High-level project goal                        |
| Credential nodes must be carefully configured with correct API keys and tokens before execution. | Setup requirement                              |
| The “Fix Credentials” code node is critical for replacing source credential references with target environment credentials. | Key logic component                            |
| For moving workflows to projects, the target n8n instance must support the relevant API endpoint. | API dependency                                 |
| Ensure proper error handling for credential failures to avoid partial deployments. | Best practice                                  |

---

Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---