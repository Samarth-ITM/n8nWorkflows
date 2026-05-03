Automatic Instagram content creation with OpenAI, Nano Banana and Telegram check

https://n8nworkflows.xyz/workflows/automatic-instagram-content-creation-with-openai--nano-banana-and-telegram-check-11985


# Automatic Instagram content creation with OpenAI, Nano Banana and Telegram check

## 1. Workflow Overview

**Purpose:**  
This workflow automates Instagram content production and publishing with an approval loop in Telegram. It generates a caption and an image (via OpenAI + “Nano Banana/OpenBanana” image endpoint), stores assets/metadata in Airtable, sends the draft to Telegram, and (upon Telegram button callback) either publishes to Instagram, cancels, or regenerates content.

**Target use cases:**
- Daily/recurring Instagram post generation with human-in-the-loop approval
- Content pipelines that must log drafts and final outcomes in Airtable
- Controlled publishing (confirm/cancel/regenerate) via Telegram

### Logical blocks
1.1 **Triggers & Entry Points** (Schedule, Manual, Webhook, Telegram callback)  
1.2 **Historic context retrieval** (fetch prior posts, map them, categorize)  
1.3 **AI content generation** (OpenAI agent produces structured output)  
1.4 **Image generation & preparation** (OpenBanana request → base64 → JPG)  
1.5 **Persistence & draft delivery** (Airtable record + upload image + Telegram draft)  
1.6 **Telegram approval routing** (callback processing → confirm/cancel/regenerate)  
1.7 **Instagram publishing** (prepare container → wait → status check → publish)  
1.8 **Outcome logging & error handling** (finish/fail Telegram messages, Airtable updates, stop)

---

## 2. Block-by-Block Analysis

### 1.1 Triggers & Entry Points
**Overview:** Provides multiple ways to start the automation: scheduled generation, manual testing, a GET webhook, and Telegram callback events for approvals.  
**Nodes involved:** Schedule Trigger, When clicking ‘Execute workflow’, Webhook Serve (GET), Telegram Trigger

#### Node: Schedule Trigger
- **Type/role:** `scheduleTrigger` – timed entry point
- **Config (interpreted):** Schedule not provided in JSON; typically cron/interval-based.
- **I/O:**  
  - Output → Call 'Main Content‑Generation Workflow'
- **Failure/edge cases:** n8n timezone mismatch; missed runs during downtime.
- **Version:** 1.2

#### Node: When clicking ‘Execute workflow’
- **Type/role:** `manualTrigger` – manual entry for testing
- **I/O:** Output → Input
- **Failure/edge cases:** none (manual only)
- **Version:** 1

#### Node: Webhook Serve (GET)
- **Type/role:** `webhook` – external HTTP trigger (GET)
- **Config:** Parameters empty in export; webhook path/method implied by node type and `webhookId`.
- **I/O:** Output → Get a record → Respond to Webhook
- **Failure/edge cases:** unauthenticated endpoint if not protected; missing query params; retries/timeouts from caller.
- **Version:** 1

#### Node: Telegram Trigger
- **Type/role:** `telegramTrigger` – receives Telegram updates (likely callback queries from inline buttons)
- **I/O:** Output → Process Callback
- **Failure/edge cases:** Telegram webhook misconfigured; bot token invalid; callback payload shape changes.
- **Version:** 1.2

---

### 1.2 Historic context retrieval
**Overview:** Pulls previously published/drafted content to provide context and avoid repetition, then shapes it into an input for categorization and prompting.  
**Nodes involved:** Input, Get Historic Posts, Map Historic Data, Define Category

#### Node: Input
- **Type/role:** `code` – constructs initial input payload for generation flow
- **Config:** Not provided; typically sets topic/brand/constraints.
- **I/O:** Output → Get Historic Posts
- **Failure/edge cases:** code exceptions; missing expected fields for later nodes.
- **Version:** 2

#### Node: Get Historic Posts
- **Type/role:** `httpRequest` – fetches historical post data from an external API (or Airtable/DB proxy)
- **Config:** Not provided (URL/auth unknown).
- **I/O:** Output → Map Historic Data
- **Failure/edge cases:** 401/403 auth; rate limits; response schema changes; pagination not handled if needed.
- **Version:** 4.2

#### Node: Map Historic Data
- **Type/role:** `code` – transforms historic response into compact context (e.g., list of captions/topics)
- **Config:** `alwaysOutputData: false` means it may output nothing if mapping fails or no items.
- **I/O:** Output → Define Category
- **Failure/edge cases:** empty history causes downstream prompt to lack context; code errors on unexpected shapes.
- **Version:** 2

#### Node: Define Category
- **Type/role:** `code` – chooses/derives a content category for the next post (based on history or rotation)
- **I/O:** Output → AI Agent
- **Failure/edge cases:** category not in allowed set; missing category leads to weak prompts.
- **Version:** 2

---

### 1.3 AI content generation
**Overview:** Uses an OpenAI-backed LangChain Agent to generate structured content (caption + image prompt), then parses it into JSON for deterministic downstream steps.  
**Nodes involved:** AI Agent, OpenAI Chat Model, Parse String to Json, Take Caption, Take Image Prompt

#### Node: OpenAI Chat Model
- **Type/role:** `lmChatOpenAi` – LLM provider for the Agent
- **Config:** Not included; typically model (e.g., gpt-4o-mini), temperature, API key credential.
- **Connections:** Provides model to AI Agent via `ai_languageModel`.
- **Failure/edge cases:** invalid API key; model not available; token limits; latency/timeouts.
- **Version:** 1.2

#### Node: AI Agent
- **Type/role:** `@n8n/n8n-nodes-langchain.agent` – orchestrates prompt/tool logic to produce final text output
- **Config:** Not included; likely instructed to output JSON text with keys like `caption` and `image_prompt`.
- **I/O:** Input from Define Category; Output → Parse String to Json
- **Failure/edge cases:** non-JSON output; hallucinated fields; refusals; overly long responses.
- **Version:** 2.2

#### Node: Parse String to Json
- **Type/role:** `code` – parses the agent’s text output into JSON
- **Config:** Not included; typically `JSON.parse($json.text)` or similar with cleanup.
- **I/O:** Output → Take Caption
- **Failure/edge cases:** invalid JSON (trailing commas, markdown fences); encoding issues.
- **Version:** 2

#### Node: Take Caption
- **Type/role:** `set` – isolates the caption field for later use
- **I/O:** Output → Take Image Prompt
- **Failure/edge cases:** caption field missing; wrong path.
- **Version:** 3.4

#### Node: Take Image Prompt
- **Type/role:** `set` – isolates the image prompt field for the image generation API
- **I/O:** Output → OpenBanana Request
- **Failure/edge cases:** prompt empty; unsafe prompt rejected by provider.
- **Version:** 3.4

---

### 1.4 Image generation & preparation
**Overview:** Generates an image from the prompt via “OpenBanana/Nano Banana” HTTP API, then converts base64 payload into a JPG binary n8n can upload.  
**Nodes involved:** OpenBanana Request, Base64 to jpg

#### Node: OpenBanana Request
- **Type/role:** `httpRequest` – calls image generation endpoint
- **Config:** Not provided (URL/body/auth unknown). Output likely includes base64 image.
- **I/O:** Output → Base64 to jpg
- **Failure/edge cases:** provider 429/rate limit; payload too large; invalid base64; API key issues.
- **Version:** 4.2

#### Node: Base64 to jpg
- **Type/role:** `code` – converts base64 string to n8n binary (JPG)
- **I/O:** Output → Create new record
- **Failure/edge cases:** base64 missing; wrong mime; large memory usage.
- **Version:** 2

---

### 1.5 Persistence & draft delivery
**Overview:** Creates an Airtable record for the draft, uploads the image into Airtable, then sends the preview to Telegram (photo + text).  
**Nodes involved:** Create new record, Upload Image To Airtable, Send a photo message, Send a text message

#### Node: Create new record
- **Type/role:** `airtable` – stores draft metadata (caption, prompt, category, status, etc.)
- **Config:** not included (base/table/fields unknown).
- **I/O:** Input from Base64 to jpg; Output → Upload Image To Airtable
- **Failure/edge cases:** Airtable auth error; field mismatch; rate limits; attachment size constraints.
- **Version:** 2.1

#### Node: Upload Image To Airtable
- **Type/role:** `httpRequest` – uploads binary/attachment to Airtable (often via Airtable API attachment field update)
- **Config:** not included.
- **I/O:** Output → Send a photo message
- **Failure/edge cases:** Airtable attachment requires public URL (Airtable API does not accept raw binary directly in all cases); request formatting issues; 413 payload too large.
- **Version:** 4.2

#### Node: Send a photo message
- **Type/role:** `telegram` – sends image preview to a Telegram chat
- **Config:** not included (chat id, photo source, inline keyboard likely).
- **I/O:** Output → Send a text message
- **Failure/edge cases:** wrong chat id; bot blocked; invalid file_id/url; Telegram rate limits.
- **Version:** 1.2

#### Node: Send a text message
- **Type/role:** `telegram` – sends caption and/or action buttons (confirm/cancel/regenerate)
- **Config:** not included.
- **I/O:** terminal for draft creation branch
- **Failure/edge cases:** message too long; markdown parse errors if formatting enabled.
- **Version:** 1.2

---

### 1.6 Telegram approval routing (callback)
**Overview:** Handles Telegram callback queries, routes by action (confirm/cancel/regenerate), edits the original message accordingly, and updates Airtable state.  
**Nodes involved:** Process Callback, Switch, Edit Confirm message, Edit cancel message, Edit regenerate message, Update record - cancel, Get a record1, Extract Input Data, Update record - regenerate, Call 'Main Content‑Generation Workflow'1

#### Node: Process Callback
- **Type/role:** `code` – extracts callback data (action, record id/message ids) from Telegram update payload
- **I/O:** Output → Switch
- **Failure/edge cases:** payload differs for messages vs callback_query; missing `callback_data`.
- **Version:** 2

#### Node: Switch
- **Type/role:** `switch` – routes actions into 3 branches
- **Config:** not shown; assumed cases: confirm / cancel / regenerate.
- **I/O:**  
  - Output 1 → Edit Confirm  message  
  - Output 2 → Edit cancel message  
  - Output 3 → Edit regenerate message
- **Failure/edge cases:** unmatched action goes nowhere; case sensitivity.
- **Version:** 3.2

#### Node: Edit cancel message
- **Type/role:** `telegram` – edits Telegram message to reflect cancellation
- **I/O:** Output → Update record - cancel
- **Failure/edge cases:** message no longer editable; wrong message_id/chat_id.
- **Version:** 1.2

#### Node: Update record - cancel
- **Type/role:** `airtable` – updates Airtable record status to “cancelled”
- **I/O:** terminal for cancel branch
- **Failure/edge cases:** record not found; Airtable rate limits.
- **Version:** 2.1

#### Node: Edit regenerate message
- **Type/role:** `telegram` – edits message to reflect regeneration request
- **I/O:** Output → Get a record1
- **Failure/edge cases:** same as edit cancel.
- **Version:** 1.2

#### Node: Get a record1
- **Type/role:** `airtable` – retrieves the record to regenerate (to reuse inputs/category/constraints)
- **I/O:** Output → Extract Input Data
- **Failure/edge cases:** record deleted; Airtable auth.
- **Version:** 2.1

#### Node: Extract Input Data
- **Type/role:** `code` – builds the input payload required to regenerate content
- **I/O:** Output → Update record - regenerate
- **Failure/edge cases:** missing fields; incompatible schema with generation workflow.
- **Version:** 2

#### Node: Update record - regenerate
- **Type/role:** `airtable` – marks record as “regenerating” (or similar)
- **I/O:** Output → Call 'Main Content‑Generation Workflow'1
- **Failure/edge cases:** partial updates; race conditions if multiple callbacks.
- **Version:** 2.1

#### Node: Call 'Main Content‑Generation Workflow'1
- **Type/role:** `executeWorkflow` – calls the same main generation routine as a sub-workflow invocation
- **Config:** workflow reference not included; expected to accept the extracted input and create a new draft.
- **I/O:** terminal for regenerate branch (in this workflow)
- **Failure/edge cases:** sub-workflow not found; input contract mismatch; execution timeouts.
- **Version:** 1.2
- **Sub-workflow reference:** “Main Content‑Generation Workflow” (invoked)

---

### 1.7 Instagram publishing (confirm branch)
**Overview:** On confirm, fetches caption/image info, prepares an Instagram post (likely via Graph API container creation), waits, checks status, and publishes; then notifies Telegram and updates Airtable.  
**Nodes involved:** Edit Confirm message, Get caption, Prepare IG Post, Wait for the preparation of the post, Get Status For Publish, If, Publish Post, Send Finish Message, Update record - confirm, Send Fail Message, Stop and Error

#### Node: Edit Confirm  message
- **Type/role:** `telegram` – edits original message to reflect confirmation
- **I/O:** Output → Get caption
- **Failure/edge cases:** non-editable message; wrong ids.
- **Version:** 1.2

#### Node: Get caption
- **Type/role:** `airtable` – retrieves caption (and likely image/IG media URL/container params)
- **I/O:** Output → Prepare IG Post
- **Failure/edge cases:** missing caption/media URL; record not found.
- **Version:** 2.1

#### Node: Prepare IG Post
- **Type/role:** `httpRequest` – creates an IG media container / initiates post preparation
- **Config:** not shown (likely Facebook/Instagram Graph API).
- **I/O:** Output → Wait for the preparation of the post
- **Failure/edge cases:** Graph API auth/token expiry; permissions missing; invalid image URL; 400 errors.
- **Version:** 4.2

#### Node: Wait for the preparation of the post
- **Type/role:** `wait` – pauses to allow IG container processing
- **Config:** not shown; could be fixed delay or webhook-based resume (node has `webhookId`).
- **I/O:** Output → Get Status For Publish
- **Failure/edge cases:** too short delay leads to not-ready status; wait timeouts.
- **Version:** 1.1

#### Node: Get Status For Publish
- **Type/role:** `httpRequest` – checks IG container status
- **I/O:** Output → If
- **Failure/edge cases:** transient Graph API errors; status field missing.
- **Version:** 4.2

#### Node: If
- **Type/role:** `if` – decides success path (ready) vs fail path
- **Config:** not shown; likely checks status == “FINISHED” / “READY”.
- **I/O:**  
  - True → Publish Post  
  - False → Send Fail Message
- **Failure/edge cases:** status values differ (“FINISHED” vs “READY”); null status.
- **Version:** 2.2

#### Node: Publish Post
- **Type/role:** `httpRequest` – publishes the prepared container
- **I/O:** Output → Send Finish Message
- **Failure/edge cases:** publish permissions missing; API 400/403; container expired.
- **Version:** 4.2

#### Node: Send Finish Message
- **Type/role:** `telegram` – notifies that publishing succeeded
- **I/O:** Output → Update record - confirm
- **Failure/edge cases:** Telegram message failure; chat id mismatch.
- **Version:** 1.2

#### Node: Update record - confirm
- **Type/role:** `airtable` – marks Airtable record as published/confirmed, stores IG post id/url
- **I/O:** terminal for confirm branch
- **Failure/edge cases:** record lock/rate limit; missing IG identifiers.
- **Version:** 2.1

#### Node: Send Fail Message
- **Type/role:** `telegram` – sends failure notification
- **I/O:** Output → Stop and Error
- **Failure/edge cases:** Telegram failure hides the underlying IG error unless logged elsewhere.
- **Version:** 1.2

#### Node: Stop and Error
- **Type/role:** `stopAndError` – halts workflow with an error state
- **I/O:** terminal
- **Failure/edge cases:** none; intended termination.
- **Version:** 1

---

### 1.8 Webhook record lookup helper
**Overview:** A small GET-webhook path that reads an Airtable record and responds to the caller. Often used for integrations or debugging.  
**Nodes involved:** Webhook Serve (GET), Get a record, Respond to Webhook

#### Node: Get a record
- **Type/role:** `airtable` – fetches a single record (likely by id passed in webhook)
- **I/O:** Output → Respond to Webhook
- **Failure/edge cases:** missing record id; Airtable auth; 404.
- **Version:** 2.1

#### Node: Respond to Webhook
- **Type/role:** `respondToWebhook` – returns data to HTTP caller
- **Config:** not shown; likely returns JSON record.
- **I/O:** terminal
- **Failure/edge cases:** response too large; wrong content-type; attempts to respond twice (not here).
- **Version:** 1.4

---

### 1.9 Scheduled sub-workflow execution
**Overview:** The schedule trigger calls a separate “Main Content‑Generation Workflow”, indicating this workflow may be an orchestrator or shares generation logic via execute-workflow nodes.  
**Nodes involved:** Schedule Trigger, Call 'Main Content‑Generation Workflow'

#### Node: Call 'Main Content‑Generation Workflow'
- **Type/role:** `executeWorkflow` – invokes external workflow
- **Config:** not provided (workflow id/name and input mapping unknown).
- **I/O:** Input from Schedule Trigger; output not used further in this JSON.
- **Failure/edge cases:** referenced workflow missing; credential context differences; input schema mismatch.
- **Version:** 1.2
- **Sub-workflow reference:** “Main Content‑Generation Workflow” (invoked)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Webhook Serve (GET) | Webhook | HTTP GET entry point | — | Get a record |  |
| Get a record | Airtable | Fetch single Airtable record for webhook | Webhook Serve (GET) | Respond to Webhook |  |
| Respond to Webhook | Respond to Webhook | Return webhook response | Get a record | — |  |
| When clicking ‘Execute workflow’ | Manual Trigger | Manual entry point | — | Input |  |
| Input | Code | Build initial generation inputs | When clicking ‘Execute workflow’ | Get Historic Posts |  |
| Get Historic Posts | HTTP Request | Retrieve historic posts/context | Input | Map Historic Data |  |
| Map Historic Data | Code | Transform history into context | Get Historic Posts | Define Category |  |
| Define Category | Code | Choose category/angle | Map Historic Data | AI Agent |  |
| OpenAI Chat Model | OpenAI Chat Model (LangChain) | LLM provider | — | AI Agent (ai_languageModel) |  |
| AI Agent | LangChain Agent | Generate caption + image prompt | Define Category | Parse String to Json |  |
| Parse String to Json | Code | Parse agent output to JSON | AI Agent | Take Caption |  |
| Take Caption | Set | Select caption field | Parse String to Json | Take Image Prompt |  |
| Take Image Prompt | Set | Select image prompt field | Take Caption | OpenBanana Request |  |
| OpenBanana Request | HTTP Request | Generate image from prompt | Take Image Prompt | Base64 to jpg |  |
| Base64 to jpg | Code | Convert base64 to JPG binary | OpenBanana Request | Create new record |  |
| Create new record | Airtable | Create draft record | Base64 to jpg | Upload Image To Airtable |  |
| Upload Image To Airtable | HTTP Request | Upload/attach image to Airtable | Create new record | Send a photo message |  |
| Send a photo message | Telegram | Send image preview to Telegram | Upload Image To Airtable | Send a text message |  |
| Send a text message | Telegram | Send caption + actions | Send a photo message | — |  |
| Schedule Trigger | Schedule Trigger | Scheduled entry point | — | Call 'Main Content‑Generation Workflow' |  |
| Call 'Main Content‑Generation Workflow' | Execute Workflow | Run external generation workflow | Schedule Trigger | — |  |
| Telegram Trigger | Telegram Trigger | Receive callback queries | — | Process Callback |  |
| Process Callback | Code | Parse callback data | Telegram Trigger | Switch |  |
| Switch | Switch | Route confirm/cancel/regenerate | Process Callback | Edit Confirm message; Edit cancel message; Edit regenerate message |  |
| Edit Confirm  message | Telegram | Edit message on confirm | Switch | Get caption |  |
| Get caption | Airtable | Fetch record fields for IG | Edit Confirm  message | Prepare IG Post |  |
| Prepare IG Post | HTTP Request | Create IG container/prepare | Get caption | Wait for the preparation of the post |  |
| Wait for the preparation of the post | Wait | Delay/await IG processing | Prepare IG Post | Get Status For Publish |  |
| Get Status For Publish | HTTP Request | Check container status | Wait for the preparation of the post | If |  |
| If | If | Ready? publish vs fail | Get Status For Publish | Publish Post; Send Fail Message |  |
| Publish Post | HTTP Request | Publish IG post | If (true) | Send Finish Message |  |
| Send Finish Message | Telegram | Notify publish success | Publish Post | Update record - confirm |  |
| Update record - confirm | Airtable | Mark record as published | Send Finish Message | — |  |
| Send Fail Message | Telegram | Notify publish failure | If (false) | Stop and Error |  |
| Stop and Error | Stop and Error | Fail workflow | Send Fail Message | — |  |
| Edit cancel message | Telegram | Edit message on cancel | Switch | Update record - cancel |  |
| Update record - cancel | Airtable | Mark record cancelled | Edit cancel message | — |  |
| Edit regenerate message | Telegram | Edit message on regenerate | Switch | Get a record1 |  |
| Get a record1 | Airtable | Fetch record to regenerate | Edit regenerate message | Extract Input Data |  |
| Extract Input Data | Code | Build regeneration input | Get a record1 | Update record - regenerate |  |
| Update record - regenerate | Airtable | Mark regenerating | Extract Input Data | Call 'Main Content‑Generation Workflow'1 |  |
| Call 'Main Content‑Generation Workflow'1 | Execute Workflow | Invoke external generation workflow | Update record - regenerate | — |  |
| Sticky Note | Sticky Note | Comment | — | — |  |
| Sticky Note1 | Sticky Note | Comment | — | — |  |
| Sticky Note2 | Sticky Note | Comment | — | — |  |
| Sticky Note3 | Sticky Note | Comment | — | — |  |
| Sticky Note4 | Sticky Note | Comment | — | — |  |
| Sticky Note6 | Sticky Note | Comment | — | — |  |
| Sticky Note7 | Sticky Note | Comment | — | — |  |
| Sticky Note8 | Sticky Note | Comment | — | — |  |

> Note: All sticky notes in the provided JSON have **empty content**, so the “Sticky Note” column remains blank for operational nodes.

---

## 4. Reproducing the Workflow from Scratch

1) **Create triggers**
1. Add **Schedule Trigger** node  
   - Configure desired interval/cron and timezone.
2. Add **Manual Trigger** node (“When clicking ‘Execute workflow’”).
3. Add **Webhook** node (“Webhook Serve (GET)”)  
   - Method: **GET**  
   - Path: choose a path (e.g., `/ig-draft`)  
   - (Optional) Enable authentication (recommended).
4. Add **Telegram Trigger** node  
   - Connect Telegram bot credentials (Bot Token).  
   - Configure updates to include **callback_query** if using inline buttons.

2) **Manual generation branch (draft creation)**
5. Add **Code** node (“Input”) after Manual Trigger  
   - Output fields you will use downstream (e.g., `brand`, `language`, `constraints`).
6. Add **HTTP Request** node (“Get Historic Posts”)  
   - Configure URL/auth to fetch prior posts (or query Airtable via API).  
7. Add **Code** node (“Map Historic Data”)  
   - Transform the historic response into a compact list (e.g., last N captions, topics).
8. Add **Code** node (“Define Category”)  
   - Decide `category` / `angle` for the next post.
9. Add **OpenAI Chat Model** node  
   - Set OpenAI credentials (API key).  
   - Choose model and parameters (temperature/token limits).
10. Add **AI Agent** node  
   - Connect **OpenAI Chat Model** into the agent’s **Language Model** input.  
   - Prompt the agent to output **valid JSON** containing at least:  
     - `caption` (string)  
     - `image_prompt` (string)
11. Add **Code** node (“Parse String to Json”)  
   - Parse agent output text into JSON; strip markdown fences if needed.
12. Add **Set** node (“Take Caption”)  
   - Keep only `caption` (and any record metadata you need).
13. Add **Set** node (“Take Image Prompt”)  
   - Keep only `image_prompt`.

3) **Image generation**
14. Add **HTTP Request** node (“OpenBanana Request”)  
   - Configure image generation endpoint and auth.  
   - Send `image_prompt` as body parameter.  
   - Ensure response includes a base64 image field (or URL you can download).
15. Add **Code** node (“Base64 to jpg”)  
   - Convert base64 string into n8n binary data (`items[0].binary...`) with mime `image/jpeg`.

4) **Airtable persistence**
16. Add **Airtable** node (“Create new record”)  
   - Configure Airtable credentials (Personal Access Token).  
   - Select Base + Table (e.g., `InstagramPosts`).  
   - Map fields like `Caption`, `ImagePrompt`, `Category`, `Status = Draft`, timestamps, etc.
17. Add **HTTP Request** node (“Upload Image To Airtable”)  
   - Implement attachment update strategy: commonly Airtable attachment fields require a **public URL**.  
   - If your image is binary, first upload it to a public storage (S3, Cloudinary, etc.), then PATCH Airtable with that URL.  
   - (This workflow shows direct HTTP request; ensure your approach matches Airtable constraints.)

5) **Telegram draft delivery**
18. Add **Telegram** node (“Send a photo message”)  
   - Send the generated image (file/binary or URL).  
   - Include inline keyboard buttons: **Confirm**, **Cancel**, **Regenerate** with `callback_data` containing action + Airtable record id.
19. Add **Telegram** node (“Send a text message”)  
   - Send caption and/or additional metadata (category, record id, etc.).

6) **Webhook helper path**
20. From **Webhook Serve (GET)** add **Airtable → Get a record**  
   - Read record id from query string (e.g., `?id=recXXXX`).
21. Add **Respond to Webhook** to return the record JSON (or selected fields).

7) **Telegram callback handling**
22. After **Telegram Trigger**, add **Code** node (“Process Callback”)  
   - Extract: `action`, `recordId`, `chatId`, `messageId`.
23. Add **Switch** node to route by `action`:
   - Case `confirm` → Edit Confirm message
   - Case `cancel` → Edit cancel message
   - Case `regenerate` → Edit regenerate message

8) **Cancel branch**
24. Add **Telegram** node (“Edit cancel message”) to update the original message.
25. Add **Airtable** node (“Update record - cancel”) to set `Status = Cancelled`.

9) **Regenerate branch**
26. Add **Telegram** node (“Edit regenerate message”).
27. Add **Airtable** node (“Get a record1”) to fetch full record details.
28. Add **Code** node (“Extract Input Data”) to rebuild generation inputs.
29. Add **Airtable** node (“Update record - regenerate”) to set `Status = Regenerating`.
30. Add **Execute Workflow** node (“Call 'Main Content‑Generation Workflow'1”)  
   - Select the external workflow to run.  
   - Pass reconstructed inputs as the called workflow’s input.

10) **Confirm branch (Instagram publish)**
31. Add **Telegram** node (“Edit Confirm message”).
32. Add **Airtable** node (“Get caption”) to fetch caption/media URL fields needed for IG.
33. Add **HTTP Request** node (“Prepare IG Post”)  
   - Implement IG Graph API container creation (image URL + caption).
34. Add **Wait** node (“Wait for the preparation of the post”)  
   - Use a fixed delay or polling loop approach.
35. Add **HTTP Request** node (“Get Status For Publish”) to poll container status.
36. Add **If** node to check readiness.
37. If ready: **HTTP Request** node (“Publish Post”) to publish container.
38. Add **Telegram** node (“Send Finish Message”).
39. Add **Airtable** node (“Update record - confirm”) to set `Status = Published` and store IG post id/url.
40. If not ready/fails: **Telegram** node (“Send Fail Message”) → **Stop and Error** node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided by user |

