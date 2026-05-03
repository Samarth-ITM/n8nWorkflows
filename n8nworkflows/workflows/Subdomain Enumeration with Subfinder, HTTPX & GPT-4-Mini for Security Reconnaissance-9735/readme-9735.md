Subdomain Enumeration with Subfinder, HTTPX & GPT-4-Mini for Security Reconnaissance

https://n8nworkflows.xyz/workflows/subdomain-enumeration-with-subfinder--httpx---gpt-4-mini-for-security-reconnaissance-9735


# Subdomain Enumeration with Subfinder, HTTPX & GPT-4-Mini for Security Reconnaissance

### 1. Workflow Overview

This workflow automates comprehensive subdomain enumeration for security reconnaissance by integrating multiple passive reconnaissance tools and an AI-powered generator to maximize subdomain discovery. It is designed for use in authorized security assessments and bug bounty programs to uncover hidden and difficult-to-detect subdomains, extending traditional enumeration techniques with intelligent AI-driven suggestions.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Preparation:** Receives a plaintext file with target domains, preprocesses the input, and prepares domain lists for scanning.
- **1.2 Passive Subdomain Enumeration:** Executes multiple passive reconnaissance tools (Subfinder, Assetfinder, crt.sh, WayBack Machine) over SSH to gather subdomains.
- **1.3 Results Aggregation and Validation:** Merges results from passive tools, validates subdomain format, and deduplicates entries.
- **1.4 Active Scanning with HTTPX:** Uses HTTPX over SSH to detect active subdomains and technology stacks by scanning enumerated subdomains.
- **1.5 AI-Driven Subdomain Generation:** Utilizes GPT-4 Mini to generate up to 5,000 unique and plausible subdomains based on discovered patterns, followed by validation and active scanning.
- **1.6 Final Results Compilation and Response:** Merges AI-generated and passive results, formats them for output, and responds to the initial webhook request.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception and Preparation

**Overview:**  
This block handles the initial reception of the input file containing target domains via a webhook, converts the file content into JSON, and formats each line (domain) into structured items for processing.

**Nodes Involved:**  
- Webhook  
- TXT to JSON  
- Prettify File Content  
- Loop Over Domains  

**Node Details:**

- **Webhook**  
  - Type: HTTP Webhook  
  - Role: Receives POST requests with a `.txt` file containing target domains.  
  - Configuration: Expects binary file under property "File". Responds with processed data.  
  - Connections: Output to "TXT to JSON".  
  - Edge Cases: Invalid or missing file input, malformed request.

- **TXT to JSON**  
  - Type: ExtractFromFile  
  - Role: Converts the text file content into JSON format for processing.  
  - Configuration: Reads from binary property "File0".  
  - Connections: Output to "Prettify File Content".  
  - Edge Cases: File encoding issues, empty files.

- **Prettify File Content**  
  - Type: Code  
  - Role: Splits file content by lines, trims whitespace, filters empty lines, and outputs each domain as a separate JSON item.  
  - Key Expression: Splits on line breaks, trims lines.  
  - Connections: Output to "Loop Over Domains".  
  - Edge Cases: Lines with whitespace only, empty input.

- **Loop Over Domains**  
  - Type: SplitInBatches  
  - Role: Processes domains one by one or in batches to enable parallel or sequential execution of enumeration tools.  
  - Connections: Outputs to four passive enumeration nodes and to "Append all Loop Items".  
  - Edge Cases: Large input file causing high batch count.

---

#### 1.2 Passive Subdomain Enumeration

**Overview:**  
Executes four passive subdomain enumeration tools over SSH or HTTP to gather existing subdomains for each target domain.

**Nodes Involved:**  
- SSH Subfinder  
- SSH Assetfinder  
- crt.sh (HTTP Request)  
- WayBack Machine (HTTP Request)  
- Filter Subdomains Subfinder (Code)  
- Filter Subdomains Assetfinder (Code)  
- Filter Subdomains WayBack Machine (Code)  
- Rename crt.sh Content (Set)  
- Summarize  

**Node Details:**

- **SSH Subfinder**  
  - Type: SSH Command  
  - Role: Runs `subfinder -d <domain> -silent -all` remotely to find subdomains.  
  - Config: Uses SSH credentials with password authentication.  
  - Connections: Output to "Filter Subdomains Subfinder".  
  - Edge Cases: SSH connection failures, subfinder command failure.

- **Filter Subdomains Subfinder**  
  - Type: Code  
  - Role: Parses subfinder stdout output, splits by newline, trims, and returns unique subdomains.  
  - Connections: Output to "Merge all Results".  
  - Edge Cases: Empty or malformed output.

- **SSH Assetfinder**  
  - Type: SSH Command  
  - Role: Runs `assetfinder -subs-only <domain>` remotely to get subdomains.  
  - Connections: Output to "Filter Subdomains Assetfinder".  
  - Edge Cases: SSH or command failures.

- **Filter Subdomains Assetfinder**  
  - Type: Code  
  - Role: Similar to Subfinder filter, splits stdout into trimmed array of subdomains.  
  - Connections: Output to "Merge all Results".  

- **crt.sh**  
  - Type: HTTP Request  
  - Role: Queries crt.sh for certificate transparency logs to find subdomains.  
  - Config: URL parameterized with domain, expects JSON output.  
  - Connections: Output to "Summarize".  
  - Edge Cases: HTTP errors, empty or invalid JSON response.

- **Summarize**  
  - Type: Summarize  
  - Role: Aggregates the `name_value` field from crt.sh JSON array into a single list.  
  - Connections: Output to "Rename crt.sh Content".

- **Rename crt.sh Content**  
  - Type: Set  
  - Role: Assigns summarized crt.sh names to a `subdomains` array for downstream merging.  
  - Connections: Output to "Merge all Results".

- **WayBack Machine**  
  - Type: HTTP Request  
  - Role: Queries Internet Archive CDX API for archived URLs under the domain, extracting subdomains.  
  - Connections: Output to "Filter Subdomains WayBack Machine".  

- **Filter Subdomains WayBack Machine**  
  - Type: Code  
  - Role: Parses returned URL list, extracts subdomains, removes duplicates.  
  - Connections: Output to "Merge all Results".

---

#### 1.3 Results Aggregation and Validation

**Overview:**  
Merges all passive enumeration results, deduplicates subdomains, validates format, and prepares a clean list for further use.

**Nodes Involved:**  
- Merge all Results  
- Unique Subdomains  
- Append all Loop Items  
- Subdomain Format Validation  
- Convert Subdomains to TXT  

**Node Details:**

- **Merge all Results**  
  - Type: Merge  
  - Role: Merges outputs from Subfinder, Assetfinder, WayBack Machine, and crt.sh filters.  
  - Config: Number of inputs = 4.  
  - Connections: Output to "Unique Subdomains".

- **Unique Subdomains**  
  - Type: Code  
  - Role: Collects all subdomains from merged input, flattening and deduplicating them.  
  - Connections: Output to "Loop Over Domains" for next passive enumeration cycle (looped scanning).

- **Append all Loop Items**  
  - Type: Code  
  - Role: Flattens and deduplicates all subdomains collected from loops for final aggregation.  
  - Connections: Output to "Subdomain Format Validation".

- **Subdomain Format Validation**  
  - Type: Code  
  - Role: Validates subdomains against regex ensuring valid format (no leading asterisks, valid domain syntax), removes ports, and outputs newline-separated list.  
  - Connections: Output to "Convert Subdomains to TXT".  

- **Convert Subdomains to TXT**  
  - Type: ConvertToFile  
  - Role: Converts validated subdomain list text into a `.txt` file for SSH upload.  
  - Config: Filename `subdomains.txt`, source property `subdomain`.  
  - Connections: Output to "SSH create subdomains.txt into /tmp/reconAI".  

---

#### 1.4 Active Scanning with HTTPX

**Overview:**  
Uploads the TXT file of subdomains to the remote server and runs HTTPX to probe for live hosts and detect technologies.

**Nodes Involved:**  
- SSH create subdomains.txt into /tmp/reconAI  
- SSH HTTPX  
- Rename Passive Results  
- Split HTTPX Results (400 Items max)  
- Loop Over 30 Subdomains  
- Append 30 Subdomains  
- Subdomain Generator AI  

**Node Details:**

- **SSH create subdomains.txt into /tmp/reconAI**  
  - Type: SSH (file upload)  
  - Role: Creates or overwrites `subdomains.txt` on remote server path `/tmp/reconAI`.  
  - Connections: Output to "SSH HTTPX".  
  - Edge Cases: SSH upload failure, permission issues.

- **SSH HTTPX**  
  - Type: SSH Command  
  - Role: Runs `httpx` on remote server with parameters: silent, tech detection, input from subdomains.txt, no color, 50 threads, timeout 4s, retries 1.  
  - Connections: Output to "Rename Passive Results".  
  - Edge Cases: SSH or command failure, timeout.

- **Rename Passive Results**  
  - Type: Set  
  - Role: Renames `stdout` from SSH HTTPX command to `Passive Results` string property for downstream processing.  
  - Connections: Outputs to "Split HTTPX Results (400 Items max)" and "Merge AI & Passive Results".

- **Split HTTPX Results (400 Items max)**  
  - Type: Code  
  - Role: Parses Passive Results string by lines, extracts URLs matching `https://<hostname>`, filters, limits to 400 items max to control batch size.  
  - Connections: Output to "Loop Over 30 Subdomains".

- **Loop Over 30 Subdomains**  
  - Type: SplitInBatches (batch size 30)  
  - Role: Processes subdomains in batches of 30 for AI subdomain generation.  
  - Connections: Outputs to "Split AI Response" and "Append 30 Subdomains".

- **Append 30 Subdomains**  
  - Type: Summarize  
  - Role: Aggregates 30 subdomains from batch into a concatenated string to feed into the AI generator.  
  - Connections: Output to "Subdomain Generator AI".

---

#### 1.5 AI-Driven Subdomain Generation

**Overview:**  
Uses GPT-4 Mini model to analyze discovered subdomains and generate up to 5,000 plausible additional subdomains not found by passive tools, enhancing coverage.

**Nodes Involved:**  
- Subdomain Generator AI  
- Split AI Response  
- AI Subdomain Format Validation  
- AI Convert Subdomains to TXT  
- AI SSH create subdomains.txt into /tmp/reconAI  
- AI SSH HTTPX  
- Rename AI Results  
- Merge AI & Passive Results  

**Node Details:**

- **Subdomain Generator AI**  
  - Type: OpenAI (Langchain)  
  - Role: Sends concatenated subdomains as prompt content to GPT-4 Mini with system instructions to generate 5,000 unique plausible subdomains in JSON format.  
  - Config: Model `gpt-4.1-mini`, single completion, JSON output enabled.  
  - Connections: Output to "Split AI Response".  
  - Edge Cases: API rate limits, unexpected AI output format.

- **Split AI Response**  
  - Type: SplitOut  
  - Role: Extracts the `message.content.subdomains` array from AI response for individual processing.  
  - Connections: Output to "AI Subdomain Format Validation".

- **AI Subdomain Format Validation**  
  - Type: Code  
  - Role: Validates AI-generated subdomains similar to earlier validation, removes duplicates, outputs newline-separated list.  
  - Connections: Output to "AI Convert Subdomains to TXT".

- **AI Convert Subdomains to TXT**  
  - Type: ConvertToFile  
  - Role: Converts AI-generated valid subdomains to `subdomains.txt` file for SSH upload.  
  - Connections: Output to "AI SSH create subdomains.txt into /tmp/reconAI".

- **AI SSH create subdomains.txt into /tmp/reconAI**  
  - Type: SSH (file upload)  
  - Role: Uploads AI-generated subdomains file to remote server location.  
  - Connections: Output to "AI SSH HTTPX".

- **AI SSH HTTPX**  
  - Type: SSH Command  
  - Role: Runs HTTPX on AI-generated subdomains to check for active hosts and tech detection.  
  - Connections: Output to "Rename AI Results".  
  - Edge Cases: SSH or command failure.

- **Rename AI Results**  
  - Type: Set  
  - Role: Renames SSH output to `AI Results` string property.  
  - Connections: Output to "Merge AI & Passive Results".

- **Merge AI & Passive Results**  
  - Type: Merge  
  - Role: Combines passive HTTPX results (`Passive Results`) and AI HTTPX results (`AI Results`) to form final dataset.  
  - Connections: Output to "Prettify Results".

---

#### 1.6 Final Results Compilation and Response

**Overview:**  
Prepares the final output by parsing, cleaning, and formatting all results, then sends the response back to the webhook caller.

**Nodes Involved:**  
- Prettify Results  
- Respond to Webhook  

**Node Details:**

- **Prettify Results**  
  - Type: Code  
  - Role: Parses `Passive Results` and `AI Results` strings, splits into arrays, trims entries, removes empty lines, combines into structured JSON.  
  - Connections: Output to "Respond to Webhook".

- **Respond to Webhook**  
  - Type: RespondToWebhook  
  - Role: Sends the combined final results as HTTP response to the original webhook request.  
  - Configuration: Responds with all incoming items.  
  - Edge Cases: Response size limits, network issues.

---

### 3. Summary Table

| Node Name                           | Node Type                  | Functional Role                                      | Input Node(s)                         | Output Node(s)                                  | Sticky Note                                                                                          |
|-----------------------------------|----------------------------|-----------------------------------------------------|-------------------------------------|------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook                           | Webhook                    | Receives input file                                 | -                                   | TXT to JSON                                    | See HOW USE instructions for file format and request examples                                      |
| TXT to JSON                      | ExtractFromFile            | Converts input text file to JSON                    | Webhook                             | Prettify File Content                           |                                                                                                    |
| Prettify File Content            | Code                       | Splits file content into domain lines              | TXT to JSON                        | Loop Over Domains                               |                                                                                                    |
| Loop Over Domains                | SplitInBatches             | Batch processes each domain for scanning            | Prettify File Content              | SSH Subfinder, SSH Assetfinder, crt.sh, WayBack Machine, Append all Loop Items |                                                                                                    |
| SSH Subfinder                   | SSH                        | Runs subfinder tool over SSH                         | Loop Over Domains                  | Filter Subdomains Subfinder                      |                                                                                                    |
| Filter Subdomains Subfinder       | Code                       | Parses subfinder output                              | SSH Subfinder                     | Merge all Results                               |                                                                                                    |
| SSH Assetfinder                 | SSH                        | Runs assetfinder tool over SSH                       | Loop Over Domains                  | Filter Subdomains Assetfinder                    |                                                                                                    |
| Filter Subdomains Assetfinder     | Code                       | Parses assetfinder output                            | SSH Assetfinder                   | Merge all Results                               |                                                                                                    |
| crt.sh                         | HTTP Request               | Queries crt.sh for subdomains                        | Loop Over Domains                  | Summarize                                       |                                                                                                    |
| Summarize                       | Summarize                  | Aggregates crt.sh `name_value` array                | crt.sh                            | Rename crt.sh Content                           |                                                                                                    |
| Rename crt.sh Content            | Set                        | Assigns crt.sh subdomains to unified array          | Summarize                        | Merge all Results                               |                                                                                                    |
| WayBack Machine                 | HTTP Request               | Queries WayBack Machine archive URLs                 | Loop Over Domains                  | Filter Subdomains WayBack Machine               |                                                                                                    |
| Filter Subdomains WayBack Machine | Code                       | Extracts and deduplicates subdomains from WayBack   | WayBack Machine                  | Merge all Results                               |                                                                                                    |
| Merge all Results               | Merge                      | Combines subdomains from all passive tools          | Filters from Subfinder, Assetfinder, WayBack, crt.sh | Unique Subdomains                               |                                                                                                    |
| Unique Subdomains               | Code                       | Deduplicates merged subdomains                        | Merge all Results                 | Loop Over Domains (loop for further processing) |                                                                                                    |
| Append all Loop Items           | Code                       | Flattens and deduplicates all collected subdomains  | Loop Over Domains                | Subdomain Format Validation                     |                                                                                                    |
| Subdomain Format Validation     | Code                       | Validates subdomain format, removes ports            | Append all Loop Items             | Convert Subdomains to TXT                        |                                                                                                    |
| Convert Subdomains to TXT       | ConvertToFile              | Converts validated subdomains to text file           | Subdomain Format Validation       | SSH create subdomains.txt into /tmp/reconAI     |                                                                                                    |
| SSH create subdomains.txt into /tmp/reconAI | SSH (file upload)         | Uploads subdomains.txt to remote server              | Convert Subdomains to TXT         | SSH HTTPX                                       |                                                                                                    |
| SSH HTTPX                      | SSH                        | Runs HTTPX tool on uploaded subdomains               | SSH create subdomains.txt into /tmp/reconAI | Rename Passive Results                          |                                                                                                    |
| Rename Passive Results          | Set                        | Renames HTTPX stdout to Passive Results              | SSH HTTPX                       | Split HTTPX Results (400 Items max), Merge AI & Passive Results |                                                                                                    |
| Split HTTPX Results (400 Items max) | Code                       | Parses Passive Results, limits to 400 items          | Rename Passive Results           | Loop Over 30 Subdomains                         |                                                                                                    |
| Loop Over 30 Subdomains         | SplitInBatches             | Processes 30 subdomains per batch                     | Split HTTPX Results (400 Items max), Split AI Response | Append 30 Subdomains, Split AI Response         |                                                                                                    |
| Append 30 Subdomains            | Summarize                  | Aggregates 30 subdomains into string                  | Loop Over 30 Subdomains          | Subdomain Generator AI                          |                                                                                                    |
| Subdomain Generator AI          | OpenAI (Langchain)         | Generates 5,000 plausible subdomains via GPT-4 Mini | Append 30 Subdomains             | Split AI Response                               |                                                                                                    |
| Split AI Response               | SplitOut                   | Extracts subdomains array from AI response            | Subdomain Generator AI           | AI Subdomain Format Validation                  |                                                                                                    |
| AI Subdomain Format Validation  | Code                       | Validates and deduplicates AI-generated subdomains   | Split AI Response                | AI Convert Subdomains to TXT                     |                                                                                                    |
| AI Convert Subdomains to TXT    | ConvertToFile              | Converts AI subdomains to text file                    | AI Subdomain Format Validation   | AI SSH create subdomains.txt into /tmp/reconAI  |                                                                                                    |
| AI SSH create subdomains.txt into /tmp/reconAI | SSH (file upload)         | Uploads AI-generated subdomains file to remote server | AI Convert Subdomains to TXT     | AI SSH HTTPX                                    |                                                                                                    |
| AI SSH HTTPX                   | SSH                        | Runs HTTPX scan on AI-generated subdomains            | AI SSH create subdomains.txt into /tmp/reconAI | Rename AI Results                               |                                                                                                    |
| Rename AI Results              | Set                        | Renames AI HTTPX stdout to AI Results                  | AI SSH HTTPX                    | Merge AI & Passive Results                       |                                                                                                    |
| Merge AI & Passive Results     | Merge                      | Combines AI and passive HTTPX scan results             | Rename Passive Results, Rename AI Results | Prettify Results                                |                                                                                                    |
| Prettify Results               | Code                       | Parses and cleans combined results for output          | Merge AI & Passive Results       | Respond to Webhook                              |                                                                                                    |
| Respond to Webhook             | RespondToWebhook           | Sends final results back to initial requester          | Prettify Results                | -                                              |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook (POST)  
   - Path: Unique webhook ID (e.g., `e0a62a2b-6669-4f09-bd16-e548ae1386cb`)  
   - Expect binary file input in property "File".

2. **Add ExtractFromFile Node (TXT to JSON)**  
   - Operation: text extraction from binary property `File0`.  
   - Connect Webhook output to this node.

3. **Add Code Node (Prettify File Content)**  
   - JavaScript to split file text into trimmed lines, filter empty lines, output each domain as separate JSON.  
   - Connect TXT to JSON output here.

4. **Add SplitInBatches Node (Loop Over Domains)**  
   - No batch size limit set (process one domain per execution).  
   - Connect Prettify File Content output here.

5. **Add SSH Nodes for Passive Tools (Subfinder and Assetfinder)**  
   - Configure SSH credentials with password or private key.  
   - Subfinder command: `subfinder -d {{ $json.domain }} -silent -all`  
   - Assetfinder command: `assetfinder -subs-only {{ $json.domain }}`  
   - Connect Loop Over Domains to each SSH node.

6. **Add HTTP Request Nodes for crt.sh and WayBack Machine**  
   - crt.sh URL: `https://crt.sh/?q=%25.{{ $json.domain }}&output=json`  
   - WayBack Machine URL: `https://web.archive.org/cdx/search/cdx?url=*.{{ $json.domain }}/*&fl=original&collapse=urlkey`  
   - Connect Loop Over Domains output here as well.

7. **Add Code Nodes to Filter Subdomains from SSH and HTTP Results**  
   - For Subfinder and Assetfinder: parse stdout, split by newline, trim, filter empty.  
   - For WayBack Machine: parse response text, extract hostname from URLs, deduplicate.

8. **Add Summarize Node**  
   - For crt.sh, aggregate `name_value` fields.

9. **Add Set Node to assign summarized crt.sh content to `subdomains` array.**

10. **Add Merge Node (4 inputs) to combine all filtered subdomain lists.**

11. **Add Code Node (Unique Subdomains)**  
    - Flatten all merged inputs, extract subdomains, deduplicate.

12. **Connect Unique Subdomains back to Loop Over Domains to enable iterative scanning.**

13. **Add Code Node (Append all Loop Items)**  
    - Flatten and deduplicate all collected subdomains from loops.

14. **Add Code Node (Subdomain Format Validation)**  
    - Validate subdomains with regex (no starting asterisk, valid domain format), remove ports.

15. **Add ConvertToFile Node**  
    - Convert validated subdomains (newline-separated string) to text file named `subdomains.txt`.

16. **Add SSH Node to upload `subdomains.txt` to `/tmp/reconAI` on remote server.**

17. **Add SSH Node to run HTTPX command:**  
    - `httpx -silent -tech-detect -l /tmp/reconAI/subdomains.txt -no-color -threads 50 -timeout 4 -retries 1`  

18. **Add Set Node to rename SSH HTTPX stdout to `Passive Results`.**

19. **Add Code Node (Split HTTPX Results)**  
    - Parse `Passive Results` string, extract URLs, limit to 400 items.

20. **Add SplitInBatches Node (Loop Over 30 Subdomains)**  
    - Batch size: 30.

21. **Add Summarize Node (Append 30 Subdomains)**  
    - Aggregate batch subdomain strings.

22. **Add OpenAI Node (Subdomain Generator AI)**  
    - Model: GPT-4 Mini (`gpt-4.1-mini`)  
    - Prompt: system role and task instructions to generate 5,000 unique plausible subdomains based on input.  
    - Input: concatenated subdomains from previous summarize node.

23. **Add SplitOut Node (Split AI Response)**  
    - Extract AI-generated subdomain list from JSON key.

24. **Add Code Node (AI Subdomain Format Validation)**  
    - Similar validation and deduplication as before.

25. **Add ConvertToFile Node (AI Convert Subdomains to TXT)**  
    - Convert validated AI subdomains to `subdomains.txt`.

26. **Add SSH Node (AI SSH create subdomains.txt)**  
    - Upload AI generated file to `/tmp/reconAI`.

27. **Add SSH Node (AI SSH HTTPX)**  
    - Run HTTPX as before on AI-generated subdomains.

28. **Add Set Node (Rename AI Results)**  
    - Rename SSH output to `AI Results`.

29. **Add Merge Node (Merge AI & Passive Results)**  
    - Combine `Passive Results` and `AI Results`.

30. **Add Code Node (Prettify Results)**  
    - Parse both results strings into arrays, trim, clean.

31. **Add RespondToWebhook Node**  
    - Send final combined results as response to the initial webhook.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                       | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| HOW TO USE: Input file must be a plain text file (`.txt`) with one domain per line. Example format provided. Use curl or Python requests to POST the file to the webhook URL.                                                                                                                       | Sticky Note on HOW USE instructions                                                                 |
| SET UP SSH ENVIRONMENT: Guidance on VPS providers, installing subfinder, assetfinder, httpx, and configuring SSH credentials in n8n. Also includes local SSH server setup instructions.                                                                                                           | Sticky Note with URLs: Hostinger, DigitalOcean, Hapi Host, WebDedis, Subfinder, Assetfinder, HTTPX  |
| OPENAI SETUP: Instructions to obtain API key, add billing funds, and configure OpenAI credentials in n8n.                                                                                                                                                                                       | Sticky Note with links to OpenAI API Keys and Billing pages                                         |
| WORKFLOW OBJECTIVE: Automates passive reconnaissance and generates up to 20,000 additional subdomains using AI, enhancing bug bounty program coverage by detecting attack surface areas missed by traditional tools.                                                                           | Sticky Note describing objective                                                                     |
| COST ESTIMATE (GPT-4-Mini): Approximate token costs for small, medium, and large scopes, useful for budgeting API usage.                                                                                                                                                                        | Sticky Note with cost estimates                                                                      |
| SUPPORT CONTACT: For consulting and customization help, contact via LinkedIn or email.                                                                                                                                                                                                           | Sticky Note with contact info: [LinkedIn](https://www.linkedin.com/in/javier-rieiro-2900b5354/), pyus3r@gmail.com |

---

_Disclaimer: This documentation is based exclusively on an automated n8n workflow for authorized security reconnaissance. It respects all applicable content policies and contains no illegal or protected elements. All data processed is legal and publicly available._