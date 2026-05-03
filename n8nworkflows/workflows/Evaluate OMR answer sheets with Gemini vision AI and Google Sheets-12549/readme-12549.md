Evaluate OMR answer sheets with Gemini vision AI and Google Sheets

https://n8nworkflows.xyz/workflows/evaluate-omr-answer-sheets-with-gemini-vision-ai-and-google-sheets-12549


# Evaluate OMR answer sheets with Gemini vision AI and Google Sheets

## 1. Workflow Overview

**Title:** Evaluate OMR answer sheets with Gemini vision AI and Google Sheets  
**Purpose:** Accept an uploaded OMR (Optical Mark Recognition) answer sheet image via HTTP webhook, use Gemini vision to extract the student’s marked answers (and basic student details), compare them against a predefined answer key, compute scoring (including per-question 1/0 binary values), append the result to Google Sheets, and return the evaluation as a webhook JSON response.

### 1.1 Input Reception & Answer Key Setup
Receives the OMR image through a webhook endpoint and prepares the correct answers key used for evaluation.

### 1.2 AI Image Analysis (Vision Extraction)
Sends the uploaded image to a Gemini vision model with a strict prompt to extract *only* student details and bubbled answers.

### 1.3 Parsing, Merging & Evaluation
Parses the AI output into a usable “UserAnswers” format, merges it with the correct answers, and calculates totals, percentage, detailed per-question results, and a Google-Sheets-friendly binary map.

### 1.4 Persistence & Response
Appends a row into Google Sheets with student info, score summary, and Q.1–Q.20 binary correctness; then returns the full computed JSON to the webhook caller.

---

## 2. Block-by-Block Analysis

### Block 1 — Input Reception & Answer Key Setup
**Overview:** Accepts an uploaded OMR image via webhook and defines the official answer key used for scoring.  
**Nodes involved:** `Send Student Ans Img`, `Set Your Correct Answer`

#### Node: Send Student Ans Img
- **Type / role:** `Webhook` (`n8n-nodes-base.webhook`) — workflow entry point; receives HTTP POST uploads.
- **Key configuration (interpreted):**
  - **HTTP Method:** POST
  - **Path:** `omr-sheet-checker`
  - **Payload expectation:** Form-data file upload (see Sticky Note).
- **Inputs / outputs:**
  - **Output →** `Analyze image` (same incoming data forwarded)
  - **Output →** `Set Your Correct Answer` (parallel branch)
- **Variables/fields expected downstream:**
  - The AI node expects an **image** to be available from the incoming webhook binary/form-data.
- **Potential failures / edge cases:**
  - Caller sends JSON instead of `multipart/form-data`.
  - File field name mismatch (workflow expects `file`).
  - Very large images may cause memory/timeout issues or model failures later.

#### Node: Set Your Correct Answer
- **Type / role:** `Set` (`n8n-nodes-base.set`) — defines the correct answer key.
- **Key configuration:**
  - Sets a string field **`CorrectAnswer`** containing:  
    `1:B, 2:A, 3:D, ... 20:C`
- **Outputs:**
  - **Output →** `Merge Answer` (input index 0)
- **Potential failures / edge cases:**
  - Formatting drift (e.g., missing commas, different separators) may break parsing later.
  - If you change the number of questions, downstream mapping to Google Sheets (Q.1–Q.20) must also be updated.

---

### Block 2 — AI Image Analysis (Vision Extraction)
**Overview:** Uses Gemini vision to read an OMR sheet image and output student details plus per-question chosen options while following strict “do not guess” rules.  
**Nodes involved:** `Analyze image`

#### Node: Analyze image
- **Type / role:** `@n8n/n8n-nodes-langchain.ollama` — used here as a vision-capable LLM call (Gemini model selected).
- **Key configuration:**
  - **Resource:** image
  - **Model:** `gemini-3-flash-preview`
  - **Prompt intent:** strict extraction of bubbled answers only; skip unclear questions; pick darkest if multiple; extract student details.
  - **Output format option:** `format: json` (the node is configured to request JSON formatting).
  - **Simplify:** disabled (returns richer structure; downstream reads `message.content`).
- **Outputs:**
  - **Output →** `Convert User Ans in Formate`
- **Important behavior to note:**
  - Downstream parsing expects the model output to appear as text in `message.content` and contain `"name"`, `"rollNumber"`, `"class"` patterns plus lines like `1. A`.
- **Potential failures / edge cases:**
  - Model returns slightly different formatting (missing quotes, different keys like `roll_no`, extra prose).
  - Returns fenced code blocks (```), which the parser partially handles by stripping ```.
  - Vision quality issues: blurred photo, rotated sheet, shadows, low contrast bubbles.
  - Authentication/model availability depending on how the Ollama/LangChain node is wired to Gemini in your environment.

---

### Block 3 — Parsing, Merging & Evaluation
**Overview:** Converts AI output into a normalized answer string, merges it with the correct answer key, then computes scoring and per-question correctness.  
**Nodes involved:** `Convert User Ans in Formate`, `Merge Answer`, `Calculate Result`

#### Node: Convert User Ans in Formate
- **Type / role:** `Code` (`n8n-nodes-base.code`) — parses the AI output into structured fields.
- **Key configuration (logic):**
  - Reads `items[0].json.message?.content` as `content`.
  - Removes triple backticks: `content.replace(/```/g, "")`.
  - Extracts student details with regexes:
    - `"name"\s*:\s*"([^"]+)"`
    - `"rollNumber"\s*:\s*"([^"]+)"`
    - `"class"\s*:\s*"([^"]+)"`
  - Extracts answers with regex: `(\d+)\s*[:\.]\s*([A-D])`
    - Produces pairs like `1:A, 2:B, ...` in one string `UserAnswers`.
  - Output JSON:
    - `student: { name, rollNumber, class }`
    - `UserAnswers: "1:A, 2:B, ..."`
- **Outputs:**
  - **Output →** `Merge Answer` (input index 1)
- **Potential failures / edge cases:**
  - If Gemini returns unquoted keys/values, student regex won’t match (student fields remain null).
  - If answers include lowercase letters or spaces like `1) A`, the regex may miss them (it only supports `:` or `.` separators).
  - If the model returns JSON proper but nested differently, `message.content` may not contain the expected plain text.

#### Node: Merge Answer
- **Type / role:** `Merge` (`n8n-nodes-base.merge`) — combines correct answers and extracted student answers into a single item.
- **Key configuration:**
  - **Mode:** Combine
  - **Combine by:** position (`combineByPosition`)
  - Takes:
    - **Input 0:** from `Set Your Correct Answer` (contains `CorrectAnswer`)
    - **Input 1:** from `Convert User Ans in Formate` (contains `student`, `UserAnswers`)
- **Outputs:**
  - **Output →** `Calculate Result`
- **Potential failures / edge cases:**
  - If either branch produces multiple items, “combine by position” can mismatch data (e.g., answer key paired with the wrong student).
  - If one input is empty (AI failed), merge may output nothing or partial results depending on runtime behavior.

#### Node: Calculate Result
- **Type / role:** `Code` (`n8n-nodes-base.code`) — parses both keys, compares answers, builds summary + detailed outputs + binary map.
- **Key configuration (logic summary):**
  - `parseAnswers(input)`:
    - If object, returns it as-is.
    - If string, normalizes:
      - `->` replaced with `:`
      - newline replaced with comma
      - splits by comma, then `q:ans`
      - stores uppercase answers keyed by question number string.
  - Reads:
    - `student` from merged item (defaults to `{name:null, rollNumber:null, class:null}`)
    - `original` from `item.chatInput || item.CorrectAnswer || {}`
      - In this workflow it will typically come from `CorrectAnswer`.
    - `user` from `item.UserAnswers || {}`
  - Determines `total` as:
    - `Object.keys(original).length || Object.keys(user).length || 0`
  - For each question `i = 1..total`:
    - `given = user[q] || "Not Attempted"`
    - `correctAns = original[q] || "Not Available"`
    - Computes `isCorrect`, increments counters, stores:
      - `results[]` with `{question, given, correctAns, status, value}`
      - `binaryResult["Q.i"] = 1/0`
  - Computes:
    - `scorePercentage = (correct/total*100).toFixed(2)` (or `"0.00"`)
  - Outputs JSON:
    - `student`, `totalQuestions`, `correct`, `incorrect`, `scorePercentage`, `results`, `binaryResult`
- **Outputs:**
  - **Output →** `Append Result in Sheet`
- **Potential failures / edge cases:**
  - If the correct answer key includes question numbers but parsing fails (e.g., unusual separators), `total` may become 0.
  - “Not Attempted” vs “Not Available” will always count as incorrect (value 0).
  - If `total > results.length` assumptions break, downstream Google Sheets mapping that indexes `results[19]` can fail (see next block).

---

### Block 4 — Persistence & Response
**Overview:** Appends results into a Google Sheet and returns the evaluation to the caller.  
**Nodes involved:** `Append Result in Sheet`, `Send Result Respond`

#### Node: Append Result in Sheet
- **Type / role:** `Google Sheets` (`n8n-nodes-base.googleSheets`) — persists computed results to a spreadsheet row.
- **Key configuration:**
  - **Operation:** Append
  - **Document:** “Students Result” (ID `15nvRI8zcJ2n-QBXNk9xyd1GM8Vj7_1lJ3K7vO7Qfq3w`)
  - **Sheet tab:** `Sheet1` (`gid=0`)
  - **Mapping mode:** Define below (explicit column mapping)
  - Writes columns:
    - Student info: `Student Name`, `Roll No`, `Class`
    - Per question correctness: `Q.1`..`Q.20` from `$json.results[n].value`
    - Summary: `Correct`, `Incorrect`, `Score Percentage` (adds `%`)
- **Inputs / outputs:**
  - **Input:** from `Calculate Result`
  - **Output →** `Send Result Respond`
- **Potential failures / edge cases:**
  - **Critical indexing risk:** If `results` has fewer than 20 entries, expressions like `$json.results[19].value` will be `undefined` and can cause append failures or empty cells (depending on n8n/expression handling).
  - Spreadsheet column names must match exactly (case/spacing).
  - OAuth credential expiry or insufficient permissions.
  - If you change total questions, you must update both the sheet columns and these expressions.

#### Node: Send Result Respond
- **Type / role:** `Respond to Webhook` (`n8n-nodes-base.respondToWebhook`) — returns the final HTTP response.
- **Key configuration:**
  - Respond with JSON:
    - `{"status":"success","data": <entire $json from previous node> }`
- **Inputs:**
  - From `Append Result in Sheet`
- **Potential failures / edge cases:**
  - If Google Sheets append fails and stops execution, this response won’t be sent (caller will see an error/timeout unless you add error handling).
  - Large payloads (detailed `results`) could be big if you scale to many questions.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Send Student Ans Img | Webhook | Entry point; receives OMR image upload via POST | — | Analyze image; Set Your Correct Answer | ## Input Section<br>Receives OMR sheet image via webhook and sets up correct answer key for comparison.<br><br>## Google Sheet<br>https://docs.google.com/spreadsheets/d/15nvRI8zcJ2n-QBXNk9xyd1GM8Vj7_1lJ3K7vO7Qfq3w/edit?usp=sharing<br>## Webhook Payload :<br>Form-Data<br>key-name : file<br>Value : Upload Img |
| Analyze image | LangChain Ollama (image/LLM) | Gemini vision extracts student details + marked answers | Send Student Ans Img | Convert User Ans in Formate | ## AI Processing<br>Gemini AI analyzes the image to extract student details and marked answers from bubbles. |
| Convert User Ans in Formate | Code | Parses AI text into `student` + `UserAnswers` string | Analyze image | Merge Answer | ## Answer Comparison<br><br>Merges student answers with correct answers, then calculates score and generates detailed results. |
| Set Your Correct Answer | Set | Defines the correct answer key as `CorrectAnswer` | Send Student Ans Img | Merge Answer | ## Input Section<br>Receives OMR sheet image via webhook and sets up correct answer key for comparison. |
| Merge Answer | Merge | Combines correct key + extracted answers into one item | Set Your Correct Answer; Convert User Ans in Formate | Calculate Result | ## Answer Comparison<br><br>Merges student answers with correct answers, then calculates score and generates detailed results. |
| Calculate Result | Code | Compares answers; computes totals, percent, per-question 1/0 values | Merge Answer | Append Result in Sheet | ## Output Section<br>Saves results to Google Sheets and returns JSON response with complete evaluation data. |
| Append Result in Sheet | Google Sheets | Appends evaluation results into Google Sheet columns | Calculate Result | Send Result Respond | ## Output Section<br>Saves results to Google Sheets and returns JSON response with complete evaluation data.<br><br>## Google Sheet<br>https://docs.google.com/spreadsheets/d/15nvRI8zcJ2n-QBXNk9xyd1GM8Vj7_1lJ3K7vO7Qfq3w/edit?usp=sharing |
| Send Result Respond | Respond to Webhook | Returns final JSON to webhook caller | Append Result in Sheet | — | ## Output Section<br>Saves results to Google Sheets and returns JSON response with complete evaluation data. |
| Sticky Note | Sticky Note | Comment: Google Sheet link + webhook payload format | — | — | ## Google Sheet<br>https://docs.google.com/spreadsheets/d/15nvRI8zcJ2n-QBXNk9xyd1GM8Vj7_1lJ3K7vO7Qfq3w/edit?usp=sharing<br>## Webhook Payload :<br>Form-Data<br>key-name : file<br>Value : Upload Img |
| Workflow Overview | Sticky Note | Comment: high-level workflow explanation + setup steps | — | — | ## How it works<br>This workflow automates OMR (Optical Mark Recognition) answer sheet evaluation. Upload a student's answer sheet image via webhook, and AI analyzes the marked bubbles to extract answers. The system compares them against correct answers, calculates scores, and stores results in Google Sheets.<br><br>Key features:<br>- AI-powered bubble detection using Gemini<br>- Automatic student detail extraction (name, roll number, class)<br>- Question-by-question comparison with correct answers<br>- Binary scoring (1/0) for each question<br>- Real-time results via webhook response<br><br>## Setup steps<br>1. Configure Ollama credentials for Gemini AI model<br>2. Set correct answers in "Set Your Correct Answer" node<br>3. Connect Google Sheets OAuth2 credentials<br>4. Test webhook with form-data (key: "file", value: OMR image)<br>5. Verify results appear in Google Sheet |
| Input Section | Sticky Note | Comment: input block description | — | — | ## Input Section<br>Receives OMR sheet image via webhook and sets up correct answer key for comparison. |
| AI Processing | Sticky Note | Comment: AI block description | — | — | ## AI Processing<br>Gemini AI analyzes the image to extract student details and marked answers from bubbles. |
| Answer Comparison | Sticky Note | Comment: merge + scoring block description | — | — | ## Answer Comparison<br><br>Merges student answers with correct answers, then calculates score and generates detailed results. |
| Output Section | Sticky Note | Comment: output block description | — | — | ## Output Section<br>Saves results to Google Sheets and returns JSON response with complete evaluation data. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**
   - Node type: **Webhook**
   - Name: `Send Student Ans Img`
   - Method: **POST**
   - Path: **omr-sheet-checker**
   - In your API client (Postman/curl), send **multipart/form-data** with:
     - key: `file`
     - value: the OMR image file

2. **Create a Set node for the answer key**
   - Node type: **Set**
   - Name: `Set Your Correct Answer`
   - Add field:
     - `CorrectAnswer` (String)
     - Value: `1:B, 2:A, 3:D, 4:C, 5:B, 6:D, 7:B, 8:C, 9:A, 10:B, 11:D, 12:C, 13:A, 14:D, 15:B, 16:C, 17:D, 18:A, 19:B, 20:C`

3. **Create the Gemini vision analysis node**
   - Node type: **LangChain Ollama** (image-capable LLM node as used in this workflow)
   - Name: `Analyze image`
   - Resource: **Image**
   - Model: **gemini-3-flash-preview**
   - Output format option: **JSON**
   - Paste the provided strict prompt (ensuring it instructs one answer per line and student details).
   - **Credentials:** configure the node’s LLM credentials according to your n8n setup (the workflow note calls this “Ollama credentials for Gemini AI model”).

4. **Create a Code node to parse AI output**
   - Node type: **Code**
   - Name: `Convert User Ans in Formate`
   - Paste logic that:
     - Reads `items[0].json.message.content`
     - Strips ``` fences
     - Regex extracts `"name"`, `"rollNumber"`, `"class"`
     - Regex extracts answers `(\d+)\s*[:\.]\s*([A-D])`
     - Outputs `{ student, UserAnswers }`

5. **Create a Merge node**
   - Node type: **Merge**
   - Name: `Merge Answer`
   - Mode: **Combine**
   - Combine by: **Position**
   - This will merge:
     - Input 0: correct answer item
     - Input 1: student extracted answers item

6. **Create a Code node to calculate results**
   - Node type: **Code**
   - Name: `Calculate Result`
   - Implement:
     - `parseAnswers()` that supports object or `1:A, 2:B` string
     - Comparison loop `1..total`
     - Output: `student`, `totalQuestions`, `correct`, `incorrect`, `scorePercentage`, `results[]` with `value` (1/0), plus `binaryResult` map (`Q.1` → 1/0).

7. **Create a Google Sheets node to append**
   - Node type: **Google Sheets**
   - Name: `Append Result in Sheet`
   - Credentials: **Google Sheets OAuth2**
   - Operation: **Append**
   - Select Spreadsheet: `Students Result` (or your own)
   - Select sheet/tab: `Sheet1`
   - Map columns (define below) for:
     - Student Name, Roll No, Class
     - Q.1..Q.20 from `$json.results[0].value` .. `$json.results[19].value`
     - Correct, Incorrect, Score Percentage (e.g. `={{ $json.scorePercentage }}%`)
   - Ensure your sheet has matching column headers.

8. **Create Respond to Webhook**
   - Node type: **Respond to Webhook**
   - Name: `Send Result Respond`
   - Respond with: **JSON**
   - Body expression:
     - `status: "success"`
     - `data: $json`

9. **Wire the nodes (connection order)**
   - `Send Student Ans Img` → `Analyze image`
   - `Send Student Ans Img` → `Set Your Correct Answer`
   - `Analyze image` → `Convert User Ans in Formate`
   - `Set Your Correct Answer` → `Merge Answer` (input 0)
   - `Convert User Ans in Formate` → `Merge Answer` (input 1)
   - `Merge Answer` → `Calculate Result`
   - `Calculate Result` → `Append Result in Sheet`
   - `Append Result in Sheet` → `Send Result Respond`

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Sheet used by this workflow | https://docs.google.com/spreadsheets/d/15nvRI8zcJ2n-QBXNk9xyd1GM8Vj7_1lJ3K7vO7Qfq3w/edit?usp=sharing |
| Webhook payload requirement | Form-Data; key-name: `file`; value: upload image |
| Workflow behavior summary and setup checklist (as embedded note) | Includes: configure Gemini/Ollama credentials, set correct answers, connect Google Sheets OAuth2, test webhook upload, verify sheet output |
| Disclaimer | Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n… Toutes les données manipulées sont légales et publiques. |