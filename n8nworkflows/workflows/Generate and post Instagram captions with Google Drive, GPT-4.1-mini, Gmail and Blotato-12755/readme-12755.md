Generate and post Instagram captions with Google Drive, GPT-4.1-mini, Gmail and Blotato

https://n8nworkflows.xyz/workflows/generate-and-post-instagram-captions-with-google-drive--gpt-4-1-mini--gmail-and-blotato-12755


# Generate and post Instagram captions with Google Drive, GPT-4.1-mini, Gmail and Blotato

## 1. Workflow Overview

**Workflow name:** Social Media Post & Caption Generator with Approval Loop  
**Purpose:** Automatically pick a random media file from a Google Drive folder, generate an Instagram caption with **GPT‑4.1‑mini**, email it for **human approval**, and—if approved—download the file and **publish the post via Blotato**. If rejected, it loops back to pick another file.

**Primary use cases**
- Creators/marketers who want consistent posting using an existing Drive content library.
- Teams needing a “human-in-the-loop” approval gate before publishing.
- Automated caption drafting from minimal metadata (file name).

### 1.1 Scheduled Run (Entry)
Runs once per day at a specified hour.

### 1.2 Content Discovery + Random Selection
Lists all files in a target Google Drive folder, then selects one random item to avoid repetition.

### 1.3 AI Caption Generation
Calls OpenAI (LangChain node) to generate an Instagram-ready caption using the selected file’s name.

### 1.4 Approval Loop (Gmail Send-and-Wait)
Sends an email containing a Drive link + the generated caption. The workflow pauses until approval/rejection.

### 1.5 Publish (Only if Approved)
Downloads the approved media from Drive, uploads it to Blotato as media, then creates a post with the AI caption and uploaded media URL.

### 1.6 Rejection Handling (Loop-back)
If not approved, loops back to the Drive search step to pick a different random file.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled Run (Entry)

**Overview:** Triggers the workflow daily at a fixed time (10:00).  
**Nodes involved:** `Schedule Trigger`

#### Node: Schedule Trigger
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — time-based entrypoint.
- **Configuration (interpreted):**
  - Runs every day at **10:00** (server/workspace timezone).
- **Inputs / outputs:**
  - **Input:** none (trigger).
  - **Output:** starts flow to **Search files and folders**.
- **Version notes:** typeVersion `1.3`.
- **Potential failures / edge cases:**
  - Timezone mismatch (n8n instance timezone vs expected posting timezone).
  - Workflow inactive (`active:false`) means it will not run until activated.

---

### Block 2 — Content Discovery + Random Selection

**Overview:** Fetches all items in a Drive folder, then chooses one randomly for variety.  
**Nodes involved:** `Search files and folders`, `Randomizer`

#### Node: Search files and folders
- **Type / role:** `n8n-nodes-base.googleDrive` — lists files/folders in a given folder.
- **Configuration (interpreted):**
  - **Resource:** File/Folder listing (`resource: fileFolder`)
  - **Folder filter:** uses a specific **folderId** (template placeholder shows `REPLACE_WITH_YOUR_FOLDER_ID` in cached URL text; actual value in node is `1Xs6aYQ...`).
  - **Return all:** enabled (`returnAll:true`) so every file is returned as an item.
- **Inputs / outputs:**
  - **Input:** from `Schedule Trigger` or rejection loop from `If` (false branch).
  - **Output:** to `Randomizer`.
- **Credentials:** Google Drive OAuth2 required (read/list access to that folder).
- **Version notes:** typeVersion `3`.
- **Potential failures / edge cases:**
  - Folder ID invalid or lacks permission → 403/404.
  - Empty folder → outputs zero items; downstream randomizer returns empty and the workflow effectively stops (no approval email).
  - Large folders may increase execution time; consider pagination/limits if needed.

#### Node: Randomizer
- **Type / role:** `n8n-nodes-base.code` — selects one random input item.
- **Configuration (interpreted):**
  - Reads all incoming items (`$input.all()`).
  - If none, returns `[]`.
  - Picks one at random and returns it as the only output item.
- **Key code behavior:**
  - `Math.floor(Math.random() * items.length)` determines the chosen file index.
- **Inputs / outputs:**
  - **Input:** all files from Drive listing.
  - **Output:** single chosen file item to `Caption Generator AI`.
- **Version notes:** typeVersion `2`.
- **Potential failures / edge cases:**
  - No items from Drive → returns empty; downstream nodes won’t run.
  - Random selection can repeat across days; no “already used” memory is implemented.

---

### Block 3 — AI Caption Generation

**Overview:** Generates an Instagram caption based on the chosen file’s name.  
**Nodes involved:** `Caption Generator AI`

#### Node: Caption Generator AI
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI chat completion via n8n LangChain integration.
- **Configuration (interpreted):**
  - **Model:** `gpt-4.1-mini`
  - **System message:** “You are a social media (Instagram) Caption specialist…”
  - **User message:** includes the selected file name:
    - `File name:{{ $json.name }}`
    - Instruction: “Create an effective caption for a social media post (Instagram) …”
- **Key expressions / variables:**
  - Uses `{{ $json.name }}` from the Randomizer-selected Drive item.
- **Inputs / outputs:**
  - **Input:** the selected Drive file item (must include `name`).
  - **Output:** AI response used by Gmail and later by Blotato post creation.
- **Version notes:** typeVersion `2.1`.
- **Potential failures / edge cases:**
  - Missing/undefined `$json.name` → prompt becomes less useful.
  - OpenAI credential missing/invalid → auth errors.
  - Model availability/permissions; rate limits or quota exceeded.
  - Output structure assumptions downstream: later nodes expect `output[0].content[0].text`.

---

### Block 4 — Email Approval (Send-and-Wait) + Decision

**Overview:** Emails the generated caption and a Drive link, then waits for approval/rejection. Approved proceeds to publishing; rejected loops back to pick another file.  
**Nodes involved:** `Send For Approval and Wait`, `If`

#### Node: Send For Approval and Wait
- **Type / role:** `n8n-nodes-base.gmail` — sends an email and pauses execution until an approval response is recorded.
- **Configuration (interpreted):**
  - **Operation:** `sendAndWait` (approval workflow)
  - **To:** `REPLACE_WITH_YOUR_EMAIL`
  - **Subject:** “Social Media Post for Approval”
  - **Body:** includes:
    - Drive view link built from the selected file id:
      - `https://drive.google.com/file/d/{{ $('Randomizer').item.json.id }}/view`
    - Caption from AI output:
      - `{{ $json.output[0].content[0].text }}`
  - **Approval options:** `approvalType: double` (requires a “double” style confirmation as configured by the node)
- **Key expressions / variables:**
  - `$('Randomizer').item.json.id` (references the Randomizer node item explicitly)
  - `$json.output[0].content[0].text` (assumes current item is AI node output)
- **Inputs / outputs:**
  - **Input:** from `Caption Generator AI`.
  - **Output:** to `If` with an approval payload, typically under something like `$json.data.approved`.
- **Credentials:** Gmail OAuth2 required; must allow sending mail and supporting the node’s approval mechanism.
- **Version notes:** typeVersion `2.2`.
- **Potential failures / edge cases:**
  - Incorrect Gmail scopes / OAuth not connected → auth errors.
  - Approval email goes to spam or blocked by domain policies.
  - If the approval payload shape changes, the downstream `If` condition may not match.
  - Using `sendAndWait` means executions remain “waiting” until action is taken—may affect concurrency/execution limits.

#### Node: If
- **Type / role:** `n8n-nodes-base.if` — branches based on approval boolean.
- **Configuration (interpreted):**
  - Condition checks whether approval flag is true:
    - `{{ $json.data.approved }}` **equals** `true`
- **Inputs / outputs:**
  - **Input:** from `Send For Approval and Wait`.
  - **True branch (approved):** to `Download Content File`.
  - **False branch (rejected):** loops back to `Search files and folders` (new random pick path).
- **Version notes:** typeVersion `2.3`.
- **Potential failures / edge cases:**
  - If approval data is missing or located at a different path → always false (infinite loop risk if approvals never set true).
  - Loop-back can create repeated rejections without any “max attempts” safeguard.

---

### Block 5 — Publish to Social (Drive Download → Blotato Upload → Blotato Post)

**Overview:** After approval, downloads the selected media from Drive, uploads it to Blotato, then publishes a post using the AI caption and uploaded media URL.  
**Nodes involved:** `Download Content File`, `Upload media`, `Create post`

#### Node: Download Content File
- **Type / role:** `n8n-nodes-base.googleDrive` — downloads the approved file as binary.
- **Configuration (interpreted):**
  - **Operation:** download
  - **File ID:** `{{ $('Randomizer').item.json.id }}`
- **Key expressions / variables:**
  - References Randomizer’s chosen file id (not the current item).
- **Inputs / outputs:**
  - **Input:** from `If` (approved branch).
  - **Output:** binary file data to `Upload media`.
- **Credentials:** Google Drive OAuth2 (download permission).
- **Version notes:** typeVersion `3`.
- **Potential failures / edge cases:**
  - File deleted/moved between selection and approval → 404.
  - Large file download may time out or exceed memory/execution limits.

#### Node: Upload media
- **Type / role:** `@blotato/n8n-nodes-blotato.blotato` — uploads media to Blotato.
- **Configuration (interpreted):**
  - **Resource:** media
  - **Use binary data:** enabled (`useBinaryData:true`) meaning it expects the Drive download as binary input.
- **Inputs / outputs:**
  - **Input:** binary from `Download Content File`.
  - **Output:** includes uploaded media URL as `$json.url` (used next).
- **Credentials:** Blotato API credential required.
- **Version notes:** typeVersion `2`.
- **Potential failures / edge cases:**
  - Unsupported file type / size limits on Blotato side.
  - Missing binary property (if Drive node output binary name differs from what Blotato expects).
  - Blotato auth/permission issues.

#### Node: Create post
- **Type / role:** `@blotato/n8n-nodes-blotato.blotato` — creates/publishes a social post via Blotato.
- **Configuration (interpreted):**
  - **Account:** selected by `accountId` (placeholder value `ID`, cached details redacted).
  - **Text content:** pulled from the AI node output:
    - `{{ $('Caption Generator AI').item.json.output[0].content[0].text }}`
  - **Media URLs:** pulled from previous Upload media output:
    - `{{ $json.url }}`
- **Inputs / outputs:**
  - **Input:** from `Upload media` (expects `url`).
  - **Output:** Blotato post creation result (post id/status, depending on API).
- **Version notes:** typeVersion `2`.
- **Potential failures / edge cases:**
  - If AI output path changes, caption becomes empty or expression fails.
  - Invalid/expired Blotato account connection; wrong `accountId`.
  - API errors if caption violates platform limits (length, restricted content).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Daily trigger at 10:00 | — | Search files and folders | Scheduled Event Trigger |
| Search files and folders | n8n-nodes-base.googleDrive | List files in a target Drive folder | Schedule Trigger; If (rejected path) | Randomizer | Search File + Randomizer |
| Randomizer | n8n-nodes-base.code | Select one random file item | Search files and folders | Caption Generator AI | Search File + Randomizer |
| Caption Generator AI | @n8n/n8n-nodes-langchain.openAi | Generate Instagram caption from file name | Randomizer | Send For Approval and Wait | Caption Generator & Email Approval |
| Send For Approval and Wait | n8n-nodes-base.gmail | Email caption + file link and pause for approval | Caption Generator AI | If | Caption Generator & Email Approval |
| If | n8n-nodes-base.if | Route based on approval boolean | Send For Approval and Wait | Download Content File (approved); Search files and folders (rejected) | Caption Generator & Email Approval |
| Download Content File | n8n-nodes-base.googleDrive | Download approved media as binary | If (approved) | Upload media | Upload to Social MEdia |
| Upload media | @blotato/n8n-nodes-blotato.blotato | Upload binary media to Blotato | Download Content File | Create post | Upload to Social MEdia |
| Create post | @blotato/n8n-nodes-blotato.blotato | Publish post with caption + media URL | Upload media | — | Upload to Social MEdia |
| Sticky Note | n8n-nodes-base.stickyNote | Workspace annotation | — | — | Scheduled Event Trigger |
| Sticky Note1 | n8n-nodes-base.stickyNote | Workspace annotation | — | — | Search File + Randomizer |
| Sticky Note2 | n8n-nodes-base.stickyNote | Workspace annotation | — | — | Caption Generator & Email Approval |
| Sticky Note3 | n8n-nodes-base.stickyNote | Workspace annotation | — | — | Upload to Social MEdia |
| Sticky Note4 | n8n-nodes-base.stickyNote | Workspace annotation | — | — | Social Media Post & Caption Generator (Google Drive → AI Caption → Approval → Auto-Post) … (contains full description + setup + troubleshooting + YouTube link) |
| Sticky Note5 | n8n-nodes-base.stickyNote | Workspace annotation | — | — | @[youtube](9XU9ECcj9dg) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named:  
   “Social Media Post & Caption Generator with Approval Loop”.

2. **Add “Schedule Trigger”**
   - Node: **Schedule Trigger**
   - Set it to run **daily at 10:00** (adjust timezone as needed).
   - Connect to the next node.

3. **Add Google Drive: “Search files and folders”**
   - Node: **Google Drive**
   - Resource: **File/Folder**
   - Action/Operation: **Search/List** (list files in a folder)
   - Set **Folder ID** to your content folder (e.g., `1abc...`).
   - Enable **Return All**.
   - **Credentials:** connect **Google Drive OAuth2**.
   - Connect to the Code node.

4. **Add Code node: “Randomizer”**
   - Node: **Code**
   - Paste logic to select one random input item (as in the workflow):
     - If no items, return empty array
     - Else return one random item
   - Connect to the OpenAI node.

5. **Add OpenAI (LangChain) node: “Caption Generator AI”**
   - Node: **OpenAI (LangChain)**
   - Model: **gpt-4.1-mini**
   - System message: Instagram caption specialist instruction
   - User message: include `File name: {{ $json.name }}` and request an effective Instagram caption.
   - **Credentials:** connect OpenAI API credential.
   - Connect to Gmail.

6. **Add Gmail node: “Send For Approval and Wait”**
   - Node: **Gmail**
   - Operation: **Send and Wait (Approval)**
   - To: your email address (replace placeholder).
   - Subject: “Social Media Post for Approval”
   - Message body:
     - Include Drive file link using the Randomizer item id:  
       `https://drive.google.com/file/d/{{ $('Randomizer').item.json.id }}/view`
     - Include the caption from the AI output:  
       `{{ $json.output[0].content[0].text }}`
   - Approval configuration: set approval type to match your needs (this workflow uses **double**).
   - **Credentials:** connect Gmail OAuth2.
   - Connect to an IF node.

7. **Add IF node: “If” (approval gate)**
   - Condition: Boolean equals:
     - Left: `{{ $json.data.approved }}`
     - Right: `true`
   - **True output:** proceed to download.
   - **False output:** loop back to Google Drive search node (to pick another file).

8. **Add Google Drive node: “Download Content File”** (approved path)
   - Node: **Google Drive**
   - Operation: **Download**
   - File ID expression: `{{ $('Randomizer').item.json.id }}`
   - **Credentials:** same Google Drive OAuth2.
   - Connect to Blotato upload.

9. **Add Blotato node: “Upload media”**
   - Node: **Blotato**
   - Resource: **Media**
   - Enable **Use Binary Data** (expects binary from Drive download).
   - **Credentials:** connect Blotato API credential.
   - Connect to “Create post”.

10. **Add Blotato node: “Create post”**
   - Node: **Blotato**
   - Resource/Action: **Create Post**
   - Select the **Account ID** (your target social profile).
   - Post text expression: `{{ $('Caption Generator AI').item.json.output[0].content[0].text }}`
   - Post media URL(s) expression: `{{ $json.url }}`
   - Save.

11. **Test**
   - Run once manually.
   - Confirm you receive the approval email.
   - Approve to confirm it downloads, uploads media, and creates the post.
   - Reject to confirm it loops and selects a new random file.

12. **Activate the workflow** once credentials and IDs are correct.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Scheduled Event Trigger” | Sticky note labeling the trigger block |
| “Search File + Randomizer” | Sticky note labeling the Drive search + random selection block |
| “Caption Generator & Email Approval” | Sticky note labeling AI caption + Gmail approval block |
| “Upload to Social MEdia” | Sticky note labeling the publish block |
| Step-by-step guide video | https://youtu.be/9XU9ECcj9dg |
| Embedded YouTube reference | @[youtube](9XU9ECcj9dg) |
| Template description includes requirements (Drive OAuth, OpenAI, Gmail OAuth, Blotato API) and troubleshooting notes (no files found, approval not received, caption quality, upload fails). | Included in the large sticky note content |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.