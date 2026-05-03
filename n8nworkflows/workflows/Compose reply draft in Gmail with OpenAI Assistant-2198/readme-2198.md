Compose reply draft in Gmail with OpenAI Assistant

https://n8nworkflows.xyz/workflows/compose-reply-draft-in-gmail-with-openai-assistant-2198


# Compose reply draft in Gmail with OpenAI Assistant

### 1. Workflow Overview

This workflow automates drafting reply emails in Gmail using OpenAI Assistant. It targets email threads labeled with a specific trigger label (e.g., "AI") and generates draft replies based on the content of the most recent message in each thread. The workflow continuously runs on a schedule, processing all threads with the trigger label, generating AI-assisted draft replies, attaching these drafts to the appropriate Gmail threads, and finally removing the trigger label to prevent reprocessing.

**Target Use Cases:**  
- Automating customer support or inquiry responses with AI-generated drafts.  
- Integrating custom knowledge bases into AI replies for personalized messaging.  
- Streamlining email response workflows with minimal manual drafting.

**Logical Blocks:**

- **1.1 Scheduling and Thread Retrieval:** Trigger workflow periodically and gather Gmail threads with the specified label.  
- **1.2 Thread and Message Handling:** Iterate through each thread, retrieve messages, and isolate the last message for AI processing.  
- **1.3 AI Processing:** Send the last message content to OpenAI Assistant to generate a draft reply.  
- **1.4 Draft Message Construction:** Convert AI response from Markdown to HTML, build a raw RFC-compliant email message, and encode it.  
- **1.5 Draft Insertion and Cleanup:** Insert the draft reply into the original Gmail thread and remove the trigger label to avoid duplicate processing.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling and Thread Retrieval

- **Overview:**  
  This block triggers the workflow every minute (configurable) and fetches Gmail threads that have the specified trigger label, enabling periodic and automated processing.

- **Nodes Involved:**  
  - Schedule trigger (1 min)  
  - Get threads with specific labels  

- **Node Details:**  

  - **Schedule trigger (1 min)**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow at fixed intervals (default every 1 minute).  
    - Configuration: Interval set to 1 minute; can be adjusted as needed.  
    - Input: N/A (trigger node)  
    - Output: Initiates downstream nodes.  
    - Edge cases: Misconfiguration could cause excessive API calls; rate limits on Gmail API.  

  - **Get threads with specific labels**  
    - Type: Gmail Node (Thread resource)  
    - Role: Fetches Gmail threads filtered by a user-defined label (e.g., "AI").  
    - Configuration: Label IDs must be set to the trigger label used in Gmail.  
    - Input: Trigger from Schedule node.  
    - Output: List of threads matching label criteria.  
    - Edge cases: Empty results if no labeled threads exist; Gmail API quota limits; label not found errors.

#### 1.2 Thread and Message Handling

- **Overview:**  
  This block processes each fetched thread individually, retrieving all messages per thread and isolating the last message for reply drafting.

- **Nodes Involved:**  
  - Loop over threads  
  - Get single message content  
  - Get thread messages  
  - Return last message in thread  

- **Node Details:**  

  - **Loop over threads**  
    - Type: SplitInBatches  
    - Role: Processes threads one by one to avoid bulk processing issues and simplify per-thread operations.  
    - Configuration: Default batch size (usually 1).  
    - Input: Threads array from "Get threads with specific labels".  
    - Output: Single thread per iteration.  
    - Edge cases: Large number of threads could slow workflow; batch processing helps avoid timeouts.  

  - **Get single message content**  
    - Type: Gmail Node (Message resource)  
    - Role: Retrieves detailed content for the first message ID in the current thread (passed from SplitInBatches).  
    - Configuration: Uses expression to get messageId from current thread item.  
    - Input: Current thread data.  
    - Output: Full message content including sender, subject, body, threadId.  
    - Edge cases: Message ID not found; API errors; auth failures.  

  - **Get thread messages**  
    - Type: Gmail Node (Thread resource)  
    - Role: Retrieves all messages in the current thread.  
    - Configuration: Uses current thread ID.  
    - Input: Current thread data.  
    - Output: Array of messages in thread.  
    - Edge cases: Empty threads; API rate limits.  

  - **Return last message in thread**  
    - Type: Limit  
    - Role: Limits the set of messages to the last one (most recent message) for AI processing.  
    - Configuration: Keep last item only.  
    - Input: Array of messages from "Get thread messages".  
    - Output: Single last message.  
    - Edge cases: Empty message lists; failure if thread has no messages.

#### 1.3 AI Processing

- **Overview:**  
  This block sends the last message text content to the OpenAI Assistant node to generate a draft reply using AI.

- **Nodes Involved:**  
  - Ask OpenAI Assistant  
  - Map fields for further processing  

- **Node Details:**  

  - **Ask OpenAI Assistant**  
    - Type: OpenAI Assistant Node (Langchain OpenAI integration)  
    - Role: Sends message text as prompt input to a configured OpenAI Assistant to generate a draft reply.  
    - Configuration: Uses the "text" input from the last message JSON; selects a specific assistant by ID (customizable).  
    - Credentials: Requires OpenAI API credentials.  
    - Input: Text content from last message.  
    - Output: AI-generated reply in Markdown.  
    - Edge cases: API quota errors, network timeouts, invalid prompt data, assistant misconfiguration.  

  - **Map fields for further processing**  
    - Type: Set Node  
    - Role: Extracts and maps relevant fields for downstream processing: AI response, threadId, recipient email (from), subject, and messageId.  
    - Configuration: Uses expressions to pull fields from previous nodes, especially from "Get single message content".  
    - Input: AI response and message metadata.  
    - Output: Structured JSON with all required data for email construction.  
    - Edge cases: Missing fields in input JSON; expression evaluation errors.

#### 1.4 Draft Message Construction

- **Overview:**  
  This block transforms the AI reply from Markdown to HTML, builds a raw RFC-compliant email message string, and encodes it in base64 for Gmail API consumption.

- **Nodes Involved:**  
  - Convert response to HTML  
  - Build email raw  
  - Convert raw to base64  

- **Node Details:**  

  - **Convert response to HTML**  
    - Type: Markdown Node  
    - Role: Converts AI-generated Markdown reply into HTML format.  
    - Configuration: Mode set to convert Markdown to HTML.  
    - Input: AI response field from "Map fields for further processing".  
    - Output: HTML formatted reply string.  
    - Edge cases: Invalid Markdown input; conversion errors.  

  - **Build email raw**  
    - Type: Set Node  
    - Role: Constructs the raw email string following RFC standards, including headers (To, Subject, Content-Type) and the HTML body.  
    - Configuration: Uses expressions to insert To address, Subject, and HTML response into raw email string.  
    - Input: HTML content and mapped fields.  
    - Output: Raw email string ready for encoding.  
    - Edge cases: Malformed email headers; missing fields causing invalid email format.  

  - **Convert raw to base64**  
    - Type: Code Node (JavaScript)  
    - Role: Encodes the raw email message string into base64, required by Gmail API for draft creation.  
    - Configuration: Node runs once per item; uses Buffer encoding.  
    - Input: Raw email string from "Build email raw".  
    - Output: base64 encoded string under 'encoded' property.  
    - Edge cases: Encoding failures; empty input strings.

#### 1.5 Draft Insertion and Cleanup

- **Overview:**  
  This block inserts the encoded draft reply into the correct Gmail thread and removes the trigger label to prevent reprocessing.

- **Nodes Involved:**  
  - Add email draft to thread  
  - Remove AI label from email  

- **Node Details:**  

  - **Add email draft to thread**  
    - Type: HTTP Request Node  
    - Role: Calls Gmail API endpoint to create a draft message attached to the original thread using the base64 encoded raw message.  
    - Configuration: POST request to `https://www.googleapis.com/gmail/v1/users/me/drafts` with JSON body containing encoded message and threadId.  
    - Input: base64 encoded email and threadId from previous nodes.  
    - Output: API response confirming draft creation.  
    - Credentials: OAuth2 credentials for Gmail required.  
    - Edge cases: API errors, insufficient permissions, invalid message encoding, threadId mismatch.  

  - **Remove AI label from email**  
    - Type: Gmail Node (Thread resource)  
    - Role: Removes the trigger label from the processed Gmail thread to avoid duplicate handling.  
    - Configuration: Uses threadId from mapped fields; operation set to "removeLabels".  
    - Input: Thread ID from previous nodes.  
    - Output: Confirmation of label removal.  
    - Credentials: OAuth2 Gmail credentials.  
    - Edge cases: Label not found; permission issues; API rate limits.

---

### 3. Summary Table

| Node Name                      | Node Type                | Functional Role                          | Input Node(s)                     | Output Node(s)                 | Sticky Note                                                                                                                              |
|--------------------------------|--------------------------|----------------------------------------|----------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule trigger (1 min)        | Schedule Trigger         | Triggers workflow at regular intervals | None                             | Get threads with specific labels| "Run the workflow in equal intervals and check for threads with specific labels (trigger labels)."                                       |
| Get threads with specific labels| Gmail (Thread resource)  | Retrieves Gmail threads with label     | Schedule trigger (1 min)          | Loop over threads              | "Run the workflow in equal intervals and check for threads with specific labels (trigger labels)."                                       |
| Loop over threads               | SplitInBatches           | Processes threads one-by-one            | Get threads with specific labels | Get single message content, Get thread messages |                                                                                                  |
| Get single message content      | Gmail (Message resource) | Retrieves single message content        | Loop over threads                 | Ask OpenAI Assistant           | "Retrieve content of the last message in the thread."                                                                                    |
| Get thread messages             | Gmail (Thread resource)  | Retrieves all messages in a thread      | Loop over threads                 | Return last message in thread  | "Get last message from thread. Return all messages for a single thread and pass for further processing only the last one."              |
| Return last message in thread   | Limit                    | Limits messages to last one for processing | Get thread messages               | Loop over threads              | "Get last message from thread. Return all messages for a single thread and pass for further processing only the last one."              |
| Ask OpenAI Assistant            | OpenAI Assistant         | Sends last message text to AI for reply draft | Get single message content        | Map fields for further processing | "Transfer email content to OpenAI Assitant and return AI-generated reply."                                                              |
| Map fields for further processing| Set                     | Maps AI response and message metadata  | Ask OpenAI Assistant              | Convert response to HTML       | "Return message content. Retrieve content of the last message in the thread."                                                           |
| Convert response to HTML        | Markdown                 | Converts AI reply from Markdown to HTML| Map fields for further processing | Build email raw               | "Create HTML message. Convert incoming Markdown from OpenAI Assistant into HTML content."                                                |
| Build email raw                | Set                      | Constructs raw RFC-compliant email string | Convert response to HTML          | Convert raw to base64          | "Build and encode message. Create raw message in RFC standard and encode it into base64 string (please see Gmail API reference)."       |
| Convert raw to base64           | Code                     | Encodes raw email string to base64     | Build email raw                  | Add email draft to thread     | "Build and encode message. Create raw message in RFC standard and encode it into base64 string (please see Gmail API reference)."       |
| Add email draft to thread       | HTTP Request             | Inserts draft into Gmail thread         | Convert raw to base64            | Remove AI label from email    | "Insert reply draft. Add reply draft from OpenAI Assistant to specific Gmail thread."                                                   |
| Remove AI label from email      | Gmail (Thread resource)  | Removes trigger label from thread       | Add email draft to thread        | None                         | "Remove label. Delete trigger label from the Gmail thread."                                                                              |
| Sticky Note                    | Sticky Note              | Workflow overview and configuration note | None                            | None                         | "Reply draft with OpenAI Assistant. This workflow automatically transfers content of incoming email messages with specific labels into OpenAI Assitant and returns reply draft. After draft is composed, trigger label is deleted from the thread. **Please remember to configure your OpenAI Assistant first.**" |
| Sticky Note1                   | Sticky Note              | Workflow usage notes and credits        | None                            | None                         | "⚠️ Note: 1. Complete video guide for this workflow is available on my YouTube. 2. Remember to add your credentials and configure nodes. 3. Subscribe to YouTube channel or newsletter." |
| Sticky Note2                   | Sticky Note              | Scheduling and thread retrieval overview| None                            | None                         | "Schedule trigger and get emails. Run the workflow in equal intervals and check for threads with specific labels (trigger labels)."     |
| Sticky Note4                   | Sticky Note              | AI response generation explanation      | None                            | None                         | "Generate reply. Transfer email content to OpenAI Assitant and return AI-generated reply."                                              |
| Sticky Note5                   | Sticky Note              | Markdown to HTML conversion explanation | None                            | None                         | "Create HTML message. Convert incoming Markdown from OpenAI Assistant into HTML content."                                                |
| Sticky Note7                   | Sticky Note              | Raw message building and encoding explanation | None                            | None                         | "Build and encode message. Create raw message in RFC standard and encode it into base64 string (please see Gmail API reference)."       |
| Sticky Note8                   | Sticky Note              | Draft insertion explanation             | None                            | None                         | "Insert reply draft. Add reply draft from OpenAI Assistant to specific Gmail thread."                                                   |
| Sticky Note9                   | Sticky Note              | Label removal explanation                | None                            | None                         | "Remove label. Delete trigger label from the Gmail thread."                                                                              |
| Sticky Note10                  | Sticky Note              | Last message content retrieval explanation | None                            | None                         | "Return message content. Retrieve content of the last message in the thread."                                                           |
| Sticky Note11                  | Sticky Note              | Last message filtering explanation       | None                            | None                         | "Get last message from thread. Return all messages for a single thread and pass for further processing only the last one."              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to every 1 minute (customizable).  
   - No credentials needed.  

2. **Create Gmail Node "Get threads with specific labels"**  
   - Resource: Thread  
   - Operation: Get  
   - Filters: Set `labelIds` to the Gmail label ID you want to monitor (e.g., "AI").  
   - Credentials: Configure Gmail OAuth2 credentials.  
   - Connect Schedule Trigger output to this node input.  

3. **Create SplitInBatches Node "Loop over threads"**  
   - Default batch size (1).  
   - Connect output of "Get threads with specific labels" to this node.  

4. **Create Gmail Node "Get single message content"**  
   - Resource: Message  
   - Operation: Get  
   - Message ID: Use expression `={{ $json.id }}` to get messageId from current thread item.  
   - Credentials: Gmail OAuth2.  
   - Connect "Loop over threads" main output (index 0) to this node.  

5. **Create Gmail Node "Get thread messages"**  
   - Resource: Thread  
   - Operation: Get  
   - Thread ID: Use expression `={{ $json.id}}` (from current thread).  
   - Credentials: Gmail OAuth2.  
   - Connect "Loop over threads" main output (index 1) to this node.  

6. **Create Limit Node "Return last message in thread"**  
   - Keep: Last item only.  
   - Connect output of "Get thread messages" to this node.  

7. **Connect "Return last message in thread" output back to "Loop over threads" input**  
   - This creates a loop to process the last message per thread.

8. **Connect "Get single message content" output to "Ask OpenAI Assistant" node**  
   - Create OpenAI Assistant Node (Langchain integration).  
   - Text input: Use expression `={{ $json.text }}` from the message content.  
   - Select your configured OpenAI Assistant (by assistantId).  
   - Credentials: OpenAI API credentials.  

9. **Create Set Node "Map fields for further processing"**  
   - Map:  
     - `response` = AI output `={{ $json.output }}`  
     - `threadId` = From "Get single message content" `={{ $('Get single message content').item.json.threadId }}`  
     - `to` = From "Get single message content" `={{ $('Get single message content').item.json.from.text }}`  
     - `subject` = From "Get single message content" `={{ $('Get single message content').item.json.subject }}`  
     - `messageId` = From "Get threads with specific labels" `={{ $('Get threads with specific labels').item.json.id }}`  
   - Connect "Ask OpenAI Assistant" output to this node.  

10. **Create Markdown Node "Convert response to HTML"**  
    - Mode: markdownToHtml  
    - Input: Use expression `={{ $json.response }}` from previous node.  
    - Connect output of "Map fields for further processing" to this node.  

11. **Create Set Node "Build email raw"**  
    - Define new field `raw` with multi-line string:  
      ```
      To: {{ $json.to }}
      Subject: {{ $json.subject }}
      Content-Type: text/html; charset="utf-8"

      {{ $json.response }}
      ```  
    - Use expressions for `to`, `subject`, and `response` fields.  
    - Connect output of "Convert response to HTML" to this node.  

12. **Create Code Node "Convert raw to base64"**  
    - JavaScript code:  
      ```javascript
      const encoded = Buffer.from($json.raw).toString('base64');
      return { encoded };
      ```  
    - Run once per item.  
    - Connect output of "Build email raw" to this node.  

13. **Create HTTP Request Node "Add email draft to thread"**  
    - Method: POST  
    - URL: `https://www.googleapis.com/gmail/v1/users/me/drafts`  
    - Body Content Type: JSON  
    - JSON Body:  
      ```json
      {
        "message": {
          "raw": "{{ $json.encoded }}",
          "threadId": "{{ $('Map fields for further processing').item.json.threadId }}"
        }
      }
      ```  
    - Send Body: True  
    - Credentials: Gmail OAuth2  
    - Connect output of "Convert raw to base64" to this node.  

14. **Create Gmail Node "Remove AI label from email"**  
    - Resource: Thread  
    - Operation: removeLabels  
    - Thread ID: `={{ $('Map fields for further processing').item.json.threadId }}`  
    - Credentials: Gmail OAuth2  
    - Connect output of "Add email draft to thread" to this node.  

15. **Add Sticky Notes** (Optional but recommended for documentation and clarity)  
    - Add the same content as in the original workflow to explain each block and provide usage notes.  

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Complete video guide available explaining this workflow in detail.                                                         | https://youtu.be/a8Dhj3Zh9vQ                                                                            |
| Guide on composing drafts with Gmail API to understand raw message format and draft creation.                              | https://developers.google.com/gmail/api/reference/rest/v1/users.drafts/create                           |
| Subscribe to YouTube channel for more workflows and automation tips.                                                       | https://www.youtube.com/@workfloows                                                                     |
| Subscribe to newsletter for workflow updates and tips.                                                                     | https://workfloows.com/                                                                                  |
| Important: Configure your OpenAI Assistant before running this workflow to ensure proper AI responses.                     | Workflow setup instructions                                                                              |
| Best practice: Adjust trigger interval according to Gmail API quota limits and workload to avoid rate limiting.            | Workflow configuration advice                                                                            |
| Ensure Gmail OAuth2 credentials have sufficient scopes for reading threads, messages, creating drafts, and modifying labels.| Gmail API credential setup                                                                                |

---

This documentation enables both advanced users and AI agents to understand, reproduce, and modify the workflow confidently. It also highlights potential failure points, credential requirements, and external references for deeper integration knowledge.