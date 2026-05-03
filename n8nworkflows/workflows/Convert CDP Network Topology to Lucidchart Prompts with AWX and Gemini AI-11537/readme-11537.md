Convert CDP Network Topology to Lucidchart Prompts with AWX and Gemini AI

https://n8nworkflows.xyz/workflows/convert-cdp-network-topology-to-lucidchart-prompts-with-awx-and-gemini-ai-11537


# Convert CDP Network Topology to Lucidchart Prompts with AWX and Gemini AI

---

### 1. Workflow Overview

This workflow automates the conversion of Layer-2 network topology data, collected via AWX (Ansible Tower), into detailed prompts for Lucidchart AI to generate network diagrams. It targets network engineers and automation specialists who want to visualize CDP (Cisco Discovery Protocol) neighbor relationships without manual parsing or diagramming.  

The workflow consists of three main logical blocks:

- **1.1 AWX Job Execution and Monitoring:**  
  Launches an AWX Job Template that runs network commands (e.g., "show cdp neighbors detail") on devices, then polls and waits until the job completes successfully, retrieving the job output.

- **1.2 Parsing and Data Aggregation:**  
  Processes the raw stdout from AWX job events to extract CDP data blocks, parse them into device objects, and aggregate these into a structured list.

- **1.3 Prompt Generation and Document Creation:**  
  Uses Google Gemini LLM nodes to transform the aggregated topology data into a normalized Layer-2 neighbor JSON, then into a Lucidchart textual prompt. Finally, it creates and updates a Google Docs document with this prompt for easy copying and use.

---

### 2. Block-by-Block Analysis

#### 2.1 AWX Job Execution and Monitoring

**Overview:**  
This block triggers an AWX job to collect network device data, then monitors the job status by polling until it completes successfully, and finally fetches the job output (stdout).

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- HTTP Request-Launch Job  
- Edit Fields Parametros Basicos (disabled)  
- HTTP Request - Get JobStatus  
- If (Check job status)  
- Wait  
- HTTP Request - Get JobStdout

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Inputs: None  
  - Outputs: HTTP Request-Launch Job  
  - Edge cases: Manual execution required; no inputs expected.

- **HTTP Request-Launch Job**  
  - Type: HTTP Request (POST)  
  - Role: Launches AWX Job Template by POSTing to the AWX API endpoint `/api/v2/job_templates/1/launch/`.  
  - Configuration: Sends JSON body containing extra_vars with commands (`cmds`) from input JSON; uses HTTP Basic Auth credential.  
  - Inputs: Manual trigger  
  - Outputs: Edit Fields Parametros Basicos (disabled)  
  - Potential failures: HTTP auth failure, network unreachable, invalid job template ID, malformed JSON in `cmds`.  
  - Sticky Note: Instruction to set AWX API URL and job template ID (e.g., `http://10.1.1.1:80/api/v2/job_templates/1/launch/`).

- **Edit Fields Parametros Basicos** (disabled)  
  - Type: Set (disabled)  
  - Role: Intended to set the job_id field from response `job` property.  
  - Since disabled, no active role.

- **HTTP Request - Get JobStatus**  
  - Type: HTTP Request (GET)  
  - Role: Polls AWX API `/api/v2/jobs/{job_id}/` to get current job status.  
  - Configuration: Uses HTTP Basic Auth; job_id dynamically retrieved from previous node output (`{{$json.job_id || $json.id}}`).  
  - Inputs: From Edit Fields Parametros Basicos (or HTTP Request-Launch Job if that is sending job_id)  
  - Outputs: If node  
  - Failures: Auth errors, job_id missing or invalid, timeouts.

- **If**  
  - Type: Conditional (If)  
  - Role: Checks if job status is `"successful"`.  
  - Condition: `{{$json.status}} == "successful"`  
  - Inputs: HTTP Request - Get JobStatus  
  - Outputs:  
    - True branch: HTTP Request - Get JobStdout  
    - False branch: Wait node (delays before next status poll)  
  - Edge cases: Status field missing or unexpected value; case sensitivity enforced.

- **Wait**  
  - Type: Wait  
  - Role: Delays workflow for 30 seconds before next job status poll.  
  - Inputs: If node (False branch)  
  - Outputs: HTTP Request - Get JobStatus  
  - Failures: None typical, but excessive wait time could delay workflow.

- **HTTP Request - Get JobStdout**  
  - Type: HTTP Request (GET)  
  - Role: Retrieves job events (stdout) from AWX API `/api/v2/jobs/{id}/job_events/`.  
  - Configuration: Requests up to 50,000 job events with JSON formatting; uses HTTP Basic Auth.  
  - Inputs: If node (True branch)  
  - Outputs: Parser node  
  - Failures: Large payloads may cause timeouts; auth errors; incomplete stdout data.

---

#### 2.2 Parsing and Data Aggregation

**Overview:**  
This block parses the job event data to extract raw CDP neighbor blocks, converts them into JSON structures per device, and aggregates all devices into a single array for further AI processing.

**Nodes Involved:**  
- Parser  
- Aggregate Devices  
- Generate L2 Topology  
- Build nodes & links

**Node Details:**

- **Parser**  
  - Type: Code (JavaScript)  
  - Role: Extracts all CDP JSON data blocks marked between `CDP_DATA_START` and `CDP_DATA_END` in the AWX job stdout event messages.  
  - Logic:  
    - Concatenates all event messages’ `msg` fields.  
    - Uses RegEx to find all CDP JSON blocks.  
    - Parses each block to a JSON object with `hostname`, `network_os`, and `cdp_output`.  
    - Returns array of device JSON objects or error if none found or parsing fails.  
  - Inputs: HTTP Request - Get JobStdout  
  - Outputs: Aggregate Devices  
  - Edge cases: No CDP blocks found, malformed JSON blocks, partial stdout data.

- **Aggregate Devices**  
  - Type: Code (JavaScript)  
  - Role: Compiles all parsed device objects into a single JSON array named `devices`.  
  - Logic: Maps incoming items to an array of objects with keys: `hostname`, `network_os`, `cdp_output`.  
  - Inputs: Parser  
  - Outputs: Generate L2 Topology  
  - Edge cases: Empty input array.

- **Generate L2 Topology**  
  - Type: Google Gemini (LLM)  
  - Role: Uses Google Gemini AI to parse raw CDP outputs into normalized JSON describing Layer-2 neighbor relationships per device.  
  - Configuration:  
    - Model: `models/gemini-2.5-flash`  
    - Prompt instructs the model to extract neighbors with fields: `local_interface`, `neighbor_id`, `neighbor_interface`, `platform`, `capabilities`.  
    - Input: JSON stringified `devices` array.  
  - Inputs: Aggregate Devices  
  - Outputs: Build nodes & links  
  - Failures: LLM API errors, malformed input JSON, response parsing errors.

- **Build nodes & links**  
  - Type: Code (JavaScript)  
  - Role: Cleans the LLM output text to remove any markdown fences or mermaid tags, preparing clean JSON for next prompt generation.  
  - Inputs: Generate L2 Topology  
  - Outputs: Prompt for Lucid  
  - Edge cases: Unexpected formatting from LLM response.

---

#### 2.3 Prompt Generation and Document Creation

**Overview:**  
Transforms the normalized topology JSON into a Lucidchart AI prompt describing devices and links, then creates and updates a Google Docs document with this prompt.

**Nodes Involved:**  
- Prompt for Lucid  
- Create a document  
- Update a document

**Node Details:**

- **Prompt for Lucid**  
  - Type: Google Gemini (LLM)  
  - Role: Converts the network topology JSON into a concise, structured textual prompt for Lucidchart AI to generate a network diagram.  
  - Configuration:  
    - Model: `models/gemini-2.5-flash`  
    - Instructions include specifying main device, device list with attributes (name, platform, IP, device type), link formatting, grouping, and layout rules.  
    - Input: JSON stringified normalized topology from Build nodes & links.  
  - Inputs: Build nodes & links  
  - Outputs: Create a document  
  - Failures: LLM API errors, prompt formatting issues.

- **Create a document**  
  - Type: Google Docs  
  - Role: Creates a new Google Docs document titled "PROMPT FOR LUCID" in the default folder.  
  - Inputs: Prompt for Lucid  
  - Outputs: Update a document  
  - Credential: Google Docs OAuth2  
  - Failures: Google API auth or quota errors.

- **Update a document**  
  - Type: Google Docs  
  - Role: Inserts the generated Lucidchart prompt text into the newly created document.  
  - Configuration: Inserts text from `Prompt for Lucid` node output at the document start.  
  - Inputs: Create a document  
  - Outputs: None (terminal)  
  - Failures: Document ID missing or invalid, Google API errors.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                    | Input Node(s)                   | Output Node(s)                       | Sticky Note                                                                                                         |
|-----------------------------|----------------------------------|---------------------------------------------------|--------------------------------|------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Entry trigger to start workflow                    | None                           | HTTP Request-Launch Job             |                                                                                                                     |
| HTTP Request-Launch Job      | HTTP Request (POST)               | Launches AWX job template                          | When clicking ‘Execute workflow’| Edit Fields Parametros Basicos (disabled) | Set url for API request in AWX; example: http://10.1.1.1:80/api/v2/job_templates/1/launch/ for the playbook (contact author) |
| Edit Fields Parametros Basicos | Set (disabled)                   | Intended to set job_id (disabled)                  | HTTP Request-Launch Job         | HTTP Request - Get JobStatus         | Section 1 Call AWX Playbook and set the job id                                                                      |
| HTTP Request - Get JobStatus | HTTP Request (GET)                | Poll AWX job status                               | Edit Fields Parametros Basicos  | If                                 | Section 2 Follow job status and Parsing Information                                                                 |
| If                          | If (Conditional)                 | Check if job status is successful                  | HTTP Request - Get JobStatus    | HTTP Request - Get JobStdout, Wait  | Section 2 Follow job status and Parsing Information                                                                 |
| Wait                        | Wait                            | Delay before next job status poll                   | If                            | HTTP Request - Get JobStatus         | Section 2 Follow job status and Parsing Information                                                                 |
| HTTP Request - Get JobStdout | HTTP Request (GET)                | Retrieve job output (stdout)                        | If                            | Parser                             | Section 2 Follow job status and Parsing Information                                                                 |
| Parser                      | Code (JavaScript)                | Extract CDP JSON blocks from job stdout             | HTTP Request - Get JobStdout    | Aggregate Devices                   | Section 2 Follow job status and Parsing Information                                                                 |
| Aggregate Devices           | Code (JavaScript)                | Combine parsed devices into one JSON array          | Parser                        | Generate L2 Topology               | Section 3 Create Prompt and save it!!                                                                                |
| Generate L2 Topology         | Google Gemini (LLM)              | Parse CDP outputs into normalized L2 neighbor JSON | Aggregate Devices              | Build nodes & links                | Section 3 Create Prompt and save it!!                                                                                |
| Build nodes & links          | Code (JavaScript)                | Clean LLM output for prompt generation              | Generate L2 Topology           | Prompt for Lucid                   | Section 3 Create Prompt and save it!!                                                                                |
| Prompt for Lucid             | Google Gemini (LLM)              | Generate Lucidchart prompt from topology JSON       | Build nodes & links            | Create a document                 | Section 3 Create Prompt and save it!!                                                                                |
| Create a document            | Google Docs                     | Create Google Docs document for prompt              | Prompt for Lucid              | Update a document                 | Section 3 Create Prompt and save it!!                                                                                |
| Update a document            | Google Docs                     | Insert prompt text into Google Docs document         | Create a document             | None                             | Section 3 Create Prompt and save it!!                                                                                |
| Sticky Note                 | Sticky Note                     | Workflow description and setup instructions          | None                         | None                             | AI Network Diagram Prompt Generator overview and setup instructions                                                 |
| Sticky Note1                | Sticky Note                     | Section 1 description                                | None                         | None                             | Section 1 Call AWX Playbook and set the job id                                                                      |
| Sticky Note2                | Sticky Note                     | Section 2 description                                | None                         | None                             | Section 2 Follow job status and Parsing Information                                                                 |
| Sticky Note3                | Sticky Note                     | Section 3 description                                | None                         | None                             | Section 3 Create Prompt and save it!!                                                                                |
| Sticky Note4                | Sticky Note                     | Example URL for AWX API request                      | None                         | None                             | Set url for api request in AWX; example: http://10.1.1.1:80/api/v2/job_templates/1/launch/ for the playbook (contact author) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Role: Entry point to start the workflow manually.

2. **Add HTTP Request Node to Launch AWX Job**  
   - Name: `HTTP Request-Launch Job`  
   - Method: POST  
   - URL: `http://<AWX_IP>:<PORT>/api/v2/job_templates/<TEMPLATE_ID>/launch/` (replace with your AWX details)  
   - Auth: HTTP Basic Auth (configure credentials with AWX username/password)  
   - Headers: Content-Type and Accept as `application/json`  
   - Body (JSON): `{ "extra_vars": { "cmds": {{$json.cmds}} } }`  
   - Connect manual trigger output to this node.

3. **(Optional) Create Set Node to assign job_id**  
   - Name: `Edit Fields Parametros Basicos`  
   - Disabled by default, but can assign `job_id` from previous node response `job` field if needed.  
   - Connect `HTTP Request-Launch Job` output to this node.

4. **Add HTTP Request Node to Get Job Status**  
   - Name: `HTTP Request - Get JobStatus`  
   - Method: GET  
   - URL: `http://<AWX_IP>:<PORT>/api/v2/jobs/{{$json.job_id || $json.id}}/`  
   - Auth: HTTP Basic Auth (same credentials as above)  
   - Connect `Edit Fields Parametros Basicos` (or `HTTP Request-Launch Job`) output to this node.

5. **Add If Node to Check Job Status**  
   - Name: `If`  
   - Condition: Check if `{{$json.status}}` equals `"successful"` (case sensitive)  
   - Connect `HTTP Request - Get JobStatus` output to this node.

6. **Add Wait Node for Polling Delay**  
   - Name: `Wait`  
   - Delay: 30 seconds  
   - Connect `If` node’s False output (job not successful) to this node.

7. **Connect Wait Node Back to Job Status Node**  
   - Connect `Wait` output to `HTTP Request - Get JobStatus` to create a polling loop.

8. **Add HTTP Request Node to Get Job Stdout**  
   - Name: `HTTP Request - Get JobStdout`  
   - Method: GET  
   - URL: `http://<AWX_IP>:<PORT>/api/v2/jobs/{{$json.id}}/job_events/?format=json&page_size=50000`  
   - Auth: HTTP Basic Auth  
   - Connect `If` node’s True output (job successful) to this node.

9. **Add Code Node to Parse CDP JSON Blocks**  
   - Name: `Parser`  
   - JavaScript: Extract `CDP_DATA_START` / `CDP_DATA_END` blocks, parse JSON, handle errors.  
   - Connect `HTTP Request - Get JobStdout` output to this node.

10. **Add Code Node to Aggregate Devices**  
    - Name: `Aggregate Devices`  
    - JavaScript: Map parsed items to array of `{ hostname, network_os, cdp_output }`.  
    - Connect `Parser` output to this node.

11. **Add Google Gemini Node to Generate L2 Topology**  
    - Name: `Generate L2 Topology`  
    - Model: `models/gemini-2.5-flash`  
    - Prompt: Instruct to parse CDP output to normalized JSON neighbor structure.  
    - Credentials: Google Gemini API configured.  
    - Connect `Aggregate Devices` output to this node.

12. **Add Code Node to Clean Gemini Output**  
    - Name: `Build nodes & links`  
    - JavaScript: Remove markdown fences and clean text from Gemini output.  
    - Connect `Generate L2 Topology` output to this node.

13. **Add Google Gemini Node to Generate Lucidchart Prompt**  
    - Name: `Prompt for Lucid`  
    - Model: `models/gemini-2.5-flash`  
    - Prompt: Convert network topology JSON into Lucidchart AI prompt with device and link details.  
    - Credentials: Google Gemini API configured.  
    - Connect `Build nodes & links` output to this node.

14. **Add Google Docs Node to Create Document**  
    - Name: `Create a document`  
    - Operation: Create  
    - Title: `"PROMPT FOR LUCID"`  
    - Folder: Default or specific Drive folder ID  
    - Credentials: Google Docs OAuth2 configured.  
    - Connect `Prompt for Lucid` output to this node.

15. **Add Google Docs Node to Update Document**  
    - Name: `Update a document`  
    - Operation: Update  
    - Document URL: `{{$json.id}}` (from `Create a document`)  
    - Action: Insert text from `Prompt for Lucid` output JSON content (the prompt text).  
    - Credentials: Google Docs OAuth2.  
    - Connect `Create a document` output to this node.

16. **Add Sticky Notes (Optional)**  
    - Add sticky notes to describe workflow sections and setup instructions for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| This workflow automatically retrieves Layer-2 topology data from network devices via AWX, parses CDP neighbor details, and generates a Lucidchart AI prompt for network diagrams. Setup requires AWX Job Template, credentials in n8n for AWX and Google Gemini, and Google Docs API credentials. Modify commands, parser logic, or prompt style to fit custom needs. | Sticky Note content: AI Network Diagram Prompt Generator overview and setup instructions.                                      |
| For AWX Job Template playbook or customization, contact the workflow author.                                                                                                                                                                                                                                                                                                             | Mentioned in sticky notes and API endpoint examples.                                                                           |
| Google Gemini (PaLM) API is used for LLM processing; ensure proper API key and quotas.                                                                                                                                                                                                                                                                                                     | Credential requirement for LLM nodes.                                                                                          |
| Google Docs OAuth2 credentials are required to create and update documents automatically.                                                                                                                                                                                                                                                                                                  | Credential requirement for Google Docs nodes.                                                                                  |
| The workflow is designed to handle large job event payloads (up to 50,000 events) but network or API rate limits may affect performance.                                                                                                                                                                                                                                                 | In HTTP Request - Get JobStdout node configuration.                                                                             |
| Customization options include integrating Slack, Drive, email, or diagram rendering via Kroki.                                                                                                                                                                                                                                                                                             | Mentioned in sticky notes under “Customization.”                                                                               |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies with all applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.

---