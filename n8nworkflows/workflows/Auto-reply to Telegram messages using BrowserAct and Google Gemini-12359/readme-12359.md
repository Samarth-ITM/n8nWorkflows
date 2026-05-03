Auto-reply to Telegram messages using BrowserAct and Google Gemini

https://n8nworkflows.xyz/workflows/auto-reply-to-telegram-messages-using-browseract-and-google-gemini-12359


# Auto-reply to Telegram messages using BrowserAct and Google Gemini

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

## 1. Workflow Overview

**Title:** Auto-reply to Telegram messages using BrowserAct & Google Gemini  
**Purpose:** An automated Telegram “support agent” that, every 15 minutes, retrieves unread/new Telegram messages via BrowserAct, drafts a personalized reply using Google Gemini (via LangChain nodes), and sends the reply back through BrowserAct (web UI automation).  
**Primary use cases:** Always-on Telegram customer support, lead qualification, community moderation, first-response automation with context from chat history.

### 1.1 Trigger & Telegram Retrieval (BrowserAct)
Runs on a schedule, launches a BrowserAct workflow that reads Telegram messages, and handles “paused” states (human verification/captcha) by waiting and polling task status.

### 1.2 Normalization & Itemization
Takes BrowserAct output (a JSON **string** containing an array), parses it, and splits it into individual n8n items (one per conversation/user).

### 1.3 AI Drafting (Gemini + Structured Output)
For each item, sends user context to a LangChain Agent backed by **Google Gemini**, forces a strict JSON response via a structured output parser, and decides whether a response is needed (`answer` vs `idle`).

### 1.4 Reply Delivery (BrowserAct)
If `answer`, cleans newline escapes, runs a second BrowserAct workflow to type/send the message in Telegram, and again handles human verification by waiting/polling.

---

## 2. Block-by-Block Analysis

### Block 1 — Scheduled trigger + read Telegram via BrowserAct (with verification handling)
**Overview:** Starts every 15 minutes, triggers BrowserAct “read Telegram” workflow, then routes based on task status (`finished` vs `paused`) to either proceed or wait/poll until verification completes.

**Nodes involved:**
- Every 15 minutes
- Read Telegram Message
- Human Verification Switch
- Give Time to Complete Verification
- Get Data From BrowserAct

#### Node: **Every 15 minutes**
- **Type / Role:** Schedule Trigger (cron-like)
- **Config:** Runs every 15 minutes.
- **Outputs:** To **Read Telegram Message**
- **Failure modes:** n8n schedule disabled, timezone expectations.

#### Node: **Read Telegram Message**
- **Type / Role:** BrowserAct node (runs a BrowserAct workflow)
- **Config choices:**
  - Executes BrowserAct **WORKFLOW** with `workflowId: 69828791477180147`
  - Uses BrowserAct API credential: **BrowserAct account**
  - `open_incognito_mode: false`
  - Input schema includes `input-Telegram` but it is marked removed; BrowserAct default inside template is used.
- **Output:** Produces a BrowserAct task object including `id` and `status`.
- **Connections:** To **Human Verification Switch**
- **Failure modes / edge cases:**
  - BrowserAct auth/key invalid
  - Workflow ID missing/wrong
  - Telegram session not logged in inside BrowserAct template
  - BrowserAct returns `paused` for captcha/2FA/human verification

#### Node: **Human Verification Switch**
- **Type / Role:** Switch (routes by task status)
- **Logic:**
  - If `{{$json.status}} == "finished"` → continue processing
  - If `{{$json.status}} == "paused"` → wait and poll
- **Outputs:**
  - Finished → **Splitting Extracted Items** (via connections; see Block 2)
  - Paused → **Give Time to Complete Verification**
- **Failure modes:** Status missing/renamed; unexpected statuses (e.g., `failed`) are not handled explicitly.

#### Node: **Give Time to Complete Verification**
- **Type / Role:** Wait
- **Config:** Wait unit is minutes (duration not specified in parameters; typically configured in UI or defaults).
- **Output:** To **Get Data From BrowserAct**
- **Edge cases:** If duration is too short, may loop too fast; too long delays responses.

#### Node: **Get Data From BrowserAct**
- **Type / Role:** HTTP Request (poll BrowserAct task)
- **Config choices:**
  - GET `https://api.browseract.com/v2/workflow/get-task`
  - Query param `task_id = {{ $('Read Telegram Message').item.json.id }}`
  - Auth: predefined credential type `browserActApi` (BrowserAct account)
- **Output:** Updated task payload including status/output.
- **Connections:** To **Splitting Extracted Items**
- **Failure modes:**
  - Wrong task_id reference if items diverge
  - API rate limits / transient 5xx
  - Task output not ready yet (still paused/running) — not explicitly re-switched here (it goes directly to splitting)

---

### Block 2 — Parse BrowserAct output string into multiple items
**Overview:** Converts BrowserAct’s extracted data (a JSON string representing an array) into discrete n8n items, then batches/loops over them.

**Nodes involved:**
- Splitting Extracted Items
- Loop Over Items

#### Node: **Splitting Extracted Items**
- **Type / Role:** Code (JavaScript) — parsing + fan-out
- **Key logic:**
  - Reads: `$input.first().json.output.string`
  - Throws if missing/empty
  - `JSON.parse(jsonString)`
  - Ensures parsed data is an array
  - Returns `parsedData.map(item => ({ json: item }))`
- **Input:** From **Get Data From BrowserAct** or directly from **Human Verification Switch** (finished path)
- **Output:** Multiple items → **Loop Over Items**
- **Failure modes / edge cases:**
  - BrowserAct output path differs (no `output.string`)
  - Output is already object/array (not a string) → parse fails
  - Malformed JSON string
  - Parsed data is not an array

#### Node: **Loop Over Items**
- **Type / Role:** Split In Batches — controls iteration over many chats/messages
- **Config:** Default options (batch size not explicitly set in JSON; n8n defaults apply).
- **Connections:**
  - Output 1 is unused in this workflow wiring
  - Output 2 → **Generate Answers for Messages**
  - Also receives inputs back from downstream nodes to continue looping (see below)
- **Loop mechanics in this workflow:**
  - After “idle” decisions and after completion wait, execution routes back into this node to proceed with next item.
- **Failure modes:** Incorrect loop wiring can cause infinite loops or skipped items; batch size too large may cause slow AI calls.

---

### Block 3 — AI response drafting with Google Gemini + structured JSON output
**Overview:** For each conversation item, Gemini drafts either an “answer” or “idle” response in a strict JSON schema, then the workflow branches based on whether a response is needed.

**Nodes involved:**
- Google Gemini
- Structured Output
- Generate Answers for Messages
- Check If Message Requires Response

#### Node: **Google Gemini**
- **Type / Role:** LangChain Chat Model (Google Gemini / PaLM credential)
- **Config:** Default options; uses credential **Google Gemini(PaLM) Api account**
- **Connections:** Provides the LLM to:
  - **Generate Answers for Messages** (as `ai_languageModel`)
  - **Structured Output** (also wired as `ai_languageModel` in this workflow)
- **Failure modes:** Invalid API key, quota limits, safety blocks, latency/timeouts.

#### Node: **Structured Output**
- **Type / Role:** LangChain Structured Output Parser
- **Config choices:**
  - `autoFix: true` (attempts to repair near-valid JSON)
  - JSON schema example enforces keys: `name`, `type`, `Answer`
- **Connections:** Feeds as `ai_outputParser` into **Generate Answers for Messages**
- **Failure modes:** Model returns non-JSON, missing keys, or incompatible types; autoFix can still fail on heavily malformed output.

#### Node: **Generate Answers for Messages**
- **Type / Role:** LangChain Agent
- **Config choices:**
  - Prompt uses item fields:
    - `Name: {{$json.Name}}`
    - `Last Message: {{$json.Last}}`
    - `User Chat history: {{$json.UserHistory}}`
    - `Bot Response history: {{$json.BotHistory}}`
  - System message instructs:
    - Decide `type`: `"answer"` if `Last` has content; else `"idle"`
    - Must mention the user `Name`
    - Must append footer: `\n\n[This is an automated bot and will answer every 15 minutes]`
    - Output must be raw JSON with exactly keys: `name`, `type`, `Answer`
  - Has output parser enabled (`hasOutputParser: true`)
- **Inputs:** One item from **Loop Over Items**
- **Outputs:** JSON under `output` (as used later: `$json.output.type`, `$json.output.Answer`, `$json.output.name`)
- **Failure modes / edge cases:**
  - If incoming item lacks expected keys (`Name`, `Last`, etc.), prompt loses meaning
  - Model might output `Answer: null` even for answer; downstream cleaning expects a string
  - The strict “ONLY raw JSON” requirement can still be violated under model drift

#### Node: **Check If Message Requires Response**
- **Type / Role:** IF node (branching)
- **Condition:** `{{$json.output.type}} == "answer"`
- **True path:** → **Cleaning Output Text**
- **False path:** → **Loop Over Items** (skip sending; move to next chat)
- **Failure modes:** `output.type` missing or different casing; loose type validation is enabled.

---

### Block 4 — Clean answer text and send via BrowserAct (with verification handling)
**Overview:** Converts escaped newlines into real newlines, triggers BrowserAct workflow to send the message, and handles “paused” verification states by waiting and polling.

**Nodes involved:**
- Cleaning Output Text
- Run Answering Workflow
- Human Verification Switch2
- Give Time to Complete Verification2
- Get Data From BrowserAct2
- Wait for Workflow Completion

#### Node: **Cleaning Output Text**
- **Type / Role:** Code (JavaScript) — formatting
- **Key logic:**
  - `rawAnswer = $input.first().json.output.Answer`
  - `cleanText = rawAnswer.replace(/\\n/g, '\n')`
  - Returns `{ json: { clean_message: cleanText } }`
- **Input:** From **Check If Message Requires Response** (true path)
- **Output:** To **Run Answering Workflow**
- **Failure modes / edge cases:**
  - If `rawAnswer` is `null` → `.replace` throws (should guard if `Answer` can be null)
  - If footer/newlines are already real newlines, replace still harmless

#### Node: **Run Answering Workflow**
- **Type / Role:** BrowserAct node (runs a BrowserAct workflow to reply)
- **Config choices:**
  - Executes BrowserAct **WORKFLOW** with `workflowId: 69845607081621235`
  - Timeout: 7200 seconds
  - Workflow inputs:
    - `input-Name = {{ $('Generate Answers for Messages').first().json.output.name }}`
    - `input-Answer = {{ $json.clean_message }}`
  - Uses BrowserAct API credential: **BrowserAct account**
- **Connections:** To **Human Verification Switch2**
- **Failure modes:**
  - Wrong workflowId/template not configured to accept these inputs
  - Telegram UI changes break automation
  - Wrong name reference: using `first()` could mismatch if multiple items are processed concurrently

#### Node: **Human Verification Switch2**
- **Type / Role:** Switch (routes by BrowserAct task status)
- **Logic:**
  - `status == finished` → continue
  - `status == paused` → wait/poll
- **Outputs:**
  - Finished → **Wait for Workflow Completion**
  - Paused → **Give Time to Complete Verification2**
- **Failure modes:** Unhandled statuses (failed/canceled), missing `status`.

#### Node: **Give Time to Complete Verification2**
- **Type / Role:** Wait (minutes)
- **Output:** To **Get Data From BrowserAct2**

#### Node: **Get Data From BrowserAct2**
- **Type / Role:** HTTP Request (poll BrowserAct task)
- **Config choices:**
  - GET `https://api.browseract.com/v2/workflow/get-task`
  - Query param:
    - `task_id = {{ $('Read Telegram Message').item.json.id }}`
- **Important note (likely issue):**
  - This references the **Read Telegram Message** task id, not the **Run Answering Workflow** task id. For correct polling, it usually should reference the task id returned by **Run Answering Workflow**.
- **Output:** To **Wait for Workflow Completion**
- **Failure modes:** Polling the wrong task leads to incorrect status and broken looping.

#### Node: **Wait for Workflow Completion**
- **Type / Role:** Wait (minutes)
- **Connections:** Outputs back to **Loop Over Items** (continues processing next extracted message item after send workflow completes/waits)
- **Edge cases:** If used as a generic delay instead of true completion tracking, messages might overlap.

---

### Block 5 — Documentation / Notes (non-executing)
**Overview:** Sticky notes provide setup requirements, links, and a video reference.

**Nodes involved:**
- Documentation
- Step 1 Explanation
- Step 2 Explanation
- Step 3 Explanation
- Sticky Note (YouTube)

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every 15 minutes | Schedule Trigger | Periodic trigger | — | Read Telegram Message | ### 📩 Step 1: Message Retrieval / The workflow triggers every 15 minutes to fetch the latest messages from Telegram via BrowserAct. It scrapes the chat history to understand the context of each user interaction. |
| Read Telegram Message | BrowserAct | Run BrowserAct workflow to read Telegram messages | Every 15 minutes | Human Verification Switch | ### 📩 Step 1: Message Retrieval / The workflow triggers every 15 minutes to fetch the latest messages from Telegram via BrowserAct. It scrapes the chat history to understand the context of each user interaction. |
| Human Verification Switch | Switch | Route by BrowserAct task status (finished/paused) | Read Telegram Message | Splitting Extracted Items; Give Time to Complete Verification | ### 📩 Step 1: Message Retrieval / The workflow triggers every 15 minutes to fetch the latest messages from Telegram via BrowserAct. It scrapes the chat history to understand the context of each user interaction. |
| Give Time to Complete Verification | Wait | Delay to allow human verification | Human Verification Switch | Get Data From BrowserAct | ### 📩 Step 1: Message Retrieval / The workflow triggers every 15 minutes to fetch the latest messages from Telegram via BrowserAct. It scrapes the chat history to understand the context of each user interaction. |
| Get Data From BrowserAct | HTTP Request | Poll BrowserAct task result | Give Time to Complete Verification | Splitting Extracted Items | ### 📩 Step 1: Message Retrieval / The workflow triggers every 15 minutes to fetch the latest messages from Telegram via BrowserAct. It scrapes the chat history to understand the context of each user interaction. |
| Splitting Extracted Items | Code | Parse BrowserAct output JSON string into items | Human Verification Switch; Get Data From BrowserAct | Loop Over Items | ### 📩 Step 1: Message Retrieval / The workflow triggers every 15 minutes to fetch the latest messages from Telegram via BrowserAct. It scrapes the chat history to understand the context of each user interaction. |
| Loop Over Items | Split In Batches | Iterate through extracted conversations/messages | Splitting Extracted Items; Check If Message Requires Response (false); Wait for Workflow Completion | Generate Answers for Messages | ### 🧠 Step 2: AI Response Drafting / An AI agent analyzes the retrieved messages to determine if a response is needed. If the user asked a question, it drafts a personalized, helpful reply, ensuring to include a disclaimer that it is an automated bot. |
| Google Gemini | LangChain Chat Model (Google Gemini/PaLM) | LLM backend for agent + parser | — | Generate Answers for Messages; Structured Output | ### 🧠 Step 2: AI Response Drafting / An AI agent analyzes the retrieved messages to determine if a response is needed. If the user asked a question, it drafts a personalized, helpful reply, ensuring to include a disclaimer that it is an automated bot. |
| Structured Output | LangChain Output Parser (Structured) | Enforce/repair strict JSON output | — (LLM wired) | Generate Answers for Messages (as output parser) | ### 🧠 Step 2: AI Response Drafting / An AI agent analyzes the retrieved messages to determine if a response is needed. If the user asked a question, it drafts a personalized, helpful reply, ensuring to include a disclaimer that it is an automated bot. |
| Generate Answers for Messages | LangChain Agent | Draft reply + decide answer vs idle | Loop Over Items | Check If Message Requires Response | ### 🧠 Step 2: AI Response Drafting / An AI agent analyzes the retrieved messages to determine if a response is needed. If the user asked a question, it drafts a personalized, helpful reply, ensuring to include a disclaimer that it is an automated bot. |
| Check If Message Requires Response | IF | Branch on `output.type == answer` | Generate Answers for Messages | Cleaning Output Text (true); Loop Over Items (false) | ### 🧠 Step 2: AI Response Drafting / An AI agent analyzes the retrieved messages to determine if a response is needed. If the user asked a question, it drafts a personalized, helpful reply, ensuring to include a disclaimer that it is an automated bot. |
| Cleaning Output Text | Code | Convert escaped `\\n` to real newlines | Check If Message Requires Response (true) | Run Answering Workflow | ### 🤖 Step 3: Automated Reply / Once a response is drafted and cleaned, BrowserAct is triggered again to type and send the message directly in the Telegram web interface, simulating a human agent's behavior. |
| Run Answering Workflow | BrowserAct | Run BrowserAct workflow to send reply | Cleaning Output Text | Human Verification Switch2 | ### 🤖 Step 3: Automated Reply / Once a response is drafted and cleaned, BrowserAct is triggered again to type and send the message directly in the Telegram web interface, simulating a human agent's behavior. |
| Human Verification Switch2 | Switch | Route by send-task status (finished/paused) | Run Answering Workflow | Wait for Workflow Completion; Give Time to Complete Verification2 | ### 🤖 Step 3: Automated Reply / Once a response is drafted and cleaned, BrowserAct is triggered again to type and send the message directly in the Telegram web interface, simulating a human agent's behavior. |
| Give Time to Complete Verification2 | Wait | Delay to allow human verification | Human Verification Switch2 | Get Data From BrowserAct2 | ### 🤖 Step 3: Automated Reply / Once a response is drafted and cleaned, BrowserAct is triggered again to type and send the message directly in the Telegram web interface, simulating a human agent's behavior. |
| Get Data From BrowserAct2 | HTTP Request | Poll BrowserAct task result (send workflow) | Give Time to Complete Verification2 | Wait for Workflow Completion | ### 🤖 Step 3: Automated Reply / Once a response is drafted and cleaned, BrowserAct is triggered again to type and send the message directly in the Telegram web interface, simulating a human agent's behavior. |
| Wait for Workflow Completion | Wait | Delay/hold before continuing loop | Human Verification Switch2; Get Data From BrowserAct2 | Loop Over Items | ### 🤖 Step 3: Automated Reply / Once a response is drafted and cleaned, BrowserAct is triggered again to type and send the message directly in the Telegram web interface, simulating a human agent's behavior. |
| Documentation | Sticky Note | Setup notes + links | — | — | ## ⚡ Workflow Overview & Setup / Summary + Requirements + links to https://docs.browseract.com |
| Step 1 Explanation | Sticky Note | Explains retrieval step | — | — | ### 📩 Step 1: Message Retrieval / (content as shown) |
| Step 2 Explanation | Sticky Note | Explains AI drafting step | — | — | ### 🧠 Step 2: AI Response Drafting / (content as shown) |
| Step 3 Explanation | Sticky Note | Explains automated reply step | — | — | ### 🤖 Step 3: Automated Reply / (content as shown) |
| Sticky Note | Sticky Note | Video reference | — | — | Video: https://www.youtube.com/watch?v=iTp4LhhjCiQ |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named: *Auto-reply to Telegram messages using BrowserAct & Google Gemini*.

2. **Add Trigger**
   1) Add **Schedule Trigger** node named **Every 15 minutes**.  
   2) Set interval: **Every 15 minutes**.

3. **Add Telegram retrieval via BrowserAct**
   1) Add **BrowserAct** node named **Read Telegram Message**.  
   2) Operation/type: **WORKFLOW**.  
   3) Set **Workflow ID** to `69828791477180147`.  
   4) Credentials: select/create **BrowserAct API** credential (API key from BrowserAct).  
   5) Keep incognito off (or match your needs).  
   6) Connect: **Every 15 minutes → Read Telegram Message**.

4. **Add status routing for verification**
   1) Add **Switch** node named **Human Verification Switch**.  
   2) Add two rules on `{{$json.status}}`:
      - equals `finished`
      - equals `paused`
   3) Connect: **Read Telegram Message → Human Verification Switch**.

5. **Add wait + poll for paused tasks**
   1) Add **Wait** node **Give Time to Complete Verification**, unit **minutes** (choose e.g. 1–2 minutes).  
   2) Add **HTTP Request** node **Get Data From BrowserAct**:
      - Method: GET
      - URL: `https://api.browseract.com/v2/workflow/get-task`
      - Authentication: **Predefined Credential Type**
      - Credential type: **BrowserAct API**
      - Query parameter: `task_id = {{ $('Read Telegram Message').item.json.id }}`
   3) Connect paused path: **Human Verification Switch (paused) → Give Time to Complete Verification → Get Data From BrowserAct**.

6. **Parse BrowserAct output and split to items**
   1) Add **Code** node **Splitting Extracted Items** with logic:
      - Read `$input.first().json.output.string`
      - JSON.parse it
      - Ensure array
      - Return each element as `{ json: element }`
   2) Connect:
      - **Human Verification Switch (finished) → Splitting Extracted Items**
      - **Get Data From BrowserAct → Splitting Extracted Items**

7. **Loop through each extracted chat item**
   1) Add **Split In Batches** node **Loop Over Items** (set batch size as desired, e.g. 1).  
   2) Connect: **Splitting Extracted Items → Loop Over Items**.

8. **Add Gemini model**
   1) Add **Google Gemini Chat Model** node named **Google Gemini**.  
   2) Credentials: create/select **Google Gemini(PaLM) API** credential.

9. **Add structured output parser**
   1) Add **Structured Output Parser** node named **Structured Output**.  
   2) Enable **Auto-fix**.  
   3) Provide a schema example with keys: `name`, `type`, `Answer`.

10. **Add AI agent for response drafting**
   1) Add **LangChain Agent** node **Generate Answers for Messages**.  
   2) Prompt (text) should include fields:
      - `Name`, `Last`, `UserHistory`, `BotHistory`
   3) System message: enforce:
      - output JSON only
      - `type` is `answer` or `idle`
      - must mention Name
      - must append the exact footer for `answer`
   4) Attach the model and parser:
      - Connect **Google Gemini** to the agent as its **language model**
      - Connect **Structured Output** to the agent as its **output parser**
   5) Connect: **Loop Over Items → Generate Answers for Messages**.

11. **Branch on answer vs idle**
   1) Add **IF** node **Check If Message Requires Response** with condition:
      - `{{$json.output.type}}` equals `answer`
   2) Connect: **Generate Answers for Messages → Check If Message Requires Response**.
   3) Connect false branch to continue looping:
      - **Check If... (false) → Loop Over Items**

12. **Clean newline formatting**
   1) Add **Code** node **Cleaning Output Text**:
      - `rawAnswer = $input.first().json.output.Answer`
      - Replace `\\n` with `\n`
      - Output `clean_message`
   2) Connect: **Check If... (true) → Cleaning Output Text**

13. **Send reply via BrowserAct**
   1) Add **BrowserAct** node **Run Answering Workflow**:
      - Type: **WORKFLOW**
      - Workflow ID: `69845607081621235`
      - Timeout: `7200` seconds
      - Inputs:
        - `input-Name = {{ $('Generate Answers for Messages').first().json.output.name }}`
        - `input-Answer = {{ $json.clean_message }}`
      - Credentials: BrowserAct API
   2) Connect: **Cleaning Output Text → Run Answering Workflow**

14. **Handle verification for sending**
   1) Add **Switch** node **Human Verification Switch2** with same rules on `{{$json.status}}` (`finished` / `paused`).  
   2) Add **Wait** node **Give Time to Complete Verification2** (minutes).  
   3) Add **HTTP Request** node **Get Data From BrowserAct2** (same endpoint).  
   4) Add **Wait** node **Wait for Workflow Completion** (minutes) to throttle before continuing.
   5) Connect:
      - **Run Answering Workflow → Human Verification Switch2**
      - paused → **Give Time…2 → Get Data…2 → Wait for Workflow Completion**
      - finished → **Wait for Workflow Completion**
      - **Wait for Workflow Completion → Loop Over Items**

   **Important correction to apply while rebuilding:** In **Get Data From BrowserAct2**, set `task_id` to the **Run Answering Workflow** task id (the task you are polling), not the “Read Telegram Message” id. Typically:
   - `task_id = {{ $('Run Answering Workflow').item.json.id }}`

15. **Add sticky notes (optional, documentation only)**
   - Add sticky notes for “Step 1/2/3” and the BrowserAct docs links and YouTube reference.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| How to Find Your BrowserAct API Key & Workflow ID | https://docs.browseract.com |
| How to Connect n8n to BrowserAct | https://docs.browseract.com |
| How to Use & Customize BrowserAct Templates | https://docs.browseract.com |
| Video reference | https://www.youtube.com/watch?v=iTp4LhhjCiQ |
| Template requirement mentioned in notes: “Telegram Personal Assistant” | Ensure this template exists/saved in your BrowserAct account and is logged into Telegram |
| Potential bug to fix | `Get Data From BrowserAct2` polls `Read Telegram Message` task id; should poll `Run Answering Workflow` task id |
| Robustness improvement suggestion | Guard against `Answer: null` in “Cleaning Output Text” to avoid `.replace` crashes |