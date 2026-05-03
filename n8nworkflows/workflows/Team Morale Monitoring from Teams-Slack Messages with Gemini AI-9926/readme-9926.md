Team Morale Monitoring from Teams/Slack Messages with Gemini AI

https://n8nworkflows.xyz/workflows/team-morale-monitoring-from-teams-slack-messages-with-gemini-ai-9926


# Team Morale Monitoring from Teams/Slack Messages with Gemini AI

### 1. Workflow Overview

This workflow is designed to monitor team morale by analyzing the sentiment and emotional tone of messages exchanged within Microsoft Teams (and optionally Slack). Its primary target users are team leads, HR professionals, and managers who want to gain actionable insights into their team’s psychological state through automated sentiment analysis driven by AI (Google Gemini) models.

The workflow operates on a weekly schedule, retrieving recent messages, analyzing their emotional content, aggregating the results, and generating a detailed morale report that is automatically posted to Slack.

Logical blocks:

- **1.1 Trigger and Configuration:** Scheduling the workflow and defining team/channel parameters.
- **1.2 Message Retrieval:** Fetching recent Teams messages for analysis (currently simulated).
- **1.3 Batch Processing & Sentiment Analysis:** Splitting messages into manageable batches and analyzing each message’s sentiment and emotional indicators with Gemini AI.
- **1.4 Aggregation & Statistics:** Combining individual message analyses to calculate weekly average sentiment and stress metrics.
- **1.5 Report Generation:** Using Gemini AI to produce a narrative morale report based on aggregated data.
- **1.6 Report Distribution:** Sending the generated morale report to a Slack channel for managerial visibility.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Configuration

- **Overview:** Initiates the workflow every Monday at 9 AM and sets essential parameters such as team and channel IDs.
- **Nodes Involved:**  
  - Weekly Morale Check Trigger  
  - Workflow Configuration

- **Node Details:**

  - **Weekly Morale Check Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow weekly (Monday 9 AM).  
    - Config: Fires at weeks interval, specifically Monday at 09:00.  
    - Connections: Outputs to Workflow Configuration.  
    - Edge cases: Misconfiguration could lead to trigger not firing; time zone discrepancies may affect timing.

  - **Workflow Configuration**  
    - Type: Set  
    - Role: Defines static parameters: Team ID, Channel ID, Manager User ID, days to analyze (7).  
    - Config: Contains placeholders `<YOUR_TEAM_ID>`, `<YOUR_CHANNEL_ID>`, `<MANAGER_USER_ID>`.  
    - Connections: Outputs to Fetch Teams Messages (Simulated)1.  
    - Edge cases: Incorrect or missing IDs will cause downstream data retrieval or posting failures.

---

#### 1.2 Message Retrieval

- **Overview:** Collects recent Microsoft Teams messages for the defined period. Currently uses a simulated data node for demonstration.
- **Nodes Involved:**  
  - Fetch Teams Messages (Simulated)1

- **Node Details:**

  - **Fetch Teams Messages (Simulated)1**  
    - Type: Code (JavaScript)  
    - Role: Placeholder node returning hardcoded sample messages with authors, timestamps, and text in Japanese.  
    - Config: Returns 3 sample messages with current timestamps.  
    - Connections: Outputs to Split In Batches.  
    - Edge cases: As a simulated node, lacks real API integration—must be replaced with actual Microsoft Graph API calls for production. Potential API auth errors, rate limits, or empty datasets in real use.

---

#### 1.3 Batch Processing & Sentiment Analysis

- **Overview:** Processes messages in batches to manage load and performs detailed sentiment and emotional analysis on each message using Gemini AI.
- **Nodes Involved:**  
  - Split In Batches  
  - Sentiment Analysis (OpenAI)1  
  - Log Progress1

- **Node Details:**

  - **Split In Batches**  
    - Type: Split In Batches  
    - Role: Divides message array into batches of 5 for sequential processing.  
    - Config: Batch size = 5.  
    - Connections: Takes input from Fetch Teams Messages; outputs to Sentiment Analysis.  
    - Edge cases: Batch size too large may cause timeouts; too small may increase execution time.

  - **Sentiment Analysis (OpenAI)1**  
    - Type: Google Gemini AI (LangChain node)  
    - Role: Analyzes each message’s sentiment, emotion, stress, engagement, key phrases, and concerns, producing structured JSON output.  
    - Config: Prompt instructs AI to output a JSON with scores and fields for each message using Gemini-2.5-flash model.  
    - Credentials: Uses Google Gemini (PaLM) API account.  
    - Connections: Outputs to Log Progress1.  
    - Edge cases: API authentication failure, rate limits, malformed response, or prompt failures can occur. Requires robust error handling.

  - **Log Progress1**  
    - Type: Code (JavaScript)  
    - Role: Logs batch processing progress to console for monitoring.  
    - Config: Runs once per item, logs "Batch processed".  
    - Connections: Outputs to Aggregate Sentiment Scores.  
    - Edge cases: Minimal risk; mainly for operational visibility.

---

#### 1.4 Aggregation & Statistics

- **Overview:** Aggregates all individual sentiment analyses to compute team-wide weekly averages for sentiment and stress.
- **Nodes Involved:**  
  - Aggregate Sentiment Scores  
  - Calculate Weekly Statistics

- **Node Details:**

  - **Aggregate Sentiment Scores**  
    - Type: Aggregate  
    - Role: Collects all processed message data into a single array for aggregate calculation.  
    - Config: Aggregates all item data (no grouping).  
    - Connections: Outputs to Calculate Weekly Statistics.  
    - Edge cases: Empty input array leads to division by zero or invalid averages; requires input validation.

  - **Calculate Weekly Statistics**  
    - Type: Code (JavaScript)  
    - Role: Calculates average sentiment and stress scores, and counts messages.  
    - Config: Reduces aggregated data to averages with two decimals.  
    - Connections: Outputs to Generate Morale Report (OpenAI)1.  
    - Edge cases: Division by zero if no messages; must handle empty inputs gracefully.

---

#### 1.5 Report Generation

- **Overview:** Generates a detailed morale report narrative based on aggregated sentiment data, using Gemini AI.
- **Nodes Involved:**  
  - Generate Morale Report (OpenAI)1

- **Node Details:**

  - **Generate Morale Report (OpenAI)1**  
    - Type: Google Gemini AI (LangChain node)  
    - Role: Produces a nuanced team morale report for managers, referencing individual message analyses without citing averages or counts.  
    - Config: Uses a detailed prompt instructing the AI to act as an organizational behavior consultant analyzing the team’s messages.  
    - Model: Gemini-2.5-flash.  
    - Connections: Outputs to Send a message1.  
    - Edge cases: Same AI integration risks as Sentiment Analysis node; prompt must be maintained for context accuracy.

---

#### 1.6 Report Distribution

- **Overview:** Sends the generated morale report to a Slack channel for managerial review and team transparency.
- **Nodes Involved:**  
  - Send a message1

- **Node Details:**

  - **Send a message1**  
    - Type: Slack node  
    - Role: Posts the morale report text into a specified Slack channel.  
    - Config: Uses OAuth2 authentication, posts to channel ID `C09LUABPRGT` (can be customized). Text content is the AI-generated report.  
    - Connections: Terminal node (outputs none).  
    - Edge cases: Slack API auth issues, invalid channel ID, or message size limits may cause failure.

---

### 3. Summary Table

| Node Name                    | Node Type                      | Functional Role                    | Input Node(s)                      | Output Node(s)                 | Sticky Note                                                                                                                    |
|------------------------------|--------------------------------|----------------------------------|----------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Weekly Morale Check Trigger   | Schedule Trigger               | Starts workflow weekly           | None                             | Workflow Configuration         | ### Weekly Trigger<br>Activates automatically every Monday at 9:00 AM.<br>You can change this schedule in the node settings.   |
| Workflow Configuration        | Set                           | Defines IDs and parameters       | Weekly Morale Check Trigger      | Fetch Teams Messages (Simulated)1 | ### Workflow Configuration<br>Defines IDs and settings such as:<br>- Team ID<br>- Channel ID<br>- Manager User ID<br>- Analysis Period (days) |
| Fetch Teams Messages (Simulated)1 | Code (JavaScript)           | Simulates message fetching       | Workflow Configuration           | Split In Batches               | ### Fetch Messages<br>Collects recent Microsoft Teams messages.<br>Replace this with a real API node for live data.            |
| Split In Batches              | Split In Batches               | Batches messages for processing | Fetch Teams Messages (Simulated)1 | Sentiment Analysis (OpenAI)1  |                                                                                                                               |
| Sentiment Analysis (OpenAI)1 | Google Gemini AI (LangChain)  | Analyzes message sentiment       | Split In Batches                 | Log Progress1                 | ### Sentiment Analysis<br>Uses Gemini or OpenAI to assess sentiment, stress, and engagement per message.                       |
| Log Progress1                | Code (JavaScript)              | Logs batch processing progress   | Sentiment Analysis (OpenAI)1     | Aggregate Sentiment Scores     |                                                                                                                               |
| Aggregate Sentiment Scores    | Aggregate                     | Aggregates sentiment data        | Log Progress1                   | Calculate Weekly Statistics    | ### Aggregate Statistics<br>Calculates weekly averages of sentiment and stress indicators.                                     |
| Calculate Weekly Statistics   | Code (JavaScript)              | Computes averages and counts     | Aggregate Sentiment Scores       | Generate Morale Report (OpenAI)1 |                                                                                                                               |
| Generate Morale Report (OpenAI)1 | Google Gemini AI (LangChain) | Creates morale report narrative  | Calculate Weekly Statistics      | Send a message1               | ### Generate Report<br>Summarizes emotional insights into a readable report for managers.                                      |
| Send a message1              | Slack                         | Posts report to Slack channel    | Generate Morale Report (OpenAI)1 | None                         | ### Post to Slack<br>Publishes the morale report to Slack or Teams for visibility.                                             |
| Template Overview            | Sticky Note                   | Documentation overview           | None                             | None                         | ## AI Team Morale Monitor<br>For team leads, HR, and managers who want to monitor the emotional tone and morale of their teams based on message sentiment.<br>... |
| Trigger Description          | Sticky Note                   | Trigger explanation              | None                             | None                         | ### Weekly Trigger<br>Activates automatically every Monday at 9:00 AM.<br>You can change this schedule in the node settings.   |
| Configuration Description   | Sticky Note                   | Configuration explanation       | None                             | None                         | ### Workflow Configuration<br>Defines IDs and settings such as:<br>- Team ID<br>- Channel ID<br>- Manager User ID<br>- Analysis Period (days) |
| Fetch Messages Description  | Sticky Note                   | Fetch messages explanation      | None                             | None                         | ### Fetch Messages<br>Collects recent Microsoft Teams messages.<br>Replace this with a real API node for live data.            |
| AI Analysis Description     | Sticky Note                   | AI analysis explanation         | None                             | None                         | ### Sentiment Analysis<br>Uses Gemini or OpenAI to assess sentiment, stress, and engagement per message.                       |
| Aggregate Description       | Sticky Note                   | Aggregate statistics explanation | None                             | None                         | ### Aggregate Statistics<br>Calculates weekly averages of sentiment and stress indicators.                                     |
| Report Description          | Sticky Note                   | Report generation explanation   | None                             | None                         | ### Generate Report<br>Summarizes emotional insights into a readable report for managers.                                      |
| Slack Description           | Sticky Note                   | Slack posting explanation       | None                             | None                         | ### Post to Slack<br>Publishes the morale report to Slack or Teams for visibility.                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: Weekly Morale Check Trigger**  
   - Type: Schedule Trigger  
   - Set to trigger every week on Monday at 9:00 AM.

2. **Create Configuration Node: Workflow Configuration**  
   - Type: Set  
   - Add parameters:  
     - `teamId` (string): `<YOUR_TEAM_ID>`  
     - `channelId` (string): `<YOUR_CHANNEL_ID>`  
     - `managerUserId` (string): `<MANAGER_USER_ID>`  
     - `daysToAnalyze` (number): `7`  
   - Connect trigger node output to this node.

3. **Create Message Fetch Node: Fetch Teams Messages (Simulated)**  
   - Type: Code  
   - Paste JavaScript code to simulate fetching messages with fields: text, author, createdDateTime.  
   - Connect Workflow Configuration output to this node.

4. **Create Batch Processing Node: Split In Batches**  
   - Type: Split In Batches  
   - Set Batch Size: 5  
   - Connect Fetch Messages output to this node.

5. **Create AI Sentiment Analysis Node: Sentiment Analysis (OpenAI)**  
   - Type: Google Gemini (LangChain node)  
   - Configure prompt to analyze message sentiment, stress, engagement, key phrases, concerns, and message text in JSON format.  
   - Set model to `models/gemini-2.5-flash`.  
   - Provide credentials for Google Gemini (PaLM) API.  
   - Connect Split In Batches output to this node.

6. **Create Logging Node: Log Progress**  
   - Type: Code  
   - JavaScript: Log "Batch processed" for each item.  
   - Connect Sentiment Analysis output to this node.

7. **Create Aggregation Node: Aggregate Sentiment Scores**  
   - Type: Aggregate  
   - Set to aggregate all input data into a single dataset.  
   - Connect Log Progress output to this node.

8. **Create Statistics Calculation Node: Calculate Weekly Statistics**  
   - Type: Code  
   - JavaScript to compute average sentiment and stress values from aggregated data; also count messages.  
   - Connect Aggregate output to this node.

9. **Create AI Report Generation Node: Generate Morale Report (OpenAI)**  
   - Type: Google Gemini (LangChain node)  
   - Prompt instructs AI to act as an organizational behavior consultant generating a detailed report based on aggregated data without citing averages or counts.  
   - Use model `models/gemini-2.5-flash`.  
   - Connect Calculate Weekly Statistics output to this node.

10. **Create Slack Posting Node: Send a message**  
    - Type: Slack  
    - Configure to post message text from AI report output (`{{ $json.content.parts[0].text }}`) to the team’s Slack channel.  
    - Use OAuth2 Slack credentials.  
    - Connect Report Generation output to this node.

11. **Set up all credentials:**  
    - Microsoft Teams (for real implementation replacing simulated fetch).  
    - Google Gemini API with LangChain integration.  
    - Slack OAuth2 authentication.

12. **Test workflow:**  
    - Run manually or wait for scheduled trigger.  
    - Verify each step processes data correctly.  
    - Adjust configuration and credentials as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow uses *only* linguistic data — no personal identifiers or private metadata are processed.                                                                                                                                                                                                                     | Template Overview sticky note                                                                        |
| Replace the simulated message fetch node with a real Microsoft Graph API integration for production use.                                                                                                                                                                                                                   | Fetch Messages Description sticky note                                                             |
| The AI models used are Google Gemini 2.5 Flash accessed through LangChain nodes; credentials are required and must be maintained securely.                                                                                                                                                                                | AI Analysis Description sticky note                                                                |
| Posting to Slack requires OAuth2 setup with appropriate permissions to send messages to channels.                                                                                                                                                                                                                          | Slack Description sticky note                                                                       |
| The workflow runs weekly but schedule can be modified in the trigger node settings.                                                                                                                                                                                                                                         | Trigger Description sticky note                                                                     |
| The morale report prompt is carefully crafted to avoid referencing data counts or averages explicitly, focusing instead on qualitative insights. Modifying the AI prompt allows customization of report style and depth.                                                                                                  | Report Description sticky note                                                                      |
| For further customization, users can swap Gemini AI with other LLM providers supported by n8n’s LangChain integration or adjust the Slack posting node to use Microsoft Teams or other communication platforms.                                                                                                              | Template Overview sticky note                                                                        |

---

**Disclaimer:** The provided content is exclusively generated from an n8n automated workflow. It complies with all applicable content policies and contains no illegal, offensive, or protected elements. All processed data are legal and public.