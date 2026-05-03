Build an AI confidence coach for women with GPT-4o, Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/build-an-ai-confidence-coach-for-women-with-gpt-4o--google-sheets-and-gmail-14020


# Build an AI confidence coach for women with GPT-4o, Google Sheets and Gmail

# 1. Workflow Overview

This workflow implements a personalized AI confidence and career coaching system for women using n8n, GPT-4o, Google Sheets, Gmail, and the n8n Chat Trigger.

It has two main entry points:

- **Chat entry point**: handles live user conversations, onboarding, profile persistence, intent-aware coaching, and response logging.
- **Scheduled entry point**: runs weekly to generate and email personalized check-ins to onboarded users.

The workflow is designed for these use cases:

- onboarding new coaching users through chat
- maintaining a lightweight user profile in Google Sheets
- generating personalized coaching replies based on message intent and recent conversation history
- storing conversation memory for continuity
- sending weekly motivational follow-up emails

## 1.1 Chat Intake and User Lookup

The workflow starts when a user sends a chat message. It extracts the session ID and message text, then loads all user profiles from Google Sheets to determine whether the sender is new, mid-onboarding, or fully onboarded.

## 1.2 Onboarding Flow

If the user has not yet completed onboarding, the workflow guides them through a 4-step setup:

1. name
2. career stage
3. biggest challenge
4. monthly goal

Each step updates the **User Profiles** sheet and returns the next onboarding question.

## 1.3 Coaching Flow with Memory and Intent Detection

If onboarding is complete, the workflow loads recent conversation history, detects the likely coaching topic from the latest message, builds a structured GPT-4o prompt, and asks the model to return a JSON response containing:

- personalized coaching advice
- one concrete action step
- one follow-up question
- detected intent

The response is parsed, logged, and returned to the user via chat.

## 1.4 Weekly Scheduled Check-ins

Every week at the scheduled time, the workflow reads all onboarded users, checks their recent conversation topics, asks GPT-4o to generate a personalized weekly email in JSON, transforms that into styled HTML, filters out users without email addresses, sends the email through Gmail, and logs the send event.

---

# 2. Block-by-Block Analysis

## 2.1 Block: Chat Trigger and Input Normalization

### Overview
This block receives user messages from the n8n chat interface and converts the incoming payload into a simpler structure used throughout the rest of the workflow. It standardizes the user ID, message text, and timestamp.

### Nodes Involved
- Chat Trigger
- Extract User & Message

### Node Details

#### Chat Trigger
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chatTrigger`; public chat entry point for interactive chat sessions.
- **Configuration choices:**
  - Public chat is enabled.
  - Response mode is set to **responseNodes**, meaning reply nodes in the workflow provide the chat output.
  - Initial welcome message asks the user to start by entering their name.
- **Key expressions or variables used:** none in parameters.
- **Input and output connections:**
  - Entry point node.
  - Outputs to **Extract User & Message**.
- **Version-specific requirements:** typeVersion `1.4`; requires the Chat Trigger feature available in current n8n versions that support LangChain chat nodes.
- **Edge cases or potential failure types:**
  - Public chat exposure if the URL is shared broadly.
  - Session handling depends on the trigger payload containing a valid `sessionId`.
  - If `chatInput` is empty, downstream logic may treat the user message as blank.
- **Sub-workflow reference:** none.

#### Extract User & Message
- **Type and technical role:** `n8n-nodes-base.code`; reshapes incoming chat payload into normalized fields.
- **Configuration choices:**
  - Reads `chatInput` from the first incoming item.
  - Reads `sessionId`, defaulting to `anonymous` if missing.
  - Trims the message and adds an ISO timestamp.
  - Outputs:
    - `user_id`
    - `message`
    - `timestamp`
- **Key expressions or variables used:**
  - `$input.first().json.chatInput`
  - `$input.first().json.sessionId`
- **Input and output connections:**
  - Input from **Chat Trigger**
  - Output to **Read All Users**
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Blank or whitespace-only messages pass through as empty strings.
  - Missing session ID leads to all unidentified users sharing `anonymous`, which can cause profile collisions.
- **Sub-workflow reference:** none.

---

## 2.2 Block: User Profile Lookup and Routing

### Overview
This block loads all stored user profiles, matches the current chat user by `user_id`, and decides whether the user is new, still onboarding, or ready for coaching. It also builds the correct next response for onboarding steps.

### Nodes Involved
- Read All Users
- Check User & Route
- IF Onboarding or Coaching

### Node Details

#### Read All Users
- **Type and technical role:** `n8n-nodes-base.googleSheets`; fetches all records from the **User Profiles** sheet.
- **Configuration choices:**
  - Reads from spreadsheet ID `1yE5kqA1bo52CrYcvKFHFuCTBYmcI1vaMPuFKKP28sEI`
  - Sheet: `User Profiles` (`gid=0`)
  - `alwaysOutputData` enabled so downstream logic still runs even if no rows exist.
- **Key expressions or variables used:** none.
- **Input and output connections:**
  - Input from **Extract User & Message**
  - Output to **Check User & Route**
- **Version-specific requirements:** Google Sheets node version `4.7`.
- **Edge cases or potential failure types:**
  - OAuth credential errors
  - Missing or renamed sheet
  - Empty sheet is tolerated because `alwaysOutputData` is true
- **Sub-workflow reference:** none.

#### Check User & Route
- **Type and technical role:** `n8n-nodes-base.code`; central routing and onboarding state machine.
- **Configuration choices:**
  - Reads current `user_id` and `message` from **Extract User & Message**
  - Uses all rows returned from **Read All Users**
  - Finds a user where sheet `user_id` equals incoming `user_id`
  - Handles these cases:
    - no existing user → create step 0 welcome response
    - step 1 → save name and ask for career stage
    - step 2 → map numeric career stage answers and ask for biggest challenge
    - step 3 → map numeric challenge answers and ask for monthly goal
    - step 4 → finalize onboarding and ask first coaching question
    - step 5+ → mark onboarding complete and pass user profile to coaching branch
  - Uses lookup maps for numbered onboarding answers.
- **Key expressions or variables used:**
  - `$('Extract User & Message').first().json.user_id`
  - `$('Extract User & Message').first().json.message`
  - `$input.all()`
- **Input and output connections:**
  - Input from **Read All Users**
  - Output to **IF Onboarding or Coaching**
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - If `onboarding_step` in the sheet is malformed, `Number(user.onboarding_step) || 0` may reset behavior unexpectedly.
  - Numeric shortcuts only support mapped options; custom text is accepted as-is.
  - New users rely entirely on `sessionId` continuity from the chat trigger.
  - A blank message during onboarding would be stored as name/challenge/goal unless separately validated.
- **Sub-workflow reference:** none.

#### IF Onboarding or Coaching
- **Type and technical role:** `n8n-nodes-base.if`; decides whether to continue onboarding or switch into the coaching branch.
- **Configuration choices:**
  - Condition: `onboarding_step < 5`
  - True branch = onboarding
  - False branch = coaching
- **Key expressions or variables used:**
  - `={{ $json.onboarding_step }}`
- **Input and output connections:**
  - Input from **Check User & Route**
  - True output to **Save Onboarding Progress**
  - False output to **Read Conversation History**
- **Version-specific requirements:** IF node version `2.3`.
- **Edge cases or potential failure types:**
  - If `onboarding_step` is a non-numeric string, strict number comparison can fail or route incorrectly.
- **Sub-workflow reference:** none.

---

## 2.3 Block: Onboarding Persistence and Chat Response

### Overview
This block saves onboarding progress into Google Sheets and returns the next onboarding prompt to the chat user. It is executed for all users whose onboarding step is below 5.

### Nodes Involved
- Save Onboarding Progress
- Respond — Onboarding

### Node Details

#### Save Onboarding Progress
- **Type and technical role:** `n8n-nodes-base.googleSheets`; upserts user profile records during onboarding.
- **Configuration choices:**
  - Operation: **appendOrUpdate**
  - Matching column: `user_id`
  - Writes to **User Profiles**
  - Stores:
    - `user_id`
    - `name`
    - `career_stage`
    - `challenge`
    - `goal`
    - `joined_date` only for new users
    - `last_active` as current date
    - `onboarding_step` as `Number($json.onboarding_step) + 1`
- **Key expressions or variables used:**
  - `={{ $json.user_id }}`
  - `={{ $json.name || '' }}`
  - `={{ $json.career_stage || '' }}`
  - `={{ $json.challenge || '' }}`
  - `={{ $json.goal || '' }}`
  - `={{ $json.is_new ? new Date().toISOString().split('T')[0] : '' }}`
  - `={{ new Date().toISOString().split('T')[0] }}`
  - `={{ Number($json.onboarding_step) + 1 }}`
- **Input and output connections:**
  - Input from **IF Onboarding or Coaching** true branch
  - Output to **Respond — Onboarding**
- **Version-specific requirements:** Google Sheets node version `4.7`.
- **Edge cases or potential failure types:**
  - If the Google Sheet lacks expected columns, write operations fail.
  - `joined_date` expression contains trailing whitespace in the JSON config; usually harmless, but worth cleaning.
  - No email field is captured here, although the weekly email flow expects one.
- **Sub-workflow reference:** none.

#### Respond — Onboarding
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chat`; returns the onboarding text to the user chat.
- **Configuration choices:**
  - Message comes from `$('Check User & Route').item.json.response`
  - `waitUserReply` is disabled
- **Key expressions or variables used:**
  - `={{ $('Check User & Route').item.json.response }}`
- **Input and output connections:**
  - Input from **Save Onboarding Progress**
  - No downstream output in this workflow
- **Version-specific requirements:** typeVersion `1`.
- **Edge cases or potential failure types:**
  - If `response` is null or undefined, the chat output may be empty.
- **Sub-workflow reference:** none.

---

## 2.4 Block: Conversation Memory, Intent Detection, and AI Coaching

### Overview
Once onboarding is complete, this block loads recent conversation history, infers the user's intent from keyword matching, builds a detailed structured prompt, sends it to GPT-4o, and parses the model’s JSON response into a user-friendly coaching reply.

### Nodes Involved
- Read Conversation History
- Build Coaching Prompt
- Confidence Coach
- Parse Coaching Response

### Node Details

#### Read Conversation History
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads prior conversation records for the current user.
- **Configuration choices:**
  - Reads from **Conversation Log**
  - Uses filter:
    - `lookupColumn = user_id`
    - `lookupValue = {{ $json.user_id }}`
  - `alwaysOutputData` is enabled
- **Key expressions or variables used:**
  - `={{ $json.user_id }}`
- **Input and output connections:**
  - Input from **IF Onboarding or Coaching** false branch
  - Output to **Build Coaching Prompt**
- **Version-specific requirements:** Google Sheets node version `4.7`.
- **Edge cases or potential failure types:**
  - If no conversation exists, the code node handles first-message behavior.
  - Filter accuracy depends on exact string matching of user IDs.
- **Sub-workflow reference:** none.

#### Build Coaching Prompt
- **Type and technical role:** `n8n-nodes-base.code`; composes the AI coaching prompt with memory and intent.
- **Configuration choices:**
  - Reads user profile data from **Check User & Route**
  - Takes up to the last 5 conversation rows from the current input
  - Builds a plain text conversation transcript from valid `user_message` and `bot_response`
  - Detects intent via keyword lists across categories:
    - salary
    - interview
    - career_change
    - leadership
    - confidence
    - balance
    - default general
  - Selects a topic-specific system instruction
  - Asks GPT-4o to respond in strict JSON with:
    - `main_advice`
    - `actionable_step`
    - `follow_up_question`
    - `intent_detected`
- **Key expressions or variables used:**
  - `$('Check User & Route').first().json`
  - `$input.all().slice(-5)`
  - `message.toLowerCase()`
- **Input and output connections:**
  - Input from **Read Conversation History**
  - Output to **Confidence Coach**
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Intent detection is keyword-based and may misclassify ambiguous messages.
  - Only the first matching intent wins.
  - If message is empty, `toLowerCase()` still works on an empty string, but output quality drops.
  - Prompt assumes user profile fields exist; blank fields reduce personalization.
- **Sub-workflow reference:** none.

#### Confidence Coach
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; calls OpenAI GPT-4o to generate the coaching payload.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Sends one response instruction containing the constructed prompt
  - No built-in tools enabled
- **Key expressions or variables used:**
  - `={{ $json.prompt }}`
- **Input and output connections:**
  - Input from **Build Coaching Prompt**
  - Output to **Parse Coaching Response**
- **Version-specific requirements:** typeVersion `2.1`; requires valid OpenAI credentials and model access.
- **Edge cases or potential failure types:**
  - Auth failures or quota/rate limit issues
  - Model may return malformed JSON despite prompt instructions
  - Long prompts could hit token limits if history or profile grows substantially
- **Sub-workflow reference:** none.

#### Parse Coaching Response
- **Type and technical role:** `n8n-nodes-base.code`; parses model output and formats final chat text.
- **Configuration choices:**
  - Reads model text from `items[0].json.output[0].content[0].text`
  - Removes optional markdown code fences
  - Parses JSON
  - Builds a chat response combining:
    - main advice
    - actionable step
    - follow-up question
  - Returns both formatted and raw structured fields
- **Key expressions or variables used:**
  - `items[0].json.output[0].content[0].text`
  - `JSON.parse(cleaned)`
  - `$('Check User & Route').first().json`
- **Input and output connections:**
  - Input from **Confidence Coach**
  - Output to **Log to Conversation Log**
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Most fragile point in the coaching branch:
    - unexpected output path from OpenAI node
    - invalid JSON
    - empty output
  - No try/catch is implemented, so malformed model output will fail the execution.
- **Sub-workflow reference:** none.

---

## 2.5 Block: Conversation Logging and Chat Response Delivery

### Overview
This block stores the coaching interaction in Google Sheets and sends the formatted coaching reply back to the chat user.

### Nodes Involved
- Log to Conversation Log
- Respond — Coaching

### Node Details

#### Log to Conversation Log
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends a persistent record of each coaching interaction.
- **Configuration choices:**
  - Operation: **append**
  - Writes to **Conversation Log**
  - Stores:
    - `user_id`
    - `timestamp`
    - `user_message`
    - `intent`
    - `bot_response`
  - `bot_response` stores only `main_advice`, not the full formatted message
- **Key expressions or variables used:**
  - `={{ $json.user_id }}`
  - `={{ new Date().toISOString() }}`
  - `={{ $json.message }}`
  - `={{ $json.intent }}`
  - `={{ $json.main_advice }}`
- **Input and output connections:**
  - Input from **Parse Coaching Response**
  - Output to **Respond — Coaching**
- **Version-specific requirements:** Google Sheets node version `4.7`.
- **Edge cases or potential failure types:**
  - If append fails, the user does not receive the response because the chat reply is downstream.
  - Storing only `main_advice` means the action step and question are not preserved in memory.
- **Sub-workflow reference:** none.

#### Respond — Coaching
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chat`; sends the final coaching response back to the user.
- **Configuration choices:**
  - Message uses `$('Parse Coaching Response').item.json.coaching_response`
  - `waitUserReply` disabled
- **Key expressions or variables used:**
  - `={{ $('Parse Coaching Response').item.json.coaching_response }}`
- **Input and output connections:**
  - Input from **Log to Conversation Log**
  - No downstream output in this workflow
- **Version-specific requirements:** typeVersion `1`.
- **Edge cases or potential failure types:**
  - Empty response if the upstream parsing node fails or returns incomplete data.
- **Sub-workflow reference:** none.

---

## 2.6 Block: Weekly Trigger and User Filtering

### Overview
This block starts the weekly outreach process, loads all user profiles, and filters the dataset so only fully onboarded users proceed to email generation.

### Nodes Involved
- Weekly Sunday 10AM Trigger
- Read All Onboarded Users
- Filter Onboarded Users

### Node Details

#### Weekly Sunday 10AM Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; scheduled entry point for weekly check-ins.
- **Configuration choices:**
  - Interval rule configured with weekly cadence and trigger hour 10.
- **Key expressions or variables used:** none.
- **Input and output connections:**
  - Entry point node
  - Outputs to **Read All Onboarded Users**
- **Version-specific requirements:** Schedule Trigger version `1.3`.
- **Edge cases or potential failure types:**
  - The exact timezone depends on the n8n instance timezone settings.
  - Despite the node name, the JSON config does not explicitly show Sunday in a human-readable way; validate schedule after import.
- **Sub-workflow reference:** none.

#### Read All Onboarded Users
- **Type and technical role:** `n8n-nodes-base.googleSheets`; loads user profiles for batch weekly processing.
- **Configuration choices:**
  - Reads the same **User Profiles** sheet
  - `executeOnce` enabled
  - `alwaysOutputData` disabled
- **Key expressions or variables used:** none.
- **Input and output connections:**
  - Input from **Weekly Sunday 10AM Trigger**
  - Output to **Filter Onboarded Users**
- **Version-specific requirements:** Google Sheets node version `4.7`.
- **Edge cases or potential failure types:**
  - If the sheet is empty, the branch stops naturally.
  - OAuth or document access issues stop all weekly sends.
- **Sub-workflow reference:** none.

#### Filter Onboarded Users
- **Type and technical role:** `n8n-nodes-base.filter`; keeps only profiles whose onboarding is complete.
- **Configuration choices:**
  - Condition: `onboarding_step >= 5`
- **Key expressions or variables used:**
  - `={{ $json.onboarding_step }}`
- **Input and output connections:**
  - Input from **Read All Onboarded Users**
  - Output to **Read Conversation Log**
- **Version-specific requirements:** Filter node version `2.3`.
- **Edge cases or potential failure types:**
  - If `onboarding_step` is stored as a non-numeric string, strict numeric comparison can exclude valid users.
- **Sub-workflow reference:** none.

---

## 2.7 Block: Weekly Prompt Generation with Recent Topics

### Overview
This block collects recent conversation intents for each onboarded user, builds a personalized weekly email prompt, calls GPT-4o, and parses the model output into an HTML email.

### Nodes Involved
- Read Conversation Log
- Build Checkin Prompt
- Weekly Coach
- Parse Weekly Message

### Node Details

#### Read Conversation Log
- **Type and technical role:** `n8n-nodes-base.googleSheets`; loads the full conversation log for use in weekly summarization.
- **Configuration choices:**
  - Reads from **Conversation Log**
  - `executeOnce` enabled
- **Key expressions or variables used:** none.
- **Input and output connections:**
  - Input from **Filter Onboarded Users**
  - Output to **Build Checkin Prompt**
- **Version-specific requirements:** Google Sheets node version `4.7`.
- **Edge cases or potential failure types:**
  - Large conversation logs may become slow or memory-heavy because the next code node processes them in memory.
- **Sub-workflow reference:** none.

#### Build Checkin Prompt
- **Type and technical role:** `n8n-nodes-base.code`; generates one personalized weekly prompt per onboarded user.
- **Configuration choices:**
  - Reads all users from **Filter Onboarded Users**
  - Reads all conversation records from **Read Conversation Log**
  - For each user:
    - filters conversation log by `user_id`
    - keeps the last 3 matching records
    - derives `lastTopics` from stored intents, defaulting to `general career growth`
    - builds a prompt requesting strict JSON:
      - `subject`
      - `greeting`
      - `progress_acknowledgment`
      - `weekly_challenge`
      - `motivational_quote`
      - `closing`
  - Exposes `email` from the user row if present
- **Key expressions or variables used:**
  - `$('Filter Onboarded Users').all()`
  - `$('Read Conversation Log').all()`
  - `String(c.json.user_id) === String(u.user_id)`
- **Input and output connections:**
  - Input from **Read Conversation Log**
  - Output to **Weekly Coach**
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - User profiles do not appear to collect email during onboarding, so many users may have empty email values unless the sheet is manually enriched.
  - Last 3 conversations are determined by row order, not explicit timestamp sorting.
  - Recent topics are intent labels only, not full message summaries.
- **Sub-workflow reference:** none.

#### Weekly Coach
- **Type and technical role:** `@n8n/n8n-nodes-langchain.openAi`; asks GPT-4o to generate structured weekly email content.
- **Configuration choices:**
  - Model: `gpt-4o`
  - Sends one prompt from `{{$json.prompt}}`
- **Key expressions or variables used:**
  - `={{ $json.prompt }}`
- **Input and output connections:**
  - Input from **Build Checkin Prompt**
  - Output to **Parse Weekly Message**
- **Version-specific requirements:** typeVersion `2.1`.
- **Edge cases or potential failure types:**
  - Same model-output risks as the coaching branch:
    - malformed JSON
    - auth/rate-limit failures
    - changed output schema
- **Sub-workflow reference:** none.

#### Parse Weekly Message
- **Type and technical role:** `n8n-nodes-base.code`; converts the AI JSON into final email subject and styled HTML body.
- **Configuration choices:**
  - Reads model text from `items[0].json.output[0].content[0].text`
  - Removes markdown code fences
  - Parses JSON
  - Reads user data from `$('Build Checkin Prompt').item.json`
  - Builds a complete HTML email with purple branding, challenge box, quote box, and footer
- **Key expressions or variables used:**
  - `items[0].json.output[0].content[0].text`
  - `$('Build Checkin Prompt').item.json`
- **Input and output connections:**
  - Input from **Weekly Coach**
  - Output to **Filter Users With Email**
- **Version-specific requirements:** Code node version `2`.
- **Edge cases or potential failure types:**
  - Invalid model JSON causes hard failure.
  - HTML escaping is not applied to model-generated text; unusual characters or injected HTML could affect rendering.
- **Sub-workflow reference:** none.

---

## 2.8 Block: Email Eligibility, Delivery, and Logging

### Overview
This block filters generated weekly messages to users with an email address, sends the HTML email through Gmail, and records the send in Google Sheets.

### Nodes Involved
- Filter Users With Email
- Send Weekly Checkin Email
- Log to Weekly Checkins

### Node Details

#### Filter Users With Email
- **Type and technical role:** `n8n-nodes-base.filter`; ensures only users with a non-empty email proceed to sending.
- **Configuration choices:**
  - Condition: `email` is not empty
- **Key expressions or variables used:**
  - `={{ $json.email }}`
- **Input and output connections:**
  - Input from **Parse Weekly Message**
  - Output to **Send Weekly Checkin Email**
- **Version-specific requirements:** Filter node version `2.3`.
- **Edge cases or potential failure types:**
  - Does not validate email format, only non-empty value.
- **Sub-workflow reference:** none.

#### Send Weekly Checkin Email
- **Type and technical role:** `n8n-nodes-base.gmail`; sends the personalized HTML email.
- **Configuration choices:**
  - Recipient: `{{$json.email}}`
  - Subject: `{{$json.email_subject}}`
  - Message body: `{{$json.email_body}}`
- **Key expressions or variables used:**
  - `={{ $json.email }}`
  - `={{ $json.email_subject }}`
  - `={{ $json.email_body }}`
- **Input and output connections:**
  - Input from **Filter Users With Email**
  - Output to **Log to Weekly Checkins**
- **Version-specific requirements:** Gmail node version `2.2`; requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Gmail OAuth expiration or revoked consent
  - Daily sending limits
  - HTML may be sent as plain content depending on Gmail node behavior/settings in the instance; test formatting after import
- **Sub-workflow reference:** none.

#### Log to Weekly Checkins
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends a weekly send log entry.
- **Configuration choices:**
  - Operation: **append**
  - Writes to **Weekly Checkins**
  - Stores:
    - `user_id`
    - `name`
    - `email`
    - `date`
    - `message_sent` = email subject
  - Pulls some values from **Filter Users With Email** via node reference expressions
- **Key expressions or variables used:**
  - `={{ new Date().toISOString().split('T')[0] }}`
  - `={{ $('Filter Users With Email').item.json.name }}`
  - `={{ $('Filter Users With Email').item.json.email }}`
  - `={{ $('Filter Users With Email').item.json.user_id }}`
  - `={{ $('Filter Users With Email').item.json.email_subject }}`
- **Input and output connections:**
  - Input from **Send Weekly Checkin Email**
  - No downstream output
- **Version-specific requirements:** Google Sheets node version `4.7`.
- **Edge cases or potential failure types:**
  - If send succeeds but logging fails, email still goes out but no audit row is written.
- **Sub-workflow reference:** none.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Chat Trigger | @n8n/n8n-nodes-langchain.chatTrigger | Public chat entry point for live coaching conversations |  | Extract User & Message | Runs on every chat message. Reads all users from Google Sheets and detects if the user is new or returning. New users go through a 4-step onboarding collecting name, career stage, challenge, and goal. All responses saved and updated in User Profiles. |
| Extract User & Message | n8n-nodes-base.code | Normalizes chat payload into user_id, message, and timestamp | Chat Trigger | Read All Users | Runs on every chat message. Reads all users from Google Sheets and detects if the user is new or returning. New users go through a 4-step onboarding collecting name, career stage, challenge, and goal. All responses saved and updated in User Profiles. |
| Read All Users | n8n-nodes-base.googleSheets | Loads user profiles from Google Sheets | Extract User & Message | Check User & Route | Runs on every chat message. Reads all users from Google Sheets and detects if the user is new or returning. New users go through a 4-step onboarding collecting name, career stage, challenge, and goal. All responses saved and updated in User Profiles. |
| Check User & Route | n8n-nodes-base.code | Determines onboarding state and routes to onboarding or coaching | Read All Users | IF Onboarding or Coaching | Runs on every chat message. Reads all users from Google Sheets and detects if the user is new or returning. New users go through a 4-step onboarding collecting name, career stage, challenge, and goal. All responses saved and updated in User Profiles. |
| IF Onboarding or Coaching | n8n-nodes-base.if | Branches between onboarding continuation and AI coaching | Check User & Route | Save Onboarding Progress; Read Conversation History | Runs on every chat message. Reads all users from Google Sheets and detects if the user is new or returning. New users go through a 4-step onboarding collecting name, career stage, challenge, and goal. All responses saved and updated in User Profiles. |
| Save Onboarding Progress | n8n-nodes-base.googleSheets | Upserts onboarding/profile progress to User Profiles sheet | IF Onboarding or Coaching | Respond — Onboarding | Runs on every chat message. Reads all users from Google Sheets and detects if the user is new or returning. New users go through a 4-step onboarding collecting name, career stage, challenge, and goal. All responses saved and updated in User Profiles. |
| Respond — Onboarding | @n8n/n8n-nodes-langchain.chat | Sends onboarding prompt back to the user | Save Onboarding Progress |  | Runs on every chat message. Reads all users from Google Sheets and detects if the user is new or returning. New users go through a 4-step onboarding collecting name, career stage, challenge, and goal. All responses saved and updated in User Profiles. |
| Read Conversation History | n8n-nodes-base.googleSheets | Retrieves prior conversations for the current user | IF Onboarding or Coaching | Build Coaching Prompt | Triggers when onboarding is complete (step 5+). Detects intent from the user's message across 6 categories — salary, interview, career change, leadership, confidence, and balance. Loads last 5 conversations for context, sends to GPT-4o with a topic-specific prompt, and responds with advice, an action step, and a follow-up question. |
| Build Coaching Prompt | n8n-nodes-base.code | Builds personalized AI prompt with intent and memory | Read Conversation History | Confidence Coach | Triggers when onboarding is complete (step 5+). Detects intent from the user's message across 6 categories — salary, interview, career change, leadership, confidence, and balance. Loads last 5 conversations for context, sends to GPT-4o with a topic-specific prompt, and responds with advice, an action step, and a follow-up question. |
| Confidence Coach | @n8n/n8n-nodes-langchain.openAi | Calls GPT-4o for structured coaching output | Build Coaching Prompt | Parse Coaching Response | Triggers when onboarding is complete (step 5+). Detects intent from the user's message across 6 categories — salary, interview, career change, leadership, confidence, and balance. Loads last 5 conversations for context, sends to GPT-4o with a topic-specific prompt, and responds with advice, an action step, and a follow-up question. |
| Parse Coaching Response | n8n-nodes-base.code | Parses AI JSON and formats final coaching reply | Confidence Coach | Log to Conversation Log | Triggers when onboarding is complete (step 5+). Detects intent from the user's message across 6 categories — salary, interview, career change, leadership, confidence, and balance. Loads last 5 conversations for context, sends to GPT-4o with a topic-specific prompt, and responds with advice, an action step, and a follow-up question. |
| Log to Conversation Log | n8n-nodes-base.googleSheets | Appends coaching interaction to conversation memory sheet | Parse Coaching Response | Respond — Coaching | Triggers when onboarding is complete (step 5+). Detects intent from the user's message across 6 categories — salary, interview, career change, leadership, confidence, and balance. Loads last 5 conversations for context, sends to GPT-4o with a topic-specific prompt, and responds with advice, an action step, and a follow-up question. |
| Respond — Coaching | @n8n/n8n-nodes-langchain.chat | Sends final coaching answer to the user | Log to Conversation Log |  | Triggers when onboarding is complete (step 5+). Detects intent from the user's message across 6 categories — salary, interview, career change, leadership, confidence, and balance. Loads last 5 conversations for context, sends to GPT-4o with a topic-specific prompt, and responds with advice, an action step, and a follow-up question. |
| Weekly Sunday 10AM Trigger | n8n-nodes-base.scheduleTrigger | Starts the weekly email batch |  | Read All Onboarded Users | Runs every Sunday at 10AM. Reads all fully onboarded users, pulls their recent conversation topics, and sends each one a personalized HTML email with a weekly challenge, progress acknowledgment, and a motivational quote from a woman leader in their field. Every send is logged to the Weekly Checkins sheet. |
| Read All Onboarded Users | n8n-nodes-base.googleSheets | Loads all user profiles for weekly processing | Weekly Sunday 10AM Trigger | Filter Onboarded Users | Runs every Sunday at 10AM. Reads all fully onboarded users, pulls their recent conversation topics, and sends each one a personalized HTML email with a weekly challenge, progress acknowledgment, and a motivational quote from a woman leader in their field. Every send is logged to the Weekly Checkins sheet. |
| Filter Onboarded Users | n8n-nodes-base.filter | Keeps only completed profiles for weekly check-ins | Read All Onboarded Users | Read Conversation Log | Runs every Sunday at 10AM. Reads all fully onboarded users, pulls their recent conversation topics, and sends each one a personalized HTML email with a weekly challenge, progress acknowledgment, and a motivational quote from a woman leader in their field. Every send is logged to the Weekly Checkins sheet. |
| Read Conversation Log | n8n-nodes-base.googleSheets | Loads conversation history for weekly topic summarization | Filter Onboarded Users | Build Checkin Prompt | Runs every Sunday at 10AM. Reads all fully onboarded users, pulls their recent conversation topics, and sends each one a personalized HTML email with a weekly challenge, progress acknowledgment, and a motivational quote from a woman leader in their field. Every send is logged to the Weekly Checkins sheet. |
| Build Checkin Prompt | n8n-nodes-base.code | Builds one weekly AI prompt per eligible user | Read Conversation Log | Weekly Coach | Runs every Sunday at 10AM. Reads all fully onboarded users, pulls their recent conversation topics, and sends each one a personalized HTML email with a weekly challenge, progress acknowledgment, and a motivational quote from a woman leader in their field. Every send is logged to the Weekly Checkins sheet. |
| Weekly Coach | @n8n/n8n-nodes-langchain.openAi | Calls GPT-4o to generate structured weekly email content | Build Checkin Prompt | Parse Weekly Message | Runs every Sunday at 10AM. Reads all fully onboarded users, pulls their recent conversation topics, and sends each one a personalized HTML email with a weekly challenge, progress acknowledgment, and a motivational quote from a woman leader in their field. Every send is logged to the Weekly Checkins sheet. |
| Parse Weekly Message | n8n-nodes-base.code | Parses AI JSON and builds styled HTML email | Weekly Coach | Filter Users With Email | Runs every Sunday at 10AM. Reads all fully onboarded users, pulls their recent conversation topics, and sends each one a personalized HTML email with a weekly challenge, progress acknowledgment, and a motivational quote from a woman leader in their field. Every send is logged to the Weekly Checkins sheet. |
| Filter Users With Email | n8n-nodes-base.filter | Keeps only users with non-empty email before sending | Parse Weekly Message | Send Weekly Checkin Email | Runs every Sunday at 10AM. Reads all fully onboarded users, pulls their recent conversation topics, and sends each one a personalized HTML email with a weekly challenge, progress acknowledgment, and a motivational quote from a woman leader in their field. Every send is logged to the Weekly Checkins sheet. |
| Send Weekly Checkin Email | n8n-nodes-base.gmail | Sends personalized weekly HTML email through Gmail | Filter Users With Email | Log to Weekly Checkins | Runs every Sunday at 10AM. Reads all fully onboarded users, pulls their recent conversation topics, and sends each one a personalized HTML email with a weekly challenge, progress acknowledgment, and a motivational quote from a woman leader in their field. Every send is logged to the Weekly Checkins sheet. |
| Log to Weekly Checkins | n8n-nodes-base.googleSheets | Logs sent weekly emails to audit sheet | Send Weekly Checkin Email |  | Runs every Sunday at 10AM. Reads all fully onboarded users, pulls their recent conversation topics, and sends each one a personalized HTML email with a weekly challenge, progress acknowledgment, and a motivational quote from a woman leader in their field. Every send is logged to the Weekly Checkins sheet. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note on overall workflow |  |  | ## Workflow Overview Every woman deserves a career coach but not everyone can afford one. This workflow gives every user a personal AI coach available 24/7 — trained to help with salary negotiation, interview prep, career changes, leadership challenges, and confidence building.  ### HOW IT WORKS Users open the chat and complete a quick 4-step onboarding — name, career stage, biggest challenge, and monthly goal. From that point every response is personalized to their profile. GPT-4o detects the intent behind every message and routes it to a topic-specific coaching prompt. Conversation history is stored so every session builds on the last. Every Sunday users get a personalized check-in email with a weekly challenge and motivational quote.  ### SETUP STEPS 1. Create the Google Sheet with 3 sheets — User Profiles, Conversation Log, Weekly Checkins 2. Connect Google Sheets, OpenAI, and Gmail credentials 3. Activate the Chat Trigger and copy the chat URL 4. Share the chat URL with your users |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note for chat/onboarding branch |  |  | Runs on every chat message. Reads all users from Google Sheets and detects if the user is new or returning. New users go through a 4-step onboarding collecting name, career stage, challenge, and goal. All responses saved and updated in User Profiles. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note for coaching branch |  |  | Triggers when onboarding is complete (step 5+). Detects intent from the user's message across 6 categories — salary, interview, career change, leadership, confidence, and balance. Loads last 5 conversations for context, sends to GPT-4o with a topic-specific prompt, and responds with advice, an action step, and a follow-up question. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note for weekly email branch |  |  | Runs every Sunday at 10AM. Reads all fully onboarded users, pulls their recent conversation topics, and sends each one a personalized HTML email with a weekly challenge, progress acknowledgment, and a motivational quote from a woman leader in their field. Every send is logged to the Weekly Checkins sheet. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:
   - `Personalized AI Confidence Coach for Women With Memory, Intent Detection, and Weekly Check-ins`

2. **Prepare Google Sheets**
   - Create one spreadsheet with three tabs:
     - `User Profiles`
     - `Conversation Log`
     - `Weekly Checkins`
   - Recommended columns:

   **User Profiles**
   - `user_id`
   - `name`
   - `career_stage`
   - `challenge`
   - `goal`
   - `email`
   - `joined_date`
   - `last_active`
   - `onboarding_step`

   **Conversation Log**
   - `user_id`
   - `timestamp`
   - `user_message`
   - `intent`
   - `bot_response`

   **Weekly Checkins**
   - `user_id`
   - `name`
   - `email`
   - `date`
   - `message_sent`

3. **Create credentials**
   - Add **Google Sheets OAuth2** credentials with access to the spreadsheet.
   - Add **OpenAI API** credentials with access to `gpt-4o`.
   - Add **Gmail OAuth2** credentials for the sender mailbox.
   - Test each credential before continuing.

4. **Create the chat entry node**
   - Add a **Chat Trigger** node.
   - Set:
     - `Public` = true
     - `Response Mode` = `responseNodes`
     - `Initial Messages` = welcome text asking for the user’s name
   - Keep note of the generated chat URL for later sharing.

5. **Add `Extract User & Message`**
   - Add a **Code** node after the Chat Trigger.
   - Paste logic that:
     - reads `chatInput`
     - reads `sessionId`
     - outputs `user_id`, `message`, `timestamp`
   - Connect:
     - `Chat Trigger -> Extract User & Message`

6. **Add `Read All Users`**
   - Add a **Google Sheets** node.
   - Configure it to read all rows from `User Profiles`.
   - Enable `Always Output Data`.
   - Connect:
     - `Extract User & Message -> Read All Users`

7. **Add `Check User & Route`**
   - Add a **Code** node.
   - Implement onboarding-state logic:
     - find existing user by `user_id`
     - if not found, create onboarding step 0 welcome response
     - if step 1, save name and ask career stage
     - if step 2, save career stage and ask challenge
     - if step 3, save challenge and ask goal
     - if step 4, save goal and mark onboarding complete
     - if step 5+, output profile and no direct response
   - Include numeric lookup maps for:
     - career stage
     - challenge
   - Connect:
     - `Read All Users -> Check User & Route`

8. **Add `IF Onboarding or Coaching`**
   - Add an **IF** node.
   - Condition:
     - numeric comparison
     - `onboarding_step < 5`
   - Connect:
     - `Check User & Route -> IF Onboarding or Coaching`

9. **Build the onboarding branch**
   - Add a **Google Sheets** node named `Save Onboarding Progress`.
   - Use operation **Append or Update**.
   - Match on `user_id`.
   - Map:
     - `user_id`
     - `name`
     - `career_stage`
     - `challenge`
     - `goal`
     - `joined_date` only if new
     - `last_active` = current date
     - `onboarding_step` = current step + 1
   - Connect:
     - `IF Onboarding or Coaching (true) -> Save Onboarding Progress`

10. **Add `Respond — Onboarding`**
    - Add a **Chat** node.
    - Set its message to the response generated in `Check User & Route`.
    - Disable waiting for user reply.
    - Connect:
      - `Save Onboarding Progress -> Respond — Onboarding`

11. **Build the coaching branch history loader**
    - Add a **Google Sheets** node named `Read Conversation History`.
    - Read from `Conversation Log`.
    - Add a filter where `user_id` equals the incoming `user_id`.
    - Enable `Always Output Data`.
    - Connect:
      - `IF Onboarding or Coaching (false) -> Read Conversation History`

12. **Add `Build Coaching Prompt`**
    - Add a **Code** node.
    - Logic should:
      - read the current user profile from `Check User & Route`
      - read last 5 conversation items
      - build `conversationHistory`
      - detect intent by keyword groups:
        - salary
        - interview
        - career_change
        - leadership
        - confidence
        - balance
        - fallback `general`
      - build a topic-specific system prompt
      - ask the model to reply strictly in JSON with:
        - `main_advice`
        - `actionable_step`
        - `follow_up_question`
        - `intent_detected`
    - Connect:
      - `Read Conversation History -> Build Coaching Prompt`

13. **Add the OpenAI node for live coaching**
    - Add an **OpenAI** node named `Confidence Coach`.
    - Choose model `gpt-4o`.
    - Set the prompt/content field to the generated `prompt`.
    - Do not enable tools unless you intentionally extend the workflow.
    - Connect:
      - `Build Coaching Prompt -> Confidence Coach`

14. **Add `Parse Coaching Response`**
    - Add a **Code** node.
    - Logic should:
      - read the text from the OpenAI node output
      - strip ```json code fences if present
      - parse JSON
      - build a formatted final response:
        - advice
        - action for today
        - follow-up question
      - return both formatted and raw fields
    - Connect:
      - `Confidence Coach -> Parse Coaching Response`

15. **Add conversation logging**
    - Add a **Google Sheets** node named `Log to Conversation Log`.
    - Operation: **Append**
    - Write:
      - `user_id`
      - `timestamp`
      - `user_message`
      - `intent`
      - `bot_response`
    - Connect:
      - `Parse Coaching Response -> Log to Conversation Log`

16. **Add live coaching response**
    - Add a **Chat** node named `Respond — Coaching`.
    - Message should use the formatted coaching response from `Parse Coaching Response`.
    - Disable waiting for reply.
    - Connect:
      - `Log to Conversation Log -> Respond — Coaching`

17. **Create the weekly schedule entry**
    - Add a **Schedule Trigger** node named `Weekly Sunday 10AM Trigger`.
    - Configure a weekly schedule at 10:00.
    - Verify the instance timezone so the send time is correct.

18. **Load all users for weekly sends**
    - Add a **Google Sheets** node named `Read All Onboarded Users`.
    - Read all rows from `User Profiles`.
    - Enable `Execute Once`.
    - Connect:
      - `Weekly Sunday 10AM Trigger -> Read All Onboarded Users`

19. **Filter for fully onboarded users**
    - Add a **Filter** node named `Filter Onboarded Users`.
    - Condition:
      - `onboarding_step >= 5`
    - Connect:
      - `Read All Onboarded Users -> Filter Onboarded Users`

20. **Load conversation history for weekly personalization**
    - Add a **Google Sheets** node named `Read Conversation Log`.
    - Read all rows from `Conversation Log`.
    - Enable `Execute Once`.
    - Connect:
      - `Filter Onboarded Users -> Read Conversation Log`

21. **Add `Build Checkin Prompt`**
    - Add a **Code** node.
    - Logic should:
      - get all onboarded users
      - get all conversation log rows
      - for each user, collect last 3 conversations
      - extract intent labels as recent topics
      - build a personalized weekly prompt asking for JSON:
        - `subject`
        - `greeting`
        - `progress_acknowledgment`
        - `weekly_challenge`
        - `motivational_quote`
        - `closing`
      - include profile fields and `email`
    - Connect:
      - `Read Conversation Log -> Build Checkin Prompt`

22. **Add the OpenAI node for weekly messages**
    - Add an **OpenAI** node named `Weekly Coach`.
    - Model: `gpt-4o`
    - Prompt/content = `{{$json.prompt}}`
    - Connect:
      - `Build Checkin Prompt -> Weekly Coach`

23. **Add `Parse Weekly Message`**
    - Add a **Code** node.
    - Logic should:
      - parse the AI JSON output
      - build HTML email markup with:
        - greeting
        - progress acknowledgment
        - weekly challenge box
        - motivational quote box
        - closing
        - footer
      - output `email_subject` and `email_body`
    - Connect:
      - `Weekly Coach -> Parse Weekly Message`

24. **Filter users with email**
    - Add a **Filter** node named `Filter Users With Email`.
    - Condition:
      - `email` is not empty
    - Connect:
      - `Parse Weekly Message -> Filter Users With Email`

25. **Send weekly email**
    - Add a **Gmail** node named `Send Weekly Checkin Email`.
    - Configure:
      - To = `{{$json.email}}`
      - Subject = `{{$json.email_subject}}`
      - Message = `{{$json.email_body}}`
    - Connect Gmail OAuth2 credentials.
    - Test HTML rendering with a real inbox.
    - Connect:
      - `Filter Users With Email -> Send Weekly Checkin Email`

26. **Log weekly sends**
    - Add a **Google Sheets** node named `Log to Weekly Checkins`.
    - Operation: **Append**
    - Write:
      - `user_id`
      - `name`
      - `email`
      - `date`
      - `message_sent`
    - Connect:
      - `Send Weekly Checkin Email -> Log to Weekly Checkins`

27. **Add optional documentation sticky notes**
    - Add one general note describing the whole solution.
    - Add one note above the chat/onboarding branch.
    - Add one note above the coaching branch.
    - Add one note above the weekly email branch.

28. **Test the onboarding path**
    - Start a fresh chat session.
    - Verify each message updates `User Profiles`.
    - Confirm `onboarding_step` increments correctly from new user through completion.

29. **Test the coaching path**
    - After onboarding, send a message containing a known keyword such as “salary”, “interview”, or “burnout”.
    - Verify:
      - intent detection works
      - response is valid
      - conversation row is appended in `Conversation Log`

30. **Test the weekly path**
    - Ensure at least one onboarded user has an `email` value in `User Profiles`.
    - Run the weekly branch manually.
    - Verify:
      - prompt generation
      - AI output parsing
      - email delivery
      - `Weekly Checkins` logging

31. **Recommended hardening before production**
    - Add validation for blank chat inputs.
    - Add try/catch error handling in both parse code nodes.
    - Add explicit email capture during onboarding if weekly emails are required.
    - Sort conversation history by timestamp before slicing.
    - Add fallback responses if OpenAI returns invalid JSON.

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and does **not** contain Execute Workflow nodes. No external workflow dependencies are required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow Overview: Every woman deserves a career coach but not everyone can afford one. This workflow gives every user a personal AI coach available 24/7 — trained to help with salary negotiation, interview prep, career changes, leadership challenges, and confidence building. | General workflow positioning |
| How it works: Users open the chat and complete a quick 4-step onboarding — name, career stage, biggest challenge, and monthly goal. From that point every response is personalized to their profile. GPT-4o detects the intent behind every message and routes it to a topic-specific coaching prompt. Conversation history is stored so every session builds on the last. Every Sunday users get a personalized check-in email with a weekly challenge and motivational quote. | General workflow behavior |
| Setup steps: 1. Create the Google Sheet with 3 sheets — User Profiles, Conversation Log, Weekly Checkins 2. Connect Google Sheets, OpenAI, and Gmail credentials 3. Activate the Chat Trigger and copy the chat URL 4. Share the chat URL with your users | Deployment/setup guidance |