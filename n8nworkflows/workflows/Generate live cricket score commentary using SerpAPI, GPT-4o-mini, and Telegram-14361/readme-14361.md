Generate live cricket score commentary using SerpAPI, GPT-4o-mini, and Telegram

https://n8nworkflows.xyz/workflows/generate-live-cricket-score-commentary-using-serpapi--gpt-4o-mini--and-telegram-14361


# Generate live cricket score commentary using SerpAPI, GPT-4o-mini, and Telegram

## 1. Workflow Overview

This workflow monitors live cricket scores for India-related matches, generates short AI-written commentary from the latest available match data, and posts the result to Telegram. It also includes two alerting mechanisms: a Gmail notification intended for AI-generation issues and a Slack notification for workflow-level failures.

Its main purpose is to automate near-real-time cricket commentary delivery for chat audiences, using:
- **SerpAPI** to fetch Google sports results
- **OpenAI GPT-4o-mini** to generate commentary
- **Telegram** to distribute the commentary
- **Gmail** and **Slack** to surface failures

### 1.1 Scheduled Input Reception and Live Score Fetch
This block starts the workflow every 10 seconds and queries SerpAPI for live cricket results related to India.

### 1.2 Match Extraction and Validation
This block parses the SerpAPI sports results payload, extracts the first game as the “latest” match, and checks whether a result/status exists before continuing.

### 1.3 AI Commentary Generation
This block sends the extracted match fields to an AI Agent powered by GPT-4o-mini and requests a structured cricket-commentary response.

### 1.4 Delivery and Failure Notification
This block sends the generated commentary to Telegram. It also includes a Gmail node intended to alert on AI failure, though the current wiring/configuration has important caveats.

### 1.5 Global Workflow Error Monitoring
This independent entry path uses an Error Trigger to catch workflow-level failures and send a Slack alert.

### 1.6 Documentation and Security Notes
Several sticky notes document setup, credentials, operational behavior, and security practices.

---

## 2. Block-by-Block Analysis

## 2.1 Scheduled Input Reception and Live Score Fetch

**Overview:**  
This block initiates the workflow on a fixed cadence and fetches live cricket data from SerpAPI using a Google-style search query. It is the source of all match data used downstream.

**Nodes Involved:**  
- Every 10 Seconds Trigger
- Fetch Live Cricket Score

### Node Details

#### 2.1.1 Every 10 Seconds Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Time-based trigger node that starts the workflow automatically.
- **Configuration choices:**  
  Configured to run every **10 seconds**.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - No input
  - Output → **Fetch Live Cricket Score**
- **Version-specific requirements:**  
  Uses **typeVersion 1.1**. Standard schedule trigger behavior in current n8n versions.
- **Edge cases or potential failure types:**  
  - Very frequent polling can consume API quota quickly.
  - Can create duplicate or near-duplicate commentary if the score/status does not materially change between runs.
  - May be too aggressive for production if SerpAPI usage limits are low.
- **Sub-workflow reference:**  
  None.

#### 2.1.2 Fetch Live Cricket Score
- **Type and technical role:** `n8n-nodes-serpapi.serpApi`  
  Performs a SerpAPI query to fetch Google search result data, including sports results.
- **Configuration choices:**  
  - Query: **“cricket live score india match”**
  - Location: **India**
  - No extra request options or additional fields are set.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input ← **Every 10 Seconds Trigger**
  - Output → **Extract Latest Match Data**
- **Version-specific requirements:**  
  Uses **typeVersion 1**. Requires the SerpAPI community/integration node to be installed and compatible with the n8n instance.
- **Edge cases or potential failure types:**  
  - Invalid or expired SerpAPI credentials
  - Quota/rate-limit exhaustion
  - Search results not containing `sports_results`
  - Geographic result variation despite the `India` location
  - Intermittent external API latency
- **Sub-workflow reference:**  
  None.

---

## 2.2 Match Extraction and Validation

**Overview:**  
This block transforms the raw SerpAPI response into a simpler match object and filters out executions where no match result/status is available. It protects the AI prompt from receiving empty match summaries, but the extraction logic itself is not defensive enough.

**Nodes Involved:**  
- Extract Latest Match Data
- Skip If No Match Result

### Node Details

#### 2.2.1 Extract Latest Match Data
- **Type and technical role:** `n8n-nodes-base.function`  
  Executes custom JavaScript to parse the incoming JSON and emit a simplified object for the latest match.
- **Configuration choices:**  
  The code:
  - Reads `sports_results.games`
  - Defaults to an empty array if missing
  - Selects `games[0]` as the latest match
  - Returns:
    - `team1`
    - `team2`
    - `score1`
    - `score2`
    - `result`
    - `tournament`
- **Key expressions or variables used:**  
  JavaScript variables:
  - `const games = $json.sports_results?.games || [];`
  - `const match = games[0];`
- **Input and output connections:**  
  - Input ← **Fetch Live Cricket Score**
  - Output → **Skip If No Match Result**
- **Version-specific requirements:**  
  Uses **typeVersion 1**. In newer n8n builds, a **Code** node is often preferred over the legacy Function node.
- **Edge cases or potential failure types:**  
  - If `games` is empty, `match` becomes `undefined`
  - Accessing `match.teams[0]` or other nested fields will throw an error if no game exists
  - Assumes exactly two teams and expected score/status structure
  - Assumes the first game is the most relevant match, which may not always be true
- **Sub-workflow reference:**  
  None.

#### 2.2.2 Skip If No Match Result
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional filter that only allows the flow to continue when `result` is not empty.
- **Configuration choices:**  
  Uses a **not empty** string condition on:
  - `{{$json.result}}`
- **Key expressions or variables used:**  
  - `={{ $json.result }}`
- **Input and output connections:**  
  - Input ← **Extract Latest Match Data**
  - True output → **Generate AI Match Commentary**
  - False output → not connected
- **Version-specific requirements:**  
  Uses **typeVersion 2.3** with condition options version 3.
- **Edge cases or potential failure types:**  
  - If upstream extraction fails before this node, this node never runs
  - “Not empty” does not guarantee meaningful content; placeholder statuses could still pass
  - If the result exists but score fields are missing, the AI prompt may still be poor
- **Sub-workflow reference:**  
  None.

---

## 2.3 AI Commentary Generation

**Overview:**  
This block sends the extracted match information into an AI Agent and asks for a concise structured commentary in a cricket expert tone. The actual language model is provided by a separate OpenAI Chat Model node attached through the AI language-model connection.

**Nodes Involved:**  
- Generate AI Match Commentary
- OpenAI GPT-4o-mini Model

### Node Details

#### 2.3.1 Generate AI Match Commentary
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  AI Agent node that builds a prompt and delegates generation to a connected language model.
- **Configuration choices:**  
  - Prompt type: **define**
  - Prompt instructs the model to act as an expert cricket analyst/commentator
  - Injects:
    - `team1`
    - `team2`
    - `tournament`
    - `score1`
    - `score2`
    - `result`
  - Requires this output structure:
    - Match Summary
    - Key Turning Point
    - Best Performance
    - Tactical Insight
  - Style rules:
    - Short and crisp
    - Expert tone
    - No generic AI phrasing
    - Engaging commentary style
  - `onError` is set to **continueErrorOutput**
  - `retryOnFail` is **false**
- **Key expressions or variables used:**  
  Prompt expressions:
  - `{{$json.team1}}`
  - `{{$json.team2}}`
  - `{{$json.tournament}}`
  - `{{$json.score1}}`
  - `{{$json.score2}}`
  - `{{$json.result}}`
- **Input and output connections:**  
  - Input ← **Skip If No Match Result**
  - AI language model input ← **OpenAI GPT-4o-mini Model**
  - Main output → **Send Commentary to Telegram**
  - Main output → **Email Alert on AI Failure**
- **Version-specific requirements:**  
  Uses **typeVersion 3**. Requires n8n versions supporting LangChain-based AI Agent nodes.
- **Edge cases or potential failure types:**  
  - OpenAI authentication errors
  - Model unavailability or rate limits
  - Prompt output may be generic if match data is sparse
  - Since `onError` is set to continue with error output, downstream nodes may receive error-shaped data instead of normal AI output
  - No retry configured
- **Sub-workflow reference:**  
  None.

#### 2.3.2 OpenAI GPT-4o-mini Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
  Provides the LLM backend used by the AI Agent.
- **Configuration choices:**  
  - Model: **gpt-4o-mini**
  - No extra options or built-in tools configured
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Output (AI language model) → **Generate AI Match Commentary**
- **Version-specific requirements:**  
  Uses **typeVersion 1.3**. Requires OpenAI credentials and a n8n version supporting this LangChain OpenAI model node.
- **Edge cases or potential failure types:**  
  - Invalid API key
  - Account quota/rate limiting
  - Model deprecation or naming changes in OpenAI APIs
  - Regional or account-level access restrictions
- **Sub-workflow reference:**  
  None.

---

## 2.4 Delivery and Failure Notification

**Overview:**  
This block sends successful commentary to Telegram and includes a Gmail alert node intended to notify on AI failure. However, the current topology does not isolate success from failure correctly, and one connection introduces an unintended loop.

**Nodes Involved:**  
- Send Commentary to Telegram
- Email Alert on AI Failure

### Node Details

#### 2.4.1 Send Commentary to Telegram
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends a message through a Telegram bot to a specified chat.
- **Configuration choices:**  
  - Message text: `{{$json.output}}`
  - `chatId`: placeholder value `YOUR_TELEGRAM_CHAT_ID`
  - Parse mode: **Markdown**
- **Key expressions or variables used:**  
  - `={{ $json.output }}`
- **Input and output connections:**  
  - Input ← **Generate AI Match Commentary**
  - No downstream output
- **Version-specific requirements:**  
  Uses **typeVersion 1.1**.
- **Edge cases or potential failure types:**  
  - Invalid Telegram bot token
  - Incorrect chat ID
  - Bot not added to target group/channel
  - Markdown parse errors if generated text contains problematic characters
  - If AI output is missing because the agent failed, this node may send an empty or invalid message
- **Sub-workflow reference:**  
  None.

#### 2.4.2 Email Alert on AI Failure
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends an email via Gmail OAuth2.
- **Configuration choices:**  
  - To: `user@example.com`
  - Subject: **Error occurred in Cricket Score Workflow**
  - Message body explains that AI commentary generation failed
- **Key expressions or variables used:**  
  None in the current body; it is static.
- **Input and output connections:**  
  - Input ← **Generate AI Match Commentary**
  - Output → **Fetch Live Cricket Score**
- **Version-specific requirements:**  
  Uses **typeVersion 2.2** and requires Gmail OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Gmail OAuth2 authorization failure
  - Sender account restrictions
  - Recipient not updated from placeholder
  - Static message gives no execution-specific diagnostics
  - **Important design issue:** this node is connected from the AI Agent’s main output, so it may run on successful executions as well, depending on how the agent output is emitted
  - **Critical loop risk:** its output is connected back to **Fetch Live Cricket Score**, which can unintentionally re-enter the fetch path after sending email, potentially causing repeated or confusing behavior
- **Sub-workflow reference:**  
  None.

**Important implementation note for this block:**  
The workflow appears to intend:
- successful AI result → Telegram
- failed AI result → Gmail

But the current design does **not** explicitly split success and failure branches. Because the AI Agent uses `continueErrorOutput`, the correct pattern would typically require checking whether the AI node produced an error item or a valid `output`, then branching with an If node. As built, this logic is ambiguous and should be treated as a configuration flaw.

---

## 2.5 Global Workflow Error Monitoring

**Overview:**  
This is a separate error-handling entry point. When the workflow fails at the execution level, it sends a Slack message to a selected channel.

**Nodes Involved:**  
- Error Trigger
- Alert on Workflow Failure

### Node Details

#### 2.5.1 Error Trigger
- **Type and technical role:** `n8n-nodes-base.errorTrigger`  
  Starts a dedicated error workflow branch whenever the parent workflow encounters an execution failure.
- **Configuration choices:**  
  No custom parameters.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - No input
  - Output → **Alert on Workflow Failure**
- **Version-specific requirements:**  
  Uses **typeVersion 1**.
- **Edge cases or potential failure types:**  
  - Only catches failures that actually propagate as workflow errors
  - May not trigger for errors that are swallowed or converted into normal output items
- **Sub-workflow reference:**  
  None.

#### 2.5.2 Alert on Workflow Failure
- **Type and technical role:** `n8n-nodes-base.slack`  
  Posts a text alert to a Slack channel.
- **Configuration choices:**  
  - Sends to a selected channel
  - Channel configured as cached result name: **workflow-errors**
  - Static message:
    - “⚠️ Hotel Pre-Arrival Workflow Error Detected”
    - “Please check the execution log for details.”
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input ← **Error Trigger**
  - No downstream output
- **Version-specific requirements:**  
  Uses **typeVersion 2.4**.
- **Edge cases or potential failure types:**  
  - Slack credential issues
  - Bot lacks permission to post in the channel
  - Static message references the wrong workflow name (“Hotel Pre-Arrival”), which is likely a copy/paste mistake and may confuse operators
- **Sub-workflow reference:**  
  None.

---

## 2.6 Documentation and Security Notes

**Overview:**  
This block contains sticky notes that explain the workflow behavior, setup requirements, credentials, and operational intent. They do not affect execution but are important for maintainability.

**Nodes Involved:**  
- Overview
- Section: Trigger & Fetch
- Section: Extract & Validate
- Section: AI Agent
- Section: Deliver & Alert
- Section: Security
- Sticky Note6

### Node Details

#### 2.6.1 Overview
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Workspace documentation note.
- **Configuration choices:**  
  Describes workflow behavior and setup steps for SerpAPI, OpenAI, Telegram, Gmail, and optional Slack.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  None.
- **Version-specific requirements:**  
  Uses **typeVersion 1**.
- **Edge cases or potential failure types:**  
  None at runtime.
- **Sub-workflow reference:**  
  None.

#### 2.6.2 Section: Trigger & Fetch
- Same technical role as above.
- Documents the polling and data-fetch stage.

#### 2.6.3 Section: Extract & Validate
- Same technical role as above.
- Documents parsing and validation behavior.

#### 2.6.4 Section: AI Agent
- Same technical role as above.
- Documents prompt intent and AI tone controls.

#### 2.6.5 Section: Deliver & Alert
- Same technical role as above.
- Documents Telegram delivery and alerting behavior.

#### 2.6.6 Section: Security
- Same technical role as above.
- Documents secure credential practices.

#### 2.6.7 Sticky Note6
- Same technical role as above.
- Documents Slack-based workflow error monitoring.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview | Sticky Note | Workspace documentation and setup guidance |  |  | ## 🏏 Live Cricket Score Commentator Bot<br>### How it works<br>This workflow runs every 10 seconds, fetches live cricket scores for India matches via SerpAPI, extracts the latest match details, and passes them to an AI agent. The agent generates a short, commentary-style match summary and sends it directly to a Telegram chat. If the AI step fails, an error email is dispatched automatically.<br>### Setup steps<br>1. **SerpAPI** — Connect your SerpAPI credential and confirm the search query targets your preferred match/team.<br>2. **OpenAI** — Add your OpenAI API key under the `OpenAI Chat Model` node.<br>3. **Telegram** — Set up your Telegram Bot token and replace the `chatId` with your target chat or group ID.<br>4. **Gmail** — Connect a Gmail OAuth2 account for error alert emails. Update the recipient address.<br>5. **Slack (optional)** — Link a Slack credential for the `Alert on Workflow Failure` node and select the right channel.<br>6. Activate the workflow and monitor the first few executions to confirm data flows correctly. |
| Section: Trigger & Fetch | Sticky Note | Documents polling and SerpAPI fetch stage |  |  | ## ⏱ Trigger & Live Data Fetch<br>Fires every 10 seconds and queries Google (via SerpAPI) for real-time India cricket scores. Adjust the interval or search query here if you want less frequent updates or a different team. |
| Section: Extract & Validate | Sticky Note | Documents match parsing and filtering stage |  |  | ## 🔍 Extract & Validate Match Data<br>Parses the raw SerpAPI response to pull the latest match — team names, scores, result, and tournament. The `If` node skips execution entirely when no match result is present, preventing empty AI prompts. |
| Section: AI Agent | Sticky Note | Documents AI commentary generation stage |  |  | ## 🤖 AI Commentary Generator<br>The AI Agent uses GPT-4o-mini to turn raw match data into a structured, expert-tone summary — covering match result, key turning point, standout performer, and a tactical insight. Edit the prompt inside the `AI Agent` node to adjust tone or output format. |
| Section: Deliver & Alert | Sticky Note | Documents Telegram delivery and alerting stage |  |  | ## 📲 Deliver & Alert<br>On success, the formatted commentary is sent to a Telegram chat using Markdown. On AI failure, a Gmail alert is dispatched so you can follow up manually. A separate error trigger also posts to a Slack channel for any workflow-level failures. |
| Section: Security | Sticky Note | Documents credential handling guidance |  |  | ## 🔐 Credentials & Security<br>Use named credentials for SerpAPI, OpenAI, Telegram, Gmail, and Slack. Never hardcode API keys or tokens directly in node parameters. Replace any personal emails or workspace IDs with placeholders before sharing this template. |
| Every 10 Seconds Trigger | Schedule Trigger | Starts workflow every 10 seconds |  | Fetch Live Cricket Score | ## ⏱ Trigger & Live Data Fetch<br>Fires every 10 seconds and queries Google (via SerpAPI) for real-time India cricket scores. Adjust the interval or search query here if you want less frequent updates or a different team. |
| Fetch Live Cricket Score | SerpAPI | Fetches live cricket search results from SerpAPI | Every 10 Seconds Trigger<br>Email Alert on AI Failure | Extract Latest Match Data | ## ⏱ Trigger & Live Data Fetch<br>Fires every 10 seconds and queries Google (via SerpAPI) for real-time India cricket scores. Adjust the interval or search query here if you want less frequent updates or a different team. |
| Extract Latest Match Data | Function | Extracts first match fields from SerpAPI response | Fetch Live Cricket Score | Skip If No Match Result | ## 🔍 Extract & Validate Match Data<br>Parses the raw SerpAPI response to pull the latest match — team names, scores, result, and tournament. The `If` node skips execution entirely when no match result is present, preventing empty AI prompts. |
| Skip If No Match Result | If | Continues only when match result/status exists | Extract Latest Match Data | Generate AI Match Commentary | ## 🔍 Extract & Validate Match Data<br>Parses the raw SerpAPI response to pull the latest match — team names, scores, result, and tournament. The `If` node skips execution entirely when no match result is present, preventing empty AI prompts. |
| Generate AI Match Commentary | AI Agent | Generates structured cricket commentary from match data | Skip If No Match Result<br>OpenAI GPT-4o-mini Model (AI language model input) | Send Commentary to Telegram<br>Email Alert on AI Failure | ## 🤖 AI Commentary Generator<br>The AI Agent uses GPT-4o-mini to turn raw match data into a structured, expert-tone summary — covering match result, key turning point, standout performer, and a tactical insight. Edit the prompt inside the `AI Agent` node to adjust tone or output format. |
| OpenAI GPT-4o-mini Model | OpenAI Chat Model | Supplies GPT-4o-mini as the LLM for the AI Agent |  | Generate AI Match Commentary | ## 🤖 AI Commentary Generator<br>The AI Agent uses GPT-4o-mini to turn raw match data into a structured, expert-tone summary — covering match result, key turning point, standout performer, and a tactical insight. Edit the prompt inside the `AI Agent` node to adjust tone or output format. |
| Send Commentary to Telegram | Telegram | Delivers generated commentary to Telegram chat | Generate AI Match Commentary |  | ## 📲 Deliver & Alert<br>On success, the formatted commentary is sent to a Telegram chat using Markdown. On AI failure, a Gmail alert is dispatched so you can follow up manually. A separate error trigger also posts to a Slack channel for any workflow-level failures. |
| Email Alert on AI Failure | Gmail | Sends email alert intended for AI-generation failure | Generate AI Match Commentary | Fetch Live Cricket Score | ## 📲 Deliver & Alert<br>On success, the formatted commentary is sent to a Telegram chat using Markdown. On AI failure, a Gmail alert is dispatched so you can follow up manually. A separate error trigger also posts to a Slack channel for any workflow-level failures. |
| Sticky Note6 | Sticky Note | Documents error-monitoring branch |  |  | ## ⚠️ Error Monitoring<br>Catches any workflow failures and sends alerts to Slack's general-information channel. Helps maintain reliability and enables quick troubleshooting. |
| Error Trigger | Error Trigger | Starts error-handling branch on workflow failure |  | Alert on Workflow Failure | ## ⚠️ Error Monitoring<br>Catches any workflow failures and sends alerts to Slack's general-information channel. Helps maintain reliability and enables quick troubleshooting. |
| Alert on Workflow Failure | Slack | Sends workflow-failure alert to Slack | Error Trigger |  | ## ⚠️ Error Monitoring<br>Catches any workflow failures and sends alerts to Slack's general-information channel. Helps maintain reliability and enables quick troubleshooting. |

---

## 4. Reproducing the Workflow from Scratch

Below is a full rebuild sequence in n8n.

### 4.1 Create the documentation notes
1. Add a **Sticky Note** named **Overview**.
   - Paste the high-level description and setup checklist.
2. Add a **Sticky Note** named **Section: Trigger & Fetch**.
3. Add a **Sticky Note** named **Section: Extract & Validate**.
4. Add a **Sticky Note** named **Section: AI Agent**.
5. Add a **Sticky Note** named **Section: Deliver & Alert**.
6. Add a **Sticky Note** named **Section: Security**.
7. Add a **Sticky Note** named **Sticky Note6** for the error-monitoring section.

These notes do not affect execution, but they help preserve the original workflow layout and intent.

---

### 4.2 Build the main polling branch

#### Step 1: Add the schedule trigger
1. Create a **Schedule Trigger** node.
2. Name it **Every 10 Seconds Trigger**.
3. Configure the rule:
   - Interval
   - Field: **seconds**
   - Seconds interval: **10**

#### Step 2: Add the SerpAPI query node
4. Create a **SerpAPI** node.
5. Name it **Fetch Live Cricket Score**.
6. Configure:
   - Query (`q`): **cricket live score india match**
   - Location: **India**
7. Attach a **SerpAPI credential**.
8. Connect:
   - **Every 10 Seconds Trigger** → **Fetch Live Cricket Score**

**Credential requirement:**  
Create or select a named SerpAPI credential using your API key.

---

### 4.3 Build the extraction and validation branch

#### Step 3: Add the Function node for match extraction
9. Create a **Function** node.
10. Name it **Extract Latest Match Data**.
11. Add logic equivalent to:
   - Read `sports_results.games`
   - Use the first game as the latest match
   - Return:
     - `team1`
     - `team2`
     - `score1`
     - `score2`
     - `result`
     - `tournament`

A safer implementation than the original should:
- verify `games.length > 0`
- verify `match.teams` exists and has two entries
- return an empty item or controlled error if no game exists

12. Connect:
   - **Fetch Live Cricket Score** → **Extract Latest Match Data**

#### Step 4: Add the validation If node
13. Create an **If** node.
14. Name it **Skip If No Match Result**.
15. Configure the condition:
   - Left value: `{{$json.result}}`
   - Operator: **is not empty**
16. Leave the false branch unconnected if you want the workflow to stop silently when no result exists.
17. Connect:
   - **Extract Latest Match Data** → **Skip If No Match Result**

---

### 4.4 Build the AI generation branch

#### Step 5: Add the AI Agent
18. Create an **AI Agent** node.
19. Name it **Generate AI Match Commentary**.
20. Set prompt type to **Define**.
21. Paste a prompt equivalent to the original:
   - Role: expert cricket analyst and commentator
   - Inputs:
     - `{{$json.team1}}`
     - `{{$json.team2}}`
     - `{{$json.tournament}}`
     - `{{$json.score1}}`
     - `{{$json.score2}}`
     - `{{$json.result}}`
   - Requested format:
     - Match Summary
     - Key Turning Point
     - Best Performance
     - Tactical Insight
   - Style rules:
     - short
     - crisp
     - expert tone
     - no generic AI phrases
     - engaging commentary
22. Set the node error behavior to **Continue to error output** if you want to preserve the original behavior.
23. Set **Retry on Fail** to **false**.
24. Connect:
   - **Skip If No Match Result** (true path) → **Generate AI Match Commentary**

#### Step 6: Add the OpenAI chat model
25. Create an **OpenAI Chat Model** node.
26. Name it **OpenAI GPT-4o-mini Model**.
27. Select model: **gpt-4o-mini**.
28. Attach an **OpenAI API credential**.
29. Connect the model to the AI Agent using the **AI language model** connection, not a normal main connection.

**Credential requirement:**  
Use an OpenAI API key with access to the selected model.

---

### 4.5 Build the delivery branch

#### Step 7: Add the Telegram delivery node
30. Create a **Telegram** node.
31. Name it **Send Commentary to Telegram**.
32. Configure it to send a text message:
   - Text: `{{$json.output}}`
   - Chat ID: replace `YOUR_TELEGRAM_CHAT_ID` with your actual chat or group ID
   - Parse mode: **Markdown**
33. Attach a **Telegram Bot** credential.
34. Connect:
   - **Generate AI Match Commentary** → **Send Commentary to Telegram**

**Credential requirement:**  
Use a Telegram bot token from BotFather. Ensure the bot is allowed to message the target chat or is added to the target group.

---

### 4.6 Build the Gmail alert branch

#### Step 8: Add the Gmail alert node
35. Create a **Gmail** node.
36. Name it **Email Alert on AI Failure**.
37. Configure:
   - To: your actual recipient address
   - Subject: **Error occurred in Cricket Score Workflow**
   - Message: a warning that AI commentary generation failed and logs should be checked
38. Attach a **Gmail OAuth2** credential.

#### Step 9: Replicate the original connection
39. Connect:
   - **Generate AI Match Commentary** → **Email Alert on AI Failure**

#### Step 10: Replicate the original loop connection
40. Connect:
   - **Email Alert on AI Failure** → **Fetch Live Cricket Score**

**Important warning:**  
This reproduces the JSON exactly, but it is not recommended. This connection can create unintended looping or confusing re-entry behavior. If your goal is reliable error handling, do this instead:
- add an **If** node after the AI Agent
- branch based on presence of `output` versus error fields
- send success to Telegram
- send failure to Gmail
- do **not** connect Gmail back to the fetch node

---

### 4.7 Build the workflow-level error branch

#### Step 11: Add the Error Trigger
41. Create an **Error Trigger** node.
42. Name it **Error Trigger**.

#### Step 12: Add the Slack alert node
43. Create a **Slack** node.
44. Name it **Alert on Workflow Failure**.
45. Configure it to post to a channel:
   - Select mode: **channel**
   - Choose the desired channel, ideally one dedicated to workflow alerts
46. Use a text message. The original says:
   - “⚠️ Hotel Pre-Arrival Workflow Error Detected”
   - “Please check the execution log for details.”
47. Connect:
   - **Error Trigger** → **Alert on Workflow Failure**

**Credential requirement:**  
Use a Slack app/bot credential authorized to post in the target channel.

**Recommended correction:**  
Replace the copied text with something accurate, for example:
- “⚠️ Cricket Score Commentary Workflow Error Detected”
- “Please check the execution log for details.”

---

### 4.8 Final workflow settings
48. Open workflow settings.
49. Set execution order to **v1** to match the exported workflow.
50. Keep the workflow **inactive** until all credentials and IDs are updated.
51. Test each branch manually:
   - SerpAPI response structure
   - extraction node output
   - AI generation
   - Telegram send
   - Gmail send
   - Slack error posting

---

### 4.9 Recommended hardening improvements after reconstruction
These are not part of the original JSON, but they are strongly advised:

1. **Replace the Function node with a defensive Code node**  
   Validate that:
   - `sports_results` exists
   - `games` is an array
   - the selected match has two teams

2. **Remove the Gmail-to-Fetch loop**  
   It serves no clear benefit and may produce unintended executions.

3. **Add explicit success/failure branching after the AI Agent**  
   If `output` exists → Telegram  
   If error payload exists → Gmail

4. **Deduplicate commentary updates**  
   Store the previous match status or score and only send when something changes.

5. **Sanitize Markdown before Telegram send**  
   AI output can sometimes contain characters that break Telegram Markdown formatting.

6. **Improve observability**  
   Include match name, tournament, and execution timestamp in email/Slack alerts.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is currently inactive. | Workflow property `active: false` |
| Main purpose: generate live cricket commentary from SerpAPI sports results and send it to Telegram. | Overall workflow intent |
| Polling frequency is every 10 seconds, which may be expensive for API usage and can generate repetitive messages. | Operational consideration |
| The Gmail alert node is described as “AI failure” handling, but the current implementation does not cleanly separate success from failure. | Design caveat |
| The Gmail node is connected back to the SerpAPI fetch node, creating a likely unintended loop. | Critical topology issue |
| The Slack alert text refers to “Hotel Pre-Arrival Workflow Error Detected,” which does not match this workflow and should be corrected. | Copy/paste inconsistency |
| Security guidance from the workflow notes: use named credentials for SerpAPI, OpenAI, Telegram, Gmail, and Slack; never hardcode tokens or API keys. | Security best practice |
| No sub-workflows are used in this workflow. | Architecture note |
| No external links are present in the sticky notes. | Documentation note |

If you want, I can also produce a **corrected version analysis** showing how this workflow should be rewired for proper AI-error branching and duplicate-message prevention.