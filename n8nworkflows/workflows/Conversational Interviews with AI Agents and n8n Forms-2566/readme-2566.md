Conversational Interviews with AI Agents and n8n Forms

https://n8nworkflows.xyz/workflows/conversational-interviews-with-ai-agents-and-n8n-forms-2566


# Conversational Interviews with AI Agents and n8n Forms

### 1. Workflow Overview

This workflow implements an automated conversational interview system using n8n forms and an AI agent powered by Groq LLM. The main goal is to conduct open-ended, ongoing interviews with users on a specific topic, dynamically generating questions and recording responses. The workflow orchestrates an interactive session that loops until the user decides to stop, stores the entire conversation in Redis for session management, and finally saves the interview transcript into Google Sheets for later analysis or sharing.

The workflow is organized into six logical blocks:

- **1.1 Interview Initiation**: Captures user entry via a form trigger, generates a unique session ID, and initializes the session storage in Redis.
- **1.2 AI Question Generation Loop**: Uses an AI agent to generate dynamic interview questions based on the topic and previous answers, looping continuously until the user signals to stop.
- **1.3 User Response Capture and Session Update**: Presents generated questions using a form node, captures user answers, and appends question-answer pairs to the Redis session.
- **1.4 Interview Termination and Cleanup**: Detects when the user requests to stop, gracefully ends the interview loop, clears AI memory buffers, and redirects to a completion screen.
- **1.5 Transcript Persistence**: Retrieves the full session from Redis and appends the data rows into a Google Sheet for external access and analysis.
- **1.6 Transcript Display via Webhook**: Provides an optional webhook flow that renders the saved transcript as an HTML page accessible via URL, handling session lookup and 404 scenarios.

---

### 2. Block-by-Block Analysis

#### 1.1 Interview Initiation

**Overview:**  
This block captures the interview start request through a form trigger, generates a unique session ID, initializes the Redis session store, and records the first question-answer pair ("What is your name?"). It also sets the interview topic and prepares the initial message for the AI agent.

**Nodes Involved:**  
- Start Interview (Form Trigger)  
- UUID (Crypto)  
- Create Session (Redis)  
- Generate Row2 (Set)  
- Update Session (Redis)  
- Set Interview Topic (Set)

**Node Details:**

- **Start Interview**  
  - Type: Form Trigger  
  - Role: Entry point form triggering the workflow, collecting user’s name to start the interview.  
  - Configuration: Form title "UK Practical Driving Test Satisfaction Interview", one required field "What is your name?", button labeled "Begin Interview!". Ignores bot submissions.  
  - Outputs user’s name as JSON.  
  - Potential failures: Webhook setup errors, form misconfiguration, bot detection false positives.

- **UUID**  
  - Type: Crypto (UUID generation)  
  - Role: Generates a unique session ID for Redis session tracking.  
  - Configuration: Generates a UUID string.  
  - Input: Triggered after form submission.  
  - Possible failure: None typical, but rare crypto generation issues.

- **Create Session**  
  - Type: Redis node  
  - Role: Initializes a Redis list keyed by `session_<UUID>`, with TTL (24 hours).  
  - Configuration: Key is dynamically set using the UUID; initializes with an empty list. TTL: 86400 seconds.  
  - Credentials: Uses Upstash Redis credentials.  
  - Edge cases: Redis errors (connectivity, auth), TTL misconfiguration.

- **Generate Row2**  
  - Type: Set node  
  - Role: Prepares the first row of the interview transcript (type: "start_interview") capturing timestamp, question ("What is your name?"), and user answer.  
  - Configuration: Timestamp ISO string, static question string, answer from form data.  
  - Output used for session update.

- **Update Session**  
  - Type: Redis node  
  - Role: Pushes the first question-answer row into the Redis session list.  
  - Input: JSON from Generate Row2.  
  - Edge cases: Redis push failures.

- **Set Interview Topic**  
  - Type: Set node  
  - Role: Sets two fields:  
    - `answer`: Greeting including user’s name from form input.  
    - `interview_topic`: Hardcoded string "Your experience preparing for and taking the UK practical driving test".  
  - This output becomes the initial input for the AI agent.  
  - Note: Marked by a sticky note as the place to set the interview topic.

---

#### 1.2 AI Question Generation Loop

**Overview:**  
This block uses the Groq Chat Model and an AI Agent node to generate open-ended interview questions dynamically based on the current conversation state. It runs in a loop, producing one question at a time, until the user requests to stop.

**Nodes Involved:**  
- Groq Chat Model (Groq LLM integration)  
- AI Researcher (LangChain AI Agent)  
- Parse Response (Set)  
- Stop Interview? (If)

**Node Details:**

- **Groq Chat Model**  
  - Type: LLM Chat model node (Groq)  
  - Role: Provides the underlying LLM generating text for the AI Researcher agent.  
  - Configuration: Model set to "llama-3.2-90b-text-preview".  
  - Credentials: Groq API credentials required.  
  - Edge cases: API rate limits, auth errors, model unavailability.

- **AI Researcher**  
  - Type: LangChain Agent  
  - Role: User research expert AI agent that asks relevant open-ended questions based on the interview topic and previous answers.  
  - Configuration:  
    - System message sets instructions to behave as a polite, expert interviewer on the specified topic.  
    - Must output JSON strictly matching a schema with two properties: `stop_interview` (boolean) and `question` (string or null).  
    - The prompt instructs to ask one question at a time, possibly one follow-on question to dig deeper.  
    - On stop request, outputs empty question and sets stop flag true.  
  - Input variable: `answer` from previous user response or greeting.  
  - Output: JSON string parsed downstream.  
  - Edge cases: Parsing failures, malformed JSON output, ambiguous user input.

- **Parse Response**  
  - Type: Set node  
  - Role: Cleans and parses the JSON output from the AI agent, extracting the `stop_interview` flag and the next `question`.  
  - Configuration: Removes markdown JSON code fences and parses the string as JSON.  
  - Failure modes: JSON parse errors if AI response is malformed.

- **Stop Interview?**  
  - Type: If node  
  - Role: Conditional node checking if the `stop_interview` flag is true to either break the loop or continue.  
  - Configuration: Checks if `output.stop_interview` is true boolean.  
  - Outputs:  
    - True branch: proceeds to graceful termination.  
    - False branch: proceeds to present next question to user.  
  - Edge cases: Incorrect flag parsing, null or missing values.

---

#### 1.3 User Response Capture and Session Update

**Overview:**  
This block displays the AI-generated question to the user using an n8n form node, collects their answer, and updates the Redis session with the new question-answer pair. It also prepares the answer to send back to the AI agent to continue the loop.

**Nodes Involved:**  
- Get Answer (Form)  
- Generate Row (Set)  
- Update Session1 (Redis)  
- Send Reply To Agent (Set)

**Node Details:**

- **Get Answer**  
  - Type: Form node  
  - Role: Presents the interview question to the user, requesting their answer or to type "stop interview" to end.  
  - Configuration:  
    - Form title bound to the AI agent’s generated question.  
    - Textarea input labeled "answer" is required.  
    - Button labeled "Next Question".  
    - Form description instructs user on how to stop.  
  - Webhook ID used for form responses.  
  - Edge cases: User submits empty answer, types irrelevant text, or stops unexpectedly.

- **Generate Row**  
  - Type: Set node  
  - Role: Creates a transcript row object representing the current Q&A (type "next_question"), with timestamp, question from AI, and user answer.  
  - Configuration: Uses current time, pulls question from "Parse Response" output, answer from "Get Answer".  
  - Output used for session update.

- **Update Session1**  
  - Type: Redis node  
  - Role: Appends the new question-answer pair to the Redis session list.  
  - Input: JSON from Generate Row.  
  - Edge cases: Redis write failures.

- **Send Reply To Agent**  
  - Type: Set node  
  - Role: Prepares the user's answer as the input for the next AI agent iteration.  
  - Configuration: Sets field `answer` with user’s response from "Get Answer".  
  - Output routed back to the AI Researcher node to continue the loop.

---

#### 1.4 Interview Termination and Cleanup

**Overview:**  
When the AI agent signals to stop the interview, this block records the termination event, clears AI memory buffers, and redirects the user to a completion screen that shows the transcript.

**Nodes Involved:**  
- Generate Row1 (Set)  
- Update Session2 (Redis)  
- Clear For Next Interview (Memory Manager)  
- Redirect to Completion Screen (Form)

**Node Details:**

- **Generate Row1**  
  - Type: Set node  
  - Role: Creates a final transcript row with type "stop_interview", marking the session end with placeholders "None" for question and answer.  
  - Timestamp generated.  
  - Output used for session update.

- **Update Session2**  
  - Type: Redis node  
  - Role: Appends the stop event row to Redis session.  
  - Input: JSON from Generate Row1.  
  - Edge cases: Redis failures.

- **Clear For Next Interview**  
  - Type: LangChain Memory Manager node  
  - Role: Clears all memory buffers associated with the session key (UUID), resetting the AI agent's context.  
  - Configuration: Mode set to delete all.  
  - Connected to memory buffer window nodes for proper session handling.  
  - Edge cases: Memory clearing failures, orphaned contexts.

- **Redirect to Completion Screen**  
  - Type: Form node with completion operation  
  - Role: Ends the form interaction by redirecting the user to a webhook URL that renders the transcript.  
  - Configuration:  
    - Redirect URL templated with session UUID for lookup.  
    - Responds with HTTP redirect.  
  - Sticky note advises to set your webhook URL here.  
  - Edge cases: Incorrect redirect, broken URLs.

---

#### 1.5 Transcript Persistence

**Overview:**  
This block extracts the full interview transcript from Redis, converts the data into suitable JSON rows, and saves it into a Google Sheet for sharing and data analysis.

**Nodes Involved:**  
- Get Session (Redis)  
- Session to List (Split Out)  
- Messages To JSON (Set)  
- Save to Google Sheet (Google Sheets)

**Node Details:**

- **Get Session**  
  - Type: Redis node  
  - Role: Retrieves the entire session list from Redis using the UUID key.  
  - Configured to execute once.  
  - Output: List of JSON strings representing transcript rows.  
  - Edge cases: Redis read failures, missing session.

- **Session to List**  
  - Type: Split Out node  
  - Role: Splits the Redis list array into individual items for further processing.  
  - Input field: session.  
  - Output: Each transcript entry as separate data item.  
  - Edge cases: Empty lists, malformed data.

- **Messages To JSON**  
  - Type: Set node  
  - Role: Converts each transcript JSON string into JSON objects with additional metadata: session_id and user name.  
  - Uses expression to parse JSON and add context fields.  
  - Edge cases: JSON parse errors.

- **Save to Google Sheet**  
  - Type: Google Sheets node  
  - Role: Appends each transcript row to a specified Google Sheet tab ("transcripts") for record-keeping.  
  - Configuration:  
    - Auto-maps input data columns to sheet columns.  
    - Uses Google OAuth2 credentials.  
  - Edge cases: Google API quota limits, auth failures, sheet access permissions.

---

#### 1.6 Transcript Display via Webhook

**Overview:**  
This optional block serves a webhook that receives a session ID, retrieves the transcript from Redis, and renders the transcript in an HTML page for user-friendly display. It handles the case of missing or expired sessions by showing a 404 page.

**Nodes Involved:**  
- Webhook (Trigger)  
- Query By Session (Redis)  
- Valid Session? (If)  
- Show Transcript (HTML)  
- 404 Not Found (HTML)  
- Respond to Webhook (Respond)

**Node Details:**

- **Webhook**  
  - Type: Webhook Trigger  
  - Role: Listens at path `/ai-interview-transcripts/:session_id` for HTTP GET requests.  
  - Configuration: Ignores bots, response mode set to use Respond to Webhook node.  
  - Inputs session_id parameter.  
  - Edge cases: Invalid URLs, bot traffic.

- **Query By Session**  
  - Type: Redis node  
  - Role: Retrieves session data (list) from Redis using the session_id from the webhook URL.  
  - Output property: `data`.  
  - Edge cases: Redis misses, expired keys.

- **Valid Session?**  
  - Type: If node  
  - Role: Checks if Redis returned any data (array exists).  
  - True branch: proceeds to Show Transcript.  
  - False branch: routes to 404 Not Found page.  
  - Edge cases: Empty arrays.

- **Show Transcript**  
  - Type: HTML node  
  - Role: Renders an HTML page showing the interview transcript with styled question-answer pairs and timestamps.  
  - Uses embedded JavaScript templating to parse and format JSON data.  
  - Provides credits and links in footer.  
  - Edge cases: Rendering failures, large transcripts.

- **404 Not Found**  
  - Type: HTML node  
  - Role: Renders a 404 error page informing the user that the requested session does not exist or has expired.  
  - Provides a link to n8n signup page as a call to action.  
  - Edge cases: None.

- **Respond to Webhook**  
  - Type: Respond to Webhook node  
  - Role: Sends the HTML content as HTTP response with content-type `text/html` and status 200.  
  - Edge cases: Response failures.

---

### 3. Summary Table

| Node Name                | Node Type                          | Functional Role                                  | Input Node(s)                | Output Node(s)                 | Sticky Note                                                                                                                               |
|--------------------------|----------------------------------|-------------------------------------------------|------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Start Interview          | Form Trigger                     | Entry point form capturing user's name          |                              | UUID                          | ## 1. Setup Interview. [Learn more about the form trigger node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.formtrigger) |
| UUID                     | Crypto (UUID)                   | Generates unique session ID                       | Start Interview              | Create Session                |                                                                                                                                           |
| Create Session           | Redis                           | Initializes Redis session list                    | UUID                         | Generate Row2                 |                                                                                                                                           |
| Generate Row2            | Set                             | Creates first transcript row                      | Create Session               | Update Session                |                                                                                                                                           |
| Update Session           | Redis                           | Appends first row to Redis session                | Generate Row2                | Set Interview Topic           |                                                                                                                                           |
| Set Interview Topic      | Set                             | Sets greeting answer and interview topic         | Update Session               | AI Researcher                | 🚨 Set Interview Topic Here!                                                                                                              |
| Groq Chat Model          | LLM Chat Model (Groq)           | Provides LLM backend                              | Send Reply To Agent          | AI Researcher                |                                                                                                                                           |
| AI Researcher            | LangChain Agent                 | Generates interview questions                     | Set Interview Topic / Send Reply To Agent | Parse Response               | ## 2. AI Researcher for Endless Interview Questions. [Learn more about the AI Agent node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/) |
| Parse Response           | Set                             | Parses AI agent JSON output                        | AI Researcher                | Stop Interview?              |                                                                                                                                           |
| Stop Interview?          | If                              | Checks if user requested to stop interview       | Parse Response               | Generate Row1 / Get Answer    |                                                                                                                                           |
| Generate Row1            | Set                             | Creates final "stop_interview" transcript row    | Stop Interview? (true branch) | Update Session2              |                                                                                                                                           |
| Update Session2          | Redis                           | Appends stop transcript row to Redis             | Generate Row1                | Clear For Next Interview      |                                                                                                                                           |
| Clear For Next Interview | Memory Manager                  | Clears AI session memory buffers                  | Update Session2              | Redirect to Completion Screen | ## 4. Graciously End the Interview. [Read more about the Chat Manager node](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.memorymanager/) |
| Redirect to Completion Screen | Form                        | Redirects user to completion transcript display  | Clear For Next Interview     | Get Session                  | 🚨 Set Your Webhook URL here!                                                                                                             |
| Get Answer               | Form                            | Presents AI question, captures user answer       | Stop Interview? (false branch) | Generate Row                 | ## 3. Record Answers and Prep for Next Question. [Learn more about the n8n Form node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.form/) |
| Generate Row             | Set                             | Creates transcript row for current Q&A            | Get Answer                  | Update Session1              |                                                                                                                                           |
| Update Session1          | Redis                           | Appends Q&A row to Redis session                  | Generate Row                | Send Reply To Agent           |                                                                                                                                           |
| Send Reply To Agent      | Set                             | Prepares user answer to send back to AI agent    | Update Session1             | Groq Chat Model / AI Researcher |                                                                                                                                           |
| Get Session              | Redis                           | Retrieves full session from Redis                  | Redirect to Completion Screen | Session to List              |                                                                                                                                           |
| Session to List          | Split Out                      | Splits session list into individual transcript items | Get Session                | Messages To JSON              |                                                                                                                                           |
| Messages To JSON         | Set                             | Parses and enriches JSON transcript messages      | Session to List             | Save to Google Sheet          |                                                                                                                                           |
| Save to Google Sheet     | Google Sheets                  | Appends transcript data to Google Sheet           | Messages To JSON            |                              | ## 5. Save the Interview to Sheets. [Read more about the Google Sheets node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/) |
| Webhook                  | Webhook Trigger                | HTTP endpoint to serve transcript display         |                              | Query By Session              | ## 6. Display the Transcript. [Read more about the Webhook Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook) |
| Query By Session         | Redis                           | Retrieves session data for given session_id       | Webhook                     | Valid Session?               |                                                                                                                                           |
| Valid Session?           | If                              | Checks if session data exists                       | Query By Session            | Show Transcript / 404 Not Found |                                                                                                                                           |
| Show Transcript          | HTML                            | Renders the interview transcript as HTML page     | Valid Session? (true branch) | Respond to Webhook           |                                                                                                                                           |
| 404 Not Found            | HTML                            | Renders 404 error page if session not found       | Valid Session? (false branch) | Respond to Webhook           |                                                                                                                                           |
| Respond to Webhook       | Respond to Webhook             | Sends HTTP response with HTML content              | Show Transcript / 404 Not Found |                              |                                                                                                                                           |
| Window Buffer Memory     | Memory Buffer Window           | AI memory buffer for session (cleared on end)     |                             | Clear For Next Interview      |                                                                                                                                           |
| Window Buffer Memory2    | Memory Buffer Window           | AI memory buffer for session (used by AI Researcher) |                            | AI Researcher                |                                                                                                                                           |
| Sticky Notes (multiple)  | Sticky Note                   | Document comments and configuration pointers       |                             |                              | Various notes covering interview setup, AI agent behavior, data saving, and redirect URLs. See detailed notes above.                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node ("Start Interview"):**  
   - Set path: `driving-lessons-survey`  
   - Form title: `UK Practical Driving Test Satisfaction Interview`  
   - Add one required text field: "What is your name?" with placeholder "ie. Sam Smith"  
   - Button label: "Begin Interview!"  
   - Ignore bots enabled  
   - Append attribution enabled  
   - Use workflow timezone enabled  
   - Add a webhook to receive submissions.

2. **Add a Crypto node ("UUID"):**  
   - Action: Generate UUID  
   - Connect output of "Start Interview" to this node.

3. **Add Redis node ("Create Session"):**  
   - Operation: Set  
   - Key: `session_{{ $json.data }}` (use UUID from previous node)  
   - Value: empty list `[]` (string)  
   - TTL: 86400 seconds (24 hours)  
   - Key type: list  
   - Use Upstash or compatible Redis credentials.  
   - Connect from "UUID".

4. **Add Set node ("Generate Row2"):**  
   - Assign fields:  
     - `timestamp`: current ISO timestamp (`={{ $now.toISO() }}`)  
     - `type`: "start_interview"  
     - `question`: "What is your name?" (static)  
     - `answer`: `={{ $('Start Interview').first().json['What is your name?'] }}`  
   - Connect from "Create Session".

5. **Add Redis node ("Update Session"):**  
   - Operation: Push (tail)  
   - List: `session_{{ $('UUID').first().json.data }}`  
   - Message data: JSON string of the new row from "Generate Row2"  
   - Credentials: same Redis credentials  
   - Connect from "Generate Row2".

6. **Add Set node ("Set Interview Topic"):**  
   - Assign fields:  
     - `answer`: `Hello, my name is {{ $('Start Interview').first().json['What is your name?'] }}`  
     - `interview_topic`: set your interview topic text (e.g. "Your experience preparing for and taking the UK practical driving test")  
   - Connect from "Update Session".

7. **Add Groq Chat Model node ("Groq Chat Model"):**  
   - Model: `llama-3.2-90b-text-preview` (or your chosen LLM)  
   - Use Groq API credentials  
   - Connect from "Send Reply To Agent" (created later).

8. **Add LangChain Agent node ("AI Researcher"):**  
   - Text input: `={{ $json.answer }}`  
   - System message: instruct the AI to act as a user research expert interviewing on the `{{ $('Set Interview Topic').first().json.interview_topic }}`  
   - Output JSON schema enforced with `stop_interview` (boolean) and `question` (string or null) only.  
   - Connect `ai_languageModel` input to "Groq Chat Model".

9. **Add Set node ("Parse Response"):**  
   - Clean AI output by removing code fences and parse JSON.  
   - Expression example:  
     ```
     {{$json.output.replace('```json','').replace('```','').parseJson()}}
     ```  
   - Connect from "AI Researcher".

10. **Add If node ("Stop Interview?"):**  
    - Condition: Check if `{{$json.output.stop_interview}}` is boolean true.  
    - Connect from "Parse Response".

11. **Add Form node ("Get Answer"):**  
    - Form title: bind to AI question `={{ $json.output.question }}`  
    - One textarea field labeled "answer", required  
    - Button label: "Next Question"  
    - Form description: instruct to answer or type "stop interview"  
    - Connect from "Stop Interview?" false branch.

12. **Add Set node ("Generate Row"):**  
    - Assign fields:  
      - `timestamp`: current ISO timestamp  
      - `type`: "next_question"  
      - `question`: from "Parse Response" question field  
      - `answer`: from "Get Answer" answer field  
    - Connect from "Get Answer".

13. **Add Redis node ("Update Session1"):**  
    - Push message to Redis list as before  
    - Connect from "Generate Row".

14. **Add Set node ("Send Reply To Agent"):**  
    - Set `answer` field from "Get Answer" answer  
    - Connect from "Update Session1".

15. **Connect "Send Reply To Agent" to "Groq Chat Model"** to complete the question-answer loop.

16. **Add Set node ("Generate Row1"):**  
    - Assign fields:  
      - `timestamp`: current ISO timestamp  
      - `type`: "stop_interview"  
      - `question`: "None"  
      - `answer`: "None"  
    - Connect from "Stop Interview?" true branch.

17. **Add Redis node ("Update Session2"):**  
    - Push stop event to Redis  
    - Connect from "Generate Row1".

18. **Add LangChain Memory Manager node ("Clear For Next Interview"):**  
    - Mode: delete all for session key  
    - Session key: `={{ $('UUID').first().json.data }}`  
    - Connect from "Update Session2".

19. **Add Form node ("Redirect to Completion Screen"):**  
    - Operation: `completion`  
    - Redirect URL: set to your webhook URL with `{{ $('UUID').first().json.data }}` as parameter, e.g.:  
      `https://<host>/webhook/<uuid-if-using-n8n-cloud>/ai-interview-transcripts/{{ $('UUID').first().json.data }}`  
    - Respond with redirect  
    - Connect from "Clear For Next Interview".

20. **Add Redis node ("Get Session"):**  
    - Operation: get full list of session rows by key  
    - Connect from "Redirect to Completion Screen".

21. **Add Split Out node ("Session to List"):**  
    - Field to split out: `session`  
    - Connect from "Get Session".

22. **Add Set node ("Messages To JSON"):**  
    - Parse each item from JSON string to object and add metadata fields `session_id` and `name`.  
    - Connect from "Session to List".

23. **Add Google Sheets node ("Save to Google Sheet"):**  
    - Operation: append rows  
    - Sheet name and document ID configured to your target sheet (create the sheet with columns: session_id, timestamp, name, type, question, answer)  
    - Auto map input data to sheet columns  
    - Connect from "Messages To JSON".

24. **Add Webhook node ("Webhook"):**  
    - Path: `ai-interview-transcripts/:session_id`  
    - Ignore bots enabled  
    - Response mode: response node  
    - This will serve transcript display requests.

25. **Add Redis node ("Query By Session"):**  
    - Operation: get  
    - Key: `session_{{ $json.params.session_id }}`  
    - Connect from "Webhook".

26. **Add If node ("Valid Session?"):**  
    - Condition: Check if data array exists and is not empty  
    - Connect from "Query By Session".

27. **Add HTML node ("Show Transcript"):**  
    - Render an HTML page displaying the JSON transcript with formatted question-answer pairs and timestamps.  
    - Connect from "Valid Session?" true branch.

28. **Add HTML node ("404 Not Found"):**  
    - Render a 404 error page with message about missing session.  
    - Connect from "Valid Session?" false branch.

29. **Add Respond to Webhook node:**  
    - Sends the HTML content from either Show Transcript or 404 node with content type `text/html`.  
    - Connect from both HTML nodes.

30. **Add two Memory Buffer Window nodes ("Window Buffer Memory" and "Window Buffer Memory2"):**  
    - Both configured with `sessionKey` as the UUID, sessionIdType "customKey"  
    - One connected as AI memory input to the AI Researcher node  
    - One connected to Clear For Next Interview node for cleanup.

31. **Add Sticky Notes:**  
    - Add descriptive sticky notes as in the original workflow for clarity and documentation on key nodes and blocks.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow leverages n8n forms for user interaction and Redis for session storage, enabling highly scalable interview sessions. Redis TTL ensures sessions expire after 24 hours automatically.                                                                                                                                                                                                                                                                                                             | Workflow design principle                                                                           |
| The AI agent uses Groq LLM via n8n LangChain agent node, but you can replace the LLM with any other supported model (OpenAI, HuggingFace, etc.) by swapping the Groq Chat Model node.                                                                                                                                                                                                                                                                                                                         | Customization suggestion                                                                            |
| The interview topic is statically set in the "Set Interview Topic" node — change this to perform interviews on different subjects.                                                                                                                                                                                                                                                                                                                                                                            | Configuration pointer                                                                              |
| The form completion redirect URL requires a valid webhook URL that responds with the transcript HTML page. Adjust this URL for your deployment environment (self-hosted or n8n cloud).                                                                                                                                                                                                                                                                                                                           | Important deployment detail                                                                         |
| Google Sheets is used for easy sharing and analysis of interview transcripts. You must configure Google OAuth2 credentials with appropriate permissions.                                                                                                                                                                                                                                                                                                                                                        | Integration requirement                                                                            |
| Redis storage uses Upstash.com in the demo; any Redis-compatible storage with suitable credentials can be used.                                                                                                                                                                                                                                                                                                                                                                                                  | Integration requirement                                                                            |
| The transcript display webhook is optional but provides a user-friendly way to view interview transcripts immediately after completion.                                                                                                                                                                                                                                                                                                                                                                        | Optional feature                                                                                   |
| Full workflow showcase and demo are available at: https://community.n8n.io/t/build-your-own-ai-interview-agents-with-n8n-forms/62312 and https://jimleuk.app.n8n.cloud/form/driving-lessons-survey                                                                                                                                                                                                                                                                                                                 | Original post and live demo                                                                         |
| For help and community support, join the n8n Discord or ask questions on the n8n Community Forum.                                                                                                                                                                                                                                                                                                                                                                                                               | https://discord.com/invite/XPKeKXeB7d, https://community.n8n.io/                                   |

---

This comprehensive reference document provides the necessary analysis, configuration details, and stepwise instructions to understand, reproduce, and modify the "Conversational Interviews with AI Agents and n8n Forms" workflow. It anticipates key failure modes, integration dependencies, and customization points for both technical users and automation agents.