Generate AI math RPG quests from study logs with Google Sheets and OpenRouter

https://n8nworkflows.xyz/workflows/generate-ai-math-rpg-quests-from-study-logs-with-google-sheets-and-openrouter-14193


# Generate AI math RPG quests from study logs with Google Sheets and OpenRouter

# 1. Workflow Overview

This workflow generates an AI-created math RPG quest from a learner’s submitted study time and stores the result in Google Sheets. Its purpose is to gamify study tracking: each study log triggers the creation of a level-based math challenge framed as an RPG encounter.

Typical use cases:
- Classroom or homeschooling reward systems
- Self-learning motivation systems
- Study habit tracking with instant AI-generated exercises
- Lightweight RPG-style educational experiences backed by spreadsheets

The workflow is composed of three main logical blocks.

## 1.1 Input Reception and Study Log Storage

The workflow starts with an n8n Form where a user submits:
- `user_id`
- `time` in minutes

The submitted data is appended to the `StudyLogs` tab in Google Sheets.

## 1.2 User Status Retrieval and AI Quest Generation

After logging the study session, the workflow reads user information from the `Users` sheet, then sends the user’s level and ID into a LangChain-based AI generation step. The AI is instructed to produce:
- a math-themed monster
- a short story/appearance description
- one math question suited to the user level
- a numeric answer only

This generation uses:
- an OpenRouter chat model
- a structured JSON output parser

## 1.3 Quest Persistence

The generated quest is appended to the `Quests` tab in Google Sheets with:
- `user_id`
- `enemy`
- `question`
- `answer`
- `status = pending`

This makes the generated quest available for a later answering/battle workflow.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Study Log Storage

### Overview

This block collects study activity from a user through an n8n form and records that study session into the `StudyLogs` sheet. It serves as the workflow entry point and provides the raw input used downstream.

### Nodes Involved

- Study Log Input Form
- Add to StudyLogs

### Node Details

#### 2.1 Study Log Input Form

- **Type and technical role:**  
  `n8n-nodes-base.formTrigger`  
  Entry-point trigger node that exposes a hosted form and starts the workflow when submitted.

- **Configuration choices:**  
  The form is configured with:
  - title: `AI Math RPG - Study Input`
  - description: `Enter your study time (in minutes) to summon a monster!`
  - required field `user_id`
  - required numeric field `time`

- **Key expressions or variables used:**  
  This node emits form values as JSON, notably:
  - `$json.user_id`
  - `$json.time`

- **Input and output connections:**  
  - Input: none, this is the workflow trigger
  - Output: connected to `Add to StudyLogs`

- **Version-specific requirements:**  
  Uses `typeVersion: 2.3`. This requires the n8n Form Trigger feature available in modern n8n builds.

- **Edge cases or potential failure types:**  
  - Form submissions may fail if the workflow is inactive or not reachable
  - Missing fields are prevented by required validation
  - `time` is numeric, but very large or negative values are not explicitly constrained here
  - Duplicate `user_id` submissions are allowed

- **Sub-workflow reference:**  
  None

---

#### 2.2 Add to StudyLogs

- **Type and technical role:**  
  `n8n-nodes-base.googleSheets`  
  Appends each submitted study log to the `StudyLogs` worksheet.

- **Configuration choices:**  
  - Operation: `append`
  - Target sheet: `StudyLogs`
  - Spreadsheet: placeholder value `ENTER_YOUR_SPREADSHEET_ID_HERE`
  - Defined column mapping:
    - `time` = `{{ $json.time }}`
    - `user_id` = `{{ $json.user_id }}`

- **Key expressions or variables used:**  
  - `={{ $json.time }}`
  - `={{ $json.user_id }}`

- **Input and output connections:**  
  - Input: `Study Log Input Form`
  - Output: `Get User Status`

- **Version-specific requirements:**  
  Uses `typeVersion: 4.7`, which expects the newer Google Sheets node behavior and OAuth2 credential support.

- **Edge cases or potential failure types:**  
  - Invalid or missing Google Sheets OAuth2 credentials
  - Spreadsheet ID not replaced from placeholder
  - Missing `StudyLogs` tab
  - Column names in sheet not matching configured schema
  - API quota/rate limiting from Google
  - Data type mismatch if the sheet enforces formatting

- **Sub-workflow reference:**  
  None

---

## Block 2 — User Status Retrieval and AI Quest Generation

### Overview

This block retrieves user information from the spreadsheet and uses AI to generate a structured math RPG quest. It is the core business logic of the workflow and depends on both spreadsheet data integrity and AI output conformance.

### Nodes Involved

- Get User Status
- OpenRouter Chat Model
- Structured Output Parser
- Basic LLM Chain

### Node Details

#### 2.3 Get User Status

- **Type and technical role:**  
  `n8n-nodes-base.googleSheets`  
  Reads data from the `Users` sheet so the workflow can access fields such as the learner’s current level.

- **Configuration choices:**  
  - Target sheet: `Users`
  - Spreadsheet: placeholder value `ENTER_YOUR_SPREADSHEET_ID_HERE`
  - No explicit operation is shown in the exported JSON, so this uses the node’s default read behavior for the selected version

- **Key expressions or variables used:**  
  No expressions are configured directly in this node, but downstream logic expects fields from this node’s output, especially:
  - `$json.user_id`
  - `$json.level`

- **Input and output connections:**  
  - Input: `Add to StudyLogs`
  - Output: `Basic LLM Chain`

- **Version-specific requirements:**  
  Uses `typeVersion: 4.7`

- **Edge cases or potential failure types:**  
  - The sheet may return all users rather than only the current user if filtering is not added
  - If no `level` field exists, the prompt may render with empty values
  - If multiple rows exist for a user, behavior may be ambiguous
  - Missing `Users` tab or wrong spreadsheet ID
  - Credential and quota issues as with all Google Sheets nodes

- **Sub-workflow reference:**  
  None

- **Important implementation note:**  
  Despite the sticky note text claiming the workflow “calculates total EXP, and retrieves the user's current level,” there is no calculation node in this workflow. The current design assumes the `Users` sheet already contains the necessary `level` information.

---

#### 2.4 OpenRouter Chat Model

- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  Supplies the language model used by the LangChain chain.

- **Configuration choices:**  
  - Model: `openai/gpt-4o-mini`
  - `maxTokens`: `500`

- **Key expressions or variables used:**  
  No dynamic expressions are used here.

- **Input and output connections:**  
  - Input connection type: `ai_languageModel`
  - Connected to: `Basic LLM Chain`

- **Version-specific requirements:**  
  Uses `typeVersion: 1` for the OpenRouter LangChain model node. Requires an OpenRouter-compatible credential configured in n8n.

- **Edge cases or potential failure types:**  
  - Missing/invalid OpenRouter API credentials
  - Selected model unavailable on the account
  - Rate limiting or provider-side outages
  - Response truncation if prompt plus output exceeds token limits
  - Non-deterministic model behavior can still produce malformed content, though the parser reduces that risk

- **Sub-workflow reference:**  
  None

---

#### 2.5 Structured Output Parser

- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces that the LLM returns a JSON structure matching the expected quest schema.

- **Configuration choices:**  
  The example schema expects:
  - `story` as string
  - `enemy` as string
  - `question` as string
  - `answer` as numeric

  Example used:
  - story: `"A slime shaped like a number emerged from the darkness!"`
  - enemy: `"Calc Slime"`
  - question: `"15 + 28 = ?"`
  - answer: `43`

- **Key expressions or variables used:**  
  None directly. The parser shapes the `Basic LLM Chain` output into structured JSON under the chain result.

- **Input and output connections:**  
  - Input connection type: `ai_outputParser`
  - Connected to: `Basic LLM Chain`

- **Version-specific requirements:**  
  Uses `typeVersion: 1.3`

- **Edge cases or potential failure types:**  
  - LLM output may fail schema validation
  - `answer` may be returned as text instead of a number
  - Extra fields are usually harmless, but missing required fields can break downstream mapping
  - If the parser fails, `Save Quest` will not receive expected `output.*` properties

- **Sub-workflow reference:**  
  None

---

#### 2.6 Basic LLM Chain

- **Type and technical role:**  
  `@n8n/n8n-nodes-langchain.chainLlm`  
  Builds the AI prompt, injects user-specific data, uses the OpenRouter model, and applies the structured parser.

- **Configuration choices:**  
  - Prompt type: `define`
  - Structured parser enabled
  - Prompt text:

  The prompt includes:
  - current `user_id`
  - current `level`
  - instruction to generate one math problem from addition, subtraction, multiplication, or division
  - instruction to tailor difficulty to level
  - request for an RPG monster name and appearance text
  - explicit instruction: the `answer` field must contain only a numeric value

- **Key expressions or variables used:**  
  - `{{$json.user_id}}`
  - `{{$json.level}}`

- **Input and output connections:**  
  - Main input: `Get User Status`
  - AI language model input: `OpenRouter Chat Model`
  - AI output parser input: `Structured Output Parser`
  - Main output: `Save Quest`

- **Version-specific requirements:**  
  Uses `typeVersion: 1.7`

- **Edge cases or potential failure types:**  
  - If `Get User Status` does not provide `user_id` or `level`, prompt quality degrades
  - If the upstream sheet returns multiple rows, the chain may generate one quest per row
  - Prompt does not explicitly restrict impossible operations such as division with non-integer answers, so the model may produce decimals unless otherwise constrained
  - Malformed output can still occur if the model ignores instructions
  - Model/auth/rate-limit failures propagate here

- **Sub-workflow reference:**  
  None

---

## Block 3 — Quest Persistence

### Overview

This block writes the generated quest into the `Quests` sheet. It stores only the fields needed for later quiz resolution and marks the new quest as pending.

### Nodes Involved

- Save Quest

### Node Details

#### 2.7 Save Quest

- **Type and technical role:**  
  `n8n-nodes-base.googleSheets`  
  Appends the generated quest record to the `Quests` worksheet.

- **Configuration choices:**  
  - Operation: `append`
  - Target sheet: `Quests`
  - Spreadsheet: placeholder value `ENTER_YOUR_SPREADSHEET_ID_HERE`
  - Column mappings:
    - `enemy` = `{{ $json.output.enemy }}`
    - `answer` = `{{ $json.output.answer }}`
    - `status` = `pending`
    - `user_id` = `{{ $('Study Log Input Form').item.json.user_id }}`
    - `question` = `{{ $json.output.question }}`

- **Key expressions or variables used:**  
  - `={{ $json.output.enemy }}`
  - `={{ $json.output.answer }}`
  - `={{ $json.output.question }}`
  - `={{ $('Study Log Input Form').item.json.user_id }}`
  - static value: `pending`

- **Input and output connections:**  
  - Input: `Basic LLM Chain`
  - Output: none

- **Version-specific requirements:**  
  Uses `typeVersion: 4.7`

- **Edge cases or potential failure types:**  
  - If the LLM chain output schema differs, `output.enemy`, `output.answer`, or `output.question` may be undefined
  - If the expression referencing `Study Log Input Form` loses item linking context, `user_id` resolution may fail
  - Missing `Quests` tab or schema mismatch
  - Invalid credentials, quota issues, or spreadsheet ID placeholder not replaced
  - `answer` is stored into a sheet column typed as string in the schema; this is acceptable but may be inconsistent with numeric downstream checks

- **Sub-workflow reference:**  
  None

---

## Non-executable Nodes — Sticky Notes

These nodes do not participate in execution but provide important workflow context and should be preserved when documenting or reconstructing the design.

### 2.8 Sticky Note

- **Type and technical role:**  
  `n8n-nodes-base.stickyNote`  
  Documentation note covering the overall workflow.

- **Configuration choices:**  
  Contains a long description including:
  - intended audience
  - high-level operation
  - setup steps
  - requirements
  - customization ideas

- **Input and output connections:**  
  None

- **Edge cases or potential failure types:**  
  None at runtime

- **Sub-workflow reference:**  
  Mentions that this workflow is “Part 1: Quest Generation” in a two-workflow system and references a second workflow called “Part 2: Quiz Battle,” but no sub-workflow node exists in this JSON.

---

### 2.9 Sticky Note1

- **Type and technical role:**  
  `n8n-nodes-base.stickyNote`  
  Visual label for the first functional block.

- **Configuration choices:**  
  Content: `Receive study time and calculate user level`

- **Input and output connections:**  
  None

- **Edge cases or potential failure types:**  
  None

- **Sub-workflow reference:**  
  None

- **Important accuracy note:**  
  The note says “calculate user level,” but the workflow does not actually perform a level calculation. It only reads data from the `Users` sheet.

---

### 2.10 Sticky Note2

- **Type and technical role:**  
  `n8n-nodes-base.stickyNote`  
  Visual label for the AI generation block.

- **Configuration choices:**  
  Content: `AI generates RPG quest`

- **Input and output connections:**  
  None

- **Edge cases or potential failure types:**  
  None

- **Sub-workflow reference:**  
  None

---

### 2.11 Sticky Note3

- **Type and technical role:**  
  `n8n-nodes-base.stickyNote`  
  Visual label for the persistence block.

- **Configuration choices:**  
  Content: `Save quest to database`

- **Input and output connections:**  
  None

- **Edge cases or potential failure types:**  
  None

- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Study Log Input Form | n8n-nodes-base.formTrigger | Receives user study log input through a hosted form |  | Add to StudyLogs | Receive study time and calculate user level |
| Add to StudyLogs | n8n-nodes-base.googleSheets | Appends submitted study session data to `StudyLogs` | Study Log Input Form | Get User Status | Receive study time and calculate user level |
| Get User Status | n8n-nodes-base.googleSheets | Reads user data, expected to include current level, from `Users` | Add to StudyLogs | Basic LLM Chain | Receive study time and calculate user level |
| OpenRouter Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides the LLM used to generate the quest |  | Basic LLM Chain | AI generates RPG quest |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces structured JSON output for the generated quest |  | Basic LLM Chain | AI generates RPG quest |
| Basic LLM Chain | @n8n/n8n-nodes-langchain.chainLlm | Generates a level-based math RPG encounter from user context | Get User Status, OpenRouter Chat Model, Structured Output Parser | Save Quest | AI generates RPG quest |
| Save Quest | n8n-nodes-base.googleSheets | Appends generated quest data to `Quests` with pending status | Basic LLM Chain |  | Save quest to database |
| Sticky Note | n8n-nodes-base.stickyNote | Overall documentation note for workflow purpose, setup, and customization |  |  | ## Generate AI math RPG quests from study logs in Google Sheets  /  ## Who it's for  /  This template is for educators, parents, or self-learners who want to gamify their study routines. It turns boring study logs into an interactive RPG math battle using AI.  /  ## How it works  /  This system consists of two workflows (Part 1: Quest Generation, Part 2: Quiz Battle).  /  In this workflow, when a user submits their study time via an n8n Form, the system logs the data into Google Sheets, calculates total EXP, and retrieves the user's current level. Then, it uses a Basic LLM Chain to generate a math question tailored to that level, along with an RPG-style monster and flavor text. The generated quest is saved back to Google Sheets with a `pending` status, ready for the user to answer.  /  ## How to set up  /  1. Create a Google Sheet with three tabs: `StudyLogs`, `Users`, and `Quests`.  /  2. Connect your Google Sheets credential and select your spreadsheet in all Google Sheets nodes.  /  3. Connect your OpenAI or OpenRouter credential.  /  4. Run the "Study Log Input Form" to generate your first monster!  /  ## Requirements  /  - A Google account (for Google Sheets)  /  - An OpenAI or OpenRouter API key  /  ## How to customize the workflow  /  You can easily change the prompt in the "Generate RPG Quest" node to create language quizzes, history trivia, or any other subject instead of math! |
| Sticky Note1 | n8n-nodes-base.stickyNote | Visual label for input and user-state section |  |  | Receive study time and calculate user level |
| Sticky Note2 | n8n-nodes-base.stickyNote | Visual label for AI generation section |  |  | AI generates RPG quest |
| Sticky Note3 | n8n-nodes-base.stickyNote | Visual label for save section |  |  | Save quest to database |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n**
   - Name it something like: `Generate AI math RPG quests from study logs in Google Sheets`.

2. **Prepare the Google Sheets file**
   - Create one spreadsheet with three tabs:
     - `StudyLogs`
     - `Users`
     - `Quests`

3. **Define recommended sheet columns**
   - In `StudyLogs`, create columns such as:
     - `user_id`
     - `time`
   - In `Users`, create columns such as:
     - `user_id`
     - `level`
   - In `Quests`, create columns such as:
     - `user_id`
     - `enemy`
     - `question`
     - `answer`
     - `status`

4. **Populate the `Users` sheet before testing**
   - Add at least one row with:
     - a valid `user_id`
     - a `level` value
   - Example:
     - `user_id = student_01`
     - `level = 3`

5. **Create the trigger node**
   - Add a **Form Trigger** node.
   - Name it: `Study Log Input Form`.
   - Set:
     - Form Title: `AI Math RPG - Study Input`
     - Form Description: `Enter your study time (in minutes) to summon a monster!`
   - Add two form fields:
     1. `user_id`
        - type: text
        - required: yes
     2. `time`
        - type: number
        - required: yes

6. **Create the study log append node**
   - Add a **Google Sheets** node.
   - Name it: `Add to StudyLogs`.
   - Connect `Study Log Input Form` → `Add to StudyLogs`.
   - Configure:
     - Resource/operation: append row
     - Spreadsheet: select your spreadsheet
     - Sheet: `StudyLogs`
   - Map fields manually:
     - `user_id` → `{{ $json.user_id }}`
     - `time` → `{{ $json.time }}`
   - Attach a valid **Google Sheets OAuth2** credential.

7. **Create the user lookup node**
   - Add another **Google Sheets** node.
   - Name it: `Get User Status`.
   - Connect `Add to StudyLogs` → `Get User Status`.
   - Point it to:
     - Spreadsheet: same spreadsheet
     - Sheet: `Users`
   - Configure it to read user data.
   - For a production-safe build, add filtering so it returns only the row matching the current `user_id`.
     - Match `user_id` in the sheet against `{{ $('Study Log Input Form').item.json.user_id }}` or current input JSON as appropriate.
   - Ensure the returned row includes `level`.

8. **Create the AI model node**
   - Add an **OpenRouter Chat Model** node from the LangChain nodes.
   - Name it: `OpenRouter Chat Model`.
   - Set:
     - Model: `openai/gpt-4o-mini`
     - Max Tokens: `500`
   - Configure an **OpenRouter** credential with your API key.

9. **Create the structured parser node**
   - Add a **Structured Output Parser** node.
   - Name it: `Structured Output Parser`.
   - Use a schema example like:
     ```json
     {
       "story": "A slime shaped like a number emerged from the darkness!",
       "enemy": "Calc Slime",
       "question": "15 + 28 = ?",
       "answer": 43
     }
     ```
   - The important expected fields are:
     - `story` as text
     - `enemy` as text
     - `question` as text
     - `answer` as number

10. **Create the LLM chain node**
    - Add a **Basic LLM Chain** node.
    - Name it: `Basic LLM Chain`.
    - Configure:
      - Prompt type: define manually
      - Enable structured output parsing
    - Paste this prompt:
      ```text
      User '{{$json.user_id}}' is currently at Level {{$json.level}}.
      Please generate ONE math problem (addition, subtraction, multiplication, or division) suitable for this level.
      Also, generate a math-themed RPG monster name and a short descriptive text of its appearance.
      * Make sure the 'answer' field contains ONLY a numeric value.
      ```
    - Connect:
      - `OpenRouter Chat Model` → `Basic LLM Chain` using the AI language model connection
      - `Structured Output Parser` → `Basic LLM Chain` using the AI output parser connection
      - `Get User Status` → `Basic LLM Chain` using the main connection

11. **Create the quest save node**
    - Add another **Google Sheets** node.
    - Name it: `Save Quest`.
    - Connect `Basic LLM Chain` → `Save Quest`.
    - Configure:
      - Operation: append row
      - Spreadsheet: same spreadsheet
      - Sheet: `Quests`
    - Map columns manually:
      - `user_id` → `{{ $('Study Log Input Form').item.json.user_id }}`
      - `enemy` → `{{ $json.output.enemy }}`
      - `question` → `{{ $json.output.question }}`
      - `answer` → `{{ $json.output.answer }}`
      - `status` → `pending`

12. **Add optional sticky notes for clarity**
    - Add one note for the first section:
      - `Receive study time and calculate user level`
    - Add one note for the AI section:
      - `AI generates RPG quest`
    - Add one note for the save section:
      - `Save quest to database`
    - Add one general note containing setup guidance and project context if desired.

13. **Verify credential setup**
    - Google Sheets nodes:
      - all should use the same Google Sheets OAuth2 credential unless intentionally separated
    - OpenRouter Chat Model:
      - must use a valid OpenRouter API credential
    - Replace any spreadsheet placeholder values with the actual spreadsheet

14. **Test with sample data**
    - Submit the form with a `user_id` that exists in `Users`
    - Example:
      - `user_id = student_01`
      - `time = 25`

15. **Validate the outputs**
    - Confirm one new row appears in `StudyLogs`
    - Confirm the workflow reads the correct user row from `Users`
    - Confirm one new row appears in `Quests` with:
      - a generated `enemy`
      - a generated `question`
      - a numeric `answer`
      - `status = pending`

16. **Recommended hardening improvements**
    - Add a filter in `Get User Status` to ensure only the submitted user is read
    - Add an IF node after lookup to handle missing users
    - Add validation to reject negative or zero `time`
    - Add retry/error workflow handling for Google Sheets and OpenRouter failures
    - Optionally store `story` too, since it is generated but currently not saved

### Sub-workflow setup

There are **no actual sub-workflow nodes** in this JSON.

However, the notes indicate this workflow is intended as **Part 1** of a two-workflow system, followed by a separate “Quiz Battle” workflow. If building the full system, the second workflow should expect to read from the `Quests` sheet and process rows where:
- `status = pending`
- `user_id` identifies the learner
- `question` and `answer` contain the generated challenge

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Generate AI math RPG quests from study logs in Google Sheets | Workflow theme / title |
| This template is for educators, parents, or self-learners who want to gamify their study routines. It turns boring study logs into an interactive RPG math battle using AI. | Audience and purpose |
| This system consists of two workflows (Part 1: Quest Generation, Part 2: Quiz Battle). | Project architecture note |
| In this workflow, when a user submits their study time via an n8n Form, the system logs the data into Google Sheets, calculates total EXP, and retrieves the user's current level. Then, it uses a Basic LLM Chain to generate a math question tailored to that level, along with an RPG-style monster and flavor text. The generated quest is saved back to Google Sheets with a `pending` status, ready for the user to answer. | High-level behavior note |
| Create a Google Sheet with three tabs: `StudyLogs`, `Users`, and `Quests`. | Setup requirement |
| Connect your Google Sheets credential and select your spreadsheet in all Google Sheets nodes. | Setup requirement |
| Connect your OpenAI or OpenRouter credential. | Setup requirement |
| Run the "Study Log Input Form" to generate your first monster! | Operational note |
| A Google account (for Google Sheets) | Requirement |
| An OpenAI or OpenRouter API key | Requirement |
| You can easily change the prompt in the "Generate RPG Quest" node to create language quizzes, history trivia, or any other subject instead of math! | Customization idea |

## Additional implementation note

There is a discrepancy between the descriptive notes and the executable logic:
- the notes say the workflow calculates EXP and user level
- the actual workflow only logs study time and reads from `Users`

So, to reproduce the behavior exactly as implemented, ensure the `Users` sheet is pre-maintained externally or extend the workflow with a proper lookup and leveling calculation layer.