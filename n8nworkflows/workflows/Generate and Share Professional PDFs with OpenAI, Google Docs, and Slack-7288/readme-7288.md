Generate and Share Professional PDFs with OpenAI, Google Docs, and Slack

https://n8nworkflows.xyz/workflows/generate-and-share-professional-pdfs-with-openai--google-docs--and-slack-7288


# Generate and Share Professional PDFs with OpenAI, Google Docs, and Slack

### 1. Workflow Overview

This workflow automates the generation, formatting, archiving, and sharing of professional PDF documents using n8n, OpenAI, Google Docs, Google Drive, and Slack. It is designed for teams and individuals who need to create well-structured documents (e.g., reports, business requirements documents, proposals) in PDF format without relying on paid or external PDF libraries.

The workflow is segmented into the following logical blocks:

- **1.1 Input Reception**: Manual trigger to start the workflow.
- **1.2 AI Content Generation**: Uses OpenAI to generate a comprehensive Markdown document demonstrating rich formatting.
- **1.3 Google Drive Setup**: Defines the target Google Drive folder and timestamps for file naming.
- **1.4 Document Creation & Conversion**: Uploads the Markdown content as a Google Doc, then exports it as a PDF.
- **1.5 Archiving & Downloading**: Archives the generated PDF in a Google Drive folder and downloads it for sharing.
- **1.6 Distribution**: Sends the PDF file as a Slack message attachment to a specified channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block starts the workflow execution manually.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow on user command.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Triggers the next node "Generate sample markdown document".  
    - Potential Failures: None typical; workflow does not run if not triggered.

---

#### 1.2 AI Content Generation

- **Overview:**  
  Generates a rich Markdown document using OpenAI's GPT-4.1 model, featuring a wide variety of Markdown syntax examples.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Generate sample markdown document

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model node  
    - Role: Interface to the GPT-4.1 model for generating text.  
    - Configuration: Uses model "gpt-4.1" with default options, authenticated via stored OpenAI API credentials.  
    - Key Expressions: None beyond selecting the model.  
    - Inputs: Receives trigger from manual trigger node.  
    - Outputs: Passes generated text to "Generate sample markdown document".  
    - Failure Modes: API key expiration, rate limits, network errors.

  - **Generate sample markdown document**  
    - Type: LangChain Agent  
    - Role: Executes a prompt to produce a detailed Markdown document demonstrating all major Markdown syntax elements.  
    - Configuration:  
      - Prompt defines the Markdown content structure (headings, lists, code blocks, tables, footnotes, emojis, etc.).  
      - Output limited strictly to valid Markdown content.  
    - Inputs: Triggered by OpenAI Chat Model's output.  
    - Outputs: JSON containing Markdown content under `output` property.  
    - Failure Modes: Prompt execution errors, malformed output, API errors.

---

#### 1.3 Google Drive Setup

- **Overview:**  
  Assigns variables for the Google Drive folder ID for storage and a timestamp string for file naming.

- **Nodes Involved:**  
  - Configure Google Drive Folder

- **Node Details:**

  - **Configure Google Drive Folder**  
    - Type: Set Node  
    - Role: Defines constants for folder ID and current datetime string.  
    - Configuration:  
      - "Drive Folder ID" set to a fixed folder `"1IPcko8bzogO3W4mxhrW2Q017QA0Lc5MI"` (Google Drive folder for storage).  
      - "Today" set dynamically using the expression `$now.format("ddMMyyyyhhmmss")` to timestamp files.  
    - Inputs: Receives Markdown document JSON from previous node.  
    - Outputs: Passes variables alongside previous data to the next node.  
    - Failure Modes: None typical; incorrect folder ID would cause errors downstream.

---

#### 1.4 Document Creation & Conversion

- **Overview:**  
  Creates a Google Doc from the Markdown content and converts the Google Doc to a PDF format.

- **Nodes Involved:**  
  - Create document file  
  - Convert document to PDF

- **Node Details:**

  - **Create document file**  
    - Type: HTTP Request  
    - Role: Uploads Markdown content as a Google Docs document using the Google Drive API multipart upload.  
    - Configuration:  
      - URL: `https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart&supportsAllDrives=true`  
      - Method: POST  
      - Body: Multipart content with JSON metadata (name, mimeType, parent folder) and Markdown content as text/markdown.  
      - Content-Type: `multipart/related; boundary=foo_bar_baz`  
      - Credentials: Google Drive OAuth2 with permissions to upload files.  
      - Name: Uses dynamic timestamp from `$json.Today` for document naming.  
      - Folder: Uses dynamic folder ID from `$json['Drive Folder ID']`.  
    - Inputs: Receives Markdown content and Drive Folder ID.  
    - Outputs: Returns Google Doc file ID for conversion.  
    - Failure Modes: Auth errors, API quota exceeded, malformed multipart body.

  - **Convert document to PDF**  
    - Type: Google Drive node  
    - Role: Converts the uploaded Google Doc to PDF and downloads the file content.  
    - Configuration:  
      - Operation: Download with Google file conversion enabled, converting from Docs to PDF (`application/pdf`).  
      - Uses the file ID from the previous node.  
      - Credentials: Same Google Drive OAuth2.  
    - Inputs: Google Doc file ID from "Create document file".  
    - Outputs: Binary PDF file data for archiving.  
    - Failure Modes: Conversion failure, file not found, permission issues.

---

#### 1.5 Archiving & Downloading

- **Overview:**  
  Archives the generated PDF into a specified Google Drive folder and downloads the archived PDF to prepare for sharing.

- **Nodes Involved:**  
  - Archiving PDF File  
  - Download attachment file

- **Node Details:**

  - **Archiving PDF File**  
    - Type: Google Drive node  
    - Role: Uploads the PDF file to a Google Drive folder for archiving.  
    - Configuration:  
      - Operation: Upload file  
      - Folder ID: Fixed folder `"1IPcko8bzogO3W4mxhrW2Q017QA0Lc5MI"` (same as configuration node)  
      - File name: Dynamic, based on `$json.name` with `.pdf` extension.  
      - Credentials: Google Drive OAuth2.  
    - Inputs: Receives PDF binary data from "Convert document to PDF".  
    - Outputs: Returns metadata including file ID for download.  
    - Failure Modes: Auth failure, quota limits, invalid folder ID.

  - **Download attachment file**  
    - Type: Google Drive node  
    - Role: Downloads the archived PDF file binary for distribution.  
    - Configuration:  
      - Operation: Download by file ID from previous node output.  
      - Credentials: Google Drive OAuth2.  
    - Inputs: File ID from "Archiving PDF File".  
    - Outputs: Binary PDF data for Slack upload.  
    - Failure Modes: File not found, permission denied.

---

#### 1.6 Distribution

- **Overview:**  
  Sends the archived PDF as a Slack message attachment to a configured Slack channel.

- **Nodes Involved:**  
  - Send message

- **Node Details:**

  - **Send message**  
    - Type: Slack node  
    - Role: Uploads the PDF file to Slack channel with an initial comment.  
    - Configuration:  
      - Resource: File  
      - Channel ID: `C097VAKKPUP` (target Slack channel)  
      - File Name: Dynamic from `$json.name` (matches PDF file name)  
      - Initial Comment: "Formatted PDF file from markdown"  
      - Authentication: OAuth2 with Slack bot token having `files:write` scope.  
    - Inputs: Receives binary PDF file from "Download attachment file".  
    - Outputs: Slack API response for upload confirmation.  
    - Failure Modes: Auth failure, Slack API rate limits, invalid channel ID.

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                    | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                    |
|---------------------------|----------------------------------|----------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Starts workflow manually          |                             | Generate sample markdown document | **Trigger the Workflow**  Start the process manually or replace with another trigger (e.g., Cron or Webhook).  |
| OpenAI Chat Model          | LangChain OpenAI Chat Model       | Calls GPT-4.1 for content         | When clicking ‘Execute workflow’ | Generate sample markdown document | **Generate Markdown Content with AI**  Use OpenAI Chat Model to create structured Markdown document.          |
| Generate sample markdown document | LangChain Agent                 | Generates detailed Markdown text  | OpenAI Chat Model            | Configure Google Drive Folder |                                                                                                               |
| Configure Google Drive Folder | Set                             | Sets Drive folder ID & timestamp  | Generate sample markdown document | Create document file          |                                                                                                               |
| Create document file       | HTTP Request                     | Uploads Markdown as Google Doc    | Configure Google Drive Folder | Convert document to PDF       | **Create Google Doc from Markdown**  Upload markdown content to Google Drive and convert to Google Doc.        |
| Convert document to PDF    | Google Drive                     | Converts Google Doc to PDF        | Create document file          | Archiving PDF File            | **Export and Archive as PDF**  Convert Doc to PDF and upload to Google Drive folder for archiving.             |
| Archiving PDF File         | Google Drive                     | Archives PDF in Drive folder      | Convert document to PDF       | Download attachment file      |                                                                                                               |
| Download attachment file   | Google Drive                     | Downloads archived PDF            | Archiving PDF File            | Send message                 | **Share the PDF via Slack**  Download archived PDF and post it to Slack channel with message and links.        |
| Send message               | Slack                           | Sends PDF file as Slack message   | Download attachment file      |                             |                                                                                                               |
| Sticky Note                | Sticky Note                     | Documentation/Notes               |                             |                             | # Free PDF Generator in n8n – No External Libraries or Paid Services (detailed description with video link)    |
| Sticky Note1               | Sticky Note                     | Documentation/Notes               |                             |                             | **Trigger the Workflow**  Start the process manually or replace with another trigger                           |
| Sticky Note2               | Sticky Note                     | Documentation/Notes               |                             |                             | **Generate Markdown Content with AI**  Use OpenAI Chat Model to create structured Markdown document             |
| Sticky Note3               | Sticky Note                     | Documentation/Notes               |                             |                             | **Create Google Doc from Markdown**  Upload and convert markdown to Google Doc (with sample PDF link)           |
| Sticky Note4               | Sticky Note                     | Documentation/Notes               |                             |                             | **Export and Archive as PDF**  Convert Google Doc to PDF and archive in Drive                                  |
| Sticky Note5               | Sticky Note                     | Documentation/Notes               |                             |                             | **Share the PDF via Slack**  Download archived PDF and post it to Slack channel                                |
| Sticky Note7 to Sticky Note11 | Sticky Note (images)           | Documentation/Notes (PDF previews) |                             |                             | Images showing final PDF rendering examples                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Add an OpenAI Chat Model node**  
   - Type: LangChain OpenAI Chat Model  
   - Set model to `gpt-4.1`  
   - Authenticate with OpenAI credentials (API key).  
   - Connect input from Manual Trigger.

3. **Add a LangChain Agent node to generate Markdown**  
   - Type: LangChain Agent  
   - Paste the detailed Markdown prompt (as per the original prompt) to generate a comprehensive Markdown document demonstrating all major syntax.  
   - No system message configured.  
   - Connect input from OpenAI Chat Model output.

4. **Add a Set node to configure Google Drive folder and timestamp**  
   - Type: Set  
   - Add two fields:  
     - `Drive Folder ID` (string) = `"1IPcko8bzogO3W4mxhrW2Q017QA0Lc5MI"` (replace with your own folder ID)  
     - `Today` (string) = expression `$now.format("ddMMyyyyhhmmss")`  
   - Connect input from the LangChain Agent node.

5. **Add an HTTP Request node to create Google Doc**  
   - Type: HTTP Request  
   - URL: `https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart&supportsAllDrives=true`  
   - Method: POST  
   - Authentication: Set to predefined Google Drive OAuth2 credential.  
   - Headers: `Content-Type: multipart/related; boundary=foo_bar_baz`  
   - Body (raw): Construct multipart body:  
     ```
     --foo_bar_baz
     Content-Type: application/json; charset=UTF-8

     {
       "name": "{{ $json.Today }}",
       "mimeType": "application/vnd.google-apps.document",
       "parents": ["{{ $json['Drive Folder ID'] }}"]
     }

     --foo_bar_baz
     Content-Type: text/markdown; charset=UTF-8

     {{ $('Generate sample markdown document').item.json.output }}

     --foo_bar_baz--
     ```
   - Connect input from Set node.

6. **Add a Google Drive node to convert the Google Doc to PDF**  
   - Type: Google Drive  
   - Operation: Download  
   - Enable Google File Conversion: convert Docs to PDF (`application/pdf`)  
   - File ID: expression `={{ $json.id }}` from previous HTTP Request node output.  
   - Connect input from HTTP Request node.

7. **Add a Google Drive node to upload (archive) the PDF**  
   - Type: Google Drive  
   - Operation: Upload file  
   - Folder ID: `"1IPcko8bzogO3W4mxhrW2Q017QA0Lc5MI"` (same as before)  
   - File Name: expression `={{ $json.name }}.pdf`  
   - Connect input from Google Drive PDF conversion node.  
   - Credentials: Google Drive OAuth2.

8. **Add a Google Drive node to download the archived PDF file**  
   - Type: Google Drive  
   - Operation: Download  
   - File ID: expression `={{ $json.id }}` from archive upload node output.  
   - Connect input from archive upload node.

9. **Add a Slack node to send the PDF as a message**  
   - Type: Slack  
   - Resource: File  
   - Channel ID: `C097VAKKPUP` (replace with your Slack channel ID)  
   - File Name: expression `={{ $json.name }}`  
   - Initial Comment: `"Formatted PDF file from markdown"`  
   - Authentication: OAuth2 with Slack bot token (must have `files:write` permission).  
   - Connect input from Google Drive download node.

10. **Configure all credentials:**  
    - OpenAI API key for OpenAI Chat Model node.  
    - Google Drive OAuth2 credentials for all Google Drive nodes and HTTP Request node.  
    - Slack OAuth2 credentials for Slack node.

11. **Test the workflow:**  
    - Execute the manual trigger.  
    - Confirm the Markdown document is generated, converted, archived in Google Drive, and the PDF is sent to Slack.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| Free PDF Generator in n8n – No External Libraries or Paid Services: This workflow uses OpenAI to generate Markdown content, Google Docs to format and convert to PDF, and integrates with Google Drive and Slack for archiving and sharing. Ideal for reports, BRDs, proposals, or any document automation directly inside n8n. Watch the demo video embedded in the sticky note.                                                    | [YouTube Demo Video](https://www.youtube.com/watch?v=BB32RPQYI94)                                                                            |
| Sample PDF generated from comprehensive markdown is available for preview to understand the output format and styling.                                                                                                                                                                                                                                                                                            | [Sample PDF](https://wisestackai.s3.ap-southeast-1.amazonaws.com/12082025052957.pdf)                                                         |
| Slack bot token must have `files:write` permission and be invited to the target channel for successful PDF uploads.                                                                                                                                                                                                                                                                                                | Slack API documentation                                                                                                                      |
| Google Drive OAuth2 credentials require read/write access to the target folders for document creation, conversion, archiving, and downloading.                                                                                                                                                                                                                                                                       | Google Drive API documentation                                                                                                              |
| To customize, change the OpenAI prompt to generate different Markdown content types, replace the manual trigger with Cron/Webhook for automation, or extend Slack sharing to multiple channels or emails.                                                                                                                                                                                                             | Workflow design notes                                                                                                                        |
| Visual aids in sticky notes include screenshots of PDF outputs showing formatting quality and examples of the document content rendering, useful for quality verification.                                                                                                                                                                                                                                          | Image links within Sticky Notes nodes (e.g., https://wisestackai.s3.ap-southeast-1.amazonaws.com/pdf-1.png)                                   |

---

**Disclaimer:** The text provided above is exclusively derived from an automated workflow created with n8n, adhering strictly to content policies without illegal or protected elements. All manipulated data is legal and public.