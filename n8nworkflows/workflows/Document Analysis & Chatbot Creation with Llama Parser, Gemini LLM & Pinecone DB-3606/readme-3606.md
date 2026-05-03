Document Analysis & Chatbot Creation with Llama Parser, Gemini LLM & Pinecone DB

https://n8nworkflows.xyz/workflows/document-analysis---chatbot-creation-with-llama-parser--gemini-llm---pinecone-db-3606


# Document Analysis & Chatbot Creation with Llama Parser, Gemini LLM & Pinecone DB

### 1. Workflow Overview

This workflow automates document content analysis and chatbot creation by integrating file upload, content parsing, translation, analysis, vector storage, and user notification. It targets users who want to upload documents for automated multilingual analysis and receive both detailed insights and an interactive chatbot link via email.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and File Handling:** Receives user files and email via an N8N form, splits multiple uploaded files for processing.
- **1.2 Document Parsing:** Uploads files to Llama Cloud API for parsing, polls for parsing completion, and retrieves parsed content in markdown format.
- **1.3 Translation:** Uses Google Gemini Translator Agent to detect and translate non-English content into English.
- **1.4 Content Analysis:** Sends translated content to Google Gemini Analyzer Agent for deep analysis and formatting.
- **1.5 Content Formatting and File Conversion:** Converts analyzed markdown to HTML, then to formatted text, and finally to a text file.
- **1.6 Vector Storage:** Stores parsed and analyzed content embeddings into Pinecone vector database for chatbot knowledge base.
- **1.7 User Notification:** Sends an email via Gmail API to the user with the analysis results and chatbot link.
- **1.8 Chatbot Interaction:** Provides a public webhook for chatbot queries, retrieving answers from Pinecone-stored knowledge using Google Gemini LLM.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Handling

- **Overview:** Captures user inputs (files and email) from a public form, splits multiple uploaded files into individual items for processing.
- **Nodes Involved:**  
  - On form submission4  
  - split the binary item  
  - Loop Over Items1

- **Node Details:**

  - **On form submission4**  
    - Type: Form Trigger  
    - Role: Entry point; triggers workflow on form submission with multiple file uploads and user email input.  
    - Config: Form fields include two file inputs (file1 required, file2 optional) and a required email field.  
    - Inputs: HTTP webhook from form submission  
    - Outputs: JSON with files and email  
    - Edge Cases: Missing required file or email; malformed uploads.

  - **split the binary item**  
    - Type: Code  
    - Role: Splits each binary file from form data into separate workflow items for individual processing.  
    - Key Code: Iterates over all binary fields, creates one item per file binary data.  
    - Inputs: Items from form submission  
    - Outputs: Multiple items each containing one binary file  
    - Edge Cases: No binary data present; multiple files in one field.

  - **Loop Over Items1**  
    - Type: SplitInBatches  
    - Role: Processes each file item in batches (default batch size) to handle parsing sequentially or in manageable chunks.  
    - Inputs: Items from split the binary item  
    - Outputs: Each batch item forwarded to parsing  
    - Edge Cases: Large number of files causing long processing times.

#### 2.2 Document Parsing

- **Overview:** Uploads each file to Llama Cloud API for parsing, polls for job completion, and retrieves parsed markdown content.
- **Nodes Involved:**  
  - Parsing the document  
  - Check the parsing status  
  - If2  
  - Provide the markdown  
  - Aggregate1

- **Node Details:**

  - **Parsing the document**  
    - Type: HTTP Request  
    - Role: Uploads binary file to Llama Cloud parsing API endpoint.  
    - Config: POST multipart/form-data with binary file under "file" parameter; Authorization header with Bearer token.  
    - Inputs: Binary file from Loop Over Items1  
    - Outputs: JSON response containing job ID and status  
    - Edge Cases: API auth failure, upload errors, file format unsupported.

  - **Check the parsing status**  
    - Type: HTTP Request  
    - Role: Polls Llama Cloud API for parsing job status using job ID.  
    - Config: GET request to job status endpoint with Authorization header.  
    - Inputs: Job ID from Parsing the document  
    - Outputs: JSON with status field (e.g., SUCCESS)  
    - Edge Cases: Timeout if job takes too long, API errors.

  - **If2**  
    - Type: If  
    - Role: Branches workflow based on parsing job status.  
    - Config: Checks if status equals "SUCCESS".  
    - Inputs: Status from Check the parsing status  
    - Outputs:  
      - True branch: Provide the markdown  
      - False branch: Check the parsing status (retry)  
    - Edge Cases: Infinite retry loops if job never succeeds.

  - **Provide the markdown**  
    - Type: HTTP Request  
    - Role: Retrieves parsed document content in markdown format from Llama Cloud API.  
    - Config: GET request to job result markdown endpoint with Authorization header.  
    - Inputs: Job ID from If2 (success branch)  
    - Outputs: JSON containing markdown content  
    - Edge Cases: API errors, missing result.

  - **Aggregate1**  
    - Type: Aggregate  
    - Role: Aggregates multiple markdown outputs into a single collection for further processing.  
    - Config: Aggregates field "markdown" from all items.  
    - Inputs: Markdown content from Provide the markdown (multiple items)  
    - Outputs: Single aggregated item with combined markdown  
    - Edge Cases: Empty aggregation if no markdown received.

#### 2.3 Translation

- **Overview:** Detects language of aggregated markdown content; if not English, translates it using Google Gemini Translator Agent.
- **Nodes Involved:**  
  - Translator Agent  
  - Google Gemini Chat Model5

- **Node Details:**

  - **Translator Agent**  
    - Type: Langchain Agent  
    - Role: Detects language and translates non-English content to English, appending untranslated English parts.  
    - Config: System message instructs to detect language and translate if needed, preserving English parts.  
    - Inputs: Aggregated markdown from Aggregate1  
    - Outputs: Translated markdown content  
    - Edge Cases: Incorrect language detection, partial translations.

  - **Google Gemini Chat Model5**  
    - Type: Google Gemini LLM  
    - Role: Provides language model backend for Translator Agent.  
    - Config: Uses "models/gemini-1.5-flash" model.  
    - Inputs: Prompt from Translator Agent  
    - Outputs: Translation response  
    - Edge Cases: API rate limits, model errors.

#### 2.4 Content Analysis

- **Overview:** Analyzes translated content deeply to extract insights and improve readability using Google Gemini Analyzer Agent.
- **Nodes Involved:**  
  - Analyzer Agent  
  - Google Gemini Chat Model6  
  - Markdown  
  - AI Agent  
  - Information Extractor  
  - Convert to File4  
  - Pinecone Vector Store

- **Node Details:**

  - **Analyzer Agent**  
    - Type: Langchain Agent  
    - Role: Performs comprehensive analysis with structured formatting and insight extraction.  
    - Config: System message instructs detailed analysis, clarity, duplicate handling, and actionable resolutions.  
    - Inputs: Translated markdown from Translator Agent  
    - Outputs: Analyzed markdown output  
    - Edge Cases: Over/under analysis, formatting issues.

  - **Google Gemini Chat Model6**  
    - Type: Google Gemini LLM  
    - Role: Backend LLM for Analyzer Agent.  
    - Config: Uses "models/gemini-1.5-flash".  
    - Inputs: Prompt from Analyzer Agent  
    - Outputs: Analyzed content  
    - Edge Cases: API errors, timeouts.

  - **Markdown**  
    - Type: Markdown Node  
    - Role: Converts analyzed markdown to HTML for formatting.  
    - Config: Mode set to markdownToHtml, source is analyzed output.  
    - Inputs: Analyzer Agent output  
    - Outputs: HTML content  
    - Edge Cases: Markdown syntax errors.

  - **AI Agent**  
    - Type: Langchain Agent  
    - Role: Enhances readability, applies structured formatting and bold text, outputs text file compatible content.  
    - Config: System message with detailed formatting rules, expects text file output.  
    - Inputs: HTML content from Markdown node  
    - Outputs: Formatted text content  
    - Edge Cases: Formatting failures, unexpected HTML input.

  - **Information Extractor**  
    - Type: Langchain Information Extractor  
    - Role: Extracts specific attributes like "Project Overview" and "System and prerequisites" from analyzed content.  
    - Config: System prompt to extract relevant info only, attributes defined explicitly.  
    - Inputs: Output from AI Agent  
    - Outputs: Extracted structured information  
    - Edge Cases: Missing attributes, extraction inaccuracies.

  - **Convert to File4**  
    - Type: Convert To File  
    - Role: Converts analyzed text content into a text file for storage and email attachment.  
    - Config: Operation "toText", source property "output".  
    - Inputs: Output from Analyzer Agent and AI Agent chain  
    - Outputs: Text file binary data  
    - Edge Cases: File conversion errors.

  - **Pinecone Vector Store**  
    - Type: Vector Store Pinecone  
    - Role: Inserts analyzed and translated content embeddings into Pinecone index "samuraichamploo" for chatbot knowledge base.  
    - Config: Mode "insert", uses Mistral Cloud embeddings.  
    - Inputs: Text file from Convert to File4, embeddings from Embeddings Mistral Cloud  
    - Outputs: Confirmation of insertion  
    - Edge Cases: Pinecone API errors, indexing failures.

#### 2.5 Content Formatting and File Conversion

- **Overview:** Converts HTML to formatted text, then to a text file for email attachment.
- **Nodes Involved:**  
  - Markdown  
  - Code  
  - Convert to File  
  - Gmail

- **Node Details:**

  - **Markdown**  
    - (Described above in 2.4)

  - **Code**  
    - Type: Code  
    - Role: Converts HTML content to formatted plain text with bold headings and spacing.  
    - Key Code: Replaces HTML tags (h1-h6 to bold, paragraphs to newlines, removes other tags).  
    - Inputs: HTML from Markdown node  
    - Outputs: Text content string  
    - Edge Cases: Unexpected HTML structure.

  - **Convert to File**  
    - Type: Convert To File  
    - Role: Converts formatted text content into a text file for email attachment.  
    - Config: Operation "toText", source property "textContent".  
    - Inputs: Text content from Code node  
    - Outputs: Text file binary data  
    - Edge Cases: Conversion errors.

  - **Gmail**  
    - Type: Gmail  
    - Role: Sends email to user with analysis results and chatbot link.  
    - Config:  
      - Recipient: User email from form input  
      - Subject: "Analysis of the documents provided"  
      - Message: Includes greeting, analysis note, and chatbot webhook URL  
      - Attachments: Text file from Convert to File  
    - Inputs: Text file attachment, user email  
    - Outputs: Email sent confirmation  
    - Edge Cases: Gmail API auth errors, invalid email addresses.

#### 2.6 Vector Storage and Chatbot Interaction

- **Overview:** Stores document embeddings in Pinecone and provides a chatbot webhook for user queries answered via Google Gemini LLM.
- **Nodes Involved:**  
  - Pinecone Vector Store1  
  - Embeddings Mistral Cloud1  
  - Vector Store Retriever  
  - Question and Answer Chain  
  - Google Gemini Chat Model2  
  - AI Agent1  
  - Google Gemini Chat Model3  
  - When chat message received

- **Node Details:**

  - **Pinecone Vector Store1**  
    - Type: Vector Store Pinecone  
    - Role: Stores embeddings for chatbot retrieval.  
    - Config: Uses same Pinecone index "samuraichamploo".  
    - Inputs: Embeddings from Embeddings Mistral Cloud1  
    - Outputs: Confirmation of storage  
    - Edge Cases: Index access errors.

  - **Embeddings Mistral Cloud1**  
    - Type: Embeddings Provider  
    - Role: Generates embeddings for chatbot queries and stored documents.  
    - Inputs: Text data from document loader or user queries  
    - Outputs: Embeddings vectors  
    - Edge Cases: API failures.

  - **Vector Store Retriever**  
    - Type: Retriever  
    - Role: Retrieves relevant document embeddings from Pinecone based on user query.  
    - Inputs: Query embeddings  
    - Outputs: Retrieved context for QA chain  
    - Edge Cases: No relevant documents found.

  - **Question and Answer Chain**  
    - Type: Retrieval QA Chain  
    - Role: Answers user questions using retrieved context with Google Gemini LLM.  
    - Config: System prompt instructs to answer only if known, else admit ignorance.  
    - Inputs: Retrieved context and user query  
    - Outputs: Answer text  
    - Edge Cases: Ambiguous queries, missing context.

  - **Google Gemini Chat Model2**  
    - Type: Google Gemini LLM  
    - Role: Backend LLM for QA chain.  
    - Inputs: QA prompt  
    - Outputs: Generated answer  
    - Edge Cases: API errors.

  - **AI Agent1**  
    - Type: Langchain Agent  
    - Role: Post-processes QA chain output, rephrases prompt, returns text-only response.  
    - Config: System message instructs rephrasing and text-only output.  
    - Inputs: QA chain output  
    - Outputs: Final chatbot response  
    - Edge Cases: Formatting issues.

  - **Google Gemini Chat Model3**  
    - Type: Google Gemini LLM  
    - Role: Backend LLM for AI Agent1.  
    - Inputs: Prompt from AI Agent1  
    - Outputs: Final chatbot text response  
    - Edge Cases: API errors.

  - **When chat message received**  
    - Type: Chat Trigger (Webhook)  
    - Role: Public webhook that triggers chatbot interaction on user message.  
    - Config: Public access enabled.  
    - Inputs: User chat message  
    - Outputs: Passes message to QA chain  
    - Edge Cases: Unauthorized access, malformed messages.

---

### 3. Summary Table

| Node Name                 | Node Type                           | Functional Role                                    | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                  |
|---------------------------|-----------------------------------|---------------------------------------------------|------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| On form submission4        | Form Trigger                      | Entry point; receives files and email             | -                            | split the binary item        |                                                                                                              |
| split the binary item      | Code                             | Splits multiple uploaded files into individual items | On form submission4           | Loop Over Items1             |                                                                                                              |
| Loop Over Items1           | SplitInBatches                   | Processes each file item in batches                | split the binary item         | Aggregate1, Parsing the document |                                                                                                              |
| Parsing the document       | HTTP Request                    | Uploads file to Llama Cloud API for parsing        | Loop Over Items1              | Check the parsing status     |                                                                                                              |
| Check the parsing status   | HTTP Request                    | Polls parsing job status                            | Parsing the document          | If2                         |                                                                                                              |
| If2                       | If                              | Branches on parsing success                         | Check the parsing status      | Provide the markdown, Check the parsing status |                                                                                                              |
| Provide the markdown       | HTTP Request                    | Retrieves parsed markdown content                   | If2                          | Loop Over Items1             |                                                                                                              |
| Aggregate1                | Aggregate                       | Aggregates markdown content from all files         | Loop Over Items1              | Translator Agent             |                                                                                                              |
| Translator Agent           | Langchain Agent                 | Detects language and translates content            | Aggregate1                   | Analyzer Agent, Convert to File4 |                                                                                                              |
| Google Gemini Chat Model5  | Google Gemini LLM               | Backend LLM for Translator Agent                    | Translator Agent             | Translator Agent             |                                                                                                              |
| Analyzer Agent             | Langchain Agent                 | Performs deep content analysis                       | Translator Agent             | Markdown, AI Agent, Information Extractor, Convert to File4 |                                                                                                              |
| Google Gemini Chat Model6  | Google Gemini LLM               | Backend LLM for Analyzer Agent                       | Analyzer Agent               | Analyzer Agent               |                                                                                                              |
| Markdown                  | Markdown                       | Converts markdown to HTML                            | Analyzer Agent               | Code                        |                                                                                                              |
| Code                      | Code                           | Converts HTML to formatted plain text               | Markdown                    | Convert to File              |                                                                                                              |
| Convert to File            | Convert To File                | Converts text to file for email attachment          | Code                        | Gmail                       |                                                                                                              |
| Gmail                     | Gmail                         | Sends email with analysis and chatbot link          | Convert to File             | -                           |                                                                                                              |
| Convert to File4           | Convert To File                | Converts analyzed text to file for vector storage   | Analyzer Agent, AI Agent     | Pinecone Vector Store        |                                                                                                              |
| Pinecone Vector Store      | Vector Store Pinecone          | Stores embeddings of analyzed content                | Convert to File4, Embeddings Mistral Cloud | -                           | Sticky Note: "This subflow is responsible for storing the translated as well as the analyzed contents into the vector database to feed as a knowledge to the chatbot" |
| Embeddings Mistral Cloud  | Embeddings Provider            | Generates embeddings for vector storage              | -                          | Pinecone Vector Store        |                                                                                                              |
| When chat message received | Chat Trigger (Webhook)          | Public webhook for chatbot queries                   | -                          | Question and Answer Chain    | Sticky Note: "The below workflow is a chatbot workflow which will be triggered when a user types his/her prompt related to document the user provided for analysis on the chatbot link which was ent to the user via mail." |
| Question and Answer Chain  | Retrieval QA Chain             | Answers user questions using Pinecone context        | Vector Store Retriever      | AI Agent1                   |                                                                                                              |
| Vector Store Retriever     | Retriever                     | Retrieves relevant embeddings from Pinecone          | Pinecone Vector Store1      | Question and Answer Chain    |                                                                                                              |
| Pinecone Vector Store1     | Vector Store Pinecone          | Stores embeddings for chatbot retrieval               | Embeddings Mistral Cloud1   | Vector Store Retriever       |                                                                                                              |
| Embeddings Mistral Cloud1 | Embeddings Provider            | Generates embeddings for chatbot queries             | -                          | Pinecone Vector Store1       |                                                                                                              |
| AI Agent                  | Langchain Agent               | Enhances readability and formatting                   | Markdown                    | Information Extractor, Convert to File4 |                                                                                                              |
| Information Extractor      | Langchain Information Extractor | Extracts structured info from analyzed content       | AI Agent                    | Google Gemini Chat Model1    | Sticky Note1: "These two subflows are for trial purpose"                                                    |
| Google Gemini Chat Model1  | Google Gemini LLM             | Backend LLM for Information Extractor                 | Information Extractor       | -                           | Sticky Note1: "These two subflows are for trial purpose"                                                    |
| AI Agent1                 | Langchain Agent               | Rephrases QA chain output for chatbot response        | Question and Answer Chain   | -                           |                                                                                                              |
| Google Gemini Chat Model3  | Google Gemini LLM             | Backend LLM for AI Agent1                              | AI Agent1                  | -                           |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Configure form with fields:  
     - file1 (file, required)  
     - file2 (file, optional)  
     - provide your mail Id (text, required)  
   - Set webhook to public access.

2. **Add Code Node "split the binary item"**  
   - Purpose: Split each uploaded file into separate items.  
   - Use provided JavaScript code to iterate over binary fields and create one item per file.

3. **Add SplitInBatches Node "Loop Over Items1"**  
   - Connect from "split the binary item".  
   - Default batch size is fine.  
   - Outputs to two branches:  
     - "Parsing the document"  
     - "Aggregate1" (for aggregated markdown later).

4. **Add HTTP Request Node "Parsing the document"**  
   - Method: POST  
   - URL: https://api.cloud.llamaindex.ai/api/parsing/upload  
   - Send binary file as multipart/form-data parameter named "file".  
   - Headers:  
     - accept: application/json  
     - Authorization: Bearer <Llama Cloud API token> (use credentials or secrets)  
   - Connect input from Loop Over Items1.

5. **Add HTTP Request Node "Check the parsing status"**  
   - Method: GET  
   - URL: https://api.cloud.llamaindex.ai/api/parsing/job/{{ $json.id }}  
   - Headers same as above.  
   - Connect from "Parsing the document".

6. **Add If Node "If2"**  
   - Condition: $json.status equals "SUCCESS".  
   - True branch: "Provide the markdown"  
   - False branch: loops back to "Check the parsing status" (polling).

7. **Add HTTP Request Node "Provide the markdown"**  
   - Method: GET  
   - URL: https://api.cloud.llamaindex.ai/api/parsing/job/{{ $json.id }}/result/markdown  
   - Headers same as above.  
   - Connect from If2 (true branch).

8. **Connect "Provide the markdown" output to "Loop Over Items1" input (second branch)**  
   - This allows aggregation of markdown from all files.

9. **Add Aggregate Node "Aggregate1"**  
   - Aggregate field: markdown  
   - Input from Loop Over Items1 (markdown outputs).

10. **Add Langchain Agent Node "Translator Agent"**  
    - Text input: aggregated markdown from Aggregate1.  
    - System message: instruct to detect language and translate non-English content to English, preserving English parts.  
    - Connect output to "Analyzer Agent" and "Convert to File4".

11. **Add Google Gemini LLM Node "Google Gemini Chat Model5"**  
    - Model: models/gemini-1.5-flash  
    - Connect as language model for "Translator Agent".

12. **Add Langchain Agent Node "Analyzer Agent"**  
    - Text input: output from Translator Agent.  
    - System message: detailed analysis instructions with clarity, structure, duplicate handling, actionable resolutions.  
    - Connect output to "Markdown", "AI Agent", "Information Extractor", and "Convert to File4".

13. **Add Google Gemini LLM Node "Google Gemini Chat Model6"**  
    - Model: models/gemini-1.5-flash  
    - Connect as language model for "Analyzer Agent".

14. **Add Markdown Node "Markdown"**  
    - Mode: markdownToHtml  
    - Source: output from Analyzer Agent.

15. **Add Code Node "Code"**  
    - JavaScript to convert HTML to formatted text (bold headings, spacing).  
    - Input: HTML from Markdown node.

16. **Add Convert To File Node "Convert to File"**  
    - Operation: toText  
    - Source property: textContent (from Code node).  
    - Connect output to Gmail node.

17. **Add Gmail Node "Gmail"**  
    - Credentials: Gmail OAuth2 with API access.  
    - Send To: user email from form input.  
    - Subject: "Analysis of the documents provided"  
    - Message: includes greeting, analysis note, and chatbot link webhook URL.  
    - Attachments: file from Convert to File node.

18. **Add Convert To File Node "Convert to File4"**  
    - Operation: toText  
    - Source property: output from Analyzer Agent and AI Agent chain.  
    - Connect output to Pinecone Vector Store.

19. **Add Pinecone Vector Store Node "Pinecone Vector Store"**  
    - Mode: insert  
    - Pinecone index: "samuraichamploo" (configure with Pinecone credentials).  
    - Connect input from Convert to File4 and Embeddings Mistral Cloud.

20. **Add Embeddings Node "Embeddings Mistral Cloud"**  
    - Connect to Pinecone Vector Store.

21. **Set up Chatbot Subflow:**

    - **When chat message received** (Chat Trigger)  
      - Public webhook enabled.

    - **Question and Answer Chain**  
      - System prompt: answer only if known, else say unknown.  
      - Connect input from Vector Store Retriever.

    - **Vector Store Retriever**  
      - Connect input from Pinecone Vector Store1.

    - **Pinecone Vector Store1**  
      - Connect input from Embeddings Mistral Cloud1.

    - **Embeddings Mistral Cloud1**  
      - Connect to Pinecone Vector Store1.

    - **Google Gemini Chat Model2**  
      - Model: models/gemini-1.5-flash  
      - Connect to Question and Answer Chain.

    - **AI Agent1**  
      - System message: rephrase prompt and provide text-only response.  
      - Connect input from Question and Answer Chain.

    - **Google Gemini Chat Model3**  
      - Model: models/gemini-1.5-flash  
      - Connect to AI Agent1.

22. **Credentials Setup:**

    - Llama Cloud API token for HTTP Requests.  
    - Google Gemini LLM API keys for Langchain Google Gemini nodes.  
    - Pinecone API key and index configuration.  
    - Gmail OAuth2 credentials with API access.

23. **Test the workflow end-to-end:**  
    - Submit files and email via form.  
    - Verify parsing, translation, analysis, vector storage.  
    - Confirm email receipt with attachment and chatbot link.  
    - Test chatbot interaction via webhook.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow enables multilingual document analysis and chatbot creation using Llama Cloud API, Google Gemini LLM, Pinecone, and Gmail. | Workflow description and purpose.                                                                     |
| Chatbot webhook URL: https://pavithranvh28.app.n8n.cloud/webhook/8c5c9e83-f595-4e4b-b45c-544a9a0840c4/chat               | Provided in email sent to users for chatbot interaction.                                              |
| To add more languages, update translation logic in Translator Agent node.                                               | Customization guidance.                                                                                |
| To adjust analysis depth, modify prompts in Analyzer Agent node.                                                        | Customization guidance.                                                                                |
| To change chatbot behavior, retrain or reconfigure chatbot to utilize Pinecone index contextually.                      | Customization guidance.                                                                                |
| Sticky Note on chatbot workflow: "The below workflow is a chatbot workflow which will be triggered when a user types his/her prompt related to document the user provided for analysis on the chatbot link which was ent to the user via mail." | Sticky note attached to chatbot trigger and related nodes.                                            |
| Sticky Note on vector storage subflow: "This subflow is responsible for storing the translated as well as the analyzed contents into the vector database to feed as a knowledge to the chatbot." | Sticky note attached to Pinecone Vector Store node.                                                   |
| Sticky Note on trial subflows: "These two subflows are for trial purpose."                                              | Sticky note attached to Information Extractor and Google Gemini Chat Model1 nodes.                     |

---

This structured documentation provides a comprehensive understanding of the workflow’s architecture, node configurations, and integration points, enabling advanced users and AI agents to reproduce, modify, and troubleshoot the workflow effectively.