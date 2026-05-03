WhatsApp Micro-CRM with Baserow & WasenderAPI

https://n8nworkflows.xyz/workflows/whatsapp-micro-crm-with-baserow---wasenderapi-6584


# WhatsApp Micro-CRM with Baserow & WasenderAPI

### 1. Workflow Overview

This workflow automates WhatsApp communication and contact management by integrating WasenderAPI with Baserow to create a micro-CRM system suitable for small businesses and freelancers. It processes incoming WhatsApp messages received via WasenderAPI webhooks, standardizes contact information, manages contact records, decrypts and uploads media content, and logs all interactions into structured Baserow tables.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Receive WhatsApp messages through a protected webhook trigger.
- **1.2 Data Preparation and Contact Management:** Standardize incoming data, extract phone numbers, and check/update or create contacts in Baserow.
- **1.3 Optional Contact Profile Picture Handling:** Fetch and store WhatsApp profile pictures for contacts.
- **1.4 Message Type Detection and Routing:** Determine if the message contains images or text and route processing accordingly.
- **1.5 Image Message Decryption and Logging:** Decrypt inbound/outbound image messages via WasenderAPI, upload images to Baserow, and log the message metadata.
- **1.6 Text Message Extraction and Logging:** Extract text message data, determine message origin, and log inbound/outbound text messages in Baserow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives WhatsApp message payloads securely through a webhook with header authentication.

**Nodes Involved:**  
- Trigger: WhatsApp Message

**Node Details:**  
- **Trigger: WhatsApp Message**  
  - Type: Webhook  
  - Role: Entry point that listens for incoming POST requests from WasenderAPI.  
  - Configuration:  
    - HTTP Method: POST  
    - Authentication: Header authentication using HTTP Header Auth credentials (`x-webhook-signature` recommended).  
    - Webhook path unique to the instance.  
  - Input: External HTTP requests from WasenderAPI.  
  - Output: JSON payload of WhatsApp message data.  
  - Edge Cases: Incorrect or missing authentication headers could cause unauthorized access or dropped messages.  
  - Requires active n8n webhook URL configuration and valid WasenderAPI webhook setup.

---

#### 2.2 Data Preparation and Contact Management

**Overview:**  
Standardizes incoming WhatsApp data, extracts a readable phone number, detects presence of image messages, searches for existing contacts in Baserow, and either updates or creates contact records accordingly.

**Nodes Involved:**  
- Prepare: Standardize Incoming Data  
- Baserow: Search Contact  
- Route: Contact Exists?  
- Create New Contact  
- Update Contact Timestamp

**Node Details:**

- **Prepare: Standardize Incoming Data**  
  - Type: Code (Python)  
  - Role: Parses the webhook payload to extract a readable phone number from the WhatsApp JID and detects if the message contains an image.  
  - Key Logic:  
    - Converts JID (e.g., `8615621124179@s.whatsapp.net`) to `+8615621124179`.  
    - Checks for `imageMessage` key in the message data.  
    - Adds `readable_phone_number`, `found_image`, and `image_url` to the item JSON.  
  - Input: Raw WhatsApp webhook JSON.  
  - Output: Enriched JSON with standardized fields.  
  - Edge Cases: Payload missing expected keys, malformed JID, or unexpected message structure.  
  - Version: Requires Python execution environment.

- **Baserow: Search Contact**  
  - Type: HTTP Request  
  - Role: Queries Baserow "Contacts" table using the readable phone number to find existing contacts.  
  - Configuration:  
    - Method: GET with query parameters `search={{ $json.readable_phone_number }}` and `user_field_names=true`.  
    - Headers: Authorization token for Baserow API.  
  - Input: Standardized item JSON.  
  - Output: Search results with contact count and records.  
  - Edge Cases: API rate limits, authentication failure, or network errors.

- **Route: Contact Exists?**  
  - Type: Switch  
  - Role: Routes based on whether the contact exists (`count > 0`) or is new (`count == 0`).  
  - Input: Search results JSON.  
  - Output: Two paths: New_Contact and Existing_Contact.

- **Create New Contact**  
  - Type: Baserow node  
  - Role: Creates a new contact record in Baserow with phone number, name, status "New", and timestamp.  
  - Input: From New_Contact route with standardized data.  
  - Configuration: Fields mapped from the standardized data node.  
  - Edge Cases: API errors, missing required fields.

- **Update Contact Timestamp**  
  - Type: Baserow node  
  - Role: Updates `last activity` timestamp field for existing contacts.  
  - Input: From Existing_Contact route, uses the contact ID from search results.  
  - Configuration: Updates only the timestamp field.  
  - Edge Cases: Row ID missing or changed, API errors.

---

#### 2.3 Optional Contact Profile Picture Handling

**Overview:**  
Fetches WhatsApp contact profile picture URLs via WasenderAPI and uploads them to Baserow, updating the contact record with the image file reference.

**Nodes Involved:**  
- WasenderAPI: Fetch Profile Picture URL  
- Baserow: Upload Profile Picture File  
- Update Contact Profile Picture Field

**Node Details:**

- **WasenderAPI: Fetch Profile Picture URL**  
  - Type: HTTP Request  
  - Role: Calls WasenderAPI endpoint to retrieve the profile picture URL for the contact's WhatsApp number.  
  - Configuration: GET request to `https://www.wasenderapi.com/api/contacts/{remoteJid}/picture` with Bearer token authorization.  
  - Input: Contact's WhatsApp JID from standardized data.  
  - Output: JSON containing image URL.  
  - Edge Cases: Missing picture, API limits, auth errors.

- **Baserow: Upload Profile Picture File**  
  - Type: HTTP Request  
  - Role: Uploads the profile picture to Baserow’s user files via URL upload API.  
  - Configuration: POST JSON body with `"url": "{{ $json.data.imgUrl }}"`, headers include Baserow API token and content-type.  
  - Input: Image URL from previous node.  
  - Output: Uploaded file metadata including file name.  
  - Edge Cases: Invalid URL, upload failures.

- **Update Contact Profile Picture Field**  
  - Type: Baserow node  
  - Role: Updates the contact record with the uploaded profile picture file name.  
  - Input: Uses the newly created contact ID and file metadata.  
  - Configuration: Sets the image field in the contact record.  
  - Edge Cases: API errors, missing row or file data.

This block is marked optional and can be removed if profile pictures are not needed.

---

#### 2.4 Message Type Detection and Routing

**Overview:**  
Determines if the incoming WhatsApp message is an image or text message and routes processing accordingly, including a special test route.

**Nodes Involved:**  
- Route: Message Type (Image/Text)  
- Extract: Image Message Data  
- Extract: Text Message Data  
- webhooktest (delete after testing)

**Node Details:**

- **Route: Message Type (Image/Text)**  
  - Type: Switch  
  - Role: Routes messages into three paths:  
    - Testing (if message contains a test flag)  
    - has_image (if an image message detected)  
    - txt_only (if no image)  
  - Input: Standardized message data with `found_image`.  
  - Output: Corresponding downstream processing node.

- **Extract: Image Message Data**  
  - Type: Set  
  - Role: Extracts relevant image message details such as message object, message ID, remoteJid, fromMe flag, readable phone number, and found_image boolean for further processing.  
  - Input: Routed image message data.  
  - Output: Structured JSON with extracted fields.

- **Extract: Text Message Data**  
  - Type: Set  
  - Role: Extracts text message details similar to the image extraction node but for text content.

- **webhooktest (delete after testing)**  
  - Type: NoOp (No operation)  
  - Role: Placeholder node for testing purposes, can be deleted after validation.

---

#### 2.5 Image Message Decryption and Logging

**Overview:**  
Decrypts outbound and inbound WhatsApp image messages via WasenderAPI, uploads decrypted images to Baserow, and logs message metadata (including image) into Baserow messages table.

**Nodes Involved:**  
- Route: Message Origin (Outbound/Inbound)1  
- WasenderAPI: Decrypt Image Outbound  
- WasenderAPI: Decrypt Image Inbound  
- Baserow Upload Message Image  
- Baserow_Upload Message Image  
- Log Outbound Image Message  
- Log Inbound Image Message

**Node Details:**

- **Route: Message Origin (Outbound/Inbound)1**  
  - Type: If  
  - Role: Determines if the image message is outbound (`fromMe == true`) or inbound (`fromMe == false`).  
  - Input: Extracted image message data.  
  - Output: Two paths to decrypt outbound or inbound.

- **WasenderAPI: Decrypt Image Outbound**  
  - Type: HTTP Request  
  - Role: Calls WasenderAPI media decryption endpoint for outbound images with message metadata in POST JSON body.  
  - Input: Outbound image message data.  
  - Output: Decrypted image URL and metadata.

- **WasenderAPI: Decrypt Image Inbound**  
  - Type: HTTP Request  
  - Role: Same as outbound decrypt but for inbound images.  
  - Input: Inbound image message data.  
  - Output: Decrypted image URL.

- **Baserow Upload Message Image** (for outbound)  
  - Type: HTTP Request  
  - Role: Uploads decrypted image URL to Baserow user files via URL upload.  
  - Input: Decrypted image public URL.  
  - Output: File metadata.

- **Baserow_Upload Message Image** (for inbound)  
  - Same as above but for inbound images.

- **Log Outbound Image Message**  
  - Type: Baserow node  
  - Role: Creates a new message record in Baserow "Messages" table for outbound image messages, including message ID, phone number, file reference, status "Unread", direction "Outbound", and timestamp.  
  - Input: Uploaded file metadata and extracted message data.  
  - Edge Cases: Missing timestamps, API errors.

- **Log Inbound Image Message**  
  - Same as above but for inbound image messages.

---

#### 2.6 Text Message Extraction and Logging

**Overview:**  
Extracts text message data, determines if the message is outbound or inbound, and logs the text message content into Baserow "Messages" table.

**Nodes Involved:**  
- Extract: Text Message Data  
- Route: Message Origin (Outbound/Inbound)  
- Log Outbound Text Message  
- Log Inbound Text Message

**Node Details:**

- **Extract: Text Message Data**  
  - Type: Set  
  - Role: Extracts message object, message ID, remoteJid, fromMe flag, and readable phone number for text messages.  
  - Input: Routed text messages.  
  - Output: Structured JSON for downstream processing.

- **Route: Message Origin (Outbound/Inbound)**  
  - Type: If  
  - Role: Checks if message is outbound or inbound based on `fromMe` boolean.  
  - Output: Routes to either outbound or inbound logging nodes.

- **Log Outbound Text Message**  
  - Type: Baserow node  
  - Role: Creates outbound text message record with message ID, phone number, status "Unread", direction "Outbound", timestamp, and message text extracted from conversation or reactionMessage.  
  - Edge Cases: Missing message text or timestamps.

- **Log Inbound Text Message**  
  - Same as above but for inbound messages.

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                             | Input Node(s)                        | Output Node(s)                            | Sticky Note                                                                                                                                                                                                                     |
|----------------------------------|---------------------|---------------------------------------------|------------------------------------|-------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note                      | Sticky Note         | Documentation overview                       | -                                  | -                                         | WhatsApp Mini-CRM with Baserow & WasenderAPI: automates WhatsApp communication and client management; setup instructions and requirements included.                                                                            |
| Sticky Note2                     | Sticky Note         | Documentation for contact management         | -                                  | -                                         | Manage Incoming Contacts: update last activity for existing, create new contact otherwise.                                                                                                                                     |
| Sticky Note3                     | Sticky Note         | Documentation for optional profile pictures  | -                                  | -                                         | Optional: Contact Profile Picture enhances CRM visual management; can be removed if undesired. Link to example gallery view included.                                                                                        |
| Sticky Note4                     | Sticky Note         | Documentation for message type detection     | -                                  | -                                         | Message Type Detection & Image Handling: decrypt images, upload to Baserow image fields.                                                                                                                                       |
| Sticky Note5                     | Sticky Note         | Documentation for text message logging       | -                                  | -                                         | Log only text messages here; IF node checks message origin from WasenderAPI number.                                                                                                                                             |
| Sticky Note6                     | Sticky Note         | Important setup note                          | -                                  | -                                         | For all messages, select `messages.upsert` event and recommend using header auth with `x-webhook-signature`.                                                                                                                  |
| Trigger: WhatsApp Message        | Webhook             | Entry point for WhatsApp messages             | -                                  | Prepare: Standardize Incoming Data         |                                                                                                                                                                                                                                 |
| Prepare: Standardize Incoming Data| Code (Python)      | Normalize WhatsApp payload and detect images | Trigger: WhatsApp Message           | Route: Message Type (Image/Text), Baserow: Search Contact |                                                                                                                                                                                                                                 |
| Baserow: Search Contact          | HTTP Request        | Search for existing contact by phone number  | Prepare: Standardize Incoming Data  | Route: Contact Exists?                      |                                                                                                                                                                                                                                 |
| Route: Contact Exists?           | Switch              | Branch for new or existing contact            | Baserow: Search Contact             | Create New Contact, Update Contact Timestamp |                                                                                                                                                                                                                                 |
| Create New Contact               | Baserow              | Create new contact record                      | Route: Contact Exists? (New_Contact) | WasenderAPI: Fetch Profile Picture URL      |                                                                                                                                                                                                                                 |
| Update Contact Timestamp         | Baserow              | Update existing contact's last activity       | Route: Contact Exists? (Existing_Contact) | -                                         |                                                                                                                                                                                                                                 |
| WasenderAPI: Fetch Profile Picture URL | HTTP Request | Retrieve contact profile picture URL          | Create New Contact                  | Baserow: Upload Profile Picture File       |                                                                                                                                                                                                                                 |
| Baserow: Upload Profile Picture File | HTTP Request     | Upload profile picture to Baserow              | WasenderAPI: Fetch Profile Picture URL | Update Contact Profile Picture Field        |                                                                                                                                                                                                                                 |
| Update Contact Profile Picture Field | Baserow           | Update contact record with profile picture    | Baserow: Upload Profile Picture File | -                                         |                                                                                                                                                                                                                                 |
| Route: Message Type (Image/Text) | Switch             | Route message based on content type           | Prepare: Standardize Incoming Data  | webhooktest, Extract: Image Message Data, Extract: Text Message Data |                                                                                                                                                                                                                                 |
| webhooktest (delete after testing) | NoOp              | Testing placeholder node                       | Route: Message Type (Image/Text)    | -                                         |                                                                                                                                                                                                                                 |
| Extract: Image Message Data      | Set                 | Extract key data from image messages           | Route: Message Type (Image/Text)    | Route: Message Origin (Outbound/Inbound)1  |                                                                                                                                                                                                                                 |
| Route: Message Origin (Outbound/Inbound)1 | If           | Determine if image message is outbound or inbound | Extract: Image Message Data         | WasenderAPI: Decrypt Image Outbound, WasenderAPI: Decrypt Image Inbound |                                                                                                                                                                                                                                 |
| WasenderAPI: Decrypt Image Outbound | HTTP Request     | Decrypt outbound image message media           | Route: Message Origin (Outbound/Inbound)1 (Outbound) | Baserow Upload Message Image                |                                                                                                                                                                                                                                 |
| WasenderAPI: Decrypt Image Inbound | HTTP Request      | Decrypt inbound image message media            | Route: Message Origin (Outbound/Inbound)1 (Inbound) | Baserow_Upload Message Image                 |                                                                                                                                                                                                                                 |
| Baserow Upload Message Image    | HTTP Request        | Upload decrypted outbound image to Baserow    | WasenderAPI: Decrypt Image Outbound | Log Outbound Image Message                   |                                                                                                                                                                                                                                 |
| Baserow_Upload Message Image    | HTTP Request        | Upload decrypted inbound image to Baserow     | WasenderAPI: Decrypt Image Inbound  | Log Inbound Image Message                     |                                                                                                                                                                                                                                 |
| Log Outbound Image Message       | Baserow             | Log outbound image message metadata            | Baserow Upload Message Image        | -                                         |                                                                                                                                                                                                                                 |
| Log Inbound Image Message        | Baserow             | Log inbound image message metadata             | Baserow_Upload Message Image        | -                                         |                                                                                                                                                                                                                                 |
| Extract: Text Message Data       | Set                 | Extract key data from text messages            | Route: Message Type (Image/Text)    | Route: Message Origin (Outbound/Inbound)   |                                                                                                                                                                                                                                 |
| Route: Message Origin (Outbound/Inbound) | If             | Determine if text message is outbound or inbound | Extract: Text Message Data           | Log Outbound Text Message, Log Inbound Text Message |                                                                                                                                                                                                                                 |
| Log Outbound Text Message        | Baserow             | Log outbound text message metadata             | Route: Message Origin (Outbound/Inbound) (Outbound) | -                                         |                                                                                                                                                                                                                                 |
| Log Inbound Text Message         | Baserow             | Log inbound text message metadata              | Route: Message Origin (Outbound/Inbound) (Inbound) | -                                         |                                                                                                                                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger Node**  
   - Type: Webhook  
   - Path: Unique path (e.g., `9218c291-18b1-44a6-9b1c-4df4ff7605f6`)  
   - HTTP method: POST  
   - Authentication: Header Auth  
   - Set credentials with HTTP Header Auth including expected header key (e.g., `x-webhook-signature`).  
   - This node receives WhatsApp messages from WasenderAPI.

2. **Add a Code Node to Standardize Incoming Data**  
   - Type: Code (Python)  
   - Paste the provided Python code to:  
     - Extract `readable_phone_number` from WhatsApp JID.  
     - Detect if the message contains an image and extract `image_url`.  
   - Connect the Webhook trigger output to this node.

3. **Add a HTTP Request Node to Search Contact in Baserow**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.baserow.io/api/database/rows/table/622539/`  
   - Query parameters:  
     - `user_field_names=true`  
     - `search={{ $json.readable_phone_number }}`  
   - Header:  
     - `Authorization: Token <your Baserow API token>`  
   - Connect the Standardize Data node output here.

4. **Add a Switch Node to Route Contact Existence**  
   - Type: Switch  
   - Condition: Check if `$json.count == 0` for new contacts (output "New_Contact")  
   - Else `$json.count > 0` for existing contacts (output "Existing_Contact")  
   - Connect Search Contact node output here.

5. **Create a Baserow Node to Create New Contact**  
   - Operation: Create  
   - Database ID: 264981 (adjust per your Baserow setup)  
   - Table ID: 622539 (Contacts table)  
   - Fields:  
     - Phone number: `={{ $json.readable_phone_number }}`  
     - Name: `={{ $json.body.data.messages.pushName }}`  
     - Status: `"New"`  
     - Last activity timestamp: `={{ $json.body.timestamp.toDateTime('ms') }}`  
   - Connect the "New_Contact" output from the switch.

6. **Create a Baserow Node to Update Existing Contact Timestamp**  
   - Operation: Update  
   - Database ID: 264981  
   - Table ID: 622539  
   - Row ID: `={{ $json.results[0].id }}` (from Search Contact result)  
   - Field to update: last activity timestamp to current time using expression  
   - Connect "Existing_Contact" output from the switch.

7. **(Optional) Add HTTP Request Node to Fetch Contact Profile Picture**  
   - Method: GET  
   - URL: `https://www.wasenderapi.com/api/contacts/{{ $json.body.data.messages.key.remoteJid }}/picture`  
   - Header: `Authorization: Bearer <your WasenderAPI token>`  
   - Connect output of Create New Contact node.

8. **Add HTTP Request Node to Upload Profile Picture to Baserow**  
   - Method: POST  
   - URL: `https://api.baserow.io/api/user-files/upload-via-url/`  
   - Body (JSON): `{"url": "{{ $json.data.imgUrl }}"}`  
   - Headers:  
     - `Authorization: Token <your Baserow API token>`  
     - `Content-Type: application/json`  
   - Connected to previous fetch profile picture node.

9. **Add Baserow Update Node to Set Profile Picture Field**  
   - Operation: Update  
   - Table ID: 622539  
   - Row ID: `={{ $('Create New Contact').item.json.id }}`  
   - Field: Profile picture field set to uploaded file name from previous node.  
   - Connect output of Upload Profile Picture node.

10. **Add a Switch Node to Route Message Type**  
    - Condition outputs:  
      - "Testing" if `$json.body.data.test == true`  
      - "has_image" if `$json.found_image == true`  
      - "txt_only" if `$json.found_image == false`  
    - Connect output of Standardize Incoming Data node.

11. **Add Set Nodes to Extract Image and Text Message Data**  
    - For image messages, extract message_object, message_id, remoteJid, fromMe, readable_phone_number, found_image.  
    - For text messages, extract similar fields without found_image.  
    - Connect respective outputs from message type switch.

12. **Add If Node to Route Image Message Origin**  
    - Condition: Check if `fromMe == true` (outbound) or false (inbound).  
    - Connect output of Extract Image Message Data.

13. **Add HTTP Request Nodes to Decrypt Images (Outbound and Inbound)**  
    - POST to `https://www.wasenderapi.com/api/decrypt-media`  
    - Body: JSON object with message key id, image message URL, mimetype, mediaKey from extracted data.  
    - Header: Authorization Bearer token from WasenderAPI.  
    - Connect respective outputs of message origin node.

14. **Add HTTP Request Nodes to Upload Decrypted Image URLs to Baserow**  
    - POST to `https://api.baserow.io/api/user-files/upload-via-url/`  
    - Body: JSON with `"url": "{{ $json.publicUrl }}"` from decrypted media response.  
    - Headers: Authorization token and Content-Type JSON.  
    - Connect from decrypt nodes.

15. **Add Baserow Nodes to Log Outbound and Inbound Image Messages**  
    - Operation: Create  
    - Table ID: 622533 (Messages table)  
    - Fields mapped from extracted data and uploaded file metadata: message_id, phone number, file name, status "Unread", direction (Inbound/Outbound), timestamp converted from metadata.  
    - Connect from respective upload image nodes.

16. **Add If Node for Text Message Origin**  
    - Similar to image origin node, check `fromMe` boolean.  
    - Connect from Extract Text Message Data node.

17. **Add Baserow Nodes to Log Outbound and Inbound Text Messages**  
    - Operation: Create  
    - Table ID: 622533  
    - Fields: message_id, phone number, status "Unread", direction, timestamp, and message text extracted from conversation or reactionMessage.  
    - Connect from text message origin node outputs.

18. **(Optional) Remove the webhooktest NoOp node after testing.**

19. **Credentials Setup**  
    - Configure Baserow API credentials with API token for HTTP Request and Baserow nodes.  
    - Configure WasenderAPI credentials with Bearer token for HTTP Request nodes accessing WasenderAPI endpoints.  
    - Configure HTTP Header Auth credentials for the webhook trigger to secure incoming requests.

20. **Set Up Baserow Tables**  
    - Duplicate "Contacts" and "Messages" tables from provided public Baserow templates.  
    - Adjust Table IDs in nodes accordingly if custom tables are used.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Setup is quick (under 15 min). Connect WasenderAPI webhooks, duplicate Baserow 'Contacts' and 'Messages' templates, and add API credentials in n8n. | Setup instructions in Sticky Note node 1 |
| For all WhatsApp message notifications, select the `messages.upsert` event in WasenderAPI for proper webhook triggering. | Sticky Note6: Important Note |
| Recommended to implement header authentication using `x-webhook-signature` header to secure webhook endpoint. | Sticky Note6 |
| Optional profile picture management enhances contact visualization, especially in Baserow gallery views. Example gallery: https://baserow.io/public/gallery/v7PfThVQLOIc6moc9j0Ebjq_Az4Nm3TMMDzpvF30Esk | Sticky Note3 |
| Baserow public templates for tables: Contacts - https://baserow.io/public/grid/a5iWkAQpu8QljUlgwgm_pour_Au5BKd3mtkfu-B6N7Y, Messages - https://baserow.io/public/grid/0H22XZitFDWnrVNnKwBfiI7M6XX5CugHrXHEzdCY4xY | Sticky Note1 |
| Keep the flow layout as is to ensure correct execution order in n8n. | Sticky Note1 |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. The processing strictly complies with current content policies and contains no illegal or protected material. All handled data is legal and public.