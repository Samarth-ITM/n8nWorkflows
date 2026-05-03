📄🌐PDF2Blog - Create Blog Post on Ghost CRM from PDF Document

https://n8nworkflows.xyz/workflows/----pdf2blog---create-blog-post-on-ghost-crm-from-pdf-document-2522


# 📄🌐PDF2Blog - Create Blog Post on Ghost CRM from PDF Document

### 1. Workflow Overview

This workflow, titled **PDF2Blog - Create Blog Post on Ghost CRM from PDF Document**, automates the transformation of PDF documents into engaging blog posts published directly on a Ghost CMS instance. It is designed for content marketing teams, academic communicators, and digital publishers who want to dramatically reduce content production time while maintaining quality and SEO readiness.

**Logical Blocks:**

- **1.1 Upload PDF and Extract Text**: The workflow begins with a web form where users upload a PDF file. The PDF’s text is then extracted for processing.
- **1.2 AI-Powered Blog Post Creation**: Using OpenAI’s GPT model and LangChain nodes, the extracted text is analyzed and transformed into a structured JSON blog post containing a title and HTML-formatted content.
- **1.3 Post-Processing and Validation**: The JSON output is parsed and separated into title and content, validated, and checked for completeness.
- **1.4 Publishing to Ghost CMS**: If the validation passes, the blog post is created as a draft on the configured Ghost CMS instance.

---

### 2. Block-by-Block Analysis

#### 2.1 Upload PDF and Extract Text

- **Overview:**  
  This block handles user input via a form to upload a PDF file and extracts raw text content from the PDF for further processing.

- **Nodes Involved:**  
  - Upload PDF (formTrigger)  
  - Extract Text (extractFromFile)  
  - Sticky Note (labeling this block)

- **Node Details:**

  - **Upload PDF**  
    - Type: Form Trigger  
    - Role: Entry point; presents an HTML form to upload a single PDF file.  
    - Configuration:  
      - Webhook path: `/pdf`  
      - Form title: "PDF2Blog"  
      - Single file upload only, restricted to `.pdf` files, required field.  
      - Description: “Transform PDFs into captivating blog posts”  
    - Inputs: None (trigger)  
    - Outputs: Binary data of PDF file under property `Upload_PDF_File`  
    - Edge Cases: Missing file upload, incorrect file type, upload cancellations.

  - **Extract Text**  
    - Type: Extract From File  
    - Role: Extracts text content from the uploaded PDF binary.  
    - Configuration:  
      - Operation: PDF text extraction  
      - Binary property: `Upload_PDF_File` (from previous node)  
    - Inputs: Binary PDF file from Upload PDF node  
    - Outputs: Extracted text as JSON/plain text in the node’s output data  
    - Edge Cases: Corrupted PDFs, encrypted files, extraction failures, large PDFs causing timeouts.

  - **Sticky Note**  
    - Type: n8n sticky note  
    - Role: Visual documentation labeled “Upload PDF and Extract Text”

#### 2.2 AI-Powered Blog Post Creation

- **Overview:**  
  This block transforms extracted PDF text into a structured blog post (title and content) using AI models configured with detailed instructions for content structure, SEO, and formatting.

- **Nodes Involved:**  
  - gpt-4o-mini (OpenAI model node)  
  - Structured Output - JSON (outputParserStructured)  
  - Create Structured Blog Post (agent)  
  - Sticky Note (labeling this block)

- **Node Details:**

  - **Extract Text → Create Structured Blog Post** (connection)  
    - The raw text extracted from the PDF is passed as input text to the Create Structured Blog Post node.

  - **gpt-4o-mini**  
    - Type: LangChain LM Chat OpenAI node  
    - Role: Language model that can be used to generate or assist with text generation; linked as a language model for the agent node.  
    - Configuration:  
      - Model: `gpt-4o-mini-2024-07-18` (specialized model version)  
      - Response format: JSON object for structured results  
      - Credentials: OpenAI API key configured  
    - Inputs/Outputs: Receives prompts, provides AI-generated JSON output  
    - Edge Cases: API rate limits, invalid API key, model response failures.

  - **Structured Output - JSON**  
    - Type: LangChain structured output parser  
    - Role: Parses the AI-generated output into a defined JSON schema containing `"title"` and `"content"` fields.  
    - Configuration:  
      - JSON schema example specifies required fields to enforce output format  
    - Inputs: AI raw output  
    - Outputs: Parsed JSON with blog post data  
    - Edge Cases: Output format inconsistencies, parsing errors.

  - **Create Structured Blog Post**  
    - Type: LangChain Agent node  
    - Role: Core AI agent that analyzes the PDF text and creates a detailed blog post in JSON format.  
    - Configuration:  
      - Text input: extracted PDF text  
      - Agent type: conversationalAgent  
      - System message: Detailed instructions specifying blog post structure, SEO guidelines, HTML formatting, tone, length constraints, and content originality.  
      - Output parser enabled to enforce JSON format with `title` and `content` keys  
      - Retry on failure enabled, improving reliability  
    - Inputs: Extracted PDF text, uses gpt-4o-mini as the language model  
    - Outputs: JSON object with title and content HTML  
    - Edge Cases: AI output not conforming to expected structure, network/API failures, prompt misinterpretation.

  - **Sticky Note**  
    - Label: “Create Blog Post”

#### 2.3 Post-Processing and Validation

- **Overview:**  
  This block extracts and validates the blog post title and content from AI output, ensuring they are complete and formatted correctly before proceeding to publishing.

- **Nodes Involved:**  
  - Separate Title & Content (code)  
  - If (conditional check)  
  - Do Nothing (no operation node)  
  - Sticky Note (labeling this block)

- **Node Details:**

  - **Separate Title & Content**  
    - Type: Code node (JavaScript)  
    - Role: Extracts title and content properties from AI output JSON; cleans content by removing redundant H1 tags; validates presence and non-emptiness of both fields.  
    - Configuration:  
      - JavaScript code:  
        - Checks input data structure and presence of required fields (`output.title` and `output.content`).  
        - Removes any `<h1>` tags from content to avoid duplication.  
        - Throws errors if validation fails, logs errors for debugging.  
        - Returns an object with `title` and cleaned `content` or error details on failure.  
    - Inputs: AI JSON output from Create Structured Blog Post  
    - Outputs: Validated title and content or error object  
    - Edge Cases: Missing fields, empty content after cleanup, malformed input JSON, runtime exceptions.

  - **If**  
    - Type: Conditional node  
    - Role: Checks that both `title` and `content` are non-empty strings before continuing.  
    - Configuration:  
      - Condition: `title` not empty AND `content` not empty (strict validation)  
    - Inputs: Output from Separate Title & Content  
    - Outputs:  
      - True branch: proceeds to publishing  
      - False branch: executes Do Nothing node (halts workflow gracefully)  
    - Edge Cases: Unexpected JSON structure, false negatives due to whitespace-only strings.

  - **Do Nothing**  
    - Type: No Operation node  
    - Role: Gracefully ends the workflow if validation fails without errors or further processing.  
    - Inputs: If node false branch  
    - Outputs: None  
    - Edge Cases: None (safely swallows invalid input)

  - **Sticky Note**  
    - Label: “Post-Processing & Validation” (implied by placement)

#### 2.4 Publish Draft Blog Post to Ghost

- **Overview:**  
  This final block publishes the validated blog post as a draft to a Ghost CMS instance using the Ghost Admin API.

- **Nodes Involved:**  
  - Post to Ghost (ghost)  
  - Sticky Note (labeling this block)

- **Node Details:**

  - **Post to Ghost**  
    - Type: Ghost node  
    - Role: Creates a new blog post on Ghost CMS using the Admin API  
    - Configuration:  
      - Operation: Create post  
      - Title: from validated AI output title  
      - Content: from validated AI output content (HTML formatted)  
      - Source: adminApi (uses admin credentials)  
      - Additional fields: none specified (defaults apply)  
      - Credentials: Ghost Admin API credentials configured  
    - Inputs: Validated title and content from If node (true branch)  
    - Outputs: Ghost API response confirming post creation  
    - Edge Cases: Authentication failures, API rate limits, network issues, invalid content causing Ghost API rejection.

  - **Sticky Note**  
    - Label: “Publish Draft Blog Post to Ghost”

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                                   | Input Node(s)           | Output Node(s)          | Sticky Note                                          |
|-------------------------|----------------------------------|--------------------------------------------------|-------------------------|-------------------------|------------------------------------------------------|
| Upload PDF              | formTrigger                      | Entry point: upload PDF via web form             | None                    | Extract Text            | ## Upload PDF and Extract Text                        |
| Extract Text            | extractFromFile                  | Extract text content from uploaded PDF           | Upload PDF              | Create Structured Blog Post | ## Upload PDF and Extract Text                        |
| gpt-4o-mini             | LangChain LM Chat OpenAI         | Language model for AI content generation          | Create Structured Blog Post (as LM) | Create Structured Blog Post | ## Create Blog Post                                   |
| Structured Output - JSON| LangChain outputParserStructured | Parse AI output into JSON with title & content   | gpt-4o-mini             | Create Structured Blog Post | ## Create Blog Post                                   |
| Create Structured Blog Post | LangChain agent                 | Generate blog post JSON from extracted text      | Extract Text, gpt-4o-mini, Structured Output - JSON | Separate Title & Content | ## Create Blog Post                                   |
| Separate Title & Content | code                            | Extract, clean, and validate title & content     | Create Structured Blog Post | If                      |                                                      |
| If                      | if                              | Check title and content presence                   | Separate Title & Content | Post to Ghost (true), Do Nothing (false) |                                                      |
| Do Nothing              | noOp                            | Graceful halt if invalid content                    | If                      | None                    |                                                      |
| Post to Ghost           | ghost                           | Publish blog post draft to Ghost CMS              | If (true branch)         | None                    | ## Publish Draft Blog Post to Ghost                   |
| Sticky Note             | stickyNote                      | Visual documentation                              | None                    | None                    | ## Upload PDF and Extract Text                        |
| Sticky Note1            | stickyNote                      | Visual documentation                              | None                    | None                    | ## Create Blog Post                                   |
| Sticky Note3            | stickyNote                      | Visual documentation                              | None                    | None                    | ## Publish Draft Blog Post to Ghost                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node named "Upload PDF":**  
   - Set webhook path to `/pdf`  
   - Add a form field:  
     - Type: file  
     - Label: "Upload PDF File"  
     - Single file only  
     - Required  
     - Accept only `.pdf` files  
   - Set form title to "PDF2Blog" and description accordingly.

2. **Add an Extract From File node named "Extract Text":**  
   - Operation: PDF text extraction  
   - Binary property name: `Upload_PDF_File` (matches form upload key)  
   - Connect "Upload PDF" node’s main output to this node.

3. **Add a LangChain LM Chat OpenAI node named "gpt-4o-mini":**  
   - Model: `gpt-4o-mini-2024-07-18`  
   - Response Format: JSON object  
   - Credentials: Configure with valid OpenAI API credentials  
   - This node will serve as the language model for the agent.

4. **Add LangChain Output Parser Structured node named "Structured Output - JSON":**  
   - JSON schema example:  
     ```json
     {
       "title": "title",
       "content": "content"
     }
     ```  
   - Connect "gpt-4o-mini" output to this parser node.

5. **Add LangChain Agent node named "Create Structured Blog Post":**  
   - Agent: conversationalAgent  
   - Text input: set to the extracted text from "Extract Text" node (`{{$json.text}}`)  
   - Use the "gpt-4o-mini" node as the language model input.  
   - Enable output parser, linking to "Structured Output - JSON" node.  
   - System message prompt: include instructions for creating an SEO-friendly blog post with detailed structure, HTML tags, tone, length requirements, and formatting as detailed in the overview.  
   - Enable retry on failure.  
   - Connect "Extract Text" main output to this node’s text input, and "gpt-4o-mini" and "Structured Output - JSON" nodes as AI model and parser respectively.

6. **Add a Code node named "Separate Title & Content":**  
   - Paste the provided JavaScript code that validates, extracts, and cleans the title and content from the AI output JSON.  
   - Connect "Create Structured Blog Post" main output to this node.

7. **Add an If node named "If":**  
   - Condition:  
     - Check that `{{$json.title}}` is not empty string  
     - AND `{{$json.content}}` is not empty string  
   - Connect "Separate Title & Content" main output to "If" input.

8. **Add a Ghost node named "Post to Ghost":**  
   - Operation: Create post  
   - Title: `{{$json.title}}`  
   - Content: `{{$json.content}}` (HTML formatted)  
   - Source: `adminApi`  
   - Credentials: Configure with Ghost Admin API credentials  
   - Connect the true output of the "If" node to this node.

9. **Add a No Operation node named "Do Nothing":**  
   - Connect the false output of the "If" node to this node to gracefully end workflow on validation failure.

10. **Add Sticky Note nodes as documentation labels:**  
    - Label each logical block accordingly for visual clarity.

11. **Set workflow execution order as default (v1).**

12. **Test the workflow:**  
    - Upload valid PDF files via the form webhook.  
    - Confirm AI-generated blog post creation and publishing to Ghost.  
    - Handle errors via logs and node error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                                                        |
|---------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| The AI prompt includes detailed formatting instructions using HTML tags like `<p>`, `<blockquote>`, and heading tags to ensure proper blog structure. | System message in "Create Structured Blog Post" node.                                                                                 |
| The workflow achieves up to 95% time reduction in content production by automating PDF-to-blog conversion.                       | Workflow description highlights this as a key benefit.                                                                               |
| The workflow uses the Ghost Admin API requiring a valid Admin API key for publishing posts programmatically.                    | See "Post to Ghost" node credentials and configuration.                                                                               |
| This automation is useful for content marketing, academic communication, and digital publishing teams needing scalable content creation. | Use cases described in the workflow overview.                                                                                        |
| For more on n8n LangChain integration and advanced AI workflows see: https://n8n.io/integrations/n8n-nodes-langchain           | External resource for LangChain nodes used in this workflow.                                                                          |

---

This document provides a complete, detailed reference for understanding, reproducing, and extending the **PDF2Blog** workflow in n8n. It covers every node, logic path, and configuration, supporting both manual and automated use cases.