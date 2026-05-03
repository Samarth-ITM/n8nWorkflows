Daily Space Quiz Bot: NASA APOD to Slack with GPT-4 Turbo

https://n8nworkflows.xyz/workflows/daily-space-quiz-bot--nasa-apod-to-slack-with-gpt-4-turbo-10994


# Daily Space Quiz Bot: NASA APOD to Slack with GPT-4 Turbo

### 1. Workflow Overview

This workflow automates a daily space-themed quiz posted to a Slack channel. It leverages NASA’s Astronomy Picture of the Day (APOD) as the quiz context, uses OpenAI’s GPT-4 Turbo model to generate quiz questions, and manages interactive voting and results announcement within Slack.

**Target Use Cases:**  
- Space education and engagement communities on Slack  
- Daily interactive trivia bots for team-building or learning  
- Automated content generation using AI with external API integration

**Logical Blocks:**

- **1.1 Trigger & NASA Data Fetch:** Schedule-trigger node initiates the workflow daily and fetches NASA APOD data.  
- **1.2 Workflow Configuration:** Central node holding configurable parameters such as Slack channel, quiz difficulty, and timing.  
- **1.3 Quiz Generation (AI Processing):** Uses OpenAI’s GPT-4 Turbo to create a multiple-choice quiz question based on NASA APOD info.  
- **1.4 Quiz Posting & Reaction Setup:** Posts quiz to Slack channel and adds reaction emojis as voting buttons.  
- **1.5 Wait & Collect Votes:** Waits for a configured time, then retrieves user reactions (votes) from Slack.  
- **1.6 Tally Results & Announce Winners:** Processes reactions to identify correct answers and winning users, then posts the results back to Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & NASA Data Fetch

- **Overview:**  
  Triggers the workflow daily at 21:00 and fetches the latest NASA Astronomy Picture of the Day (APOD) data.

- **Nodes Involved:**  
  - Schedule: Daily at 21:00  
  - Get APOD

- **Node Details:**  

  - **Schedule: Daily at 21:00**  
    - Type: Schedule Trigger  
    - Configuration: Triggers once daily at 21:00 hours  
    - Inputs: None (trigger node)  
    - Outputs: Triggers next node "Get APOD"  
    - Edge Cases: Missed triggers if n8n server downtime; time zone considerations may affect trigger time.

  - **Get APOD**  
    - Type: NASA API Node  
    - Configuration: No additional fields set; defaults to latest APOD  
    - Inputs: Trigger from schedule  
    - Outputs: NASA APOD data including title, explanation, image URL  
    - Edge Cases: API rate limits or unavailability; invalid or missing data from NASA API.

#### 1.2 Workflow Configuration

- **Overview:**  
  Holds core configuration parameters such as Slack channel ID, quiz difficulty, response length, tone, and timeout duration.

- **Nodes Involved:**  
  - Workflow Configuration

- **Node Details:**  

  - **Workflow Configuration**  
    - Type: Set Node  
    - Configuration: Assigns variables:  
      - quizDifficulty: "beginner"  
      - explanationMaxChar: 400  
      - llmTone: "friendly and engaging"  
      - channelId: "YOUR_SLACK_CHANNEL_ID" (to be replaced)  
      - answerTimeoutMin: 15 (minutes)  
    - Inputs: Receives APOD data from previous node  
    - Outputs: Provides config variables to the AI node  
    - Edge Cases: Misconfigured channel ID leads to posting failure; invalid parameter types cause expression errors.

#### 1.3 Quiz Generation (AI Processing)

- **Overview:**  
  Uses OpenAI GPT-4 Turbo to generate a single multiple-choice quiz question derived from NASA APOD data with specific formatting and tone.

- **Nodes Involved:**  
  - OpenAI: Create Quiz

- **Node Details:**  

  - **OpenAI: Create Quiz**  
    - Type: OpenAI (LangChain) Node  
    - Configuration:  
      - Model: gpt-4-turbo  
      - Prompt: Detailed instructions to generate one JSON object quiz question with 4 options (A-D), concise explanation under configured max characters, tone, and difficulty from Workflow Configuration.  
      - Output format: JSON object with keys (question, choices, answer, explanation, source_url)  
    - Inputs: Receives APOD data and config variables  
    - Outputs: JSON quiz question passed to Slack posting node  
    - Edge Cases: API timeouts, malformed JSON response requiring error handling, token limits for prompt length.

#### 1.4 Quiz Posting & Reaction Setup

- **Overview:**  
  Posts the generated quiz question to the configured Slack channel and adds four emoji reactions (1️⃣ to 4️⃣) to act as voting buttons.

- **Nodes Involved:**  
  - Slack: Post Quiz  
  - Slack: Add Reaction 1  
  - Slack: Add Reaction 2  
  - Slack: Add Reaction 3  
  - Slack: Add Reaction 4

- **Node Details:**  

  - **Slack: Post Quiz**  
    - Type: Slack Node (message posting)  
    - Configuration:  
      - Channel ID from Workflow Configuration  
      - Message text formatted with quiz question, choices, voting instructions, and source image link  
      - Unfurl links enabled  
    - Inputs: Receives quiz JSON from OpenAI node  
    - Outputs: Slack message metadata (timestamp, channel) used for reactions and later queries  
    - Edge Cases: Slack API rate limits or invalid channel ID, formatting issues if JSON parsing fails.

  - **Slack: Add Reaction 1-4**  
    - Type: Slack Node (reaction add)  
    - Configuration for each:  
      - Reaction names: "one", "two", "three", "four" respectively  
      - Channel ID from Workflow Configuration  
      - Timestamp from Slack: Post Quiz node message metadata  
    - Inputs: Chain-connected sequentially after Slack: Post Quiz and previous reaction nodes  
    - Outputs: None (used for adding emoji reactions)  
    - Edge Cases: Reaction failure if message timestamp invalid or Slack permissions missing; race conditions if reactions added too fast.

#### 1.5 Wait & Collect Votes

- **Overview:**  
  Waits for configured timeout duration to allow users to vote by reacting, then collects reactions from Slack for tallying.

- **Nodes Involved:**  
  - Wait for Answers  
  - Slack: Get Reactions

- **Node Details:**  

  - **Wait for Answers**  
    - Type: Wait Node  
    - Configuration: Waits for number of minutes defined by `answerTimeoutMin` (default 15)  
    - Inputs: After all reactions added  
    - Outputs: Triggers next node to collect reactions  
    - Edge Cases: Workflow execution pauses; if workflow is disabled or interrupted, tally will not proceed.

  - **Slack: Get Reactions**  
    - Type: Slack Node (reaction retrieval)  
    - Configuration:  
      - Resource: reaction  
      - Channel ID and message timestamp from Slack: Post Quiz node  
    - Inputs: Output from Wait node  
    - Outputs: List of reactions and user IDs who reacted  
    - Edge Cases: Slack API limits, invalid message timestamp, or reaction data missing.

#### 1.6 Tally Results & Announce Winners

- **Overview:**  
  Processes Slack reactions to identify users who selected the correct answer, excluding bot’s own reactions, counts participants, and posts results with winners.

- **Nodes Involved:**  
  - Code: Tally Correct Answers  
  - Slack: Post Results

- **Node Details:**  

  - **Code: Tally Correct Answers**  
    - Type: Code Node (JavaScript)  
    - Configuration:  
      - Extracts correct answer letter from OpenAI quiz data  
      - Maps Slack emoji names ("one", "two", "three", "four") to quiz options (A-D)  
      - Filters out bot user ID from reactions  
      - Counts unique users who reacted with correct answer and total participants  
    - Inputs: Reaction data from Slack: Get Reactions, quiz data from OpenAI node, Slack post user ID  
    - Outputs: JSON object with quiz data, correct answer, list of correct users, counts  
    - Edge Cases: Missing or malformed reaction data, bot user ID not found, emoji name mismatch.

  - **Slack: Post Results**  
    - Type: Slack Node (message posting)  
    - Configuration:  
      - Posts summary message including correct answer, explanation, and mentions winners (or fallback text if no winners)  
      - Uses channel from original quiz post for posting results  
    - Inputs: Data from Code node  
    - Outputs: Final workflow completion  
    - Edge Cases: Slack post failure, mention formatting issues, empty winners list.

---

### 3. Summary Table

| Node Name             | Node Type                      | Functional Role                  | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                           |
|-----------------------|--------------------------------|---------------------------------|-----------------------------|------------------------------|-----------------------------------------------------------------------------------------------------|
| Schedule: Daily at 21:00 | Schedule Trigger              | Initiates daily workflow trigger | None                        | Get APOD                     |                                                                                                     |
| Get APOD              | NASA API Node                  | Fetches NASA Astronomy Picture of the Day | Schedule: Daily at 21:00    | Workflow Configuration       |                                                                                                     |
| Workflow Configuration | Set Node                      | Stores configurable parameters   | Get APOD                    | OpenAI: Create Quiz          | ## Workflow Configuration<br>Set Slack channel, quiz difficulty, and main variables here.           |
| OpenAI: Create Quiz    | OpenAI GPT-4 Turbo Node        | Generates quiz question & answers | Workflow Configuration      | Slack: Post Quiz             | ## AI Quiz Generation<br>Prompt generates quiz JSON with tone, difficulty, and output format.       |
| Slack: Post Quiz       | Slack Message Post Node        | Posts quiz message to Slack channel | OpenAI: Create Quiz          | Slack: Add Reaction 1        |                                                                                                     |
| Slack: Add Reaction 1  | Slack Reaction Add Node        | Adds reaction "one" emoji for voting | Slack: Post Quiz             | Slack: Add Reaction 2        | ## Add Voting Buttons<br>Bot adds 1️⃣, 2️⃣, 3️⃣, 4️⃣ reactions to quiz message automatically.      |
| Slack: Add Reaction 2  | Slack Reaction Add Node        | Adds reaction "two" emoji         | Slack: Add Reaction 1        | Slack: Add Reaction 3        |                                                                                                     |
| Slack: Add Reaction 3  | Slack Reaction Add Node        | Adds reaction "three" emoji       | Slack: Add Reaction 2        | Slack: Add Reaction 4        |                                                                                                     |
| Slack: Add Reaction 4  | Slack Reaction Add Node        | Adds reaction "four" emoji        | Slack: Add Reaction 3        | Wait for Answers             |                                                                                                     |
| Wait for Answers       | Wait Node                     | Waits for voting period to elapse | Slack: Add Reaction 4        | Slack: Get Reactions         |                                                                                                     |
| Slack: Get Reactions   | Slack Reaction Get Node        | Retrieves user reactions          | Wait for Answers            | Code: Tally Correct Answers  |                                                                                                     |
| Code: Tally Correct Answers | Code Node (JavaScript)       | Tallies correct votes excluding bot | Slack: Get Reactions         | Slack: Post Results          | ## Tally Correct Answers<br>Filters bot reactions and counts unique correct user votes.             |
| Slack: Post Results    | Slack Message Post Node        | Posts quiz results and winners    | Code: Tally Correct Answers | None                        |                                                                                                     |
| Sticky Note            | Sticky Note                   | Documentation note                | None                        | None                        | ## Daily Space Quiz Bot<br>Workflow overview and setup instructions.                                |
| Sticky Note1           | Sticky Note                   | Documentation note                | None                        | None                        | ## Workflow Configuration<br>Explains configuration variables node.                                |
| Sticky Note2           | Sticky Note                   | Documentation note                | None                        | None                        | ## Add Voting Buttons<br>Describes automatic addition of reaction emojis for voting.                |
| Sticky Note3           | Sticky Note                   | Documentation note                | None                        | None                        | ## Tally Correct Answers<br>Explanation of vote tally code node.                                   |
| Sticky Note4           | Sticky Note                   | Documentation note                | None                        | None                        | ## AI Quiz Generation<br>Explains AI prompt and customization.                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n** named "Daily Space Quiz Bot with NASA APOD Integration".

2. **Add a Schedule Trigger node**  
   - Name: "Schedule: Daily at 21:00"  
   - Set trigger rule to fire daily at 21:00 hours (local timezone or UTC as preferred).

3. **Add NASA node to fetch APOD**  
   - Name: "Get APOD"  
   - Connect Schedule Trigger output to this node input.  
   - Use NASA API credentials configured in n8n credentials manager.  
   - Leave additional fields empty to fetch latest APOD.

4. **Add a Set node for workflow configuration**  
   - Name: "Workflow Configuration"  
   - Connect "Get APOD" output to this node input.  
   - Add string fields and numeric fields:  
     - quizDifficulty: "beginner"  
     - explanationMaxChar: 400  
     - llmTone: "friendly and engaging"  
     - channelId: "YOUR_SLACK_CHANNEL_ID" (replace with actual Slack channel ID)  
     - answerTimeoutMin: 15

5. **Add OpenAI node for quiz generation**  
   - Name: "OpenAI: Create Quiz"  
   - Connect "Workflow Configuration" output to this node input.  
   - Set model to "gpt-4-turbo".  
   - Use API credentials for OpenAI (API key).  
   - Set prompt with detailed instructions:  
     - Include NASA APOD title, explanation, image URL from "Get APOD" node.  
     - Configure tone and difficulty from "Workflow Configuration".  
     - Request output as strict JSON with keys: question, choices (A-D), answer, explanation, source_url.  
     - Limit explanation length as configured.

6. **Add Slack node to post quiz**  
   - Name: "Slack: Post Quiz"  
   - Connect "OpenAI: Create Quiz" output to this node input.  
   - Use Slack API credentials with chat:write and reactions permissions.  
   - Set channel to the value from "Workflow Configuration" (`channelId`).  
   - Construct message text using expressions to pull quiz question and choices from AI output JSON.  
   - Enable unfurl links for source image.

7. **Add four Slack nodes to add reactions**  
   - Names: "Slack: Add Reaction 1", "Slack: Add Reaction 2", "Slack: Add Reaction 3", "Slack: Add Reaction 4"  
   - Connect sequentially, starting from "Slack: Post Quiz".  
   - For each node, configure reaction names: "one", "two", "three", "four" respectively.  
   - Use channel ID from "Workflow Configuration" and timestamp from "Slack: Post Quiz" node.  
   - These reactions serve as voting buttons.

8. **Add Wait node to pause for voting**  
   - Name: "Wait for Answers"  
   - Connect from last reaction node output.  
   - Set wait duration to minutes equal to `answerTimeoutMin` from "Workflow Configuration".

9. **Add Slack node to get reactions**  
   - Name: "Slack: Get Reactions"  
   - Connect from "Wait for Answers".  
   - Configure to get reactions for the message posted in "Slack: Post Quiz" using channel ID and timestamp from that node.

10. **Add Code node to tally votes**  
    - Name: "Code: Tally Correct Answers"  
    - Connect from "Slack: Get Reactions".  
    - Paste JavaScript code that:  
      - Extracts correct answer from AI quiz data.  
      - Maps Slack emoji names to quiz choices.  
      - Filters out bot user ID from reaction user lists.  
      - Counts unique users who voted correctly and total participants.  
      - Returns JSON with quiz data, correct users, counts.

11. **Add Slack node to post results**  
    - Name: "Slack: Post Results"  
    - Connect from Code node.  
    - Configure message to announce:  
      - Time’s up notice.  
      - Correct answer with explanation.  
      - Mentions winners or fallback message if none.  
      - Link to original NASA image.  
    - Post in the same channel as quiz.

12. **Configure Credentials:**  
    - NASA API key in NASA node credentials.  
    - OpenAI API key in OpenAI node credentials.  
    - Slack OAuth2 with chat:write, reactions:write, reactions:read, channels:read, chat:write.public scopes for Slack nodes.

13. **Test workflow:**  
    - Run manually once to verify each step works.  
    - Check Slack for quiz post, reaction buttons, and results posting after wait.

14. **Activate workflow** to run daily automatically.

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow integrates NASA APOD, OpenAI GPT-4 Turbo, and Slack to create an interactive daily space quiz bot.               | Overview for users and developers.                                                                        |
| Setup requires valid API credentials for NASA, OpenAI, and Slack with proper permissions/scopes for posting and reactions.    | Refer to NASA API docs, OpenAI API docs, Slack API (chat and reactions scopes).                           |
| Slack message reactions (1️⃣ to 4️⃣) are used as voting buttons; users vote by clicking emoji reactions.                      | Slack UX design choice for simplicity.                                                                    |
| The code node excludes bot's own reactions to avoid skewing results.                                                           | Important for accurate tallying of human participants.                                                    |
| Maximum explanation length is configurable to keep quiz concise and engaging.                                                  | Controlled by `explanationMaxChar` parameter.                                                             |
| The AI prompt is customizable for tone, difficulty, and output format.                                                         | Allows flexible quiz style changes without workflow redesign.                                            |
| NASA APOD data includes title, explanation, and image URL used as quiz context and source reference.                           | Adds educational value and credibility.                                                                   |
| Slack channel ID must be replaced with actual channel where the quiz will be posted.                                            | Essential setup step before activation.                                                                    |
| For detailed Slack API permission setup and OAuth flow, see https://api.slack.com/authentication/oauth-v2                     | Slack developer documentation.                                                                             |

---

**Disclaimer:** The text provided is generated exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.