Generate Comprehensive & Abstract Summaries from Jotform Data with Gemini AI

https://n8nworkflows.xyz/workflows/generate-comprehensive---abstract-summaries-from-jotform-data-with-gemini-ai-9367


# Generate Comprehensive & Abstract Summaries from Jotform Data with Gemini AI

### 1. Workflow Overview

This workflow is designed to generate both comprehensive and abstract summaries from data submitted via Jotform. It leverages Google Gemini AI (PaLM API) to produce two types of summaries for each submission:  
- **Comprehensive Summary:** A detailed, fact-preserving summary covering all key points.  
- **Abstract Summary:** A conceptual, generative summary that synthesizes and interprets the underlying meaning.

The workflow handles data reception from Jotform through a webhook, processes the text with AI, parses structured output, and persists the results in multiple storage locations including a DataTable, Google Sheets, and Google Docs. It includes rich explanatory notes to clarify the summarization types.

Logical blocks:  
- **1.1 Input Reception:** Receives Jotform submission data via webhook and extracts key fields.  
- **1.2 AI Summarization:** Sends extracted content to Google Gemini Chat Model for summarization.  
- **1.3 Output Parsing:** Parses the AI response into structured JSON with comprehensive and abstract summaries.  
- **1.4 Persistence:** Saves the summaries into a DataTable, Google Sheets, and creates/updates a Google Docs document.  
- **1.5 Documentation & Notes:** Provides visual sticky notes explaining the summarization logic and branding.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Captures incoming POST requests from Jotform submissions, extracts form title, submission ID, and the full submission body for further processing.

**Nodes Involved:**  
- Webhook  
- Set the Input Fields

**Node Details:**

- **Webhook**  
  - Type: Webhook (HTTP POST endpoint)  
  - Configuration: Listens for POST requests at path `/f3c34cda-d603-4923-883b-500576200322`  
  - Inputs: External HTTP POST from Jotform  
  - Outputs: JSON payload with submission data  
  - Potential Failures: Network connectivity issues, invalid/malformed JSON payloads, unauthorized requests (no auth set)  
  - Notes: Entry point of the workflow

- **Set the Input Fields**  
  - Type: Set node  
  - Configuration: Extracts and assigns:  
    - `FormTitle` from `body.formTitle`  
    - `SubmissionID` from `body.submissionID`  
    - `body` as a JSON string of the entire submission (`body.toJsonString()`)  
  - Inputs: Webhook output JSON  
  - Outputs: Structured JSON with simplified keys for downstream nodes  
  - Edge Cases: Missing expected keys in incoming JSON could cause empty or null values

---

#### 1.2 AI Summarization

**Overview:**  
Sends the extracted Jotform submission content to the Google Gemini Chat Model using a prompt focused on comprehensive summarization, then runs a summarization chain to generate both summary types.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Comprehensive & Abstract Summarizer  
- Structured Output Parser

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini Chat Model node  
  - Configuration: Uses model `"models/gemini-2.0-flash-exp"` with Google PaLM API credentials  
  - Inputs: Receives prompt from "Comprehensive & Abstract Summarizer" node's AI language model parameter  
  - Outputs: Raw AI response for parsing  
  - Potential Failures: API authentication errors, rate limits, model unavailability, network timeouts

- **Comprehensive & Abstract Summarizer**  
  - Type: Langchain Chain LLM node  
  - Configuration:  
    - Prompt: "Build a comprehensive summary of the following {{ $json.body.pretty }}"  
    - Message: "You are an expert comprehensive summarizer"  
    - Uses output parser (Structured Output Parser)  
  - Inputs: Receives processed input fields and AI model output  
  - Outputs: JSON containing both `comprehensive_summary` and `abstract_summary`  
  - Edge Cases: Model hallucinations, incomplete or malformed summary output

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Configuration: JSON schema example requiring keys:  
    - `comprehensive_summary` (string)  
    - `abstract_summary` (string)  
  - Inputs: Raw AI output from "Google Gemini Chat Model"  
  - Outputs: Structured JSON to be used downstream  
  - Potential Failures: Parsing errors if AI output deviates from schema

---

#### 1.3 Persistence

**Overview:**  
Stores the generated summaries into three different storage solutions to ensure availability and accessibility: a DataTable, Google Sheets, and Google Docs.

**Nodes Involved:**  
- Persist On DataTable  
- Append or update row in sheet  
- Create a document  
- Update a document

**Node Details:**

- **Persist On DataTable**  
  - Type: DataTable node  
  - Configuration:  
    - Stores `abstract_summary` and `comprehensive_summary` into a DataTable named "JotformRegistration"  
    - Mapping mode: explicit column definition  
  - Inputs: Structured summary JSON from summarizer node  
  - Outputs: None (final store)  
  - Edge Cases: DataTable access permission issues, schema mismatch

- **Append or update row in sheet**  
  - Type: Google Sheets node  
  - Configuration:  
    - Operation: Append or update  
    - Sheet: "Sheet1" in document with ID `1JOEbaPC2T06_O6Jb_UMbS7oG3z5RncY7Wk1gNAcpicg`  
    - Matches rows by `comprehensive_summary` column to update or append  
    - Stores both summaries in respective columns  
  - Inputs: Structured summary JSON  
  - Credentials: Google Sheets OAuth2 credentials required  
  - Potential Failures: API quota exceeded, invalid permissions, network issues

- **Create a document**  
  - Type: Google Docs node  
  - Configuration:  
    - Creates a new Google Doc named as `{FormTitle}-{SubmissionID}`  
    - Stores in default folder  
  - Inputs: Field values extracted in "Set the Input Fields" node  
  - Credentials: Google Docs OAuth2 credentials required  
  - Outputs: Newly created document metadata (including ID)  
  - Edge Cases: API limit errors, naming collisions, permission issues

- **Update a document**  
  - Type: Google Docs node (Update operation)  
  - Configuration:  
    - Inserts text into the newly created document  
    - Text includes both summaries labeled accordingly  
    - Document URL derived from the output of "Create a document" node  
  - Inputs: Document metadata and summary JSON  
  - Credentials: Google Docs OAuth2 credentials required  
  - Potential Failures: Document not found, permission denied, network errors

---

#### 1.4 Documentation & Notes

**Overview:**  
Provides visual sticky notes inside the workflow explaining the summarization types and branding.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2

**Node Details:**

- **Sticky Note**  
  - Content: Displays Jotform logo and brief explanation of using Google Gemini AI for summarization  
  - Visual aid, no inputs or outputs

- **Sticky Note1**  
  - Content: Explains the concept and ideal use cases of Comprehensive Summarization  
  - Covers completeness, factual coverage, and use cases like customer service and research

- **Sticky Note2**  
  - Content: Explains Abstract Summarization, emphasizing generative, interpretive summaries  
  - Highlights use cases like executive summaries and marketing content

---

### 3. Summary Table

| Node Name                    | Node Type                           | Functional Role                          | Input Node(s)              | Output Node(s)                                | Sticky Note                                                                                   |
|------------------------------|-----------------------------------|----------------------------------------|----------------------------|-----------------------------------------------|-----------------------------------------------------------------------------------------------|
| Webhook                      | Webhook                           | Receives Jotform submission data       | External HTTP POST          | Set the Input Fields                          |                                                                                               |
| Set the Input Fields         | Set                               | Extracts key fields from submission    | Webhook                    | Comprehensive & Abstract Summarizer           |                                                                                               |
| Google Gemini Chat Model     | Langchain LM Chat Google Gemini   | Provides AI model for summarization    | Connected via Summarizer AI | Comprehensive & Abstract Summarizer (model)  |                                                                                               |
| Comprehensive & Abstract Summarizer | Langchain Chain LLM             | Generates comprehensive and abstract summaries | Set the Input Fields; Google Gemini Chat Model; Structured Output Parser | Persist On DataTable; Append or update row in sheet; Create a document |                                                                                               |
| Structured Output Parser     | Langchain Output Parser Structured| Parses AI output into structured JSON  | Google Gemini Chat Model    | Comprehensive & Abstract Summarizer           |                                                                                               |
| Persist On DataTable         | DataTable                        | Stores summaries in DataTable           | Comprehensive & Abstract Summarizer | None                                      |                                                                                               |
| Append or update row in sheet| Google Sheets                    | Stores summaries in Google Sheets       | Comprehensive & Abstract Summarizer | None                                      |                                                                                               |
| Create a document            | Google Docs                      | Creates new Google Docs document         | Comprehensive & Abstract Summarizer | Update a document                          |                                                                                               |
| Update a document            | Google Docs                      | Inserts summaries into Google Doc        | Create a document          | None                                          |                                                                                               |
| Sticky Note                  | Sticky Note                     | Branding and workflow summary note       | None                       | None                                          | ![Logo](https://www.jotform.com/resources/assets/logo-nb/min/jotform-logo-white-400x200.png) Uses Google Gemini AI for the Comprehensive and Abstract Summarization of Jotform content |
| Sticky Note1                 | Sticky Note                     | Explains Comprehensive Summarization    | None                       | None                                          | Comprehensive Summarization focuses on covering all key points, ideal for reports and surveys |
| Sticky Note2                 | Sticky Note                     | Explains Abstract Summarization         | None                       | None                                          | Abstract Summarization is generative, ideal for executive summaries and marketing content     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `f3c34cda-d603-4923-883b-500576200322`  
   - No authentication configured  
   - Purpose: Receive Jotform submission JSON payload

2. **Create Set Node "Set the Input Fields"**  
   - Connect Webhook output to this node  
   - Assign three fields:  
     - `FormTitle`: Expression: `$json["body"]["formTitle"]`  
     - `SubmissionID`: Expression: `$json["body"]["submissionID"]`  
     - `body`: Expression: `$json["body"].toJsonString()` (convert entire body to JSON string)

3. **Create Google Gemini Chat Model Node**  
   - Type: Langchain Google Gemini Chat Model  
   - Model Name: `models/gemini-2.0-flash-exp`  
   - Credentials: Set up Google PaLM API credentials with appropriate access  
   - No other parameter changes needed

4. **Create Structured Output Parser Node**  
   - Type: Langchain Structured Output Parser  
   - Define JSON schema example:  
     ```json
     {
       "comprehensive_summary": "",
       "abstract_summary": ""
     }
     ```
   - This will parse AI responses into structured JSON with these keys

5. **Create Chain LLM Node "Comprehensive & Abstract Summarizer"**  
   - Connect "Set the Input Fields" node as input  
   - Use prompt: `Build a comprehensive summary of the following {{ $json.body.pretty }}`  
   - Add message: `"You are an expert comprehensive summarizer"`  
   - Reference the Google Gemini Chat Model node as AI language model  
   - Set output parser to "Structured Output Parser" node  
   - This node outputs JSON with comprehensive and abstract summary fields

6. **Create DataTable Node "Persist On DataTable"**  
   - Configure to store:  
     - `comprehensive_summary` from `{{$json.output.comprehensive_summary}}`  
     - `abstract_summary` from `{{$json.output.abstract_summary}}`  
   - Map to an existing or new DataTable named "JotformRegistration" with these columns  
   - Connect output of summarizer node here

7. **Create Google Sheets Node "Append or update row in sheet"**  
   - Operation: Append or Update  
   - Document ID: your Google Sheets document ID  
   - Sheet Name: the sheet where data is stored (e.g., "Sheet1")  
   - Matching column: `comprehensive_summary`  
   - Columns: same field mappings as DataTable  
   - Credentials: Google Sheets OAuth2  
   - Connect output of summarizer node here

8. **Create Google Docs Node "Create a document"**  
   - Operation: Create Document  
   - Title: Use expression `={{ $json.FormTitle }}-{{ $json.SubmissionID }}`  
   - Folder ID: default or specify a folder  
   - Credentials: Google Docs OAuth2  
   - Connect output of summarizer node here

9. **Create Google Docs Node "Update a document"**  
   - Operation: Update Document  
   - Document URL: Use expression from "Create a document" output, e.g., `={{ $json.id }}`  
   - Action: Insert text with both summaries:  
     ```
     Comprehensive Summary -  
     {{ $json.output.comprehensive_summary }}

     Abstract Summary -  
     {{ $json.output.abstract_summary }}
     ```  
   - Credentials: Google Docs OAuth2  
   - Connect output of "Create a document" node here

10. **Connect Workflow Nodes**  
    - Webhook → Set the Input Fields → Comprehensive & Abstract Summarizer  
    - Comprehensive & Abstract Summarizer → Persist On DataTable  
    - Comprehensive & Abstract Summarizer → Append or update row in sheet  
    - Comprehensive & Abstract Summarizer → Create a document → Update a document  
    - Google Gemini Chat Model used internally by summarizer node  
    - Structured Output Parser used internally by summarizer node

11. **Add Sticky Notes (Optional for clarity)**  
    - Add three sticky notes with content as per original workflow to explain summarization types and branding.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                  | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| ![Logo](https://www.jotform.com/resources/assets/logo-nb/min/jotform-logo-white-400x200.png) Uses Google Gemini AI for the Comprehensive and Abstract Summarization of Jotform content | Branding and workflow purpose note                                                              |
| Comprehensive Summarization focuses on covering all key points from the source text in a factual, detail-preserving way without new info. Ideal for customer service reports, surveys, etc. | Sticky Note1 content explaining comprehensive summarization                                     |
| Abstract Summarization is conceptual and generative, synthesizing and interpreting content for human-like overviews, ideal for executive summaries and marketing content.                   | Sticky Note2 content explaining abstract summarization                                          |

---

**Disclaimer:** The provided text is generated exclusively from an automated workflow created with n8n, respecting all content policies and legal standards. All data handled is public and lawful.