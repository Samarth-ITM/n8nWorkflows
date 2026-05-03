Automated CVE Scanning of Bug Bounty Programs with Nuclei and Project Discovery

https://n8nworkflows.xyz/workflows/automated-cve-scanning-of-bug-bounty-programs-with-nuclei-and-project-discovery-10054


# Automated CVE Scanning of Bug Bounty Programs with Nuclei and Project Discovery

### 1. Workflow Overview

This workflow automates the scanning of bug bounty program domains for newly released CVE (Common Vulnerabilities and Exposures) templates using the Project Discovery Nuclei tool. It is designed for security researchers and bug bounty hunters to efficiently identify vulnerable targets from aggregated bug bounty platforms, applying the latest vulnerability templates and receiving results via email.

The workflow logically divides into four main blocks:

- **1.1 Input Reception:** Periodic trigger to fetch the latest list of bug bounty program domains.

- **1.2 CVE Template Acquisition:** Retrieving new CVE vulnerability templates from Project Discovery API and filtering them.

- **1.3 Template Creation and Execution:** Creating template files on a remote system, running Nuclei scans against the domains with these templates, and cleaning up.

- **1.4 Results Handling and Notification:** Processing Nuclei scan results and sending notifications via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow on a scheduled basis and downloads an up-to-date list of bug bounty target domains from a public GitHub repository.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get All Bug Bounty Domains  
  - Create domains.txt  
  - Upload domains.txt  

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Scheduler node  
    - Role: Initiates workflow execution periodically (default: every hour)  
    - Config: Uses default interval (empty object means default scheduling)  
    - Inputs: None (start node)  
    - Outputs: Connects to "Get All Bug Bounty Domains"  
    - Edge Cases: Misconfiguration of schedule interval could affect timely execution.

  - **Get All Bug Bounty Domains**  
    - Type: HTTP Request  
    - Role: Fetches a plaintext list of bug bounty domains from a GitHub raw URL  
    - Config: URL points to `https://raw.githubusercontent.com/arkadiyt/bounty-targets-data/refs/heads/main/data/domains.txt`  
    - Inputs: Output of Schedule Trigger  
    - Outputs: JSON data containing domain list text to "Create domains.txt"  
    - Edge Cases: Network failure, GitHub resource unavailable, or format changes of source file.

  - **Create domains.txt**  
    - Type: Convert To File  
    - Role: Converts fetched domain list text into a file named `domains.txt`  
    - Config: Operation set to convert text property `data` to file  
    - Inputs: JSON text data from HTTP Request  
    - Outputs: File resource to "Upload domains.txt"  
    - Edge Cases: Empty or malformed input data.

  - **Upload domains.txt**  
    - Type: SSH  
    - Role: Uploads `domains.txt` file to remote system directory `/tmp/nuclei` for scanning  
    - Config: Resource set to file upload, path `/tmp/nuclei`  
    - Credentials: SSH Password authentication must be configured (root or relevant user)  
    - Inputs: File from "Create domains.txt"  
    - Outputs: Triggers "GET Last CVEs (PROJECT DISCOVERY)"  
    - Edge Cases: SSH connection issues, permission problems, disk space.

---

#### 1.2 CVE Template Acquisition

- **Overview:**  
  Retrieves the latest CVE vulnerability templates from the Project Discovery API, splits them for batch processing, and filters templates based on existence and creation date.

- **Nodes Involved:**  
  - GET Last CVEs (PROJECT DISCOVERY)  
  - Split CVEs  
  - Loop Over CVEs  
  - Template Exists Filter  
  - Date Filter  
  - Set Variables  
  - Set Null Variable  

- **Node Details:**

  - **GET Last CVEs (PROJECT DISCOVERY)**  
    - Type: HTTP Request  
    - Role: Queries Project Discovery’s API for the newest vulnerability templates (limit 40)  
    - Config: Query parameters include `scope=public`, `facet_size=40`, `offset=0`, `limit=40`  
    - Inputs: Output from "Upload domains.txt"  
    - Outputs: JSON response with CVE template results to "Split CVEs"  
    - Edge Cases: API rate limits, downtime, invalid responses, unauthorized errors.

  - **Split CVEs**  
    - Type: Split Out  
    - Role: Splits the `results` array from API response into individual CVE items  
    - Config: Field to split out is `results`  
    - Inputs: API response JSON  
    - Outputs: Individual CVE JSON objects to "Loop Over CVEs"  
    - Edge Cases: Empty results array, malformed data.

  - **Loop Over CVEs**  
    - Type: Split In Batches  
    - Role: Processes CVE items one by one (batch size default)  
    - Inputs: Split CVE items  
    - Outputs: Two branches depending on conditions: "Template Exists Filter" and "Date Filter"  
    - Edge Cases: Large number of CVEs may affect performance.

  - **Template Exists Filter**  
    - Type: Filter  
    - Role: Checks that the `Template` field exists (non-empty) in the CVE data  
    - Config: Condition: `$json.Template` exists  
    - Inputs: CVE item from Loop Over CVEs  
    - Outputs: Pass to "Loop Over Templates" if true  
    - Edge Cases: Missing or null template data.

  - **Date Filter**  
    - Type: If  
    - Role: Filters CVEs based on creation date being after or equal to one day before the last workflow run  
    - Config: Extracts date via regex from `$json.created_at`, compares to timestamp of Schedule Trigger minus 1 day  
    - Inputs: CVE item from Loop Over CVEs  
    - Outputs: True branch to "Set Variables", false to "Set Null Variable"  
    - Edge Cases: Malformed date strings, time zone differences, missing `created_at`.

  - **Set Variables**  
    - Type: Set  
    - Role: Maps CVE properties into workflow variables for further processing: CVE ID, CVSS score, raw template, references  
    - Inputs: CVE passing date filter  
    - Outputs: To "Loop Over CVEs" (continuation)  
    - Edge Cases: Missing JSON fields causing undefined values.

  - **Set Null Variable**  
    - Type: Set  
    - Role: Clears variables or sets placeholders where date filter fails  
    - Inputs: CVE failing date filter  
    - Outputs: Back to "Loop Over CVEs"  
    - Edge Cases: None significant.

---

#### 1.3 Template Creation and Execution

- **Overview:**  
  Creates template files on the remote scanning server, uploads them, converts the file extension to `.yaml`, executes Nuclei scans against domains using these templates, and removes templates after use.

- **Nodes Involved:**  
  - Loop Over Templates  
  - Remove Items  
  - Create Template  
  - Upload Template  
  - Convert Template to .yaml  
  - Execute Nuclei  
  - Remove Templates  

- **Node Details:**

  - **Loop Over Templates**  
    - Type: Split In Batches  
    - Role: Iterates over templates filtered for existence, batching template creation and upload  
    - Inputs: Output of "Template Exists Filter"  
    - Outputs: Two branches: "Remove Items" and "Create Template"  
    - Edge Cases: Large number of templates may slow down processing.

  - **Remove Items**  
    - Type: Summarize  
    - Role: Helper node; aggregates or clears batch data before creating template files  
    - Inputs: From "Loop Over Templates"  
    - Outputs: To "Execute Nuclei" (indirectly via continuation)  
    - Edge Cases: None significant.

  - **Create Template**  
    - Type: Convert To File  
    - Role: Converts raw template content into text files named after CVE ID with `.txt` extension  
    - Config: Filename constructed as `${CVE}.txt` using variable from "Set Variables"  
    - Inputs: Batch item from "Loop Over Templates"  
    - Outputs: File to "Upload Template"  
    - Edge Cases: Missing template content causes empty files.

  - **Upload Template**  
    - Type: SSH  
    - Role: Uploads template text files to remote `/tmp/nuclei-templates` directory  
    - Config: SSH Password credentials (same as domain upload)  
    - Inputs: Files from "Create Template"  
    - Outputs: Triggers "Convert Template to .yaml"  
    - Edge Cases: SSH failures, permission issues.

  - **Convert Template to .yaml**  
    - Type: SSH  
    - Role: Renames uploaded template files from `.txt` to `.yaml` extension on remote server  
    - Config: Executes `mv /tmp/nuclei-templates/{CVE}.txt /tmp/nuclei-templates/{CVE}.yaml` command  
    - Inputs: Triggered from "Upload Template"  
    - Outputs: Loops back to "Loop Over Templates" to continue batch processing  
    - Edge Cases: File not found errors, rename failures.

  - **Execute Nuclei**  
    - Type: SSH  
    - Role: Runs the Nuclei scanning tool on the remote server with all uploaded templates against uploaded domains  
    - Config:  
      - Command: `nuclei -l /tmp/nuclei/domains.txt -t /tmp/nuclei-templates -ss host-spray -c 10 -bs 50 -rl 100 -timeout 10 -retries 1 -silent -etags info,low`  
      - Parameters: concurrency 10, batch size 50, rate limit 100, timeout 10s, retry once, silent mode, excludes info/low severity tags  
    - Inputs: Triggered after "Remove Items"  
    - Outputs: To "Remove Templates"  
    - Edge Cases: SSH failures, Nuclei command errors, timeout, large scan duration.

  - **Remove Templates**  
    - Type: SSH  
    - Role: Cleans up all template files from remote `/tmp/nuclei-templates` directory after scan completes  
    - Config: Executes `rm /tmp/nuclei-templates/*` command  
    - Inputs: From "Execute Nuclei"  
    - Outputs: To "Set Results Variable"  
    - Edge Cases: File permission issues, partial deletion.

---

#### 1.4 Results Handling and Notification

- **Overview:**  
  Captures Nuclei scan output, checks for non-empty results, and sends an email notification with findings.

- **Nodes Involved:**  
  - Set Results Variable  
  - Check Results  
  - Send a message  

- **Node Details:**

  - **Set Results Variable**  
    - Type: Set  
    - Role: Stores the stdout output of Nuclei scan in variable `Nuclei Results` for further processing  
    - Inputs: Output from "Remove Templates" (Nuclei command result)  
    - Outputs: To "Check Results"  
    - Edge Cases: Empty or malformed output.

  - **Check Results**  
    - Type: If  
    - Role: Checks if `Nuclei Results` string is not empty before sending notification  
    - Config: Condition tests non-empty string of `Nuclei Results`  
    - Inputs: From "Set Results Variable"  
    - Outputs: True branch to "Send a message", false branch discards  
    - Edge Cases: False negatives if output is whitespace only.

  - **Send a message**  
    - Type: Gmail  
    - Role: Sends email containing Nuclei scan results to configured recipient  
    - Config:  
      - Recipient: `pyus3r@gmail.com`  
      - Subject: `CVE Hunter`  
      - Message body: contents of `Nuclei Results`  
    - Credentials: OAuth2 Gmail credentials required  
    - Inputs: From "Check Results"  
    - Outputs: None (end node)  
    - Edge Cases: Gmail API quota exceeded, OAuth token expiration, invalid recipient.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                               | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                                                                         |
|--------------------------|---------------------|-----------------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule trigger    | Periodic start of the workflow                | None                        | Get All Bug Bounty Domains   |                                                                                                                                                                                     |
| Get All Bug Bounty Domains| HTTP Request       | Fetches list of bug bounty domains             | Schedule Trigger            | Create domains.txt           | ## Step 1 - Get All Bug Bounty Programs Domains                                                                                                                                     |
| Create domains.txt       | Convert To File     | Converts domain list text to file               | Get All Bug Bounty Domains  | Upload domains.txt           | ## Step 1 - Get All Bug Bounty Programs Domains                                                                                                                                     |
| Upload domains.txt       | SSH                 | Uploads domains.txt to remote server            | Create domains.txt          | GET Last CVEs (PROJECT DISCOVERY) | ## Step 1 - Get All Bug Bounty Programs Domains                                                                                                                                     |
| GET Last CVEs (PROJECT DISCOVERY) | HTTP Request | Retrieves latest CVE templates from API         | Upload domains.txt          | Split CVEs                  | ## Step 2 - Get New CVEs Templates                                                                                                                                                   |
| Split CVEs              | Split Out           | Splits API results into individual CVE items   | GET Last CVEs               | Loop Over CVEs              | ## Step 2 - Get New CVEs Templates                                                                                                                                                   |
| Loop Over CVEs          | Split In Batches    | Iterates over CVE items                          | Split CVEs                  | Template Exists Filter, Date Filter | ## Step 2 - Get New CVEs Templates                                                                                                                                                   |
| Template Exists Filter   | Filter              | Filters CVEs where template exists               | Loop Over CVEs              | Loop Over Templates         | ## Step 2 - Get New CVEs Templates                                                                                                                                                   |
| Date Filter             | If                  | Filters CVEs created after last run date        | Loop Over CVEs              | Set Variables, Set Null Variable | ## Step 2 - Get New CVEs Templates                                                                                                                                                   |
| Set Variables           | Set                 | Sets CVE properties into variables               | Date Filter (true)          | Loop Over CVEs              | ## Step 2 - Get New CVEs Templates                                                                                                                                                   |
| Set Null Variable       | Set                 | Sets empty variables when CVE filtered out       | Date Filter (false)         | Loop Over CVEs              | ## Step 2 - Get New CVEs Templates                                                                                                                                                   |
| Loop Over Templates     | Split In Batches    | Iterates over existing templates for processing | Template Exists Filter      | Remove Items, Create Template | ## Step 3 - Create & Execute Templates                                                                                                                                                |
| Remove Items            | Summarize           | Aggregates data before execution                  | Loop Over Templates         | Execute Nuclei              | ## Step 3 - Create & Execute Templates                                                                                                                                                |
| Create Template         | Convert To File     | Converts template JSON to text files               | Loop Over Templates         | Upload Template             | ## Step 3 - Create & Execute Templates                                                                                                                                                |
| Upload Template         | SSH                 | Uploads template files to remote server            | Create Template             | Convert Template to .yaml   | ## Step 3 - Create & Execute Templates                                                                                                                                                |
| Convert Template to .yaml | SSH                 | Renames template files to `.yaml` extension        | Upload Template             | Loop Over Templates         | ## Step 3 - Create & Execute Templates                                                                                                                                                |
| Execute Nuclei          | SSH                 | Runs nuclei scan using uploaded domains and templates | Remove Items                | Remove Templates            | ## Step 3 - Create & Execute Templates                                                                                                                                                |
| Remove Templates        | SSH                 | Deletes template files post scan                    | Execute Nuclei              | Set Results Variable        | ## Step 3 - Create & Execute Templates                                                                                                                                                |
| Set Results Variable    | Set                 | Stores nuclei stdout results                        | Remove Templates            | Check Results               | ## Step 4 - Send Results via Gmail                                                                                                                                                   |
| Check Results           | If                  | Checks if nuclei results are non-empty             | Set Results Variable        | Send a message              | ## Step 4 - Send Results via Gmail                                                                                                                                                   |
| Send a message          | Gmail               | Sends email with nuclei results                      | Check Results               | None                       | ## Step 4 - Send Results via Gmail                                                                                                                                                   |
| Sticky Note             | Sticky Note          | Setup instructions for SSH, OpenAI, Gmail          | None                       | None                       | # SET UP (See "General Notes & Resources" section below)                                                                                                                             |
| Sticky Note1            | Sticky Note          | Step 1 title block: Get Bug Bounty Domains          | None                       | None                       | ## Step 1 - Get All Bug Bounty Programs Domains                                                                                                                                     |
| Sticky Note2            | Sticky Note          | Step 2 title block: Get New CVEs Templates           | None                       | None                       | ## Step 2 - Get New CVEs Templates                                                                                                                                                   |
| Sticky Note3            | Sticky Note          | Step 3 title block: Create & Execute Templates       | None                       | None                       | ## Step 3 - Create & Execute Templates                                                                                                                                                |
| Sticky Note4            | Sticky Note          | Step 4 title block: Send Results via Gmail           | None                       | None                       | ## Step 4 - Send Results via Gmail                                                                                                                                                   |
| Sticky Note8            | Sticky Note          | Setup detailed instructions and links for environment | None                       | None                       | # SET UP (See "General Notes & Resources" section below)                                                                                                                             |
| Sticky Note9            | Sticky Note          | Contact info for support and customization            | None                       | None                       | # 📬 Need Help or Want to Customize This? Contact via LinkedIn/Email                                                                                                                 |
| Sticky Note13           | Sticky Note          | Workflow objective description                         | None                       | None                       | # Objective of the workflow: Runs new Project Discovery templates on bug bounty domains from HackerOne, Bugcrowd, Intigriti, YesWeHack                                            |
| Sticky Note10           | Sticky Note          | Marking INPUT section                                   | None                       | None                       | ## INPUT                                                                                                                                                                             |
| Sticky Note11           | Sticky Note          | Marking CORE section                                    | None                       | None                       | ## CORE                                                                                                                                                                              |
| Sticky Note12           | Sticky Note          | Marking OUTPUT section                                  | None                       | None                       | ## OUTPUT                                                                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Node Type: Schedule Trigger  
   - Configure to execute periodically (default every hour or as needed).

2. **Get All Bug Bounty Domains**  
   - Node Type: HTTP Request  
   - Method: GET  
   - URL: `https://raw.githubusercontent.com/arkadiyt/bounty-targets-data/refs/heads/main/data/domains.txt`  
   - Connect Schedule Trigger output to this node.

3. **Create domains.txt**  
   - Node Type: Convert To File  
   - Operation: toText  
   - Source Property: `data` (text content from HTTP request)  
   - File Name: `domains.txt`  
   - Connect output of "Get All Bug Bounty Domains" to this node.

4. **Upload domains.txt**  
   - Node Type: SSH  
   - Resource: File Upload  
   - Path: `/tmp/nuclei`  
   - Credential: Set SSH Password credential with correct host, user, and authentication (password/private key)  
   - Connect output of "Create domains.txt" to this node.

5. **GET Last CVEs (PROJECT DISCOVERY)**  
   - Node Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.projectdiscovery.io/v2/template/search`  
   - Query Parameters:  
     - scope=public  
     - facet_size=40  
     - offset=0  
     - limit=40  
   - Connect output of "Upload domains.txt" to this node.

6. **Split CVEs**  
   - Node Type: Split Out  
   - Field to Split Out: `results`  
   - Connect output of "GET Last CVEs (PROJECT DISCOVERY)" to this node.

7. **Loop Over CVEs**  
   - Node Type: Split In Batches  
   - Default batch size (can be adjusted)  
   - Connect output of "Split CVEs" to this node.

8. **Template Exists Filter**  
   - Node Type: Filter  
   - Condition: Check if `Template` field exists and is non-empty  
   - Connect the first output of "Loop Over CVEs" to this node.

9. **Date Filter**  
   - Node Type: If  
   - Condition: The `created_at` date (first 10 characters) is after or equal to (Schedule Trigger timestamp - 1 day)  
   - Use expression to parse date and compare  
   - Connect the second output of "Loop Over CVEs" to this node.

10. **Set Variables**  
    - Node Type: Set  
    - Assign:  
      - CVE = `classification["cve-id"][0]`  
      - CVSS-Score = `classification["cvss-score"]`  
      - Template = `raw`  
      - References = `references`  
    - Connect "Date Filter" true output to this node.

11. **Set Null Variable**  
    - Node Type: Set  
    - No assignments (clears or sets nulls)  
    - Connect "Date Filter" false output to this node.

12. **Loop Over Templates**  
    - Node Type: Split In Batches  
    - Connect output of "Template Exists Filter" to this node.

13. **Remove Items**  
    - Node Type: Summarize  
    - Fields to Summarize: append `code` field (if applicable)  
    - Connect first output of "Loop Over Templates" to this node.

14. **Create Template**  
    - Node Type: Convert To File  
    - Operation: toText  
    - Source Property: `Template` variable  
    - File Name: `{{ $json.CVE }}.txt`  
    - Connect second output of "Loop Over Templates" to this node.

15. **Upload Template**  
    - Node Type: SSH  
    - Resource: File Upload  
    - Path: `/tmp/nuclei-templates`  
    - Credentials: Same SSH Password credential as before  
    - Connect output of "Create Template" to this node.

16. **Convert Template to .yaml**  
    - Node Type: SSH  
    - Command: `mv /tmp/nuclei-templates/{{ $json.CVE }}.txt /tmp/nuclei-templates/{{ $json.CVE }}.yaml`  
    - Credentials: Same SSH credential  
    - Connect output of "Upload Template" to this node.

17. **Execute Nuclei**  
    - Node Type: SSH  
    - Command: `nuclei -l /tmp/nuclei/domains.txt -t /tmp/nuclei-templates -ss host-spray -c 10 -bs 50 -rl 100 -timeout 10 -retries 1 -silent -etags info,low`  
    - Credentials: SSH Password credential  
    - Connect output of "Remove Items" to this node.

18. **Remove Templates**  
    - Node Type: SSH  
    - Command: `rm /tmp/nuclei-templates/*`  
    - Credentials: Same SSH credential  
    - Connect output of "Execute Nuclei" to this node.

19. **Set Results Variable**  
    - Node Type: Set  
    - Assignment: `Nuclei Results` = output `stdout` of "Execute Nuclei" node  
    - Connect output of "Remove Templates" to this node.

20. **Check Results**  
    - Node Type: If  
    - Condition: `Nuclei Results` string is not empty  
    - Connect output of "Set Results Variable" to this node.

21. **Send a message**  
    - Node Type: Gmail  
    - Recipient: `pyus3r@gmail.com`  
    - Subject: `CVE Hunter`  
    - Message: `{{ $json["Nuclei Results"] }}`  
    - Credentials: Gmail OAuth2 credentials properly set up with API access  
    - Connect true output of "Check Results" to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| SET UP Instructions for SSH Environment including VPS options, installation of Nuclei, and SSH credential setup in n8n. | See Sticky Note8 content; includes links to Hostinger, DigitalOcean, Hapi Host, WebDedis, and Nuclei installation docs: https://docs.projectdiscovery.io/opensource/nuclei/install |
| Gmail API configuration steps for OAuth2 setup required for the Gmail node. | Google Cloud Console instructions included in Sticky Note8. |
| Workflow Objective: Runs new vulnerability templates released by Project Discovery on aggregated bug bounty scope from HackerOne, Bugcrowd, Intigriti, and YesWeHack. | Sticky Note13 |
| Contact for consulting and support: LinkedIn (https://www.linkedin.com/in/javier-rieiro-2900b5354/), Email: pyus3r@gmail.com | Sticky Note9 |
| Step-wise logical structure visualized by sticky notes marking INPUT, CORE, and OUTPUT blocks. | Sticky Notes10, 11, 12 |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.