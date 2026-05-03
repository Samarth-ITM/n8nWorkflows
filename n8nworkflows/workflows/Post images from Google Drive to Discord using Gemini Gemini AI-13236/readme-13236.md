Post images from Google Drive to Discord using Gemini Gemini AI

https://n8nworkflows.xyz/workflows/post-images-from-google-drive-to-discord-using-gemini-gemini-ai-13236


# Post images from Google Drive to Discord using Gemini Gemini AI

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Post images from Google Drive to Discord using Gemini Gemini AI

**Purpose:**  
Automatically watch a Google Drive folder for newly uploaded images, download the file, ask Google Gemini (via n8n’s LangChain Agent) to generate a Discord-ready **thread title**, **engagement description**, and **new filename**, then publish the image + text to Discord as a **new thread**. Finally, it **renames** and **moves** the processed file to a “Processed” Drive folder.

**Target use cases:**
- Keeping a Discord community active with AI-generated discussion prompts
- Automating content ops (capture → analyze → publish → archive)
- Turning technical images (code screenshots, mockups) into marketing-friendly posts

### Logical blocks
1. **1.1 Trigger & Context Setup (Google Drive → internal fields)**
2. **1.2 File Retrieval (Download image binary)**
3. **1.3 AI Analysis & Structured Generation (Gemini + structured JSON)**
4. **1.4 Merge & Post Payload Assembly**
5. **1.5 Discord Publishing (create thread with image + copy)**
6. **1.6 Drive Cleanup (rename + move to processed folder)**

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Context Setup (Google Drive → internal fields)

**Overview:**  
Detects new files created in a specific Google Drive folder and normalizes fields needed downstream (file name/id, Discord channel info, processed folder id).

**Nodes involved:**
- **Check For A New File to post on laughing-everyday** (Google Drive Trigger)
- **Get File & Set Channel** (Set)

#### Node: Check For A New File to post on laughing-everyday
- **Type / role:** `googleDriveTrigger` — polls Drive and emits an item when a new file is created.
- **Configuration choices:**
  - **Event:** `fileCreated`
  - **Trigger scope:** `specificFolder`
  - **Polling:** every minute
  - **Folder to watch:** currently empty (`""`) → must be set to your Input folder ID.
- **Inputs/Outputs:** Entry node → outputs file metadata (not binary).
- **Credentials:** Google Drive OAuth2 (“Google Drive account”)
- **Edge cases / failures:**
  - Folder ID missing/invalid → trigger never fires or errors.
  - OAuth scope/permission issues → cannot list changes.
  - High upload volume + polling → may miss rapid changes depending on Drive API and polling behavior.

#### Node: Get File & Set Channel
- **Type / role:** `set` — maps Drive trigger output to the workflow’s expected variables.
- **Configuration choices (assignments):**
  - `file_name = {{$json.name}}`
  - `file_id = {{$json.id}}`
  - `channel_id = "channel_id"` *(hardcoded placeholder)*
  - `to_channel = "to_channel"` *(hardcoded placeholder)*
  - `processed_folder_id = "processed_folder_id"` *(hardcoded placeholder)*
- **Inputs/Outputs:** Takes trigger item → outputs enriched JSON for downstream nodes.
- **Key notes:**
  - `channel_id`, `to_channel`, `processed_folder_id` must be replaced with real values (or expressions).
  - `to_channel` is carried through but not actually used by the Discord HTTP node in this workflow.
- **Edge cases:**
  - Leaving placeholders breaks Discord posting and Drive move steps later.

---

### 2.2 File Retrieval (Download image binary)

**Overview:**  
Downloads the newly detected file from Drive so the image binary can be passed to AI (for vision analysis) and later uploaded to Discord.

**Nodes involved:**
- **Download file** (Google Drive)

#### Node: Download file
- **Type / role:** `googleDrive` (operation: download) — fetches the binary content.
- **Configuration choices:**
  - **File ID:** `={{ $json.file_id }}`
  - **Operation:** `download`
- **Inputs/Outputs:**
  - **Input:** item containing `file_id`
  - **Output:** item includes **binary** data (field typically named `data` in n8n’s Google Drive download output) + metadata
  - Feeds:
    - **AI Agent** (main input)
    - **Merge** (main input index 0)
- **Credentials:** Google Drive OAuth2 (“Template”)
- **Edge cases / failures:**
  - File permissions / token expired → 401/403 errors
  - Non-image file uploaded → AI vision may fail or produce irrelevant output; Discord upload may still work but content may be wrong
  - Large files may cause memory/timeouts depending on n8n limits

---

### 2.3 AI Analysis & Structured Generation (Gemini + structured JSON)

**Overview:**  
Sends the image + prompt to a LangChain AI Agent using Google Gemini chat model, forcing a strict JSON output schema (title, description, new file name).

**Nodes involved:**
- **AI Agent** (LangChain Agent)
- **Chat Model** (Google Gemini)
- **Structured Output** (Structured Output Parser)

#### Node: Chat Model
- **Type / role:** `lmChatGoogleGemini` — provides the LLM backend for the agent.
- **Configuration choices:** default options (none specified).
- **Connections:**
  - Connected to **AI Agent** via `ai_languageModel`.
- **Credentials:** Google PaLM / Gemini credential (“Template”)
- **Edge cases:**
  - Model access not enabled / billing not set → auth or quota errors
  - Safety filters can refuse certain content depending on image/prompt

#### Node: Structured Output
- **Type / role:** `outputParserStructured` — enforces/validates the model output as JSON.
- **Configuration choices:**
  - Schema example:
    ```json
    { "title": "String", "description": "String", "new_file_name": "String" }
    ```
- **Connections:**
  - Connected to **AI Agent** via `ai_outputParser`.
- **Edge cases:**
  - If the model returns non-JSON or mismatched keys/types, parsing fails and the workflow stops unless error handling is added.

#### Node: AI Agent
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates the prompt + image context with the model and produces structured output.
- **Configuration choices:**
  - **Prompt text** references the file name: `{{$json.file_name}}`
  - Instructs: analyze image with IT + marketing lens; generate SEO title; description under 200 words with icons; include question; line breaks; output strict JSON.
  - **System message** defines role, constraints, and exact output JSON format.
  - **Has structured output parser enabled:** yes.
- **Inputs/Outputs:**
  - **Main input:** from **Download file** (includes binary image + metadata)
  - **Main output:** goes to **Merge** (index 1)
  - Output JSON is expected under `output` (as later referenced by Set node):  
    - `{{$json.output.title}}`, `{{$json.output.description}}`, `{{$json.output.new_file_name}}`
- **Edge cases:**
  - If the agent doesn’t receive the binary image field as expected, it may behave like “text-only” and produce generic content.
  - Prompt contains: `Ensure the output strictly follows the JSON schema provided in your system instructions."` (extra quote at end). Usually harmless, but can slightly degrade compliance.

---

### 2.4 Merge & Post Payload Assembly

**Overview:**  
Combines the downloaded binary (for Discord upload) with the AI-generated fields (title/description/new name) into a single item for posting and cleanup.

**Nodes involved:**
- **Merge** (Merge by position)
- **Get Downloaded File & Set Post Data** (Set)

#### Node: Merge
- **Type / role:** `merge` — combines two incoming streams into one item.
- **Configuration choices:**
  - **Mode:** `combine`
  - **Combine by:** `position` (first item from input 0 + first item from input 1)
- **Inputs/Outputs:**
  - **Input 0:** from **Download file** (binary)
  - **Input 1:** from **AI Agent** (structured output)
  - **Output:** to **Get Downloaded File & Set Post Data**
- **Edge cases:**
  - If one branch produces 0 items (AI failure, download failure), merge outputs nothing and downstream nodes won’t run.
  - If multiple files arrive quickly, position-based merging can mismatch items if concurrency/batching occurs (rare here, but possible).

#### Node: Get Downloaded File & Set Post Data
- **Type / role:** `set` — constructs final fields for Discord + Drive operations while keeping binary data.
- **Configuration choices:**
  - Keeps binary field `data` (explicitly included)
  - Assigns:
    - `file_id = {{$json.file_id}}`
    - `channel_id = {{$json.channel_id}}`
    - `to_channel = {{$json.to_channel}}` (not used later)
    - `title = {{$json.output.title}}`
    - `description = {{$json.output.description}}`
    - `processed_folder_id = {{$json.processed_folder_id}}`
- **Inputs/Outputs:** From **Merge** → to **Post To Discord Channel**
- **Edge cases:**
  - If `output` is not present (parser failure), expressions will fail.
  - If binary field name differs from `data`, Discord upload will fail (because the HTTP node expects `files[0]` from `data`).

---

### 2.5 Discord Publishing (create thread with image + copy)

**Overview:**  
Creates a new Discord thread in a channel and posts the AI-generated description with the image attached.

**Nodes involved:**
- **Post To Discord Channel** (HTTP Request)

#### Node: Post To Discord Channel
- **Type / role:** `httpRequest` — calls Discord REST API to create a thread with an initial message + attachment.
- **Configuration choices:**
  - **Method:** POST
  - **URL:** `https://discord.com/api/v10/channels/{{$json.channel_id}}/threads`
  - **Auth:** predefined credential type `discordBotApi`
  - **Body:** multipart/form-data:
    - `payload_json`:
      - `name`: thread name from `{{$json.title.toJsonString()}}`
      - `message.content`: from `{{ $json.description.toJsonString() }}`
    - `files[0]`: binary upload from input field `data`
- **Inputs/Outputs:** From Set node → outputs Discord API response → to **Update file**
- **Edge cases / failures:**
  - Wrong `channel_id` or bot lacks permissions (Create Public Threads, Send Messages, Attach Files) → 403.
  - If the target channel does not support threads or thread creation rules differ → 400.
  - Payload format: Discord thread creation endpoints can be sensitive; if `payload_json` structure is wrong, you’ll get 400.
  - Title length or invalid characters could be rejected by Discord.
  - Large file exceeds Discord upload limits for the server tier.

---

### 2.6 Drive Cleanup (rename + move to processed folder)

**Overview:**  
Renames the processed file using the AI-suggested filename, then moves it into the processed folder for organization.

**Nodes involved:**
- **Update file** (Google Drive: update metadata)
- **Move file** (Google Drive: move)
- **Done !** (NoOp)

#### Node: Update file
- **Type / role:** `googleDrive` (operation: update) — renames the file.
- **Configuration choices:**
  - **File ID:** `={{ $('Get Downloaded File & Set Post Data').item.json.file_id }}`
  - **New updated file name:** `={{ $('AI Agent').item.json.output.new_file_name }}`
- **Inputs/Outputs:** From Discord node → to **Move file**
- **Edge cases:**
  - The expression references **AI Agent** directly. If you later change execution mode or items, `.item` alignment issues can occur.
  - Invalid filename characters for Drive are usually tolerated, but extremely long names may be problematic.
  - Permission issues on file → 403.

#### Node: Move file
- **Type / role:** `googleDrive` (operation: move) — moves the file into the processed folder.
- **Configuration choices:**
  - **File ID:** `={{ $json.id }}`
    - Important: after **Update file**, the Drive node output typically includes the file `id`, so this should work if the update returns metadata.
  - **Folder ID:** `={{ $('Get Downloaded File & Set Post Data').item.json.processed_folder_id }}`
  - **Drive:** “My Drive”
- **Inputs/Outputs:** From **Update file** → to **Done !**
- **Edge cases:**
  - If Update file does not return `id` as expected, this move will fail.
  - Processed folder ID invalid or not shared with the credential → 404/403.

#### Node: Done !
- **Type / role:** `noOp` — explicit end marker.
- **Inputs/Outputs:** From **Move file** → ends.
- **Edge cases:** none.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Check For A New File to post on laughing-everyday | Google Drive Trigger | Detect new file in Drive folder | — | Get File & Set Channel | ## 🚀 AI-Powered Visual Content Pipeline<br>This workflow automates the process of turning raw images into engaging community content. It connects your file storage directly to your social platform, using AI to bridge the creative gap. |
| Get File & Set Channel | Set | Normalize trigger output + set channel/folder variables | Check For A New File to post on laughing-everyday | Download file | ### 🛠️ Setup steps<br>1. **Google Drive:** Create two folders ("Input" and "Processed"). Copy their IDs into the *Google Drive Trigger* and the *Set* nodes respectively.<br>2. **Credentials:** Connect your Google Drive and Google Gemini (PaLM) accounts.<br>3. **Discord:** Add your Bot Token to the *HTTP Request* node and input your Channel ID in the *Set* node variables.<br>4. **Test:** Drop an image into your Input folder to see the magic happen! |
| Download file | Google Drive | Download file binary from Drive | Get File & Set Channel | AI Agent; Merge | ### ⚙️ How it works<br>1. **Trigger:** The automation starts instantly when a new image file is uploaded to a designated Google Drive folder.<br>2. **Analysis:** The file is downloaded and passed to AI. The AI analyzes the image context and generates a catchy title, a community-focused description, and a new filename.<br>3. **Publishing:** The workflow sends the image and the AI-generated text to a specific Discord channel, creating a new thread to encourage discussion.<br>4. **Cleanup:** Finally, the original file is renamed and moved to a "Processed" folder in Google Drive to keep your workspace organized. |
| AI Agent | LangChain Agent | Analyze image and generate JSON (title/description/new filename) | Download file (main); Chat Model (ai_languageModel); Structured Output (ai_outputParser) | Merge | ## 💡 Quick Demo<br>1. Upload a photo of code or a design mockup to your Drive folder.<br>2. Wait a few seconds.<br>3. See a new thread appear in Discord with the image and a question like "How would you optimize this function?"<br><br>## 🎯 Perfect For<br>* **Community Managers** keeping servers active.<br>* **Digital Marketers** automating asset distribution.<br>* **Content Creators** streamlining their "capture-to-publish" workflow. |
| Chat Model | Google Gemini Chat Model | LLM provider for the agent | — | AI Agent (ai_languageModel) |  |
| Structured Output | Structured Output Parser | Enforce strict JSON schema output | — | AI Agent (ai_outputParser) |  |
| Merge | Merge | Combine AI output + binary file | Download file; AI Agent | Get Downloaded File & Set Post Data |  |
| Get Downloaded File & Set Post Data | Set | Build posting fields while keeping binary | Merge | Post To Discord Channel |  |
| Post To Discord Channel | HTTP Request | Create Discord thread + upload image + message | Get Downloaded File & Set Post Data | Update file |  |
| Update file | Google Drive | Rename file based on AI suggestion | Post To Discord Channel | Move file |  |
| Move file | Google Drive | Move file to “Processed” folder | Update file | Done ! |  |
| Done ! | NoOp | End marker | Move file | — |  |
| Sticky Note | Sticky Note | Comment | — | — | ## 💡 Quick Demo<br>1. Upload a photo of code or a design mockup to your Drive folder.<br>2. Wait a few seconds.<br>3. See a new thread appear in Discord with the image and a question like "How would you optimize this function?"<br><br>## 🎯 Perfect For<br>* **Community Managers** keeping servers active.<br>* **Digital Marketers** automating asset distribution.<br>* **Content Creators** streamlining their "capture-to-publish" workflow. |
| Sticky Note1 | Sticky Note | Comment | — | — | ## 🚀 AI-Powered Visual Content Pipeline<br>This workflow automates the process of turning raw images into engaging community content. It connects your file storage directly to your social platform, using AI to bridge the creative gap. |
| Sticky Note4 | Sticky Note | Comment | — | — | ### ⚙️ How it works<br>1. **Trigger:** The automation starts instantly when a new image file is uploaded to a designated Google Drive folder.<br>2. **Analysis:** The file is downloaded and passed to AI. The AI analyzes the image context and generates a catchy title, a community-focused description, and a new filename.<br>3. **Publishing:** The workflow sends the image and the AI-generated text to a specific Discord channel, creating a new thread to encourage discussion.<br>4. **Cleanup:** Finally, the original file is renamed and moved to a "Processed" folder in Google Drive to keep your workspace organized. |
| Sticky Note5 | Sticky Note | Comment | — | — | ### 🛠️ Setup steps<br>1. **Google Drive:** Create two folders ("Input" and "Processed"). Copy their IDs into the *Google Drive Trigger* and the *Set* nodes respectively.<br>2. **Credentials:** Connect your Google Drive and Google Gemini (PaLM) accounts.<br>3. **Discord:** Add your Bot Token to the *HTTP Request* node and input your Channel ID in the *Set* node variables.<br>4. **Test:** Drop an image into your Input folder to see the magic happen! |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Google Drive folders**
   1) Create an **Input** folder (watched folder).  
   2) Create a **Processed** folder (archive).  
   Copy both folder IDs.

2. **Add node: Google Drive Trigger**
   - Node: **Google Drive Trigger**
   - Event: **File Created**
   - Trigger on: **Specific Folder**
   - Folder to watch: paste **Input folder ID**
   - Polling: **Every minute**
   - Credentials: connect a **Google Drive OAuth2** account with access to both folders.

3. **Add node: Set (“Get File & Set Channel”)**
   - Map fields:
     - `file_name = {{$json.name}}`
     - `file_id = {{$json.id}}`
     - `channel_id = "<YOUR_DISCORD_CHANNEL_ID>"`
     - `processed_folder_id = "<YOUR_PROCESSED_FOLDER_ID>"`
     - (Optional) `to_channel` if you plan to use it later
   - Connect: **Drive Trigger → Set**

4. **Add node: Google Drive (“Download file”)**
   - Operation: **Download**
   - File ID: `={{ $json.file_id }}`
   - Ensure binary output is enabled by default (Drive download outputs binary).
   - Connect: **Set → Download file**

5. **Add AI nodes**
   1) **Chat Model: Google Gemini**
      - Node: **Google Gemini Chat Model**
      - Credentials: connect **Google PaLM/Gemini** API credential
   2) **Structured Output Parser**
      - Schema keys: `title`, `description`, `new_file_name` as strings
   3) **AI Agent**
      - Prompt: include instructions to analyze the *attached image* and produce strict JSON
      - System message: define the exact JSON object format
      - Connect:
        - **Chat Model → AI Agent** (ai_languageModel connection)
        - **Structured Output → AI Agent** (ai_outputParser connection)
        - **Download file → AI Agent** (main)

6. **Add node: Merge**
   - Mode: **Combine**
   - Combine by: **Position**
   - Connect:
     - **Download file → Merge** (main input 0)
     - **AI Agent → Merge** (main input 1)

7. **Add node: Set (“Get Downloaded File & Set Post Data”)**
   - Keep binary field (ensure `data` remains available).
   - Assign:
     - `title = {{$json.output.title}}`
     - `description = {{$json.output.description}}`
     - `file_id = {{$json.file_id}}`
     - `channel_id = {{$json.channel_id}}`
     - `processed_folder_id = {{$json.processed_folder_id}}`
   - Connect: **Merge → Set**

8. **Add node: HTTP Request (“Post To Discord Channel”)**
   - Method: **POST**
   - URL: `https://discord.com/api/v10/channels/{{$json.channel_id}}/threads`
   - Authentication: **Discord Bot credential**
     - Create credential with your bot token
     - Ensure the bot is in the server and has permissions: **Send Messages**, **Create Public Threads**, **Attach Files**
   - Content type: **multipart/form-data**
   - Body parameters:
     1) `payload_json` (text):
        - Value:
          - `{"name": {{$json.title.toJsonString()}}, "message": {"content": {{ $json.description.toJsonString() }} }}`
     2) `files[0]` (binary):
        - Parameter type: **formBinaryData**
        - Input data field name: `data`
   - Connect: **Set → HTTP Request**

9. **Add node: Google Drive (“Update file”)**
   - Operation: **Update**
   - File ID: `={{ $('Get Downloaded File & Set Post Data').item.json.file_id }}`
   - New filename: `={{ $('AI Agent').item.json.output.new_file_name }}`
   - Connect: **HTTP Request → Update file**

10. **Add node: Google Drive (“Move file”)**
    - Operation: **Move**
    - File ID: `={{ $json.id }}`
    - Folder ID: `={{ $('Get Downloaded File & Set Post Data').item.json.processed_folder_id }}`
    - Drive: **My Drive** (or your shared drive if relevant)
    - Connect: **Update file → Move file**

11. **Add node: NoOp (“Done !”)**
    - Connect: **Move file → Done !**

**Optional hardening (recommended):**
- Add an **IF** node to ensure the created file is an image MIME type before downloading/posting.
- Add error handling (Error Trigger or workflow-level “continue on fail” strategies) for Discord 403/400 and AI parser failures.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “🚀 AI-Powered Visual Content Pipeline… connects your file storage directly to your social platform, using AI to bridge the creative gap.” | Sticky note (overview) |
| “⚙️ How it works: Trigger → Analysis → Publishing → Cleanup” | Sticky note (process explanation) |
| “🛠️ Setup steps: create Input/Processed folders; connect Drive + Gemini; add Bot Token; test with an image.” | Sticky note (setup guidance) |
| “💡 Quick Demo… see a new thread appear in Discord… Perfect for Community Managers / Digital Marketers / Content Creators.” | Sticky note (use cases) |