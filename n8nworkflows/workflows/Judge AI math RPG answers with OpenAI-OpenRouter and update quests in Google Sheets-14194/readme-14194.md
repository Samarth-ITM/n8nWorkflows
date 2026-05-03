Judge AI math RPG answers with OpenAI/OpenRouter and update quests in Google Sheets

https://n8nworkflows.xyz/workflows/judge-ai-math-rpg-answers-with-openai-openrouter-and-update-quests-in-google-sheets-14194


# Judge AI math RPG answers with OpenAI/OpenRouter and update quests in Google Sheets

# 1. Workflow Overview

This workflow judges a player’s submitted answer in an “AI Math RPG” system and updates the corresponding quest in Google Sheets. It is the answer-validation part of the game loop: a user submits an answer through an n8n form, the workflow looks up that user’s pending quest, compares the submitted answer with the stored correct answer, and then either marks the quest as solved and generates an RPG-style victory message, or returns a miss message.

Typical use cases:
- Gamified math practice for students
- Parent- or teacher-managed study quests
- Self-study systems using Google Sheets as lightweight quest storage
- Low-cost validation where simple answer matching is preferred over model-based grading

## 1.1 Input Reception and Quest Lookup

The workflow starts with an n8n Form submission containing `user_id` and `answer`. It then searches the `Quests` sheet in Google Sheets for a row where the same `user_id` has a `pending` status.

## 1.2 Answer Validation

Once a pending quest row is found, the workflow compares the submitted answer from the form against the stored answer from Google Sheets using a native IF node. This avoids unnecessary LLM cost and latency.

## 1.3 Success Path: Quest Update and Victory Message

If the answer matches, the workflow updates the matching Google Sheets row: status becomes `solved`, and the question field is cleared. It then calls a language model through OpenRouter to generate an enthusiastic RPG-style victory message.

## 1.4 Failure Path: Miss Response

If the answer does not match, the workflow returns a simple miss message along with the original question so the player can try again.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Pending Quest Retrieval

### Overview
This block receives the player’s answer and retrieves the corresponding pending quest from Google Sheets. It establishes the game context for validation by identifying the active unanswered quest for the given user.

### Nodes Involved
- Quiz Answer Form
- Find Pending Quest

### Node Details

#### 1. Quiz Answer Form
- **Type and technical role:** `n8n-nodes-base.formTrigger`  
  Entry-point node that exposes a hosted n8n form and starts the workflow when submitted.
- **Configuration choices:**
  - Form title: `AI Math RPG - Attack (Answer) Form`
  - Description instructs the user to answer the monster’s question
  - Two required fields:
    - `user_id`
    - `answer`
- **Key expressions or variables used:**
  - Outputs:
    - `{{$json.user_id}}`
    - `{{$json.answer}}`
- **Input and output connections:**
  - No input; this is the workflow trigger
  - Output goes to **Find Pending Quest**
- **Version-specific requirements:**
  - Uses `typeVersion 2.3`
  - Requires n8n Form Trigger support in the instance
- **Edge cases or potential failure types:**
  - User submits blank-like but technically non-empty values
  - Unexpected formatting in `answer` such as spaces, commas, or string/number mismatch
  - Public form exposure if authentication or sharing is not controlled externally
- **Sub-workflow reference:** None

#### 2. Find Pending Quest
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Reads rows from Google Sheets by filtering for the user’s pending quest.
- **Configuration choices:**
  - Google Sheets document ID must be manually replaced from placeholder:
    - `ENTER_YOUR_SPREADSHEET_ID_HERE`
  - Sheet name: `Quests`
  - Filter conditions:
    - `user_id` equals submitted form `user_id`
    - `status` equals `pending`
- **Key expressions or variables used:**
  - `={{ $('Quiz Answer Form').item.json.user_id }}`
- **Input and output connections:**
  - Input from **Quiz Answer Form**
  - Output to **Check Answer**
- **Version-specific requirements:**
  - Uses `typeVersion 4.7`
  - Requires valid Google Sheets OAuth2 credentials
- **Edge cases or potential failure types:**
  - Spreadsheet ID not replaced
  - Missing or invalid Google credentials
  - Sheet `Quests` not found
  - No rows returned for user with `pending` status
  - Multiple pending rows for the same user, which may create multiple downstream executions
  - Column names in Google Sheets not matching expected names:
    - `user_id`, `enemy`, `question`, `answer`, `status`, `row_number`
- **Sub-workflow reference:** None

---

## Block 2 — Answer Validation

### Overview
This block compares the submitted answer to the stored correct answer using a deterministic IF condition. It is intentionally lightweight and avoids AI-based grading for simple math answers.

### Nodes Involved
- Check Answer

### Node Details

#### 3. Check Answer
- **Type and technical role:** `n8n-nodes-base.if`  
  Branching node that decides whether the answer is correct.
- **Configuration choices:**
  - Single condition comparing:
    - Left value: form-submitted `answer`
    - Right value: row value `answer`
  - Condition combinator: `and`
  - Strict validation enabled
  - Case-sensitive comparison
- **Key expressions or variables used:**
  - Left:
    - `={{ $('Quiz Answer Form').item.json.answer }}`
  - Right:
    - `={{ $json.answer }}`
- **Input and output connections:**
  - Input from **Find Pending Quest**
  - True output to **Update Quest Status**
  - False output to **Return Miss Message**
- **Version-specific requirements:**
  - Uses `typeVersion 2.3`
- **Important implementation note:**
  - The configured operator shown is `dateTime -> equals`, even though the values being compared are answer fields. In practice this may be fragile or incorrect depending on how n8n validates and coerces the values.
- **Edge cases or potential failure types:**
  - Numeric/string mismatch, e.g. `4` vs `4.0`
  - Whitespace mismatch, e.g. `"12"` vs `" 12 "`
  - Operator type mismatch due to `dateTime` comparison being used on answer values
  - Multiple quest rows causing multiple branch evaluations
  - No incoming item if Google Sheets returns nothing
- **Sub-workflow reference:** None

---

## Block 3 — Success Path: Update Quest and Generate Victory Fanfare

### Overview
If the submitted answer is judged correct, this block updates the quest status in Google Sheets so the user cannot repeatedly farm rewards from the same quest. It then generates a celebratory RPG-style message through an LLM.

### Nodes Involved
- Update Quest Status
- OpenRouter Chat Model
- Generate Victory Message

### Node Details

#### 4. Update Quest Status
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Updates the matched quest row in Google Sheets.
- **Configuration choices:**
  - Operation: `update`
  - Spreadsheet ID placeholder must be replaced
  - Sheet name: `Quests`
  - Matching column: `row_number`
  - Values written:
    - `status` = `solved`
    - `question` = empty string
    - `row_number` = current item’s `row_number`
- **Key expressions or variables used:**
  - `={{ $json.row_number }}`
- **Input and output connections:**
  - Input from **Check Answer** true branch
  - Output to **Generate Victory Message**
- **Version-specific requirements:**
  - Uses `typeVersion 4.7`
  - Requires row-number-based update support in this Google Sheets node version
- **Edge cases or potential failure types:**
  - `row_number` missing or incorrect
  - Spreadsheet structure changed after lookup
  - OAuth token expired or insufficient permissions
  - Attempting update on a protected sheet/range
  - Clearing `question` may not be desirable if you want to preserve audit history
- **Sub-workflow reference:** None

#### 5. OpenRouter Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
  Provides the language model used by the downstream LLM Chain node.
- **Configuration choices:**
  - Provider credential: OpenRouter
  - Model: `openai/gpt-4o-mini`
  - Max tokens: `300`
- **Key expressions or variables used:**
  - No prompt here; this node supplies the model connection
- **Input and output connections:**
  - No main input connection
  - AI language model output connected to **Generate Victory Message**
- **Version-specific requirements:**
  - Uses `typeVersion 1`
  - Requires LangChain-compatible n8n AI nodes
  - Requires valid OpenRouter API credentials
- **Edge cases or potential failure types:**
  - Invalid API key
  - Model unavailable on OpenRouter
  - Token limit or provider-side rate limiting
  - Network/API timeout
  - Output variability in style or length
- **Sub-workflow reference:** None

#### 6. Generate Victory Message
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chainLlm`  
  Builds a prompt and sends it to the configured chat model to generate a victory message.
- **Configuration choices:**
  - Prompt type: manually defined
  - Prompt includes:
    - User ID from form
    - Enemy name from the pending quest row
  - Requests:
    - enthusiastic victory message
    - short compliment
    - RPG fanfare style
- **Key expressions or variables used:**
  - `{{ $('Quiz Answer Form').item.json.user_id }}`
  - `{{ $("Find Pending Quest").item.json.enemy }}`
- **Input and output connections:**
  - Main input from **Update Quest Status**
  - AI model input from **OpenRouter Chat Model**
  - Final output is the generated text
- **Version-specific requirements:**
  - Uses `typeVersion 1.7`
  - Requires a connected AI language model node
- **Edge cases or potential failure types:**
  - If `enemy` is missing in the sheet row, prompt becomes incomplete
  - Model may return text in unexpected tone or format
  - If upstream update fails, this node never runs
  - If there were multiple pending rows, message generation may occur multiple times
- **Sub-workflow reference:** None

---

## Block 4 — Failure Path: Miss Response

### Overview
If the answer does not match, this block returns a simple failure response to encourage another attempt. It also includes the original question from the quest row.

### Nodes Involved
- Return Miss Message

### Node Details

#### 7. Return Miss Message
- **Type and technical role:** `n8n-nodes-base.code`  
  Produces a custom JSON response for incorrect answers.
- **Configuration choices:**
  - Mode: `runOnceForEachItem`
  - JavaScript returns:
    - `message`: `❌ You missed! Try again!`
    - `question`: current row’s `question`
- **Key expressions or variables used:**
  - Uses `$json.question` from the incoming item
- **Input and output connections:**
  - Input from **Check Answer** false branch
  - No further output connections
- **Version-specific requirements:**
  - Uses `typeVersion 2`
  - JavaScript execution must be allowed in the n8n environment
- **Edge cases or potential failure types:**
  - If incoming item lacks `question`, output question will be undefined
  - Multiple pending rows may return multiple miss messages
  - If no pending quest was found, this node never receives input
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Quiz Answer Form | n8n-nodes-base.formTrigger | Entry form collecting `user_id` and `answer` |  | Find Pending Quest | Receive answer & find pending quest |
| Find Pending Quest | n8n-nodes-base.googleSheets | Lookup pending quest row in Google Sheets for the submitted user | Quiz Answer Form | Check Answer | Receive answer & find pending quest |
| Check Answer | n8n-nodes-base.if | Compare submitted answer with stored correct answer | Find Pending Quest | Update Quest Status; Return Miss Message | Check answer  \n(No AI needed!) |
| Update Quest Status | n8n-nodes-base.googleSheets | Mark quest as solved and clear question field | Check Answer | Generate Victory Message | Update status to 'solved' & Generate victory fanfare |
| OpenRouter Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provide chat model for LLM response generation |  | Generate Victory Message | Update status to 'solved' & Generate victory fanfare |
| Generate Victory Message | @n8n/n8n-nodes-langchain.chainLlm | Generate celebratory RPG-style success message | Update Quest Status; OpenRouter Chat Model |  | Update status to 'solved' & Generate victory fanfare |
| Return Miss Message | n8n-nodes-base.code | Return failure response with retry encouragement | Check Answer |  | Return miss message |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like:
     - `Judge AI math RPG answers and update quest status in Google Sheets`

2. **Add a Form Trigger node**
   - Node type: **Form Trigger**
   - Name: `Quiz Answer Form`
   - Set:
     - Form title: `AI Math RPG - Attack (Answer) Form`
     - Description: `Enter the answer to the monster's question to attack!`
   - Add two required fields:
     1. `user_id`
     2. `answer`
   - This will be the workflow entry point.

3. **Add a Google Sheets node to find the pending quest**
   - Node type: **Google Sheets**
   - Name: `Find Pending Quest`
   - Connect it from `Quiz Answer Form`
   - Authenticate with **Google Sheets OAuth2**
   - Select your spreadsheet document
   - Set sheet name to `Quests`
   - Configure the node to read/filter rows
   - Add filters:
     - `user_id` equals `{{$('Quiz Answer Form').item.json.user_id}}`
     - `status` equals `pending`
   - Ensure your sheet contains at least these columns:
     - `user_id`
     - `enemy`
     - `question`
     - `answer`
     - `status`
     - `row_number`

4. **Add an IF node for validation**
   - Node type: **IF**
   - Name: `Check Answer`
   - Connect it from `Find Pending Quest`
   - Configure one condition comparing:
     - Left value: `{{$('Quiz Answer Form').item.json.answer}}`
     - Right value: `{{$json.answer}}`
   - Recommended setup:
     - Use a general equality comparison appropriate for strings or numbers
   - Note: the provided workflow uses a `dateTime equals` operator, which is unusual for answer checking. For rebuilding, prefer a standard equality operator compatible with your data type.

5. **Configure the true branch for correct answers**
   - From the **true** output of `Check Answer`, add a **Google Sheets** node
   - Name: `Update Quest Status`
   - Set operation to **Update**
   - Use the same spreadsheet and `Quests` sheet
   - Match using `row_number`
   - Map fields:
     - `status` = `solved`
     - `question` = empty string
     - `row_number` = `{{$json.row_number}}`
   - This ensures only the matched row is updated.

6. **Add the chat model node**
   - Node type: **OpenRouter Chat Model**
   - Name: `OpenRouter Chat Model`
   - Create or attach **OpenRouter API** credentials
   - Set:
     - Model: `openai/gpt-4o-mini`
     - Max tokens: `300`

7. **Add an LLM Chain node for the victory response**
   - Node type: **Basic LLM Chain** / **LLM Chain**
   - Name: `Generate Victory Message`
   - Connect its main input from `Update Quest Status`
   - Connect the AI model input from `OpenRouter Chat Model`
   - Use a manually defined prompt such as:
     - `User '{{ $('Quiz Answer Form').item.json.user_id }}' successfully defeated the monster '{{ $("Find Pending Quest").item.json.enemy }}'! Generate an enthusiastic victory message and a short compliment, like an RPG fanfare.`
   - Keep the output as generated text.

8. **Configure the false branch for incorrect answers**
   - From the **false** output of `Check Answer`, add a **Code** node
   - Name: `Return Miss Message`
   - Set mode to `Run Once for Each Item`
   - Use code equivalent to:
     ```javascript
     return {
       message: "❌ You missed! Try again!",
       question: $json.question
     };
     ```
   - This returns a simple retry message and re-sends the current question.

9. **Verify all connections**
   - `Quiz Answer Form` → `Find Pending Quest`
   - `Find Pending Quest` → `Check Answer`
   - `Check Answer` true → `Update Quest Status`
   - `Update Quest Status` → `Generate Victory Message`
   - `OpenRouter Chat Model` → `Generate Victory Message` via AI language model connection
   - `Check Answer` false → `Return Miss Message`

10. **Set credentials**
    - **Google Sheets OAuth2**
      - Must allow read and update access to the target spreadsheet
    - **OpenRouter API**
      - Must allow use of `openai/gpt-4o-mini` or your chosen equivalent model

11. **Prepare the Google Sheet structure**
    - Create a spreadsheet with a sheet named `Quests`
    - Include columns:
      - `user_id`
      - `enemy`
      - `question`
      - `answer`
      - `status`
      - `row_number`
    - Ensure `row_number` is usable for row updates
    - Store only one `pending` quest per user if you want deterministic behavior

12. **Replace placeholders**
    - In both Google Sheets nodes, replace:
      - `ENTER_YOUR_SPREADSHEET_ID_HERE`
      with the real spreadsheet ID

13. **Test the workflow**
    - Open the form from `Quiz Answer Form`
    - Submit a known `user_id` and answer
    - Confirm:
      - Correct answer updates the row to `solved` and returns a victory message
      - Wrong answer returns the miss message and question

14. **Recommended hardening improvements**
    - Normalize answers before comparison:
      - trim whitespace
      - convert numeric strings if needed
    - Add a branch to handle “no pending quest found”
    - Prevent multiple pending quests per user
    - Preserve the original question instead of clearing it if audit/history matters
    - Consider returning structured success JSON around the LLM output for easier downstream automation

### Sub-workflow setup
- This workflow does **not** invoke any sub-workflow.
- However, it depends logically on an earlier system that populates the `Quests` sheet with pending quests and correct answers.
- Expected upstream data in Google Sheets:
  - one pending quest row per user
  - valid `answer`
  - valid `enemy`
  - valid `row_number`

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Judge AI math RPG answers and update quest status in Google Sheets | Workflow title |
| This template is for educators, parents, or self-learners who want to gamify their study routines. This is Part 2 of the "AI Math RPG" system. It handles the quiz judgment and status updates without using expensive AI tokens for basic math checks. | Template context |
| When a user submits their answer via an n8n Form, this workflow searches Google Sheets for their `pending` quest. It uses a fast and reliable IF node to check if the user's answer matches the correct answer generated previously. | Functional description |
| If correct, it updates the quest status to `solved` in Google Sheets to prevent infinite EXP farming, and then uses a Basic LLM Chain to generate an enthusiastic, RPG-style victory fanfare. If incorrect, it returns a friendly "try again" message. | Functional description |
| Ensure you have set up Part 1 of this system: "Generate AI math RPG quests from study logs". | Dependency on earlier workflow |
| Connect your Google Sheets credential and replace `ENTER_YOUR_SPREADSHEET_ID_HERE` with your actual Sheet ID. | Setup note |
| Connect your OpenAI or OpenRouter credential in the LLM node. | Setup note |
| Open the "Quiz Answer Form" and enter your user ID and answer to test the battle. | Setup note |
| Requirements: A Google account for Google Sheets, and an OpenAI or OpenRouter API key. | Requirements |
| You can customize the "Generate Victory Message" prompt to match different themes, like a sci-fi battle, a magic school, or historical events. | Customization note |

## Additional implementation observations
- The workflow has a single entry point: **Quiz Answer Form**.
- There are no sub-workflows.
- The workflow is currently **inactive** in the exported JSON.
- Execution order is set to `v1`.
- The most important technical risk is the answer comparison configuration in **Check Answer**, because the operator appears to be set to a datetime equality check rather than a plain string/number equality check.