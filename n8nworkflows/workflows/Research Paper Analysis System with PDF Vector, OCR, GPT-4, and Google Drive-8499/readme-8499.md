Research Paper Analysis System with PDF Vector, OCR, GPT-4, and Google Drive

https://n8nworkflows.xyz/workflows/research-paper-analysis-system-with-pdf-vector--ocr--gpt-4--and-google-drive-8499


# Research Paper Analysis System with PDF Vector, OCR, GPT-4, and Google Drive

### 1. Workflow Overview

This workflow, titled **Research Paper Analysis System with PDF Vector, OCR, GPT-4, and Google Drive**, automates the end-to-end process of academic paper analysis. It is designed to support researchers and academic professionals by searching, retrieving, parsing, extracting, analyzing, and storing structured insights from research papers.

The workflow is logically divided into the following key blocks:

- **1.1 Input Reception:** Manual initiation of the workflow and retrieval of the research paper PDF from Google Drive.
- **1.2 Paper Parsing and Data Extraction:** Using PDF Vector nodes and OCR capabilities to parse the PDF file and extract structured metadata and content.
- **1.3 AI Processing:** Utilizing OpenAI's GPT-4 to generate detailed summaries and insights based on the extracted content.
- **1.4 Data Preparation and Storage:** Combining all extracted and generated data into a structured format and storing it into a PostgreSQL research database.

Supporting sticky notes provide contextual explanations for each stage and highlight user-facing features.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block begins the workflow manually and retrieves the specified research paper PDF file from Google Drive for further processing.

**Nodes Involved:**  
- Manual Trigger  
- Google Drive - Get Paper

**Node Details:**

- **Manual Trigger**  
  - *Type:* Trigger node  
  - *Role:* Starts the workflow execution manually by the user.  
  - *Configuration:* Default, no parameters.  
  - *Input:* None (trigger node).  
  - *Output:* Initiates the workflow; outputs JSON data containing `fileId` required by the next node.  
  - *Edge Cases:* User must supply a valid `fileId` parameter externally for the file retrieval to succeed. Missing or invalid `fileId` results in failure on the Google Drive node.  

- **Google Drive - Get Paper**  
  - *Type:* Google Drive node  
  - *Role:* Downloads the research paper PDF file from Google Drive using the provided file ID.  
  - *Configuration:*  
    - Operation: `download`  
    - File ID: Expression `={{ $json.fileId }}` dynamically passed from input JSON.  
  - *Input:* Receives `fileId` from Manual Trigger or upstream data.  
  - *Output:* Provides the binary file data of the paper, along with metadata such as `webViewLink` for referencing the paper.  
  - *Version Specific:* Uses v3 Google Drive node, ensure OAuth2 credentials are properly configured.  
  - *Edge Cases:*  
    - Invalid or missing `fileId` causes file retrieval failure.  
    - Permissions errors if OAuth2 credentials lack access to the file.  
    - Network or API rate limits may cause timeouts or errors.

---

#### 2.2 Paper Parsing and Data Extraction

**Overview:**  
This block parses the downloaded PDF document using PDF Vector technology, applying OCR if needed, and extracts structured metadata and content elements.

**Nodes Involved:**  
- PDF Vector - Parse Paper  
- PDF Vector - Extract Data

**Node Details:**

- **PDF Vector - Parse Paper**  
  - *Type:* PDF Vector node  
  - *Role:* Converts the PDF binary into raw text content, preparing it for structured extraction.  
  - *Configuration:*  
    - Operation: `parse`  
    - Resource: `document`  
    - Input Type: `file` (binary data from Google Drive node)  
    - Use LLM (Large Language Model): Always enabled to enhance parsing accuracy.  
    - Binary Property Name: `data`  
  - *Input:* Binary PDF file from Google Drive node.  
  - *Output:* JSON containing raw extracted content of the paper (`content` field).  
  - *Edge Cases:*  
    - Corrupted or encrypted PDFs may fail to parse.  
    - Very large files may cause timeouts or memory issues.  

- **PDF Vector - Extract Data**  
  - *Type:* PDF Vector node  
  - *Role:* Extracts detailed structured data such as title, authors, abstract, methodology, and references from the parsed content. Utilizes OCR if the document is scanned or image-based.  
  - *Configuration:*  
    - Operation: `extract`  
    - Resource: `document`  
    - Input Type: `file` (same binary data)  
    - Prompt: Detailed extraction instructions covering title, authors (with affiliation and email), abstract, keywords, research questions, methodology, findings, conclusions, limitations, future work, and reference count.  
    - Schema: JSON schema validating output structure for robust integration.  
    - Binary Property Name: `data`  
  - *Input:* Same binary PDF file as parse node (not the parsed text JSON).  
  - *Output:* JSON object with fields for all requested paper metadata and analysis points.  
  - *Edge Cases:*  
    - OCR errors with low-quality scans can reduce extraction accuracy.  
    - Missing or inconsistent document formatting may cause incomplete extraction.  
    - Validation errors if output does not conform to schema.

---

#### 2.3 AI Processing

**Overview:**  
Generates an advanced AI summary and classification of the research paper leveraging OpenAI’s GPT-4 model, based on the parsed textual content.

**Nodes Involved:**  
- Generate AI Summary

**Node Details:**

- **Generate AI Summary**  
  - *Type:* OpenAI node  
  - *Role:* Uses GPT-4 to generate a concise summary, main contributions, potential applications, and classification tags for the paper.  
  - *Configuration:*  
    - Model: `gpt-4`  
    - Messages: System prompt requesting a structured output with specific elements: summary (150 words), main contribution (2-3 sentences), impact, and classification tags.  
    - Input Content: Injects the parsed paper content from the "PDF Vector - Parse Paper" node (`{{ $node['PDF Vector - Parse Paper'].json.content }}`).  
  - *Input:* JSON content field from parsing node.  
  - *Output:* JSON with AI-generated textual response inside `choices[0].message.content`.  
  - *Edge Cases:*  
    - API rate limits or quota errors from OpenAI.  
    - Model response variability or hallucination risks.  
    - Requires valid OpenAI credentials and internet access.  
  - *Version Specific:* Requires n8n version supporting OpenAI GPT-4 model integration.

---

#### 2.4 Data Preparation and Storage

**Overview:**  
Combines structured extracted data and AI-generated summaries into a unified JSON object, computes metadata such as reading time, and stores the entire record into a PostgreSQL database.

**Nodes Involved:**  
- Prepare Database Entry  
- Store in Database

**Node Details:**

- **Prepare Database Entry**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Aggregates data from previous nodes, computes additional metadata (like word count and reading time), and prepares a comprehensive database record.  
  - *Configuration:*  
    - JavaScript code accesses:  
      - Parsed content and full text (`PDF Vector - Parse Paper`)  
      - Extracted structured data (`PDF Vector - Extract Data`)  
      - AI summary text (`Generate AI Summary`)  
      - Paper URL (`Google Drive - Get Paper`)  
    - Calculates reading time assuming 250 words per minute.  
    - Constructs a searchable text field combining title, abstract, and keywords.  
    - Adds timestamp (`processedAt`) for record keeping.  
  - *Input:* JSON data from prior nodes.  
  - *Output:* Single JSON object representing the fully prepared research paper record.  
  - *Edge Cases:*  
    - Missing fields from extraction nodes could cause undefined errors.  
    - Large text fields may require database column size consideration.  

- **Store in Database**  
  - *Type:* PostgreSQL node  
  - *Role:* Inserts the prepared paper analysis record into the `research_papers` table of a PostgreSQL database.  
  - *Configuration:*  
    - Operation: `insert`  
    - Table: `research_papers`  
    - Columns: `title, authors, url, abstract, keywords, ai_summary, methodology, findings, processed_at, search_text`  
  - *Input:* JSON from the Prepare Database Entry node.  
  - *Output:* Confirmation of successful insert or error details.  
  - *Credential Requirements:* Valid PostgreSQL credentials with write permissions on the target table.  
  - *Edge Cases:*  
    - Database connectivity issues or permission errors.  
    - Schema mismatches or data type conflicts.  
    - Handling of array or JSON fields depends on DB schema design.

---

### 3. Summary Table

| Node Name                | Node Type            | Functional Role               | Input Node(s)             | Output Node(s)              | Sticky Note                                                                                       |
|--------------------------|----------------------|------------------------------|---------------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| Research Overview        | Sticky Note          | Overview of workflow purpose | None                      | None                        | ## 📚 Research Paper Analyzer\n\nAcademic research automation:\n• **Searches** arXiv, PubMed, Scholar\n• **Downloads** papers automatically\n• **Extracts** key findings with AI\n• **Summarizes** methodology & results\n• **Formats** citations (APA/MLA/BibTeX) |
| Academic Search          | Sticky Note          | Explains paper search sources | None                      | None                        | ## 🔍 Paper Search\n\nSearches databases:\n• arXiv (CS, Physics, Math)\n• PubMed (Medical)\n• Semantic Scholar\n\n💡 Returns top 10 relevant |
| Paper Extraction         | Sticky Note          | Explains data extraction      | None                      | None                        | ## 📄 Extraction\n\nPDF Vector extracts:\n• Title & authors\n• Abstract\n• Methodology\n• Results & findings\n• References\n\n🎯 Structured data |
| AI Analysis              | Sticky Note          | Explains AI summary           | None                      | None                        | ## 🤖 AI Summary\n\nGenerates:\n• Executive summary\n• Key contributions\n• Methodology critique\n• Future directions\n• Formatted citations\n\n✨ Publication ready! |
| Manual Trigger           | Manual Trigger       | Starts workflow manually      | None                      | Google Drive - Get Paper    | Start paper analysis                                                                            |
| Google Drive - Get Paper | Google Drive         | Downloads paper PDF           | Manual Trigger            | PDF Vector - Parse Paper    | Retrieve paper from Drive                                                                       |
| PDF Vector - Parse Paper | PDF Vector           | Parses PDF to raw text        | Google Drive - Get Paper  | PDF Vector - Extract Data   | Parse research paper                                                                           |
| PDF Vector - Extract Data| PDF Vector           | Extracts structured data      | PDF Vector - Parse Paper  | Generate AI Summary         | Extract structured data                                                                        |
| Generate AI Summary      | OpenAI GPT-4         | Creates AI summary            | PDF Vector - Extract Data | Prepare Database Entry      | Create AI summary                                                                             |
| Prepare Database Entry   | Code (JavaScript)    | Combines and formats data     | Generate AI Summary       | Store in Database           | Combine all data                                                                              |
| Store in Database        | PostgreSQL           | Saves record to database      | Prepare Database Entry    | None                       | Save to research database                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `Manual Trigger`  
   - Purpose: To start the workflow manually.

2. **Add a Google Drive node**  
   - Name: `Google Drive - Get Paper`  
   - Operation: `download`  
   - File ID: Set expression as `={{ $json.fileId }}` to dynamically accept input.  
   - Credentials: Configure Google Drive OAuth2 credentials with access to the file.

3. **Connect Manual Trigger → Google Drive - Get Paper**

4. **Add a PDF Vector node**  
   - Name: `PDF Vector - Parse Paper`  
   - Operation: `parse`  
   - Resource: `document`  
   - Input Type: `file`  
   - Binary Property Name: `data`  
   - Use LLM: Set to `always` to enable enhanced parsing.  

5. **Connect Google Drive - Get Paper → PDF Vector - Parse Paper**

6. **Add another PDF Vector node**  
   - Name: `PDF Vector - Extract Data`  
   - Operation: `extract`  
   - Resource: `document`  
   - Input Type: `file`  
   - Binary Property Name: `data`  
   - Prompt:  
     ```
     Extract key information from this research document or image including title, authors with affiliations, abstract, keywords, research questions, methodology, key findings, conclusions, limitations, and future work suggestions. Use OCR if this is a scanned document or image.
     ```  
   - Schema: Use the JSON schema as defined in the original workflow to validate output structure.

7. **Connect PDF Vector - Parse Paper → PDF Vector - Extract Data**

8. **Add an OpenAI node**  
   - Name: `Generate AI Summary`  
   - Model: `gpt-4`  
   - Messages:  
     - Content:  
       ```
       Based on this research paper, provide:

       1. A concise summary (150 words) suitable for a research database
       2. The main contribution to the field (2-3 sentences)
       3. Potential applications or impact
       4. Classification tags (e.g., empirical study, theoretical framework, review, etc.)

       Paper content:
       {{ $node['PDF Vector - Parse Paper'].json.content }}
       ```  
   - Credentials: Configure OpenAI API credentials with GPT-4 access.

9. **Connect PDF Vector - Extract Data → Generate AI Summary**

10. **Add a Code node (JavaScript)**  
    - Name: `Prepare Database Entry`  
    - Paste the following code snippet:  
      ```javascript
      // Combine all analysis data
      const parsedContent = $node['PDF Vector - Parse Paper'].json;
      const extractedData = $node['PDF Vector - Extract Data'].json.data;
      const aiSummary = $node['Generate AI Summary'].json.choices[0].message.content;

      // Calculate reading time (assuming 250 words per minute)
      const wordCount = parsedContent.content.split(' ').length;
      const readingTimeMinutes = Math.ceil(wordCount / 250);

      // Prepare database entry
      const paperAnalysis = {
        title: extractedData.title,
        authors: extractedData.authors,
        url: $node['Google Drive - Get Paper'].json.webViewLink,
        abstract: extractedData.abstract,
        keywords: extractedData.keywords,
        fullText: parsedContent.content,
        aiSummary: aiSummary,
        methodology: extractedData.methodology,
        findings: extractedData.findings,
        conclusions: extractedData.conclusions,
        limitations: extractedData.limitations,
        futureWork: extractedData.futureWork,
        wordCount: wordCount,
        readingTimeMinutes: readingTimeMinutes,
        referenceCount: extractedData.references || 0,
        processedAt: new Date().toISOString(),
        searchText: `${extractedData.title} ${extractedData.abstract} ${extractedData.keywords.join(' ')}`.toLowerCase()
      };

      return [{ json: paperAnalysis }];
      ```
    - Version: Use n8n v2 code node syntax.

11. **Connect Generate AI Summary → Prepare Database Entry**

12. **Add a PostgreSQL node**  
    - Name: `Store in Database`  
    - Operation: `insert`  
    - Table: `research_papers`  
    - Columns: `title,authors,url,abstract,keywords,ai_summary,methodology,findings,processed_at,search_text`  
    - Map the input JSON fields accordingly from the code node output.  
    - Credentials: Configure PostgreSQL credentials with insert privileges.

13. **Connect Prepare Database Entry → Store in Database**

14. **Add Sticky Notes for documentation** (Optional but recommended):  
    - Place notes describing the workflow overview, academic search scope, data extraction details, and AI summary output near their respective logical blocks for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow automates academic research paper analysis including search, download, extraction, summarization, and database storage. | Workflow purpose summary from sticky notes                                                             |
| Uses OpenAI GPT-4 model for AI-driven summarization—ensure proper API access and quota.                        | OpenAI API documentation: https://platform.openai.com/docs/models/gpt-4                                |
| Google Drive node requires OAuth2 credentials with file access permissions.                                   | Google Drive API documentation: https://developers.google.com/drive/api/v3/about-sdk                    |
| PDF Vector nodes use OCR if PDFs are scanned images; document quality affects extraction accuracy.            | PDF Vector integration details: https://pdfvector.com/docs                                              |
| Database schema must match inserted columns and support JSON/array types for authors, keywords, methodology.  | PostgreSQL JSON support: https://www.postgresql.org/docs/current/datatype-json.html                      |
| Reading time calculated at 250 words per minute as a standard academic reading speed.                         | Common metric for reading time estimation                                                               |
| The workflow assumes manual input of Google Drive fileId; automation or integration may require additional nodes. | Consider adding search or file selection nodes to automate fileId retrieval                             |

---

This documentation provides a comprehensive, stepwise understanding of the Research Paper Analysis System workflow, enabling replication, maintenance, and extension by advanced users or AI agents.