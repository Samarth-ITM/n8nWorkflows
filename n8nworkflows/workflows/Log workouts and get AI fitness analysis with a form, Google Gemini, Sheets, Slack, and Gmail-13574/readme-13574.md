Log workouts and get AI fitness analysis with a form, Google Gemini, Sheets, Slack, and Gmail

https://n8nworkflows.xyz/workflows/log-workouts-and-get-ai-fitness-analysis-with-a-form--google-gemini--sheets--slack--and-gmail-13574


# Log workouts and get AI fitness analysis with a form, Google Gemini, Sheets, Slack, and Gmail

## 1. Workflow Overview

**Purpose:** Collect a workout entry via an n8n form, use **Google Gemini** to turn free-text workout notes into structured data (exercises, muscles, calories, intensity, tips), **store the result in Google Sheets**, then send **Slack** and **Gmail** notifications with a formatted summary.

**Target use cases:**
- Personal workout tracking with lightweight data entry (mobile-friendly).
- Automated training insights (muscle groups hit, estimated calories, next session suggestions).
- Team/accountability logging via Slack plus email archive.

### 1.1 Input Reception (Form)
Captures workout data (description, feeling, duration, notes) from the user.

### 1.2 AI Processing (Gemini via LangChain node)
Sends the form data to Gemini with a strict “JSON-only” instruction to produce structured analysis.

### 1.3 Validation & Normalization
Parses the AI response, extracts/validates JSON, fills defaults on parse failures, and produces a normalized single record.

### 1.4 Persistence (Google Sheets)
Appends/updates a row in a “Workouts” tab with the normalized fields.

### 1.5 Notifications (Slack + Gmail)
Builds a human-friendly Slack message and HTML email and sends them out.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception

**Overview:** Presents a public form endpoint and collects structured fields for a workout log entry.  
**Nodes involved:** `Workout Log Form`

#### Node: Workout Log Form
- **Type / role:** `Form Trigger` (`n8n-nodes-base.formTrigger`) — workflow entry point via hosted n8n form.
- **Key configuration:**
  - **Form title:** “Workout Logger”
  - **Description:** “Log your workout and get AI-powered analysis…”
  - **Fields:**
    1. **Textarea (required):** “What did you do today?”
    2. **Dropdown (required):** “How do you feel? (1-10)” (options include “1 - Exhausted” … “10 - Amazing”)
    3. **Number (required):** “Duration (minutes)”
    4. **Textarea (optional):** “Notes”
  - **Path/webhook:** Uses a generated path (UUID-like). This is the URL you bookmark/share.
- **Inputs:** None (trigger).
- **Outputs / connections:** Main output → `Analyze Workout with AI`
- **Edge cases / failures:**
  - Missing required fields prevented at the form level.
  - “Feeling” is a label string (“5 - Normal”), not a pure number—handled later in parsing.
- **Version notes:** Uses Form Trigger **typeVersion 2** (ensure your n8n supports hosted forms).

---

### Block 2 — AI Processing (Gemini)

**Overview:** Sends the workout text to Gemini with instructions to return a strict JSON object containing exercises, summary, tips, and a next workout suggestion.  
**Nodes involved:** `Analyze Workout with AI`, `Google Gemini`

#### Node: Analyze Workout with AI
- **Type / role:** LangChain chain LLM node (`@n8n/n8n-nodes-langchain.chainLlm`) — builds a prompt and calls a connected chat model.
- **Key configuration choices:**
  - **Prompt mode:** “define” (prompt authored directly in node).
  - **Prompt content:** Instructs model to act as “certified personal trainer and sports scientist” and **return ONLY valid JSON** in an exact schema:
    - `exercises[]` with fields (name, type, muscle_groups, sets, reps, weight_kg, duration_min, estimated_calories)
    - `summary` (total_exercises, total_calories, primary_type, muscle_groups_hit, intensity)
    - `tips[]`
    - `next_workout_suggestion`
  - Uses expressions from the form:
    - `{{ $json['What did you do today?'] }}`
    - `{{ $json['Duration (minutes)'] }}`
    - `{{ $json['How do you feel? (1-10)'] }}`
    - `{{ $json['Notes'] || 'None' }}`
- **Inputs:** Output of `Workout Log Form`
- **Outputs / connections:** Main output → `Parse AI Results`
- **Model binding:** Receives its language model from `Google Gemini` via `ai_languageModel` connection (not via main data path).
- **Edge cases / failures:**
  - Model may still wrap JSON in markdown fences or add text; downstream parser attempts to recover.
  - If Gemini returns invalid JSON, downstream code falls back to defaults.
- **Version notes:** Node shows **typeVersion 1.4**; requires n8n’s LangChain integration nodes.

#### Node: Google Gemini
- **Type / role:** LangChain Chat Model provider (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`) — supplies the Gemini model to the chain node.
- **Key configuration choices:**
  - **Model:** `models/gemini-1.5-flash-latest`
  - **Temperature:** `0.2` (bias toward consistent structure/JSON validity)
- **Inputs / connections:**
  - Connected to `Analyze Workout with AI` via **ai_languageModel** (provider relationship).
- **Credentials required:** Google Gemini / Google AI Studio API key (node credential).
- **Edge cases / failures:**
  - Invalid/expired API key, quota exhaustion, model name changes, region restrictions.
  - Latency/timeouts on large prompts; keep entries reasonably sized.
- **Version notes:** **typeVersion 1**.

---

### Block 3 — Validation & Normalization

**Overview:** Extracts JSON from the AI response, validates it, backfills defaults if parsing fails, and merges original form values into a clean record for storage/notifications.  
**Nodes involved:** `Parse AI Results`

#### Node: Parse AI Results
- **Type / role:** `Code` (`n8n-nodes-base.code`) — transforms AI output into normalized fields.
- **Key logic and configuration (interpreted):**
  - Reads AI text from: `const aiResponse = $json.response || $json.text || ''`
  - Attempts to extract JSON fenced by triple backticks:
    - Regex: `/```(?:json)?\s*([\s\S]*?)```/`
  - `JSON.parse()`; if parsing fails, uses a **safe fallback object**:
    - empty exercises
    - summary defaults (type unknown, intensity moderate, etc.)
    - a generic tip advising more specificity
    - next suggestion “Rest day or light cardio”
  - Pulls original form data via:
    - `const form = $('Workout Log Form').first().json;`
  - Parses feeling from label string (e.g., “5 - Normal”) using:
    - `parseInt(feelingRaw.match(/\d+/)?.[0] || '5')`
  - Outputs a single normalized object:
    - `date` (YYYY-MM-DD)
    - `workout_description`, `duration_min`, `feeling`, `notes`
    - `exercises` (array), plus `exercise_count`
    - `total_calories`, `primary_type`, `muscle_groups` (comma-joined), `intensity`
    - `tips` joined with `" | "`
    - `next_suggestion`
    - `logged_at` (ISO timestamp)
- **Inputs:** Output of `Analyze Workout with AI`
- **Outputs / connections:** Main output → `Save to Workout Sheet`
- **Edge cases / failures:**
  - If the AI node returns neither `response` nor `text`, parsing falls back to defaults.
  - If `Workout Log Form` is not in the same execution path (it is here), `$('Workout Log Form')` would break; keep this node in the same workflow and ensure execution includes the trigger.
  - `Duration (minutes)` may be non-numeric; guarded with `parseInt(...) || 0`.
- **Version notes:** Code node **typeVersion 2**.

---

### Block 4 — Persistence (Google Sheets)

**Overview:** Writes the normalized workout record into a Google Sheet tab called “Workouts” using defined column mappings.  
**Nodes involved:** `Save to Workout Sheet`

#### Node: Save to Workout Sheet
- **Type / role:** `Google Sheets` (`n8n-nodes-base.googleSheets`) — append or update row.
- **Key configuration choices:**
  - **Operation:** `appendOrUpdate`
  - **Document ID:** `YOUR_GOOGLE_SHEET_ID` (must be replaced)
  - **Sheet name:** `Workouts`
  - **Mapping mode:** “defineBelow” (explicit column-to-expression mapping)
  - **Cell format:** `USER_ENTERED` (Sheets will interpret numbers/dates where applicable)
  - **Columns mapped:**
    - Date, Workout, Duration, Feeling, Type, Exercises, Calories, Muscles, Intensity, Tips, Next Suggestion, Notes, Logged At
- **Inputs:** Output of `Parse AI Results`
- **Outputs / connections:** Main output → `Build Summary Message`
- **Credentials required:** Google Sheets OAuth2 / Service Account (depending on your n8n setup).
- **Edge cases / failures:**
  - Missing sheet/tab “Workouts” or header mismatch (append/update behavior depends on node settings and sheet structure).
  - Permissions: spreadsheet not shared with the credential’s account.
  - `appendOrUpdate` can behave unexpectedly without a stable key column configured (if not configured, it may default to append behavior or attempt matching). Verify how your node is configured in UI for “matching columns”.
- **Version notes:** **typeVersion 4.4**.

---

### Block 5 — Notifications (Slack + Gmail)

**Overview:** Creates a Slack-friendly message and an HTML email, then sends both notifications.  
**Nodes involved:** `Build Summary Message`, `Notify on Slack`, `Email Summary`

#### Node: Build Summary Message
- **Type / role:** `Code` — builds message strings for downstream notification nodes.
- **Key logic and configuration (interpreted):**
  - Uses `feeling` to choose an emoji (🔥 / 💪 / 😤).
  - Uses `primary_type` to choose an emoji (🏋️, 🏃, 🧘, etc.).
  - Uses `intensity` to choose a colored indicator (🟢/🟡/🟠/🔴).
  - Produces:
    - `slack_message`: multi-line markdown-like Slack text with workout + tips + next suggestion.
    - `email_html`: formatted HTML summary; converts tips from `" | "` into `<br>`.
  - Returns merged object `{ ...w, slack_message, email_html }` so subsequent nodes still have all fields.
- **Inputs:** Output of `Save to Workout Sheet`
- **Outputs / connections:** Main output → `Notify on Slack` and → `Email Summary` (parallel fan-out)
- **Edge cases / failures:**
  - If `tips` is empty, output still formats but may look sparse.
  - HTML is not escaped; if a user enters special characters/HTML in notes/description, it could affect email rendering (consider escaping if needed).
- **Version notes:** Code node **typeVersion 2**.

#### Node: Notify on Slack
- **Type / role:** `Slack` (`n8n-nodes-base.slack`) — sends a channel message.
- **Key configuration choices:**
  - **Select:** channel
  - **Channel ID:** `YOUR_SLACK_CHANNEL_ID` (must be replaced)
  - **Text:** `{{ $json.slack_message }}`
- **Inputs:** From `Build Summary Message`
- **Outputs:** None (end branch)
- **Credentials required:** Slack OAuth token with permission to post messages to the target channel.
- **Edge cases / failures:**
  - Wrong channel ID, bot not invited to channel, missing `chat:write` scope.
  - Slack rate limits (unlikely at low volume).
- **Version notes:** **typeVersion 2.2**.

#### Node: Email Summary
- **Type / role:** `Gmail` (`n8n-nodes-base.gmail`) — sends an email.
- **Key configuration choices:**
  - **To:** `YOUR_EMAIL_ADDRESS` (must be replaced)
  - **Subject:** `Workout logged: {{ $json.primary_type }} {{ $json.duration_min }}min — {{ $json.total_calories }} kcal ({{ $json.date }})`
  - **Message (HTML):** `{{ $json.email_html }}`
- **Inputs:** From `Build Summary Message`
- **Outputs:** None (end branch)
- **Credentials required:** Gmail OAuth2 (n8n Gmail credentials).
- **Edge cases / failures:**
  - OAuth token revoked/expired, insufficient Gmail scopes, sending limits.
  - If your n8n Gmail node expects plain text vs HTML depending on configuration, confirm “message” field behavior in your version.
- **Version notes:** **typeVersion 2.1**.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workout Log Form | formTrigger | Entry form to capture workout details | — | Analyze Workout with AI | ## Workout tracker with AI analysis… Setup steps… Tip… / ## Log & analyze… |
| Analyze Workout with AI | chainLlm (LangChain) | Prompt + structured AI analysis request | Workout Log Form | Parse AI Results | ## Log & analyze… |
| Google Gemini | lmChatGoogleGemini (LangChain) | Provides Gemini chat model to chain | — (model provider link) | Analyze Workout with AI (ai_languageModel) | ## Log & analyze… |
| Parse AI Results | code | Extract/validate JSON, normalize fields | Analyze Workout with AI | Save to Workout Sheet | ## Validate & save… |
| Save to Workout Sheet | googleSheets | Append/update workout row in Sheets | Parse AI Results | Build Summary Message | ## Validate & save… |
| Build Summary Message | code | Create Slack text + HTML email | Save to Workout Sheet | Notify on Slack; Email Summary | ## Notifications… |
| Notify on Slack | slack | Post workout summary to Slack channel | Build Summary Message | — | ## Notifications… |
| Email Summary | gmail | Send email recap | Build Summary Message | — | ## Notifications… |
| Overview | stickyNote | Documentation block | — | — | ## Workout tracker with AI analysis… (this note itself) |
| Input and AI Section | stickyNote | Documentation block | — | — | ## Log & analyze… (this note itself) |
| Process and Save Section | stickyNote | Documentation block | — | — | ## Validate & save… (this note itself) |
| Notify Section | stickyNote | Documentation block | — | — | ## Notifications… (this note itself) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it:  
   “Log workouts and get AI fitness analysis with a form, Google Gemini, Sheets, Slack, and Gmail”.

2. **Add node: Form Trigger**
   - Node type: **Form Trigger**
   - Form Title: `Workout Logger`
   - Form Description: `Log your workout and get AI-powered analysis on muscle groups, calories, and progress tips.`
   - Add fields:
     1. Textarea (required)  
        Label: `What did you do today?`  
        Placeholder: `e.g. Bench press 60kg x 10 x 3, Running 5km 28min, Squats bodyweight 20 reps x 4`
     2. Dropdown (required)  
        Label: `How do you feel? (1-10)`  
        Options: `1 - Exhausted, 2, 3, 4, 5 - Normal, 6, 7, 8, 9, 10 - Amazing`
     3. Number (required)  
        Label: `Duration (minutes)`
     4. Textarea (optional)  
        Label: `Notes`  
        Placeholder: `Any injuries, PRs, or things to remember`
   - Keep/generate a unique **path** (this becomes the form URL).

3. **Add node: Google Gemini (LangChain Chat Model)**
   - Node type: **Google Gemini Chat Model** (LangChain)
   - Credentials: configure **Google AI Studio / Gemini API key** in n8n credentials.
   - Model: `models/gemini-1.5-flash-latest`
   - Temperature: `0.2`

4. **Add node: Analyze Workout with AI (LangChain Chain LLM)**
   - Node type: **Chain LLM**
   - Prompt type: **Define**
   - Paste a prompt that:
     - Includes the form fields via expressions
     - Requires **ONLY valid JSON** with the schema:
       `exercises[]`, `summary{...}`, `tips[]`, `next_workout_suggestion`
   - Connect **Form Trigger → Analyze Workout with AI** (main connection).
   - Connect **Google Gemini → Analyze Workout with AI** using the **AI Language Model** connection type.

5. **Add node: Parse AI Results (Code)**
   - Node type: **Code**
   - Paste code that:
     - Reads AI response from `$json.response || $json.text`
     - Extracts JSON from code fences if present
     - `JSON.parse()` with fallback defaults if parsing fails
     - Pulls original form data via `$('Workout Log Form').first().json`
     - Normalizes fields (date, duration_min, feeling numeric, muscle_groups joined, tips joined, timestamps)
   - Connect **Analyze Workout with AI → Parse AI Results**.

6. **Prepare Google Sheets**
   - Create a spreadsheet with a tab named **Workouts**
   - Add headers (columns) exactly (recommended):  
     `Date, Workout, Duration, Feeling, Type, Exercises, Calories, Muscles, Intensity, Tips, Next Suggestion, Notes, Logged At`

7. **Add node: Save to Workout Sheet (Google Sheets)**
   - Node type: **Google Sheets**
   - Credentials: Google Sheets OAuth2 (or your preferred method).
   - Operation: **Append or Update**
   - Document ID: set to your spreadsheet ID
   - Sheet name: `Workouts`
   - Column mapping (define below):
     - Map each column to the normalized fields, e.g.:
       - `Date` → `{{$json.date}}`
       - `Workout` → `{{$json.workout_description}}`
       - `Duration` → `{{$json.duration_min}}`
       - `Feeling` → `{{$json.feeling}}`
       - `Type` → `{{$json.primary_type}}`
       - `Exercises` → `{{$json.exercise_count}}`
       - `Calories` → `{{$json.total_calories}}`
       - `Muscles` → `{{$json.muscle_groups}}`
       - `Intensity` → `{{$json.intensity}}`
       - `Tips` → `{{$json.tips}}`
       - `Next Suggestion` → `{{$json.next_suggestion}}`
       - `Notes` → `{{$json.notes}}`
       - `Logged At` → `{{$json.logged_at}}`
   - Connect **Parse AI Results → Save to Workout Sheet**.

8. **Add node: Build Summary Message (Code)**
   - Node type: **Code**
   - Create:
     - `slack_message` string (multi-line) using key fields and emojis based on feeling/type/intensity
     - `email_html` string (simple HTML summary)
   - Connect **Save to Workout Sheet → Build Summary Message**.

9. **Add node: Notify on Slack**
   - Node type: **Slack**
   - Credentials: Slack OAuth with `chat:write`
   - Send to: **Channel**
   - Channel ID: choose your target channel (ensure bot is invited)
   - Text: `{{$json.slack_message}}`
   - Connect **Build Summary Message → Notify on Slack**.

10. **Add node: Email Summary (Gmail)**
   - Node type: **Gmail**
   - Credentials: Gmail OAuth2
   - To: your email address
   - Subject: `Workout logged: {{$json.primary_type}} {{$json.duration_min}}min — {{$json.total_calories}} kcal ({{$json.date}})`
   - Message body: `{{$json.email_html}}`
   - Connect **Build Summary Message → Email Summary**.

11. **Test end-to-end**
   - Open the form URL, submit a sample workout entry.
   - Confirm:
     - AI returns parseable JSON (or fallback is used)
     - A new row appears in Sheets
     - Slack message posts
     - Email arrives with HTML formatting

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| “Workout tracker with AI analysis… Setup steps… Tip: Works great on your phone between sets…” | From sticky note **Overview** (in-workflow documentation). |
| Google Sheets setup requirement: create “Workouts” tab with specified headers | Mentioned in sticky note **Overview**. |
| Gemini API key requirement via Google AI Studio | Mentioned in sticky note **Overview**. |
| Slack channel selection and Gmail connection required | Mentioned in sticky note **Overview** and **Notifications** sticky note. |

Disclaimer: Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.