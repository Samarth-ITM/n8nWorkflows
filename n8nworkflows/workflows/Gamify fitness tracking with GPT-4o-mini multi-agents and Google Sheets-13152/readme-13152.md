Gamify fitness tracking with GPT-4o-mini multi-agents and Google Sheets

https://n8nworkflows.xyz/workflows/gamify-fitness-tracking-with-gpt-4o-mini-multi-agents-and-google-sheets-13152


# Gamify fitness tracking with GPT-4o-mini multi-agents and Google Sheets

## 1. Workflow Overview

**Purpose:**  
This workflow turns fitness tracking into a “pirate bounty” game. A user sends a chat message describing what they ate, how they trained, or how they feel (injury/fatigue). An AI model classifies the intent and scores the effort (0–100). The workflow converts that score into a bounty increase, updates the user’s total bounty in Google Sheets, routes the conversation to a themed “crew member” agent, and finally logs the exchange to Google Sheets.

**Target use cases:**
- Gamified daily check-ins for nutrition, workouts, and recovery
- Lightweight coaching responses in different “voices” (chef/trainer/doctor/navigator)
- Progress tracking with a persistent score (“Total_Bounty”) and conversation log

### Logical Blocks
**1.1 Input Reception & User State Fetch**  
Chat trigger receives the message, then fetches the user’s current bounty from the `Profile` sheet.

**1.2 AI Appraisal (Intent + Score)**  
AI classifier returns raw JSON with `role` and `score`.

**1.3 Bounty Computation & Persistence**  
Compute `new_bounty` (existing bounty + score × 100000) and update `Profile`.

**1.4 Intent Routing**  
Route to one of four themed agents based on `role`.

**1.5 Agent Response & Logging**  
Generate themed response and append the conversation to the `Log` sheet (per crew).

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & User State Fetch

**Overview:**  
Receives a chat message and looks up the user’s current total bounty in Google Sheets so the new bounty can be computed.

**Nodes involved:**
- When chat message received
- Get row(s) in sheet

#### Node: **When chat message received**
- **Type / Role:** `@n8n/n8n-nodes-langchain.chatTrigger` — entry point for chat-based executions.
- **Configuration (interpreted):**
  - Standard chat trigger with default options.
  - Provides `chatInput` in the output JSON (used widely downstream).
- **Key variables/fields:**
  - `$('When chat message received').first().json.chatInput`
- **Connections:**
  - **Output →** Get row(s) in sheet (main)
- **Potential failures / edge cases:**
  - Empty or extremely long `chatInput` may degrade classification quality or exceed model context limits later.
  - If used outside n8n’s chat interface context, may not fire as expected.

#### Node: **Get row(s) in sheet**
- **Type / Role:** `n8n-nodes-base.googleSheets` — read current user profile row.
- **Configuration (interpreted):**
  - Operation: “Get row(s)” (read).
  - Uses a filter/lookup on the `ID` column (lookupColumn set to `ID`).
  - Spreadsheet/document is placeholder: `YOUR_SPREADSHEET_ID`.
  - The workflow expects a `Total_Bounty` field to exist in the returned row.
- **Key expressions/fields:**
  - Downstream expects: `$node["Get row(s) in sheet"].json.Total_Bounty`
- **Connections:**
  - **Input ←** When chat message received
  - **Output →** AI Scorer & Intent Classifier
- **Potential failures / edge cases:**
  - **Auth errors** (expired/invalid Google OAuth2).
  - Spreadsheet/tab not found (wrong Spreadsheet ID or sheetName).
  - No matching row for the given ID → `Total_Bounty` may be `undefined`, handled downstream with `|| 0`.
  - Filter is configured but the workflow hardcodes user ID later during update (see “Update User Bounty”), which can cause mismatch if you expect per-user tracking.

---

### 2.2 AI Appraisal (Intent + Score)

**Overview:**  
A central AI agent (“Bounty Appraiser”) classifies the user input into a role and assigns a numeric score (0–100) returned as **raw JSON**.

**Nodes involved:**
- OpenRouter Chat Model
- AI Scorer & Intent Classifier

#### Node: **OpenRouter Chat Model**
- **Type / Role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — shared chat model provider for all LLM chain nodes.
- **Configuration (interpreted):**
  - Model: `openai/gpt-4o-mini` via OpenRouter.
  - Connected to multiple nodes through the `ai_languageModel` connection type (centralized model node).
- **Connections:**
  - **AI language model →** AI Scorer & Intent Classifier, Chef Agent, Samurai Agent, Doctor Agent, Navigator Agent
- **Version-specific notes:**
  - Requires n8n LangChain nodes support and the OpenRouter integration node.
- **Potential failures / edge cases:**
  - Missing/invalid OpenRouter credentials or model access.
  - Rate limits/timeouts from OpenRouter.
  - Model may output non-JSON despite instruction; downstream parses JSON and can fail.

#### Node: **AI Scorer & Intent Classifier**
- **Type / Role:** `@n8n/n8n-nodes-langchain.chainLlm` — prompt-driven classifier/scorer.
- **Configuration (interpreted):**
  - Prompt instructs the model to output **only** JSON of the form:
    - `role`: one of `chef | samurai | doctor | other`
    - `score`: 0–100
  - Scoring rubric included (healthy meals, intense workout, rest/recovery).
  - Injects user text: `{{ $('When chat message received').item.json.chatInput }}`
- **Key expressions/fields:**
  - Output expected in `$json.text` (string) containing raw JSON.
- **Connections:**
  - **Input ←** Get row(s) in sheet
  - **AI model ←** OpenRouter Chat Model (ai_languageModel)
  - **Output →** Bounty Calculator (JS Logic)
- **Potential failures / edge cases:**
  - **JSON parsing risk:** if the model returns markdown, backticks, extra text, or malformed JSON.
  - **Role drift:** model might output unexpected role values; routing relies on exact string matches.
  - Score outside range (negative or >100) is not clamped; affects bounty.

---

### 2.3 Bounty Computation & Persistence

**Overview:**  
Parses the classifier JSON, computes the new bounty, stores `ai_role` and `new_bounty` in the item, then updates the user’s `Total_Bounty` in Google Sheets.

**Nodes involved:**
- Bounty Calculator (JS Logic)
- Update User Bounty

#### Node: **Bounty Calculator (JS Logic)**
- **Type / Role:** `n8n-nodes-base.set` — used as a calculation/transform step (despite name referencing JS).
- **Configuration (interpreted):**
  - Creates two fields:
    - `ai_role` from parsed classifier JSON
    - `new_bounty` = current `Total_Bounty` + (`score` × 100000)
  - It strips potential ```json fences before parsing (basic sanitation).
- **Key expressions/variables:**
  - `ai_role`:
    - `={{ JSON.parse($node["AI Scorer & Intent Classifier"].json.text.replace(/```json|```/g, "").trim()).role }}`
  - `new_bounty`:
    - `={{ Number($node["Get row(s) in sheet"].json.Total_Bounty || 0) + (Number(JSON.parse($json.text.replace(/```json|```/g, "").trim()).score || 0) * 100000) }}`
- **Connections:**
  - **Input ←** AI Scorer & Intent Classifier
  - **Output →** Update User Bounty
- **Potential failures / edge cases:**
  - If `$json.text` isn’t valid JSON even after fence removal → expression throws and the node fails.
  - If `Total_Bounty` includes commas/currency or is non-numeric text → `Number(...)` becomes `NaN` and `new_bounty` becomes `NaN`.
  - Bounty scaling factor is fixed at `100000`; large scores can quickly create huge numbers (possible sheet formatting/limits issues).

#### Node: **Update User Bounty**
- **Type / Role:** `n8n-nodes-base.googleSheets` — updates the profile row with the newly computed bounty.
- **Configuration (interpreted):**
  - Operation: `update`
  - Matching column: `ID`
  - Values written:
    - `ID`: hardcoded to `"1"`
    - `Total_Bounty`: `={{ $json.new_bounty }}`
  - Spreadsheet/document placeholders:
    - DocumentId: `YOUR_SPREADSHEET_ID`
    - SheetName: `YOUR_SHEET_NAME_OR_ID` (should be the `Profile` tab)
  - `alwaysOutputData: true` ensures downstream routing continues even if the update returns minimal data.
- **Connections:**
  - **Input ←** Bounty Calculator (JS Logic)
  - **Output →** Switch
- **Credentials:**
  - Google Sheets OAuth2 required.
- **Potential failures / edge cases:**
  - **Hardcoded ID risk:** always updates ID=1 regardless of who chatted. If you want per-user tracking, you must map an actual user identifier from the chat trigger.
  - No matching row for ID=1 → update may fail or update nothing depending on node behavior.
  - Auth / permissions errors on the sheet.

---

### 2.4 Intent Routing

**Overview:**  
Routes execution to one of four themed agents based on `ai_role` produced by the classifier.

**Nodes involved:**
- Switch

#### Node: **Switch**
- **Type / Role:** `n8n-nodes-base.switch` — conditional branching by role.
- **Configuration (interpreted):**
  - 4 rules (string equals):
    1. `chef`
    2. `samurai`
    3. `doctor`
    4. `other`
  - Each rule compares:
    - `={{ $('Bounty Calculator (JS Logic)').item.json.ai_role }}`
- **Connections:**
  - **Input ←** Update User Bounty
  - **Outputs →**
    - Output 0 → Chef Agent
    - Output 1 → Samurai Agent
    - Output 2 → Doctor Agent
    - Output 3 → Navigator Agent
- **Potential failures / edge cases:**
  - If `ai_role` is missing/undefined or not one of the four expected values, none of the rules match → no agent runs (silent “dead end”).
  - Case sensitivity is enabled; `Chef` would not match `chef`.

---

### 2.5 Agent Response & Logging

**Overview:**  
A specialized persona agent generates a themed response (recipe / exercise / recovery advice / off-topic scolding), then the workflow appends the exchange to the `Log` sheet with metadata.

**Nodes involved:**
- Chef Agent → Save Chef Log
- Samurai Agent → Save Samurai Log
- Doctor Agent → Save Doctor Log
- Navigator Agent → Save Navigator Log

#### Node: **Chef Agent**
- **Type / Role:** `@n8n/n8n-nodes-langchain.chainLlm` — generates nutrition/recipe response.
- **Configuration (interpreted):**
  - Persona: pirate ship cook, nutrition-focused, pirate slang, non-offensive.
  - Uses:
    - User input: `{{ $('When chat message received').first().json.chatInput }}`
    - New bounty: `{{ $('Bounty Calculator (JS Logic)').first().json.new_bounty }}`
  - Output is in `$json.text`.
- **Connections:**
  - **Input ←** Switch (chef path)
  - **AI model ←** OpenRouter Chat Model
  - **Output →** Save Chef Log
- **Potential failures / edge cases:**
  - If `new_bounty` is `NaN`, the response will contain `NaN Berries`.
  - Prompt includes gendered language (“ladies”/“lads”); may not fit all audiences.

#### Node: **Save Chef Log**
- **Type / Role:** `n8n-nodes-base.googleSheets` — append log row.
- **Configuration (interpreted):**
  - Operation: `append`
  - Writes to `Log` tab (intended) with fields:
    - `Date`: `={{ $now }}`
    - `Crew`: `"Chef"`
    - `Inquiry`: chatInput
    - `Response`: agent output text (`={{ $json.text }}`)
- **Connections:**
  - **Input ←** Chef Agent
- **Potential failures / edge cases:**
  - Sheet name/ID placeholder must point to `Log`.
  - If the `Log` sheet columns differ, append may mis-map.

#### Node: **Samurai Agent**
- **Type / Role:** `@n8n/n8n-nodes-langchain.chainLlm` — generates training command response.
- **Configuration:** Stoic swordsman persona; commands bodyweight exercise; mentions bounty; ends with “lost directions” line.
- **Key fields:** user input + `new_bounty`.
- **Connections:** Switch → Samurai Agent → Save Samurai Log; AI model from OpenRouter.
- **Potential failures:** same as Chef Agent (NaN bounty, model issues).

#### Node: **Save Samurai Log**
- **Type / Role:** Google Sheets append to `Log`.
- **Fields:** `Crew="Samurai"`, `Date=$now`, `Inquiry=chatInput`, `Response=$json.text`.
- **Connections:** Input from Samurai Agent.

#### Node: **Doctor Agent**
- **Type / Role:** `chainLlm` — recovery/rest advice persona.
- **Configuration:** Reindeer doctor persona; explains rest; mentions bounty as “get-well gift.”
- **Connections:** Switch → Doctor Agent → Save Doctor Log; AI model from OpenRouter.

#### Node: **Save Doctor Log**
- **Type / Role:** Google Sheets append to `Log`.
- **Fields:** `Crew="Doctor"` etc.
- **Connections:** Input from Doctor Agent.

#### Node: **Navigator Agent**
- **Type / Role:** `chainLlm` — “other/off-topic” handling.
- **Configuration:** Navigator persona; scolds off-topic chat; pushes user back to training/eating; mentions bounty.
- **Connections:** Switch → Navigator Agent → Save Navigator Log; AI model from OpenRouter.

#### Node: **Save Navigator Log**
- **Type / Role:** Google Sheets append to `Log`.
- **Fields:** `Crew="Navigator"` etc.
- **Connections:** Input from Navigator Agent.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Chat entry point | — | Get row(s) in sheet | Step 1: Input & Data |
| Get row(s) in sheet | n8n-nodes-base.googleSheets | Read user bounty from Profile sheet | When chat message received | AI Scorer & Intent Classifier | Step 1: Input & Data |
| AI Scorer & Intent Classifier | @n8n/n8n-nodes-langchain.chainLlm | Classify intent + score (JSON output) | Get row(s) in sheet | Bounty Calculator (JS Logic) | Step 2: AI Appraisal |
| Bounty Calculator (JS Logic) | n8n-nodes-base.set | Parse JSON, compute new bounty, set ai_role | AI Scorer & Intent Classifier | Update User Bounty | Step 2: AI Appraisal |
| Update User Bounty | n8n-nodes-base.googleSheets | Update Profile.Total_Bounty | Bounty Calculator (JS Logic) | Switch | Step 3: Intent Routing |
| Switch | n8n-nodes-base.switch | Route by ai_role | Update User Bounty | Chef Agent; Samurai Agent; Doctor Agent; Navigator Agent | Step 3: Intent Routing |
| Chef Agent | @n8n/n8n-nodes-langchain.chainLlm | Food/nutrition themed response | Switch | Save Chef Log | Step 4: Agent Responses |
| Save Chef Log | n8n-nodes-base.googleSheets | Append log row (Chef) | Chef Agent | — | Step 4: Agent Responses |
| Samurai Agent | @n8n/n8n-nodes-langchain.chainLlm | Training themed response | Switch | Save Samurai Log | Step 4: Agent Responses |
| Save Samurai Log | n8n-nodes-base.googleSheets | Append log row (Samurai) | Samurai Agent | — | Step 4: Agent Responses |
| Doctor Agent | @n8n/n8n-nodes-langchain.chainLlm | Recovery themed response | Switch | Save Doctor Log | Step 4: Agent Responses |
| Save Doctor Log | n8n-nodes-base.googleSheets | Append log row (Doctor) | Doctor Agent | — | Step 4: Agent Responses |
| Navigator Agent | @n8n/n8n-nodes-langchain.chainLlm | Off-topic handling themed response | Switch | Save Navigator Log | Step 4: Agent Responses |
| Save Navigator Log | n8n-nodes-base.googleSheets | Append log row (Navigator) | Navigator Agent | — | Step 4: Agent Responses |
| OpenRouter Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Shared LLM provider | — (model provider) | AI Scorer & Intent Classifier; Chef Agent; Samurai Agent; Doctor Agent; Navigator Agent (ai_languageModel) |  |
| Sticky Note | n8n-nodes-base.stickyNote | Comment | — | — | Step 1: Input & Data |
| Sticky Note7 | n8n-nodes-base.stickyNote | Comment | — | — | Step 2: AI Appraisal |
| Sticky Note1 | n8n-nodes-base.stickyNote | Comment | — | — | Step 3: Intent Routing |
| Sticky Note2 | n8n-nodes-base.stickyNote | Comment | — | — | Step 4: Agent Responses |
| Sticky Note6 | n8n-nodes-base.stickyNote | Comment / Overview & setup instructions | — | — | Title:\nGamify fitness tracking with AI multi-agents and Google Sheets\n## Overview\nThis template transforms fitness tracking into a gamified pirate adventure using AI multi-agents. It scores your health activities and assigns a dynamic "Bounty" reward.\n\n## How it works\n1. **Fetch Data**: Retrieves your current "Bounty" from Google Sheets.\n2. **AI Analysis**: A central AI Scorer evaluates your report (meals, workouts, or injuries) and assigns a score from 0-100.\n3. **Logic**: JavaScript calculates the bounty increase based on the AI's score.\n4. **Routing**: The Switch node routes you to a specialized agent (Chef, Samurai, Doctor, or Navigator).\n5. **Logging**: Saves the conversation to Google Sheets for progress tracking.\n\n## Setup steps\n1. **Google Sheets**: Create a sheet with two tabs:\n   - `Profile` (Columns: ID, Total_Bounty)\n   - `Log` (Columns: Date, Crew, Inquiry, Response)\n2. **Spreadsheet ID**: Replace `YOUR_SPREADSHEET_ID` in all 6 Google Sheets nodes.\n3. **Credentials**: Connect your Google Sheets and AI (OpenRouter/OpenAI) accounts. |

---

## 4. Reproducing the Workflow from Scratch

1) **Create a new workflow**  
   - Name it: *Gamify fitness tracking with AI multi-agents and Google Sheets* (or your preferred title).

2) **Add Chat Trigger**
   - Add node: **Chat Trigger** (`When chat message received`)
   - Keep default options.
   - This will produce `chatInput` in the output.

3) **Prepare Google Sheets**
   - In Google Sheets, create one spreadsheet with two tabs:
     - **Profile** with columns: `ID`, `Total_Bounty`
     - **Log** with columns: `Date`, `Crew`, `Inquiry`, `Response`
   - Add at least one row in `Profile` (e.g., `ID = 1`, `Total_Bounty = 0`).

4) **Add Google Sheets “Get row(s)” (Profile lookup)**
   - Add node: **Google Sheets** → operation **Get row(s)**
   - Select your Google Sheets OAuth2 credentials.
   - Document/Spreadsheet: set your Spreadsheet ID.
   - Sheet: select `Profile`.
   - Add a filter/lookup:
     - Lookup column: `ID`
     - (Adjust as desired; the provided workflow assumes an ID-based lookup.)
   - Connect: **Chat Trigger → Get row(s)**

5) **Add OpenRouter model node**
   - Add node: **OpenRouter Chat Model**
   - Choose model: `openai/gpt-4o-mini`
   - Configure OpenRouter credentials (API key) in n8n.
   - This node will be connected as the **AI Language Model** to the LLM chain nodes.

6) **Add the AI classifier/scorer**
   - Add node: **Chain LLM** named **AI Scorer & Intent Classifier**
   - Prompt: instruct it to return ONLY raw JSON with:
     - `role`: `chef | samurai | doctor | other`
     - `score`: 0–100
     - Include your rubric.
     - Insert the user report with an expression from chatInput.
   - Connect:
     - **Get row(s) → AI Scorer & Intent Classifier** (main)
     - **OpenRouter Chat Model → AI Scorer & Intent Classifier** (AI Language Model connection)

7) **Add bounty calculation (Set node)**
   - Add node: **Set** named **Bounty Calculator (JS Logic)**
   - Add fields:
     - `ai_role` (string): parse JSON from classifier output text.
     - `new_bounty` (number): existing `Total_Bounty` + (`score` × `100000`)
   - Ensure you defensively handle missing values (`|| 0`) and strip potential ``` fences.
   - Connect: **AI Scorer & Intent Classifier → Bounty Calculator (JS Logic)**

8) **Add Google Sheets “Update” (Profile update)**
   - Add node: **Google Sheets** named **Update User Bounty**
   - Operation: **Update**
   - Document: your Spreadsheet ID
   - Sheet: `Profile`
   - Matching column: `ID`
   - Set values:
     - `ID`: (as in the provided workflow) `"1"`  
       - If you want per-user support, replace this with a real user identifier from the trigger.
     - `Total_Bounty`: expression from `new_bounty`
   - Enable “Always Output Data” (so downstream always has an item to route).
   - Connect: **Bounty Calculator → Update User Bounty**

9) **Add routing Switch**
   - Add node: **Switch**
   - Create 4 rules (string equals) on the value `ai_role`:
     - equals `chef`
     - equals `samurai`
     - equals `doctor`
     - equals `other`
   - Connect: **Update User Bounty → Switch**

10) **Add four persona agent nodes (Chain LLM)**
   - Add **Chef Agent** (Chain LLM) with a recipe/nutrition pirate persona prompt.
   - Add **Samurai Agent** with training/exercise persona prompt.
   - Add **Doctor Agent** with recovery persona prompt.
   - Add **Navigator Agent** for off-topic scolding/redirect prompt.
   - Each prompt should reference:
     - `chatInput` from trigger
     - `new_bounty` from bounty calculator
   - Connect **OpenRouter Chat Model** to each agent via **AI Language Model** connection.
   - Connect **Switch outputs**:
     - chef path → Chef Agent
     - samurai path → Samurai Agent
     - doctor path → Doctor Agent
     - other path → Navigator Agent

11) **Add four Google Sheets append nodes (Log tab)**
   - Add **Save Chef Log** (Google Sheets append) to `Log`
     - `Date = $now`
     - `Crew = "Chef"`
     - `Inquiry = chatInput`
     - `Response = agent $json.text`
   - Repeat similarly for Samurai/Doctor/Navigator, changing only the `Crew` label.
   - Connect each agent → its corresponding Save Log node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Title:\nGamify fitness tracking with AI multi-agents and Google Sheets\n## Overview\nThis template transforms fitness tracking into a gamified pirate adventure using AI multi-agents. It scores your health activities and assigns a dynamic "Bounty" reward.\n\n## How it works\n1. **Fetch Data**: Retrieves your current "Bounty" from Google Sheets.\n2. **AI Analysis**: A central AI Scorer evaluates your report (meals, workouts, or injuries) and assigns a score from 0-100.\n3. **Logic**: JavaScript calculates the bounty increase based on the AI's score.\n4. **Routing**: The Switch node routes you to a specialized agent (Chef, Samurai, Doctor, or Navigator).\n5. **Logging**: Saves the conversation to Google Sheets for progress tracking.\n\n## Setup steps\n1. **Google Sheets**: Create a sheet with two tabs:\n   - `Profile` (Columns: ID, Total_Bounty)\n   - `Log` (Columns: Date, Crew, Inquiry, Response)\n2. **Spreadsheet ID**: Replace `YOUR_SPREADSHEET_ID` in all 6 Google Sheets nodes.\n3. **Credentials**: Connect your Google Sheets and AI (OpenRouter/OpenAI) accounts. | From workflow sticky note (template overview and setup) |
| disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques. | Provided by user / workflow context |