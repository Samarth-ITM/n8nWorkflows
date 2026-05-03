Analyze YouTube videos and auto-generate AI reports in Google Docs with DeepSeek

https://n8nworkflows.xyz/workflows/analyze-youtube-videos-and-auto-generate-ai-reports-in-google-docs-with-deepseek-13887


# Analyze YouTube videos and auto-generate AI reports in Google Docs with DeepSeek

# 1. Workflow Overview

This workflow accepts a YouTube URL from an n8n form, retrieves the video transcript through Supadata.ai, sends that transcript to an AI agent powered by OpenRouter/DeepSeek for structured analysis, extracts a document title from the generated analysis, and finally creates a Google Docs file containing the report.

Its main use case is automated video review and documentation. It is suitable for teams or individuals who want to convert YouTube content into reusable written reports without manually transcribing, summarizing, and formatting the findings.

## 1.1 Input Reception

The workflow starts from a form-based trigger where a user submits a YouTube URL. This is the only actual execution entry point in the current workflow.

## 1.2 Transcript Retrieval

The submitted URL is passed to an HTTP Request node that calls the Supadata.ai transcript API. The returned transcript text becomes the analysis input.

## 1.3 AI Analysis and Metadata Extraction

The transcript is sent to a LangChain Agent node configured with an OpenRouter chat model using DeepSeek. The agent produces a structured plain-text report. A second AI-based extraction node then derives a document title from that generated report.

## 1.4 Result Consolidation

The extracted title and the generated report are combined through a Merge node and then reshaped with an Aggregate node so downstream Google Docs nodes can access both values cleanly.

## 1.5 Google Docs Creation and Population

A new Google Docs document is created using the extracted title. The report body is then inserted into the created document.

---

# 2. Block-by-Block Analysis

## Block 1: Input Reception

### Overview

This block collects the YouTube URL from the end user through an n8n-hosted form. It serves as the workflow’s trigger and provides the initial payload used by the transcription API.

### Nodes Involved

- Upload File or YouTube Link

### Node Details

#### Upload File or YouTube Link

- **Type and technical role:** `n8n-nodes-base.formTrigger`
  - Form-based trigger node that starts the workflow when a user submits the form.
- **Configuration choices:**
  - Form title: `Video Link`
  - One form field named `youtube_url`
  - Placeholder indicates the field is optional: `Enter YouTube URL (optional)`
- **Key expressions or variables used:**
  - Outputs form data as JSON, including `youtube_url`
- **Input and output connections:**
  - No input, as it is the trigger
  - Output goes to `Transcription using Supadata.ai`
- **Version-specific requirements:**
  - Uses `typeVersion 2.2`, so the node should be available in relatively recent n8n versions supporting Form Trigger
- **Edge cases or potential failure types:**
  - If the field is left empty, downstream transcription will likely fail because the API URL depends on `youtube_url`
  - Invalid YouTube links may still pass the form but fail at the transcript API
  - Public form access settings may matter depending on deployment
- **Sub-workflow reference:**
  - None

---

## Block 2: Transcript Retrieval

### Overview

This block sends the submitted YouTube URL to Supadata.ai and attempts to fetch a transcript. The transcript text is the foundational data used by the analysis agent.

### Nodes Involved

- Transcription using Supadata.ai

### Node Details

#### Transcription using Supadata.ai

- **Type and technical role:** `n8n-nodes-base.httpRequest`
  - Makes an HTTP API call to retrieve a YouTube transcript from Supadata.ai.
- **Configuration choices:**
  - Method is implicitly GET
  - URL is expression-based and includes the submitted YouTube URL
  - A custom header `x-api-key` is required for authentication
  - Query sending is enabled, but no meaningful query parameter is configured
- **Key expressions or variables used:**
  - URL expression:
    - `https://api.supadata.ai/v1/youtube/transcript?url={{ $json.youtube_url }}/TLEz9eiSc5s?v=dQw4w9WgXcQ&text=true`
  - Header:
    - `x-api-key: <API Key>`
- **Input and output connections:**
  - Input from `Upload File or YouTube Link`
  - Output to `Analyser`
- **Version-specific requirements:**
  - Uses `typeVersion 4.2`
- **Edge cases or potential failure types:**
  - The URL expression appears malformed. It appends `/TLEz9eiSc5s?v=dQw4w9WgXcQ&text=true` after `{{ $json.youtube_url }}`, which is likely unintended and may break valid requests
  - If `youtube_url` is missing, the request may produce an invalid API call
  - Invalid or private YouTube videos may not be transcribable
  - Missing or incorrect API key causes 401/403 authentication errors
  - Rate limits or API downtime may interrupt execution
  - Transcript response structure must contain fields expected downstream, especially `text` or `content`
- **Sub-workflow reference:**
  - None

---

## Block 3: AI Analysis and Metadata Extraction

### Overview

This block transforms the transcript into a structured written report and derives a document title from that report. It contains the workflow’s core intelligence layer and relies on an OpenRouter-hosted DeepSeek model.

### Nodes Involved

- OpenRouter Chat Model
- Analyser
- File Name Detector

### Node Details

#### OpenRouter Chat Model

- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`
  - Provides the chat language model used by LangChain AI nodes.
- **Configuration choices:**
  - Model: `deepseek/deepseek-r1-distill-llama-70b`
  - No advanced options are set
  - Uses OpenRouter API credentials
- **Key expressions or variables used:**
  - No dynamic expressions
- **Input and output connections:**
  - Connected through `ai_languageModel` to:
    - `Analyser`
    - `File Name Detector`
- **Version-specific requirements:**
  - Uses `typeVersion 1`
  - Requires n8n with LangChain AI node support and OpenRouter integration
- **Edge cases or potential failure types:**
  - Invalid OpenRouter credentials
  - Model unavailability or routing errors
  - Token/context overrun if transcripts are very large
  - Provider-side throttling or timeout
- **Sub-workflow reference:**
  - None

#### Analyser

- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`
  - AI agent node that takes transcript text and generates a structured report.
- **Configuration choices:**
  - Prompt is explicitly defined
  - Output parser is enabled
  - Retry on fail is enabled
  - Prompt instructs the model to produce plain text only, with labeled sections:
    - Executive Summary
    - Key Topics and Themes
    - Important Insights
    - Content Structure
    - Technical Quality
    - Recommendations
  - Prompt also instructs the model to state when analysis cannot be performed due to missing content
- **Key expressions or variables used:**
  - Main prompt includes:
    - `{{ $json.text }}{{ $json.content }}`
  - This indicates the node expects transcript data under either `text` or `content`
- **Input and output connections:**
  - Input from `Transcription using Supadata.ai`
  - AI language model input from `OpenRouter Chat Model`
  - Main outputs to:
    - `File Name Detector`
    - `Merge` input 1
- **Version-specific requirements:**
  - Uses `typeVersion 1.8`
  - Requires AI Agent functionality in n8n
- **Edge cases or potential failure types:**
  - If the transcription node does not return `text` or `content`, the prompt receives empty content
  - Large transcripts may exceed model context limits
  - Output format may still deviate from the plain-text instruction
  - Retry may help transient provider errors, but not malformed input
- **Sub-workflow reference:**
  - None

#### File Name Detector

- **Type and technical role:** `@n8n/n8n-nodes-langchain.informationExtractor`
  - Structured extraction node that derives a title from the generated report.
- **Configuration choices:**
  - Extracts from `{{ $json.output }}`
  - System prompt says to only extract relevant information and omit unknown values
  - One required attribute:
    - `Title` described as `Document Title`
  - Retry on fail enabled
- **Key expressions or variables used:**
  - Input text:
    - `{{ $json.output }}`
- **Input and output connections:**
  - Input from `Analyser`
  - AI language model input from `OpenRouter Chat Model`
  - Output to `Merge` input 0
- **Version-specific requirements:**
  - Uses `typeVersion 1`
  - Requires LangChain extraction support in n8n
- **Edge cases or potential failure types:**
  - If `Analyser` fails or returns empty output, title extraction may fail
  - Because `Title` is required, the node may produce unstable or invented values when the report lacks a clear title
  - Different extraction node versions may return arrays or objects; downstream logic assumes `output.Title`
- **Sub-workflow reference:**
  - None

---

## Block 4: Result Consolidation

### Overview

This block combines the title extraction result with the original analysis output and reshapes the data so the Google Docs nodes can consume a title and report body. It is essentially a data preparation layer.

### Nodes Involved

- Merge
- Aggregate

### Node Details

#### Merge

- **Type and technical role:** `n8n-nodes-base.merge`
  - Combines two streams:
    - title extraction output
    - full analysis output
- **Configuration choices:**
  - Default merge behavior is used because no custom parameters are defined
- **Key expressions or variables used:**
  - None directly in configuration
- **Input and output connections:**
  - Input 0 from `File Name Detector`
  - Input 1 from `Analyser`
  - Output to `Aggregate`
- **Version-specific requirements:**
  - Uses `typeVersion 3.1`
  - Merge behavior depends on n8n version and default mode
- **Edge cases or potential failure types:**
  - If one input produces no item, merge may not emit output depending on default mode
  - If both branches output multiple items unexpectedly, pairing behavior may become inconsistent
  - This node assumes one item from each source
- **Sub-workflow reference:**
  - None

#### Aggregate

- **Type and technical role:** `n8n-nodes-base.aggregate`
  - Reshapes merged data into named fields used downstream.
- **Configuration choices:**
  - Aggregates:
    - `output.Title` into `Title`
    - `output` into `Summary`
  - Rename field enabled for both
- **Key expressions or variables used:**
  - Aggregation source fields:
    - `output.Title`
    - `output`
- **Input and output connections:**
  - Input from `Merge`
  - Output to `Creating New File`
- **Version-specific requirements:**
  - Uses `typeVersion 1`
- **Edge cases or potential failure types:**
  - The field path assumptions are fragile
  - If `Merge` outputs a different structure, `output.Title` may not exist
  - `Creating New File` later uses `{{ $json.Title[0] }}`, implying aggregated `Title` is an array
  - If no title is extracted, array access may fail or produce empty title
- **Sub-workflow reference:**
  - None

---

## Block 5: Google Docs Creation and Population

### Overview

This block creates a new Google Docs file and inserts the analysis report into it. It is the storage and delivery layer of the workflow.

### Nodes Involved

- Creating New File
- Updating Content in File

### Node Details

#### Creating New File

- **Type and technical role:** `n8n-nodes-base.googleDocs`
  - Creates a new Google Docs document.
- **Configuration choices:**
  - Title is set dynamically from `{{ $json.Title[0] }}`
  - Folder ID is fixed: `191GOMY_IVzDJtDeHc5j65MOUWpC623LD`
  - Uses Google Docs OAuth2 credentials
- **Key expressions or variables used:**
  - Title:
    - `{{ $json.Title[0] }}`
- **Input and output connections:**
  - Input from `Aggregate`
  - Output to `Updating Content in File`
- **Version-specific requirements:**
  - Uses `typeVersion 2`
  - Requires Google Docs node support and valid OAuth2 credentials
- **Edge cases or potential failure types:**
  - If `Title[0]` is undefined, document creation may fail or create an untitled file
  - Folder ID must exist and be writable by the authenticated Google account
  - OAuth scopes must permit file creation
- **Sub-workflow reference:**
  - None

#### Updating Content in File

- **Type and technical role:** `n8n-nodes-base.googleDocs`
  - Updates the newly created Google Doc by inserting analysis text.
- **Configuration choices:**
  - Operation: `update`
  - Document URL field receives `{{ $json.id }}`
  - Action list contains one insert action with text from the `Analyser` node
- **Key expressions or variables used:**
  - Document URL:
    - `{{ $json.id }}`
  - Insert text:
    - `{{ $('Analyser').item.json.output }}`
- **Input and output connections:**
  - Input from `Creating New File`
  - No downstream node
- **Version-specific requirements:**
  - Uses `typeVersion 2`
- **Edge cases or potential failure types:**
  - Parameter name says `documentURL`, but the expression passes `id`; compatibility depends on node implementation
  - Cross-node reference `$('Analyser').item.json.output` assumes a corresponding item is available in execution context
  - If the Google Doc is created but not accessible immediately, rare timing issues may occur
  - Insert action may fail if content is too large or if the API rejects formatting assumptions
- **Sub-workflow reference:**
  - None

---

## Non-Executable Nodes: Sticky Notes

These nodes do not process data but provide embedded documentation.

### Sticky Note

- Covers the transcription section
- Content explains that the workflow accepts a YouTube link or uploaded video, transcribes it, and passes the transcript to AI analysis

### Sticky Note1

- Covers the AI analysis section
- Content explains that the AI agent reads the transcript and generates a structured report plus metadata

### Sticky Note2

- Covers the storage section
- Content explains that a Google Doc is created and populated with the analysis

### Sticky Note3

- Covers the overall workflow
- Includes a general description, high-level steps, and useful links:
  - Demo & Setup Video: https://drive.google.com/file/d/1g0rY7bbZrsFLnZjDfhXURR5WViywnYNw/view?usp=sharing
  - Course: https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Upload File or YouTube Link | n8n-nodes-base.formTrigger | Entry point that collects the YouTube URL from the user |  | Transcription using Supadata.ai | A compact n8n workflow that accepts a YouTube link or uploaded video, pulls a transcript via Supadata.ai, runs a language-model-based video analysis agent to produce a structured report, extracts a title/metadata, then creates and updates a Google Doc with the analysis. It's designed to automate transcription → analysis → document creation for fast, repeatable video reviews. |
| Upload File or YouTube Link | n8n-nodes-base.formTrigger | Entry point that collects the YouTube URL from the user |  | Transcription using Supadata.ai | # How it works |
| Upload File or YouTube Link | n8n-nodes-base.formTrigger | Entry point that collects the YouTube URL from the user |  | Transcription using Supadata.ai | 1. Trigger — Upload File or YouTube Link |
| Upload File or YouTube Link | n8n-nodes-base.formTrigger | Entry point that collects the YouTube URL from the user |  | Transcription using Supadata.ai | 2. Transcription — Transcription using Supadata.ai |
| Upload File or YouTube Link | n8n-nodes-base.formTrigger | Entry point that collects the YouTube URL from the user |  | Transcription using Supadata.ai | 3. Analysis — Analyser |
| Upload File or YouTube Link | n8n-nodes-base.formTrigger | Entry point that collects the YouTube URL from the user |  | Transcription using Supadata.ai | 4. Metadata extraction — File Name Detector |
| Upload File or YouTube Link | n8n-nodes-base.formTrigger | Entry point that collects the YouTube URL from the user |  | Transcription using Supadata.ai | 5. Aggregation & Merge |
| Upload File or YouTube Link | n8n-nodes-base.formTrigger | Entry point that collects the YouTube URL from the user |  | Transcription using Supadata.ai | 6. Document Creation |
| Upload File or YouTube Link | n8n-nodes-base.formTrigger | Entry point that collects the YouTube URL from the user |  | Transcription using Supadata.ai | Demo & Setup Video: https://drive.google.com/file/d/1g0rY7bbZrsFLnZjDfhXURR5WViywnYNw/view?usp=sharing |
| Upload File or YouTube Link | n8n-nodes-base.formTrigger | Entry point that collects the YouTube URL from the user |  | Transcription using Supadata.ai | Course: https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC |
| Transcription using Supadata.ai | n8n-nodes-base.httpRequest | Retrieves transcript text from Supadata.ai | Upload File or YouTube Link | Analyser | ### Transcribing |
| Transcription using Supadata.ai | n8n-nodes-base.httpRequest | Retrieves transcript text from Supadata.ai | Upload File or YouTube Link | Analyser | The workflow starts by accepting a YouTube link or uploaded video. |
| Transcription using Supadata.ai | n8n-nodes-base.httpRequest | Retrieves transcript text from Supadata.ai | Upload File or YouTube Link | Analyser | It then sends the video to the transcription API to convert the audio into text. |
| Transcription using Supadata.ai | n8n-nodes-base.httpRequest | Retrieves transcript text from Supadata.ai | Upload File or YouTube Link | Analyser | This transcript becomes the input for the AI analysis. |
| OpenRouter Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Supplies the LLM used by AI nodes |  | Analyser, File Name Detector | ### Analyzing |
| OpenRouter Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Supplies the LLM used by AI nodes |  | Analyser, File Name Detector | The transcript is passed to an AI agent that acts as a video analyst. |
| OpenRouter Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Supplies the LLM used by AI nodes |  | Analyser, File Name Detector | It reads the full transcript, extracts insights, and generates a structured analysis report along with key metadata like the title. |
| Analyser | @n8n/n8n-nodes-langchain.agent | Generates a structured analysis report from transcript content | Transcription using Supadata.ai; OpenRouter Chat Model | File Name Detector, Merge | ### Analyzing |
| Analyser | @n8n/n8n-nodes-langchain.agent | Generates a structured analysis report from transcript content | Transcription using Supadata.ai; OpenRouter Chat Model | File Name Detector, Merge | The transcript is passed to an AI agent that acts as a video analyst. |
| Analyser | @n8n/n8n-nodes-langchain.agent | Generates a structured analysis report from transcript content | Transcription using Supadata.ai; OpenRouter Chat Model | File Name Detector, Merge | It reads the full transcript, extracts insights, and generates a structured analysis report along with key metadata like the title. |
| File Name Detector | @n8n/n8n-nodes-langchain.informationExtractor | Extracts a document title from the analysis output | Analyser; OpenRouter Chat Model | Merge | ### Analyzing |
| File Name Detector | @n8n/n8n-nodes-langchain.informationExtractor | Extracts a document title from the analysis output | Analyser; OpenRouter Chat Model | Merge | The transcript is passed to an AI agent that acts as a video analyst. |
| File Name Detector | @n8n/n8n-nodes-langchain.informationExtractor | Extracts a document title from the analysis output | Analyser; OpenRouter Chat Model | Merge | It reads the full transcript, extracts insights, and generates a structured analysis report along with key metadata like the title. |
| Merge | n8n-nodes-base.merge | Combines title extraction and report output | File Name Detector, Analyser | Aggregate | ### Storing |
| Merge | n8n-nodes-base.merge | Combines title extraction and report output | File Name Detector, Analyser | Aggregate | Finally, the workflow creates a Google Docs file using the extracted title. |
| Merge | n8n-nodes-base.merge | Combines title extraction and report output | File Name Detector, Analyser | Aggregate | The generated analysis report is inserted into the document, allowing easy sharing, documentation, and future reference. |
| Aggregate | n8n-nodes-base.aggregate | Reshapes merged data into title and summary fields | Merge | Creating New File | ### Storing |
| Aggregate | n8n-nodes-base.aggregate | Reshapes merged data into title and summary fields | Merge | Creating New File | Finally, the workflow creates a Google Docs file using the extracted title. |
| Aggregate | n8n-nodes-base.aggregate | Reshapes merged data into title and summary fields | Merge | Creating New File | The generated analysis report is inserted into the document, allowing easy sharing, documentation, and future reference. |
| Creating New File | n8n-nodes-base.googleDocs | Creates a new Google Docs file | Aggregate | Updating Content in File | ### Storing |
| Creating New File | n8n-nodes-base.googleDocs | Creates a new Google Docs file | Aggregate | Updating Content in File | Finally, the workflow creates a Google Docs file using the extracted title. |
| Creating New File | n8n-nodes-base.googleDocs | Creates a new Google Docs file | Aggregate | Updating Content in File | The generated analysis report is inserted into the document, allowing easy sharing, documentation, and future reference. |
| Updating Content in File | n8n-nodes-base.googleDocs | Inserts the analysis report into the created Google Doc | Creating New File |  | ### Storing |
| Updating Content in File | n8n-nodes-base.googleDocs | Inserts the analysis report into the created Google Doc | Creating New File |  | Finally, the workflow creates a Google Docs file using the extracted title. |
| Updating Content in File | n8n-nodes-base.googleDocs | Inserts the analysis report into the created Google Doc | Creating New File |  | The generated analysis report is inserted into the document, allowing easy sharing, documentation, and future reference. |
| Sticky Note | n8n-nodes-base.stickyNote | Visual documentation for transcription block |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual documentation for analysis block |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual documentation for storage block |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual documentation for full workflow overview |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it `Video Analysis Agent`.

2. **Add a Form Trigger node**
   - Node type: `Form Trigger`
   - Name it `Upload File or YouTube Link`
   - Set the form title to `Video Link`
   - Add one field:
     - Label: `youtube_url`
     - Placeholder: `Enter YouTube URL (optional)`
   - This will be the workflow entry point.

3. **Add an HTTP Request node for transcription**
   - Node type: `HTTP Request`
   - Name it `Transcription using Supadata.ai`
   - Connect it from `Upload File or YouTube Link`
   - Configure:
     - Method: `GET`
     - URL: use an expression that passes the submitted URL to Supadata.ai
   - Recommended correct form:
     - `https://api.supadata.ai/v1/youtube/transcript?url={{ $json.youtube_url }}&text=true`
   - Add a header:
     - `x-api-key` = your Supadata API key
   - Keep response format as JSON if the API returns structured JSON.

4. **Add the OpenRouter chat model node**
   - Node type: `OpenRouter Chat Model`
   - Name it `OpenRouter Chat Model`
   - Select model:
     - `deepseek/deepseek-r1-distill-llama-70b`
   - Create or attach OpenRouter credentials:
     - API key from your OpenRouter account

5. **Add an AI Agent node**
   - Node type: `AI Agent`
   - Name it `Analyser`
   - Connect the main input from `Transcription using Supadata.ai`
   - Connect the `OpenRouter Chat Model` node to its `ai_languageModel` port
   - Set prompt mode to defined text
   - Paste a prompt equivalent to:
     - Instruct the model to act as an expert video analyst
     - Use transcript content from `{{ $json.text }}{{ $json.content }}`
     - Return a plain-text structured report
     - Include sections:
       - Executive Summary
       - Key Topics and Themes
       - Important Insights
       - Content Structure
       - Technical Quality
       - Recommendations
     - Instruct it not to use markdown symbols
     - Instruct it to say analysis cannot be performed if content is missing
   - Enable output parser if your node/version supports it
   - Enable retry on fail

6. **Add an Information Extractor node**
   - Node type: `Information Extractor`
   - Name it `File Name Detector`
   - Connect it from `Analyser`
   - Connect `OpenRouter Chat Model` to its `ai_languageModel` input as well
   - Configure text input:
     - `{{ $json.output }}`
   - Add a system prompt similar to:
     - “You are an expert extraction algorithm. Only extract relevant information from the text. If you do not know the value of an attribute asked to extract, you may omit the attribute's value.”
   - Define one attribute:
     - Name: `Title`
     - Required: true
     - Description: `Document Title`
   - Enable retry on fail

7. **Add a Merge node**
   - Node type: `Merge`
   - Name it `Merge`
   - Connect:
     - `File Name Detector` to input 1 or input 0
     - `Analyser` to the other input
   - Keep the default mode if you expect one item from each branch
   - Verify during testing that the merged structure contains both the extractor output and the analyzer output

8. **Add an Aggregate node**
   - Node type: `Aggregate`
   - Name it `Aggregate`
   - Connect it from `Merge`
   - Configure fields to aggregate:
     - Aggregate `output.Title` and rename output field to `Title`
     - Aggregate `output` and rename output field to `Summary`
   - Test the output shape
   - If your n8n version produces a different merge structure, adjust paths accordingly

9. **Add a Google Docs node to create the document**
   - Node type: `Google Docs`
   - Name it `Creating New File`
   - Connect it from `Aggregate`
   - Configure it to create a new document
   - Set title to:
     - `{{ $json.Title[0] }}`
   - Set the destination folder ID:
     - `191GOMY_IVzDJtDeHc5j65MOUWpC623LD`
   - Create or attach Google Docs OAuth2 credentials
   - Ensure the authenticated account has write access to that folder

10. **Add a second Google Docs node to insert content**
    - Node type: `Google Docs`
    - Name it `Updating Content in File`
    - Connect it from `Creating New File`
    - Set operation to `update`
    - Point the document reference to the created document
    - In the provided workflow, the field is:
      - `{{ $json.id }}`
    - Add one action:
      - Action: `insert`
      - Text:
        - `{{ $('Analyser').item.json.output }}`
    - Test that your version of the Google Docs node accepts the created document identifier in this field. If not, use the actual document URL returned by the create node.

11. **Connect the nodes in this exact order**
    - `Upload File or YouTube Link` → `Transcription using Supadata.ai`
    - `Transcription using Supadata.ai` → `Analyser`
    - `OpenRouter Chat Model` → `Analyser` via AI language model port
    - `Analyser` → `File Name Detector`
    - `OpenRouter Chat Model` → `File Name Detector` via AI language model port
    - `File Name Detector` → `Merge`
    - `Analyser` → `Merge`
    - `Merge` → `Aggregate`
    - `Aggregate` → `Creating New File`
    - `Creating New File` → `Updating Content in File`

12. **Add optional sticky notes for documentation**
    - Add notes for:
      - Transcribing
      - Analyzing
      - Storing
      - Overall workflow description and useful links

13. **Configure credentials**
    - **OpenRouter**
      - Required for the chat model node
      - Supply a valid OpenRouter API key
    - **Google Docs OAuth2**
      - Required for both Google Docs nodes
      - Grant document creation and editing permissions
    - **Supadata API key**
      - Add as HTTP header in the transcription node

14. **Test with a public YouTube URL**
    - Submit a valid video URL through the form
    - Check:
      - Transcript API response includes usable text
      - AI agent returns `output`
      - Title extractor returns a `Title`
      - Merge and Aggregate produce the expected shape
      - Google Doc is created and populated

15. **Recommended hardening improvements**
    - Make `youtube_url` required unless you also implement real file upload support
    - Fix the malformed Supadata URL
    - Add validation for URL format
    - Add an IF node after the trigger to reject empty input
    - Add error handling for missing transcript or failed title extraction
    - Consider truncating or chunking very large transcripts before sending to the model

### Sub-workflow setup

There are no sub-workflows in this JSON. No Execute Workflow node is present.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| A compact n8n workflow that accepts a YouTube link or uploaded video, pulls a transcript via Supadata.ai, runs a language-model-based video analysis agent to produce a structured report, extracts a title/metadata, then creates and updates a Google Doc with the analysis. It's designed to automate transcription → analysis → document creation for fast, repeatable video reviews. | Overall workflow description |
| Demo & Setup Video | https://drive.google.com/file/d/1g0rY7bbZrsFLnZjDfhXURR5WViywnYNw/view?usp=sharing |
| Course | https://www.udemy.com/course/n8n-automation-mastery-build-ai-powered-enterprise-ready/?referralCode=2EAE71591D3BEB80F2CC |

## Additional implementation notes

- The title says the workflow can analyze YouTube videos and auto-generate AI reports in Google Docs with DeepSeek. That is accurate for the current implementation.
- The description mentions accepting a YouTube link or uploaded video, but the actual trigger only includes a `youtube_url` text field. No file upload handling is implemented in this JSON.
- The Supadata request URL should be corrected before production use.
- The Google Docs update node inserts the analyzer output directly by cross-node reference rather than using the aggregated `Summary` field. This works only if item linkage remains stable. A more robust approach would use the immediately previous node’s data.