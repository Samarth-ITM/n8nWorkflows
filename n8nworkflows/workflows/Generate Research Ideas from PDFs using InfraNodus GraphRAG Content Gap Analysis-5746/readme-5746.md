Generate Research Ideas from PDFs using InfraNodus GraphRAG Content Gap Analysis

https://n8nworkflows.xyz/workflows/generate-research-ideas-from-pdfs-using-infranodus-graphrag-content-gap-analysis-5746


# Generate Research Ideas from PDFs using InfraNodus GraphRAG Content Gap Analysis

### 1. Workflow Overview

This n8n workflow is designed to generate novel research ideas by analyzing PDF documents uploaded by users. It leverages InfraNodus GraphRAG technology to identify content gaps in the knowledge represented by the PDFs and produces research questions and responses that bridge these gaps. The workflow is aimed at researchers, academics, or knowledge workers who want to extract meaningful insights and unexplored topics from document collections.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** User uploads PDF files via a form trigger.
- **1.2 File Processing:** Conversion of uploaded binary data to PDF files, followed by text extraction from PDFs.
- **1.3 Text Preparation:** Combining extracted text from all PDFs into a single string and preparing parameters for InfraNodus analysis.
- **1.4 InfraNodus GraphRAG Integration:** Sending the combined text to InfraNodus API to generate a content gap question and then an AI-generated response based on that question.
- **1.5 Output Presentation:** Displaying the generated research question and AI response back to the user via a form response.

Additionally, there is an optional high-quality PDF-to-text conversion node using ConvertAPI, recommended for better text extraction fidelity.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block handles user interaction by exposing a form where users upload their PDF files for analysis.

**Nodes Involved:**  
- On form submission  
- Sticky Note (Step 1)

**Node Details:**

- **On form submission**  
  - Type: `formTrigger`  
  - Role: Entry point node that triggers the workflow when a user submits a form with PDF files attached.  
  - Configuration:  
    - Webhook enabled with a unique webhook ID.  
    - Form titled "Find Content Gaps in Your PDF Files".  
    - Single file upload field accepting `.pdf` files only.  
    - Description instructs users to upload files for content gap analysis.  
  - Inputs: None (trigger node)  
  - Outputs: Connects to "Convert binary files to PDF" node.  
  - Edge cases: Missing file upload, unsupported file type, webhook failures.

- **Sticky Note** (Step 1)  
  - Content provides context for this block, explaining the form usage and public availability.  
  - No direct influence on workflow execution.

#### 2.2 File Processing

**Overview:**  
This block converts uploaded binaries to PDF files and extracts plain text from them for further analysis.

**Nodes Involved:**  
- Convert binary files to PDF  
- Extract text from PDF files  
- Sticky Notes (Step 2, Step 3)  
- Optional: Convert File to PDF (disabled)

**Node Details:**

- **Convert binary files to PDF**  
  - Type: `code`  
  - Role: Iterates over incoming binary data from the form submission, creates separate items for each binary file with file name keys adjusted to avoid spaces.  
  - Key code logic: Loops through all binary properties, replaces spaces with underscores in keys, and restructures items accordingly.  
  - Inputs: Receives form submission items with binary files.  
  - Outputs: Multiple items each containing one PDF binary.  
  - Edge cases: Binary data missing or malformed; large file sizes.

- **Extract text from PDF files**  
  - Type: `extractFromFile`  
  - Role: Extracts text content from each PDF binary using n8n’s built-in PDF text extraction.  
  - Configuration:  
    - Operation set to "pdf".  
    - Binary property name dynamically linked to the file name key from previous node.  
  - Inputs: PDF binary files from previous node.  
  - Outputs: JSON with extracted text.  
  - Edge cases: Corrupted PDFs, extraction failures, empty text results.

- **Convert File to PDF** (disabled HTTP Request node)  
  - Role: Optional high-quality PDF-to-text conversion using ConvertAPI service.  
  - Disabled by default.  
  - Requires ConvertAPI credentials and mapping of output text for use downstream.

- **Sticky Notes (Step 2 and Step 3)**  
  - Explain the rationale for binary to PDF conversion and recommend ConvertAPI for higher-quality text extraction.

#### 2.3 Text Preparation

**Overview:**  
Combines all extracted text from the PDFs into one large plain text string and selects a random gap depth parameter for InfraNodus analysis.

**Nodes Involved:**  
- Prepare for InfraNodus  
- Sticky Note (Step 4)

**Node Details:**

- **Prepare for InfraNodus**  
  - Type: `code`  
  - Role: Concatenates extracted text fields into a single string separated by double newlines. Randomly selects a gap depth integer (0 to 2) to guide InfraNodus’ gap analysis.  
  - Outputs JSON containing:  
    - `text`: combined text string.  
    - `randomNum`: gap depth parameter.  
  - Inputs: All extracted text items from previous node.  
  - Outputs: Single JSON object for InfraNodus input.  
  - Edge cases: Empty or very short text input; random number generation failures.

- **Sticky Note (Step 4)**  
  - Describes the purpose of combining texts and preparing parameters for InfraNodus.

#### 2.4 InfraNodus GraphRAG Integration

**Overview:**  
Sends prepared text to InfraNodus API to generate a research question based on content gaps and then generates an AI response to that question.

**Nodes Involved:**  
- InfraNodus GraphRAG Question Generator  
- InfraNodus GraphRAG Response Generator  
- Sticky Notes (Step 5, Step 6, Step 9)

**Node Details:**

- **InfraNodus GraphRAG Question Generator**  
  - Type: `httpRequest`  
  - Role: Sends combined text and gap depth parameter to InfraNodus API endpoint to generate a research question about content gaps.  
  - Configuration:  
    - POST method.  
    - URL dynamically includes gap depth parameter.  
    - Sends JSON body with parameters: aiTopics=true, requestMode=question, text=combined text.  
    - Requires InfraNodus API key with HTTP Bearer Authentication.  
  - Inputs: JSON with `text` and `randomNum` from previous node.  
  - Outputs: JSON containing AI-generated question and graph summary.  
  - Edge cases: API key missing or invalid, network timeouts, malformed request/response.

- **InfraNodus GraphRAG Response Generator**  
  - Type: `httpRequest`  
  - Role: Uses the question generated above to request an AI-generated response from InfraNodus, potentially incorporating other texts for cross-disciplinary insights.  
  - Configuration:  
    - POST method.  
    - Sends JSON body with parameters: aiTopics=true, requestMode=response, text (original combined text), prompt (question from previous node).  
    - Requires InfraNodus API key.  
  - Inputs: Outputs from "Prepare for InfraNodus" and question generator nodes.  
  - Outputs: AI-generated response JSON.  
  - Edge cases: Same as above, plus handling empty or unexpected AI response.

- **Sticky Notes (Steps 5, 6, 9)**  
  - Explain the InfraNodus GraphRAG process, API key requirement, and optional use of different document collections for answers.

#### 2.5 Output Presentation

**Overview:**  
Displays the generated research question and AI response back to the user on the original form interface.

**Nodes Involved:**  
- Display on the Form to the User  
- Sticky Note (Step 7)

**Node Details:**

- **Display on the Form to the User**  
  - Type: `form`  
  - Role: Sends back generated research question and AI response to the user interface as an HTML formatted output.  
  - Configuration:  
    - Uses unique webhook ID for response.  
    - Response text includes HTML headers and pulls in the question and response texts from the InfraNodus response node using expressions.  
  - Inputs: Output from InfraNodus Response Generator.  
  - Outputs: Response shown on the form page.  
  - Edge cases: Missing or malformed response data, expression resolution errors.

- **Sticky Note (Step 7)**  
  - Notes that this step shows the final question and answer, and can be extended to feed other workflows or apps.

---

### 3. Summary Table

| Node Name                          | Node Type           | Functional Role                                         | Input Node(s)                  | Output Node(s)                            | Sticky Note                                                                                                                    |
|-----------------------------------|---------------------|---------------------------------------------------------|-------------------------------|------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| On form submission                | formTrigger         | Entry point; receives user PDF uploads via form         | None                          | Convert binary files to PDF               | Step 1: User uploads the PDF files for analysis                                                                               |
| Convert binary files to PDF       | code                | Converts uploaded binaries into separate PDF files       | On form submission            | Extract text from PDF files               | Step 2: Convert uploaded binaries into PDF files                                                                              |
| Extract text from PDF files       | extractFromFile     | Extracts plain text content from PDFs                     | Convert binary files to PDF   | Prepare for InfraNodus                    | Step 3: Extract plain text from PDF files                                                                                     |
| Prepare for InfraNodus            | code                | Combines all extracted texts into one string and sets gap depth | Extract text from PDF files   | InfraNodus GraphRAG Question Generator   | Step 4: Combine extracted text into a text string                                                                             |
| InfraNodus GraphRAG Question Generator | httpRequest      | Sends text to InfraNodus API to generate research question | Prepare for InfraNodus        | InfraNodus GraphRAG Response Generator  | Step 5: Use InfraNodus GraphRAG to generate a research question based on content gaps                                          |
| InfraNodus GraphRAG Response Generator | httpRequest      | Uses question to generate AI response from InfraNodus API | InfraNodus GraphRAG Question Generator | Display on the Form to the User           | Step 6: Generate a Response using InfraNodus GraphRAG API <br> Step 9: Optional different document sets for answers           |
| Display on the Form to the User   | form                | Displays generated question and response to user          | InfraNodus GraphRAG Response Generator | None                                     | Step 7: Show question / prompt to the user                                                                                     |
| Convert File to PDF               | httpRequest         | Optional: high-quality PDF to text conversion via ConvertAPI | None (disabled)              | None                                     | Optional: Better PDF Conversion using ConvertAPI                                                                               |
| Sticky Note                      | stickyNote          | Various explanatory notes                                 | None                          | None                                     | Multiple sticky notes provide explanations for steps 1 through 7 and InfraNodus methodology                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node ("On form submission"):**  
   - Type: `formTrigger`  
   - Configure webhook with a unique webhook ID.  
   - Title the form "Find Content Gaps in Your PDF Files".  
   - Add a single file upload field accepting `.pdf` files only.  
   - Add description: "Upload the files you'd like to analyze and we will extract content gaps and interesting questions based on them."  

2. **Add a Code Node ("Convert binary files to PDF"):**  
   - Purpose: Convert each uploaded binary file into separate items with cleaned file name keys.  
   - JavaScript code:  
     ```js
     let results = [];
     for (let item of items) {
       if (item.binary) {
         for (let key in item.binary) {
           let binaryKey = key.replace(/\s/g, '_');
           results.push({
             json: { fileName: binaryKey },
             binary: { [binaryKey]: item.binary[key] }
           });
         }
       }
     }
     return results;
     ```  
   - Connect output of form trigger to this node.

3. **Add Extract From File Node ("Extract text from PDF files"):**  
   - Operation: `pdf`  
   - Binary Property Name: Use expression `={{ $json.fileName }}` to dynamically reference the binary data key.  
   - Connect output of the code node.

4. **(Optional) Add HTTP Request Node for ConvertAPI ("Convert File to PDF"):**  
   - Disabled by default.  
   - Configure with ConvertAPI URL and your API key.  
   - Use to replace or augment PDF text extraction if better quality is needed.

5. **Add a Code Node ("Prepare for InfraNodus"):**  
   - Combine all extracted texts into a single string with double newlines between documents.  
   - Generate a random integer between 0 and 2 for gap depth.  
   - Example code:  
     ```js
     let plainText = '';
     const randomNum = Math.floor(Math.random() * 3);
     for (let item of items) {
       plainText += item.json.text + '\n\n';
     }
     return { text: plainText, randomNum };
     ```  
   - Connect output of text extraction node here.

6. **Add HTTP Request Node ("InfraNodus GraphRAG Question Generator"):**  
   - Method: POST  
   - URL: `https://infranodus.com/api/v1/graphAndAdvice?doNotSave=true&optimize=develop&includeGraph=false&includeGraphSummary=true&gapDepth={{ $json.randomNum }}`  
   - Authentication: HTTP Bearer with InfraNodus API key credential.  
   - Body Parameters (JSON):  
     - aiTopics: true  
     - requestMode: question  
     - text: `={{ $json.text }}`  
   - Connect output of "Prepare for InfraNodus" here.

7. **Add HTTP Request Node ("InfraNodus GraphRAG Response Generator"):**  
   - Method: POST  
   - URL: `https://infranodus.com/api/v1/graphAndAdvice?doNotSave=true&optimize=imagine&includeGraph=false&includeGraphSummary=true`  
   - Authentication: HTTP Bearer with same InfraNodus API key.  
   - Body Parameters (JSON):  
     - aiTopics: true  
     - requestMode: response  
     - text: `={{ $('Prepare for InfraNodus').item.json.text }}`  
     - prompt: `={{ $json.aiAdvice[0].text }}` (the question generated previously)  
   - Connect output of Question Generator node here.

8. **Add Form Node ("Display on the Form to the User"):**  
   - Operation: `completion`  
   - Respond With: `showText`  
   - Response Text (HTML):  
     ```html
     <br>
     <h2>Question:</h2>
     <h3>{{ $('InfraNodus GraphRAG Question Generator').item.json.aiAdvice[0].text }}</h3>
     <br>
     <h2>Response:</h2>
     <h3>{{ $json.aiAdvice[0].text }}</h3>
     <br>
     ```  
   - Configure webhook ID to a unique value.  
   - Connect output of Response Generator node here.

9. **Credential Setup:**  
   - InfraNodus API Key credential with HTTP Bearer authentication.  
   - (Optional) ConvertAPI credential if using the optional ConvertAPI node.

10. **Test Workflow:**  
    - Deploy and expose the form trigger webhook URL.  
    - Upload PDF files via the form.  
    - Verify that the workflow extracts text, generates questions and responses, and shows output correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                            |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Optional better PDF conversion using [ConvertAPI](https://convertapi.com?ref=4l54n) gives higher-quality text respecting original document layout, avoiding short chunk splitting. Requires API key and correct data mapping.                        | Sticky Note 5; ConvertAPI documentation                     |
| InfraNodus GraphRAG generates research questions by constructing a knowledge graph from text, detecting least-connected concept clusters (structural gaps), and using AI to propose bridging questions. See: https://infranodus.com                  | Sticky Note 7; InfraNodus official site                      |
| You can use a different set of PDFs or saved InfraNodus expert graphs to generate answers for cross-disciplinary research, enhancing the diversity and depth of insights.                                                                             | Sticky Note 9                                               |
| The workflow is designed to handle multiple PDFs at once, combining their content for holistic analysis rather than isolated document processing.                                                                                                   | General workflow design                                     |
| API keys for InfraNodus must be securely stored and provided in the HTTP Request nodes' credentials section to enable interaction with the GraphRAG endpoints.                                                                                      | InfraNodus API usage                                        |
| The random gap depth parameter (0-2) controls the depth of the content gap search; setting it to 0 targets the biggest gap. Adjusting this can influence the novelty and focus of generated questions.                                                | Text preparation node explanation                           |
| The workflow is not activated by default; ensure to enable and test before production use.                                                                                                                                                            | Metadata and workflow status                                |

---

**Disclaimer:**  
The provided information comes exclusively from an n8n automated workflow. This processing strictly complies with content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.