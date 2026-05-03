Transform Long Content into Bite-sized Learning Modules with GPT-4 & Google Docs

https://n8nworkflows.xyz/workflows/transform-long-content-into-bite-sized-learning-modules-with-gpt-4---google-docs-8265


# Transform Long Content into Bite-sized Learning Modules with GPT-4 & Google Docs

---

### 1. Workflow Overview

This workflow, titled **"Transform Long Content into Bite-sized Learning Modules with GPT-4 & Google Docs"**, is designed to convert lengthy textual content into structured micro-learning modules using AI, then format and distribute the results via Google Docs and Slack notifications.

**Target Use Cases:**  
- Organizations or educators wanting to automatically convert large documents into digestible e-learning pieces.  
- Content creators seeking AI-assisted content chunking and summarization for training or onboarding materials.  
- Teams requiring automated notifications and document generation for training delivery.

**Logical Blocks:**

- **1.1 Input Reception & Validation**: Receives raw content via webhook, extracts and validates it to ensure presence of the learning material.  
- **1.2 Content Chunking**: Splits the input content into logical, topic-based chunks using a sophisticated JavaScript chunking algorithm to optimize AI processing.  
- **1.3 AI Content Processing**: Sends each content chunk to GPT-4 AI to generate 2-4 micro-learning modules per chunk with structured fields (titles, explanations, quizzes).  
- **1.4 AI Response Formatting**: Parses and merges AI responses, deduplicates modules, enriches with metadata, and prepares multi-channel content formats (email, Slack, mobile).  
- **1.5 Output Splitting & Formatting**: Splits merged modules for individual processing and formats them into a professional Google Docs structure including course overview and detailed module sections.  
- **1.6 Google Docs Document Handling**: Creates a new Google Docs document and updates it with the formatted course content.  
- **1.7 Notifications & Responses**: Sends course overview notifications to Slack channels and responds to the original webhook caller with the final processed JSON or error messages.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Validation

- **Overview:**  
Receives incoming content via HTTP POST webhook, extracts the content field, and verifies its existence before processing.

- **Nodes Involved:**  
  - Content Input (Webhook)  
  - Extract Content (Set)  
  - Validate Content (If)  
  - Error Response (Respond to Webhook)  

- **Node Details:**  

  - **Content Input**  
    - *Type:* Webhook  
    - *Role:* Entry point receiving POST requests at path `/micro-learning-v2`.  
    - *Configuration:* HTTP Method POST; response mode set to return last node output.  
    - *Expressions:* None.  
    - *Connections:* Outputs to Extract Content.  
    - *Failure Modes:* Missing HTTP requests or malformed JSON bodies.  

  - **Extract Content**  
    - *Type:* Set  
    - *Role:* Extracts and standardizes the `content` field from the webhook JSON body.  
    - *Configuration:* Assigns `body.content` from incoming JSON's `body.content`.  
    - *Expressions:* `={{ $json.body.content }}`  
    - *Connections:* Outputs to Validate Content.  
    - *Failure Modes:* If content field is undefined or empty, downstream validation will catch.  

  - **Validate Content**  
    - *Type:* If  
    - *Role:* Checks if `body.content` is not empty.  
    - *Configuration:* Condition checks for non-empty string on `{{$json.body.content}}`.  
    - *Connections:* True to Chunker; False to Error Response.  
    - *Failure Modes:* Empty or missing content triggers error response.  

  - **Error Response**  
    - *Type:* Respond to Webhook  
    - *Role:* Returns JSON error message instructing user to provide content.  
    - *Configuration:* Fixed JSON with error and example content.  
    - *Connections:* Ends workflow on error branch.  

---

#### 1.2 Content Chunking

- **Overview:**  
Splits large documents into logical topic-based chunks suitable for AI processing, preserving section boundaries and creating chunks sized between 1000 and 4000 characters.

- **Nodes Involved:**  
  - Chunker (Code)  

- **Node Details:**  

  - **Chunker**  
    - *Type:* Code (JavaScript)  
    - *Role:* Implements a multi-strategy chunking algorithm: tries to split by headers using regex patterns; falls back to paragraph splits; groups sections into chunks respecting size limits.  
    - *Configuration:*  
      - Max chunk size: 4000 characters  
      - Min chunk size: 1000 characters  
      - Patterns searched: numbered sections, UPPERCASE headers, markdown headers, policy/procedure keywords, common section names  
    - *Variables/Expressions:* Uses input content from `body.content` of incoming JSON.  
    - *Outputs:* An array of chunks with metadata: chunk_id, content, topic_hint, sections_count, length, chunking_strategy.  
    - *Failure Modes:*  
      - Content without clear headers results in paragraph-based chunking.  
      - Extremely large content may produce many chunks, requiring attention to rate limits downstream.  
      - Potential for empty chunks if content is malformed.  
    - *Logging:* Logs chunking process and final chunk summaries for debugging.  

---

#### 1.3 AI Content Processing

- **Overview:**  
Sends each chunk to GPT-4 with a detailed prompt instructing the AI to generate 2-4 focused micro-learning modules per chunk, each including title, key concept, explanation, practical example, and quiz question.

- **Nodes Involved:**  
  - AI Content Analyzer (OpenAI)  

- **Node Details:**  

  - **AI Content Analyzer**  
    - *Type:* OpenAI Node (Chat Completion)  
    - *Role:* Calls GPT-4.1 model with system and user messages, providing chunk content and instructions for module creation.  
    - *Configuration:*  
      - Model: GPT-4.1  
      - Temperature: 0.5 for balanced creativity  
      - TopP: 1  
      - Frequency penalty: 0  
      - Prompt messages include system role explaining micro-learning expert persona and detailed module requirements.  
    - *Inputs:* Receives chunked content with chunk metadata.  
    - *Outputs:* AI response containing JSON array of modules.  
    - *Credentials:* OpenAI API credentials required.  
    - *Failure Modes:*  
      - API authentication failure  
      - Timeout or rate limit errors  
      - AI hallucination if prompt misunderstood (mitigated by prompt instructions)  
      - Invalid or unparsable AI output (handled in next block)  

---

#### 1.4 AI Response Formatting

- **Overview:**  
Processes AI responses by parsing JSON, consolidates modules from all chunks, removes duplicates based on title, enriches data with metadata and multi-channel content formats.

- **Nodes Involved:**  
  - Format_Output (Code)  

- **Node Details:**  

  - **Format_Output**  
    - *Type:* Code (JavaScript)  
    - *Role:*  
      - Iterates over AI responses, parses JSON content safely.  
      - Logs and collects parsing errors.  
      - Concatenates all modules into one list.  
      - Deduplicates modules by normalized title.  
      - Adds metadata: course ID, total modules, estimated time, difficulty level (based on module count).  
      - Formats multi-channel versions: email, Slack message, mobile text.  
      - Assigns sequence numbers and IDs to modules.  
    - *Expressions:* Uses `item.json.message.content` from AI output.  
    - *Outputs:* Single JSON object representing the entire course with all modules and metadata.  
    - *Failure Modes:*  
      - JSON parse errors from malformed AI output.  
      - Duplicate titles requiring deduplication logic.  
      - Missing module fields (title, content) default to placeholders.  

---

#### 1.5 Output Splitting & Formatting

- **Overview:**  
Splits the merged course modules array into individual modules for separate processing and formats them into a professional Google Docs friendly text document with course overview and module details.

- **Nodes Involved:**  
  - Split Out  
  - Google Formatter (Code)  
  - Edit Fields (Set)  
  - Merge  

- **Node Details:**  

  - **Split Out**  
    - *Type:* SplitOut  
    - *Role:* Splits the `modules` array into individual items for parallel processing and Slack notification.  
    - *Configuration:* Splits on field `modules`.  

  - **Google Formatter**  
    - *Type:* Code (JavaScript)  
    - *Role:*  
      - Accepts either a single course object with modules array or multiple individual module items.  
      - Sorts modules by sequence.  
      - Builds a formatted course text string with headers: course overview, each module’s content (title, key concept, content, example, quiz), and completion summary.  
      - Calculates estimated pages based on character count.  
      - Outputs the formatted text as `google_docs_content` along with metadata and debug info.  
    - *Failure Modes:*  
      - Missing module fields gracefully handled with defaults.  
      - Unexpected input format fallback logic.  

  - **Edit Fields**  
    - *Type:* Set  
    - *Role:* Sets course summary fields (course_id, total_modules, estimated_total_time, difficulty_level, created_at) from formatted output for Slack notifications and merging.  

  - **Merge**  
    - *Type:* Merge (Combine)  
    - *Role:* Combines formatted Google Docs content and course metadata for document creation and updates.  
    - *Configuration:* Combine by position of incoming items.  

---

#### 1.6 Google Docs Document Handling

- **Overview:**  
Creates a new Google Docs document with a title based on course ID, then updates the document with the formatted content generated previously.

- **Nodes Involved:**  
  - Create a document (Google Docs)  
  - Update a document (Google Docs)  

- **Node Details:**  

  - **Create a document**  
    - *Type:* Google Docs node  
    - *Role:* Creates a new Google Docs document in the default folder with title derived from `Format_Output.course_id`.  
    - *Configuration:* Uses dynamic expression for title.  
    - *Credentials:* Google Docs OAuth2 credentials required.  
    - *Execute Once:* True (runs only once per workflow execution).  
    - *Failure Modes:*  
      - Auth failure if credentials invalid.  
      - Folder permission errors.  

  - **Update a document**  
    - *Type:* Google Docs node  
    - *Role:* Updates the newly created Google Docs document with the fully formatted content.  
    - *Configuration:* Inserts the full text content into the document using the `insert` action.  
    - *Inputs:* Document URL from the Create node; content from `Merge` node.  
    - *Failure Modes:*  
      - Document not found or access denied errors.  
      - Insertion failure if content too large or invalid.  

---

#### 1.7 Notifications & Responses

- **Overview:**  
Sends Slack notifications about the course creation and individual modules, then responds to the original webhook call with success or error JSON.

- **Nodes Involved:**  
  - Slack Modules (Slack)  
  - Slack Course Details (Slack)  
  - Success Response (Respond to Webhook)  
  - No Operation nodes (placeholders)  

- **Node Details:**  

  - **Slack Modules**  
    - *Type:* Slack  
    - *Role:* Sends detailed notifications for each individual module to a configured Slack channel, including module title, ID, duration, and formatted Slack message content.  
    - *Configuration:* Uses Slack webhook with channel ID from credential cache.  
    - *Failure Modes:*  
      - Slack API rate limiting or auth failure.  
      - Missing channel or invalid format errors.  

  - **Slack Course Details**  
    - *Type:* Slack  
    - *Role:* Sends a summary notification about the newly created course including ID, total modules, and estimated time.  
    - *Execute Once:* True (single message per execution).  

  - **Success Response**  
    - *Type:* Respond to Webhook  
    - *Role:* Sends the final JSON response back to the webhook caller upon successful processing, containing all course and module data.  

  - **No Operation, do nothing / No Operation, do nothing1**  
    - *Type:* NoOp  
    - *Role:* Placeholder nodes for error path or end of Slack notifications without further processing.  

---

### 3. Summary Table

| Node Name            | Node Type              | Functional Role                              | Input Node(s)     | Output Node(s)            | Sticky Note                                                                                         |
|----------------------|------------------------|----------------------------------------------|-------------------|---------------------------|---------------------------------------------------------------------------------------------------|
| Content Input        | Webhook                | Receives raw content input                    | -                 | Extract Content           |                                                                                                   |
| Extract Content      | Set                    | Extracts and standardizes content field      | Content Input     | Validate Content          | **CONTENT PREPARATION** Extracts and standardizes input content for processing.                    |
| Validate Content     | If                     | Validates content presence                     | Extract Content   | Chunker / Error Response  |                                                                                                   |
| Error Response       | Respond to Webhook     | Returns error JSON if content missing         | Validate Content  | No Operation, do nothing1 |                                                                                                   |
| Chunker              | Code                   | Splits content into topic-based chunks        | Validate Content  | AI Content Analyzer       | **SMART CONTENT CHUNKING** Breaks large docs into logical chunks preserving section boundaries.    |
| AI Content Analyzer  | OpenAI                 | Generates micro-learning modules from chunks  | Chunker           | Format_Output             | **AI CONTENT PROCESSING** Converts chunks into structured learning modules using GPT-4.           |
| Format_Output        | Code                   | Parses, deduplicates, and enriches AI modules | AI Content Analyzer | Split Out, Edit Fields   | **MODULE FORMATTER** Processes AI responses into structured module data with multi-channel formats.|
| Split Out            | SplitOut               | Splits modules array into individual modules  | Format_Output     | Google Formatter, Slack Modules, Slack Course Details |                                                                                                   |
| Google Formatter     | Code                   | Formats modules into professional Google Docs | Split Out         | Merge                     | **GOOGLE DOCS CONVERTER** Creates structured Google Docs content from modules and course metadata.|
| Edit Fields          | Set                    | Sets course summary fields for notifications  | Format_Output     | Merge                     |                                                                                                   |
| Merge                | Merge (Combine)        | Combines formatted document content and metadata | Google Formatter, Edit Fields | Create a document      |                                                                                                   |
| Create a document    | Google Docs             | Creates new Google Docs document               | Merge             | Update a document         | **CREATE GOOGLE DOC** Creates Google Doc to store course content.                                 |
| Update a document    | Google Docs             | Inserts formatted content into Google Doc     | Create a document | Success Response          |                                                                                                   |
| Slack Modules        | Slack                   | Sends individual module notifications          | Split Out         | Success Response          | **TEAM NOTIFICATIONS** Sends module details to Slack channel; optional step.                       |
| Slack Course Details | Slack                   | Sends overall course notification to Slack    | Split Out         | No Operation, do nothing  |                                                                                                   |
| Success Response     | Respond to Webhook      | Returns final JSON response on success         | Update a document, Slack Modules | -                  |                                                                                                   |
| No Operation, do nothing | NoOp                | Placeholder for error or end paths             | Error Response, Slack Course Details | -                  |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node ("Content Input")**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `micro-learning-v2`  
   - Response Mode: `lastNode`  
   - No credentials needed.

2. **Create Set Node ("Extract Content")**  
   - Assign `body.content` = `={{ $json.body.content }}` from webhook input.  
   - Connect from "Content Input".

3. **Create If Node ("Validate Content")**  
   - Condition: `{{$json.body.content}}` is not empty string.  
   - Connect True to next step; False to error response.

4. **Create Respond to Webhook Node ("Error Response")**  
   - Response Body: JSON with error message and example content field.  
   - Connect False output of validation here.

5. **Create Function/Code Node ("Chunker")**  
   - Use provided JavaScript code implementing topic-based chunking.  
   - Input: `body.content`  
   - Outputs array of chunks with metadata.  
   - Connect True output from validation.

6. **Create OpenAI Node ("AI Content Analyzer")**  
   - Model: GPT-4.1  
   - Temperature: 0.5, TopP: 1, Frequency Penalty: 0  
   - System prompt: Micro-learning expert instructions including chunk info.  
   - User prompt: Break down chunk content into 2-4 micro-learning modules as JSON.  
   - Credentials: OpenAI API credentials configured.  
   - Connect from "Chunker".

7. **Create Code Node ("Format_Output")**  
   - Implement code to parse AI response JSON, deduplicate modules, enrich metadata, and create multi-channel formats.  
   - Connect from "AI Content Analyzer".

8. **Create SplitOut Node ("Split Out")**  
   - Split field: `modules` (array of modules).  
   - Connect from "Format_Output".

9. **Create Code Node ("Google Formatter")**  
   - Format modules and course metadata into a structured Google Docs text document with overview and modules.  
   - Connect from "Split Out".

10. **Create Set Node ("Edit Fields")**  
    - Assign course summary fields (`course_id`, `total_modules`, etc.) from `Format_Output`.  
    - Connect from "Format_Output".

11. **Create Merge Node ("Merge")**  
    - Mode: Combine  
    - Combine incoming items by position.  
    - Connect inputs: from "Google Formatter" and "Edit Fields".

12. **Create Google Docs Node ("Create a document")**  
    - Operation: Create  
    - Title: `={{ $('Format_Output').item.json.course_id }}` (dynamic)  
    - Folder: Default or specify folder ID  
    - Credentials: Google Docs OAuth2  
    - Connect from "Merge".

13. **Create Google Docs Node ("Update a document")**  
    - Operation: Update  
    - Document URL: from "Create a document" output  
    - Action: Insert  
    - Text: from `Merge` node's `google_docs_content` field  
    - Connect from "Create a document".

14. **Create Slack Node ("Slack Modules")**  
    - Send message to configured Slack channel with module details using expressions from module data.  
    - Credentials: Slack API credentials  
    - Connect from "Split Out".

15. **Create Slack Node ("Slack Course Details")**  
    - Send a single message summarizing course creation event, including course ID, total modules, and estimated time.  
    - Credentials: Slack API credentials  
    - Execute once per run  
    - Connect from "Split Out".

16. **Create Respond to Webhook Node ("Success Response")**  
    - Respond with JSON, body: `={{ $json }}` (final course data).  
    - Connect from "Update a document" and "Slack Modules".

17. **Create NoOp Nodes ("No Operation, do nothing" and "No Operation, do nothing1")**  
    - Placeholders for error and notification completion paths.  
    - Connect error paths accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The chunking algorithm uses regex patterns tuned for business documents, with fallback to paragraph splits for unstructured content. | See "Chunker" node code comments for detailed patterns and chunking strategy.                    |
| AI prompt explicitly instructs GPT-4 to avoid hallucination and only use provided text to generate modules. | Important for content accuracy and avoiding extraneous information in modules.                   |
| Module formatter supports multi-channel content: email, Slack, and mobile formatted strings for flexible distribution. | Useful for reusing modules across different communication platforms.                             |
| Slack notification nodes are optional and can be disabled if team alerts are not desired.     | To disable, remove or disable both "Slack Modules" and "Slack Course Details" nodes.             |
| Google Docs creation requires valid OAuth2 credentials and appropriate folder permissions.    | Update credentials and folder ID in "Create a document" node settings as needed.                 |
| The "Format_Output" node includes a note suggesting to regenerate code if using alternative AI models like Claude. | Allows easy adaptation of the workflow to other AI providers.                                   |
| Execution settings save all execution data and progress for debugging and audit purposes.     | Useful for troubleshooting and workflow improvement.                                            |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---