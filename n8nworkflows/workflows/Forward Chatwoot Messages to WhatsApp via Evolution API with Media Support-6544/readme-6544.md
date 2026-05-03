Forward Chatwoot Messages to WhatsApp via Evolution API with Media Support

https://n8nworkflows.xyz/workflows/forward-chatwoot-messages-to-whatsapp-via-evolution-api-with-media-support-6544


# Forward Chatwoot Messages to WhatsApp via Evolution API with Media Support

### 1. Workflow Overview

This workflow is designed to forward messages received in Chatwoot to WhatsApp via the Evolution API, with full support for media attachments. It is intended for scenarios where messages originating from Chatwoot need to be relayed to WhatsApp users, including text messages, media files (images, videos, audio, documents), and special survey messages. The workflow handles message type classification, privacy checks, and media type detection to ensure that the forwarded content is formatted correctly for WhatsApp via Evolution API.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives incoming data from another workflow, including the Evolution API instance and Chatwoot message JSON.

- **1.2 Message Event Filtering:** Differentiates message events such as new messages, assignment changes, and survey messages.

- **1.3 Message Privacy and Type Classification:** Checks if the message is private and identifies incoming, outgoing, or survey messages.

- **1.4 Attachment Presence and Content Validation:** Determines if the message contains attachments and/or text content.

- **1.5 Attachment Processing Loop:** Iterates through all attachments, detecting file types and dispatching the appropriate Evolution API request to send media to WhatsApp.

- **1.6 Message Sending:** Sends either text messages or media messages (image, video, audio, document) to WhatsApp through Evolution API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow by receiving inputs from another workflow. It expects two inputs: the Evolution API instance name and the Chatwoot message JSON object.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Switch (initial event filter)  
- Object message (extract Chatwoot message object)

**Node Details:**

- **When Executed by Another Workflow**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point triggered by another workflow, accepts inputs `nm_instancia_evolution` (API instance name) and `ds_msg` (Chatwoot message JSON).  
  - Inputs: External trigger with JSON inputs  
  - Outputs: Passes inputs downstream  
  - Failure Modes: Missing or malformed inputs may cause downstream errors.

- **Switch (initial event filter)**  
  - Type: Switch  
  - Role: Filters incoming events by `ds_msg.event` to identify "message_created" (new message) or "assignee_changed" (agent assignment changes).  
  - Outputs: Routes new messages for processing; ignores or no-ops for other events.  
  - Expressions: Uses expression `={{ $json.ds_msg?.event }}`  
  - Failure Modes: Event field missing or unexpected values.

- **Object message**  
  - Type: Code  
  - Role: Extracts the Chatwoot message object from the input JSON (`ds_msg`).  
  - Configuration: Returns `ds_msg` JSON for further processing.  
  - Failure Modes: If `ds_msg` is missing or malformed, will fail.

---

#### 2.2 Message Privacy and Type Classification

**Overview:**  
This block checks if the message is private and classifies it as incoming, outgoing, or satisfaction survey message.

**Nodes Involved:**  
- Check if message is private (If)  
- Gateway (Switch for message_type and content_type)  
- No Operation, do nothing (no-op for private or unhandled messages)  
- Send text - Satisfaction Survey (Link chatwoot)  

**Node Details:**

- **Check if message is private**  
  - Type: If  
  - Role: Checks if the message field `private` is false (not private).  
  - Expression: `={{ $json.private }}` with boolean false condition  
  - Outputs: Continues processing if not private; no-ops if private.  
  - Failure Modes: Missing `private` field or unexpected data type.

- **Gateway**  
  - Type: Switch  
  - Role: Based on `message_type` and `content_type`, routes messages into three categories: Incoming, Outgoing, or Satisfaction Survey.  
  - Conditions:  
    - Incoming: `message_type == "incoming"`  
    - Outgoing: `message_type == "outgoing"`  
    - Satisfaction Survey: `content_type == "input_csat"`  
  - Failure Modes: Missing fields or unexpected values.

- **No Operation, do nothing**  
  - Type: NoOp  
  - Role: Used to explicitly halt processing for private messages or unhandled message types.

- **Send text - Satisfaction Survey (Link chatwoot)**  
  - Type: Evolution API node  
  - Role: Sends satisfaction survey messages as plain text to WhatsApp.  
  - Configuration: Uses `remoteJid` from sender phone number, sends message text from `content`.  
  - Credentials: Requires Evolution API credentials.  
  - Failure Modes: API authentication errors, network timeouts.

---

#### 2.3 Attachment Presence and Content Validation

**Overview:**  
Determines if the message contains attachments and/or text content to decide the sending path.

**Nodes Involved:**  
- Has attachment? (If)  
- Possui texto? (If)  
- Array of attachments (Code)  

**Node Details:**

- **Has attachment?**  
  - Type: If  
  - Role: Checks if `attachments.length` > 0.  
  - Condition: Numeric greater than zero on attachments array length.  
  - Outputs: Routes to attachment processing or to text content check.  
  - Failure Modes: Missing `attachments` field or empty array.

- **Possui texto?** ("Has Text?")  
  - Type: If  
  - Role: Checks if the message `content` is not empty.  
  - Condition: String not empty on `content` field.  
  - Outputs: Sends text message if true.  
  - Failure Modes: Missing or empty `content` field.

- **Array of attachments**  
  - Type: Code  
  - Role: Extracts the `attachments` array from the message JSON for iteration.  
  - Output: Array of attachment objects.  
  - Failure Modes: If `attachments` is malformed or missing.

---

#### 2.4 Attachment Processing Loop

**Overview:**  
Loops over each attachment to detect its file type and sends it via the appropriate Evolution API message type.

**Nodes Involved:**  
- Loop array of attachments (SplitInBatches)  
- Check attachment file type (Switch)  
- Send document message  
- Send image message  
- Send audio message  
- Send video message  

**Node Details:**

- **Loop array of attachments**  
  - Type: SplitInBatches  
  - Role: Iterates over each attachment independently.  
  - Batch size: Default (one item per batch).  
  - Failure Modes: Large number of attachments may impact performance.

- **Check attachment file type**  
  - Type: Switch  
  - Role: Routes attachment based on `file_type` to document, image, audio, or video sending nodes.  
  - Conditions:  
    - File: `file_type == "file"` (PDF, Word, Excel, etc.)  
    - Image: `file_type == "image"`  
    - Audio: `file_type == "audio"`  
    - Video: `file_type == "video"`  
  - Failure Modes: Unknown or unsupported file types cause no output.

- **Send document message**  
  - Type: Evolution API  
  - Role: Sends document files with caption and filename derived from URL.  
  - Parameters:  
    - `media`: attachment data URL  
    - `caption`: message content text  
    - `fileName`: decoded filename from URL  
    - `remoteJid`: recipient phone number  
  - Failure Modes: Invalid media URL, API errors.

- **Send image message**  
  - Type: Evolution API  
  - Role: Sends images with optional caption.  
  - Parameters similar to document message.  
  - Failure Modes: Unsupported image formats, API errors.

- **Send audio message**  
  - Type: Evolution API  
  - Role: Sends audio files with optional delay.  
  - Parameters:  
    - `delay`: 1200 ms to avoid API overload.  
  - Failure Modes: Unsupported audio formats, API errors.

- **Send video message**  
  - Type: Evolution API  
  - Role: Sends video files with quoted message referencing.  
  - Failure Modes: Large video files may cause timeouts, API errors.

---

#### 2.5 Message Sending

**Overview:**  
Sends the final text or media message to the WhatsApp user via Evolution API.

**Nodes Involved:**  
- Send message text (Evolution API)  

**Node Details:**

- **Send message text**  
  - Type: Evolution API  
  - Role: Sends plain text messages to the recipient WhatsApp number.  
  - Parameters:  
    - `remoteJid`: recipient phone number extracted from conversation metadata  
    - `messageText`: message content  
    - `options_message`: includes delay and quoted message ID for context  
  - Failure Modes: API errors, invalid phone numbers, message content issues.

---

### 3. Summary Table

| Node Name                     | Node Type                     | Functional Role                            | Input Node(s)                | Output Node(s)                         | Sticky Note                                                                                                      |
|-------------------------------|-------------------------------|--------------------------------------------|-----------------------------|---------------------------------------|------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger       | Entry point, receives inputs from other workflow | External trigger             | Switch                                | ## INPUT To use this function, you must provide the Evolution API instance and the JSON body of the message sent by Chatwoot. |
| Switch                        | Switch                        | Filters event type (new message, assignment change) | When Executed by Another Workflow | Object message, No Operation, do nothing | ## CHECK IF MESSAGE IS PRIVATE If the user is sending a private message, it will not be forwarded to the recipient's WhatsApp. |
| Object message                | Code                          | Extracts Chatwoot message JSON object      | Switch                      | Check if message is private            | ## MESSAGE TYPE Checks whether the message is an incoming message, outgoing message, or a Chatwoot survey submission. |
| Check if message is private   | If                            | Checks if message is private                | Object message              | Gateway, No Operation, do nothing      |                                                                                                                  |
| Gateway                      | Switch                        | Routes message by type: incoming, outgoing, survey | Check if message is private  | Has attachment?, Send text - Satisfaction Survey (Link chatwoot) |                                                                                                                  |
| Has attachment?               | If                            | Checks if message contains attachments      | Gateway                     | Array of attachments, Possui texto?    | ## DOES THE MESSAGE CONTAIN A FILE? Checks if the user is sending any type of file in the message.               |
| Possui texto?                 | If                            | Checks if message contains text content     | Has attachment?             | Send message text                      |                                                                                                                  |
| Array of attachments          | Code                          | Extracts attachments array for processing   | Has attachment?             | Loop array of attachments              | ## FILES LOOP Checks the media type attached to the message received in Chatwoot to ensure the content is correctly sent to WhatsApp using the compatible type in the Evolution API. |
| Loop array of attachments     | SplitInBatches                | Iterates over each attachment                | Array of attachments        | Check attachment file type             |                                                                                                                  |
| Check attachment file type    | Switch                        | Routes attachment by file type               | Loop array of attachments   | Send document message, Send image message, Send audio message, Send video message |                                                                                                                  |
| Send document message         | Evolution API                 | Sends document type media                     | Check attachment file type  | Loop array of attachments              |                                                                                                                  |
| Send image message            | Evolution API                 | Sends image media                             | Check attachment file type  | Loop array of attachments              |                                                                                                                  |
| Send audio message            | Evolution API                 | Sends audio media                             | Check attachment file type  | Loop array of attachments              |                                                                                                                  |
| Send video message            | Evolution API                 | Sends video media                             | Check attachment file type  | Loop array of attachments              |                                                                                                                  |
| Send message text             | Evolution API                 | Sends plain text message                       | Possui texto?               |                                       |                                                                                                                  |
| No Operation, do nothing      | NoOp                         | Halts processing for private or unhandled messages | Switch, Check if message is private |                                       |                                                                                                                  |
| Send text - Satisfaction Survey (Link chatwoot) | Evolution API                 | Sends satisfaction survey messages as text   | Gateway                     |                                       |                                                                                                                  |
| Sticky Note                  | Sticky Note                   | Describes file presence check                 |                             |                                       | ## DOES THE MESSAGE CONTAIN A FILE? Checks if the user is sending any type of file in the message.               |
| Sticky Note1                 | Sticky Note                   | Describes message type differentiation        |                             |                                       | ## MESSAGE TYPE Checks whether the message is an incoming message, outgoing message, or a Chatwoot survey submission. |
| Sticky Note2                 | Sticky Note                   | Describes private message check                |                             |                                       | ## CHECK IF MESSAGE IS PRIVATE If the user is sending a private message, it will not be forwarded to the recipient's WhatsApp. |
| Sticky Note3                 | Sticky Note                   | Describes files loop for media processing     |                             |                                       | ## FILES LOOP Checks the media type attached to the message received in Chatwoot to ensure the content is correctly sent to WhatsApp using the compatible type in the Evolution API. |
| Sticky Note4                 | Sticky Note                   | Describes input requirements                    |                             |                                       | ## INPUT To use this function, you must provide the Evolution API instance and the JSON body of the message sent by Chatwoot. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When Executed by Another Workflow" node**  
   - Type: Execute Workflow Trigger  
   - Configure inputs:  
     - `nm_instancia_evolution` (string)  
     - `ds_msg` (object)  
   - Position: Start of workflow

2. **Add a "Switch" node**  
   - Type: Switch  
   - Configure rules on `{{$json.ds_msg?.event}}`:  
     - Output "NOVA MENSAGEM" if equals "message_created"  
     - Output "AGENTE ATRIBUÍDO" if equals "assignee_changed"  
   - Connect output "NOVA MENSAGEM" to next processing, others to No Operation

3. **Add "Object message" node**  
   - Type: Code  
   - JavaScript code: `return $('When Executed by Another Workflow').first().json.ds_msg;`  
   - Connect from "Switch" node's "NOVA MENSAGEM" output

4. **Add "Check if message is private" node**  
   - Type: If  
   - Condition: Boolean false on `{{$json.private}}`  
   - Connect input from "Object message"  
   - Outputs: true to "Gateway", false to "No Operation"

5. **Add "Gateway" node**  
   - Type: Switch  
   - Configure conditions on fields of the message JSON:  
     - Incoming message: `message_type == "incoming"`  
     - Outgoing message: `message_type == "outgoing"`  
     - Satisfaction Survey Message: `content_type == "input_csat"`  
   - Connect true output to "Has attachment?" (incoming/outgoing) or "Send text - Satisfaction Survey" for survey messages

6. **Add "Has attachment?" node**  
   - Type: If  
   - Condition: `attachments.length > 0`  
   - Connect from "Gateway" outputs for Incoming/Outgoing messages  
   - True output to "Array of attachments"  
   - False output to "Possui texto?"

7. **Add "Possui texto?" node**  
   - Type: If  
   - Condition: `content` is not empty string  
   - True output to "Send message text"  
   - False output to end (no further processing)

8. **Add "Array of attachments" node**  
   - Type: Code  
   - JavaScript code: `return $input.first().json.attachments;`  
   - Connect from "Has attachment?" true output

9. **Add "Loop array of attachments" node**  
   - Type: SplitInBatches  
   - Connect from "Array of attachments"

10. **Add "Check attachment file type" node**  
    - Type: Switch  
    - Conditions on `file_type`:  
      - "file" → Send document message  
      - "image" → Send image message  
      - "audio" → Send audio message  
      - "video" → Send video message  
    - Connect from "Loop array of attachments"

11. **Add "Send document message" node**  
    - Type: Evolution API  
    - Configure:  
      - `media`: `{{$json.data_url}}`  
      - `caption`: `={{ $('Object message').first().json.content }}`  
      - `fileName`: `={{ decodeURIComponent($json.data_url.split('/').pop()) }}`  
      - `remoteJid`: phone number from message metadata  
      - `instanceName`: from input `nm_instancia_evolution`  
      - Set message quoting referencing message ID  
    - Connect from "Check attachment file type" output "File"

12. **Add "Send image message" node**  
    - Type: Evolution API  
    - Parameters similar to document message, operation "send-image"  
    - Connect from "Check attachment file type" output "Image"

13. **Add "Send audio message" node**  
    - Type: Evolution API  
    - Parameters similar, with `delay` 1200 ms, operation "send-audio"  
    - Connect from "Check attachment file type" output "Audio"

14. **Add "Send video message" node**  
    - Type: Evolution API  
    - Parameters similar to image, operation "send-video"  
    - Connect from "Check attachment file type" output "Video"

15. **Connect outputs of all media sending nodes back to "Loop array of attachments" to continue processing next attachment**

16. **Add "Send message text" node**  
    - Type: Evolution API  
    - Parameters:  
      - `remoteJid` from message metadata  
      - `messageText` from message content  
      - `instanceName` from input  
      - Optional delay and message quoting  
    - Connect from "Possui texto?" true output

17. **Add "Send text - Satisfaction Survey" node**  
    - Type: Evolution API  
    - Parameters: similar to Send message text, but specifically for survey messages  
    - Connect from "Gateway" output for survey messages

18. **Add "No Operation, do nothing" node**  
    - Type: NoOp  
    - Connect outputs for private messages or unhandled events

19. **Set up credentials for Evolution API nodes**  
    - Use OAuth2 or API key credentials as required by Evolution API  
    - Ensure credential references are configured for all Evolution API nodes

20. **Add Sticky Notes** (optional for documentation)  
    - Place descriptive sticky notes near relevant nodes for clarity, with content from the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                       | Context or Link                                                               |
|----------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| The workflow requires the Evolution API instance name and a valid Chatwoot message JSON structure as input.                                        | Input requirements as described in Sticky Note4                               |
| Private messages are filtered out and not forwarded to WhatsApp.                                                                                    | Sticky Note2 and Sticky Note describing privacy check                         |
| Attachment types supported: image, video, audio, document (including PDFs, Word, Excel, etc.)                                                      | Sticky Note and Switch node for file type detection                           |
| Evolution API nodes handle various media sending operations with options like message quoting and delays to handle API constraints.               | Evolution API nodes configuration details                                    |
| The workflow is designed for integration between Chatwoot and WhatsApp via Evolution API, useful for customer support automation scenarios.        | General context                                                               |
| For best performance, limit the number of attachments per message due to batch loop processing.                                                    | Performance consideration                                                     |
| Evolution API credentials must be pre-configured in n8n for authentication with the API.                                                           | Credential setup requirement                                                  |
| See official Evolution API documentation for media sending capabilities and limitations.                                                           | External documentation (not linked here)                                     |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data are legal and public.