Simple Eval for Legal Benchmarking

https://n8nworkflows.xyz/workflows/simple-eval-for-legal-benchmarking-4712


# Simple Eval for Legal Benchmarking

---

### 1. Workflow Overview

This workflow, titled **Simple Eval for Legal Benchmarking**, is designed to evaluate the accuracy of AI-generated extractions from legal documents by benchmarking them against source PDFs. It targets use cases where legal AI outputs need to be systematically judged for factual correctness and completeness, enabling quality control and refinement of legal language models.

The workflow is structured into six logical blocks:

- **1.1 Input Reception and Test Case Fetching:** Trigger and retrieve test cases from a Google Sheet.
- **1.2 Filtering and Looping:** Filter test cases to those referencing PDFs and loop through each item.
- **1.3 Document Retrieval and Text Extraction:** Download PDFs from Google Drive and extract text content.
- **1.4 AI-Based Judgement:** Submit extracted text and AI outputs to an LLM judge that evaluates accuracy.
- **1.5 Result Parsing and Update:** Parse the structured LLM judge’s output and append results back to Google Sheets.
- **1.6 Rate Limiting Wait:** Introduce a delay to avoid API rate limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Test Case Fetching

- **Overview:**  
  This block starts the workflow manually and fetches the list of test cases from a specified Google Sheet. It serves as the entry point and data acquisition phase.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Get Tests (Google Sheets)  
  - Sticky Note (explanatory)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow manually on user command.  
    - *Config:* No parameters; simple manual trigger.  
    - *Connections:* Outputs to Get Tests node.  
    - *Potential Failures:* None typical; manual activation required.

  - **Get Tests**  
    - *Type:* Google Sheets  
    - *Role:* Retrieves rows from a Google Sheet containing test cases.  
    - *Config:* Reads from the sheet named “Tests” (gid=0) in the document with ID `10l_gMtPsge00eTTltGrgvAo54qhh3_twEDsETrQLAGU`.  
    - *Expressions:* Uses dynamic referencing for sheet name and document ID.  
    - *Connections:* Outputs to Is PDF? node.  
    - *Credentials:* Google Sheets OAuth2.  
    - *Edge Cases:* Google Sheets API quota exceeded; empty sheets; invalid document or sheet IDs.  
    - *Sticky Note:* Explains the test cases structure and data format (see section 3).

  - **Sticky Note (ID: 80a9da50-93df-476a-a39f-ffece642a934)**  
    - Content explains the source Google Sheet and filters for PDF documents.

#### 1.2 Filtering and Looping

- **Overview:**  
  Filters test cases to only those referencing PDF files and loops through each test case individually for processing.

- **Nodes Involved:**  
  - Is PDF? (If)  
  - Loop Over Items (Split In Batches)  
  - Sticky Notes (two notes explaining looping and filtering)

- **Node Details:**

  - **Is PDF?**  
    - *Type:* If (version 2.2)  
    - *Role:* Checks if the "Relevant Source Reference" field contains ".pdf" (case-sensitive contains check).  
    - *Config:* Condition checks substring presence in the filename.  
    - *Connections:* True branch to Loop Over Items; False branch ends flow for that item.  
    - *Edge Cases:* Missing or malformed source reference; case sensitivity could exclude PDFs with uppercase extensions.  
    - *Sticky Note:* See Sticky Note explaining filtering for PDFs.

  - **Loop Over Items**  
    - *Type:* Split In Batches  
    - *Role:* Processes one test case at a time sequentially to visualize workflow progress.  
    - *Config:* Batch size = 1, no reset on failure.  
    - *Connections:*  
      - Main output (empty array) goes nowhere (end).  
      - Secondary output connects to Google Drive node for file download.  
    - *Edge Cases:* Large datasets may slow workflow; batch size control critical for API limits.  
    - *Sticky Note:* Explains the looping mechanism.

  - **Sticky Notes:**  
    - Note 1: Explains the looping rationale.  
    - Note 2: Explains filtering for PDFs.

#### 1.3 Document Retrieval and Text Extraction

- **Overview:**  
  Downloads the referenced PDF from Google Drive, checks if a valid file was retrieved, and extracts its text content for evaluation.

- **Nodes Involved:**  
  - Google Drive (Download)  
  - Is a file? (If)  
  - Extract from File (PDF Extraction)  
  - Sticky Note (explaining extraction)

- **Node Details:**

  - **Google Drive**  
    - *Type:* Google Drive (version 3)  
    - *Role:* Downloads the file from Google Drive using a file ID extracted dynamically from the URL field.  
    - *Config:* Extracts fileId via regex from URL JSON attribute. Operation: download.  
    - *Credentials:* Google Drive OAuth2.  
    - *Connections:* Outputs to Is a file? node.  
    - *Edge Cases:* Invalid or expired file URLs; permission denied; download timeout; empty or corrupt files.

  - **Is a file?**  
    - *Type:* If (version 2.2)  
    - *Role:* Checks if the downloaded binary data is not empty to confirm a valid file.  
    - *Config:* Condition checks that binary data is not empty.  
    - *Connections:*  
      - True branch to Extract from File node.  
      - False branch back to Loop Over Items (skipping processing).  
    - *Edge Cases:* Corrupt or zero-byte downloads; false negatives if file is empty but valid.

  - **Extract from File**  
    - *Type:* Extract From File (version 1)  
    - *Role:* Extracts text from the downloaded PDF file for later evaluation.  
    - *Config:* Operation set to "pdf" extraction. Default options.  
    - *Connections:* Outputs to LLM Judge node.  
    - *Edge Cases:* Encrypted or scanned PDFs that cannot be extracted; extraction failures; malformed PDFs.

  - **Sticky Note:**  
    - Explains PDF download and text extraction step.

#### 1.4 AI-Based Judgement

- **Overview:**  
  Uses a language model via OpenRouter to judge the accuracy of AI-extracted information compared to the source PDF text. The LLM outputs a structured JSON judgement of Pass/Fail with reasoning.

- **Nodes Involved:**  
  - OpenRouter Chat Model (LM)  
  - LLM Judge (LangChain Chain LLM)  
  - Structured Output Parser (LangChain Output Parser)  
  - Sticky Note (explaining AI judging)

- **Node Details:**

  - **OpenRouter Chat Model**  
    - *Type:* LangChain LM Chat (OpenRouter)  
    - *Role:* Provides the GPT-4.1 language model interface via OpenRouter API.  
    - *Config:* Model set to "openai/gpt-4.1". No special options set.  
    - *Credentials:* OpenRouter API credentials configured.  
    - *Connections:* Connected as the language model input for LLM Judge node.  
    - *Edge Cases:* API rate limits; network errors; model unavailability; credential expiry.

  - **LLM Judge**  
    - *Type:* LangChain Chain LLM (version 1.4)  
    - *Role:* Sends prompt to the LLM to evaluate the AI output against the source text and returns JSON with decision and reasoning.  
    - *Config:*  
      - Prompt includes:  
        - Task input from current loop item’s "Input " field (note trailing space).  
        - Source text extracted from PDF.  
        - AI output from current loop item’s "Output " field.  
      - Instructions for the LLM to judge factual accuracy, ignoring style.  
      - Output parser attached to ensure JSON format.  
    - *Expressions:* Dynamic references to loop item fields and extracted text.  
    - *Connections:* Outputs to Update Results node.  
    - *Edge Cases:* Prompt errors; output not matching expected JSON schema; model hallucinations or misinterpretations.

  - **Structured Output Parser**  
    - *Type:* LangChain Output Parser Structured (version 1.2)  
    - *Role:* Enforces the LLM output to match a defined JSON schema with fields "reasoning" and "decision".  
    - *Config:* Example JSON schema provided in parameters for validation.  
    - *Connections:* Feeds parsed output into LLM Judge node.  
    - *Edge Cases:* Parsing failures if LLM output invalid or malformed.

  - **Sticky Note:**  
    - Describes the judgment logic and use of OpenRouter for LLM selection.

#### 1.5 Result Parsing and Update

- **Overview:**  
  Appends the original test data along with the LLM judge's decision and reasoning into a "Results" Google Sheet for record-keeping and further analysis.

- **Nodes Involved:**  
  - Update Results (Google Sheets)  
  - Sticky Note (explaining results update)

- **Node Details:**

  - **Update Results**  
    - *Type:* Google Sheets (version 4.5)  
    - *Role:* Appends a new row to the "Results" sheet in the same Google Sheet document.  
    - *Config:*  
      - Maps columns from the Loop Over Items input JSON and LLM Judge output JSON to sheet columns: ID, URL, Input, Output, Decision, Reasoning, Test No., AI Platform, Relevant Source Reference.  
      - Sheet ID: 537199982 (Results tab).  
      - Append operation used.  
    - *Credentials:* Google Sheets OAuth2.  
    - *Connections:* Outputs to Wait node.  
    - *Edge Cases:* API quota limits; sheet locking; malformed data causing append failures.

  - **Sticky Note:**  
    - Explains creation of new rows with judgment results.

#### 1.6 Rate Limiting Wait

- **Overview:**  
  Introduces a short delay after each append operation to avoid hitting OpenAI API rate limits.

- **Nodes Involved:**  
  - Wait (Delay)  
  - Sticky Note (explaining delay)

- **Node Details:**

  - **Wait**  
    - *Type:* Wait (version 1.1)  
    - *Role:* Pauses workflow for 0.5 seconds before processing next batch/item.  
    - *Config:* Fixed wait time 0.5 seconds.  
    - *Connections:* Loops back to Loop Over Items node to process next item.  
    - *Edge Cases:* Insufficient delay may cause rate limit errors; too long delays slow down processing.

  - **Sticky Note:**  
    - Explains the purpose of waiting to prevent API rate limit errors.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                         | Input Node(s)              | Output Node(s)         | Sticky Note                                                                                                      |
|----------------------------|----------------------------------|---------------------------------------|----------------------------|------------------------|-----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Starts workflow manually               |                            | Get Tests               |                                                                                                                 |
| Get Tests                  | Google Sheets                    | Fetches test cases from Google Sheet  | When clicking ‘Test workflow’ | Is PDF?                | Explains test cases sheet structure and data format (ID, Test No., AI Platform, Relevant Source, URL, Input, Output) |
| Is PDF?                    | If                              | Filters test cases for PDFs            | Get Tests                  | Loop Over Items         | Explains filtering to only PDF files                                                                            |
| Loop Over Items            | Split In Batches                 | Loops through test cases one at a time| Is PDF?                    | Google Drive            | Explains looping mechanism                                                                                       |
| Google Drive               | Google Drive                    | Downloads file from Google Drive       | Loop Over Items            | Is a file?              | Explains PDF download and extraction                                                                             |
| Is a file?                 | If                              | Checks valid file was downloaded       | Google Drive               | Extract from File; Loop Over Items |                                                                                                                 |
| Extract from File          | Extract From File                | Extracts text from PDF                  | Is a file?                 | LLM Judge               |                                                                                                                 |
| OpenRouter Chat Model      | LangChain LM Chat (OpenRouter)  | Provides GPT-4.1 model via OpenRouter |                            | LLM Judge (ai_languageModel) | Explains AI judging with OpenRouter and LLM                                                                      |
| Structured Output Parser   | LangChain Output Parser          | Parses LLM output to JSON               |                            | LLM Judge (ai_outputParser) |                                                                                                                 |
| LLM Judge                  | LangChain Chain LLM             | Judges AI output accuracy               | Extract from File; OpenRouter Chat Model; Structured Output Parser | Update Results           |                                                                                                                 |
| Update Results             | Google Sheets                   | Appends judgment results to sheet      | LLM Judge                  | Wait                    | Explains updating results in Google Sheet                                                                        |
| Wait                      | Wait                            | Pauses workflow to avoid rate limits   | Update Results             | Loop Over Items          | Explains 0.5s wait to prevent OpenAI API rate limit errors                                                      |
| Sticky Note (1 - 6)        | Sticky Note                     | Explanatory notes                      |                            |                        | See detailed notes linked to nodes above                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters needed  
   - Name: "When clicking ‘Test workflow’"

2. **Add Google Sheets Node to Get Test Cases**  
   - Operation: Read rows  
   - Document ID: `10l_gMtPsge00eTTltGrgvAo54qhh3_twEDsETrQLAGU`  
   - Sheet Name: `gid=0` (Tests sheet)  
   - Credentials: Google Sheets OAuth2  
   - Name: "Get Tests"  
   - Connect output from Manual Trigger to this node.

3. **Add If Node “Is PDF?” to Filter Only PDF Files**  
   - Condition: Check if `Relevant Source Reference` contains `.pdf` (case-sensitive)  
   - Version: 2.2  
   - Name: "Is PDF?"  
   - Connect output of Get Tests to this node.

4. **Add Split In Batches Node to Loop Over Items**  
   - Batch Size: 1 (process one test case at a time)  
   - Options: Reset off  
   - Name: "Loop Over Items"  
   - Connect True output of Is PDF? to this node.

5. **Add Google Drive Node to Download File**  
   - Operation: Download  
   - File ID: Extract from URL field using expression: `{{$json["URL"].match(/[-\w]{25,}/)[0]}}`  
   - Credentials: Google Drive OAuth2  
   - Name: "Google Drive"  
   - Connect output of Loop Over Items (secondary output) to this node.

6. **Add If Node “Is a file?” to Check File Validity**  
   - Condition: Check binary data is not empty  
   - Version: 2.2  
   - Name: "Is a file?"  
   - Connect output of Google Drive to this node.

7. **Add Extract From File Node to Extract Text from PDF**  
   - Operation: PDF extraction  
   - Name: "Extract from File"  
   - Connect True output of Is a file? to this node.

8. **Add LangChain OpenRouter Chat Model Node**  
   - Model: `openai/gpt-4.1`  
   - Credentials: OpenRouter API configured  
   - Name: "OpenRouter Chat Model"

9. **Add LangChain Structured Output Parser Node**  
   - Provide JSON schema example with fields: `decision` and `reasoning`  
   - Name: "Structured Output Parser"

10. **Add LangChain Chain LLM Node “LLM Judge”**  
    - Prompt:  
      ```
      INPUT:
      {
        "task": {{ $('Loop Over Items').item.json['Input '] }},
        "source": {{ $json.text }},
        "output": {{ $('Loop Over Items').item.json['Output '] }}
      }
      OUTPUT:
      ```
    - Messages: Detailed instructions for legal evaluator (see node details)  
    - Attach OpenRouter Chat Model as language model  
    - Attach Structured Output Parser for output  
    - Name: "LLM Judge"  
    - Connect Extract from File to LLM Judge input  
    - Connect OpenRouter Chat Model and Structured Output Parser to LLM Judge ai_languageModel and ai_outputParser inputs respectively.

11. **Add Google Sheets Node “Update Results”**  
    - Operation: Append row  
    - Document ID: Same as "Get Tests"  
    - Sheet Name: `537199982` (Results tab)  
    - Map columns: Copy fields from Loop Over Items and add `decision` and `reasoning` from LLM Judge output  
    - Credentials: Google Sheets OAuth2  
    - Name: "Update Results"  
    - Connect output of LLM Judge to this node.

12. **Add Wait Node to Delay Processing**  
    - Wait time: 0.5 seconds  
    - Name: "Wait"  
    - Connect output of Update Results to this node.

13. **Connect Wait Node back to Loop Over Items to Process Next Item**

14. **Add Connections for False Branches:**  
    - From Is PDF? False branch: No connection (skip non-PDF tests)  
    - From Is a file? False branch: Connect back to Loop Over Items to continue with next item

15. **Add Sticky Notes** to describe each logical block and important configurations for clarity and documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                       | Context or Link                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Workflow relies on a Google Sheets document with structured columns: ID, Test No., AI Platform, Relevant Source, URL, Input, Output.                             | Google Sheets document: https://docs.google.com/spreadsheets/d/10l_gMtPsge00eTTltGrgvAo54qhh3_twEDsETrQLAGU/edit    |
| Uses OpenRouter as a flexible interface to GPT-4.1, allowing easy swapping/tuning of LLM backends.                                                               | OpenRouter Documentation: https://openrouter.ai                                                                      |
| Legal evaluator prompt is carefully designed to judge factual accuracy ignoring style/format, focusing on omissions, hallucinations, and distortions only.       |                                                                                                                     |
| Rate limiting is handled by a fixed 0.5-second wait after each API call to prevent OpenAI throttling or errors.                                                  |                                                                                                                     |
| The workflow is designed for PDFs only; DOCX or other formats require separate handling.                                                                          |                                                                                                                     |

---

*Disclaimer: The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.*