Create an auto-journal from WhatsApp with AI transcription

https://n8nworkflows.xyz/workflows/create-an-auto-journal-from-whatsapp-with-ai-transcription-13073


# Create an auto-journal from WhatsApp with AI transcription

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Purpose:**  
This workflow receives incoming WhatsApp messages (text, audio, image) via the WhatsApp Business Cloud API, filters senders by an allowlist of phone numbers, then logs the content into a single Google Doc. Audio messages are transcribed with **Google Gemini** (LangChain node), and both audio/images are uploaded to **Google Drive** with a link stored in the Google Doc. Finally, it sends a confirmation message back on WhatsApp.

**Target use cases:**
- Personal “auto-journal” or daily log from WhatsApp messages
- Lightweight capture of voice notes with transcription
- Centralized archive of images/audio with Drive links

### Logical blocks
1.1 **Input Reception (WhatsApp Trigger)**  
1.2 **Security Filter (Authorized phone numbers)**  
1.3 **Message Type Routing (text/audio/image)**  
1.4 **Text Flow (append text to Google Doc + confirm)**  
1.5 **Audio Flow (download → transcribe → upload to Drive → append to Doc → confirm)**  
1.6 **Image Flow (download → upload to Drive → append link to Doc → confirm)**

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception (WhatsApp Trigger)

**Overview:**  
Listens for inbound WhatsApp messages and forwards the payload into the workflow for validation and routing.

**Nodes involved:**
- **WhatsApp Trigger**
- (Sticky notes: **Trigger Group**)

**Node details**

#### WhatsApp Trigger
- **Type / role:** `n8n-nodes-base.whatsAppTrigger` — entry point webhook trigger for WhatsApp Cloud API events.
- **Configuration choices:**
  - Subscribed update type: **messages**
  - Uses a WhatsApp Trigger webhook in n8n (internally managed by WhatsApp node).
- **Key data used later:**
  - `contacts[0].wa_id` (sender phone)
  - `contacts[0].profile.name` (sender name)
  - `messages[0].type` (text/audio/image)
  - `messages[0].text.body`, `messages[0].audio.id`, `messages[0].image.id`
- **Connections:**
  - Output → **Authorized Numbers Check**
- **Failure / edge cases:**
  - WhatsApp webhook verification / subscription misconfiguration
  - Payload shape differences (e.g., missing `contacts`, multiple messages in one event)
  - Non-message events (if WhatsApp sends other updates; workflow only subscribes to messages)
- **Version notes:** Node typeVersion `1` (Cloud API trigger).

---

### 2.2 Security Filter (Authorized numbers)

**Overview:**  
Allows processing only if the sender’s WhatsApp ID matches one of three configured phone numbers.

**Nodes involved:**
- **Authorized Numbers Check**
- (Sticky notes: **Security Group**)

**Node details**

#### Authorized Numbers Check
- **Type / role:** `n8n-nodes-base.if` — conditional gatekeeper.
- **Configuration choices:**
  - Condition group: **OR**
  - Compares `{{ $json.contacts[0].wa_id }}` to three hardcoded allowlist entries:
    - `YOURNUMBERPHONE(1)`
    - `YOURNUMBERPHONE(2)`
    - `YOURNUMBERPHONE(3)`
- **Connections:**
  - True output → **Route by Message Type**
  - False output → **(not connected)** → unauthorized messages are silently dropped.
- **Failure / edge cases:**
  - If `contacts[0].wa_id` is missing, expression can fail or evaluate empty → sender blocked.
  - Phone format must match WhatsApp `wa_id` (international format, typically without `+`).
  - Consider adding a “false path” to send an “unauthorized” reply or log attempts.

---

### 2.3 Message Type Routing

**Overview:**  
Routes authorized messages into one of three branches based on `messages[0].type`.

**Nodes involved:**
- **Route by Message Type**
- (Sticky notes: **Router Group**)

**Node details**

#### Route by Message Type
- **Type / role:** `n8n-nodes-base.switch` — multi-branch router.
- **Configuration choices:**
  - Rules (renamed outputs):
    - **Text** when `{{ $json.messages[0].type }}` equals `"text"`
    - **Audio** when equals `"audio"`
    - **Image** when equals `"image"`
- **Connections:**
  - Text → **Save Text to Google Doc**
  - Audio → **Get Audio URL**
  - Image → **Get Image URL**
- **Failure / edge cases:**
  - Other WhatsApp types (e.g., `document`, `video`, `sticker`, `reaction`) are not handled → message dropped.
  - If `messages[0]` is absent (rare but possible in malformed events), switch evaluation fails.

---

### 2.4 Text Flow (Google Doc append + WhatsApp confirmation)

**Overview:**  
Appends the text message into a Google Doc with a timestamp and sender name, then confirms back to the sender.

**Nodes involved:**
- **Save Text to Google Doc**
- **Send Text Confirmation**
- (Sticky notes: **Text Processing Group**)

**Node details**

#### Save Text to Google Doc
- **Type / role:** `n8n-nodes-base.googleDocs` — updates an existing Google Doc by inserting text.
- **Configuration choices:**
  - Operation: **update**
  - Action: **insert** (appends a formatted block)
  - Authentication: **Google Service Account**
  - Document URL: placeholder `YOUR_GOOGLE_DOC_URL` (must be replaced)
- **Key expressions:**
  - Timestamp: `{{ $now.format('DD HH:mm:ss') }}`
  - Sender name: `{{ $('WhatsApp Trigger').item.json.contacts[0].profile.name }}`
  - Text body: `{{ $('WhatsApp Trigger').item.json.messages[0].text.body }}`
- **Connections:**
  - Input ← **Route by Message Type (Text)**
  - Output → **Send Text Confirmation**
- **Failure / edge cases:**
  - Service account missing access to the target Doc (must share the Doc with the service account email).
  - If sender name or text body is missing, insert content may be blank or expression error.
  - Document URL invalid or points to an inaccessible doc.

#### Send Text Confirmation
- **Type / role:** `n8n-nodes-base.whatsApp` — sends a WhatsApp outbound message.
- **Configuration choices:**
  - Operation: **send**
  - Text body: `😉 Document updated!`
  - `phoneNumberId`: `YOUR_WHATSAPP_PHONE_NUMBER_ID` (must be replaced)
  - Recipient: `{{ $('WhatsApp Trigger').item.json.contacts[0].wa_id }}`
- **Connections:**
  - Input ← **Save Text to Google Doc**
- **Failure / edge cases:**
  - WhatsApp credentials / permissions / number ID incorrect.
  - Recipient phone formatting mismatch.
  - Rate limiting or template requirements (depending on WhatsApp account status and session window).

---

### 2.5 Audio Flow (download → transcribe → upload → doc → confirm)

**Overview:**  
Retrieves the audio media URL from WhatsApp, downloads it, transcribes it using Google Gemini, re-downloads as binary for Drive upload, then inserts both Drive link and transcription into the Google Doc and confirms back to the sender.

**Nodes involved:**
- **Get Audio URL**
- **Download Audio File**
- **Transcribe Audio with AI**
- **Convert Audio to Binary**
- **Upload Audio to Drive**
- **Save Audio Transcription to Google Doc**
- **Send Audio Confirmation**
- (Sticky notes: **Audio Processing Group**)

**Node details**

#### Get Audio URL
- **Type / role:** `n8n-nodes-base.whatsApp` — calls WhatsApp media endpoint to obtain a temporary download URL.
- **Configuration choices:**
  - Resource: **media**
  - Operation: **mediaUrlGet**
  - Media ID: `{{ $json.messages[0].audio.id }}`
- **Connections:**
  - Input ← **Route by Message Type (Audio)**
  - Output → **Download Audio File**
- **Failure / edge cases:**
  - Missing `messages[0].audio.id` (non-audio payload or unexpected structure)
  - Expired/invalid media ID
  - WhatsApp token permission issues

#### Download Audio File
- **Type / role:** `n8n-nodes-base.httpRequest` — downloads the audio file from WhatsApp-hosted URL.
- **Configuration choices:**
  - URL: `{{ $json.url }}` (from **Get Audio URL**)
  - Auth: **predefinedCredentialType** = `whatsAppApi`
- **Connections:**
  - Input ← **Get Audio URL**
  - Output → **Transcribe Audio with AI**
- **Failure / edge cases:**
  - URL expires quickly (WhatsApp media URLs are short-lived)
  - Response not treated as binary depending on node settings (workflow later uses Gemini “binary input”; ensure the HTTP node outputs binary as expected in your n8n version)
  - Large audio may cause timeouts/memory pressure

#### Transcribe Audio with AI
- **Type / role:** `@n8n/n8n-nodes-langchain.googleGemini` — Gemini audio transcription.
- **Configuration choices:**
  - Resource: **audio**
  - Model: `models/gemini-2.5-flash`
  - Input type: **binary**
  - `simplify: false` (keeps the full Gemini response structure)
  - `alwaysOutputData: true` (continues even if the node has issues; may output empty structure)
- **Connections:**
  - Input ← **Download Audio File**
  - Output → **Convert Audio to Binary**
- **Key output used later:**
  - Transcription path: `candidates[0].content.parts[0].text`
- **Failure / edge cases:**
  - Missing/incorrect Gemini credentials
  - Audio encoding not supported / corrupted downloads
  - Response shape may differ; relying on `candidates[0]...` can break if Gemini returns no candidates or an error object

#### Convert Audio to Binary
- **Type / role:** `n8n-nodes-base.httpRequest` — downloads the audio again, intended to ensure proper binary for Drive upload.
- **Configuration choices:**
  - URL: `{{ $('Get Audio URL').item.json.url }}`
  - Auth: predefined `whatsAppApi`
- **Connections:**
  - Input ← **Transcribe Audio with AI**
  - Output → **Upload Audio to Drive**
- **Failure / edge cases:**
  - Same short-lived URL expiry risk (and now it’s used later than the first download)
  - This is redundant if **Download Audio File** already provides correct binary; but it can be a workaround depending on how Gemini consumed the binary

#### Upload Audio to Drive
- **Type / role:** `n8n-nodes-base.googleDrive` — uploads audio into a specific Drive folder.
- **Configuration choices:**
  - File name: `{{ $now.format('DD HH:mm:ss') }}.ogg`
  - Drive: “My Drive”
  - Folder: `YOUR_AUDIO_FOLDER_ID` (must be replaced)
  - Input binary field: `data` (set via `inputDataFieldName = =data`)
- **Connections:**
  - Input ← **Convert Audio to Binary**
  - Output → **Save Audio Transcription to Google Doc**
- **Output used later:**
  - `webViewLink` (Drive link to file)
- **Failure / edge cases:**
  - Service account permission to folder (folder must be shared with service account)
  - Binary field mismatch (`data` not present) → upload fails
  - File extension `.ogg` may not match actual codec/format

#### Save Audio Transcription to Google Doc
- **Type / role:** `n8n-nodes-base.googleDocs` — appends Drive link + transcription text to the Doc.
- **Configuration choices:**
  - Operation: update → insert
  - Document URL: `YOUR_GOOGLE_DOC_URL`
  - Auth: service account
- **Key expressions:**
  - Drive link: `{{ $json.webViewLink }}` (from Drive upload node output)
  - Transcript: `{{ $('Transcribe Audio with AI').item.json.candidates[0].content.parts[0].text }}`
  - Timestamp & sender: same pattern as text flow
- **Connections:**
  - Input ← **Upload Audio to Drive**
  - Output → **Send Audio Confirmation**
- **Failure / edge cases:**
  - Transcript path missing leads to expression failure or empty content
  - Google Docs insert limits for very long transcripts

#### Send Audio Confirmation
- **Type / role:** `n8n-nodes-base.whatsApp` — confirms completion.
- **Configuration choices:**
  - Text body: `😊 Document updated and audio uploaded!`
  - Recipient: sender `wa_id`
  - `phoneNumberId`: `YOUR_WHATSAPP_PHONE_NUMBER_ID`
- **Connections:**
  - Input ← **Save Audio Transcription to Google Doc**
- **Failure / edge cases:** same as other WhatsApp send node.

---

### 2.6 Image Flow (download → upload → doc → confirm)

**Overview:**  
Fetches the WhatsApp image URL, downloads it, uploads it to Google Drive, inserts the Drive link into the Google Doc, then sends confirmation.

**Nodes involved:**
- **Get Image URL**
- **Download Image File**
- **Upload Image to Drive**
- **Save Image Link to Google Doc**
- **Send Image Confirmation**
- (Sticky notes: **Image Processing Group**)

**Node details**

#### Get Image URL
- **Type / role:** `n8n-nodes-base.whatsApp` — retrieve WhatsApp media download URL.
- **Configuration choices:**
  - Resource: media
  - Operation: mediaUrlGet
  - Media ID: `{{ $json.messages[0].image.id }}`
- **Connections:**
  - Input ← **Route by Message Type (Image)**
  - Output → **Download Image File**
- **Failure / edge cases:** invalid/expired media ID, missing field.

#### Download Image File
- **Type / role:** `n8n-nodes-base.httpRequest` — download image bytes from WhatsApp URL.
- **Configuration choices:**
  - URL: `{{ $json.url }}`
  - Auth: predefined `whatsAppApi`
- **Connections:**
  - Input ← **Get Image URL**
  - Output → **Upload Image to Drive**
- **Failure / edge cases:**
  - URL expiry, binary handling, large image.

#### Upload Image to Drive
- **Type / role:** `n8n-nodes-base.googleDrive` — upload image to Drive folder.
- **Configuration choices:**
  - File name: `{{ $now.format('DD HH:mm:ss') }}.jpg`
  - Folder: `YOUR_IMAGE_FOLDER_ID` (must be replaced)
  - Drive: My Drive
- **Connections:**
  - Input ← **Download Image File**
  - Output → **Save Image Link to Google Doc**
- **Failure / edge cases:**
  - Service account access to folder
  - If downloaded image is not JPG, extension mismatches content-type.

#### Save Image Link to Google Doc
- **Type / role:** `n8n-nodes-base.googleDocs` — insert a block containing Drive link.
- **Configuration choices:**
  - Operation: update → insert
  - Document URL: `YOUR_GOOGLE_DOC_URL`
- **Key expressions:**
  - Link: `{{ $json.webViewLink }}`
  - Timestamp & sender name from trigger
- **Connections:**
  - Input ← **Upload Image to Drive**
  - Output → **Send Image Confirmation**
- **Failure / edge cases:** Doc permissions, missing `webViewLink`.

#### Send Image Confirmation
- **Type / role:** `n8n-nodes-base.whatsApp` — sends confirmation message.
- **Configuration choices:**
  - Text body: `😃 Document updated and image uploaded!`
  - Recipient: sender `wa_id`
  - `phoneNumberId`: `YOUR_WHATSAPP_PHONE_NUMBER_ID`
- **Connections:**
  - Input ← **Save Image Link to Google Doc**

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Main Instructions | Sticky Note | Documentation / setup guidance | — | — | # WhatsApp Auto-Journal with AI Transcription… (setup steps and placeholders) |
| Trigger Group | Sticky Note | Block label: input | — | — | ## WhatsApp Input Receives all incoming WhatsApp messages. |
| Security Group | Sticky Note | Block label: security | — | — | ## Security Filter Validates incoming messages against authorized phone numbers only. |
| Router Group | Sticky Note | Block label: routing | — | — | ## Message Type Router Routes messages based on content type (text, audio, or image). |
| Text Processing Group | Sticky Note | Block label: text flow | — | — | ## Text Flow Processes text messages and saves them to Google Doc. |
| Audio Processing Group | Sticky Note | Block label: audio flow | — | — | ## Audio Flow Downloads audio, transcribes with AI, uploads to Drive, and saves transcription to Doc. |
| Image Processing Group | Sticky Note | Block label: image flow | — | — | ## Image Flow Downloads image, uploads to Drive, and saves link to Doc. |
| WhatsApp Trigger | WhatsApp Trigger | Receive inbound WhatsApp messages | — | Authorized Numbers Check | ## WhatsApp Input Receives all incoming WhatsApp messages. |
| Authorized Numbers Check | IF | Allowlist sender validation | WhatsApp Trigger | Route by Message Type | ## Security Filter Validates incoming messages against authorized phone numbers only. |
| Route by Message Type | Switch | Branch by message type | Authorized Numbers Check | Save Text to Google Doc; Get Audio URL; Get Image URL | ## Message Type Router Routes messages based on content type (text, audio, or image). |
| Save Text to Google Doc | Google Docs | Append text message to Doc | Route by Message Type | Send Text Confirmation | ## Text Flow Processes text messages and saves them to Google Doc. |
| Send Text Confirmation | WhatsApp | Confirm text saved | Save Text to Google Doc | — | ## Text Flow Processes text messages and saves them to Google Doc. |
| Get Audio URL | WhatsApp | Fetch media URL for audio | Route by Message Type | Download Audio File | ## Audio Flow Downloads audio, transcribes with AI, uploads to Drive, and saves transcription to Doc. |
| Download Audio File | HTTP Request | Download audio from URL | Get Audio URL | Transcribe Audio with AI | ## Audio Flow Downloads audio, transcribes with AI, uploads to Drive, and saves transcription to Doc. |
| Transcribe Audio with AI | Google Gemini (LangChain) | Transcribe audio binary | Download Audio File | Convert Audio to Binary | ## Audio Flow Downloads audio, transcribes with AI, uploads to Drive, and saves transcription to Doc. |
| Convert Audio to Binary | HTTP Request | Re-download audio as binary for Drive upload | Transcribe Audio with AI | Upload Audio to Drive | ## Audio Flow Downloads audio, transcribes with AI, uploads to Drive, and saves transcription to Doc. |
| Upload Audio to Drive | Google Drive | Upload audio file and produce share link | Convert Audio to Binary | Save Audio Transcription to Google Doc | ## Audio Flow Downloads audio, transcribes with AI, uploads to Drive, and saves transcription to Doc. |
| Save Audio Transcription to Google Doc | Google Docs | Append audio link + transcript to Doc | Upload Audio to Drive | Send Audio Confirmation | ## Audio Flow Downloads audio, transcribes with AI, uploads to Drive, and saves transcription to Doc. |
| Send Audio Confirmation | WhatsApp | Confirm audio logged | Save Audio Transcription to Google Doc | — | ## Audio Flow Downloads audio, transcribes with AI, uploads to Drive, and saves transcription to Doc. |
| Get Image URL | WhatsApp | Fetch media URL for image | Route by Message Type | Download Image File | ## Image Flow Downloads image, uploads to Drive, and saves link to Doc. |
| Download Image File | HTTP Request | Download image from URL | Get Image URL | Upload Image to Drive | ## Image Flow Downloads image, uploads to Drive, and saves link to Doc. |
| Upload Image to Drive | Google Drive | Upload image file and produce share link | Download Image File | Save Image Link to Google Doc | ## Image Flow Downloads image, uploads to Drive, and saves link to Doc. |
| Save Image Link to Google Doc | Google Docs | Append image link to Doc | Upload Image to Drive | Send Image Confirmation | ## Image Flow Downloads image, uploads to Drive, and saves link to Doc. |
| Send Image Confirmation | WhatsApp | Confirm image logged | Save Image Link to Google Doc | — | ## Image Flow Downloads image, uploads to Drive, and saves link to Doc. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   `Create an auto-journal from WhatsApp with AI transcription`

2. **Add WhatsApp Trigger**
   - Node: **WhatsApp Trigger**
   - Updates: **messages**
   - Configure WhatsApp Cloud API credentials in n8n (Meta app + webhook setup).
   - This becomes the **entry node**.

3. **Add IF node for allowlist**
   - Node: **IF** named `Authorized Numbers Check`
   - Condition: OR
   - Left value (each rule): `{{ $json.contacts[0].wa_id }}`
   - Right values: replace with your authorized numbers:
     - `YOURNUMBERPHONE(1)`, `YOURNUMBERPHONE(2)`, `YOURNUMBERPHONE(3)`
   - Connect: **WhatsApp Trigger → Authorized Numbers Check (main)**

4. **Add Switch node for message types**
   - Node: **Switch** named `Route by Message Type`
   - Add 3 rules (each equals on `{{ $json.messages[0].type }}`):
     - Output “Text” when `text`
     - Output “Audio” when `audio`
     - Output “Image” when `image`
   - Connect: **Authorized Numbers Check (true) → Route by Message Type**

### Text branch
5. **Add Google Docs node (text append)**
   - Node: **Google Docs** named `Save Text to Google Doc`
   - Authentication: **Service Account**
   - Operation: **Update**
   - Action: **Insert text**
   - Document URL: replace `YOUR_GOOGLE_DOC_URL`
   - Insert text template (same structure):
     - timestamp: `{{ $now.format('DD HH:mm:ss') }}`
     - name: `{{ $('WhatsApp Trigger').item.json.contacts[0].profile.name }}`
     - body: `{{ $('WhatsApp Trigger').item.json.messages[0].text.body }}`
   - Connect: **Route by Message Type (Text) → Save Text to Google Doc**

6. **Add WhatsApp send confirmation (text)**
   - Node: **WhatsApp** named `Send Text Confirmation`
   - Operation: **send**
   - `phoneNumberId`: replace `YOUR_WHATSAPP_PHONE_NUMBER_ID`
   - Recipient: `{{ $('WhatsApp Trigger').item.json.contacts[0].wa_id }}`
   - Text: `😉 Document updated!`
   - Connect: **Save Text to Google Doc → Send Text Confirmation**

### Audio branch
7. **Add WhatsApp “Get media URL” (audio)**
   - Node: **WhatsApp** named `Get Audio URL`
   - Resource: **media**
   - Operation: **mediaUrlGet**
   - Media ID: `{{ $json.messages[0].audio.id }}`
   - Connect: **Route by Message Type (Audio) → Get Audio URL**

8. **Add HTTP Request (download audio)**
   - Node: **HTTP Request** named `Download Audio File`
   - URL: `{{ $json.url }}`
   - Authentication: **Predefined credential type**
   - Credential type: `whatsAppApi`
   - Ensure it outputs binary (adjust “Response Format” / “Download” options depending on n8n version).
   - Connect: **Get Audio URL → Download Audio File**

9. **Add Google Gemini transcription node**
   - Node: **Google Gemini** (LangChain) named `Transcribe Audio with AI`
   - Resource: **audio**
   - Model: `models/gemini-2.5-flash`
   - Input type: **binary**
   - Simplify: **off** (false)
   - Configure Gemini credentials (API key / Google AI credentials as required by your n8n node version).
   - Connect: **Download Audio File → Transcribe Audio with AI**

10. **Add HTTP Request (re-download for binary upload)**
   - Node: **HTTP Request** named `Convert Audio to Binary`
   - URL: `{{ $('Get Audio URL').item.json.url }}`
   - Auth: predefined `whatsAppApi`
   - Make sure output binary field is named `data` or adjust the Drive node accordingly.
   - Connect: **Transcribe Audio with AI → Convert Audio to Binary**

11. **Add Google Drive upload (audio)**
   - Node: **Google Drive** named `Upload Audio to Drive`
   - Operation: **Upload**
   - File name: `{{ $now.format('DD HH:mm:ss') }}.ogg`
   - Folder ID: replace `YOUR_AUDIO_FOLDER_ID`
   - Authentication: **Service Account**
   - Input data field name: `data`
   - Connect: **Convert Audio to Binary → Upload Audio to Drive**

12. **Add Google Docs node (audio append)**
   - Node: **Google Docs** named `Save Audio Transcription to Google Doc`
   - Operation: Update → Insert
   - Document URL: `YOUR_GOOGLE_DOC_URL`
   - Insert text includes:
     - Drive link: `{{ $json.webViewLink }}`
     - Transcript: `{{ $('Transcribe Audio with AI').item.json.candidates[0].content.parts[0].text }}`
   - Connect: **Upload Audio to Drive → Save Audio Transcription to Google Doc**

13. **Add WhatsApp send confirmation (audio)**
   - Node: **WhatsApp** named `Send Audio Confirmation`
   - Operation: send
   - `phoneNumberId`: `YOUR_WHATSAPP_PHONE_NUMBER_ID`
   - Recipient: sender `wa_id`
   - Text: `😊 Document updated and audio uploaded!`
   - Connect: **Save Audio Transcription to Google Doc → Send Audio Confirmation**

### Image branch
14. **Add WhatsApp “Get media URL” (image)**
   - Node: **WhatsApp** named `Get Image URL`
   - Resource: media
   - Operation: mediaUrlGet
   - Media ID: `{{ $json.messages[0].image.id }}`
   - Connect: **Route by Message Type (Image) → Get Image URL**

15. **Add HTTP Request (download image)**
   - Node: **HTTP Request** named `Download Image File`
   - URL: `{{ $json.url }}`
   - Auth: predefined `whatsAppApi`
   - Ensure binary output.
   - Connect: **Get Image URL → Download Image File**

16. **Add Google Drive upload (image)**
   - Node: **Google Drive** named `Upload Image to Drive`
   - File name: `{{ $now.format('DD HH:mm:ss') }}.jpg`
   - Folder ID: `YOUR_IMAGE_FOLDER_ID`
   - Auth: service account
   - Connect: **Download Image File → Upload Image to Drive**

17. **Add Google Docs node (image link append)**
   - Node: **Google Docs** named `Save Image Link to Google Doc`
   - Insert includes Drive `{{ $json.webViewLink }}`
   - Document URL: `YOUR_GOOGLE_DOC_URL`
   - Connect: **Upload Image to Drive → Save Image Link to Google Doc**

18. **Add WhatsApp send confirmation (image)**
   - Node: **WhatsApp** named `Send Image Confirmation`
   - Text: `😃 Document updated and image uploaded!`
   - Recipient: sender `wa_id`
   - Connect: **Save Image Link to Google Doc → Send Image Confirmation**

19. **Credentials & access checks**
   - Share the Google Doc and the two Drive folders with the **service account email**.
   - Confirm WhatsApp Cloud API webhook is subscribed to messages and points to your n8n instance.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automatically save your WhatsApp messages, voice notes, and images to a Google Doc with AI-powered transcription. | Workflow purpose (from “Main Instructions”) |
| Replace placeholders: `YOURNUMBERPHONE(1/2/3)`, `YOUR_GOOGLE_DOC_URL`, `YOUR_AUDIO_FOLDER_ID`, `YOUR_IMAGE_FOLDER_ID`, `YOUR_WHATSAPP_PHONE_NUMBER_ID`. | Required setup (from “Main Instructions”) |
| Processing rules: Text saved directly; Audio transcribed via Gemini + uploaded to Drive + transcript saved; Images uploaded to Drive + link saved. | Functional behavior (from “Main Instructions”) |
| Test by sending a message to your WhatsApp Business number. | Validation step (from “Main Instructions”) |