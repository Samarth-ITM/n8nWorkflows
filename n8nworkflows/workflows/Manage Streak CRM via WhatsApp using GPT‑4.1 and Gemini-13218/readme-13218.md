Manage Streak CRM via WhatsApp using GPT‑4.1 and Gemini

https://n8nworkflows.xyz/workflows/manage-streak-crm-via-whatsapp-using-gpt-4-1-and-gemini-13218


# Manage Streak CRM via WhatsApp using GPT‑4.1 and Gemini

## 1. Workflow Overview

**Title:** Manage Streak CRM via WhatsApp using GPT‑4.1 and Gemini

**Purpose:**  
This workflow turns a WhatsApp Business number into a conversational Streak CRM assistant. Users can send **text**, **voice notes (audio)**, or **images** via WhatsApp; the workflow converts those inputs into a normalized text prompt, then an **AI Agent** (GPT‑4.1-mini with Gemini fallback) decides which Streak actions to execute (create/update contacts, create/update boxes, link contacts to boxes, add comments, create/get tasks, and run custom Streak API requests). Finally, it replies back to the user on WhatsApp.

**Main use cases**
- “Create contact John Doe john@acme.com”
- “Move Brian to Follow-up”
- “Add a note to Nick’s box: called and left voicemail”
- “Create a box for Acme in pipeline X and link the contact”
- Send an **image** of a job card → extract details → create/update contact/box
- Send a **voice note** → translate → apply CRM actions

### Logical blocks
**1.1 WhatsApp Ingestion & Routing**  
Receives WhatsApp messages and routes by media type (text/audio/image).

**1.2 Media Retrieval & Transcription/Description**  
For audio: fetch media → translate to text. For image: fetch media → analyze image → produce description.

**1.3 Prompt Normalization**  
Maps each input type into a single `text` field.

**1.4 AI Agent + Memory + Tooling (Streak CRM Ops)**  
Agent uses memory and a toolset (native Streak nodes + generic HTTP tools + Streak docs search) to perform actions.

**1.5 WhatsApp Response**  
Sends the agent’s final output back to the originating WhatsApp user.

---

## 2. Block-by-Block Analysis

### 2.1 WhatsApp Ingestion & Routing

**Overview:**  
Listens for incoming WhatsApp messages, then branches execution depending on whether the content is text, audio, or image.

**Nodes involved**
- WhatsApp Trigger
- Route input Types

#### Node: WhatsApp Trigger
- **Type / role:** `n8n-nodes-base.whatsAppTrigger` — entry point webhook for WhatsApp Business updates.
- **Config (interpreted):**
  - Subscribes to `updates: ["messages"]` (message events).
  - Uses WhatsApp Trigger credentials `delta40 wa`.
  - `retryOnFail: true` to retry transient failures.
- **Key data used later:**
  - Sender WA ID: `contacts[0].wa_id`
  - Message type: `messages[0].type`
- **Outputs:** to **Route input Types**.
- **Edge cases / failures:**
  - WhatsApp webhook verification / permission issues.
  - Payload shape differences (some messages may not include `contacts`, or `messages[0]` missing).
  - Duplicate delivery (WhatsApp can retry webhooks).

#### Node: Route input Types
- **Type / role:** `n8n-nodes-base.switch` — conditional branching by `messages[0].type`.
- **Config (interpreted):**
  - If `messages[0].type == "text"` → output “Text”
  - If `messages[0].type == "audio"` → output “Audio”
  - If `messages[0].type == "image"` → output “Image”
  - There is also a rule for `"document"` but **it is not connected to any downstream nodes**, so document messages effectively stop here.
- **Outputs:**
  - Text → **Map text prompt**
  - Audio → **Gets WhatsApp Voicemail Source URL**
  - Image → **Gets WhatsApp Image Source URL**
- **Edge cases / failures:**
  - If message type is unsupported (e.g., `document`, `interactive`, `sticker`) nothing proceeds.
  - If payload doesn’t match strict validation, rule may not match and flow ends.

---

### 2.2 Media Retrieval & Transcription/Description (Audio + Image)

**Overview:**  
When the message contains audio or image media, the workflow retrieves a downloadable URL from WhatsApp, downloads the file, then uses OpenAI to convert it to text (audio translation) or to a textual description (image analysis).

**Nodes involved**
- Gets WhatsApp Voicemail Source URL
- Download Voicemail
- OpenAI (audio translate)
- Gets WhatsApp Image Source URL
- Download Image
- OpenAI1 (image analyze)

#### Node: Gets WhatsApp Voicemail Source URL
- **Type / role:** `n8n-nodes-base.whatsApp` — WhatsApp Cloud API call to get media URL for an audio message.
- **Config:**
  - Resource: `media`
  - Operation: `mediaUrlGet`
  - Media ID: `{{$json.messages[0].audio.id}}`
- **Output:** to **Download Voicemail** (expects JSON containing `url`).
- **Edge cases:**
  - Missing `messages[0].audio.id` (non-audio payload, corrupted message).
  - WhatsApp credential scope / permission errors.
  - Media might have expired.

#### Node: Download Voicemail
- **Type / role:** `n8n-nodes-base.httpRequest` — downloads media binary from URL.
- **Config:**
  - URL: `={{ $json.url }}`
  - Authentication: “predefinedCredentialType” with `whatsAppApi` credential (so it can access WhatsApp-hosted media).
- **Output:** to **OpenAI** (audio translate).
- **Edge cases:**
  - HTTP 401/403 if auth headers not applied correctly.
  - Large audio file timeouts.
  - If response is not treated as binary as expected, downstream OpenAI may fail.

#### Node: OpenAI (audio)
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — OpenAI audio translation (voice → text).
- **Config:**
  - Resource: `audio`
  - Operation: `translate`
  - Uses OpenAI credential `OpenAi Delta40`
- **Output:** to **Chat with Streak CRM** (agent input).
- **Edge cases:**
  - Unsupported audio codec/container.
  - Audio too long; rate limits; OpenAI API errors.

---

#### Node: Gets WhatsApp Image Source URL
- **Type / role:** `n8n-nodes-base.whatsApp` — fetch media URL for image.
- **Config:**
  - Resource: `media`
  - Operation: `mediaUrlGet`
  - Media ID: `={{ $json.messages[0].image.id }}`
- **Output:** to **Download Image**.
- **Edge cases:** same class as audio: missing ID, expired media, auth errors.

#### Node: Download Image
- **Type / role:** `n8n-nodes-base.httpRequest` — downloads image.
- **Config:**
  - URL: `={{ $json.url }}`
  - Auth via WhatsApp API credential.
- **Output:** to **OpenAI1** (image analysis).
- **Edge cases:**
  - Large images; format issues; download timeouts.

#### Node: OpenAI1 (image)
- **Type / role:** `@n8n/n8n-nodes-langchain.openAi` — image analysis to produce a textual description.
- **Config:**
  - Resource: `image`
  - Operation: `analyze`
  - Input type: `base64`
  - Model: `gpt-4o-mini`
  - Option `detail: auto`
- **Output:** to **Map image prompt** (expects `content` field from analysis result).
- **Edge cases:**
  - If the HTTP download doesn’t produce expected binary/base64, analysis fails.
  - Image contains sensitive/low-quality text; description may be incomplete.

---

### 2.3 Prompt Normalization (Create the `text` field)

**Overview:**  
Normalizes every input modality into one string field named `text`, which becomes the agent’s instruction prompt.

**Nodes involved**
- Map text prompt
- Map image prompt  
*(Audio path bypasses “Map … prompt” and goes directly into the agent with OpenAI’s translation output.)*

#### Node: Map text prompt
- **Type / role:** `n8n-nodes-base.set` — maps WhatsApp text body into `text`.
- **Config:**
  - Sets `text = {{$json.messages[0].text.body}}`
- **Input:** from **Route input Types** (Text branch).
- **Output:** to **Chat with Streak CRM**.
- **Edge cases:**
  - Some message types may have no `messages[0].text.body` even if type is `text` (rare payload variants).

#### Node: Map image prompt
- **Type / role:** `n8n-nodes-base.set` — builds a combined prompt from AI image description + user caption.
- **Config:**
  - `text = "User image description: {{ $json.content }}\n\nUser image caption: {{ $('Route input Types').item.json.messages[0].image.caption }}"`
  - Uses:
    - `$json.content` from **OpenAI1** image analysis output
    - Original caption from the **Route input Types** item via node reference
- **Input:** from **OpenAI1**.
- **Output:** to **Chat with Streak CRM**.
- **Edge cases:**
  - `messages[0].image.caption` may be missing → expression may evaluate to `undefined` (still usually safe, but results in messy prompt).
  - If multiple items exist, node referencing `$('Route input Types').item` assumes same item mapping; mismatches can occur if batching changes.

---

### 2.4 AI Agent + Memory + Tooling (Streak CRM Operations)

**Overview:**  
The LangChain agent receives the normalized `text` prompt and uses conversational memory plus a collection of tools to query/modify Streak CRM. It can also consult Streak API documentation via Tavily. The agent is configured with detailed operational rules, especially for correctly linking contacts to boxes.

**Nodes involved**
- Chat with Streak CRM (Agent)
- Simple Memory
- OpenAI Chat Model
- Google Gemini Chat Model
- Get_boxes_details
- Update_box_in_Streak
- Create a contact
- Update a contact
- Create a box in a pipeline in Streak
- Get_user_in_Streak
- Get_pipeline-fileds
- Create_task_in_box_Streak
- Get a task in Streak
- Streak_Doc
- Get_Custom_Operation
- POST_Custom_Operation
- Link_contact_to_box
- Comment tag

#### Node: Chat with Streak CRM
- **Type / role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates LLM reasoning + tool calling.
- **Config (interpreted):**
  - Input text: `={{ $json.text }}`
  - Prompt type: “define” with a large **system message** that:
    - Defines allowed operations
    - Lists tools and how to use them
    - Includes strict rules for linking contacts to boxes (must use `Link_contact_to_box` and correct JSON format)
    - Lists pipeline stage IDs (5001/5002/5003)
    - Specifies response style (“brief and friendly”, confirmation messages)
  - Error handling:
    - `onError: continueRegularOutput`
    - `maxTries: 2`
    - `retryOnFail: true`
    - `alwaysOutputData: true`
  - **Language models attached:**
    - Primary: **OpenAI Chat Model** (index 0)
    - Fallback: **Google Gemini Chat Model** (index 1) because `needsFallback: true`
  - **Memory attached:** Simple Memory (buffer window)
  - **Tools attached:** all nodes connected via `ai_tool`
- **Output:** goes to **Send message** node, expects `output` field in JSON (`Send message` uses `{{$json.output}}`).
- **Edge cases / failures:**
  - Tool execution errors (auth, invalid keys, Streak validation errors).
  - Model may call tools with incomplete parameters; `$fromAI()` arguments can be empty strings causing invalid URLs/bodies.
  - If model output does not include `output`, WhatsApp reply will be blank/undefined.

#### Node: Simple Memory
- **Type / role:** `memoryBufferWindow` — short-term conversational state.
- **Config:**
  - Session key: `memory_{{ $('WhatsApp Trigger').item.json.contacts[0].wa_id }}`
  - Window length: 20 turns
- **Connection:** provides `ai_memory` to the Agent.
- **Edge cases:**
  - If `contacts[0].wa_id` missing, memory session key becomes invalid.
  - Memory persistence depends on n8n/LangChain memory backend behavior in your environment.

#### Node: OpenAI Chat Model
- **Type / role:** `lmChatOpenAi` — main LLM for agent reasoning.
- **Config:** model `gpt-4.1-mini`.
- **Connection:** `ai_languageModel` → Agent (index 0).
- **Edge cases:** OpenAI rate limits, invalid key, model availability.

#### Node: Google Gemini Chat Model
- **Type / role:** `lmChatGoogleGemini` — fallback LLM.
- **Config:** default options, credential “Google Gemini Delta40”.
- **Connection:** `ai_languageModel` → Agent (index 1).
- **Edge cases:** Gemini API limits, mismatched tool-calling behavior vs OpenAI (agent must be robust).

---

#### Streak “native” tools (streakTool nodes)

These are purpose-built tools the agent can call.

##### Node: Get_boxes_details
- **Type / role:** `n8n-nodes-streak-crm.streakTool` — searches/returns boxes in a pipeline.
- **Config:**
  - Resource: `box`
  - Return all: enabled
  - Pipeline key: “Delta40 Sandbox Pipeline (…)”
  - Search query: `$fromAI('Search_Query')` (agent supplies a string)
  - Stage filter: present but effectively set to `=` (likely unused/empty)
- **Connection:** `ai_tool` → Agent.
- **Failure modes:**
  - Wrong pipeline key.
  - Search query too broad → many boxes; the agent should ask for clarification but may not.
  - Streak API auth errors (OAuth/Key issues).

##### Node: Update_box_in_Streak
- **Type / role:** `streakTool` — updates a box (notes, stage, assignment).
- **Config:**
  - boxKey from `$fromAI('Box_Key')`
  - update fields:
    - notes from `$fromAI('Notes')`
    - stageKey from `$fromAI('Stage')` (string)
    - assignedToTeamKeyOrUserKey from `$fromAI('Assigned_To__Team_User_Key_')`
- **Connection:** `ai_tool` → Agent.
- **Edge cases:**
  - Invalid stageKey (must match pipeline stages).
  - Missing boxKey or wrong boxKey → 404.
  - Permission errors if user cannot modify that box.

##### Node: Create a contact
- **Type / role:** `streakTool` — creates a Streak contact.
- **Config:**
  - teamKey fixed: `agxz...`
  - givenName / familyName from `$fromAI(...)`
  - emailAddresses[0] from `$fromAI('emailAddresses0_Email_Addresses')`
- **Connection:** `ai_tool` → Agent.
- **Edge cases:**
  - Missing required fields; invalid email format.
  - Duplicates: Streak may allow multiple; agent instructions suggest searching first, but this node doesn’t enforce it.

##### Node: Update a contact
- **Type / role:** `streakTool` — updates a contact.
- **Config:**
  - contactKey from `$fromAI('Contact_Key')`
  - updateFields is empty `{}` in the node configuration (meaning the agent must rely on node UI fields—but as configured, it will not change anything unless n8n tool framework injects fields; typically this results in a no-op or API validation error).
- **Connection:** `ai_tool` → Agent.
- **Important risk:** As configured, it likely cannot update anything.
- **Recommendation:** Add explicit update fields (phone, emails, addresses, custom fields) you expect the agent to modify.

##### Node: Create a box in a pipeline in Streak
- **Type / role:** `streakTool` — creates a box.
- **Config:**
  - boxName from `$fromAI('Box_Name')`
  - pipelineKey fixed to “Delta40 Sandbox Pipeline”
  - stageKey from `$fromAI('Stage')`
  - additional fields:
    - notes from `$fromAI('Notes')`
    - assignedToTeamKeyOrUserKey from `$fromAI('Assigned_To__Team_User_Key_')`
- **Connection:** `ai_tool` → Agent.
- **Edge cases:** invalid stageKey; missing boxName; assignment key invalid.

##### Node: Get_user_in_Streak
- **Type / role:** `streakTool` — (operation not set in JSON) used as a tool to fetch Streak user keys.
- **Config:** empty parameters block `{}`; behavior depends on node defaults.
- **Connection:** `ai_tool` → Agent.
- **Risk:** If the node requires an explicit resource/operation, it may fail at runtime.

##### Node: Get_pipeline-fileds
- **Type / role:** `streakTool` — lists pipeline fields.
- **Config:**
  - resource: `field`
  - pipelineKey fixed
  - returnAll is provided by agent: `$fromAI('Return_All', boolean)`
- **Connection:** `ai_tool` → Agent.
- **Edge cases:** agent may pass non-boolean; field listing could be large.

##### Node: Create_task_in_box_Streak
- **Type / role:** `streakTool` — creates a task in a box.
- **Config:**
  - text from `$fromAI('Task_Text')`
  - boxKey from `$fromAI('Box')`
  - pipelineKey from `$fromAI('Pipeline_Key')`
- **Connection:** `ai_tool` → Agent.
- **Edge cases:** pipelineKey/boxKey mismatch; missing required task fields.

##### Node: Get a task in Streak
- **Type / role:** `streakTool` — retrieves a task.
- **Config:**
  - resource: `task`
  - taskKey from `$fromAI('Task_Key')`
- **Connection:** `ai_tool` → Agent.
- **Edge cases:** taskKey unknown; agent may need list-first (system message suggests using custom GET to fetch keys).

---

#### Generic HTTP tools (power-user endpoints)

These are exposed to the agent as “Super Admin” tools to hit any Streak endpoint.

##### Node: Get_Custom_Operation
- **Type / role:** `n8n-nodes-base.httpRequestTool` — arbitrary GET request.
- **Config:**
  - URL is constructed to search contacts:
    - `https://api.streak.com/api/v1/search?query=<encoded>&type=contact`
  - Query comes from `$fromAI('searchQuery')`
  - Auth: `httpBasicAuth` (“streak basic credential”)
- **Connection:** `ai_tool` → Agent.
- **Edge cases:**
  - Search results can include multiple entity types; but `type=contact` restricts it.
  - Basic Auth must be Streak API key as username (typical Streak pattern), empty password.

##### Node: POST_Custom_Operation
- **Type / role:** `httpRequestTool` — arbitrary POST (legacy/experimental usage).
- **Config:**
  - URL: `https://api.streak.com/api/v1/boxes/<boxKey>`
  - Body: `{"contactKeys":["<contactKey>"]}` constructed as a JSON string.
  - Auth: Basic Auth.
- **Connection:** `ai_tool` → Agent.
- **Important note:** The system message explicitly says **this does NOT work** for linking contacts and must not be used for that purpose.
- **Edge cases:** Because JSON is built as a string, quoting/escaping errors are possible.

##### Node: Link_contact_to_box
- **Type / role:** `httpRequestTool` — the correct contact-to-box linking endpoint usage.
- **Config:**
  - URL: `https://api.streak.com/api/v1/boxes/<boxKey>`
  - Method: POST
  - JSON body (stringified):
    - `{ contacts: [{ key: <contactKey> }] }`
  - Auth: Basic Auth.
- **Connection:** `ai_tool` → Agent.
- **Failure modes:**
  - Wrong API version/endpoint behavior changes; but this is stated as the correct method in the system message.
  - If the API expects `contacts` vs `contactKeys` (per the note), using the wrong shape will silently fail.

##### Node: Comment tag
- **Type / role:** `httpRequestTool` — posts a comment/note (timeline) to a box.
- **Config:**
  - URL from `$fromAI('URL')` (agent must provide full endpoint, e.g. `https://api.streak.com/api/v2/boxes/{boxKey}/comments`)
  - Method POST
  - JSON body from `$fromAI('JSON')`
  - Auth: Basic Auth.
- **Edge cases:**
  - Agent must choose v2 endpoint for comments (system message suggests v2).
  - Wrong body schema → 400 errors.

---

#### Documentation search tool

##### Node: Streak_Doc
- **Type / role:** `@tavily/n8n-nodes-tavily.tavilyTool` — web research constrained to Streak API reference.
- **Config:**
  - Query from `$fromAI('Query', default: "Use this tool to research on how to approach any streak api endpoint")`
  - Domain allowlist: `https://streak.readme.io/reference`
- **Connection:** `ai_tool` → Agent.
- **Edge cases:**
  - Tavily quota / auth failures.
  - Docs content changes; agent may hallucinate if tool returns partial info.

---

### 2.5 WhatsApp Response

**Overview:**  
Formats and sends the agent result back to the same WhatsApp user.

**Nodes involved**
- Send message

#### Node: Send message
- **Type / role:** `n8n-nodes-base.whatsApp` — sends a WhatsApp message.
- **Config:**
  - Operation: `send`
  - Recipient: `={{ $('WhatsApp Trigger').item.json.contacts[0].wa_id }}`
  - Text body: `={{ $json.output }}`
  - `previewUrl: false`
  - phoneNumberId: `871225226084214`
- **Input:** from **Chat with Streak CRM**.
- **Edge cases:**
  - If agent returns no `output` field, message body becomes empty.
  - Recipient WA ID missing for some webhook events.
  - WhatsApp send limits / template requirements (depending on business account status and conversation window rules).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| WhatsApp Trigger | whatsAppTrigger | Receive incoming WhatsApp messages | — | Route input Types |  |
| Route input Types | switch | Branch by message type (text/audio/image/document) | WhatsApp Trigger | Map text prompt; Gets WhatsApp Voicemail Source URL; Gets WhatsApp Image Source URL |  |
| Map text prompt | set | Normalize text message to `text` field | Route input Types (Text) | Chat with Streak CRM | ## text media type |
| Gets WhatsApp Voicemail Source URL | whatsApp | Fetch WhatsApp media URL for audio | Route input Types (Audio) | Download Voicemail | ## Upload Audio files; To quickly chat with your CRM |
| Download Voicemail | httpRequest | Download audio binary from WhatsApp media URL | Gets WhatsApp Voicemail Source URL | OpenAI | ## Upload Audio files; To quickly chat with your CRM |
| OpenAI | langchain.openAi | Translate audio to text | Download Voicemail | Chat with Streak CRM | ## Upload Audio files; To quickly chat with your CRM |
| Gets WhatsApp Image Source URL | whatsApp | Fetch WhatsApp media URL for image | Route input Types (Image) | Download Image | ## image media type; Upload job cards, or media that could be used to enrich contacts |
| Download Image | httpRequest | Download image binary from WhatsApp media URL | Gets WhatsApp Image Source URL | OpenAI1 | ## image media type; Upload job cards, or media that could be used to enrich contacts |
| OpenAI1 | langchain.openAi | Analyze image to produce description text | Download Image | Map image prompt | ## image media type; Upload job cards, or media that could be used to enrich contacts |
| Map image prompt | set | Build `text` prompt from image description + caption | OpenAI1 | Chat with Streak CRM | ## image media type; Upload job cards, or media that could be used to enrich contacts |
| Simple Memory | memoryBufferWindow | Session memory per WhatsApp user | — | Chat with Streak CRM (ai_memory) |  |
| OpenAI Chat Model | lmChatOpenAi | Primary LLM for agent | — | Chat with Streak CRM (ai_languageModel idx0) |  |
| Google Gemini Chat Model | lmChatGoogleGemini | Fallback LLM for agent | — | Chat with Streak CRM (ai_languageModel idx1) |  |
| Chat with Streak CRM | langchain.agent | Decide actions + call tools + produce final reply | Map text prompt / Map image prompt / OpenAI | Send message |  |
| Get_boxes_details | streakTool | Search/list boxes in pipeline | — | Chat with Streak CRM (ai_tool) |  |
| Update_box_in_Streak | streakTool | Update box (stage/notes/assignee) | — | Chat with Streak CRM (ai_tool) |  |
| Create a contact | streakTool | Create Streak contact | — | Chat with Streak CRM (ai_tool) |  |
| Update a contact | streakTool | Update Streak contact (currently empty updateFields) | — | Chat with Streak CRM (ai_tool) |  |
| Create a box in a pipeline in Streak | streakTool | Create a box in a pipeline | — | Chat with Streak CRM (ai_tool) |  |
| Get_user_in_Streak | streakTool | Fetch user/team keys (operation unspecified) | — | Chat with Streak CRM (ai_tool) |  |
| Get_pipeline-fileds | streakTool | List pipeline fields | — | Chat with Streak CRM (ai_tool) |  |
| Create_task_in_box_Streak | streakTool | Create task in a box | — | Chat with Streak CRM (ai_tool) |  |
| Get a task in Streak | streakTool | Retrieve a task by taskKey | — | Chat with Streak CRM (ai_tool) |  |
| Streak_Doc | tavilyTool | Search Streak API docs | — | Chat with Streak CRM (ai_tool) |  |
| Get_Custom_Operation | httpRequestTool | Custom GET (contact search endpoint builder) | — | Chat with Streak CRM (ai_tool) |  |
| POST_Custom_Operation | httpRequestTool | Custom POST (legacy attempt, not for linking) | — | Chat with Streak CRM (ai_tool) | ## Note; on every http node use streak API version1 V1 endpoint unless the resources you need are only served on version 2 |
| Link_contact_to_box | httpRequestTool | Correct POST to link contact to box | — | Chat with Streak CRM (ai_tool) | ## Note; on every http node use streak API version1 V1 endpoint unless the resources you need are only served on version 2 |
| Comment tag | httpRequestTool | Post comment to box timeline | — | Chat with Streak CRM (ai_tool) | ## Note; on every http node use streak API version1 V1 endpoint unless the resources you need are only served on version 2 |
| Send message | whatsApp | Send final response to WhatsApp user | Chat with Streak CRM | — |  |
| Sticky Note | stickyNote | Comment | — | — | ## text media type |
| Sticky Note1 | stickyNote | Comment | — | — | ## image media type; Upload job cards, or media that could be used to enrich contacts |
| Sticky Note2 | stickyNote | Comment | — | — | ## Upload Audio files; To quickly chat with your CRM |
| Sticky Note3 | stickyNote | Comment | — | — | ### How it works; This AI agent manages your Streak CRM via WhatsApp… (setup + customization + note about docs access) |
| Sticky Note4 | stickyNote | Comment | — | — | ## Note; on every http node use streak API version1 V1 endpoint unless the resources you need are only served on version 2 |

---

## 4. Reproducing the Workflow from Scratch

1) **Create credentials**
   1. **WhatsApp Trigger credential** (WhatsApp Business / Cloud API) for receiving webhooks.
   2. **WhatsApp API credential** for sending messages and downloading media (must authorize media retrieval URLs).
   3. **OpenAI API credential** (for GPT-4.1-mini chat + audio translation + image analysis).
   4. **Google Gemini (PaLM/Gemini) credential** for fallback chat model.
   5. **Streak credential (native Streak node)** for `n8n-nodes-streak-crm` operations.
   6. **HTTP Basic Auth credential** for Streak REST calls:
      - Username typically = Streak API key
      - Password empty (or per your Streak setup)
   7. **Tavily API credential** for Streak documentation search.

2) **Add the trigger**
   1. Add node **WhatsApp Trigger**.
   2. Configure it to listen to **messages** updates.
   3. Connect your WhatsApp Trigger credential.
   4. Activate webhook in WhatsApp developer settings as required by n8n.

3) **Route by message type**
   1. Add a **Switch** node named **Route input Types**.
   2. Add rules on `{{$json.messages[0].type}}`:
      - equals `text`
      - equals `audio`
      - equals `image`
      - (optional) equals `document` (but ensure you connect it if you want support)
   3. Connect: **WhatsApp Trigger → Route input Types**.

4) **Text path**
   1. Add a **Set** node named **Map text prompt**.
   2. Create field:
      - `text` (string) = `{{$json.messages[0].text.body}}`
   3. Connect: **Route input Types (text) → Map text prompt**.

5) **Audio path**
   1. Add **WhatsApp** node named **Gets WhatsApp Voicemail Source URL**:
      - Resource: Media
      - Operation: Get media URL
      - Media ID: `{{$json.messages[0].audio.id}}`
   2. Add **HTTP Request** node named **Download Voicemail**:
      - URL: `{{$json.url}}`
      - Authentication: use predefined credential type → WhatsApp API credential
   3. Add **OpenAI (LangChain OpenAI)** node named **OpenAI**:
      - Resource: Audio
      - Operation: Translate
      - OpenAI credential
   4. Connect:  
      **Route input Types (audio) → Gets WhatsApp Voicemail Source URL → Download Voicemail → OpenAI**

6) **Image path**
   1. Add **WhatsApp** node named **Gets WhatsApp Image Source URL**:
      - Resource: Media
      - Operation: Get media URL
      - Media ID: `{{$json.messages[0].image.id}}`
   2. Add **HTTP Request** node named **Download Image**:
      - URL: `{{$json.url}}`
      - Authentication: predefined credential type → WhatsApp API credential
   3. Add **OpenAI (LangChain OpenAI)** node named **OpenAI1**:
      - Resource: Image
      - Operation: Analyze
      - Input type: base64
      - Model: `gpt-4o-mini`
      - Option detail: auto
   4. Add **Set** node named **Map image prompt**:
      - `text` (string) =
        - `User image description: {{ $json.content }}`
        - `User image caption: {{ $('Route input Types').item.json.messages[0].image.caption }}`
   5. Connect:  
      **Route input Types (image) → Gets WhatsApp Image Source URL → Download Image → OpenAI1 → Map image prompt**

7) **Create the AI models**
   1. Add **OpenAI Chat Model** node (`lmChatOpenAi`):
      - Model: `gpt-4.1-mini`
   2. Add **Google Gemini Chat Model** node (`lmChatGoogleGemini`):
      - Default options are fine
   3. Add **Simple Memory** node (`memoryBufferWindow`):
      - Session ID type: custom key
      - Session key: `memory_{{ $('WhatsApp Trigger').item.json.contacts[0].wa_id }}`
      - Context window length: 20

8) **Create Streak tools (native nodes)**
   Add these `streakTool` nodes and connect each to the agent as **AI Tool**:
   1. **Get_boxes_details**
      - Resource: Box
      - Return all: true
      - Pipeline key: set your pipeline
      - Search query: use `$fromAI('Search_Query')`
   2. **Update_box_in_Streak**
      - Resource: Box → Update box
      - boxKey from `$fromAI('Box_Key')`
      - stageKey / notes / assignedTo… from `$fromAI(...)`
   3. **Create a contact**
      - Resource: Contact → Create
      - teamKey: your Streak team key
      - givenName/familyName/emailAddresses from `$fromAI(...)`
   4. **Update a contact**
      - Resource: Contact → Update
      - contactKey from `$fromAI('Contact_Key')`
      - Configure real update fields you want to allow (recommended; otherwise it’s ineffective)
   5. **Create a box in a pipeline in Streak**
      - Resource: Box → Create
      - pipelineKey set
      - boxName / stageKey / notes / assignedTo… from `$fromAI(...)`
   6. **Get_user_in_Streak**
      - Configure the correct resource/operation (list users) per the Streak node’s UI.
   7. **Get_pipeline-fileds**
      - Resource: Field → list for pipelineKey
   8. **Create_task_in_box_Streak**
      - Resource: Task → Create
      - Needs pipelineKey + boxKey + task text from `$fromAI(...)`
   9. **Get a task in Streak**
      - Resource: Task → Get
      - taskKey from `$fromAI('Task_Key')`

9) **Create HTTP tools (custom Streak endpoints)**
   Add as **HTTP Request Tool** nodes (so the agent can call them):
   1. **Get_Custom_Operation** (GET)
      - URL expression building:
        - `https://api.streak.com/api/v1/search?query={{encodeURIComponent($fromAI('searchQuery'))}}&type=contact`
      - Auth: HTTP Basic Auth (Streak API key)
   2. **POST_Custom_Operation** (POST)
      - URL: `https://api.streak.com/api/v1/boxes/{{$fromAI('boxKey')}}`
      - Body: JSON (note: this node builds a string in the provided workflow; prefer using real JSON to avoid escaping issues)
   3. **Link_contact_to_box** (POST) **(the correct one)**
      - URL: `https://api.streak.com/api/v1/boxes/{{$fromAI('boxKey')}}`
      - Body JSON:
        - `{ "contacts": [ { "key": "<contactKey>" } ] }`
      - Auth: HTTP Basic Auth
   4. **Comment tag** (POST)
      - URL: `$fromAI('URL')` (agent supplies)
      - JSON body: `$fromAI('JSON')`
      - Auth: HTTP Basic Auth
   5. Add **Tavily Tool** node **Streak_Doc**
      - Include domain: `https://streak.readme.io/reference`

10) **Create the agent**
   1. Add **AI Agent** node named **Chat with Streak CRM**.
   2. Set input text to `{{$json.text}}`.
   3. Paste/configure the **system message** (ensure it includes the “CRITICAL: LINKING CONTACTS…” rule and your pipeline stage IDs).
   4. Enable fallback (`needsFallback`) and connect:
      - OpenAI Chat Model as primary language model
      - Gemini as secondary language model
   5. Connect memory: **Simple Memory → Agent (AI Memory)**
   6. Connect tools: all Streak nodes + HTTP tools + Tavily tool → Agent (AI Tool).

11) **Wire inputs into the agent**
   - **Map text prompt → Agent**
   - **Map image prompt → Agent**
   - **OpenAI (audio translation output) → Agent**  
     (If needed, add a Set node after audio translation to map translated text into `text` so the agent input is consistent.)

12) **Send response to WhatsApp**
   1. Add **WhatsApp** node named **Send message**:
      - Operation: Send
      - Recipient: `{{ $('WhatsApp Trigger').item.json.contacts[0].wa_id }}`
      - Text: `{{ $json.output }}`
   2. Connect: **Agent → Send message**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “This AI agent manages your Streak CRM via WhatsApp… Send messages to create contacts, add boxes to pipelines, update stages, link contacts to boxes, and retrieve information, add and get tasks…” | Sticky note “How it works” (high-level behavior + suggested setup and customization) |
| “Connect your Streak API credentials (Basic Auth)… Configure WhatsApp Business integration… Add your Streak pipeline keys… Test with ‘create contact John Doe’ ” | Sticky note “Setup steps” |
| “The agent worked perfectly when you give it access to streak API documentation…” | Sticky note “Customization tips” (reinforces using the Tavily/Streak_Doc tool) |
| “on every http node use streak API version1 V1 endpoint unless the resources you need are only served on version 2” | Sticky note “Note” (applies to Comment tag / Link_contact_to_box / custom HTTP operations; comments may require v2 depending on endpoint) |

French disclaimer (as provided):  
Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.