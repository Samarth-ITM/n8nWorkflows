Create AI-Generated Books with GPT-4.1-mini, DALL-E, Google Drive and AWS S3

https://n8nworkflows.xyz/workflows/create-ai-generated-books-with-gpt-4-1-mini--dall-e--google-drive-and-aws-s3-7482


# Create AI-Generated Books with GPT-4.1-mini, DALL-E, Google Drive and AWS S3

### 1. Workflow Overview

This workflow automates the creation of AI-generated books using a multi-agent orchestration approach with GPT-4.1-mini, DALL·E, Google Drive, and AWS S3 integrations. It targets content creators, educators, and developers interested in automated book generation and multi-agent AI workflows. The workflow is organized into the following logical blocks:

- **1.1 Input Reception**  
  Begins with receiving a chat message as a trigger to start the book creation process.

- **1.2 Book Brief Creation**  
  Generates a structured book brief including title, subtitle, audience, language, chapter count, and other metadata from the initial user request.

- **1.3 Content Expansion via Multi-Agent Collaboration**  
  Orchestrates specialized AI agents:  
  - *Designer Agent* creates a DALL·E-ready prompt for the book cover design.  
  - *Content Writer Agent* produces the full book content in Markdown format.  
  These are coordinated by the *Book Writer Agent* to produce final structured outputs.

- **1.4 Cover Image Generation and Storage**  
  Generates the book cover image with DALL·E, then uploads it to AWS S3 for secure storage.

- **1.5 Metadata Configuration and Book Format Conversion**  
  Configures metadata variables and converts the Markdown book content into HTML.

- **1.6 Google Drive Upload and PDF Conversion**  
  Uploads the HTML book to Google Drive, converts it to a PDF document, and archives the final PDF in a specified Google Drive folder for safe keeping.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  The workflow is initiated by a chat message, serving as the entry point for user input to trigger book creation.

- **Nodes Involved:**  
  - When chat message received  
  - Sticky Note1

- **Node Details:**  
  - **When chat message received**  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Trigger node that listens for incoming chat messages to start the workflow.  
    - Configuration: Default webhook with no special options.  
    - Inputs: External chat interface or webhook trigger.  
    - Outputs: Triggers the next node, *Book Brief Agent*.  
    - Edge Cases: Possible webhook timeout or failure to receive messages. Requires proper webhook exposure.

#### 2.2 Book Brief Creation

- **Overview:**  
  Processes the initial chat message to generate a structured JSON book brief including title, subtitle, audience, and writing parameters.

- **Nodes Involved:**  
  - Structured Output Parser  
  - Book Brief Agent  
  - gpt-4.1-mini  
  - Sticky Note2

- **Node Details:**  
  - **Book Brief Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Generates planning JSON with book metadata from user input.  
    - Configuration: Uses GPT-4.1-mini model via `gpt-4.1-mini` node; system prompt enforces strict JSON output with fields `book_brief`, `cover_design_brief`, and `writing_brief`.  
    - Key Expressions: Injects user chat input into prompt for topic; includes default parameters for audience (Kid), language (English), target word count (10,000), chapter count (5), author (Trung Tran).  
    - Inputs: Output from chat trigger (user message).  
    - Outputs: JSON book brief parsed by *Structured Output Parser*.  
    - Edge Cases: Potential JSON parse errors if model output deviates; prompt strictness helps mitigate.

  - **Structured Output Parser**  
    - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
    - Role: Parses raw LLM output into structured JSON matching schema example.  
    - Configuration: Uses example JSON schema for validation.  
    - Inputs: Raw text from *Book Brief Agent*.  
    - Outputs: Parsed JSON for downstream use.

  - **gpt-4.1-mini**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Language model backend for *Book Brief Agent*.  
    - Configuration: Model set to `gpt-4.1-mini`, OpenAI API credentials configured.  
    - Inputs/Outputs: Connected upstream to trigger; downstream to agent node.

#### 2.3 Content Expansion via Multi-Agent Collaboration

- **Overview:**  
  Coordinates two specialized AI agents to generate the book cover prompt and write the full book content in Markdown format, orchestrated by the *Book Writer Agent*.

- **Nodes Involved:**  
  - Book Writer Agent  
  - Designer Agent  
  - Content Writer Agent  
  - gpt-4.1-mini1  
  - gpt-4.1-mini2  
  - Structured Output Parser1  
  - OpenAI Chat Model  
  - Sticky Note3

- **Node Details:**  
  - **Book Writer Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Orchestrates *Designer Agent* and *Content Writer Agent* to produce final JSON output with cover prompt and book Markdown content.  
    - Configuration: System message instructs orchestration logic; input is JSON book brief output; output must be JSON with fields `cover_prompt` and `book_content_markdown`.  
    - Inputs: Output from *Book Brief Agent*.  
    - Outputs: Routed to *Structured Output Parser1*.  
    - Edge Cases: Coordination failures if sub-agent outputs are malformed or delayed.

  - **Designer Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agentTool`  
    - Role: Generates a DALL·E-ready prompt for book cover design based on cover design brief.  
    - Configuration: Prompt specifies visual motifs, composition, style, color palette, typography, and aspect ratio; returns prompt only, no images.  
    - Inputs: Text from *OpenAI Chat Model* or *Book Writer Agent*.  
    - Outputs: Sends prompt string back to *Book Writer Agent*.  
    - Edge Cases: Prompt generation failure or incomplete prompt.

  - **Content Writer Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agentTool`  
    - Role: Writes the full book content in Markdown format following writing brief instructions.  
    - Configuration: Output constrained to Markdown only, no extraneous text or fabricated citations; includes full chapters and sections.  
    - Inputs: Writing brief JSON passed from *Book Writer Agent*.  
    - Outputs: Markdown string back to *Book Writer Agent*.  
    - Edge Cases: Incomplete content generation or formatting errors.

  - **gpt-4.1-mini1 and gpt-4.1-mini2**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Provide the GPT-4.1-mini LLM backend for *Book Writer Agent* and *Content Writer Agent* respectively.  
    - Inputs/Outputs: Connected to respective agent nodes.

  - **Structured Output Parser1**  
    - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
    - Role: Parses final JSON output from *Book Writer Agent* containing cover prompt and book Markdown.  
    - Configuration: Uses example schema with `bookCoverPrompt` and `bookContentMarkdown`.

  - **OpenAI Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Backend LLM node used by *Designer Agent*.  
    - Configuration: Model set to `gpt-4.1-mini`.

#### 2.4 Cover Image Generation and Storage

- **Overview:**  
  Uses the DALL·E model to generate a book cover image from the cover prompt, then uploads the image to AWS S3 for public storage and access.

- **Nodes Involved:**  
  - Generate cover image  
  - Upload to AWS S3  
  - Configure metadata  
  - Sticky Note4

- **Node Details:**  
  - **Generate cover image**  
    - Type: `@n8n/n8n-nodes-langchain.openAi`  
    - Role: Uses OpenAI’s DALL·E 2 to generate an image from the cover prompt.  
    - Configuration: Model set to `dall-e-2`, prompt dynamically injected from prior agent output `bookCoverPrompt`.  
    - Inputs: Cover prompt string from *Book Writer Agent*.  
    - Outputs: Image file data (binary).  
    - Edge Cases: API rate limits, prompt errors, or image generation failure.

  - **Upload to AWS S3**  
    - Type: `n8n-nodes-base.awsS3`  
    - Role: Uploads the generated cover image to a specified S3 bucket with public-read access.  
    - Configuration: Filename dynamically generated from book title and workflow ID; ACL set to `publicRead`; bucket name specified.  
    - Inputs: Image file from *Generate cover image*.  
    - Outputs: Metadata including public URL for the image.  
    - Credentials: AWS credentials configured.  
    - Edge Cases: Network errors, permission issues, bucket misconfiguration.

  - **Configure metadata**  
    - Type: `n8n-nodes-base.set`  
    - Role: Sets workflow variables including Google Drive folder ID, S3 base URL, and constructs final picture URL using book title and workflow ID.  
    - Configuration: String assignments with dynamic expressions for picture URL.  
    - Inputs: Metadata and uploaded file info.  
    - Outputs: Variables used downstream for HTML generation.

#### 2.5 Metadata Configuration and Book Format Conversion

- **Overview:**  
  Converts the Markdown book content into HTML and uploads the content to Google Drive for storage and further processing.

- **Nodes Involved:**  
  - Build book html  
  - Upload to Google Drive  
  - Sticky Note5

- **Node Details:**  
  - **Build book html**  
    - Type: `n8n-nodes-base.markdown`  
    - Role: Converts Markdown book content into HTML format.  
    - Configuration: Mode set to `markdownToHtml`; Markdown source combines cover image URL and book content Markdown replacing horizontal rules with line breaks.  
    - Inputs: Markdown content and picture URL from previous nodes.  
    - Outputs: HTML content for upload.  
    - Edge Cases: Markdown syntax errors or invalid image URLs.

  - **Upload to Google Drive**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: Uploads the HTML book as a Google Docs file using Google Drive API multipart upload.  
    - Configuration: Multipart HTTP POST with JSON metadata (title, mimeType, parents folder) and HTML content in the body; uses OAuth2 credentials.  
    - Inputs: HTML content from *Build book html* and Drive folder ID from metadata.  
    - Outputs: File metadata including Google Drive file ID.  
    - Credentials: Google Drive OAuth2 API.  
    - Edge Cases: API quota limits, authentication failures, malformed multipart request.

#### 2.6 Google Drive PDF Conversion and Archival

- **Overview:**  
  Converts the Google Docs HTML document into a PDF, then archives the PDF in a specified Google Drive folder.

- **Nodes Involved:**  
  - Convert document to PDF  
  - Archive to Drive Folder  
  - Sticky Note6

- **Node Details:**  
  - **Convert document to PDF**  
    - Type: `n8n-nodes-base.googleDrive`  
    - Role: Downloads the Google Docs document as a PDF file.  
    - Configuration: Uses Google Drive API export operation with MIME type `application/pdf`.  
    - Inputs: Google Docs file ID from upload node.  
    - Outputs: PDF file binary data.  
    - Credentials: Google Drive OAuth2 API.  
    - Edge Cases: File permission issues, conversion failures.

  - **Archive to Drive Folder**  
    - Type: `n8n-nodes-base.googleDrive`  
    - Role: Uploads the generated PDF file to a specific Google Drive folder for archiving.  
    - Configuration: Sets file name based on book title and PDF extension; target folder ID specified.  
    - Inputs: PDF binary data from conversion node.  
    - Outputs: Confirmation metadata of archived file.  
    - Credentials: Google Drive OAuth2 API.  
    - Edge Cases: Upload failures, quota exceedance, folder permission issues.

---

### 3. Summary Table

| Node Name                 | Node Type                                    | Functional Role                         | Input Node(s)                    | Output Node(s)                   | Sticky Note                                               |
|---------------------------|----------------------------------------------|---------------------------------------|---------------------------------|---------------------------------|-----------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger         | Workflow trigger on user chat input   | -                               | Book Brief Agent                | ### 1. Trigger on Chat Message                            |
| Book Brief Agent           | @n8n/n8n-nodes-langchain.agent               | Generate structured book brief JSON   | When chat message received       | Book Writer Agent              | ### 2. Generate Book Brief                                |
| Structured Output Parser   | @n8n/n8n-nodes-langchain.outputParserStructured | Parse Book Brief JSON output          | Book Brief Agent                | Book Brief Agent (main)         |                                                           |
| gpt-4.1-mini              | @n8n/n8n-nodes-langchain.lmChatOpenAi         | LLM backend for Book Brief Agent      | When chat message received       | Book Brief Agent                |                                                           |
| Book Writer Agent          | @n8n/n8n-nodes-langchain.agent               | Orchestrate Designer & Content Writer | Book Brief Agent                | Generate cover image           | ### 3. Expand with Book Writer Agent                      |
| Designer Agent             | @n8n/n8n-nodes-langchain.agentTool           | Generate DALL·E prompt for cover      | OpenAI Chat Model               | Book Writer Agent (ai_tool)    |                                                           |
| Content Writer Agent       | @n8n/n8n-nodes-langchain.agentTool           | Write full book content in Markdown   | Book Writer Agent              | Book Writer Agent (ai_tool)    |                                                           |
| gpt-4.1-mini1             | @n8n/n8n-nodes-langchain.lmChatOpenAi         | LLM backend for Book Writer Agent     | Book Brief Agent                | Book Writer Agent              |                                                           |
| gpt-4.1-mini2             | @n8n/n8n-nodes-langchain.lmChatOpenAi         | LLM backend for Content Writer Agent  | Book Writer Agent              | Content Writer Agent           |                                                           |
| Structured Output Parser1  | @n8n/n8n-nodes-langchain.outputParserStructured | Parse final JSON from Book Writer     | Book Writer Agent              | Book Writer Agent (main)       |                                                           |
| OpenAI Chat Model          | @n8n/n8n-nodes-langchain.lmChatOpenAi         | LLM backend for Designer Agent        | Book Writer Agent              | Designer Agent                |                                                           |
| Generate cover image       | @n8n/n8n-nodes-langchain.openAi                | Generate book cover image via DALL·E | Book Writer Agent              | Upload to AWS S3              | ### 4. Create Book Cover Image                           |
| Upload to AWS S3           | n8n-nodes-base.awsS3                          | Upload generated cover image          | Generate cover image            | Configure metadata            |                                                           |
| Configure metadata         | n8n-nodes-base.set                            | Set variables for URLs and IDs        | Upload to AWS S3               | Build book html              |                                                           |
| Build book html            | n8n-nodes-base.markdown                       | Convert Markdown to HTML               | Configure metadata             | Upload to Google Drive       | ### 5. Build Book in HTML                                |
| Upload to Google Drive     | n8n-nodes-base.httpRequest                     | Upload HTML book file to Google Drive | Build book html               | Convert document to PDF      |                                                           |
| Convert document to PDF    | n8n-nodes-base.googleDrive                    | Convert Google Docs to PDF             | Upload to Google Drive         | Archive to Drive Folder       | ### 6. Convert to PDF                                   |
| Archive to Drive Folder    | n8n-nodes-base.googleDrive                    | Archive PDF in Google Drive folder    | Convert document to PDF        | -                           |                                                           |
| Sticky Note                | n8n-nodes-base.stickyNote                      | Documentation note                    | -                             | -                           | See detailed notes at start                              |
| Sticky Note1               | n8n-nodes-base.stickyNote                      | Documentation note on trigger         | -                             | -                           | ### 1. Trigger on Chat Message                            |
| Sticky Note2               | n8n-nodes-base.stickyNote                      | Documentation note on book brief      | -                             | -                           | ### 2. Generate Book Brief                                |
| Sticky Note3               | n8n-nodes-base.stickyNote                      | Documentation note on book writer     | -                             | -                           | ### 3. Expand with Book Writer Agent                      |
| Sticky Note4               | n8n-nodes-base.stickyNote                      | Documentation note on cover image     | -                             | -                           | ### 4. Create Book Cover Image                           |
| Sticky Note5               | n8n-nodes-base.stickyNote                      | Documentation note on HTML build      | -                             | -                           | ### 5. Build Book in HTML                                |
| Sticky Note6               | n8n-nodes-base.stickyNote                      | Documentation note on PDF conversion  | -                             | -                           | ### 6. Convert to PDF                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create trigger node: "When chat message received"**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Configuration: Default webhook trigger to start on incoming chat messages.

2. **Add "gpt-4.1-mini" LLM node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Model: set to `gpt-4.1-mini`  
   - Credentials: Configure OpenAI API key.

3. **Add "Structured Output Parser" node**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Paste JSON schema example for book brief parsing.

4. **Add "Book Brief Agent" node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - System prompt: Enforce strict JSON output of book brief, cover design brief, writing brief as per schema.  
   - Input: Use expression to inject chat message topic.  
   - Connect LLM node as language model.  
   - Connect preceding chat trigger node as input.

5. **Add "Book Writer Agent" node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - System prompt: Orchestration agent to coordinate Designer and Content Writer agents; output JSON with `cover_prompt` and `book_content_markdown`.  
   - Connect input from *Book Brief Agent* output parser.  
   - Connect separate LLM node "gpt-4.1-mini1" for this agent.

6. **Add "Designer Agent" node**  
   - Type: `@n8n/n8n-nodes-langchain.agentTool`  
   - Prompt: Detailed instructions for DALL·E cover prompt creation based on cover design brief.  
   - Connect LLM node "OpenAI Chat Model" as backend.  
   - Connect output to *Book Writer Agent* as tool.

7. **Add "Content Writer Agent" node**  
   - Type: `@n8n/n8n-nodes-langchain.agentTool`  
   - Prompt: Detailed instructions for writing full book content in Markdown format following writing brief.  
   - Connect LLM node "gpt-4.1-mini2" as backend.  
   - Connect output to *Book Writer Agent* as tool.

8. **Add "Structured Output Parser1" node**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Schema: `{ "bookCoverPrompt": "string", "bookContentMarkdown": "string" }`  
   - Connect input from *Book Writer Agent* output.

9. **Add "Generate cover image" node**  
   - Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Model: `dall-e-2`  
   - Prompt: Use expression to inject `bookCoverPrompt` from *Book Writer Agent* output.  
   - Credentials: OpenAI API key.

10. **Add "Upload to AWS S3" node**  
    - Type: `n8n-nodes-base.awsS3`  
    - Operation: Upload file  
    - Filename: Dynamic based on book title and workflow ID.  
    - Bucket: Your configured S3 bucket name.  
    - ACL: `publicRead` for public access.  
    - Credentials: AWS credentials.

11. **Add "Configure metadata" node**  
    - Type: `n8n-nodes-base.set`  
    - Assign variables: Drive Folder ID, S3 Base URL, Picture URL dynamically constructed from book title and workflow ID.

12. **Add "Build book html" node**  
    - Type: `n8n-nodes-base.markdown`  
    - Mode: Convert Markdown to HTML  
    - Markdown input: Combine image markdown with book content markdown (replace horizontal rules with line breaks).

13. **Add "Upload to Google Drive" node**  
    - Type: `n8n-nodes-base.httpRequest`  
    - Method: POST multipart upload to Google Drive API  
    - URL: `https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart&supportsAllDrives=true`  
    - Body: Multipart content with JSON metadata and HTML content  
    - Credentials: Google Drive OAuth2 API  
    - Parameters: Set file name and parent folder ID.

14. **Add "Convert document to PDF" node**  
    - Type: `n8n-nodes-base.googleDrive`  
    - Operation: Download file with export to PDF MIME type.  
    - Input: Google Docs file ID from previous node.  
    - Credentials: Google Drive OAuth2 API.

15. **Add "Archive to Drive Folder" node**  
    - Type: `n8n-nodes-base.googleDrive`  
    - Operation: Upload file  
    - Folder: Specify archive folder ID.  
    - Credentials: Google Drive OAuth2 API.

16. **Connect nodes in sequence:**  
    When chat message received → Book Brief Agent (via gpt-4.1-mini & Structured Output Parser) → Book Writer Agent (with Designer and Content Writer Agents and respective LLMs) → Structured Output Parser1 → Generate cover image → Upload to AWS S3 → Configure metadata → Build book html → Upload to Google Drive → Convert document to PDF → Archive to Drive Folder

17. **Add Sticky Notes** with workflow description, block explanations, and usage instructions for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| This workflow is a complete multi-agent orchestration template for automated book generation combining multiple AI agents with structured output parsing and cloud storage integrations. It demonstrates best practices for using n8n's AI Tool Node, OpenAI GPT-4.1-mini, DALL·E for image generation, AWS S3 for image hosting, and Google Drive for document management. The workflow is freely reusable and customizable.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Workflow description and usage notes from Sticky Note at workflow start.                                                       |
| Video demonstration available: [Watch the video](https://www.youtube.com/watch?v=o1x8Tw_7FwQ) with preview image linked.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | https://www.youtube.com/watch?v=o1x8Tw_7FwQ                                                                                     |
| To customize: switch GPT models for speed or cost, add agents for editing or fact-checking, change output formats (e.g., EPUB, DOCX), integrate other cloud storage providers (Dropbox, OneDrive), or modify triggers to webhook or scheduled runs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Customization tips from Sticky Note content.                                                                                   |
| Requires n8n version supporting AI Tool Node and OpenAI API integration, AWS S3 bucket with public read access, Google Drive OAuth2 credentials with access to target folders. Ensure rate limits and API quotas are managed to avoid failures.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Setup requirements and credentials notes.                                                                                      |
| Source code is structured for debugging and extension: maintain strict JSON schema adherence in prompts and output parsers to minimize errors. Use error handling on each cloud integration node for robustness.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Best practices for error handling and structured prompt design.                                                               |

---

**Disclaimer:** The provided text and workflow exclusively originate from an automated n8n workflow. It fully complies with current content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.