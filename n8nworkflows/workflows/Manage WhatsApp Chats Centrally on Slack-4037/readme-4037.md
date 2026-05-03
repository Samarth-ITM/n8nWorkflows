Manage WhatsApp Chats Centrally on Slack

https://n8nworkflows.xyz/workflows/manage-whatsapp-chats-centrally-on-slack-4037


# Manage WhatsApp Chats Centrally on Slack

### 1. Workflow Overview

This workflow facilitates centralized management of WhatsApp chats and Slack channels, enabling seamless two-way communication between WhatsApp contacts and Slack users. It targets use cases where teams want to monitor, respond to, and archive WhatsApp messages inside Slack channels, and reciprocally send Slack messages back to WhatsApp contacts.

The logic is divided into two main blocks:

- **1.1 WhatsApp to Slack Flow**: Listens for incoming WhatsApp messages, routes them by type (text, audio, image, document), manages Slack private channels per WhatsApp contact, and posts the messages or media into the appropriate Slack channel.

- **1.2 Slack to WhatsApp Flow**: Listens for Slack messages in monitored channels, determines if the message is text or media, downloads media if necessary, and sends the message or media back to the corresponding WhatsApp contact.

---

### 2. Block-by-Block Analysis

#### 2.1 WhatsApp to Slack Flow

**Overview:**  
This block triggers on new WhatsApp messages. It ensures a private Slack channel exists for the WhatsApp sender, processes the message by type (text, audio, image, document), downloads media if applicable, and posts the content into the appropriate Slack channel.

**Nodes Involved:**  
- WhatsApp Trigger  
- Create Channel  
- get All Channels  
- Filter Channel by Name  
- Message Type  
- get audio URL  
- get image URL  
- get document URL  
- download media  
- upload media  
- Send Message in Channel  
- Sticky Note1 (documentation)

**Node Details:**

- **WhatsApp Trigger**  
  - *Type & Role:* Webhook trigger node for WhatsApp messages.  
  - *Configuration:* Listens for WhatsApp "messages" updates via WhatsApp OAuth credentials.  
  - *Inputs:* N/A (trigger)  
  - *Outputs:* Emits WhatsApp message JSON including message type and sender info.  
  - *Potential Failures:* Auth failure, webhook connectivity issues.  
  - *Notes:* Entry point for inbound WhatsApp messages.

- **Create Channel**  
  - *Type & Role:* Slack API node for creating private channels.  
  - *Configuration:* Creates a private Slack channel named after WhatsApp sender's phone number (`messages[0].from`).  
  - *Inputs:* Output from WhatsApp Trigger.  
  - *Outputs:* Slack channel creation response.  
  - *On Error:* Continues workflow to handle existing channels gracefully.  
  - *Potential Failures:* Slack API rate limits, permission denied errors.

- **get All Channels**  
  - *Type & Role:* Slack API node fetching all private channels.  
  - *Configuration:* Retrieves all private channels (returnAll enabled).  
  - *Inputs:* Output from Create Channel.  
  - *Outputs:* List of private Slack channels.  
  - *Potential Failures:* Slack API errors, connectivity.

- **Filter Channel by Name**  
  - *Type & Role:* Filter node to find matching Slack channels.  
  - *Configuration:* Filters channels where channel name equals WhatsApp sender ID.  
  - *Inputs:* List of Slack channels.  
  - *Outputs:* Channels matching WhatsApp sender.  
  - *Potential Failures:* No matching channel found, expression errors.

- **Message Type**  
  - *Type & Role:* Switch node routing based on WhatsApp message type.  
  - *Configuration:* Routes messages to one of four outputs: text, audio, image, or document, based on `messages[0].type`.  
  - *Inputs:* Filtered Slack channel info.  
  - *Outputs:* Routes to corresponding media processing or text posting flow.  
  - *Potential Failures:* Unexpected message types, missing fields.

- **get audio URL / get image URL / get document URL**  
  - *Type & Role:* WhatsApp API nodes to retrieve media URLs.  
  - *Configuration:* Calls WhatsApp API with media ID to get download URL for respective media type.  
  - *Inputs:* WhatsApp message media ID from Message Type output.  
  - *Outputs:* Media URL JSON.  
  - *Potential Failures:* Media not found, expired media URL, API auth failures.

- **download media**  
  - *Type & Role:* HTTP Request node to download media content from WhatsApp URL.  
  - *Configuration:* Performs authenticated HTTP GET with WhatsApp API Token in headers.  
  - *Inputs:* Media URL from get audio/image/document URL nodes.  
  - *Outputs:* Binary media data for upload.  
  - *Potential Failures:* HTTP errors, token expiration, large file size issues.

- **upload media**  
  - *Type & Role:* Slack API node to upload media files to Slack channel.  
  - *Configuration:* Uploads downloaded media to Slack channel identified by channel ID from Message Type output.  
  - *Inputs:* Binary media data from download media node.  
  - *Outputs:* Slack file upload confirmation.  
  - *Potential Failures:* Slack file size limits, upload failures, permission errors.

- **Send Message in Channel**  
  - *Type & Role:* Slack API node to send plain text messages.  
  - *Configuration:* Sends WhatsApp text message body to Slack channel identified by channel ID.  
  - *Inputs:* Text messages routed from Message Type text output.  
  - *Outputs:* Slack message confirmation.  
  - *Potential Failures:* Message formatting errors, Slack API rate limits.

- **Sticky Note1**  
  - *Content:* Summarizes WhatsApp to Slack Flow logic for clarity.

---

#### 2.2 Slack to WhatsApp Flow

**Overview:**  
This block triggers on Slack messages in monitored channels, determines message type (text or media), downloads media if necessary, and sends the message or media back to the WhatsApp contact linked to the Slack channel.

**Nodes Involved:**  
- Slack Trigger  
- Get Channel by ID  
- Checking Message Type  
- Get Media URL  
- Download Media  
- Send Message  
- Send Message1  
- Sticky Note (documentation)

**Node Details:**

- **Slack Trigger**  
  - *Type & Role:* Webhook trigger node listening for Slack messages in workspace.  
  - *Configuration:* Listens to all message events in the workspace via Slack OAuth credentials.  
  - *Inputs:* N/A (trigger)  
  - *Outputs:* Slack message JSON including text and files metadata.  
  - *Potential Failures:* Slack API rate limits, webhook misconfiguration.

- **Get Channel by ID**  
  - *Type & Role:* Slack API node retrieving channel info by ID.  
  - *Configuration:* Retrieves Slack channel info using channel ID extracted from Slack Trigger event.  
  - *Inputs:* Slack Trigger output.  
  - *Outputs:* Channel info JSON including channel name (which maps to WhatsApp number).  
  - *Potential Failures:* Channel not found, permission errors.

- **Checking Message Type**  
  - *Type & Role:* Switch node that routes based on Slack message content type.  
  - *Configuration:* Routes messages to “media” output if files array is non-empty, or “text” output if text field is non-empty.  
  - *Inputs:* Channel info from Get Channel by ID.  
  - *Outputs:* To Get Media URL or Send Message nodes accordingly.  
  - *Potential Failures:* Messages with neither text nor files, malformed Slack events.

- **Get Media URL**  
  - *Type & Role:* Slack API node retrieving private URL of uploaded file.  
  - *Configuration:* Uses file ID from Slack Trigger to fetch file metadata including private download URL.  
  - *Inputs:* Slack Trigger files metadata.  
  - *Outputs:* Metadata including `url_private_download`.  
  - *Potential Failures:* File not found, permission denied.

- **Download Media**  
  - *Type & Role:* HTTP Request node downloading Slack media file.  
  - *Configuration:* Downloads media binary using authenticated HTTP GET with Slack OAuth token in headers.  
  - *Inputs:* URL from Get Media URL.  
  - *Outputs:* Binary media data for WhatsApp upload.  
  - *Potential Failures:* HTTP errors, token expiration.

- **Send Message**  
  - *Type & Role:* WhatsApp API node sending text messages.  
  - *Configuration:* Sends Slack text message body back to WhatsApp using recipient phone number derived from Slack channel name.  
  - *Inputs:* Text routed from Checking Message Type.  
  - *Outputs:* WhatsApp message confirmation.  
  - *Potential Failures:* WhatsApp API errors, invalid phone numbers.

- **Send Message1**  
  - *Type & Role:* WhatsApp API node sending media messages (documents).  
  - *Configuration:* Sends downloaded Slack media as WhatsApp document message.  
  - *Inputs:* Binary media from Download Media node.  
  - *Outputs:* WhatsApp media message confirmation.  
  - *Potential Failures:* Media upload errors, file size limits.

- **Sticky Note**  
  - *Content:* Summarizes Slack to WhatsApp Flow logic.

---

### 3. Summary Table

| Node Name             | Node Type                | Functional Role                                    | Input Node(s)          | Output Node(s)             | Sticky Note                                                                                              |
|-----------------------|--------------------------|--------------------------------------------------|------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| WhatsApp Trigger      | WhatsApp Trigger          | Trigger on new WhatsApp messages                   | -                      | Create Channel              |                                                                                                        |
| Create Channel        | Slack                     | Creates Slack private channel for WhatsApp user   | WhatsApp Trigger       | get All Channels            |                                                                                                        |
| get All Channels      | Slack                     | Retrieves all private Slack channels               | Create Channel         | Filter Channel by Name      |                                                                                                        |
| Filter Channel by Name| Filter                    | Filters Slack channels by WhatsApp sender ID      | get All Channels       | Message Type                |                                                                                                        |
| Message Type          | Switch                    | Routes message by WhatsApp message type            | Filter Channel by Name | Send Message in Channel, get audio URL, get image URL, get document URL |                                                                                                        |
| get audio URL         | WhatsApp                  | Retrieves WhatsApp audio media download URL        | Message Type (audio)   | download media              |                                                                                                        |
| get image URL         | WhatsApp                  | Retrieves WhatsApp image media download URL        | Message Type (image)   | download media              |                                                                                                        |
| get document URL      | WhatsApp                  | Retrieves WhatsApp document media download URL     | Message Type (document)| download media              |                                                                                                        |
| download media        | HTTP Request              | Downloads WhatsApp media file                        | get audio/image/document URL | upload media           |                                                                                                        |
| upload media          | Slack                     | Uploads media file into Slack channel               | download media         |                            |                                                                                                        |
| Send Message in Channel| Slack                     | Sends WhatsApp text messages to Slack channel      | Message Type (text)    |                            |                                                                                                        |
| Slack Trigger         | Slack Trigger             | Trigger on Slack messages                           | -                      | Get Channel by ID           |                                                                                                        |
| Get Channel by ID     | Slack                     | Retrieves Slack channel info by ID                  | Slack Trigger          | Checking Message Type       |                                                                                                        |
| Checking Message Type | Switch                    | Routes Slack message as media or text               | Get Channel by ID      | Get Media URL, Send Message |                                                                                                        |
| Get Media URL         | Slack                     | Gets Slack media file download URL                  | Checking Message Type (media) | Download Media           |                                                                                                        |
| Download Media        | HTTP Request              | Downloads Slack media file                           | Get Media URL          | Send Message1               |                                                                                                        |
| Send Message          | WhatsApp                  | Sends Slack text messages back to WhatsApp          | Checking Message Type (text) |                          |                                                                                                        |
| Send Message1         | WhatsApp                  | Sends Slack media files back to WhatsApp as documents | Download Media      |                            |                                                                                                        |
| Sticky Note           | Sticky Note               | Documents Slack to WhatsApp Flow                     | -                      |                            | ## Slack to WhatsApp Flow<br>- Trigger: Slack Trigger listens for Slack messages (text or media).<br>- Message Type Handling: Checking Message Type uses a Switch to determine whether it’s text or media.<br>- Text Handling: Extracts text and sends it back to WhatsApp via Send Message.<br>- Media Handling: Gets media URL (Get Media URL).<br>- Downloads the file (Download Media) and (presumably in the missing portion) sends it to WhatsApp as a document/media. |
| Sticky Note1          | Sticky Note               | Documents WhatsApp to Slack Flow                     | -                      |                            | ## WhatsApp to Slack Flow<br>- Trigger: WhatsApp Trigger listens for new WhatsApp messages.<br>- Message Type Routing: Message Type node uses a Switch to check if the message is text, audio, image, or document.<br>- Slack Channel Management: Checks if a private Slack channel already exists for the WhatsApp contact (from field).<br>- If not, creates one using Create Channel.<br>- Media Processing (for non-text): Gets WhatsApp media URL via get audio/image/document URL.<br>- Downloads it (download media) and uploads to Slack using upload media.<br>- Text Handling: For text, sends message directly to Slack using Send Message in Channel. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create WhatsApp Trigger node**  
   - Type: WhatsApp Trigger  
   - Configure to listen for "messages" updates  
   - Use WhatsApp OAuth credentials  
   - Position: top-left (e.g., [-2740, 120])

2. **Create Slack Create Channel node**  
   - Type: Slack  
   - Operation: Create private channel  
   - Channel name: `={{ $('WhatsApp Trigger').item.json.messages[0].from }}`  
   - Use Slack OAuth credentials  
   - Connect WhatsApp Trigger → Create Channel

3. **Create Slack Get All Channels node**  
   - Type: Slack  
   - Operation: Get all private channels (return all enabled)  
   - Use Slack OAuth credentials  
   - Connect Create Channel → Get All Channels

4. **Create Filter node ("Filter Channel by Name")**  
   - Type: Filter  
   - Condition: `$json.name == {{$('WhatsApp Trigger').item.json.messages[0].from}}`  
   - Connect Get All Channels → Filter Channel by Name

5. **Create Switch node ("Message Type")**  
   - Type: Switch  
   - Rules: Check `$('WhatsApp Trigger').item.json.messages[0].type` equals to "text", "audio", "image", or "document"  
   - Connect Filter Channel by Name → Message Type

6. **Create WhatsApp nodes to get media URLs:**  
   - For audio, image, and document types:  
     - Type: WhatsApp  
     - Operation: mediaUrlGet  
     - mediaGetId: `={{ $('WhatsApp Trigger').item.json.messages[0].<media_type>.id }}` (replace `<media_type>` accordingly)  
     - Use WhatsApp API credentials  
   - Connect Message Type outputs to respective get audio/image/document URL nodes

7. **Create HTTP Request node ("download media")**  
   - Type: HTTP Request  
   - URL: `={{ $json.url }}` (from media URL nodes)  
   - Authentication: Use WhatsApp API Token in header  
   - Connect all get media URL nodes → download media

8. **Create Slack Upload Media node**  
   - Type: Slack  
   - Resource: File upload  
   - Channel ID: `={{ $('Message Type').item.json.id }}` (channel from Message Type)  
   - Use Slack OAuth credentials  
   - Connect download media → upload media

9. **Create Slack Send Message node ("Send Message in Channel")**  
   - Type: Slack  
   - Text: `={{ $('WhatsApp Trigger').item.json.messages[0].text.body }}`  
   - Channel ID: `={{ $json.id }}` (from Message Type output for text)  
   - Use Slack OAuth credentials  
   - Connect Message Type (text output) → Send Message in Channel

---

10. **Create Slack Trigger node**  
    - Type: Slack Trigger  
    - Trigger on message events in workspace  
    - Use Slack OAuth credentials  
    - Position: bottom-left (e.g., [-2700, 920])

11. **Create Slack Get Channel by ID node**  
    - Type: Slack  
    - Operation: Get channel info by channelId `={{ $json.channel }}` (from Slack Trigger)  
    - Use Slack OAuth credentials  
    - Connect Slack Trigger → Get Channel by ID

12. **Create Switch node ("Checking Message Type")**  
    - Type: Switch  
    - Rules:  
      - Output "media" if `$('Slack Trigger').item.json.files` is not empty  
      - Output "text" if `$('Slack Trigger').item.json.text` is not empty  
    - Connect Get Channel by ID → Checking Message Type

13. **Create Slack Get Media URL node**  
    - Type: Slack  
    - Operation: Get file info by fileId `={{ $('Slack Trigger').item.json.files[0].id }}`  
    - Use Slack OAuth credentials  
    - Connect Checking Message Type (media output) → Get Media URL

14. **Create HTTP Request node ("Download Media")**  
    - Type: HTTP Request  
    - URL: `={{ $json.url_private_download }}` (from Get Media URL)  
    - Authentication: Slack OAuth token in header  
    - Connect Get Media URL → Download Media

15. **Create WhatsApp Send Media node ("Send Message1")**  
    - Type: WhatsApp  
    - Operation: send  
    - Message type: document  
    - Media path: use binary data from Download Media node  
    - Phone number ID: set to your WhatsApp phone number ID  
    - Recipient phone number: `={{ $('Get Channel by ID').item.json.name }}` (Slack channel name)  
    - Use WhatsApp API credentials  
    - Connect Download Media → Send Message1

16. **Create WhatsApp Send Text node ("Send Message")**  
    - Type: WhatsApp  
    - Operation: send  
    - Text body: `={{ $('Slack Trigger').item.json.blocks[0].elements[0].elements[0].text }}`  
    - Phone number ID: your WhatsApp phone number ID  
    - Recipient phone number: `={{ $('Get Channel by ID').item.json.name }}`  
    - Use WhatsApp API credentials  
    - Connect Checking Message Type (text output) → Send Message

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Slack private channels are created per WhatsApp sender phone number, ensuring message segregation per contact.       | Best practice for chat management                                                             |
| WhatsApp media URLs expire quickly; immediate download after retrieval is recommended to avoid broken links.         | WhatsApp API documentation                                                                     |
| Slack API token requires scopes for channel creation, messaging, file upload, and reading channel info.               | Slack OAuth app configuration                                                                  |
| WhatsApp API credentials require phone number ID and OAuth-based access for sending and receiving messages.          | WhatsApp Business API setup                                                                     |
| Workflow includes error handling for channel creation to continue if channel already exists, preventing workflow halt.| Slack API returns error if channel exists; handled with "continue on error" option              |
| Media file size limits in Slack and WhatsApp may restrict very large files; consider adding size checks if needed.   | Slack and WhatsApp API limits                                                                  |
| Sticky notes within the workflow provide concise documentation of each flow direction for maintainability.           | Visible in n8n editor for user guidance                                                        |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.