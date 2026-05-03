Automated Gmail Reply Drafts with OpenAI Assistant for Labeled Emails

https://n8nworkflows.xyz/workflows/automated-gmail-reply-drafts-with-openai-assistant-for-labeled-emails-6442


# Automated Gmail Reply Drafts with OpenAI Assistant for Labeled Emails

### 1. Workflow Overview

This workflow automates drafting replies to incoming Gmail messages that have specific trigger labels, using OpenAI Assistant to generate the reply content. It is designed for email management scenarios where users want AI-assisted reply drafts directly integrated into Gmail threads, improving response times and consistency.

The logical flow is divided into these functional blocks:

- **1.1 Schedule Trigger and Email Retrieval:** Periodically (every minute) checks Gmail for threads labeled with specific trigger labels.
- **1.2 Thread and Message Processing:** Iterates over each email thread, retrieves messages, and isolates the last message content for AI processing.
- **1.3 OpenAI Assistant Interaction:** Sends the last email message content to an OpenAI Assistant for generating a draft reply.
- **1.4 Reply Draft Preparation:** Converts the AI response from Markdown to HTML, builds a properly formatted raw email message, and encodes it to base64.
- **1.5 Gmail Draft Creation and Label Management:** Creates a draft reply in the Gmail thread and removes the trigger label from the thread to mark it as processed.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger and Email Retrieval

- **Overview:**  
  Triggers the workflow every minute and fetches all Gmail threads with certain labels configured as triggers.

- **Nodes Involved:**  
  - Schedule trigger (1 min)  
  - Get threads with specific labels  
  - Loop over threads

- **Node Details:**

  - **Schedule trigger (1 min)**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution every 1 minute  
    - Configuration: Interval set to 1 minute  
    - Inputs: None  
    - Outputs to: Get threads with specific labels  
    - Edge cases: Workflow could overload Gmail API if interval too short or many threads exist.

  - **Get threads with specific labels**  
    - Type: Gmail node  
    - Role: Retrieves all email threads with trigger labels (labelIds array is empty by default, needs to be set)  
    - Configuration: Resource=thread, returnAll=true, filter by labelIds (must be configured with trigger label IDs)  
    - Credentials: Gmail OAuth2  
    - Inputs: From Schedule trigger  
    - Outputs to: Loop over threads  
    - Edge cases: Auth failure, empty labelIds leads to no filtering, rate limits

  - **Loop over threads**  
    - Type: SplitInBatches  
    - Role: Processes each thread one-by-one to avoid throttling and simplify downstream logic  
    - Configuration: Default batch size (1)  
    - Inputs: From Get threads with specific labels  
    - Outputs to: Get single message content (main output 0), Get thread messages (additional output 1)  
    - Edge cases: Failures in batch processing pause entire workflow

---

#### 2.2 Thread and Message Processing

- **Overview:**  
  For each thread, retrieves messages and isolates the last message in the thread to serve as input for AI reply generation.

- **Nodes Involved:**  
  - Get single message content  
  - Get thread messages  
  - Return last message in thread

- **Node Details:**

  - **Get single message content**  
    - Type: Gmail node  
    - Role: Fetches full content of the message with ID from the current thread item  
    - Configuration: Operation=get, resource=message, messageId from current item’s id  
    - Credentials: Gmail OAuth2  
    - Inputs: Loop over threads (main output 0)  
    - Outputs to: Ask OpenAI Assistant  
    - Edge cases: Message ID missing or deleted, auth errors

  - **Get thread messages**  
    - Type: Gmail node  
    - Role: Retrieves all messages of a thread for further processing  
    - Configuration: Operation=get, resource=thread, threadId from current item, returnOnlyMessages=true  
    - Credentials: Gmail OAuth2  
    - Inputs: Loop over threads (main output 1)  
    - Outputs to: Return last message in thread  
    - Edge cases: Large threads may cause timeouts, API limits

  - **Return last message in thread**  
    - Type: Limit node  
    - Role: Keeps only the last message from the list of thread messages  
    - Configuration: keep=lastItems (1 by default)  
    - Inputs: From Get thread messages  
    - Outputs to: Loop over threads (to continue batch processing)  
    - Edge cases: Empty thread messages

---

#### 2.3 OpenAI Assistant Interaction

- **Overview:**  
  Sends the last message content to the OpenAI Assistant and receives an AI-generated reply draft.

- **Nodes Involved:**  
  - Ask OpenAI Assistant  
  - Map fields for further processing

- **Node Details:**

  - **Ask OpenAI Assistant**  
    - Type: LangChain OpenAI node  
    - Role: Sends text input to OpenAI Assistant resource to generate reply draft  
    - Configuration:  
      - Input text: `{{$json.text}}` (text from last message)  
      - AssistantId: preset to "Customer assistant" (ID: asst_kmKeAtwF2rv0vgF0ujY4jlp6)  
    - Credentials: OpenAI API key  
    - Inputs: Get single message content  
    - Outputs to: Map fields for further processing  
    - Edge cases: API key issues, rate limits, empty input text, assistant misconfiguration

  - **Map fields for further processing**  
    - Type: Set node  
    - Role: Prepares data fields for email draft creation, including AI response, threadId, recipient, subject, messageId  
    - Configuration: Assigns variables like response (AI output), threadId, to, subject, messageId from previous nodes  
    - Inputs: Ask OpenAI Assistant  
    - Outputs to: Convert response to HTML  
    - Edge cases: Missing or malformed previous node data

---

#### 2.4 Reply Draft Preparation

- **Overview:**  
  Converts AI response from Markdown to HTML, builds full RFC 822 raw email message, and encodes it to base64 for Gmail API consumption.

- **Nodes Involved:**  
  - Convert response to HTML  
  - Build email raw  
  - Convert raw to base64

- **Node Details:**

  - **Convert response to HTML**  
    - Type: Markdown node  
    - Role: Transforms Markdown reply from AI into HTML format  
    - Configuration: markdownToHtml mode, input from `response` field  
    - Inputs: Map fields for further processing  
    - Outputs to: Build email raw  
    - Edge cases: Malformed Markdown, empty response

  - **Build email raw**  
    - Type: Set node  
    - Role: Constructs raw email string with headers (To, Subject) and HTML body  
    - Configuration: Uses Mustache expressions to build RFC message:  
      ```
      To: {{ $json.to }}
      Subject: {{ $json.subject }}
      Content-Type: text/html; charset="utf-8"

      {{ $json.response }}
      ```  
    - Inputs: Convert response to HTML  
    - Outputs to: Convert raw to base64  
    - Edge cases: Missing fields, invalid email addresses

  - **Convert raw to base64**  
    - Type: Code node (JavaScript)  
    - Role: Encodes raw email string into base64 format (required by Gmail API)  
    - Configuration: Runs once per item, uses Node.js Buffer  
    - Inputs: Build email raw  
    - Outputs to: Add email draft to thread  
    - Edge cases: Encoding errors, empty raw message

---

#### 2.5 Gmail Draft Creation and Label Management

- **Overview:**  
  Creates a draft reply in the Gmail thread with the encoded message and removes the AI trigger label from the thread to indicate processing completion.

- **Nodes Involved:**  
  - Add email draft to thread  
  - Remove AI label from email

- **Node Details:**

  - **Add email draft to thread**  
    - Type: HTTP Request  
    - Role: Calls Gmail API to create a draft with the encoded raw message in the specified thread  
    - Configuration:  
      - URL: `https://www.googleapis.com/gmail/v1/users/me/drafts`  
      - Method: POST  
      - Body JSON:  
        ```json
        {
          "message": {
            "raw": "{{ $json.encoded }}",
            "threadId": "{{ $('Map fields for further processing').item.json['threadId'] }}"
          }
        }
        ```  
      - Authentication: Predefined Gmail OAuth2 credential  
    - Inputs: Convert raw to base64  
    - Outputs to: Remove AI label from email  
    - Edge cases: API errors, auth failure, invalid threadId

  - **Remove AI label from email**  
    - Type: Gmail node  
    - Role: Removes the trigger label from the Gmail thread to mark it as processed  
    - Configuration:  
      - Resource: thread  
      - Operation: removeLabels  
      - ThreadId: from Map fields for further processing  
      - LabelIds: must be set to the trigger label(s) to remove (not shown in JSON)  
    - Inputs: Add email draft to thread  
    - Outputs: Workflow end  
    - Edge cases: Label not found, permission errors, threadId invalid

---

### 3. Summary Table

| Node Name                      | Node Type                  | Functional Role                           | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                                                                       |
|--------------------------------|----------------------------|-----------------------------------------|----------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Schedule trigger (1 min)        | Schedule Trigger           | Initiates workflow every 1 min           | None                             | Get threads with specific labels | ### Schedule trigger and get emails Run the workflow in equal intervals and check for threads with specific labels (trigger labels). |
| Get threads with specific labels| Gmail                     | Retrieves threads with trigger labels    | Schedule trigger (1 min)          | Loop over threads                |                                                                                                                   |
| Loop over threads               | SplitInBatches             | Processes threads one by one              | Get threads with specific labels | Get single message content, Get thread messages |                                                                                                                   |
| Get single message content      | Gmail                     | Retrieves full content of single message | Loop over threads                | Ask OpenAI Assistant            | ### Return message content Retrieve content of the last message in the thread.                                  |
| Get thread messages             | Gmail                     | Retrieves all messages in a thread        | Loop over threads                | Return last message in thread   | ### Get last message from thread Return all messages for a single thread and pass for further processing only the last one. |
| Return last message in thread   | Limit                     | Returns only the last message in thread   | Get thread messages              | Loop over threads               |                                                                                                                   |
| Ask OpenAI Assistant            | LangChain OpenAI          | Sends message text to AI assistant        | Get single message content       | Map fields for further processing | ### Generate reply Transfer email content to OpenAI Assitant and return AI-generated reply.                      |
| Map fields for further processing| Set                      | Prepares variables for email draft        | Ask OpenAI Assistant             | Convert response to HTML        |                                                                                                                   |
| Convert response to HTML        | Markdown                  | Converts AI Markdown reply to HTML        | Map fields for further processing | Build email raw                | ### Create HTML message Convert incoming Markdown from OpenAI Assistant into HTML content.                       |
| Build email raw                | Set                       | Builds raw RFC email string                | Convert response to HTML         | Convert raw to base64           | ### Build and encode message Create raw message in RFC standard and encode it into base64 string (see Gmail API reference). |
| Convert raw to base64           | Code                      | Encodes raw email to base64 format         | Build email raw                 | Add email draft to thread       |                                                                                                                   |
| Add email draft to thread       | HTTP Request              | Creates Gmail draft with encoded message   | Convert raw to base64            | Remove AI label from email      | ### Insert reply draft Add reply draft from OpenAI Assistant to specific Gmail thread.                           |
| Remove AI label from email      | Gmail                     | Removes trigger label from Gmail thread   | Add email draft to thread        | None                           | ### Remove label Delete trigger label from the Gmail thread.                                                     |
| Sticky Note                    | Sticky Note               | Workflow overview and instructions         | None                           | None                           | ## Reply draft with OpenAI Assistant This workflow automatically transfers content of incoming email messages with specific labels into OpenAI Assitant and returns reply draft. After draft is composed, trigger label is deleted from the thread. **Please remember to configure your OpenAI Assistant first.** |
| Sticky Note2                   | Sticky Note               | Explains schedule trigger and email retrieval| None                           | None                           | ### Schedule trigger and get emails Run the workflow in equal intervals and check for threads with specific labels (trigger labels). |
| Sticky Note1                   | Sticky Note               | Notes and video guide links                 | None                           | None                           | ## ⚠️ Note 1. Complete video guide for this workflow is available [on my YouTube](https://youtu.be/a8Dhj3Zh9vQ). 2. Remember to add your credentials and configure nodes (covered in the video guide). 3. If you like this workflow, please subscribe to [my YouTube channel](https://www.youtube.com/@workfloows) and/or [my newsletter](https://workfloows.com/). **Thank you for your support!** |
| Sticky Note4                   | Sticky Note               | Explains OpenAI reply generation stage      | None                           | None                           | ### Generate reply Transfer email content to OpenAI Assitant and return AI-generated reply.                      |
| Sticky Note5                   | Sticky Note               | Explains Markdown to HTML conversion stage | None                           | None                           | ### Create HTML message Convert incoming Markdown from OpenAI Assistant into HTML content.                       |
| Sticky Note7                   | Sticky Note               | Explains building and encoding email message| None                           | None                           | ### Build and encode message Create raw message in RFC standard and encode it into base64 string (please see [Gmail API reference](https://developers.google.com/gmail/api/reference/rest/v1/users.drafts/create) for more details). |
| Sticky Note8                   | Sticky Note               | Explains inserting draft into Gmail thread | None                           | None                           | ### Insert reply draft Add reply draft from OpenAI Assistant to specific Gmail thread.                           |
| Sticky Note9                   | Sticky Note               | Explains removing label from Gmail thread  | None                           | None                           | ### Remove label Delete trigger label from the Gmail thread.                                                     |
| Sticky Note10                  | Sticky Note               | Explains getting message content             | None                           | None                           | ### Return message content Retrieve content of the last message in the thread.                                  |
| Sticky Note11                  | Sticky Note               | Explains getting last message from thread    | None                           | None                           | ### Get last message from thread Return all messages for a single thread and pass for further processing only the last one. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**
   - Type: Schedule Trigger  
   - Parameters: Set interval to every 1 minute  
   - No credentials needed  
   - This starts the workflow periodically.

2. **Create a Gmail node to Get Threads with Specific Labels**
   - Type: Gmail  
   - Operation: Get Threads  
   - Resource: Thread  
   - Return All: True  
   - Filter: Set `labelIds` to the Gmail label IDs you want to monitor (e.g., "AI reply trigger")  
   - Credentials: Configure Gmail OAuth2 with an account authorized to access the mailbox  
   - Connect output of Schedule Trigger to this node.

3. **Create a SplitInBatches node to Loop over Threads**
   - Type: SplitInBatches  
   - No special parameters, default batch size 1  
   - Connect output of Gmail Get Threads to this node.

4. **Create two Gmail nodes to fetch messages:**

   - **Get Single Message Content**  
     - Type: Gmail  
     - Operation: Get Message  
     - Resource: Message  
     - Message ID: Expression `{{$json["id"]}}` (from the current item)  
     - Credentials: Gmail OAuth2 (same as above)  
     - Connect main output (index 0) of SplitInBatches to this node.

   - **Get Thread Messages**  
     - Type: Gmail  
     - Operation: Get Thread  
     - Resource: Thread  
     - Thread ID: Expression `{{$json["id"]}}` (current thread ID)  
     - Return Only Messages: True  
     - Credentials: Gmail OAuth2  
     - Connect secondary output (index 1) of SplitInBatches to this node.

5. **Create a Limit node to Return Last Message in Thread**
   - Type: Limit  
   - Parameters: Keep last item (1)  
   - Connect output of Get Thread Messages to this node.

6. **Loop output of Return Last Message in Thread back to SplitInBatches**  
   - Connect output of Limit node back to SplitInBatches node to continue processing batches.

7. **Create an OpenAI Assistant node (LangChain OpenAI)**
   - Type: LangChain OpenAI  
   - Resource: Assistant  
   - Assistant ID: Select or provide the assistant configured for customer replies (e.g., "asst_kmKeAtwF2rv0vgF0ujY4jlp6")  
   - Text: Expression `{{$json["text"]}}` (text content from the message)  
   - Credentials: OpenAI API credentials configured  
   - Connect output of Get Single Message Content to this node.

8. **Create a Set node to Map Fields for Further Processing**
   - Type: Set  
   - Assign variables:  
     - response = `{{$json["output"]}}` (OpenAI reply)  
     - threadId = `{{$node["Get single message content"].json["threadId"]}}`  
     - to = `{{$node["Get single message content"].json["from"]["text"]}}`  
     - subject = `{{$node["Get single message content"].json["subject"]}}`  
     - messageId = `{{$node["Get threads with specific labels"].json["id"]}}`  
   - Connect output of OpenAI Assistant to this node.

9. **Create a Markdown node to Convert Response to HTML**
   - Type: Markdown  
   - Mode: markdownToHtml  
   - Input markdown: Expression `{{$json["response"]}}`  
   - Connect output of Map Fields node to this node.

10. **Create a Set node to Build Raw Email Message**
    - Type: Set  
    - Assign raw field with the following template:  
      ```
      To: {{$json["to"]}}
      Subject: {{$json["subject"]}}
      Content-Type: text/html; charset="utf-8"

      {{$json["response"]}}
      ```  
    - Connect output of Convert Response to HTML node to this node.

11. **Create a Code node to Convert Raw Message to Base64**
    - Type: Code  
    - Language: JavaScript  
    - Mode: runOnceForEachItem  
    - Code:  
      ```js
      const encoded = Buffer.from($json.raw).toString('base64');
      return { encoded };
      ```  
    - Connect output of Build email raw node to this node.

12. **Create an HTTP Request node to Add Email Draft to Thread**
    - Type: HTTP Request  
    - URL: `https://www.googleapis.com/gmail/v1/users/me/drafts`  
    - Method: POST  
    - Authentication: Use predefined Gmail OAuth2 credentials  
    - Body content type: JSON  
    - Body JSON:  
      ```json
      {
        "message": {
          "raw": "{{$json.encoded}}",
          "threadId": "{{$node['Map fields for further processing'].json['threadId']}}"
        }
      }
      ```  
    - Connect output of Convert raw to base64 node to this node.

13. **Create a Gmail node to Remove AI Label from Email Thread**
    - Type: Gmail  
    - Operation: removeLabels  
    - Resource: thread  
    - Thread ID: Expression `{{$node['Map fields for further processing'].json['threadId']}}`  
    - Label IDs: Set to the label(s) that triggered the workflow (must be configured)  
    - Credentials: Gmail OAuth2  
    - Connect output of Add email draft to thread to this node.

14. **Add Sticky Notes for Documentation**
    - Add sticky notes at relevant points to explain each block and node roles, including links to Gmail API references and video guides.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Complete video guide for this workflow is available on YouTube. Remember to add your credentials and configure nodes properly. Subscribe for more workflows and tutorials.                                                              | https://youtu.be/a8Dhj3Zh9vQ and https://www.youtube.com/@workfloows                                   |
| Gmail API reference for creating drafts with raw base64 encoded messages.                                                                                                                                                                | https://developers.google.com/gmail/api/reference/rest/v1/users.drafts/create                          |
| This workflow requires proper setup of Gmail OAuth2 credentials and OpenAI API keys with access to the configured assistant.                                                                                                            | n8n credential setup documentation                                                                     |
| The workflow deletes the trigger label after creating the draft to avoid duplicate processing. Ensure label IDs are correctly configured to match your Gmail labels.                                                                    | Important to avoid infinite loops or duplicated drafts                                                |
| The workflow processes threads one-by-one (batch size 1) to avoid hitting Gmail API rate limits and to simplify error handling.                                                                                                         | Best practice for Gmail API integrations                                                              |
| Markdown conversion to HTML is necessary because OpenAI replies are in Markdown, but Gmail email bodies require HTML formatting for proper rendering.                                                                                   | Markdown node in n8n documentation                                                                    |
| Buffer encoding in the Code node uses Node.js Buffer API, which requires n8n version supporting this feature (n8n v0.155.0 or higher recommended).                                                                                       | Version-specific requirement                                                                           |

---

**Disclaimer:**  
The provided content originates exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.