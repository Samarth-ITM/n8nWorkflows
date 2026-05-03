Generate AI-Ready llms.txt Files from Screaming Frog Website Crawls

https://n8nworkflows.xyz/workflows/generate-ai-ready-llms-txt-files-from-screaming-frog-website-crawls-3219


# Generate AI-Ready llms.txt Files from Screaming Frog Website Crawls

### 1. Workflow Overview

This workflow automates the generation of an **llms.txt** file from a **Screaming Frog website crawl export**. The llms.txt format is useful for preparing structured content data to aid large language models (LLMs) in content discovery and training.

**Target Use Cases:**  
- SEO specialists or content strategists who want to convert website crawl data into a structured text file for AI consumption.  
- Users who have crawled a website with Screaming Frog and want to generate an AI-ready content summary file quickly.  
- Optional AI-powered filtering to intelligently select high-value URLs for inclusion.

**Logical Blocks:**

- **1.1 Input Reception:** Form trigger node to collect website metadata and Screaming Frog CSV export.  
- **1.2 Data Extraction & Normalization:** Extract CSV data and normalize key fields across multiple languages.  
- **1.3 URL Filtering:** Filter URLs based on status, indexability, and content type.  
- **1.4 Optional AI Filtering:** (Deactivated by default) Use a text classifier powered by an LLM to further refine URLs.  
- **1.5 Formatting for llms.txt:** Format each URL entry into the llms.txt row structure.  
- **1.6 Concatenation & File Preparation:** Concatenate all rows and prepend website metadata.  
- **1.7 File Generation:** Convert the final content into a downloadable llms.txt file.  
- **1.8 Optional Upload:** Placeholder node for uploading the file to external storage (e.g., Google Drive).

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Collects user input including website name, description, and the Screaming Frog CSV export file. Triggers the workflow upon form submission.

- **Nodes Involved:**  
  - Form - Screaming frog internal_html.csv upload  
  - Sticky Note (Form explanation)

- **Node Details:**  
  - **Form - Screaming frog internal_html.csv upload**  
    - Type: Form Trigger  
    - Configuration:  
      - Three input fields:  
        1. Website name (required)  
        2. Short website description (required)  
        3. File upload for Screaming Frog export CSV (required, accepts .csv)  
      - Button label: "Get the llms.txt file"  
      - Response mode: Last node output  
    - Input: User form submission  
    - Output: JSON with form fields and binary file data  
    - Edge cases:  
      - Missing required fields will prevent submission  
      - Uploading non-CSV or wrong CSV format may cause downstream failures  
    - Sticky Note: Explains form purpose and recommended CSV export type.

---

#### 1.2 Data Extraction & Normalization

- **Overview:**  
  Extracts data from the uploaded CSV file and normalizes key fields to handle multiple languages (English, French, Italian, German, Spanish).

- **Nodes Involved:**  
  - Extract data from Screaming Frog file  
  - Set useful fields  
  - Sticky Notes (Extraction and field setting explanations)

- **Node Details:**  
  - **Extract data from Screaming Frog file**  
    - Type: Extract From File  
    - Configuration:  
      - Operation: XLS (handles CSV as spreadsheet)  
      - Binary property: "screaming_frog_export" (file uploaded in form)  
    - Input: Binary CSV file  
    - Output: JSON array of rows from CSV  
    - Edge cases:  
      - If file is not a valid Screaming Frog export, subsequent nodes may fail due to missing columns.

  - **Set useful fields**  
    - Type: Set  
    - Configuration:  
      - Assigns 7 key fields from CSV columns, with fallback for multilingual column names:  
        - url ← Address / Adresse / Dirección / Indirizzo  
        - title ← Title 1 / Titolo 1 / Título 1 / Titel 1  
        - description ← Meta Description 1 / Meta description 1  
        - statut (status) ← Status Code / Code HTTP / Status-Code / Código de respuesta / Codice di stato  
        - indexability ← Indexability / Indexabilité / Indicizzabilità / Indexabilidad / Indexierbarkeit  
        - content_type ← Content Type / Type de contenu / Tipo di contenuto / Tipo de contenido / Inhaltstyp  
        - word_count ← Word Count / Nombre de mots / Conteggio delle parole / Recuento de palabras / Wortanzahl  
    - Input: JSON rows from extraction node  
    - Output: JSON with normalized fields  
    - Edge cases:  
      - Missing columns may result in empty fields, affecting filtering downstream.

---

#### 1.3 URL Filtering

- **Overview:**  
  Filters URLs to keep only those that are indexable, have status 200, and content type containing "text/html". This ensures only relevant pages are included.

- **Nodes Involved:**  
  - Filter URLs  
  - Sticky Note (Filter explanation)

- **Node Details:**  
  - **Filter URLs**  
    - Type: Filter  
    - Configuration:  
      - Conditions combined with AND:  
        1. status code equals 200 (converted to number)  
        2. indexability is one of ["Indexable", "Indicizzabile", "Indexierbar"] (case-sensitive)  
        3. content_type contains "text/html"  
    - Input: JSON with normalized fields  
    - Output: Filtered JSON with only matching URLs  
    - Edge cases:  
      - Non-standard status codes or indexability values may exclude valid URLs  
      - Content types not exactly containing "text/html" will be excluded  
    - Sticky Note: Suggests adding additional filters (e.g., word count, URL path, meta description).

---

#### 1.4 Optional AI Filtering (Deactivated by Default)

- **Overview:**  
  Uses an LLM-powered text classifier to categorize URLs into "useful_content" or "other_content" for more intelligent filtering.

- **Nodes Involved:**  
  - Text Classifier (disabled by default)  
  - OpenAI Chat Model (used as AI backend)  
  - No Operation, do nothing (for non-useful content)  
  - Sticky Note (Text Classifier explanation)

- **Node Details:**  
  - **Text Classifier**  
    - Type: Langchain Text Classifier  
    - Configuration:  
      - Input text composed of URL, title, description, and word count  
      - Categories:  
        - useful_content: Pages likely containing high-quality content  
        - other_content: Pages to exclude (pagination, low-value)  
      - Disabled by default; user can enable and customize category descriptions  
    - Input: Filtered URLs from previous block  
    - Output: Two outputs:  
      - useful_content → to formatting nodes  
      - other_content → to No Operation node (discarded)  
    - Edge cases:  
      - API timeouts or quota limits on OpenAI calls  
      - Misclassification if input data is sparse or ambiguous  
      - Token usage increases with number of URLs processed  
    - **OpenAI Chat Model** node:  
      - Type: Langchain OpenAI Chat Model  
      - Model: gpt-4o-mini  
      - Credentials: OpenAI API key required  
      - Input: Feeds the Text Classifier node  
      - Edge cases: API authentication errors, rate limits

  - **No Operation, do nothing**  
    - Type: NoOp  
    - Purpose: Sink for discarded URLs from classifier

---

#### 1.5 Formatting for llms.txt

- **Overview:**  
  Formats each URL into a markdown-style row for the llms.txt file, including title, URL, and optional description.

- **Nodes Involved:**  
  - Set Field - llms.txt Row  
  - Sticky Note (Row format explanation)

- **Node Details:**  
  - **Set Field - llms.txt Row**  
    - Type: Set  
    - Configuration:  
      - Creates a new field `llmTxtRow` with value:  
        `- [title](url): description` if description exists, else `- [title](url)`  
      - Uses expressions to conditionally append description  
    - Input: URLs from classifier or filter node  
    - Output: JSON with `llmTxtRow` field  
    - Edge cases:  
      - Missing title or URL will produce malformed rows  
      - Empty description handled gracefully

---

#### 1.6 Concatenation & File Preparation

- **Overview:**  
  Concatenates all formatted rows into a single string with line breaks and prepends website metadata from the form.

- **Nodes Involved:**  
  - Summarize - Concatenate  
  - Set Fields - llms.txt Content  
  - Sticky Notes (Concatenation and content setting explanations)

- **Node Details:**  
  - **Summarize - Concatenate**  
    - Type: Summarize  
    - Configuration:  
      - Aggregates all `llmTxtRow` fields, concatenated with newline separator  
    - Input: Multiple JSON items with `llmTxtRow`  
    - Output: Single JSON item with concatenated string in `concatenated_llmTxtRow`  
    - Edge cases:  
      - Empty input results in empty concatenation

  - **Set Fields - llms.txt Content**  
    - Type: Set  
    - Configuration:  
      - Creates a field `llmsTxtFile` with content:  
        ```
        # {{website name}}
        > {{website description}}

        {{concatenated llmTxtRow}}
        ```  
      - Uses expressions to pull website name and description from form node  
    - Input: Concatenated rows  
    - Output: JSON with full llms.txt content string  
    - Edge cases:  
      - Missing form inputs will produce incomplete header

---

#### 1.7 File Generation

- **Overview:**  
  Converts the final llms.txt content string into a downloadable UTF-8 encoded text file.

- **Nodes Involved:**  
  - Generate llms.txt file  
  - Sticky Note (File generation explanation)

- **Node Details:**  
  - **Generate llms.txt file**  
    - Type: Convert To File  
    - Configuration:  
      - Operation: toText  
      - Source property: `llmsTxtFile` (string content)  
      - Encoding: UTF-8  
      - Filename: llms.txt  
    - Input: JSON with llms.txt content string  
    - Output: Binary file data for download  
    - Edge cases:  
      - Large content may affect performance  
      - User must download manually from n8n UI unless upload node is added

---

#### 1.8 Optional Upload

- **Overview:**  
  Placeholder node for uploading the generated file to external storage services.

- **Nodes Involved:**  
  - upload file anywhere (No Operation)  
  - Sticky Note (Upload instructions)

- **Node Details:**  
  - **upload file anywhere**  
    - Type: NoOp (placeholder)  
    - Purpose: User can replace with Google Drive, OneDrive, or other upload node  
    - Edge cases: None (inactive)  
    - Sticky Note: Advises naming files properly and replacing this node for automation

---

### 3. Summary Table

| Node Name                                | Node Type                          | Functional Role                          | Input Node(s)                             | Output Node(s)                          | Sticky Note                                                                                              |
|-----------------------------------------|----------------------------------|----------------------------------------|------------------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------|
| Form - Screaming frog internal_html.csv upload | Form Trigger                     | Input reception and workflow trigger   | None                                     | Extract data from Screaming Frog file | Explains form fields and recommended CSV export                                                        |
| Extract data from Screaming Frog file   | Extract From File                 | Extract CSV data from uploaded file    | Form - Screaming frog internal_html.csv upload | Set useful fields                     | Warns about potential failure if file format is incorrect                                              |
| Set useful fields                       | Set                              | Normalize and assign key fields        | Extract data from Screaming Frog file    | Filter URLs                           | Lists key fields extracted and multilingual compatibility                                              |
| Filter URLs                            | Filter                           | Filter URLs by status, indexability, content type | Set useful fields                        | Text Classifier (optional)            | Explains filtering criteria and suggests additional filters                                            |
| Text Classifier                        | Langchain Text Classifier        | AI-powered content classification      | Filter URLs                             | Set Field - llms.txt Row (useful_content), No Operation (other_content) | Explains optional AI filtering, usage, and token considerations                                         |
| OpenAI Chat Model                     | Langchain OpenAI Chat Model      | Provides LLM backend for classifier    | None (used internally by Text Classifier) | Text Classifier                      | None                                                                                                   |
| No Operation, do nothing               | NoOp                             | Sink for discarded URLs                 | Text Classifier (other_content output)  | None                                  | None                                                                                                   |
| Set Field - llms.txt Row               | Set                              | Format each URL into llms.txt row      | Text Classifier (useful_content) or Filter URLs (if classifier disabled) | Summarize - Concatenate              | Explains row format with conditional description                                                       |
| Summarize - Concatenate                | Summarize                       | Concatenate all rows into one string   | Set Field - llms.txt Row                 | Set Fields - llms.txt Content         | Explains concatenation of rows                                                                         |
| Set Fields - llms.txt Content           | Set                              | Prepend website metadata and prepare final content | Summarize - Concatenate                 | Generate llms.txt file                | Explains assembling final llms.txt content                                                             |
| Generate llms.txt file                 | Convert To File                  | Convert content string to downloadable file | Set Fields - llms.txt Content            | None                                  | Explains file generation and download process                                                          |
| upload file anywhere                   | NoOp                             | Placeholder for file upload             | Generate llms.txt file                    | None                                  | Advises replacing with Drive node for automatic upload                                                 |
| Sticky Note (various)                  | Sticky Note                     | Documentation and explanations          | None                                     | None                                  | Multiple notes explaining each block and node                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Type: Form Trigger  
   - Name: "Form - Screaming frog internal_html.csv upload"  
   - Configure form with 3 fields:  
     - Text: "What is the name of your website?" (required)  
     - Text: "Can you provide a short description of your website? (in the language of the website)" (required)  
     - File upload: "screaming_frog_export" (accept .csv, single file, required)  
   - Set button label: "Get the llms.txt file"  
   - Set response mode: Last node output

2. **Add Extract From File Node:**  
   - Type: Extract From File  
   - Name: "Extract data from Screaming Frog file"  
   - Operation: XLS (to handle CSV as spreadsheet)  
   - Binary property name: "screaming_frog_export" (from form node)  
   - Connect form node output to this node input

3. **Add Set Node to Normalize Fields:**  
   - Type: Set  
   - Name: "Set useful fields"  
   - Assign fields with expressions to handle multilingual columns:  
     - url = `{{$json.Address || $json.Adresse || $json.Dirección || $json.Indirizzo}}`  
     - title = `{{$json['Title 1'] || $json['Titolo 1'] || $json['Título 1'] || $json['Titel 1']}}`  
     - description = `{{$json['Meta Description 1'] || $json['Meta description 1']}}`  
     - statut = `{{$json['Status Code'] || $json['Code HTTP'] || $json['Status-Code'] || $json['Código de respuesta'] || $json['Codice di stato']}}`  
     - indexability = `{{$json.Indexability || $json.Indexabilité || $json.Indicizzabilità || $json.Indexabilidad || $json.Indexierbarkeit}}`  
     - content_type = `{{$json['Content Type'] || $json['Type de contenu'] || $json['Tipo di contenuto'] || $json['Tipo de contenido'] || $json['Inhaltstyp']}}`  
     - word_count = `{{$json['Word Count'] || $json['Nombre de mots'] || $json['Conteggio delle parole'] || $json['Recuento de palabras'] || $json['Wortanzahl']}}`  
   - Connect extract node output to this node input

4. **Add Filter Node to Select URLs:**  
   - Type: Filter  
   - Name: "Filter URLs"  
   - Conditions (AND):  
     - Number(statut) equals 200  
     - indexability in ["Indexable", "Indicizzabile", "Indexierbar"] (case-sensitive)  
     - content_type contains "text/html"  
   - Connect "Set useful fields" node output to this node input

5. **(Optional) Add AI Text Classifier Node:**  
   - Type: Langchain Text Classifier  
   - Name: "Text Classifier"  
   - Disabled by default; enable if desired  
   - Input text template:  
     ```
     url : {{ $json.url }}
     title : {{ $json.title }}
     description : {{ $json.description }}
     words count : {{ $json.word_count }}
     ```  
   - Categories:  
     - useful_content: "Pages that are likely to contain high-quality content..."  
     - other_content: "Pages that should not be included..."  
   - Connect "Filter URLs" node output to this node input  
   - Add OpenAI Chat Model node as AI backend:  
     - Type: Langchain OpenAI Chat Model  
     - Model: gpt-4o-mini  
     - Connect to Text Classifier AI input  
     - Configure OpenAI credentials  
   - Connect "useful_content" output of classifier to next node  
   - Connect "other_content" output to No Operation node (sink)

6. **Add Set Node to Format llms.txt Rows:**  
   - Type: Set  
   - Name: "Set Field - llms.txt Row"  
   - Assign field `llmTxtRow` with expression:  
     ```
     =- [{{ $json.title }}]({{ $json.url }}){{ $json.description ? ': ' + $json.description : '' }}
     ```  
   - Connect from either "Text Classifier" useful_content output or "Filter URLs" output if classifier disabled

7. **Add Summarize Node to Concatenate Rows:**  
   - Type: Summarize  
   - Name: "Summarize - Concatenate"  
   - Fields to summarize: `llmTxtRow`  
   - Aggregation: Concatenate with separator `\n` (newline)  
   - Connect "Set Field - llms.txt Row" output to this node input

8. **Add Set Node to Prepare Final llms.txt Content:**  
   - Type: Set  
   - Name: "Set Fields - llms.txt Content"  
   - Assign field `llmsTxtFile` with expression:  
     ```
     =# {{ $('Form - Screaming frog internal_html.csv upload').item.json['What is the name of your website?'] }}
     > {{ $('Form - Screaming frog internal_html.csv upload').item.json['Can you provide a short description of your website? (in the language of the website)'] }}

     {{ $json.concatenated_llmTxtRow }}
     ```  
   - Connect "Summarize - Concatenate" output to this node input

9. **Add Convert To File Node to Generate Downloadable File:**  
   - Type: Convert To File  
   - Name: "Generate llms.txt file"  
   - Operation: toText  
   - Source property: `llmsTxtFile`  
   - Encoding: UTF-8  
   - Filename: llms.txt  
   - Connect "Set Fields - llms.txt Content" output to this node input

10. **(Optional) Add Upload Node:**  
    - Replace the placeholder No Operation node with a Google Drive, OneDrive, or other storage node  
    - Configure credentials and folder destination  
    - Connect "Generate llms.txt file" output to upload node input

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow helps generate an llms.txt file from Screaming Frog exports, useful for AI content discovery and training.                                                                                                        | https://towardsdatascience.com/llms-txt-414d5121bcb3/                                             |
| Screaming Frog is a popular website crawler; export the "internal_html" section in CSV format for best results.                                                                                                                 | https://www.screamingfrog.co.uk/seo-spider/                                                       |
| The form node triggers the workflow and accepts website name, description, and Screaming Frog CSV export.                                                                                                                       |                                                                                                    |
| The AI-powered Text Classifier node is disabled by default; enable it to apply intelligent filtering using OpenAI GPT-4o-mini.                                                                                                |                                                                                                    |
| Download the generated llms.txt file directly from the n8n UI after workflow completion, or add a Drive node to automate file uploads.                                                                                        |                                                                                                    |
| For large websites, consider looping over items to handle API quotas and timeouts when using the AI classifier.                                                                                                                |                                                                                                    |
| The workflow supports multiple languages for Screaming Frog exports by mapping column names accordingly.                                                                                                                       |                                                                                                    |
| Example screenshot of Screaming Frog internal_html export included in workflow description.                                                                                                                                     | https://i.imgur.com/M0nJQiV.png                                                                    |

---

This documentation provides a comprehensive understanding of the workflow structure, node configurations, and usage instructions to enable reproduction, modification, and troubleshooting.