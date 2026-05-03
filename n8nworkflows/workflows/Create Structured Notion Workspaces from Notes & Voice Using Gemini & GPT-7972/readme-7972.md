Create Structured Notion Workspaces from Notes & Voice Using Gemini & GPT

https://n8nworkflows.xyz/workflows/create-structured-notion-workspaces-from-notes---voice-using-gemini---gpt-7972


# Create Structured Notion Workspaces from Notes & Voice Using Gemini & GPT

### 1. Workflow Overview

This workflow automates the creation of well-structured Notion workspaces from various input notes and voice recordings. It is designed to organize content intelligently by leveraging AI models from Gemini, OpenAI GPT, and Google Vertex, integrating with Google Drive for input files and Notion for output databases and pages.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Preprocessing:** Triggered by file uploads in Google Drive, downloads files, and determines whether the input is an image or audio recording.
- **1.2 Content Transcription and Analysis:** Uses Gemini AI to transcribe audio or analyze images, then applies OpenAI GPT and Google Vertex models to analyze and extract structured content.
- **1.3 Structure and Database Generation:** Generates structured database schemas and content mappings for Notion using chain LLMs and parsers.
- **1.4 Notion Workspace Creation and Population:** Creates Notion databases and pages, populating them with analyzed and structured content.
- **1.5 Iterative Processing and Completion:** Processes multiple items in batches, handles responses, and marks workflow completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preprocessing

**Overview:**  
This block detects new files in Google Drive, downloads them, and routes processing based on file type (image or audio).

**Nodes Involved:**  
- Google Drive Trigger  
- Download file  
- If  
- Analyze image  
- Transcribe a recording  
- Edit Fields  
- Generate Request ID

**Node Details:**

- **Google Drive Trigger**  
  - *Type:* Trigger  
  - *Role:* Watches a specific Google Drive folder for new files to start the workflow.  
  - *Config:* Uses Google Drive OAuth2 credentials, configured to monitor relevant folders.  
  - *Inputs:* None (trigger)  
  - *Outputs:* File metadata  
  - *Edge Cases:* Permissions errors, no new files, or unsupported file types.

- **Download file**  
  - *Type:* Google Drive node  
  - *Role:* Downloads the detected file for processing.  
  - *Config:* Uses file ID from trigger node.  
  - *Inputs:* File metadata  
  - *Outputs:* Binary file content  
  - *Failure:* File not found, download timeout.

- **If**  
  - *Type:* Conditional node  
  - *Role:* Checks file type to branch processing: image or audio.  
  - *Config:* Condition on file MIME type or extension.  
  - *Inputs:* File metadata and content  
  - *Outputs:* Branch 1 (image), Branch 2 (audio)  
  - *Edge Cases:* Unsupported file types.

- **Analyze image**  
  - *Type:* Gemini Google AI node  
  - *Role:* Analyzes image content using Gemini AI.  
  - *Config:* Configured with Gemini credentials and analysis parameters.  
  - *Inputs:* Image binary data  
  - *Outputs:* Textual or structured analysis results  
  - *Failures:* API quota, invalid image data.

- **Transcribe a recording**  
  - *Type:* Gemini Google AI node  
  - *Role:* Transcribes audio recording to text.  
  - *Config:* Gemini credentials, audio transcription settings.  
  - *Inputs:* Audio binary data  
  - *Outputs:* Transcribed text  
  - *Failures:* Audio format issues, transcription errors.

- **Edit Fields**  
  - *Type:* Set node  
  - *Role:* Modifies or adds metadata fields for downstream use.  
  - *Config:* Sets or edits key-value pairs such as filenames, timestamps, or IDs.  
  - *Inputs:* Transcription or analysis results  
  - *Outputs:* Enhanced data object  
  - *Edge Cases:* Missing fields.

- **Generate Request ID**  
  - *Type:* Crypto node  
  - *Role:* Generates a unique identifier for the request, ensuring traceability.  
  - *Config:* Uses random or hash function.  
  - *Inputs:* None or previous node data  
  - *Outputs:* Unique ID string  
  - *Edge Cases:* ID collisions unlikely but possible.

---

#### 2.2 Content Transcription and Analysis

**Overview:**  
This block processes the textual content extracted from files by applying AI models for content understanding and categorization.

**Nodes Involved:**  
- Content Analyzer  
- Content Analyzer LLM  
- Content Analysis Parser  
- Structure Generator LLM  
- Structure Parser  
- Database Structure Generator  
- Data Formatter2

**Node Details:**

- **Content Analyzer**  
  - *Type:* Langchain AI Agent  
  - *Role:* Orchestrates AI analysis of content using LLMs.  
  - *Config:* Uses AI models for semantic understanding.  
  - *Inputs:* Transcribed or analyzed content  
  - *Outputs:* Semantic analysis results  
  - *Failures:* AI rate limits, unhandled content types.

- **Content Analyzer LLM**  
  - *Type:* OpenAI GPT Chat model  
  - *Role:* Provides deep content analysis leveraging chat completion.  
  - *Config:* Model version 1.2, tuned for content understanding.  
  - *Inputs:* Text content  
  - *Outputs:* AI-generated analysis text  
  - *Edge Cases:* Model timeout or API errors.

- **Content Analysis Parser**  
  - *Type:* Structured output parser  
  - *Role:* Parses AI output into structured JSON data for subsequent nodes.  
  - *Config:* Parsing rules expect specific JSON formats.  
  - *Inputs:* AI text output  
  - *Outputs:* Structured data objects  
  - *Failures:* Parsing errors if AI output is malformed.

- **Structure Generator LLM**  
  - *Type:* OpenAI GPT Chat model  
  - *Role:* Generates database and content structure proposals.  
  - *Config:* Chat model tuned for schema generation.  
  - *Inputs:* Parsed content analysis  
  - *Outputs:* Structured schema text  
  - *Edge Cases:* Ambiguous or conflicting schema suggestions.

- **Structure Parser**  
  - *Type:* Structured output parser  
  - *Role:* Converts schema text into structured data for database creation.  
  - *Config:* Parsing rules for expected schema format.  
  - *Inputs:* Schema text  
  - *Outputs:* Parsed database schema objects  
  - *Failures:* Parsing errors.

- **Database Structure Generator**  
  - *Type:* Chain LLM  
  - *Role:* Coordinates schema generation and formatting tasks.  
  - *Config:* LLM chain settings for multi-step generation.  
  - *Inputs:* Parsed schema data  
  - *Outputs:* Formatted database schema  
  - *Failures:* Chain execution errors.

- **Data Formatter2**  
  - *Type:* Set node  
  - *Role:* Prepares data in the correct format for Notion API calls.  
  - *Config:* Sets necessary fields and adjusts data structures.  
  - *Inputs:* Database schema object  
  - *Outputs:* Formatted data object for Notion  
  - *Edge Cases:* Missing required fields.

---

#### 2.3 Structure and Database Generation

**Overview:**  
This block uses the prepared schema to create Notion databases and prepares the data for populating those databases.

**Nodes Involved:**  
- Prepare Notion Data1  
- Create Notion Database4  
- AI Agent  
- LLM Smart Mapping

**Node Details:**

- **Prepare Notion Data1**  
  - *Type:* Code node  
  - *Role:* Custom JavaScript to finalize data structure for Notion API.  
  - *Config:* Formats and enriches data objects as per Notion API requirements.  
  - *Inputs:* Formatted schema data  
  - *Outputs:* Finalized data ready for database creation  
  - *Edge Cases:* Code errors due to unexpected data.

- **Create Notion Database4**  
  - *Type:* HTTP Request node  
  - *Role:* Sends HTTP request to Notion API to create the database.  
  - *Config:* Uses Notion API credentials and endpoint for database creation.  
  - *Inputs:* Database schema data  
  - *Outputs:* API response with database ID  
  - *Failures:* Auth errors, rate limits, malformed requests.

- **AI Agent**  
  - *Type:* Langchain AI Agent  
  - *Role:* Coordinates AI tasks relevant to database and content structuring.  
  - *Config:* Connected to smart mapping and other AI nodes.  
  - *Inputs:* Data from Notion database creation and content analysis  
  - *Outputs:* AI-driven instructions for content mapping  
  - *Failures:* AI unavailability.

- **LLM Smart Mapping**  
  - *Type:* OpenAI GPT Chat model (or alternative)  
  - *Role:* Maps content intelligently to database schema fields.  
  - *Config:* AI parameters tuned for mapping accuracy.  
  - *Inputs:* Content and schema  
  - *Outputs:* Mapping instructions for content placement  
  - *Edge Cases:* Incorrect mappings or missing fields.

---

#### 2.4 Notion Workspace Creation and Population

**Overview:**  
This block handles creating rows/pages in the Notion database and populating them with structured data.

**Nodes Involved:**  
- Report Page Generator  
- Structured Output Parser4  
- Create Row  
- Get Existing Row  
- Set Done  
- Create Sample Pages  
- Loop Over Items  
- Split Out  
- Parse LLM Response

**Node Details:**

- **Report Page Generator**  
  - *Type:* Chain LLM  
  - *Role:* Generates content for Notion pages based on AI analysis.  
  - *Config:* Uses structured output parser to format output.  
  - *Inputs:* AI-generated content and request IDs  
  - *Outputs:* Content ready for page creation  
  - *Failures:* AI errors.

- **Structured Output Parser4**  
  - *Type:* Structured output parser  
  - *Role:* Parses the LLM output to structured data for page creation.  
  - *Inputs:* LLM output text  
  - *Outputs:* Structured page content  
  - *Failures:* Parsing errors.

- **Create Row**  
  - *Type:* Notion node  
  - *Role:* Creates a new row in the Notion database for the content.  
  - *Inputs:* Structured page content  
  - *Outputs:* Confirmation of row creation  
  - *Failures:* API errors, data validation failures.

- **Get Existing Row**  
  - *Type:* Notion node  
  - *Role:* Checks for existing rows to avoid duplicates.  
  - *Inputs:* Query by content or ID  
  - *Outputs:* Existing row info or empty  
  - *Failures:* Query errors.

- **Set Done**  
  - *Type:* Notion node  
  - *Role:* Marks the item as processed/completed in Notion or internal status.  
  - *Inputs:* Processed item data  
  - *Outputs:* Updated status confirmation  
  - *Failures:* Write errors.

- **Create Sample Pages**  
  - *Type:* HTTP Request node  
  - *Role:* Optionally creates sample pages in Notion for demonstration or testing.  
  - *Inputs:* Prepared data batches  
  - *Outputs:* API response from Notion  
  - *Failures:* API rate limits.

- **Loop Over Items**  
  - *Type:* SplitInBatches node  
  - *Role:* Processes multiple items sequentially or in defined batch sizes.  
  - *Inputs:* Array of items  
  - *Outputs:* Single item per iteration  
  - *Edge Cases:* Large batch size causing timeouts.

- **Split Out**  
  - *Type:* SplitOut node  
  - *Role:* Splits batch processing output to individual items for looping.  
  - *Inputs:* Batch data  
  - *Outputs:* Individual item data  
  - *Edge Cases:* Empty arrays.

- **Parse LLM Response**  
  - *Type:* Code node  
  - *Role:* Custom parsing of AI agent responses to usable data formats.  
  - *Inputs:* AI agent raw outputs  
  - *Outputs:* Parsed structured data for further processing  
  - *Failures:* Parsing or runtime errors.

---

### 3. Summary Table

| Node Name               | Node Type                                  | Functional Role                              | Input Node(s)                   | Output Node(s)              | Sticky Note                                         |
|-------------------------|--------------------------------------------|----------------------------------------------|--------------------------------|-----------------------------|-----------------------------------------------------|
| Google Drive Trigger     | Google Drive Trigger                       | Detect new files in Drive                     | -                              | Download file               |                                                     |
| Download file           | Google Drive                               | Download file content                         | Google Drive Trigger            | If                          |                                                     |
| If                      | Conditional                               | Branch processing by file type               | Download file                  | Analyze image, Transcribe a recording |                                                     |
| Analyze image            | Gemini Google AI                          | Analyze image content                         | If (image branch)               | Edit Fields                 |                                                     |
| Transcribe a recording   | Gemini Google AI                          | Transcribe audio recordings                   | If (audio branch)               | Edit Fields                 |                                                     |
| Edit Fields             | Set                                       | Add/edit metadata fields                       | Analyze image, Transcribe a recording | Generate Request ID        |                                                     |
| Generate Request ID      | Crypto                                    | Unique ID generation                          | Edit Fields                    | Report Page Generator       |                                                     |
| Report Page Generator    | Chain LLM                                 | Generate content for Notion pages             | Generate Request ID            | Create Row                  |                                                     |
| Structured Output Parser4| Structured Output Parser                   | Parse LLM output for page content             | Report Page Generator          | Create Row                  |                                                     |
| Create Row              | Notion                                    | Create Notion database row                     | Report Page Generator          | Get Existing Row            |                                                     |
| Get Existing Row        | Notion                                    | Check for existing row to avoid duplication   | Create Row                    | Content Analyzer            |                                                     |
| Content Analyzer        | Langchain AI Agent                         | Content semantic analysis                      | Get Existing Row              | Database Structure Generator |                                                     |
| Content Analyzer LLM    | OpenAI Chat Model                         | AI content analysis                            | Content Analyzer              | Content Analyzer            |                                                     |
| Content Analysis Parser | Structured Output Parser                   | Parse content analysis output                  | Content Analyzer LLM          | Content Analyzer            |                                                     |
| Structure Generator LLM | OpenAI Chat Model                         | Generate database schema proposals             | Content Analysis Parser       | Database Structure Generator |                                                     |
| Structure Parser        | Structured Output Parser                   | Parse schema generation output                 | Structure Generator LLM       | Database Structure Generator |                                                     |
| Database Structure Generator | Chain LLM                             | Coordinate schema generation and formatting    | Structure Parser              | Data Formatter2             |                                                     |
| Data Formatter2         | Set                                       | Format data for Notion API                      | Database Structure Generator  | Prepare Notion Data1        |                                                     |
| Prepare Notion Data1    | Code                                      | Finalize data structure for Notion API         | Data Formatter2              | Create Notion Database4     |                                                     |
| Create Notion Database4 | HTTP Request                              | Create new Notion database                      | Prepare Notion Data1          | AI Agent                   |                                                     |
| AI Agent                | Langchain AI Agent                         | AI orchestration for content mapping           | Create Notion Database4       | Parse LLM Response, LLM Smart Mapping |                                                     |
| LLM Smart Mapping       | OpenAI Chat Model                         | Map content intelligently to database fields   | AI Agent                     | AI Agent                   |                                                     |
| Parse LLM Response      | Code                                      | Parse AI agent response                         | AI Agent                     | Split Out                  |                                                     |
| Split Out               | SplitOut                                  | Split batch output into items                   | Parse LLM Response            | Loop Over Items            |                                                     |
| Loop Over Items         | SplitInBatches                            | Process items in batches                         | Split Out                    | Set Done, Create Sample Pages |                                                     |
| Set Done                | Notion                                    | Mark item as processed                           | Loop Over Items              | -                          |                                                     |
| Create Sample Pages     | HTTP Request                              | Create sample pages in Notion                    | Loop Over Items              | Loop Over Items            |                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: Google Drive Trigger**  
   - Set credentials for Google Drive OAuth2.  
   - Configure to monitor the folder where notes/recordings are saved.

2. **Add Download File Node**  
   - Connect from Google Drive Trigger.  
   - Use file ID from trigger.  
   - Set to download file content.

3. **Add If Node**  
   - Connect from Download File.  
   - Condition: check MIME type or file extension to distinguish image vs audio.

4. **Branch 1 (Image Processing): Add Analyze Image Node**  
   - Connect from If true (image branch).  
   - Configure Gemini Google AI credentials and image analysis parameters.

5. **Branch 2 (Audio Processing): Add Transcribe a Recording Node**  
   - Connect from If false (audio branch).  
   - Configure Gemini Google AI for audio transcription.

6. **Add Edit Fields Node**  
   - Connect both Analyze Image and Transcribe a Recording to Edit Fields (merge branches).  
   - Configure to set/add relevant metadata fields (e.g., filename, source).

7. **Add Generate Request ID Node**  
   - Connect from Edit Fields.  
   - Use Crypto node to generate a unique request ID.

8. **Add Report Page Generator Node**  
   - Connect from Generate Request ID.  
   - Configure as a Chain LLM node using OpenAI GPT (Chat Model4).  
   - Set prompts/templates for generating Notion page content.

9. **Add Structured Output Parser Node**  
   - Connect from Report Page Generator AI output.  
   - Configure to parse output into structured JSON.

10. **Add Create Row Node**  
    - Connect from Structured Output Parser.  
    - Set up Notion credentials and configure to create a row in the target database.

11. **Add Get Existing Row Node**  
    - Connect from Create Row.  
    - Configure to query Notion database to check for duplicates by content or ID.

12. **Add Content Analyzer Node**  
    - Connect from Get Existing Row.  
    - Configure as Langchain AI Agent node to analyze content semantically.

13. **Add Content Analyzer LLM Node**  
    - Connect from Content Analyzer.  
    - Configure OpenAI GPT model to perform deeper content analysis.

14. **Add Content Analysis Parser Node**  
    - Connect from Content Analyzer LLM.  
    - Configure structured output parser for AI content analysis.

15. **Add Structure Generator LLM Node**  
    - Connect from Content Analysis Parser.  
    - Configure OpenAI GPT model for database schema generation.

16. **Add Structure Parser Node**  
    - Connect from Structure Generator LLM AI output.  
    - Configure to parse database schema into structured format.

17. **Add Database Structure Generator Node**  
    - Connect from Structure Parser.  
    - Configure as Chain LLM node to coordinate schema formatting.

18. **Add Data Formatter2 Node**  
    - Connect from Database Structure Generator.  
    - Configure Set node to prepare data for Notion API.

19. **Add Prepare Notion Data1 Node**  
    - Connect from Data Formatter2.  
    - Configure JavaScript code node to finalize Notion data structure.

20. **Add Create Notion Database4 Node**  
    - Connect from Prepare Notion Data1.  
    - Set HTTP request node with Notion API credentials.  
    - Endpoint: Create database.

21. **Add AI Agent Node**  
    - Connect from Create Notion Database4.  
    - Configure Langchain AI Agent for orchestrating further AI tasks.

22. **Add LLM Smart Mapping Node**  
    - Connect from AI Agent (ai_languageModel input).  
    - Configure OpenAI GPT for intelligent content-to-schema mapping.

23. **Connect AI Agent main output to Parse LLM Response Node**  
    - Add code node to parse AI response into usable data.

24. **Add Split Out Node**  
    - Connect from Parse LLM Response node.  
    - Configure to split batch data into individual items.

25. **Add Loop Over Items Node**  
    - Connect from Split Out node.  
    - Configure SplitInBatches with batch size as appropriate.

26. **Add Set Done Node**  
    - Connect from Loop Over Items.  
    - Configure Notion node to mark item processed or update status.

27. **Add Create Sample Pages Node**  
    - Connect from Loop Over Items.  
    - Use HTTP Request node to create sample Notion pages for testing/demo.

28. **Final connections and error handling**  
    - Ensure all nodes have proper error workflows or retries configured.

29. **Credentials Required:**  
    - Google Drive OAuth2  
    - Notion API with database permissions  
    - OpenAI API key for GPT models  
    - Google Gemini credentials for image/audio AI  
    - Any other API tokens as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow integrates multiple AI models (Gemini, OpenAI GPT, Google Vertex) for robust analysis. | Demonstrates hybrid AI orchestration for document processing and workspace generation.          |
| Uses Notion API v2.2 with HTTP requests and native Notion nodes for database and page management.   | Ensures compatibility with latest Notion API features.                                         |
| Google Drive Trigger node starts workflow on file uploads, enabling real-time automation.           | Useful for automating note capture from mobile or desktop clients synced to Drive.              |
| For detailed AI prompt templates and chain configurations, refer to Langchain documentation.       | https://js.langchain.com/docs/                                                                    |
| Workflow designed for batch processing with SplitInBatches node for scalability.                     | Helps avoid rate limits and timeout errors during processing.                                   |

---

This document provides a comprehensive reference to understand, reproduce, and maintain the “Create Structured Notion Workspaces from Notes & Voice Using Gemini & GPT” workflow, ensuring effective integration and AI-powered content management.