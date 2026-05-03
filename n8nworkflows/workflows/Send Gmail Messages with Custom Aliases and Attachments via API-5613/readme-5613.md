Send Gmail Messages with Custom Aliases and Attachments via API

https://n8nworkflows.xyz/workflows/send-gmail-messages-with-custom-aliases-and-attachments-via-api-5613


# Send Gmail Messages with Custom Aliases and Attachments via API

### 1. Workflow Overview

This workflow enables sending Gmail messages through the Gmail API using custom verified alias email addresses and supports attachments. It is designed for scenarios where emails need to be sent programmatically with the possibility to include multiple file attachments, where attachment URLs are provided via an incoming webhook. The workflow handles attachment downloading, email payload formatting with MIME multipart message construction, and finally sends the email via Gmail API.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Validation:** Receiving the HTTP POST request with email details and attachment URLs.
- **1.2 Attachment Handling:** Conditional processing to detect, split, and download attachments from given URLs.
- **1.3 Email Construction:** Validating required fields, processing attachments, building MIME message, and encoding it for Gmail API.
- **1.4 Sending Email:** Making an authenticated HTTP request to Gmail API to send the constructed email message.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

- **Overview:**  
  This block listens for incoming webhook requests containing email details and optionally attachment URLs. It verifies the presence of attachments and routes the workflow accordingly.

- **Nodes Involved:**  
  - Webhook Trigger  
  - If Attachments

- **Node Details:**

  - **Webhook Trigger**  
    - Type: Webhook Trigger  
    - Role: Entry point to accept HTTP POST requests on path `/send-gmail-as-alias`.  
    - Configuration: HTTP POST, response mode set to output from last node executed.  
    - Inputs: External HTTP request with JSON body containing email fields and optional `body.file_urls` array.  
    - Outputs: Passes received data downstream.  
    - Failures: Misconfigured webhook path or method, request payload missing required fields (validated later).  
    - Notes: This is the start node for the entire workflow.

  - **If Attachments**  
    - Type: If (conditional check)  
    - Role: Determines if attachment URLs are present in the incoming payload under `body.file_urls`.  
    - Configuration: Checks if `body.file_urls` array is not empty (strict array notEmpty).  
    - Inputs: From Webhook Trigger.  
    - Outputs:  
      - True branch if attachments exist (proceeds to process attachments).  
      - False branch if no attachments (not used explicitly here; the workflow implicitly skips attachment processing).  
    - Failures: Expression evaluation errors if input structure is unexpected.  
    - Edge cases: Empty or missing `file_urls` leads to skipping attachment download.

#### 2.2 Attachment Handling

- **Overview:**  
  If attachments are present, this block splits the list of URLs into individual items and downloads each attachment.

- **Nodes Involved:**  
  - Split Out Attachments  
  - Download Attachments

- **Node Details:**

  - **Split Out Attachments**  
    - Type: Split Out  
    - Role: Splits the array of attachment URLs (`body.file_urls`) into separate workflow items for individual processing.  
    - Configuration: Field to split out: `body.file_urls`, includes all other fields in each output item.  
    - Inputs: From the "true" path of If Attachments node.  
    - Outputs: Each output item contains a single attachment URL for downloading.  
    - Failures: Fails if `body.file_urls` is not an array or missing.  
    - Edge cases: Handles empty arrays gracefully (no iteration).

  - **Download Attachments**  
    - Type: HTTP Request  
    - Role: Downloads the attachment file from the URL provided in each item.  
    - Configuration: URL parameter dynamically set from `{{$json['body.file_urls']}}` (each item contains a single URL after split). No authentication or special headers configured.  
    - Inputs: From Split Out Attachments node.  
    - Outputs: The binary data of each downloaded file is attached to the item.  
    - Failures: Network errors, 404 not found, or invalid URLs can cause download failure.  
    - Edge cases: Large files may cause timeouts or memory issues.

#### 2.3 Email Construction

- **Overview:**  
  This block validates the presence of required email fields, processes any attachments (including binary data), constructs a multipart MIME email message with attachments, encodes it for Gmail API, and outputs the encoded payload.

- **Nodes Involved:**  
  - Format Email Payload

- **Node Details:**

  - **Format Email Payload**  
    - Type: Code (JavaScript)  
    - Role: Implements field validation, attachment processing, MIME email construction, and Gmail API base64url encoding.  
    - Configuration:  
      - Checks presence of required fields `fromEmail`, `toEmail`, `subject`, and `htmlBody` in the webhook payload; throws error if any missing.  
      - Extracts optional fields (`senderName`, `replyTo`).  
      - Processes all binary data in input to identify attachments, falling back to alternative binary storage if first attempt yields none.  
      - Replaces `{{FILE_COUNT}}` placeholder in HTML body with the count of attachments.  
      - Creates a unique boundary string to separate MIME parts.  
      - Builds the full RFC2822 compliant MIME message with proper headers, body, and base64 encoded attachments.  
      - Encodes entire message in base64url format as required by Gmail API.  
      - Returns `encoded_message` for sending and detailed debug info for troubleshooting.  
    - Inputs: From Download Attachments node (or directly from If Attachments false branch if no attachments).  
    - Outputs: JSON object with `encoded_message` string and debug information.  
    - Failures:  
      - Missing required fields error thrown early.  
      - Base64 encoding errors if input data corrupted.  
      - Unexpected input structure causing script failure.  
    - Edge cases:  
      - No attachments scenario handled gracefully.  
      - Binary data with missing `fileName` or `mimeType` defaults applied (`.pdf` and `application/pdf`).  
    - Version-specific: Uses Node.js Buffer and n8n JavaScript environment capabilities.

#### 2.4 Sending Email

- **Overview:**  
  Sends the constructed email message to Gmail via Gmail API using OAuth2 credentials.

- **Nodes Involved:**  
  - Send Gmail as Alias

- **Node Details:**

  - **Send Gmail as Alias**  
    - Type: HTTP Request  
    - Role: Calls Gmail API endpoint `users/me/messages/send` with POST to send the email.  
    - Configuration:  
      - URL: `https://gmail.googleapis.com/gmail/v1/users/me/messages/send`  
      - Method: POST  
      - Authentication: Uses predefined Gmail OAuth2 credentials named `vendramin.work`.  
      - Body parameter: `{ raw: encoded_message }` where `encoded_message` comes from the previous node.  
      - Sends the body as JSON payload.  
      - Executes once per workflow run.  
    - Inputs: From Format Email Payload node.  
    - Outputs: API response with send status.  
    - Failures:  
      - OAuth token expiry or invalid credentials.  
      - Gmail API quota limits or permissions errors.  
      - Malformed message errors from Gmail API.  
    - Edge cases:  
      - Alias must be verified in Gmail otherwise sending fails.  
      - Large attachments may cause API request rejection.

---

### 3. Summary Table

| Node Name             | Node Type         | Functional Role                           | Input Node(s)           | Output Node(s)          | Sticky Note                                                |
|-----------------------|-------------------|-----------------------------------------|------------------------|-------------------------|------------------------------------------------------------|
| Webhook Trigger       | Webhook Trigger   | Receive HTTP POST with email data       | -                      | If Attachments           |                                                            |
| If Attachments        | If                | Check if attachment URLs present        | Webhook Trigger         | Split Out Attachments (true), none (false) |                                                            |
| Split Out Attachments | Split Out         | Split attachment URLs into individual items | If Attachments         | Download Attachments     |                                                            |
| Download Attachments  | HTTP Request      | Download each attachment file            | Split Out Attachments   | Format Email Payload     |                                                            |
| Format Email Payload  | Code (JavaScript) | Validate fields, build MIME message, encode | Download Attachments or If Attachments (if no files) | Send Gmail as Alias     | See code comments for detailed debug logging and attachment handling |
| Send Gmail as Alias   | HTTP Request      | Send email via Gmail API                  | Format Email Payload    | -                       | Requires Gmail OAuth2 credential named "vendramin.work"    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Trigger node:**  
   - Name: `Webhook Trigger`  
   - Path: `send-gmail-as-alias`  
   - Method: `POST`  
   - Response Mode: `Last Node`  
   - This node will receive the email data and attachment URLs.

2. **Create an If node:**  
   - Name: `If Attachments`  
   - Condition: Check if `body.file_urls` is a non-empty array.  
   - Expression: `{{$json.body.file_urls}}` with operator `array notEmpty` (strict).  
   - Connect input from `Webhook Trigger`.

3. **Create a Split Out node:**  
   - Name: `Split Out Attachments`  
   - Field to split out: `body.file_urls`  
   - Include all other fields in outputs.  
   - Connect true output of `If Attachments` to this node.

4. **Create HTTP Request node to download files:**  
   - Name: `Download Attachments`  
   - Method: GET (default)  
   - URL: Expression set to current item URL: `{{$json['body.file_urls']}}`  
   - Connect input from `Split Out Attachments`.

5. **Create a Code node for email construction:**  
   - Name: `Format Email Payload`  
   - Language: JavaScript  
   - Paste the provided JavaScript code that:  
     - Validates required fields (`fromEmail`, `toEmail`, `subject`, `htmlBody`).  
     - Processes attachments from binary data.  
     - Constructs MIME multipart email with attachments.  
     - Encodes message in base64url for Gmail API.  
     - Returns `encoded_message` and debug info.  
   - Connect input from `Download Attachments` node.  
   - For the false output of `If Attachments` (no attachments), connect directly here as well to handle emails without attachments.

6. **Create HTTP Request node to send email:**  
   - Name: `Send Gmail as Alias`  
   - Method: POST  
   - URL: `https://gmail.googleapis.com/gmail/v1/users/me/messages/send`  
   - Authentication: Use Gmail OAuth2 credentials (create and configure in n8n credentials section) with name `vendramin.work` or your own.  
   - Body Parameters: JSON with key `raw` set to expression `{{$json.encoded_message}}`.  
   - Set "Send Body" to true.  
   - Connect input from `Format Email Payload`.

7. **Credential Setup:**  
   - Create Gmail OAuth2 credential in n8n.  
   - Ensure the associated Gmail account has the alias verified for sending.  
   - Assign this credential to the `Send Gmail as Alias` node.

8. **Connect the flow:**  
   - Webhook Trigger → If Attachments  
   - If Attachments true → Split Out Attachments → Download Attachments → Format Email Payload → Send Gmail as Alias  
   - If Attachments false → Format Email Payload → Send Gmail as Alias

9. **Test:**  
   - Send a POST request to the webhook URL with a JSON body containing required fields:  
     - `fromEmail` (alias email)  
     - `toEmail`  
     - `subject`  
     - `htmlBody` (can contain `{{FILE_COUNT}}` placeholder)  
     - Optional `senderName` and `replyTo`  
     - Optional `body.file_urls` array for attachments URLs.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                                         |
|------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| The workflow uses Gmail API's `messages.send` endpoint which requires base64url encoded RFC2822 compliant MIME messages. | Gmail API reference: https://developers.google.com/gmail/api/v1/reference/users/messages/send                            |
| Attachments are downloaded via URLs, so ensure URLs are publicly accessible or accessible with no authentication needed. | HTTPS URLs only; if authentication needed, adapt the Download Attachments node accordingly.                              |
| The `Format Email Payload` node includes extensive debug logging in JavaScript for troubleshooting email sending issues. | Helps identify missing fields, attachment counts, and binary data presence.                                             |
| Alias email addresses must be verified in Gmail settings to be used in the `fromEmail` field.                         | Gmail alias verification guide: https://support.google.com/mail/answer/22370                                          |
| The workflow assumes that attachments are PDF by default if MIME type or filename is not specified.                    | Modify the code if other file types are expected frequently.                                                            |

---

Disclaimer: The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.