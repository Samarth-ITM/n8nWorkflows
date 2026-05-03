Parse & Analyze Research Papers with PDF Vector, GPT-4 and Database Storage

https://n8nworkflows.xyz/workflows/parse---analyze-research-papers-with-pdf-vector--gpt-4-and-database-storage-7351


# Parse & Analyze Research Papers with PDF Vector, GPT-4 and Database Storage

### 1. Workflow Overview

This workflow automates the parsing and analysis of research papers in PDF format by leveraging AI and database storage. It is designed for researchers, analysts, or knowledge managers who want to extract structured insights from academic papers and store them for further use. The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual initiation of the workflow with a PDF URL input.
- **1.2 Document Parsing:** Parsing the PDF research paper into clean Markdown text using PDF Vector.
- **1.3 AI Analysis:** Using OpenAI’s GPT-4 model to analyze the parsed content and extract key research elements.
- **1.4 Data Persistence:** Inserting the extracted insights and metadata into a PostgreSQL database for storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow manually, allowing a user to start the process by supplying or defining the PDF URL externally.

- **Nodes Involved:**  
  - Manual Trigger

- **Node Details:**  
  - **Manual Trigger**  
    - Type: Manual Trigger node  
    - Configuration: No parameters needed; it simply fires the workflow on manual start or can be replaced by a webhook for automation.  
    - Key variables: Expects that the input JSON includes `pdfUrl` which provides the PDF document URL.  
    - Input connections: None (start node)  
    - Output connections: Connected to "PDF Vector - Parse Paper" node  
    - Edge cases: If no `pdfUrl` is provided in the input, subsequent nodes will fail or produce empty results. No validation is performed here.

#### 2.2 Document Parsing

- **Overview:**  
  This block uses the PDF Vector node to parse the input research paper PDF from its URL into clean Markdown text, preparing it for AI analysis.

- **Nodes Involved:**  
  - PDF Vector - Parse Paper

- **Node Details:**  
  - **PDF Vector - Parse Paper**  
    - Type: PDF Vector node specialized for document parsing  
    - Configuration:  
      - Operation: 'parse'  
      - Resource: 'document'  
      - Document URL: taken dynamically from `{{$json.pdfUrl}}` (input JSON field)  
      - Use LLM: 'auto' (lets the node decide if an LLM is needed for parsing)  
    - Key variables: `{{$json.pdfUrl}}` for input, outputs `content` field with parsed Markdown text  
    - Input connections: Manual Trigger node  
    - Output connections: OpenAI - Analyze Paper  
    - Edge cases:  
      - Invalid or inaccessible PDF URL leads to failure or empty output.  
      - Parsing errors if PDF is scanned image without OCR or malformed.  
      - Network timeouts when fetching the PDF.  
    - Version-specific: none reported

#### 2.3 AI Analysis

- **Overview:**  
  This block sends the parsed Markdown content to OpenAI’s GPT-4 to extract structured insights such as research question, methodology, findings, and suggestions.

- **Nodes Involved:**  
  - OpenAI - Analyze Paper

- **Node Details:**  
  - **OpenAI - Analyze Paper**  
    - Type: OpenAI node (GPT-4 model)  
    - Configuration:  
      - Model: `gpt-4`  
      - Messages: A system prompt framed as an instruction to analyze the paper and extract six key sections (research question, methodology, key findings, conclusions, limitations, future work suggestions), with the paper content interpolated into the prompt via `{{$json.content}}`.  
    - Key expressions:  
      - The prompt uses `{{ $json.content }}` to pass the parsed Markdown content from the previous node.  
    - Input connections: PDF Vector - Parse Paper  
    - Output connections: Store Analysis  
    - Edge cases:  
      - If `content` is empty or malformed, AI output may be irrelevant or fail.  
      - API rate limits or authentication errors with OpenAI.  
      - Prompt length exceeding model limits can cause truncation or errors.  
    - Credential requirements: OpenAI API key configured in n8n credentials.

#### 2.4 Data Persistence

- **Overview:**  
  This block inserts the structured analysis results and metadata into a PostgreSQL database for record keeping and further querying.

- **Nodes Involved:**  
  - Store Analysis

- **Node Details:**  
  - **Store Analysis**  
    - Type: PostgreSQL node  
    - Configuration:  
      - Operation: Insert  
      - Table: `research_papers`  
      - Columns: `title, summary, methodology, findings, url, analyzed_at`  
      - Data: Mapped from AI analysis output and input metadata (mapping details assumed but not explicitly shown in JSON)  
    - Input connections: OpenAI - Analyze Paper  
    - Output connections: None (end of workflow)  
    - Edge cases:  
      - Database connection or authentication failures.  
      - Column data mismatches or missing fields from AI analysis.  
      - Duplicate entries if not handled externally.  
    - Credential requirements: PostgreSQL database credentials configured in n8n.

---

### 3. Summary Table

| Node Name              | Node Type               | Functional Role                  | Input Node(s)           | Output Node(s)           | Sticky Note                                      |
|------------------------|-------------------------|---------------------------------|------------------------|--------------------------|-------------------------------------------------|
| Manual Trigger         | Manual Trigger          | Start workflow manually          | None                   | PDF Vector - Parse Paper  | Start the workflow manually or replace with webhook trigger |
| PDF Vector - Parse Paper | PDF Vector             | Parse PDF into Markdown          | Manual Trigger         | OpenAI - Analyze Paper    | Parse the research paper into clean Markdown format |
| OpenAI - Analyze Paper | OpenAI (GPT-4)          | Extract key insights using AI    | PDF Vector - Parse Paper | Store Analysis            | Use AI to extract key insights from the paper    |
| Store Analysis         | PostgreSQL              | Store results in database        | OpenAI - Analyze Paper | None                     | Store the analysis results in your database      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a Manual Trigger node to start the workflow manually.  
   - No parameters needed.  
   - This node will expect an input JSON with a field `pdfUrl` containing the URL of the PDF document.

2. **Add PDF Vector Node for Parsing**  
   - Add a PDF Vector node, name it `"PDF Vector - Parse Paper"`.  
   - Set Resource to `document`.  
   - Set Operation to `parse`.  
   - Set Document URL to use the expression: `{{$json.pdfUrl}}`.  
   - Set Use LLM to `auto`.  
   - Connect the Manual Trigger’s output to this node’s input.

3. **Add OpenAI Node for Analysis**  
   - Add an OpenAI node, name it `"OpenAI - Analyze Paper"`.  
   - Select model: `gpt-4`.  
   - In the Messages parameter, add a system/user message with the following content (use expression to insert paper content):  
     ```
     You are a research paper analyst. Analyze the following paper and extract:
     1. Main research question
     2. Methodology
     3. Key findings
     4. Conclusions
     5. Limitations
     6. Future work suggestions

     Paper content:
     {{ $json.content }}
     ```
   - Connect the output of the PDF Vector node to this OpenAI node.

4. **Add PostgreSQL Node to Store Results**  
   - Add a PostgreSQL node, name it `"Store Analysis"`.  
   - Set Operation to `Insert`.  
   - Set Table to `research_papers`.  
   - Set Columns to `title, summary, methodology, findings, url, analyzed_at`.  
   - Map the columns to corresponding data from the OpenAI analysis output and input PDF URL (this mapping must be defined explicitly based on the AI output structure).  
   - Connect the output of the OpenAI node to this PostgreSQL node.

5. **Configure Credentials**  
   - For OpenAI node: create or select OpenAI credentials with valid API key allowing GPT-4 usage.  
   - For PostgreSQL node: configure database connection credentials (host, user, password, database) matching your PostgreSQL instance.

6. **Validate and Test**  
   - Test the workflow by manually triggering it with a JSON input containing a valid `pdfUrl` key.  
   - Confirm each node runs without error and that database entries are created as expected.

---

### 5. General Notes & Resources

| Note Content                                                            | Context or Link                                               |
|-------------------------------------------------------------------------|---------------------------------------------------------------|
| Replace Manual Trigger with a webhook for automated ingestion workflows | Manual Trigger sticky note                                    |
| Use OpenAI GPT-4 for advanced NLP tasks and ensure API key has access   | OpenAI node requirements                                      |
| PostgreSQL table `research_papers` must exist with appropriate columns   | Database node setup; ensure schema compatibility              |
| Parsing may fail on scanned PDFs without OCR - consider preprocessing    | PDF Vector limitations                                         |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.