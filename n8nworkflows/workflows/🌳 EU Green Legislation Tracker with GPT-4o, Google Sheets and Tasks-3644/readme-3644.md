🌳 EU Green Legislation Tracker with GPT-4o, Google Sheets and Tasks

https://n8nworkflows.xyz/workflows/---eu-green-legislation-tracker-with-gpt-4o--google-sheets-and-tasks-3644


# 🌳 EU Green Legislation Tracker with GPT-4o, Google Sheets and Tasks

### 1. Workflow Overview

This workflow automates the monitoring of new European Union legislative procedures related to environmental sustainability. It scrapes the EU Parliament’s legislative portal for procedures scheduled on a specific day (yesterday by default), uses OpenAI’s GPT-4-turbo model to classify whether each procedure is related to sustainability, filters relevant items, stores them in a Google Sheet, and creates Google Tasks for follow-up review.

The workflow is logically divided into four main blocks:

- **1.1 Data Extraction:** Scraping the EU Parliament legislative portal to extract legislative procedure data.
- **1.2 AI Classification:** Using OpenAI GPT-4-turbo to classify each procedure’s relevance to sustainability.
- **1.3 Filtering and Storage:** Filtering out irrelevant procedures and saving relevant ones into Google Sheets.
- **1.4 Task Creation:** Creating Google Tasks for each sustainability-related procedure to facilitate review.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Extraction

**Overview:**  
This block scrapes the EU Parliament legislative portal for procedures scheduled on a target date (default: yesterday minus 18 days due to portal constraints). It extracts HTML blocks representing each procedure and parses key details such as Reference Number, Committee, Rapporteur, Title/Description, PDF link, and Date.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Extract Yesterday Records (HTTP Request)  
- Extract HTML Blocks (HTML Extract)  
- Parse Blocks (HTML Extract)  
- Edit Links (Set)  
- Loop Over Items (SplitInBatches)  
- Sticky Note1 (Documentation)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution  
  - Configuration: Default, no parameters  
  - Inputs: None  
  - Outputs: Extract Yesterday Records  
  - Edge cases: None; manual trigger only

- **Extract Yesterday Records**  
  - Type: HTTP Request  
  - Role: Fetches the HTML page from the EU Parliament legislative portal filtered by date  
  - Configuration:  
    - URL dynamically constructed with date parameter: `{{$now.minus(18,'days').format('yyyyMMdd')}}` (adjusted to 18 days ago)  
    - Method: GET (default)  
  - Inputs: Manual Trigger  
  - Outputs: Raw HTML content  
  - Edge cases: HTTP errors, network timeouts, portal changes affecting URL or page structure

- **Extract HTML Blocks**  
  - Type: HTML Extract  
  - Role: Extracts HTML blocks representing individual legislative procedures from the fetched page  
  - Configuration:  
    - CSS selector: `.erpl_document-wrapper`  
    - Returns array of HTML strings (each block)  
  - Inputs: HTTP Request output  
  - Outputs: Array of HTML blocks under key `Blocks`  
  - Edge cases: Page structure changes, empty results if no procedures found

- **Parse Blocks**  
  - Type: HTML Extract  
  - Role: Parses each HTML block to extract procedure details  
  - Configuration:  
    - Data property: `Blocks` (array of HTML snippets)  
    - Extracted fields with CSS selectors:  
      - Reference Number: `h3 span.t-item`  
      - Committee: `span.erpl_badge-committee`  
      - Rapporteur: `span.erpl_document-subtitle-author`  
      - Title/Description: `div.erpl_document-body p`  
      - PDF Link: `a.erpl_document-subtitle-pdf` (attribute `href`)  
      - Date: `div.mt-1 p`  
  - Inputs: Extract HTML Blocks output  
  - Outputs: JSON objects with extracted fields  
  - Edge cases: Missing fields, malformed HTML, selector changes

- **Edit Links**  
  - Type: Set  
  - Role: Prepends base URL to relative PDF links to form full URLs  
  - Configuration:  
    - Sets field `PDF Link\t` to `https://oeil.secure.europarl.europa.eu{{ $json['PDF Link\t'] }}`  
    - Includes all other fields unchanged  
  - Inputs: Parse Blocks output  
  - Outputs: Updated JSON with full PDF links  
  - Edge cases: Missing or malformed PDF link fields

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each procedure individually for AI classification  
  - Configuration: Default batch size (1 item per batch)  
  - Inputs: Edit Links output  
  - Outputs: Single procedure JSON per batch  
  - Edge cases: Large number of items may slow processing

- **Sticky Note1**  
  - Type: Sticky Note (Documentation)  
  - Content: Explains the scraping block and setup instructions  
  - No input/output connections

---

#### 1.2 AI Classification

**Overview:**  
This block sends each legislative procedure’s title and committee information to OpenAI GPT-4-turbo to classify whether it relates to sustainability. The AI is instructed to answer strictly “yes” or “no” based on environmental impact criteria.

**Nodes Involved:**  
- Classification Agent (OpenAI)  
- Collect Answer (Set)  
- Merge (Merge)  
- Sticky Note (Documentation)

**Node Details:**

- **Classification Agent**  
  - Type: OpenAI (LangChain Agent)  
  - Role: Sends procedure data to GPT-4-turbo for classification  
  - Configuration:  
    - Model: `gpt-4-turbo`  
    - Messages:  
      - User prompt includes Title and Committee fields, asks for “yes” or “no” answer strictly based on sustainability relevance  
      - System prompt provides sample output format: `{"answer": "yes"}`  
    - JSON output enabled to parse AI response  
  - Inputs: Loop Over Items output (single procedure)  
  - Outputs: AI response JSON with `answer` field  
  - Edge cases: API rate limits, authentication errors, malformed AI responses, timeout, unexpected output format

- **Collect Answer**  
  - Type: Set  
  - Role: Extracts AI answer from response and assigns it to a new field `sustainability`  
  - Configuration:  
    - Sets `sustainability` = `{{$json.message.content.answer}}`  
  - Inputs: Classification Agent output  
  - Outputs: JSON with `sustainability` field added  
  - Edge cases: Missing or malformed AI response content

- **Merge**  
  - Type: Merge  
  - Role: Combines AI classification results with original procedure data  
  - Configuration:  
    - Mode: `combineBySql` (merges based on SQL-like keys)  
  - Inputs:  
    - Main input 1: Original procedure data (from Loop Over Items)  
    - Main input 2: AI classification data (from Collect Answer)  
  - Outputs: Combined JSON with all fields including `sustainability`  
  - Edge cases: Mismatched data, missing keys causing merge failure

- **Sticky Note (Sticky Note2)**  
  - Type: Sticky Note (Documentation)  
  - Content: Explains AI classification block and setup instructions for OpenAI node

---

#### 1.3 Filtering and Storage

**Overview:**  
This block filters procedures classified as sustainability-related and appends them to a Google Sheet for record-keeping.

**Nodes Involved:**  
- If (Conditional)  
- Record Sustainability Procedures (Google Sheets)  
- Sticky Note2 (Documentation)

**Node Details:**

- **If**  
  - Type: If (Conditional)  
  - Role: Filters procedures where `sustainability` field equals `"yes"`  
  - Configuration:  
    - Condition: `{{$json.sustainability}}` equals `"yes"` (case-sensitive, strict)  
  - Inputs: Merge output  
  - Outputs:  
    - True branch: sustainability-related procedures  
    - False branch: non-sustainability procedures  
  - Edge cases: Missing `sustainability` field, unexpected values

- **Record Sustainability Procedures**  
  - Type: Google Sheets  
  - Role: Appends filtered procedures to a Google Sheet  
  - Configuration:  
    - Operation: Append  
    - Document ID: Configured to target the “Sustainability Content” Google Sheet  
    - Sheet Name: `gid=0` (default sheet)  
    - Columns mapped: Reference Number, Committee, Rapporteur, Title/Description, PDF Link, Date  
    - Credentials: Google Sheets OAuth2 API  
  - Inputs: If node’s True output  
  - Outputs: Confirmation of append operation  
  - Edge cases: Authentication errors, quota limits, sheet access issues, mapping errors

- **Sticky Note2**  
  - Type: Sticky Note (Documentation)  
  - Content: Explains filtering and Google Sheets storage setup

---

#### 1.4 Task Creation

**Overview:**  
For each sustainability-related procedure stored, this block creates a Google Task in a specified task list to prompt review of the legislation.

**Nodes Involved:**  
- Google Tasks (Google Tasks)  
- Sticky Note3 (Documentation)

**Node Details:**

- **Google Tasks**  
  - Type: Google Tasks  
  - Role: Creates a task for each relevant legislative procedure  
  - Configuration:  
    - Task List ID: `MTIxODU0NDk4MzM3NzAxMTQ0NzY6MDow` (configured task list)  
    - Title: `Study {{ Reference Number }} - EU Legislation`  
    - Notes: Includes detailed info (Title, Reference Number, Committee, Rapporteur, PDF Link, Date)  
    - Status: `needsAction` (task is pending)  
    - Credentials: Google Tasks OAuth2 API  
  - Inputs: If node’s True output (same as Google Sheets)  
  - Outputs: Confirmation of task creation  
  - Edge cases: Authentication errors, quota limits, invalid task list ID, API errors

- **Sticky Note3**  
  - Type: Sticky Note (Documentation)  
  - Content: Explains Google Tasks creation and setup instructions

---

### 3. Summary Table

| Node Name                 | Node Type               | Functional Role                               | Input Node(s)                  | Output Node(s)                | Sticky Note                                                                                      |
|---------------------------|-------------------------|-----------------------------------------------|-------------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger          | Entry point to start the workflow manually    | None                          | Extract Yesterday Records     |                                                                                                |
| Extract Yesterday Records  | HTTP Request            | Fetches EU Parliament legislative page HTML   | When clicking ‘Test workflow’ | Extract HTML Blocks           |                                                                                                |
| Extract HTML Blocks        | HTML Extract            | Extracts HTML blocks for each legislative procedure | Extract Yesterday Records      | Parse Blocks                 |                                                                                                |
| Parse Blocks              | HTML Extract            | Parses procedure details from HTML blocks      | Extract HTML Blocks            | Edit Links                   |                                                                                                |
| Edit Links                | Set                     | Converts relative PDF links to full URLs       | Parse Blocks                  | Loop Over Items              |                                                                                                |
| Loop Over Items           | SplitInBatches          | Processes each procedure individually          | Edit Links                   | Classification Agent, Merge  |                                                                                                |
| Classification Agent      | OpenAI (LangChain Agent)| Classifies procedures as sustainability-related | Loop Over Items               | Collect Answer               | Sticky Note2: Setup OpenAI node for classification with GPT-4-turbo                            |
| Collect Answer            | Set                     | Extracts AI classification answer              | Classification Agent          | Merge                       |                                                                                                |
| Merge                     | Merge                   | Combines original data with AI classification  | Loop Over Items, Collect Answer | If                         |                                                                                                |
| If                        | If                      | Filters procedures classified as sustainability | Merge                        | Record Sustainability Procedures, Loop Over Items |                                                                                                |
| Record Sustainability Procedures | Google Sheets          | Appends sustainability procedures to Google Sheet | If (True)                   | Google Tasks                | Sticky Note2: Setup Google Sheets node to record filtered procedures                           |
| Google Tasks              | Google Tasks            | Creates Google Tasks for sustainability procedures | Record Sustainability Procedures | Loop Over Items             | Sticky Note3: Setup Google Tasks node to create review tasks                                  |
| Sticky Note1              | Sticky Note             | Documentation for scraping block                | None                         | None                        | Explains scraping block and setup instructions                                                |
| Sticky Note               | Sticky Note             | Documentation for AI classification block       | None                         | None                        | Explains AI classification block and OpenAI node setup                                       |
| Sticky Note2              | Sticky Note             | Documentation for filtering and storage block   | None                         | None                        | Explains filtering and Google Sheets storage setup                                           |
| Sticky Note3              | Sticky Note             | Documentation for task creation block            | None                         | None                        | Explains Google Tasks creation and setup instructions                                        |
| Sticky Note4              | Sticky Note             | Tutorial link and branding                        | None                         | None                        | [🎥 Check My Tutorial](https://www.youtube.com/watch?v=f_nyArpH6kk)                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To manually start the workflow.

2. **Add an HTTP Request node**  
   - Name: `Extract Yesterday Records`  
   - URL: `https://oeil.secure.europarl.europa.eu/oeil/en/search?sessionDay.allDays=false&sessionDay.day={{$now.minus(18,'days').format('yyyyMMdd')}}&sessionDay.type=ALL`  
   - Method: GET  
   - Purpose: Fetch the EU Parliament legislative portal page filtered by date.

3. **Add an HTML Extract node**  
   - Name: `Extract HTML Blocks`  
   - Operation: Extract HTML content  
   - Extraction:  
     - Key: `Blocks`  
     - CSS Selector: `.erpl_document-wrapper`  
     - Return Array: true  
   - Purpose: Extract each legislative procedure block as HTML.

4. **Add another HTML Extract node**  
   - Name: `Parse Blocks`  
   - Operation: Extract HTML content  
   - Data Property Name: `Blocks`  
   - Extraction fields:  
     - Reference Number: `h3 span.t-item`  
     - Committee: `span.erpl_badge-committee`  
     - Rapporteur: `span.erpl_document-subtitle-author`  
     - Title/Description: `div.erpl_document-body p`  
     - PDF Link: `a.erpl_document-subtitle-pdf` (attribute `href`)  
     - Date: `div.mt-1 p`  
   - Purpose: Parse details from each HTML block.

5. **Add a Set node**  
   - Name: `Edit Links`  
   - Add field:  
     - Name: `PDF Link\t`  
     - Value: `https://oeil.secure.europarl.europa.eu{{ $json['PDF Link\t'] }}`  
   - Include all other fields unchanged.  
   - Purpose: Convert relative PDF links to full URLs.

6. **Add a SplitInBatches node**  
   - Name: `Loop Over Items`  
   - Purpose: Process each procedure individually for AI classification.

7. **Add an OpenAI node (LangChain Agent)**  
   - Name: `Classification Agent`  
   - Credentials: Configure OpenAI credentials with GPT-4-turbo access  
   - Model: `gpt-4-turbo`  
   - Messages:  
     - User:  
       ```
       Is the following legislative document related to sustainability? Answer "yes" or "no".

       Title: {{ $json['Title/Description'] }}
       Committee: {{ $json["Committee"] }}

       Be strict: Only answer "yes" if the topic directly impacts environmental or climate sustainability, circular economy, resource conservation, or pollution reduction.
       ```  
     - System:  
       ```
       Sample output:
       {"answer": "yes"}
       ```  
   - JSON Output: Enabled  
   - Purpose: Classify sustainability relevance.

8. **Add a Set node**  
   - Name: `Collect Answer`  
   - Assign field:  
     - Name: `sustainability`  
     - Value: `{{$json.message.content.answer}}`  
   - Purpose: Extract AI answer into a field.

9. **Add a Merge node**  
   - Name: `Merge`  
   - Mode: `combineBySql`  
   - Purpose: Combine original procedure data with AI classification.

10. **Add an If node**  
    - Name: `If`  
    - Condition: `{{$json.sustainability}}` equals `"yes"` (case-sensitive, strict)  
    - Purpose: Filter only sustainability-related procedures.

11. **Add a Google Sheets node**  
    - Name: `Record Sustainability Procedures`  
    - Operation: Append  
    - Document ID: Select or enter your Google Sheet ID for sustainability records  
    - Sheet Name: Select the target sheet (e.g., `gid=0`)  
    - Map columns: Reference Number, Committee, Rapporteur, Title/Description, PDF Link, Date  
    - Credentials: Configure Google Sheets OAuth2 credentials  
    - Purpose: Store filtered procedures.

12. **Add a Google Tasks node**  
    - Name: `Google Tasks`  
    - Task List ID: Enter your Google Task List ID  
    - Title: `Study {{ $json['Reference Number'] }} - EU Legislation`  
    - Notes: Include detailed info (Title, Reference Number, Committee, Rapporteur, PDF Link, Date)  
    - Status: `needsAction`  
    - Credentials: Configure Google Tasks OAuth2 credentials  
    - Purpose: Create review tasks for sustainability procedures.

13. **Connect nodes in order:**  
    - Manual Trigger → Extract Yesterday Records → Extract HTML Blocks → Parse Blocks → Edit Links → Loop Over Items → Classification Agent → Collect Answer → Merge → If  
    - If (True) → Record Sustainability Procedures → Google Tasks  
    - If (False) → Loop Over Items (to continue processing remaining items)

14. **Add Sticky Notes** (optional but recommended) for documentation and setup instructions as per the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is designed for sustainability consultants, NGOs, companies, and policy analysts. | Use case context                                                                                  |
| AI filters are strict; customize the system prompt to adjust classification criteria.            | Customization advice                                                                             |
| Workflow built with n8n version 1.85.4                                                          | Version requirement                                                                             |
| Tutorial video available for setup and usage                                                    | [🎥 Watch My Tutorial](https://www.youtube.com/watch?v=f_nyArpH6kk)                              |
| Founder and author: Samir, Supply Chain Engineer and Data Scientist, LogiGreen Consulting       | [LogiGreen Consulting](https://logi-green.com), [LinkedIn](https://www.linkedin.com/in/samir-saci) |
| Workflow monitors legislative risk related to climate regulations                               | Project purpose                                                                                  |
| Workflow images and branding available at:                                                     | ![Legislative Observatory](https://www.samirsaci.com/content/images/2025/04/image-10.png)        |
|                                                                                                 | ![EU Legislation Workflow](https://www.samirsaci.com/content/images/size/w1000/2025/04/image-8.png) |

---

This structured documentation provides a comprehensive understanding of the workflow’s logic, node-by-node details, reproduction steps, and contextual resources for advanced users and AI agents alike.