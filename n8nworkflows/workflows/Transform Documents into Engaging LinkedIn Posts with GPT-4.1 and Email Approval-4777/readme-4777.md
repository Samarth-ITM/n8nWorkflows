Transform Documents into Engaging LinkedIn Posts with GPT-4.1 and Email Approval

https://n8nworkflows.xyz/workflows/transform-documents-into-engaging-linkedin-posts-with-gpt-4-1-and-email-approval-4777


# Transform Documents into Engaging LinkedIn Posts with GPT-4.1 and Email Approval

### 1. Workflow Overview

This workflow automates the transformation of documents submitted via a form or received by email into engaging LinkedIn posts using GPT-4.1 AI processing, followed by an email approval step before publication. It targets professionals who want to easily convert various document formats into personalized, high-engagement LinkedIn content, leveraging AI to optimize messaging and style.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception and Document Extraction:** Receives documents either through a form submission or an email trigger, then detects file type and extracts text content accordingly.

- **1.2 AI Content Generation:** Sends extracted text to a GPT-4.1-powered AI agent specialized in creating personalized LinkedIn posts using a defined 4-step editorial model.

- **1.3 Approval Process via Email:** Sends the generated LinkedIn post draft via email to a reviewer, who can approve or reject it through a custom response form.

- **1.4 Conditional LinkedIn Posting:** If approved, publishes the post to LinkedIn with optional media.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Document Extraction

**Overview:**  
This block handles receiving documents either through a web form or an IMAP email trigger. It identifies the file type and extracts text content using appropriate methods.

**Nodes Involved:**  
- On form submission  
- Email Trigger (IMAP)  
- Switch  
- Extract from File (PDF)  
- HTTP Request2  
- Google Docs  
- Extract from File1 (Text extraction fallback)

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Initiates the workflow upon user form submission containing either a file upload or text input.  
  - Configuration: Form titled "Test" with two fields: a file upload ("fichier") and a text input ("Text").  
  - Inputs: Webhook receives form data.  
  - Outputs: Passes form data to the Switch node.  
  - Edge cases: Missing file or text input; form validation errors.

- **Email Trigger (IMAP)**  
  - Type: IMAP Email Read  
  - Role: Listens for new unread emails from "mammam-ia-v2@mail.beehiiv.com" to trigger the workflow.  
  - Configuration: Custom IMAP filter for unseen emails from specific sender.  
  - Inputs: Incoming emails.  
  - Outputs: Passes email content to AI Agent node directly (bypasses extraction as email content is presumably text).  
  - Edge cases: IMAP connectivity issues, email format inconsistencies.

- **Switch**  
  - Type: Switch (Decision)  
  - Role: Determines the type of uploaded file or presence of text input to route extraction accordingly.  
  - Configuration: Checks mimetype for "pdf", "document" (e.g. docx), or "text", or presence of plain text.  
  - Inputs: Form submission data.  
  - Outputs: Branches to Extract from File (PDF), HTTP Request2 (for Google Docs), Extract from File1 (text), or directly to AI Agent.  
  - Edge cases: Unsupported file types; empty or malformed input data.

- **Extract from File (PDF)**  
  - Type: File Extractor  
  - Role: Extracts text content from PDF binary uploaded via form.  
  - Configuration: Operation set to PDF extraction; uses binary property "fichier".  
  - Inputs: PDF file from Switch node.  
  - Outputs: Extracted text passed to AI Agent.  
  - Edge cases: Encrypted or scanned PDFs; corrupted files.

- **HTTP Request2**  
  - Type: HTTP Request  
  - Role: Requests content from a remote URL—specifically, it posts the file binary to an external service (URL provided) to retrieve Google Docs link or content.  
  - Configuration: POST method, contentType "binaryData", inputDataFieldName "fichier".  
  - Inputs: Document file from Switch node.  
  - Outputs: Passes response data (Google Docs URL) to Google Docs node.  
  - Edge cases: Network timeouts, service errors, invalid file data.

- **Google Docs**  
  - Type: Google Docs API  
  - Role: Fetches content from a Google Docs document using the provided URL.  
  - Configuration: Uses Google Docs OAuth2 credentials; operation "get" with document URL from previous node.  
  - Inputs: Google Docs URL from HTTP Request2.  
  - Outputs: Document content passed to AI Agent.  
  - Edge cases: OAuth token expiration, invalid document URL or permission issues.

- **Extract from File1 (Text extraction)**  
  - Type: File Extractor  
  - Role: Extracts text from text files uploaded via form.  
  - Configuration: Operation "text" on binary property "fichier".  
  - Inputs: Text file from Switch node.  
  - Outputs: Extracted text sent to AI Agent.  
  - Edge cases: Encoding issues, empty files.

---

#### 2.2 AI Content Generation

**Overview:**  
Processes the extracted document content with an AI agent using OpenAI GPT-4.1, applying a detailed 4-step LinkedIn content creation methodology to generate a personalized, engaging post.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model

**Node Details:**

- **AI Agent**  
  - Type: LangChain AI Agent  
  - Role: Generates LinkedIn post text from input document content.  
  - Configuration:  
    - Text prompt dynamically constructed from various possible input fields (e.g., Title, text content, data).  
    - System message instructs the AI to write in French, adopting a first-person voice ("Je"), using emoticons moderately to maximize engagement, and following a 4-step content strategy (defining game, ideal client, editorial line, addressing TPE).  
    - Output parsing enabled to format the response appropriately.  
  - Inputs: Extracted text from earlier nodes or direct email content.  
  - Outputs: Generated LinkedIn post passed to Send Email node.  
  - Credentials: Uses OpenAI Chat Model node as language model backend.  
  - Edge cases: OpenAI API rate limits, prompt formatting errors, unexpected input structure.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides GPT-4.1 language model interface with controlled temperature (0.7) for creativity.  
  - Configuration: Model set to "gpt-4.1-mini", temperature 0.7.  
  - Inputs: Queries from AI Agent node.  
  - Outputs: AI generated completion.  
  - Credentials: OpenAI API key required.  
  - Edge cases: API key invalid or expired; network errors.

---

#### 2.3 Approval Process via Email

**Overview:**  
Sends the AI-generated LinkedIn post draft in an email to a reviewer for validation, allowing them to approve or reject the content via a custom response form.

**Nodes Involved:**  
- Send Email  
- Switch1

**Node Details:**

- **Send Email**  
  - Type: Email Send with Response Form  
  - Role: Sends an email containing the LinkedIn post draft to a defined recipient ("yjoly@yjoly.fr") requesting validation.  
  - Configuration:  
    - Email subject: "LinkedIn ?"  
    - From: "yannick.joly@agroparistech.fr"  
    - Message body includes the AI-generated post content with a prompt "Tu valides ou pas ?"  
    - Waits for a custom form response with fields: file upload for Image, and a required text input "Tu valides (oui/non)".  
  - Inputs: AI Agent output (LinkedIn post).  
  - Outputs: Passes response data to Switch1 for decision.  
  - Credentials: SMTP account configured.  
  - Edge cases: SMTP connectivity issues, user non-response or invalid form data.

- **Switch1**  
  - Type: Switch (Decision)  
  - Role: Routes workflow based on reviewer’s approval response ("oui" or "non").  
  - Configuration: Checks response field "Tu valides (oui/non)" for "oui" or "non".  
  - Inputs: Email response data.  
  - Outputs:  
    - "oui" branch connects to LinkedIn posting node.  
    - "non" branch ends workflow (no output).  
  - Edge cases: Unexpected or missing response values.

---

#### 2.4 Conditional LinkedIn Posting

**Overview:**  
Posts the validated content to LinkedIn on behalf of a specified user, optionally including an image.

**Nodes Involved:**  
- LinkedIn

**Node Details:**

- **LinkedIn**  
  - Type: LinkedIn Node (Official API)  
  - Role: Publishes the approved LinkedIn post text and optional image to a specified LinkedIn user’s feed.  
  - Configuration:  
    - Text content dynamically taken from AI Agent output.  
    - Person ID specified ("9wNP5HmVRE").  
    - Visibility set to "PUBLIC".  
    - Binary property "Image" may be attached as media.  
  - Inputs: From Switch1 approval "oui" branch.  
  - Credentials: OAuth2 LinkedIn account connected.  
  - Edge cases: OAuth token expiration, LinkedIn API rate limits, media upload failures.

---

### 3. Summary Table

| Node Name           | Node Type                    | Functional Role                          | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                 |
|---------------------|------------------------------|----------------------------------------|-----------------------------|---------------------------|---------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger                 | Receives user document submission      | —                           | Switch                    |                                                                                             |
| Email Trigger (IMAP) | IMAP Email Read              | Triggers workflow on email receipt     | —                           | AI Agent                  |                                                                                             |
| Switch              | Switch                      | Determines file type or text input     | On form submission          | Extract from File, HTTP Request2, Extract from File1, AI Agent |                                                                                             |
| Extract from File    | Extract from File (PDF)      | Extracts text from PDF files            | Switch                      | AI Agent                  |                                                                                             |
| HTTP Request2       | HTTP Request                | Posts file to external service for Docs URL | Switch                  | Google Docs               |                                                                                             |
| Google Docs         | Google Docs API             | Retrieves Google Docs content           | HTTP Request2               | AI Agent                  |                                                                                             |
| Extract from File1   | Extract from File (Text)     | Extracts text from text files           | Switch                      | AI Agent                  |                                                                                             |
| AI Agent            | LangChain AI Agent          | Generates personalized LinkedIn post   | Extract nodes, Email Trigger (IMAP) | Send Email            |                                                                                             |
| OpenAI Chat Model   | LangChain OpenAI Chat Model | GPT-4.1 language model backend          | AI Agent                    | AI Agent                  |                                                                                             |
| Send Email          | Email Send with Response    | Sends post for approval and awaits reply | AI Agent                  | Switch1                   |                                                                                             |
| Switch1             | Switch                      | Routes based on approval response       | Send Email                  | LinkedIn (on approval)    |                                                                                             |
| LinkedIn            | LinkedIn API Node           | Publishes approved post to LinkedIn    | Switch1                     | —                         |                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node**  
   - Type: Form Trigger  
   - Configure form title "Test" with two fields:  
     - File upload field labeled "fichier"  
     - Text input field labeled "Text" with placeholder "Copier le texte ici"  
   - This node listens for form submissions.

2. **Create "Email Trigger (IMAP)" node**  
   - Type: Email Read (IMAP)  
   - Configure with IMAP credentials for the target mailbox.  
   - Set filter to unread emails from "mammam-ia-v2@mail.beehiiv.com".

3. **Create "Switch" node**  
   - Connect "On form submission" node output to "Switch" input.  
   - Add rules to detect mimetype of uploaded file or presence of text input:  
     - If mimetype contains "pdf" → output "pdf"  
     - If mimetype contains "document" → output "docx"  
     - If mimetype contains "text" → output "txt"  
     - If "Text" field is not empty → direct pass (no file).

4. **Create "Extract from File" node for PDF**  
   - Operation: PDF extraction  
   - Binary property: "fichier"  
   - Connect "pdf" output of Switch to this node.

5. **Create "HTTP Request2" node**  
   - Method: POST  
   - URL: "https://hook.integrator.boost.space/w71nefqvpxa60j1u58gd44nwbogpoo25"  
   - Content-Type: binaryData  
   - Input data field name: "fichier"  
   - Connect "docx" output of Switch to this node.

6. **Create "Google Docs" node**  
   - Operation: Get document content  
   - Document URL: set dynamically from HTTP Request2 response (`{{$json.data}}`)  
   - Connect output of HTTP Request2 to Google Docs node.

7. **Create "Extract from File1" node for text**  
   - Operation: Text extraction  
   - Binary property: "fichier"  
   - Connect "txt" output of Switch to this node.

8. **Create "AI Agent" node**  
   - Type: LangChain AI Agent  
   - Text prompt:  
     ```
     Voici un doc peux tu m'en faire un post linkedin : Son titre est {{ $json.info.Title }} son contenu est : {{ $json.text }}, {{ $json.data }}, {{ $json.content }}, {{ $json.Text }},{{ $json.text }}
     ```  
   - System message: Paste the detailed 4-step LinkedIn content creation instructions (in French), instructing the AI to use first-person, emoticons sparingly, and follow the editorial method.  
   - Enable output parser.  
   - Connect outputs from "Extract from File", "Google Docs", "Extract from File1" and "Email Trigger (IMAP)" nodes to AI Agent input.

9. **Create "OpenAI Chat Model" node**  
   - Model: "gpt-4.1-mini"  
   - Temperature: 0.7  
   - Connect AI Agent node’s AI language model input to this node.  
   - Configure OpenAI API credentials.

10. **Create "Send Email" node**  
    - Operation: sendAndWait (waits for form response)  
    - From: "yannick.joly@agroparistech.fr"  
    - To: "yjoly@yjoly.fr"  
    - Subject: "LinkedIn ?"  
    - Message body:  
      ```
      <p>Tu valides ou pas ?</p>
      
      {{ $json.output }}
      ```  
    - Form fields for response:  
      - File upload labeled "Image" (optional)  
      - Text input labeled "Tu valides (oui/non)" (required)  
    - Connect AI Agent output to Send Email input.  
    - Configure SMTP credentials.

11. **Create "Switch1" node**  
    - Connect Send Email output to Switch1 input.  
    - Add conditions:  
      - If "Tu valides (oui/non)" contains "oui" → route to LinkedIn node  
      - If contains "non" → end workflow (no output)

12. **Create "LinkedIn" node**  
    - Person: "9wNP5HmVRE" (LinkedIn user ID)  
    - Text: pulled from AI Agent output (`{{ $('AI Agent').item.json.output }}`)  
    - Visibility: PUBLIC  
    - Binary property: "Image" (optional)  
    - Connect "oui" output of Switch1 to LinkedIn node input.  
    - Configure LinkedIn OAuth2 credentials.

13. **Connect all nodes as per the above steps**, ensuring data flows from inputs → extraction → AI → approval → posting.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                          |
|------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
| The AI Agent uses a specialized 4-step content creation method to maximize LinkedIn post engagement, emphasizing first-person tone and moderate emoticon use. | Embedded in AI Agent system message.                                    |
| The workflow supports multiple document input types including PDFs, Google Docs (via external conversion service), and plain text files. | Multi-format support section.                                            |
| Email approval includes a custom form response with an optional image upload field for richer LinkedIn posts.                     | Send Email node configuration.                                           |
| LinkedIn posting requires OAuth2 credentials and a valid LinkedIn person ID configured in the node.                               | LinkedIn node configuration.                                             |
| The external HTTP service used to convert documents to Google Docs URL is critical; any downtime affects docx input processing.    | HTTP Request2 node dependency.                                           |
| IMAP email trigger filters for unseen emails from a specific sender to automate workflows from incoming messages.                  | Email Trigger (IMAP) node configuration.                                |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.