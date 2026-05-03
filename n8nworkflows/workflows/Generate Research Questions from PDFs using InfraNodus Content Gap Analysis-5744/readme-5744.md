Generate Research Questions from PDFs using InfraNodus Content Gap Analysis

https://n8nworkflows.xyz/workflows/generate-research-questions-from-pdfs-using-infranodus-content-gap-analysis-5744


# Generate Research Questions from PDFs using InfraNodus Content Gap Analysis

### 1. Workflow Overview

This workflow automates the generation of research questions based on content gaps identified within PDF documents uploaded by users. Its primary use case is to assist researchers, analysts, and knowledge workers in discovering unexplored or underdeveloped topics by analyzing the textual content of one or more PDF files. The workflow leverages InfraNodus, an AI-driven graph analysis platform, to create a knowledge graph from the extracted text, identify gaps, and generate insightful research prompts.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Collect user-uploaded PDF files via a web form.
- **1.2 Binary to PDF Conversion:** Prepare uploaded binary files for text extraction.
- **1.3 Text Extraction:** Extract plain text content from PDF files.
- **1.4 Text Aggregation:** Combine extracted texts into a single string, prepare parameters for InfraNodus.
- **1.5 InfraNodus GraphRAG Integration:** Send combined text to InfraNodus API to generate research questions based on content gaps.
- **1.6 User Feedback Display:** Present the generated research questions back to the user through the form.

Optional logic includes a high-quality PDF-to-text conversion method using ConvertAPI, recommended for better text extraction fidelity.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Receives PDF files from users through a customizable web form endpoint exposed publicly or internally.

- **Nodes Involved:**  
  - On form submission  
  - Sticky Note (Step 1 explanation)

- **Node Details:**

  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Entry point collecting PDF files uploaded by users.  
    - *Configuration:* Form titled "Find Content Gaps in Your PDF Files" with a single file upload field accepting `.pdf` files. No attribution appended.  
    - *Connections:* Output connects to "Convert binary files to PDF" node.  
    - *Edge Cases:* File upload failures, unsupported file types, or large files exceeding limits may cause errors or failures.

  - **Sticky Note (Step 1)**  
    - *Purpose:* Describes this step's role as the user input interface, encouraging public exposure for organizational use.

#### 2.2 Binary to PDF Conversion

- **Overview:**  
  Converts uploaded binary data into individual PDF files, preparing each for text extraction.

- **Nodes Involved:**  
  - Convert binary files to PDF  
  - Sticky Note1 (Step 2 explanation)

- **Node Details:**

  - **Convert binary files to PDF**  
    - *Type:* Code  
    - *Role:* Splits incoming binary data into separate items, each representing one PDF file, with clean file names (spaces replaced by underscores).  
    - *Key Logic:* Iterates over binary data keys, creates new items with binary data attached under sanitized keys.  
    - *Input:* Receives form submission items with binary file data.  
    - *Output:* Sends separated PDF files downstream.  
    - *Edge Cases:* Missing binary data or unsupported binary keys may cause empty outputs.

  - **Sticky Note1**  
    - *Purpose:* Explains necessity of this conversion before text extraction.

#### 2.3 Text Extraction

- **Overview:**  
  Extracts plain text content from each PDF file item for analysis.

- **Nodes Involved:**  
  - Extract text from PDF files  
  - Sticky Note2 (Step 3 explanation)  
  - Convert File to PDF (disabled alternative node)  
  - Sticky Note5 (Optional high-quality PDF conversion)

- **Node Details:**

  - **Extract text from PDF files**  
    - *Type:* Extract From File  
    - *Role:* Performs PDF to text extraction using n8n's built-in extractor.  
    - *Configuration:* Uses the corresponding binary property name dynamically based on the file name.  
    - *Input:* Receives individual PDF binary files.  
    - *Output:* Outputs extracted plain text per item.  
    - *Edge Cases:* Poor extraction quality if PDFs have complex layouts; may produce fragmented text chunks.

  - **Convert File to PDF (disabled)**  
    - *Type:* HTTP Request  
    - *Role:* Alternative method to convert PDF to text using ConvertAPI for better layout preservation.  
    - *Note:* Disabled by default; requires ConvertAPI credentials.  
    - *Edge Cases:* Requires API key; potential HTTP errors or rate limits.

  - **Sticky Note2 & Sticky Note5**  
    - *Purpose:* Notes on improving text extraction quality by optionally using ConvertAPI.

#### 2.4 Text Aggregation

- **Overview:**  
  Combines all extracted text snippets into a single string and randomly selects a gap depth parameter for InfraNodus analysis.

- **Nodes Involved:**  
  - Prepare for InfraNodus  
  - Sticky Note3 (Step 4 explanation)

- **Node Details:**

  - **Prepare for InfraNodus**  
    - *Type:* Code  
    - *Role:* Concatenates the text from all PDF items into one string, separated by double newlines. Generates a random number (0–2) as a gap depth indicator for InfraNodus.  
    - *Output:* JSON containing combined `text` and `randomNum` for API query.  
    - *Edge Cases:* Empty input items result in empty text; random number generation is simple and could be replaced by a fixed value if desired.

  - **Sticky Note3**  
    - *Purpose:* Explains the rationale for text concatenation and gap depth parameter.

#### 2.5 InfraNodus GraphRAG Integration

- **Overview:**  
  Sends the aggregated text and configuration to InfraNodus API to build a knowledge graph, detect content gaps, and generate targeted research questions.

- **Nodes Involved:**  
  - InfraNodus GraphRAG Question Generator  
  - Sticky Note4 (Step 5 explanation)

- **Node Details:**

  - **InfraNodus GraphRAG Question Generator**  
    - *Type:* HTTP Request  
    - *Role:* Posts combined text to InfraNodus endpoint with parameters to request AI-generated questions based on graph gap analysis.  
    - *Configuration:*  
      - URL dynamically constructed to include gap depth from `randomNum`.  
      - POST body includes `aiTopics: true`, `requestMode: question`, and text payload.  
      - Uses HTTP Bearer Authentication with InfraNodus API key credential.  
    - *Input:* Receives JSON with `text` and `randomNum`.  
    - *Output:* InfraNodus response containing generated research questions and graph summary.  
    - *Edge Cases:* API key missing or invalid, network errors, API rate limiting, or malformed text input.

  - **Sticky Note4**  
    - *Purpose:* Describes InfraNodus's knowledge graph methodology and emphasizes the need for API key.

#### 2.6 User Feedback Display

- **Overview:**  
  Presents the generated research question or AI prompt back to the user via the original form interface.

- **Nodes Involved:**  
  - Display on the Form to the User  
  - Sticky Note6 (Step 6 explanation)

- **Node Details:**

  - **Display on the Form to the User**  
    - *Type:* Form  
    - *Role:* Completes the form trigger cycle by displaying AI-generated suggestions as HTML formatted text.  
    - *Configuration:* Uses `completion` operation to show text, embedding the first response from InfraNodus under `aiAdvice[0].text`.  
    - *Input:* Receives InfraNodus API response.  
    - *Output:* Delivers textual output to the user’s browser form.  
    - *Edge Cases:* Missing or malformed API response may result in empty or broken display.

  - **Sticky Note6**  
    - *Purpose:* Suggests possible extensions like forwarding output to other workflows or custom apps.

---

### 3. Summary Table

| Node Name                        | Node Type           | Functional Role                              | Input Node(s)              | Output Node(s)                       | Sticky Note                                                                                             |
|---------------------------------|---------------------|----------------------------------------------|----------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------------|
| On form submission              | Form Trigger        | Entry point for user to upload PDF files      | —                          | Convert binary files to PDF          | Step 1: User uploads the PDF files for analysis. You can expose this endpoint publicly.                |
| Convert binary files to PDF      | Code                | Splits uploaded binaries into individual PDFs | On form submission          | Extract text from PDF files          | Step 2: Convert uploaded binaries into PDF files for text extraction.                                 |
| Extract text from PDF files      | Extract From File   | Extracts plain text from each PDF file         | Convert binary files to PDF | Prepare for InfraNodus               | Step 3: Extract plain text from PDFs; optional ConvertAPI for better quality.                         |
| Convert File to PDF (disabled)   | HTTP Request        | Optional higher-quality PDF to text conversion | —                          | —                                   | Optional: Use ConvertAPI for better PDF-to-text conversion preserving layout.                          |
| Prepare for InfraNodus           | Code                | Aggregates all extracted text and sets gap depth | Extract text from PDF files | InfraNodus GraphRAG Question Generator | Step 4: Combine extracted text into a single string and prepare parameters.                           |
| InfraNodus GraphRAG Question Generator | HTTP Request        | Calls InfraNodus API to generate research questions | Prepare for InfraNodus      | Display on the Form to the User      | Step 5: Use InfraNodus to build knowledge graph and identify content gaps. Provide API key.           |
| Display on the Form to the User  | Form                | Shows AI-generated research questions to user | InfraNodus GraphRAG Question Generator | —                                   | Step 6: Show question/prompt to user; can forward output to other workflows or apps.                   |
| Sticky Note                     | Sticky Note         | Explanations and documentation                 | —                          | —                                   | Various sticky notes explaining workflow steps and concepts, including InfraNodus methodology.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node ("On form submission"):**  
   - Set form title: "Find Content Gaps in Your PDF Files"  
   - Add a file upload field labeled "Add Your Files" accepting `.pdf` only  
   - Disable attribution append  
   - Expose this webhook endpoint publicly or internally as needed.

2. **Add a Code node ("Convert binary files to PDF"):**  
   - Purpose: For each uploaded file in binary format, create a separate item with binary data keyed by sanitized file name.  
   - Use this JavaScript code:
     ```javascript
     let results = [];
     for (let item of items) {
         if (item.binary) {
             for (let key in item.binary) {
                 let binaryKey = key.replace(/\s/g, '_');
                 results.push({
                     json: { fileName: binaryKey },
                     binary: { [binaryKey]: item.binary[key] },
                 });
             }
         }
     }
     return results;
     ```
   - Connect "On form submission" main output to this node.

3. **Add an Extract From File node ("Extract text from PDF files"):**  
   - Operation: PDF  
   - Binary Property Name: Use expression `{{$json.fileName}}` to dynamically select the binary file.  
   - Connect output of "Convert binary files to PDF" to this node.

4. **(Optional) Add an HTTP Request node ("Convert File to PDF") for better text extraction:**  
   - Method: POST  
   - URL: `https://v2.convertapi.com/convert/pdf/to/txt`  
   - Content-Type: multipart/form-data  
   - Authentication: HTTP Bearer with ConvertAPI key  
   - Body: Include binary file under "file" form parameter  
   - Note: Replace "Extract text from PDF files" with this node if preferred, and adjust downstream accordingly.

5. **Add a Code node ("Prepare for InfraNodus"):**  
   - Purpose: Concatenate all extracted text into a single string separated by double newlines; generate random gap depth.  
   - Use code:
     ```javascript
     let plainText = '';
     for (let item of items) {
         plainText += item.json.text + '\n\n';
     }
     const randomNum = Math.floor(Math.random() * 3);
     return { json: { text: plainText, randomNum } };
     ```
   - Connect "Extract text from PDF files" output here.

6. **Add an HTTP Request node ("InfraNodus GraphRAG Question Generator"):**  
   - Method: POST  
   - URL: `https://infranodus.com/api/v1/graphAndAdvice?doNotSave=true&optimize=develop&includeGraph=false&includeGraphSummary=true&gapDepth={{ $json.randomNum }}`  
   - Authentication: HTTP Bearer with InfraNodus API key credential  
   - Body Parameters (form or JSON):  
     - `aiTopics: true`  
     - `requestMode: question`  
     - `text: {{$json.text}}`  
   - Connect "Prepare for InfraNodus" output here.

7. **Add a Form node ("Display on the Form to the User"):**  
   - Operation: Completion  
   - Respond With: Show Text  
   - Response Text:  
     ```html
     <br><h3>{{ $json.aiAdvice[0].text }}</h3><br>
     ```  
   - Connect output of InfraNodus HTTP Request node here.

8. **Set Credentials:**  
   - For ConvertAPI (optional): Create HTTP Bearer credential with your API key.  
   - For InfraNodus: Create HTTP Bearer credential with your InfraNodus API key.

9. **Configure Connections and Flow:**  
   - Connect nodes as per above sequence:  
     `On form submission → Convert binary files to PDF → Extract text from PDF files → Prepare for InfraNodus → InfraNodus GraphRAG Question Generator → Display on the Form to the User`

10. **Optional Enhancements:**  
    - Replace text extraction with ConvertAPI node for better quality.  
    - Forward output from the final form node to other workflows or external apps via webhook or iframe embedding.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                             | Context or Link                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| For improved PDF to text conversion preserving paragraph and layout integrity, use ConvertAPI's service. This helps avoid short text chunks common with standard extractors.                              | https://convertapi.com?ref=4l54n                     |
| InfraNodus builds knowledge graphs from text to identify structural gaps and generate insightful prompts by connecting distant topic clusters. This reduces generic AI bias.                            | https://infranodus.com                              |
| InfraNodus API requires an API key to authenticate requests; it is critical to keep this key secure and provide it in the HTTP Bearer credentials.                                                      | InfraNodus API documentation                         |
| The workflow’s web form can be exposed publicly to enable organizational users to upload PDFs and retrieve research questions on demand.                                                                | n8n Webhook and Form Trigger documentation           |
| The workflow logic demonstrates integration of file handling, text extraction, AI-driven graph analysis, and dynamic user interaction within a single automated pipeline.                              | n8n official workflow best practices                  |
| The sticky notes embedded throughout the workflow provide in-context explanations and usage instructions, useful for onboarding and modification.                                                       | Visible in workflow editor UI                          |

---

**Disclaimer:**  
The provided content is derived exclusively from an n8n automated workflow designed for legal, public data processing. It complies strictly with applicable content policies and contains no illegal, offensive, or protected information.