Send a message with an inline embedded image with Gmail

https://n8nworkflows.xyz/workflows/send-a-message-with-an-inline-embedded-image-with-gmail-2280


# Send a message with an inline embedded image with Gmail

### 1. Workflow Overview

This workflow demonstrates how to send an email via Gmail with an inline embedded image using n8n. Since the native Gmail node does not support embedding images directly in the email body, the workflow uses the HTTP Request node to interact with the Gmail API directly. It achieves this by downloading an image from a URL, converting it to a base64-encoded string, constructing a multipart MIME email with embedded image references, and sending it using Gmail’s API.

**Target Use Case:**  
Send richly formatted HTML emails that contain images embedded inline (not as attachments), useful for newsletters, marketing emails, or any scenario where inline images improve the message presentation.

**Logical Blocks:**

- **1.1 Input Setup:** Manual trigger and message parameter configuration  
- **1.2 Image Acquisition and Encoding:** Download image and convert to base64  
- **1.3 MIME Email Construction:** Build the multipart MIME message with embedded image reference  
- **1.4 Email Sending:** Send the constructed message to Gmail API using HTTP node

---

### 2. Block-by-Block Analysis

#### 1.1 Input Setup

- **Overview:**  
  This block handles the initiation of the workflow and sets the email metadata such as sender, recipient, subject, and HTML body content including the placeholder for the embedded image.

- **Nodes Involved:**  
  - When clicking "Test workflow"  
  - Message settings  
  - Sticky Note3 (instructional note)

- **Node Details:**

  1. **When clicking "Test workflow"**  
     - Type: Manual Trigger  
     - Role: Entry point to start workflow execution manually  
     - Configuration: No parameters  
     - Inputs: None  
     - Outputs: Connects to "Message settings"  
     - Edge Cases: None (manual initiation)  

  2. **Message settings**  
     - Type: Set  
     - Role: Defines email parameters (from, to, subject, body_html) as JSON properties  
     - Configuration:  
       - from: sender@example.com (change to actual sender email)  
       - to: recipient@example.com (change to actual recipient email)  
       - subject: "Email with embedded image"  
       - body_html: HTML including `<img src='cid:image1'>` placeholder for inline image  
     - Inputs: From manual trigger  
     - Outputs: Connects to "Get image" node  
     - Edge Cases:  
       - Incorrect email addresses will cause Gmail API to reject the message  
       - Invalid HTML could affect email rendering  

  3. **Sticky Note3**  
     - Type: Sticky Note  
     - Role: Guidance note reminding the user to configure Gmail credentials, sender, recipient, and trigger the workflow manually  
     - Inputs/Outputs: None  

---

#### 1.2 Image Acquisition and Encoding

- **Overview:**  
  Downloads the specified image from the internet and converts the binary image data into a base64 string to embed it in the email.

- **Nodes Involved:**  
  - Get image  
  - Convert image to base64  
  - Sticky Note (instructional note about image source)

- **Node Details:**

  1. **Get image**  
     - Type: HTTP Request  
     - Role: Fetches image binary data from a fixed URL  
     - Configuration:  
       - URL: https://thistleandrose.co.uk/img/userimages/Page/0/bgmainfront.jpg (replaceable with any image URL)  
       - Method: GET (default)  
       - Response: Binary data ("data" property expected)  
     - Inputs: From "Message settings"  
     - Outputs: Connects to "Convert image to base64"  
     - Edge Cases:  
       - URL unreachable or returns error -> workflow failure  
       - Large image sizes may cause processing delays or memory issues  

  2. **Convert image to base64**  
     - Type: Extract From File  
     - Role: Converts binary image data to a base64-encoded string stored in JSON property `chart1`  
     - Configuration:  
       - Operation: binaryToProperty  
       - Destination Key: chart1  
       - Binary Property: defaults to node input binary named "data"  
     - Inputs: From "Get image"  
     - Outputs: Connects to "Compose message"  
     - Edge Cases:  
       - Missing or corrupt binary data causes failure  
       - Misnamed binary property input causes failure  

  3. **Sticky Note**  
     - Type: Sticky Note  
     - Role: Reminder that the image URL can be replaced and that the binary property should be named "data"  
     - Inputs/Outputs: None  

---

#### 1.3 MIME Email Construction

- **Overview:**  
  Constructs a multipart MIME email message string embedding the base64 image inline. The email headers and body are dynamically composed using expression syntax referencing previous node outputs.

- **Nodes Involved:**  
  - Compose message

- **Node Details:**

  1. **Compose message**  
     - Type: Set  
     - Role: Builds the raw email string with multipart/related content type and boundary, including headers (From, To, Subject), HTML body, and base64-encoded image as inline attachment with Content-ID "image1"  
     - Configuration:  
       - Sets a single property `raw` containing the full MIME message as a string, constructed with n8n expression syntax referencing:  
         - Email parameters from "Message settings" (from, to, subject, body_html)  
         - Image mimeType from "Get image" node binary metadata  
         - Base64 image string from "Convert image to base64" (`$json.chart1`)  
       - Boundary string fixed as "boundary1"  
       - Email body references image as `<img src='cid:image1'>` matching Content-ID of embedded image  
     - Inputs: From "Convert image to base64"  
     - Outputs: Connects to "Send message"  
     - Edge Cases:  
       - Expression failures if referenced nodes are missing or outputs are malformed  
       - Incorrect MIME formatting can cause Gmail API to reject message  
       - Boundary string conflicts could corrupt message parsing  

---

#### 1.4 Email Sending

- **Overview:**  
  Sends the constructed raw email message to the Gmail API endpoint using an HTTP POST request authenticated with OAuth2 credentials.

- **Nodes Involved:**  
  - Send message  
  - Sticky Note1 (instructional note about credentials)

- **Node Details:**

  1. **Send message**  
     - Type: HTTP Request  
     - Role: Sends the email via Gmail API endpoint `/gmail/v1/users/me/messages/send`  
     - Configuration:  
       - URL: https://www.googleapis.com/gmail/v1/users/me/messages/send  
       - Method: POST  
       - Body: JSON with property `raw` containing base64-encoded email MIME message (encoded using `$json.raw.base64Encode()`)  
       - Content-Type: application/json (raw)  
       - Authentication: OAuth2 via predefined Gmail credential  
       - Credential: "Gmail account (David)" (OAuth2 credential configured in n8n)  
     - Inputs: From "Compose message"  
     - Outputs: None (end of workflow)  
     - Edge Cases:  
       - OAuth token expired or invalid -> authentication error  
       - API quota exceeded or Gmail API errors  
       - Malformed raw message -> API rejects email sending  
       - Network errors or timeouts  

  2. **Sticky Note1**  
     - Type: Sticky Note  
     - Role: Reminder to add Gmail credentials in this HTTP Request node for authentication  
     - Inputs/Outputs: None  

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                                   | Input Node(s)           | Output Node(s)         | Sticky Note                                                      |
|----------------------------|--------------------|-------------------------------------------------|-------------------------|------------------------|-----------------------------------------------------------------|
| When clicking "Test workflow" | Manual Trigger     | Starts the workflow manually                      | —                       | Message settings       |                                                                 |
| Message settings           | Set                | Defines email metadata and HTML body              | When clicking "Test workflow" | Get image           | To use the image in the body of the email, insert <img src='cid:image1'> |
| Get image                  | HTTP Request       | Downloads image binary data from specified URL    | Message settings         | Convert image to base64 | Gets a random image from the internet. Replace this with your image (should be called 'data') |
| Convert image to base64    | Extract From File  | Converts binary image data to base64 string        | Get image                | Compose message        |                                                                 |
| Compose message            | Set                | Builds multipart MIME email string with inline image | Convert image to base64   | Send message           |                                                                 |
| Send message               | HTTP Request       | Sends constructed email via Gmail API with OAuth2 | Compose message          | —                      | We use an HTTP node rather than the Gmail node. Add your Gmail creds here |
| Sticky Note3               | Sticky Note        | Instructions for usage                             | —                       | —                      | ## Try me out 1. Make sure you add your Gmail credential in the last node 2. Update the sender and recipient in the 'Message settings' node 3. Click 'test workflow' |
| Sticky Note                | Sticky Note        | Explains the image source and binary property     | —                       | —                      | Gets a random image from the internet. Replace this with your image (should be called 'data') |
| Sticky Note1               | Sticky Note        | Reminder to set Gmail credentials                  | —                       | —                      | We use an HTTP node rather than the Gmail node. Add your Gmail creds here |
| Sticky Note2               | Sticky Note        | Explains how to embed image in email body         | —                       | —                      | To use the image in the body of the email, insert <img src='cid:image1'> |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters needed  
   - Position: Start of workflow  

2. **Create Set Node "Message settings"**  
   - Connect input from Manual Trigger  
   - Add properties:  
     - `from` (string): sender@example.com (replace with actual sender)  
     - `to` (string): recipient@example.com (replace with actual recipient)  
     - `subject` (string): Email with embedded image  
     - `body_html` (string): `<p>This email contains an embedded image:</p><p><img src='cid:image1'></p>`  
   - Position: After manual trigger  

3. **Create HTTP Request Node "Get image"**  
   - Connect input from "Message settings"  
   - Method: GET  
   - URL: https://thistleandrose.co.uk/img/userimages/Page/0/bgmainfront.jpg (replace with your image URL if needed)  
   - Response Format: Binary (ensure response is stored in binary data named "data")  
   - Position: After "Message settings"  

4. **Create Extract From File Node "Convert image to base64"**  
   - Connect input from "Get image"  
   - Operation: binaryToProperty  
   - Destination Key: chart1  
   - Input Binary Property: data (default)  
   - Position: After "Get image"  

5. **Create Set Node "Compose message"**  
   - Connect input from "Convert image to base64"  
   - Add string property `raw` with value constructed as:  

```
From: {{$node["Message settings"].json.from}}
To: {{$node["Message settings"].json.to}}
Subject: {{$node["Message settings"].json.subject}}
MIME-Version: 1.0
Content-Type: multipart/related; boundary=boundary1

--boundary1
Content-Type: text/html; charset=UTF-8

<html>
<body>
{{$node["Message settings"].json.body_html}}
</body>
</html>

--boundary1
Content-Type: {{$node["Get image"].item.binary.data.mimeType}}
Content-Transfer-Encoding: base64
Content-Disposition: inline
Content-ID: <image1>

{{$json.chart1}}

--boundary1--
```

   - Position: After "Convert image to base64"  

6. **Create HTTP Request Node "Send message"**  
   - Connect input from "Compose message"  
   - URL: https://www.googleapis.com/gmail/v1/users/me/messages/send  
   - Method: POST  
   - Content-Type: application/json (raw)  
   - Body:  
     ```json
     {
       "raw": "{{$json.raw.base64Encode()}}"
     }
     ```  
   - Authentication: OAuth2 with Gmail credentials (set up Gmail OAuth2 credentials before)  
   - Position: After "Compose message"  

7. **Configure Gmail OAuth2 Credentials**  
   - In n8n Credentials, add Gmail OAuth2 credentials with proper client ID, secret, and scopes to allow sending emails  

8. **Optional: Add Sticky Notes for Instructions**  
   - Add notes to remind user to replace image URL, configure emails, and add credentials  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                            |
|-----------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| The native Gmail node does not support inline images in email body yet, so HTTP node is used. | Explanation of why HTTP node is necessary                                                  |
| Inline images require multipart MIME emails with Content-ID references in HTML body.           | General email MIME formatting best practices                                              |
| Gmail API requires base64url-encoded raw email message in JSON payload.                        | Gmail API documentation: https://developers.google.com/gmail/api/v1/reference/users/messages/send |
| Change the image URL in "Get image" node to use your own image if needed.                      | Workflow flexibility for different image sources                                           |
| Make sure Gmail OAuth2 credentials have the scope: `https://www.googleapis.com/auth/gmail.send`| Gmail API OAuth2 scopes documentation                                                     |

---

This reference document enables full understanding, reproduction, and modification of the workflow, while anticipating key edge cases and integration constraints.