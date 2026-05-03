Automatically Rename Gmail Attachments with GPT-4o and Save to Google Drive

https://n8nworkflows.xyz/workflows/automatically-rename-gmail-attachments-with-gpt-4o-and-save-to-google-drive-7585


# Automatically Rename Gmail Attachments with GPT-4o and Save to Google Drive

### 1. Workflow Overview

This workflow is designed to automatically process unread Gmail messages that have attachments, rename those attachments intelligently using GPT-4o AI model, and save the renamed attachments to Google Drive. It also marks the processed emails as read to avoid duplicate processing.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Trigger & Email Retrieval:** Periodically triggers the process and fetches unread Gmail messages containing attachments.
- **1.2 Email Metadata Preparation:** Extracts and sets necessary metadata fields for further processing.
- **1.3 Attachment Download & Content Extraction:** Downloads attachments from emails and extracts text content from PDF files for AI analysis.
- **1.4 AI-Based Renaming:** Uses GPT-4o via an AI Agent to generate meaningful and expert file names based on extracted text and email metadata.
- **1.5 File Upload to Google Drive:** Uploads the renamed attachments to a designated Google Drive folder.
- **1.6 Email Post-Processing:** Marks processed emails as read to prevent reprocessing.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Email Retrieval

- **Overview:** Initiates the workflow at scheduled intervals; fetches unread Gmail messages with attachments.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Unread Messages with Attachments

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Periodically triggers the workflow every hour.  
    - Configuration: Interval set to trigger every 1 hour.  
    - Inputs: None (start node)  
    - Outputs: Triggers "Get Unread Messages with Attachments"  
    - Edge Cases: If the scheduler fails or is disabled, workflow won’t run. Timezone considerations might affect trigger timing.

  - **Get Unread Messages with Attachments**  
    - Type: Gmail node  
    - Role: Retrieves all unread emails from Gmail inbox that contain attachments.  
    - Configuration: Filter active for "readStatus" = "unread", operation set to "getAll".  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Passes message data to "Set Required Fields"  
    - Edge Cases: Gmail API rate limits, network errors, or no unread messages with attachments.

---

#### 1.2 Email Metadata Preparation

- **Overview:** Prepares key metadata fields from the fetched emails to be used in subsequent steps.  
- **Nodes Involved:**  
  - Set Required Fields

- **Node Details:**

  - **Set Required Fields**  
    - Type: Set node  
    - Role: Extracts and assigns important fields such as email ID, thread ID, sender's name, and date for streamlined use downstream.  
    - Configuration: Sets fields:
      - `id` from email JSON `id`
      - `threadId` from `threadId`
      - `from.value[0].name` from sender's name
      - `date` from email date  
    - Inputs: Output of "Get Unread Messages with Attachments"  
    - Outputs: Passes to "Download Attachments"  
    - Edge Cases: Missing or malformed email fields could cause issues; node assumes these fields exist.

---

#### 1.3 Attachment Download & Content Extraction

- **Overview:** Downloads attachments from emails and extracts text content from the first PDF attachment to provide context for AI file renaming.  
- **Nodes Involved:**  
  - Download Attachments  
  - Extract from Attachments

- **Node Details:**

  - **Download Attachments**  
    - Type: Gmail node  
    - Role: Downloads all attachments from the email specified by message ID.  
    - Configuration: Operation "get", with option to download attachments enabled. Message ID is dynamically pulled from previous node.  
    - Inputs: From "Set Required Fields"  
    - Outputs: Passes attachments to "Extract from Attachments"  
    - Edge Cases: Attachments could be missing or corrupted; large attachments might cause timeouts.

  - **Extract from Attachments**  
    - Type: Extract From File node  
    - Role: Extracts text from the first attachment (assumed to be a PDF) to provide content for AI analysis.  
    - Configuration: Operation "pdf", binary property set to "attachment_0" (first attachment).  
    - Inputs: From "Download Attachments"  
    - Outputs: 
      - One output to "AI Agent" containing extracted text and metadata.
      - Another output to "Merge Data for Upload" passing attachment binary data for upload.  
    - Edge Cases: If the attachment is not a PDF or is corrupted, extraction could fail or produce empty text.

---

#### 1.4 AI-Based Renaming

- **Overview:** Utilizes GPT-4o AI model to analyze extracted content and email metadata to generate a meaningful and context-aware file name for the attachment.  
- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model node  
    - Role: Provides GPT-4o-mini model access as a language model backend for AI Agent.  
    - Configuration: Model selected: "gpt-4o-mini". No additional options set.  
    - Inputs: Connected as language model input from AI Agent  
    - Outputs: N/A (used internally by AI Agent)  
    - Edge Cases: API quota limits, authentication errors, model availability.

  - **AI Agent**  
    - Type: Langchain Agent node  
    - Role: Coordinates the prompt to GPT-4o, passing a detailed instruction to rename files based on extracted text and email date.  
    - Configuration:  
      - Prompt text includes:  
        ```
        You are an expert content analyser and file naming assistant...
        text: {{ $json.text }}
        date: {{ $json.info.CreationDate }}
        ```  
      - Uses "define" prompt type.  
    - Inputs: Receives extracted text and metadata from "Extract from Attachments" (main output 0) and language model from "OpenAI Chat Model".  
    - Outputs: Passes AI-generated filename to "Merge Data for Upload"  
    - Edge Cases: Prompt expression errors, malformed input data, API failures, or unexpected AI output format.

---

#### 1.5 File Upload to Google Drive

- **Overview:** Combines AI-generated filename with attachment binary data and uploads the renamed file to Google Drive.  
- **Nodes Involved:**  
  - Merge Data for Upload  
  - Upload file

- **Node Details:**

  - **Merge Data for Upload**  
    - Type: Merge node  
    - Role: Combines two separate data streams by position: AI Agent’s filename output and attachment binary data for upload.  
    - Configuration: Mode "combine" by position.  
    - Inputs:  
      - Input 0: Attachment binary data from "Extract from Attachments" (output 1)  
      - Input 1: AI-generated filename from "AI Agent" (output 0)  
    - Outputs: Passes combined data to "Upload file"  
    - Edge Cases: Misaligned data positions could cause incorrect merging, empty inputs.

  - **Upload file**  
    - Type: Google Drive node  
    - Role: Uploads the attachment to Google Drive with the AI-generated name.  
    - Configuration:  
      - File name set dynamically from AI Agent output: `={{ $json.output }}`  
      - Drive set to "My Drive"  
      - Folder ID left empty (uploads to root or default folder)  
      - Input binary data field set to "attachment_1" (binary from merged data)  
    - Inputs: From "Merge Data for Upload"  
    - Outputs: Passes to "Mark message as read"  
    - Edge Cases: Google Drive API errors, permission issues, invalid folder ID, large file size limits.

---

#### 1.6 Email Post-Processing

- **Overview:** Marks processed Gmail messages as read to prevent repeat processing in future workflow runs.  
- **Nodes Involved:**  
  - Mark message as read

- **Node Details:**

  - **Mark message as read**  
    - Type: Gmail node  
    - Role: Marks the Gmail message as read using message ID to avoid duplicated processing.  
    - Configuration:  
      - Operation: "markAsRead"  
      - Message ID dynamically taken from "Set Required Fields" node’s JSON `id` field.  
    - Inputs: From "Upload file"  
    - Outputs: None (terminal node)  
    - Edge Cases: Gmail API errors, message ID missing or invalid, race conditions if email state changes externally.

---

### 3. Summary Table

| Node Name                     | Node Type                         | Functional Role                           | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                         |
|-------------------------------|----------------------------------|------------------------------------------|---------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger                 | Initiates workflow every hour             | None                            | Get Unread Messages with Attachments |                                                                                                   |
| Get Unread Messages with Attachments | Gmail                           | Retrieves unread emails with attachments  | Schedule Trigger                | Set Required Fields             |                                                                                                   |
| Set Required Fields           | Set                             | Extracts key metadata fields               | Get Unread Messages with Attachments | Download Attachments            |                                                                                                   |
| Download Attachments          | Gmail                           | Downloads attachments from email           | Set Required Fields             | Extract from Attachments         |                                                                                                   |
| Extract from Attachments      | Extract From File                | Extracts text from first PDF attachment    | Download Attachments            | AI Agent, Merge Data for Upload  |                                                                                                   |
| AI Agent                     | Langchain Agent                 | Generates AI-based file name                | Extract from Attachments, OpenAI Chat Model | Merge Data for Upload           |                                                                                                   |
| OpenAI Chat Model             | Langchain OpenAI Chat Model     | Provides GPT-4o-mini AI model               | AI Agent (as language model)    | AI Agent                       |                                                                                                   |
| Merge Data for Upload         | Merge                           | Combines AI filename and attachment binary | Extract from Attachments, AI Agent | Upload file                    |                                                                                                   |
| Upload file                  | Google Drive                    | Uploads renamed attachment to Drive        | Merge Data for Upload           | Mark message as read            |                                                                                                   |
| Mark message as read          | Gmail                           | Marks email as read after processing        | Upload file                    | None                          |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set interval to every 1 hour (or desired frequency).

2. **Add a Gmail node to get unread messages with attachments:**  
   - Operation: getAll  
   - Filters: readStatus = unread  
   - Connect Schedule Trigger output to this node's input.

3. **Add a Set node to extract required fields:**  
   - Assign fields:  
     - `id` = `{{$json["id"]}}`  
     - `threadId` = `{{$json["threadId"]}}`  
     - `from.value[0].name` = `{{$json["from"]["value"][0]["name"]}}`  
     - `date` = `{{$json["date"]}}`  
   - Connect Gmail unread messages node output to this Set node.

4. **Add a Gmail node to download attachments:**  
   - Operation: get  
   - Enable option to download attachments.  
   - Message ID: `={{$json["id"]}}` (from Set node)  
   - Connect Set node output to this node.

5. **Add Extract From File node:**  
   - Operation: pdf  
   - Binary Property Name: `attachment_0` (first attachment)  
   - Connect Download Attachments output to this node.

6. **Add Langchain OpenAI Chat Model node:**  
   - Select model: `gpt-4o-mini`  
   - No special options needed.  
   - This node serves as the language model backend.

7. **Add Langchain AI Agent node:**  
   - Set prompt type to "define"  
   - Enter prompt text:  
     ```
     You are an expert content analyser and file naming assistant...
     text: {{ $json.text }}
     date: {{ $json.info.CreationDate }}
     ```  
   - Connect OpenAI Chat Model as language model input to this node.  
   - Connect Extract from Attachments node output (main 0) as input to AI Agent.

8. **Add Merge node:**  
   - Mode: Combine  
   - Combine By: Position  
   - Connect:  
     - Input 0: Extract from Attachments output (main 1) — binary attachment data.  
     - Input 1: AI Agent output — AI-generated filename.

9. **Add Google Drive node to upload file:**  
   - Operation: upload  
   - File Name: `={{ $json.output }}` (AI-generated filename)  
   - Drive ID: "My Drive" (or specific drive)  
   - Folder ID: leave empty or specify folder URL if desired  
   - Input Data Field Name: `attachment_1` (binary data from merged input)  
   - Connect Merge node output to this node.

10. **Add Gmail node to mark message as read:**  
    - Operation: markAsRead  
    - Message ID: `={{ $('Set Required Fields').item.json.id }}` (from Set Required Fields node)  
    - Connect Google Drive upload output to this node.

11. **Save and activate the workflow.**

**Credentials Required:**  
- Gmail OAuth2 credentials with read/write access.  
- Google Drive OAuth2 credentials with file upload permissions.  
- OpenAI API key configured in Langchain OpenAI Chat Model node.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                   |
|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| GPT-4o-mini model is selected for advanced text understanding to generate expert file names.         | OpenAI model selection in Langchain OpenAI Chat Model node.      |
| Ensure Google Drive folder permissions allow uploads from the configured OAuth2 credentials.         | Google Drive API and OAuth2 configuration.                       |
| Gmail API quota limits and network issues can affect message retrieval and marking as read.          | Gmail API documentation for rate limits and error handling.      |
| Workflow assumes the first attachment is a PDF for text extraction; non-PDF attachments may fail.    | Extract From File node operation configured for PDF extraction.  |
| Marking emails as read avoids duplicate processing in subsequent runs.                                | Mark message as read node after successful upload.               |

---

*Disclaimer: The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.*