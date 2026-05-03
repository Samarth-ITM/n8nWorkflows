Send scheduled IELTS grammar, writing and reading practice via Telegram with Gemini AI

https://n8nworkflows.xyz/workflows/send-scheduled-ielts-grammar--writing-and-reading-practice-via-telegram-with-gemini-ai-14673


# Send scheduled IELTS grammar, writing and reading practice via Telegram with Gemini AI

### 1. Workflow Overview

This workflow is an **AI-powered IELTS Practice Bot** that automatically generates and delivers IELTS-style Grammar, Writing, and Reading exercises to a Telegram chat on a recurring schedule. It uses **Google Gemini (gemini-2.5-flash)** to produce fresh, exam-relevant content and delivers it as formatted HTML Telegram messages.

**Day-based delivery schedule:**

| Day | Exercise Type | Output Branch |
|---|---|---|
| Monday | Grammar Practice | Branch 0 |
| Wednesday | Writing Task | Branch 1 |
| Friday | Reading Practice | Branch 2 |

**Logical blocks (in execution order):**

1. **Scheduling & Routing** — A daily cron-like trigger fires once per day; a Switch node routes execution to one of three branches based on the current day of the week.
2. **AI Content Generation** — Three Gemini AI nodes (one per exercise type) call the `gemini-2.5-flash` model with carefully crafted prompts that request strictly-structured JSON output.
3. **JSON Parsing** — Three Code nodes extract the raw text from the Gemini response and parse it into a proper JSON object; invalid JSON throws an error.
4. **Message Formatting** — Three Code nodes transform the parsed JSON into an HTML-formatted Telegram message (bold titles, bullet lists, question numbering, etc.).
5. **Message Splitting** — Three Code nodes chunk messages longer than 3,500 characters into multiple items, respecting Telegram's message length limits.
6. **Telegram Delivery** — Three Telegram nodes send each chunk to a pre-configured chat using HTML parse mode.

---

### 2. Block-by-Block Analysis

#### Block 1 — Scheduling & Routing

**Overview:** This block initiates the workflow on a daily schedule and determines which exercise type to generate based on the current day of the week. Only one branch executes per run.

**Nodes Involved:**
- Schedule Trigger
- Select Test by Day

**Node Details:**

| Attribute | Schedule Trigger | Select Test by Day |
|---|---|---|
| **Type** | `n8n-nodes-base.scheduleTrigger` (v1.2) | `n8n-nodes-base.switch` (v3.2) |
| **Technical Role** | Time-based entry point; emits one item per execution | Routes execution to the correct exercise branch based on day name |
| **Configuration** | Interval-based trigger with default empty interval (runs daily) | Three output rules: Monday → output "Monday", Wednesday → output "Wednesday", Friday → output "Friday". Each condition compares `$json['Day of week']` (string) to the literal day name using strict equals. |
| **Key Expressions** | — | `={{ $json['Day of week'] }}` evaluated against `"Monday"`, `"Wednesday"`, `"Friday"` |
| **Input Connections** | None (entry node) | From Schedule Trigger (main, index 0) |
| **Output Connections** | → Select Test by Day | Output 0 → Generate Grammar; Output 1 → Generate Writing; Output 2 → Generate Reading |
| **Edge Cases / Failure Types** | — If the schedule is misconfigured (e.g., wrong timezone), the day-of-week value may not match any rule, resulting in no output. <br> — The `$json['Day of week']` field depends on n8n's server timezone. | — If execution occurs on a Tuesday, Thursday, Saturday, or Sunday, no output branch matches and the workflow ends silently. <br> — `caseSensitive: true` and `typeValidation: "strict"` mean `"monday"` (lowercase) would not match. |

---

#### Block 2 — AI Content Generation

**Overview:** Three Gemini AI nodes generate IELTS practice content in strict JSON format. Each node uses a different, highly detailed system prompt tailored to the exercise type (Grammar, Writing, or Reading). All three use the `gemini-2.5-flash` model with JSON output enabled where applicable.

**Nodes Involved:**
- Generate Grammar
- Generate Writing
- Generate Reading

**Node Details — Generate Grammar:**

| Attribute | Value |
|---|---|
| **Type** | `@n8n/n8n-nodes-langchain.googleGemini` (v1) |
| **Technical Role** | Calls Gemini to produce a 10-question grammar exercise as JSON |
| **Configuration** | Model: `models/gemini-2.5-flash` (list mode). `jsonOutput: true`. Single message with a comprehensive prompt specifying: IELTS examiner persona, grammar focus areas (tenses, subject-verb agreement, articles, prepositions, conditionals, passive voice, relative clauses, modals, sentence structure), exactly 10 questions across 5 types (fill-in-the-blank, error correction, sentence transformation, multiple choice, sentence rewriting), Band 6.0–7.5 difficulty, and a strict JSON-only output schema including title, topic, grammar_focus, questions (with number, type, question, options, instruction), answer_key, explanations, and next_exercise. |
| **Key Expressions** | Topic placeholder `[INSERT TOPIC HERE]` — must be manually replaced for custom topics. |
| **Input Connections** | From Select Test by Day, output 0 (Monday) |
| **Output Connections** | → Parse Grammar JSON |
| **Credential** | Google Gemini (PaLM) API account |
| **Edge Cases / Failure Types** | — Gemini may return invalid JSON despite `jsonOutput: true` (markdown wrapping, trailing commas). <br> — The `[INSERT TOPIC HERE]` placeholder is not dynamically replaced; the AI will generate content based on the literal string unless edited. <br> — API rate limits or quota exhaustion can cause 429 errors. |

**Node Details — Generate Writing:**

| Attribute | Value |
|---|---|
| **Type** | `@n8n/n8n-nodes-langchain.googleGemini` (v1) |
| **Technical Role** | Calls Gemini to produce a single IELTS Writing task (Task 1 or Task 2) as JSON |
| **Configuration** | Model: `models/gemini-2.5-flash`. No `jsonOutput` flag (prompt-only enforcement). Single message prompt requesting: IELTS Writing examiner persona, Task 1 or Task 2 type, realistic IELTS question style, Band 6–7.5 difficulty, JSON-only output with title, task_type, question, instructions, and tips array. |
| **Key Expressions** | None |
| **Input Connections** | From Select Test by Day, output 1 (Wednesday) |
| **Output Connections** | → Parse Writing JSON |
| **Credential** | Google Gemini (PaLM) API account |
| **Edge Cases / Failure Types** | — Same JSON validity risk as Grammar node, compounded by no `jsonOutput` flag. <br> — May occasionally produce Task 1 (graph/letter) instead of Task 2 (essay) or vice versa. |

**Node Details — Generate Reading:**

| Attribute | Value |
|---|---|
| **Type** | `@n8n/nodes-base.googleGemini` (v1) — actually `@n8n/n8n-nodes-langchain.googleGemini` |
| **Technical Role** | Calls Gemini to produce a full IELTS Academic Reading passage with 13 questions as JSON |
| **Configuration** | Model: `models/gemini-2.5-flash`. `jsonOutput: true`. Single message prompt specifying: IELTS Academic Reading examiner persona, passage length 650–850 words (3–5 paragraphs, formal academic tone), exactly 13 questions across at least 5 different IELTS question types (matching headings, T/F/NG, multiple choice, sentence completion, summary completion, short answer, matching information, table/flow-chart/note completion), Band 6.5–7.5 difficulty, JSON-only output with title, topic, passage, questions (with number, type, question, options, answer_format), answer_key, and explanations. |
| **Key Expressions** | Topic placeholder `[INSERT TOPIC HERE]` — must be manually replaced. |
| **Input Connections** | From Select Test by Day, output 2 (Friday) |
| **Output Connections** | → Parse Reading JSON |
| **Credential** | Google Gemini (PaLM) API account |
| **Edge Cases / Failure Types** | — Passage may exceed 850 words or fall short of 650 despite instructions. <br> — Long passages may produce very large Telegram messages requiring splitting. <br> — Same JSON validity and API rate limit concerns. |

---

#### Block 3 — JSON Parsing

**Overview:** Three Code nodes extract the raw text from Gemini's response object (`content.parts[0].text`) and parse it into a JavaScript object. If the text is not valid JSON, the node throws an error, halting execution.

**Nodes Involved:**
- Parse Grammar JSON
- Parse Writing JSON
- Parse Reading JSON

**Node Details — all three share identical logic:**

| Attribute | Value |
|---|---|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Extracts and parses the JSON string from Gemini's structured response |
| **Configuration** | JavaScript code: reads `$input.first().json.content.parts[0].text`, attempts `JSON.parse()`, throws `"Invalid JSON from Gemini"` on failure, returns `[{ json: parsed }]`. |
| **Key Expressions** | `$input.first().json.content.parts[0].text` |
| **Input Connections** | Parse Grammar JSON ← Generate Grammar; Parse Writing JSON ← Generate Writing; Parse Reading JSON ← Generate Reading |
| **Output Connections** | Parse Grammar JSON → Format Grammar Message; Parse Writing JSON → Format Writing Message; Parse Reading JSON → Format Reading Message |
| **Edge Cases / Failure Types** | — If Gemini wraps JSON in markdown code fences (e.g., ` ```json … ``` `), `JSON.parse` will fail. <br> — If `content.parts` is empty or structured differently (e.g., `candidates[0].content.parts`), the expression will return `undefined` and throw a runtime error. <br> — The error message is generic; consider logging the raw text for debugging. |

---

#### Block 4 — Message Formatting

**Overview:** Three Code nodes transform the parsed JSON into human-readable, HTML-formatted Telegram messages. Each uses Telegram-compatible HTML tags (`<b>`, `<i>`) and structures the content differently based on exercise type.

**Nodes Involved:**
- Format Grammar Message
- Format Writing Message
- Format Reading Message

**Node Details — Format Grammar Message:**

| Attribute | Value |
|---|---|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Builds an HTML Telegram message with grammar exercise title, topic, grammar focus items, and numbered questions (type tag, question text, lettered options, instruction in italics). |
| **Configuration** | Builds string with `<b>` tags for titles, `<i>` for instructions, enumerates options as A/B/C/D using `String.fromCharCode(65 + i)`, handles missing arrays gracefully. |
| **Key Expressions** | `$input.first().json` |
| **Input Connections** | ← Parse Grammar JSON |
| **Output Connections** | → Split Grammar Message |
| **Edge Cases / Failure Types** | — If `data.questions` is empty or undefined, the message falls back to "No questions available." <br> — No HTML escaping is performed; special characters in questions could break Telegram's HTML parser. |

**Node Details — Format Writing Message:**

| Attribute | Value |
|---|---|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Builds an HTML Telegram message with writing task title, task type, question, instructions, numbered tips, and a call-to-action to write and submit. |
| **Configuration** | Uses an `esc()` helper function that escapes `&`, `<`, `>` to HTML entities (`&amp;`, `&lt;`, `&gt;`). Collapses triple+ newlines in the question to double. Trims whitespace. |
| **Key Expressions** | `$input.first().json`, `esc()` helper |
| **Input Connections** | ← Parse Writing JSON |
| **Output Connections** | → Split Writing Message |
| **Edge Cases / Failure Types** | — The `esc()` function does not handle double quotes or other Telegram-incompatible entities. <br> — If `data.title` is missing, defaults to "IELTS Writing Practice". |

**Node Details — Format Reading Message:**

| Attribute | Value |
|---|---|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Builds an HTML Telegram message with reading passage title, topic, a truncated passage (first 800 characters + "…"), and numbered questions. |
| **Configuration** | Passages are sliced to 800 characters (`data.passage.slice(0, 800)`) to avoid extremely long messages. Questions are listed by number and text. |
| **Key Expressions** | `$input.first().json` |
| **Input Connections** | ← Parse Reading JSON |
| **Output Connections** | → Split Reading Message |
| **Edge Cases / Failure Types** | — Passage truncation at 800 chars may cut mid-word or mid-sentence, producing an awkward "..." ending. <br> — No HTML escaping is applied, so special characters in the passage could break formatting. <br> — If `data.passage` is undefined, `.slice()` will throw a runtime error. |

---

#### Block 5 — Message Splitting

**Overview:** Three Code nodes split the formatted HTML message into chunks of at most 3,500 characters each. This is necessary because Telegram limits message length (officially 4,096 UTF-8 characters for HTML-formatted messages; 3,500 provides a safety margin).

**Nodes Involved:**
- Split Grammar Message
- Split Writing Message
- Split Reading Message

**Node Details — all three share identical logic:**

| Attribute | Value |
|---|---|
| **Type** | `n8n-nodes-base.code` (v2) |
| **Technical Role** | Splits a single long `telegram_message` string into multiple output items, each containing a chunk of ≤ 3,500 characters |
| **Configuration** | JavaScript code: reads `$input.first().json.telegram_message`, iterates in steps of 3,500 characters, creates one output item per chunk via `text.slice(i, i + chunkSize)`. |
| **Key Expressions** | `chunkSize = 3500` |
| **Input Connections** | Split Grammar Message ← Format Grammar Message; Split Writing Message ← Format Writing Message; Split Reading Message ← Format Reading Message |
| **Output Connections** | Split Grammar Message → Send Grammar; Split Writing Message → Send Writing; Split Reading Message → Send Reading |
| **Edge Cases / Failure Types** | — Splitting at an arbitrary character boundary may break an HTML tag (e.g., splitting inside `<b>Word` could produce invalid HTML in one chunk). <br> — For most exercise types, messages typically stay under 3,500 characters, so only one chunk is produced. <br> — The next Telegram node will send each chunk as a separate message, so the user receives them in rapid succession. |

---

#### Block 6 — Telegram Delivery

**Overview:** Three Telegram nodes send the formatted (and possibly chunked) messages to a specific chat. All three use HTML parse mode and reference a placeholder chat ID that must be replaced by the user.

**Nodes Involved:**
- Send Grammar
- Send Writing
- Send Reading

**Node Details — all three share identical configuration:**

| Attribute | Value |
|---|---|
| **Type** | `n8n-nodes-base.telegram` (v1.2) |
| **Technical Role** | Sends a Telegram message to a specific chat |
| **Configuration** | Text: `={{ $json.telegram_message }}`. Chat ID: `REPLACE_WITH_YOUR_CHAT_ID` (literal placeholder). Additional fields: `parse_mode: HTML`. |
| **Key Expressions** | `={{ $json.telegram_message }}` |
| **Input Connections** | Send Grammar ← Split Grammar Message; Send Writing ← Split Writing Message; Send Reading ← Split Reading Message |
| **Output Connections** | None (terminal nodes) |
| **Credential** | Telegram API credential ("Telegram account") |
| **Edge Cases / Failure Types** | — **Critical:** If the chat ID placeholder is not replaced, the Telegram API will return a `400 Bad Request: chat not found` error. <br> — If the bot has not been started by the user (no prior `/start` or message), the API may reject the send. <br> — Invalid HTML in the message (e.g., unclosed tags from splitting) results in a `400 Bad Request: can't parse entities` error. <br> — If Telegram rate limits are hit (bulk sending), 429 errors may occur. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | `n8n-nodes-base.scheduleTrigger` (v1.2) | Daily time-based trigger | — | Select Test by Day | Schedule Triggered — This workflow runs every day on a schedule. The output type changes depending on the day. |
| Select Test by Day | `n8n-nodes-base.switch` (v3.2) | Route execution by day of week | Schedule Trigger | Generate Grammar (out 0), Generate Writing (out 1), Generate Reading (out 2) | Schedule Triggered — This workflow runs every day on a schedule. The output type changes depending on the day. |
| Generate Grammar | `@n8n/n8n-nodes-langchain.googleGemini` (v1) | Generate 10 IELTS grammar questions via Gemini | Select Test by Day (Monday branch) | Parse Grammar JSON | AI Generated — This workflow uses AI to generate IELTS content. Make sure to Connect your AI credentials (Gemini / OpenAI) |
| Generate Writing | `@n8n/n8n-nodes-langchain.googleGemini` (v1) | Generate one IELTS writing task via Gemini | Select Test by Day (Wednesday branch) | Parse Writing JSON | AI Generated — This workflow uses AI to generate IELTS content. Make sure to Connect your AI credentials (Gemini / OpenAI) |
| Generate Reading | `@n8n/n8n-nodes-langchain.googleGemini` (v1) | Generate IELTS reading passage + 13 questions via Gemini | Select Test by Day (Friday branch) | Parse Reading JSON | AI Generated — This workflow uses AI to generate IELTS content. Make sure to Connect your AI credentials (Gemini / OpenAI) |
| Parse Grammar JSON | `n8n-nodes-base.code` (v2) | Parse Gemini raw text output into JSON object | Generate Grammar | Format Grammar Message | Parse — Parse AI output (Gemini) from raw text into valid JSON. Throws error if the response is not valid JSON |
| Parse Writing JSON | `n8n-nodes-base.code` (v2) | Parse Gemini raw text output into JSON object | Generate Writing | Format Writing Message | Parse — Parse AI output (Gemini) from raw text into valid JSON. Throws error if the response is not valid JSON |
| Parse Reading JSON | `n8n-nodes-base.code` (v2) | Parse Gemini raw text output into JSON object | Generate Reading | Format Reading Message | Parse — Parse AI output (Gemini) from raw text into valid JSON. Throws error if the response is not valid JSON |
| Format Grammar Message | `n8n-nodes-base.code` (v2) | Format parsed grammar JSON into HTML Telegram message | Parse Grammar JSON | Split Grammar Message | Format Data into Telegram Message — Convert AI output into a readable Telegram message with formatting and question list |
| Format Writing Message | `n8n-nodes-base.code` (v2) | Format parsed writing JSON into HTML Telegram message (with HTML escaping) | Parse Writing JSON | Split Writing Message | Format Data into Telegram Message — Convert AI output into a readable Telegram message with formatting and question list |
| Format Reading Message | `n8n-nodes-base.code` (v2) | Format parsed reading JSON into HTML Telegram message (passage truncated to 800 chars) | Parse Reading JSON | Split Reading Message | Format Data into Telegram Message — Convert AI output into a readable Telegram message with formatting and question list |
| Split Grammar Message | `n8n-nodes-base.code` (v2) | Split long grammar message into ≤3500-char chunks | Format Grammar Message | Send Grammar | Split Long Data — Split long Telegram message into smaller chunks to avoid message length limits. |
| Split Writing Message | `n8n-nodes-base.code` (v2) | Split long writing message into ≤3500-char chunks | Format Writing Message | Send Writing | Split Long Data — Split long Telegram message into smaller chunks to avoid message length limits. |
| Split Reading Message | `n8n-nodes-base.code` (v2) | Split long reading message into ≤3500-char chunks | Format Reading Message | Send Reading | Split Long Data — Split long Telegram message into smaller chunks to avoid message length limits. |
| Send Grammar | `n8n-nodes-base.telegram` (v1.2) | Send grammar exercise to Telegram chat | Split Grammar Message | — | Send data to Telegram — Send the formatted grammar message to Telegram. Don't forget to replace the Chat ID in the Send nodes. |
| Send Writing | `n8n-nodes-base.telegram` (v1.2) | Send writing task to Telegram chat | Split Writing Message | — | Send data to Telegram — Send the formatted grammar message to Telegram. Don't forget to replace the Chat ID in the Send nodes. |
| Send Reading | `n8n-nodes-base.telegram` (v1.2) | Send reading exercise to Telegram chat | Split Reading Message | — | Send data to Telegram — Send the formatted grammar message to Telegram. Don't forget to replace the Chat ID in the Send nodes. |
| Sticky Note8 | `n8n-nodes-base.stickyNote` (v1) | Overview documentation | — | — | 📘 IELTS Practice Bot (Telegram + AI). This workflow automatically generates IELTS practice content: Monday → Grammar Practice, Wednesday → Writing Task, Friday → Reading Practice. The content is generated using AI and delivered via Telegram. Flow: Schedule Trigger → Select Test by Day → AI → Parse → Format → Split → Send Telegram. ⚙️ Create Telegram BOT: 1. Create a Telegram Bot using @BotFather. 2. Copy your Bot Token. 3. Add Telegram credentials in n8n. 4. Add your AI (Gemini/OpenAI) credentials. 5. Activate the workflow. Make sure your bot has started and received at least one message. ⚠️ Important: You must replace the Chat ID in the Send nodes. Example: Find: REPLACE_WITH_YOUR_CHAT_ID — Replace with your own Chat ID. To get your Chat ID: 1. Send a message to your bot. 2. Open: https://api.telegram.org/bot<TOKEN>/getUpdates. 3. Copy "chat.id". |

---

### 4. Reproducing the Workflow from Scratch

Below is a step-by-step guide to recreate this workflow manually in n8n.

**Prerequisites:**
- A Google Gemini API key (with access to `gemini-2.5-flash`)
- A Telegram Bot token (obtained from @BotFather)
- Your Telegram Chat ID (obtained via `getUpdates`)

---

#### Step 1: Create the Schedule Trigger

1. Add a **Schedule Trigger** node.
2. Set the interval to run **once per day** (default empty interval in v1.2 triggers daily).
3. Ensure the n8n server timezone matches the day-of-week expectation (Monday/Wednesday/Friday).

#### Step 2: Add the Switch Node (Select Test by Day)

1. Add a **Switch** node (type: `n8n-nodes-base.switch`, version 3.2).
2. Name it **Select Test by Day**.
3. Add three output rules:
   - **Rule 1 (output key: Monday):** Condition — `$json['Day of week']` equals `"Monday"` (string, case-sensitive, strict type validation).
   - **Rule 2 (output key: Wednesday):** Condition — `$json['Day of week']` equals `"Wednesday"`.
   - **Rule 3 (output key: Friday):** Condition — `$json['Day of week']` equals `"Friday"`.
4. Connect **Schedule Trigger → Select Test by Day** (main, index 0).

#### Step 3: Add the Three Gemini AI Nodes

**3a — Generate Grammar:**
1. Add a **Google Gemini** node (`@n8n/n8n-nodes-langchain.googleGemini`, version 1).
2. Name it **Generate Grammar**.
3. Set model to `models/gemini-2.5-flash` (list mode).
4. Enable **JSON Output** (`jsonOutput: true`).
5. Add a single message with the full grammar exercise prompt (see prompt text below).
6. Configure credential: **Google Gemini (PaLM) API account**.
7. Connect **Select Test by Day, output 0 → Generate Grammar**.

**3b — Generate Writing:**
1. Add a **Google Gemini** node.
2. Name it **Generate Writing**.
3. Set model to `models/gemini-2.5-flash`.
4. Do **not** enable JSON Output (prompt-only enforcement).
5. Add a single message with the writing task prompt.
6. Use the same **Google Gemini (PaLM) API account** credential.
7. Connect **Select Test by Day, output 1 → Generate Writing**.

**3c — Generate Reading:**
1. Add a **Google Gemini** node.
2. Name it **Generate Reading**.
3. Set model to `models/gemini-2.5-flash`.
4. Enable **JSON Output** (`jsonOutput: true`).
5. Add a single message with the reading passage prompt.
6. Use the same **Google Gemini (PaLM) API account** credential.
7. Connect **Select Test by Day, output 2 → Generate Reading**.

**Grammar prompt summary:** Instruct the AI to act as a Band 9 IELTS examiner, generate exactly 10 grammar questions across 5 types (fill-in-the-blank, error correction, sentence transformation, multiple choice, sentence rewriting), Band 6.0–7.5 level, with strict JSON-only output containing title, topic, grammar_focus array, questions array (with number, type, question, options, instruction), answer_key, explanations, and next_exercise. Replace `[INSERT TOPIC HERE]` with your desired topic.

**Writing prompt summary:** Instruct the AI to act as an IELTS Writing examiner, generate one Task 1 or Task 2, realistic style, Band 6–7.5, JSON-only output with title, task_type, question, instructions, and tips array.

**Reading prompt summary:** Instruct the AI to act as a Band 9 IELTS Academic Reading examiner, produce a 650–850 word academic passage with exactly 13 questions across at least 5 question types, Band 6.5–7.5, JSON-only output with title, topic, passage, questions (with number, type, question, options, answer_format), answer_key, and explanations. Replace `[INSERT TOPIC HERE]` with your desired topic.

#### Step 4: Add the Three JSON Parsing Code Nodes

For each branch, add a **Code** node (`n8n-nodes-base.code`, version 2) with identical logic:

```javascript
const raw = $input.first().json.content.parts[0].text;
let parsed;
try {
  parsed = JSON.parse(raw);
} catch (e) {
  throw new Error("Invalid JSON from Gemini");
}
return [{ json: parsed }];
```

- **Parse Grammar JSON** ← Generate Grammar
- **Parse Writing JSON** ← Generate Writing
- **Parse Reading JSON** ← Generate Reading

#### Step 5: Add the Three Message Formatting Code Nodes

**5a — Format Grammar Message:**
1. Add a **Code** node, name it **Format Grammar Message**.
2. Code builds an HTML string with:
   - Bold title and topic
   - Bullet-listed grammar_focus items
   - Numbered questions with type tag `[type]`, question text, lettered options (A–D), italic instructions
   - Fallback "No questions available." if questions array is empty
3. Returns `[{ json: { telegram_message: message } }]`.
4. Connect **Parse Grammar JSON → Format Grammar Message**.

**5b — Format Writing Message:**
1. Add a **Code** node, name it **Format Writing Message**.
2. Code includes an `esc()` helper that escapes `&`, `<`, `>` to HTML entities.
3. Builds HTML with bold title (default "IELTS Writing Practice" if missing), task type, question (triple newlines collapsed), instructions, numbered tips, and a call-to-action.
4. Returns `[{ json: { telegram_message: message } }]`.
5. Connect **Parse Writing JSON → Format Writing Message**.

**5c — Format Reading Message:**
1. Add a **Code** node, name it **Format Reading Message**.
2. Code builds HTML with bold title and topic, passage truncated to first 800 characters + "…", and numbered questions.
3. Returns `[{ json: { telegram_message: message } }]`.
4. Connect **Parse Reading JSON → Format Reading Message**.

#### Step 6: Add the Three Message Splitting Code Nodes

For each branch, add a **Code** node with identical logic:

```javascript
const text = $input.first().json.telegram_message;
const chunkSize = 3500;
const chunks = [];
for (let i = 0; i < text.length; i += chunkSize) {
  chunks.push({ json: { telegram_message: text.slice(i, i + chunkSize) } });
}
return chunks;
```

- **Split Grammar Message** ← Format Grammar Message
- **Split Writing Message** ← Format Writing Message
- **Split Reading Message** ← Format Reading Message

#### Step 7: Add the Three Telegram Send Nodes

For each branch:

1. Add a **Telegram** node (`n8n-nodes-base.telegram`, version 1.2).
2. Configure:
   - **Text:** `={{ $json.telegram_message }}`
   - **Chat ID:** Replace `REPLACE_WITH_YOUR_CHAT_ID` with your actual numeric chat ID.
   - **Additional Fields → Parse Mode:** `HTML`
3. Configure credential: **Telegram account** (add your bot token).
4. Connections:
   - **Send Grammar** ← Split Grammar Message
   - **Send Writing** ← Split Writing Message
   - **Send Reading** ← Split Reading Message

#### Step 8: Credential Setup

**Google Gemini (PaLM) API:**
1. In n8n, go to Credentials → Add → search "Google Gemini (PaLM) API".
2. Paste your Gemini API key.
3. Save with a recognizable name (e.g., "Google Gemini(PaLM) Api account").

**Telegram API:**
1. In n8n, go to Credentials → Add → search "Telegram API".
2. Paste your bot token from @BotFather.
3. Save with a recognizable name (e.g., "Telegram account").

**Obtaining your Telegram Chat ID:**
1. Send any message to your bot in Telegram.
2. Open in a browser: `https://api.telegram.org/bot<TOKEN>/getUpdates` (replace `<TOKEN>` with your bot token).
3. Find the `chat.id` value in the JSON response and note it.
4. Use this numeric ID in all three Send nodes.

#### Step 9: Activate and Test

1. Save the workflow.
2. **Test manually first** — click "Execute Workflow" to run through each branch.
3. If all three branches work, **activate** the workflow (toggle switch in top-right).
4. The workflow will now run daily on schedule and send the appropriate exercise on Monday, Wednesday, and Friday.

#### Step 10: Optional Customization

- Replace `[INSERT TOPIC HERE]` in the Grammar and Reading prompts with dynamic or rotating topics.
- Adjust the `chunkSize` (3,500) in Split nodes if you experience Telegram message truncation.
- Modify the passage truncation length (800 chars) in Format Reading Message for longer or shorter previews.
- Add error handling (e.g., an Error Trigger node) to catch and log failures from Gemini or Telegram.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is inactive by default and must be activated manually after configuration. | Workflow `active: false` in the JSON |
| The day-of-week routing relies on n8n's server timezone. Ensure it is set correctly to match your intended schedule (e.g., UTC vs. local time). | Select Test by Day node |
| The `[INSERT TOPIC HERE]` placeholders in Grammar and Reading prompts are not dynamically resolved. Edit the prompts or replace them with expressions for rotating topics. | Generate Grammar and Generate Reading nodes |
| To get your Telegram Chat ID, send a message to your bot then visit: `https://api.telegram.org/bot<TOKEN>/getUpdates` | https://api.telegram.org/bot<TOKEN>/getUpdates |
| Create your Telegram Bot via @BotFather in Telegram. | Telegram setup instructions |
| The Split nodes perform character-level chunking which may break HTML tags at chunk boundaries. For production use, consider tag-aware splitting. | Split Grammar/Writing/Reading Message nodes |
| The Format Reading Message truncates the passage to 800 characters. Users do not see the full passage in Telegram; the full text exists only in the parsed JSON. | Format Reading Message node |
| HTML escaping is only implemented in Format Writing Message (the `esc()` function). Format Grammar Message and Format Reading Message do not escape HTML, which could cause Telegram HTML parse errors if the AI generates content with `<`, `>`, or `&`. | Format Grammar Message and Format Reading Message nodes |
| The workflow uses execution order `v1` as specified in settings. | Workflow settings |
| Gemini API free tier has rate limits. If you encounter 429 errors, consider adding a Wait node or reducing call frequency. | All Generate nodes |