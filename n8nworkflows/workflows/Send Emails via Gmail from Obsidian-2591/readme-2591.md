Send Emails via Gmail from Obsidian

https://n8nworkflows.xyz/workflows/send-emails-via-gmail-from-obsidian-2591


# Send Emails via Gmail from Obsidian

### 1. Workflow Overview

This workflow enables sending emails directly from Obsidian notes using n8n automation. It integrates with the Obsidian Post Webhook plugin to trigger email dispatches based on note content enriched with YAML frontmatter metadata. The workflow supports both plain emails and those with attachments, handling base64 encoding and conversion automatically. It also includes a testing mechanism to validate setup before sending real emails.

The workflow logic is organized into the following functional blocks:

- **1.1 Input Reception**: Receives HTTP POST requests from Obsidian containing note content, metadata, and attachments.
- **1.2 Attachment Processing**: Checks for attachments, processes and converts them to binary files suitable for email.
- **1.3 Email Dispatch**: Sends emails either with or without attachments, based on input data.
- **1.4 Response Handling**: Confirms email sending status and returns a response to Obsidian.
- **1.5 Testing Logic**: Allows for testing the webhook integration without sending actual emails.
- **1.6 Informational Sticky Notes**: Provide documentation and instructions throughout the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for incoming POST webhook requests from Obsidian notes, initiating the workflow with note content and metadata.

**Nodes Involved:**  
- Webhook

**Node Details:**

- **Webhook**  
  - Type: Webhook (Trigger)  
  - Configuration:  
    - HTTP Method: POST  
    - Path: Unique webhook path (`e634d721-48b0-4985-8a57-62ca4c7b3cfb`)  
    - Allowed Origins: `*` (accepts all origins)  
    - Response Mode: Response node (delays response until explicitly sent)  
  - Inputs: External HTTP POST from Obsidian Post Webhook plugin  
  - Outputs: Passes the JSON body containing note text, YAML frontmatter fields, and attachments (if any)  
  - Potential Failures: Invalid webhook URL or payload format; network issues; unauthorized calls if origin restrictions are added later  
  - Version: v2 (n8n node version 2)  

---

#### 2.2 Attachment Processing

**Overview:**  
Determines if attachments exist in the note. If yes, splits attachments for individual processing, fixes base64 encoding, converts to binary, aggregates them, and prepares for email sending.

**Nodes Involved:**  
- Check if attachments exist (IF)  
- Separate attachment data (Split Out)  
- Fix Base64 string (Set)  
- Process Each Attachment (Split In Batches)  
- Convert Attachment to File (Convert to File)  
- Prepare Attachments for Email (Aggregate)

**Node Details:**

- **Check if attachments exist**  
  - Type: IF  
  - Logic: Checks if `body.attachments` is a non-empty array  
  - Outputs:  
    - True: Proceed with attachment processing  
    - False: Skip to test or no-attachment email sending  
  - Potential Failures: Missing or malformed `attachments` field; expression evaluation errors  

- **Separate attachment data**  
  - Type: Split Out  
  - Logic: Splits `body.attachments` array into individual items for processing  
  - Inputs: True output from IF node  
  - Outputs: Single attachment data per item  

- **Fix Base64 string**  
  - Type: Set  
  - Logic: Removes data URI prefix from base64 strings in attachment data for email compatibility  
  - Expression: Uses regex to strip prefix from `$json.data` (e.g., `data:image/png;base64,` removed)  
  - Inputs: Each attachment item from Split Out  
  - Outputs: Cleaned base64 string in `data` field  

- **Process Each Attachment**  
  - Type: Split In Batches  
  - Logic: Processes attachments one by one to avoid memory overload  
  - Inputs: Fixed base64 strings  
  - Outputs: Each attachment processed sequentially, with two parallel paths downstream  

- **Convert Attachment to File**  
  - Type: Convert To File  
  - Logic: Converts base64 string to binary file, setting filename from attachment name  
  - Inputs: One attachment data from Process Each Attachment  
  - Outputs: Binary data suitable for email attachment nodes  
  - Potential Issues: Invalid base64 data causing conversion failure  

- **Prepare Attachments for Email**  
  - Type: Aggregate  
  - Logic: Collects all binary attachments back into a single item including their binaries, preparing for Gmail node  
  - Inputs: Process Each Attachment outputs (one path)  
  - Outputs: Aggregated binary attachments for sending  

---

#### 2.3 Email Dispatch

**Overview:**  
Sends the email either with attachments or without, depending on the presence of attachments. Email metadata and addresses are dynamically populated from YAML frontmatter.

**Nodes Involved:**  
- Email With Attachments (Gmail)  
- Email Without Attachments (Gmail)

**Node Details:**

- **Email With Attachments**  
  - Type: Gmail node (send email)  
  - Configuration:  
    - To: Dynamic list from `Webhook` JSON `body.to` (joins array with semicolons if multiple)  
    - Subject: From `body.subject`  
    - Message: `body.content` (note content excluding frontmatter)  
    - CC, BCC, Reply-To, Sender Name: Optional, from respective YAML fields if present  
    - Attachments: Uses all binary data aggregated in previous block  
    - Email Type: Text (plain text email)  
    - Credentials: Gmail OAuth2 (preconfigured in n8n)  
  - Inputs: Aggregated attachments and webhook data  
  - Outputs: Success triggers date capture and response nodes  
  - Potential Failures: Authentication errors, invalid email addresses, attachment size limits, Gmail API rate limits  

- **Email Without Attachments**  
  - Type: Gmail node (send email)  
  - Configuration: Same as above but no attachments included  
  - Inputs: Webhook data when no attachments present and not a test  
  - Outputs: Success triggers date capture and response nodes  
  - Potential Failures: Same as Email With Attachments except attachment-related errors  

---

#### 2.4 Response Handling

**Overview:**  
After sending the email, this block records the current date/time and sends a confirmation response back to Obsidian via the webhook response.

**Nodes Involved:**  
- Get date (DateTime)  
- Respond to Obsidian (Respond to Webhook)

**Node Details:**

- **Get date**  
  - Type: DateTime  
  - Logic: Captures current date/time for timestamping response  
  - Inputs: Output from Gmail nodes (email sent)  
  - Outputs: Passes date/time info downstream  

- **Respond to Obsidian**  
  - Type: Respond to Webhook  
  - Logic: Returns a human-readable confirmation message including the formatted sending date/time  
  - Response Body: Text formatted with day/month/year and 24-hour time, e.g. "E-mail sent on 25 June 2024 14h30"  
  - Inputs: From Get date node  
  - Outputs: HTTP response to webhook caller (Obsidian)  
  - Potential Failures: Delayed response timeout, broken date formatting expression  

---

#### 2.5 Testing Logic

**Overview:**  
Enables a test mode by checking for a `test` flag in incoming webhook requests to verify the webhook and workflow setup without sending real emails.

**Nodes Involved:**  
- Check if it is a test (IF)  
- Test Successful (Respond to Webhook)  
- Email Without Attachments (conditional path if not test)

**Node Details:**

- **Check if it is a test**  
  - Type: IF  
  - Logic: Checks if `body.test` boolean is true  
  - Outputs:  
    - True: Sends back a "Test successful" message  
    - False: Proceeds with email sending logic  
  - Potential Failures: Missing or malformed `test` field causing false negatives  

- **Test Successful**  
  - Type: Respond to Webhook  
  - Logic: Sends plain text response "Test successful" to confirm webhook reception and workflow trigger  
  - Inputs: True output from test IF node  
  - Outputs: Ends workflow with success message  

---

#### 2.6 Informational Sticky Notes

**Overview:**  
Multiple sticky notes provide in-workflow documentation, instructions, sample YAML, and branding.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5

**Node Details:**

- **Sticky Note**  
  - Content: Overview of the workflow purpose and key features, including link to the Obsidian Post Webhook plugin  
- **Sticky Note1**  
  - Content: YAML frontmatter example showing fields like to, cc, bcc, subject, sender-name, and send-replies-to  
- **Sticky Note2**  
  - Content: Setup instructions for installing and using the Obsidian Post Webhook plugin, including how to trigger sending from Obsidian  
- **Sticky Note3**  
  - Content: Details about automatic attachment handling and conversion to binary files  
- **Sticky Note4**  
  - Content: Obsidian logo image for branding  
- **Sticky Note5**  
  - Content: Explains the email sending and response confirmation process  

---

### 3. Summary Table

| Node Name                 | Node Type         | Functional Role                   | Input Node(s)                 | Output Node(s)                          | Sticky Note                                                                                                    |
|---------------------------|-------------------|---------------------------------|------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Webhook                   | Webhook           | Receives POST from Obsidian     | None                         | Check if attachments exist             |                                                                                                               |
| Check if attachments exist| IF                | Tests for attachments presence  | Webhook                      | Separate attachment data, Check if it is a test |                                                                                                               |
| Separate attachment data  | Split Out         | Splits attachments array        | Check if attachments exist   | Fix Base64 string                      |                                                                                                               |
| Fix Base64 string         | Set               | Cleans base64 strings           | Separate attachment data     | Process Each Attachment                |                                                                                                               |
| Process Each Attachment   | Split In Batches  | Processes attachments one by one| Fix Base64 string            | Prepare Attachments for Email, Convert Attachment to File |                                                                                                               |
| Convert Attachment to File| Convert To File   | Converts base64 to binary       | Process Each Attachment      | Process Each Attachment                |                                                                                                               |
| Prepare Attachments for Email | Aggregate     | Aggregates binaries for Gmail   | Process Each Attachment      | Email With Attachments                 |                                                                                                               |
| Email With Attachments    | Gmail             | Sends email with attachments    | Prepare Attachments for Email| Get date                              | After the email is sent, the workflow confirms the email's status and sends a response back to Obsidian.       |
| Email Without Attachments | Gmail             | Sends email without attachments | Check if it is a test        | Get date                              |                                                                                                               |
| Check if it is a test     | IF                | Detects test mode requests      | Check if attachments exist   | Test Successful, Email Without Attachments |                                                                                                               |
| Test Successful           | Respond to Webhook| Sends test confirmation         | Check if it is a test        | None                                  |                                                                                                               |
| Get date                  | DateTime          | Captures current date-time      | Email With Attachments, Email Without Attachments | Respond to Obsidian                    |                                                                                                               |
| Respond to Obsidian       | Respond to Webhook| Sends confirmation response     | Get date                    | None                                  |                                                                                                               |
| Sticky Note               | Sticky Note       | Documentation overview          | None                        | None                                  | ## Obsidian to Email Overview; includes link to Obsidian Post Webhook plugin                                 |
| Sticky Note1              | Sticky Note       | YAML frontmatter example        | None                        | None                                  | ## YAML Frontmatter Example                                                                                   |
| Sticky Note2              | Sticky Note       | Obsidian plugin setup instructions| None                      | None                                  | ## Obsidian Configuration; includes link to Obsidian Post Webhook plugin                                     |
| Sticky Note3              | Sticky Note       | Attachment handling explanation | None                        | None                                  | ## Attachment Handling                                                                                        |
| Sticky Note4              | Sticky Note       | Branding (logo image)            | None                        | None                                  | Displays Obsidian logo                                                                                        |
| Sticky Note5              | Sticky Note       | Email sending and response explanation| None                   | None                                  | ## Send Email and Respond                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., `e634d721-48b0-4985-8a57-62ca4c7b3cfb`)  
   - Allowed Origins: `*`  
   - Response Mode: Response node  
   - Purpose: Accept POST requests from Obsidian notes with email data  
   
2. **Add IF Node "Check if attachments exist"**  
   - Condition: Check if `{{$json["body"]["attachments"]}}` is a non-empty array  
   - True Output: Process attachments  
   - False Output: Proceed to test check or direct email send  

3. **Add Split Out Node "Separate attachment data"**  
   - Field to Split Out: `body.attachments`  
   - Connect True output of IF node to this node  

4. **Add Set Node "Fix Base64 string"**  
   - Assign field `data` to expression: `{{$json.data.replace(/^data:.*?,/, '')}}`  
   - Include all other fields  
   - Connect output of Split Out node here  

5. **Add Split In Batches Node "Process Each Attachment"**  
   - No special options needed  
   - Connect output of Set node here  

6. **Add Convert To File Node "Convert Attachment to File"**  
   - Operation: To Binary  
   - Source Property: `data`  
   - File Name: Use expression `{{$json.name}}`  
   - Connect one output of Split In Batches node here  

7. **Connect the other output of Split In Batches node directly to Aggregate node**  

8. **Add Aggregate Node "Prepare Attachments for Email"**  
   - Include Binaries: Enabled  
   - Field to Aggregate: `data`  
   - Connect outputs of Convert To File and Split In Batches nodes appropriately  

9. **Add Gmail Node "Email With Attachments"**  
   - Use Gmail OAuth2 credentials  
   - Send To: If `body.to` is array, join with `; ` else use string directly:  
     `={{ Array.isArray($('Webhook').item.json.body.to) ? $('Webhook').item.json.body.to.join('; ') : $('Webhook').item.json.body.to }}`  
   - Subject: `={{ $('Webhook').item.json.body.subject }}`  
   - Message: `={{ $('Webhook').item.json.body.content }}` (email body)  
   - CC, BCC, Reply-To, Sender Name: Use optional YAML fields with null coalescing (e.g. `={{ $('Webhook').item.json.body.cc ?? '' }}`)  
   - Attachments Binary: Use the binaries aggregated, dynamically referencing keys:  
     `={{ Object.keys($binary).join(',') }}`  
   - Email Type: Text  
   - Connect output of Aggregate node here  

10. **Add Gmail Node "Email Without Attachments"**  
    - Same as above but no attachments field  
    - Send To, Subject, Message, and options from webhook body JSON  
    - Connect False output of IF "Check if attachments exist" to this node (via test check)  

11. **Add IF Node "Check if it is a test"**  
    - Condition: Check if `{{$json.body.test}}` is `true` (boolean)  
    - True Output: Respond with test confirmation  
    - False Output: Proceed to email sending nodes  

12. **Add Respond to Webhook Node "Test Successful"**  
    - Response Body: `"Test successful"`  
    - Connect True output of test IF node here  

13. **Add DateTime Node "Get date"**  
    - No special config  
    - Connect outputs of both Gmail nodes to this node  

14. **Add Respond to Webhook Node "Respond to Obsidian"**  
    - Response Body:  
      ```
      =E-mail sent on  {{ new Date($json.currentDate).toLocaleString('en-GB', { day: '2-digit', month: 'long', year: 'numeric', hour: '2-digit', minute: '2-digit', hour12: false }).replace(':', 'h') }}
      ```  
    - Connect output of DateTime node here  

15. **Add Sticky Notes at appropriate locations**  
    - Add notes explaining the workflow overview, YAML example, setup instructions, attachment handling, email sending confirmation, and branding/logo  
    - Use markdown and images as needed for clarity  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow leverages the [Obsidian Post Webhook plugin](https://github.com/Masterb1234/obsidian-post-webhook/) to trigger email sending from notes. | Plugin repository for setup and configuration                                                        |
| Ensure Gmail OAuth2 credentials are configured in n8n before activating email nodes.                 | Gmail OAuth2 credential setup required in n8n                                                        |
| The workflow automatically converts base64 attachments from Obsidian to binary files suitable for Gmail. | Attachment handling automation                                                                       |
| Use YAML frontmatter in your Obsidian notes to define email metadata such as recipients, subject, sender name, and reply-to address. | YAML frontmatter example in Sticky Note1                                                             |
| Test your webhook connection using the `test` flag in the POST payload before sending real emails. | Testing mechanism to prevent accidental email sends                                                  |
| For troubleshooting, check n8n execution logs for errors related to OAuth2 authentication, attachment encoding, or webhook payload parsing. | Troubleshooting guide                                                                                 |

---

This documentation fully captures the workflow’s structure, node-by-node functionality, and instructions for rebuilding and maintaining it. It is designed to support advanced modifications and integration with Obsidian notes for seamless email dispatch.