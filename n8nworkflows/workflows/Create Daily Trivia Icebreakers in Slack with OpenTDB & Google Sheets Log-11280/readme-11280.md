Create Daily Trivia Icebreakers in Slack with OpenTDB & Google Sheets Log

https://n8nworkflows.xyz/workflows/create-daily-trivia-icebreakers-in-slack-with-opentdb---google-sheets-log-11280


# Create Daily Trivia Icebreakers in Slack with OpenTDB & Google Sheets Log

### 1. Workflow Overview

This workflow automates the delivery of a daily trivia question as a Slack message while simultaneously logging the trivia data into a Google Sheets spreadsheet for archival and future use. It is designed for teams or communities seeking a fun daily icebreaker and for creators wanting to build a trivia question database over time.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Difficulty Selection:** A scheduled trigger activates the workflow once per day at a set time. A node randomly selects the trivia difficulty level (easy, medium, or hard).

- **1.2 Difficulty-Based Trivia Fetching:** Using a switch node, the flow routes to one of three HTTP request nodes querying the Open Trivia Database API with the selected difficulty level.

- **1.3 Trivia Data Normalization:** Each difficulty branch processes and reformats the API response into a unified trivia data structure, including metadata and formatted Slack message content.

- **1.4 Merging Trivia Streams:** The three difficulty branches merge back into a single stream.

- **1.5 Slack Posting and Google Sheets Logging:** The merged trivia item is posted as a formatted message to a Slack channel and appended as a new row in a Google Sheets spreadsheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Difficulty Selection

**Overview:**  
This block sets the workflow’s start time and randomly determines the trivia difficulty for the daily question.

**Nodes Involved:**  
- Schedule Trigger  
- Set: Choose Trivia Type  
- Switch: triviaType

**Node Details:**

- **Schedule Trigger**  
  - Type: Trigger node  
  - Role: Initiates the workflow at a fixed daily time (08:00 AM)  
  - Configuration: Interval trigger set to trigger once per day at 8:00 AM  
  - Inputs: None (trigger node)  
  - Outputs: One output connected to "Set: Choose Trivia Type"  
  - Failures: Possible failure if n8n server is down or scheduler misconfigured

- **Set: Choose Trivia Type**  
  - Type: Set node  
  - Role: Assigns a random difficulty value (`easy`, `medium`, or `hard`) to the item  
  - Configuration: Sets a single string field `difficulty` with a random choice from the three difficulties using JavaScript expression: `['easy','medium','hard'][Math.floor(Math.random() * 3)]`  
  - Inputs: From Schedule Trigger  
  - Outputs: Feeds into Switch: triviaType  
  - Edge cases: Random function could theoretically be biased; no fallback if empty or invalid difficulty generated (unlikely)

- **Switch: triviaType**  
  - Type: Switch node  
  - Role: Routes execution based on the `difficulty` value  
  - Configuration: Checks string value of `difficulty` field, routes to one of three outputs (easy, medium, hard) depending on value  
  - Inputs: From Set: Choose Trivia Type  
  - Outputs: Three outputs connected respectively to HTTP OpenTDB Easy, Medium, and Hard nodes  
  - Edge cases: If difficulty is not exactly one of the three values, no route will be taken, causing workflow to stop silently

---

#### 2.2 Difficulty-Based Trivia Fetching

**Overview:**  
Each branch queries the Open Trivia Database API for one multiple-choice trivia question of the specified difficulty.

**Nodes Involved:**  
- HTTP OpenTDB Easy  
- HTTP OpenTDB Medium  
- HTTP OpenTDB Hard

**Node Details:**

- **HTTP OpenTDB Easy**  
  - Type: HTTP Request node  
  - Role: Fetches one easy difficulty multiple-choice question from OpenTDB  
  - Configuration: URL `https://opentdb.com/api.php?amount=1&difficulty=easy&type=multiple`  
  - Inputs: From Switch: triviaType (easy output)  
  - Outputs: JSON response passed to Set Message Easy  
  - Failures: Network errors, API downtime, malformed response, rate limits

- **HTTP OpenTDB Medium**  
  - Type: HTTP Request node  
  - Role: Fetches one medium difficulty multiple-choice question  
  - Configuration: URL `https://opentdb.com/api.php?amount=1&difficulty=medium&type=multiple`  
  - Inputs: From Switch: triviaType (medium output)  
  - Outputs: JSON response passed to Set Message Medium  
  - Failures: Same as HTTP OpenTDB Easy

- **HTTP OpenTDB Hard**  
  - Type: HTTP Request node  
  - Role: Fetches one hard difficulty multiple-choice question  
  - Configuration: URL `https://opentdb.com/api.php?amount=1&difficulty=hard&type=multiple`  
  - Inputs: From Switch: triviaType (hard output)  
  - Outputs: JSON response passed to Set Message Hard  
  - Failures: Same as HTTP OpenTDB Easy

---

#### 2.3 Trivia Data Normalization

**Overview:**  
Each difficulty branch reshapes the API response into a common format with fields useful for Slack posting and logging.

**Nodes Involved:**  
- Set Message Easy  
- Set Message Medium  
- Set Message Hard

**Node Details:**

- **Set Message Easy**  
  - Type: Set node  
  - Role: Extracts and normalizes fields from easy difficulty API response  
  - Configuration: Sets fields:  
    - `timestamp`: current ISO timestamp  
    - `date`: current date (YYYY-MM-DD)  
    - `difficulty`: "easy"  
    - `category`, `question`, `correct`, `incorrect`: extracted from API response at `$json["results"][0]`  
    - `messageTitle`: "🟢 今日のクイズ（EASY）" (Japanese for "Today's Quiz (EASY)")  
    - `messageBody`: formatted string combining category, question, correct answer, and incorrect answers  
  - Inputs: From HTTP OpenTDB Easy  
  - Outputs: To Merge node  
  - Edge cases: If API response lacks expected fields, expressions may fail causing empty or error items

- **Set Message Medium**  
  - Type: Set node  
  - Role: Same as Set Message Easy but for medium difficulty  
  - Configuration: Similar fields as above but `difficulty` = "medium"  
  - Inputs: From HTTP OpenTDB Medium  
  - Outputs: To Merge node  
  - Edge cases: Same as Set Message Easy

- **Set Message Hard**  
  - Type: Set node  
  - Role: Same as Set Message Easy but for hard difficulty  
  - Configuration: Similar fields as above but `difficulty` = "hard"  
  - Inputs: From HTTP OpenTDB Hard  
  - Outputs: To Merge node  
  - Edge cases: Same as Set Message Easy

---

#### 2.4 Merging Trivia Streams

**Overview:**  
This node consolidates the three difficulty branches back into a single data stream for further processing.

**Nodes Involved:**  
- Merge

**Node Details:**

- **Merge**  
  - Type: Merge node  
  - Role: Combines the three separate inputs into one output stream  
  - Configuration: Number of inputs set to 3 (easy, medium, hard branches)  
  - Inputs: From Set Message Easy (input 1), Set Message Medium (input 2), Set Message Hard (input 3)  
  - Outputs: To Slack: Post Trivia node  
  - Edge cases: If any of the inputs do not output an item, merge might produce fewer or no items; node expects exactly one item per input

---

#### 2.5 Slack Posting and Google Sheets Logging

**Overview:**  
This block posts the trivia question to Slack and appends the trivia data as a new row in a Google Sheets spreadsheet.

**Nodes Involved:**  
- Slack: Post Trivia  
- Sheets: Append Trivia

**Node Details:**

- **Slack: Post Trivia**  
  - Type: Slack node  
  - Role: Posts a formatted trivia question message to a specified Slack channel  
  - Configuration:  
    - Text composed of `messageTitle` and `messageBody` fields concatenated with line breaks  
    - Channel set to `#general` (modifiable)  
    - Authentication via OAuth2 (requires Slack app credentials and OAuth2 setup)  
  - Inputs: From Merge node  
  - Outputs: To Sheets: Append Trivia node  
  - Failures: Authentication failure, channel access denied, Slack API rate limits

- **Sheets: Append Trivia**  
  - Type: Google Sheets node  
  - Role: Appends a new row with trivia data to a specified Google Sheets spreadsheet and sheet  
  - Configuration:  
    - Document ID linked to a specific Google Sheets document  
    - Sheet ID corresponds to a particular sheet tab  
    - Columns mapped explicitly: date, correct, category, question, incorrect, timestamp, difficulty  
  - Inputs: From Slack: Post Trivia node  
  - Outputs: None (end of flow)  
  - Failures: Authentication failure, insufficient permissions, sheet not found, network errors

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                          | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                                          |
|-----------------------|---------------------|----------------------------------------|-------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger    | Starts workflow daily at 08:00 AM      | None                    | Set: Choose Trivia Type | ## Trigger and difficulty routing - Schedule Trigger starts the workflow once per day at your chosen time.                           |
| Set: Choose Trivia Type| Set                 | Randomly selects trivia difficulty     | Schedule Trigger        | Switch: triviaType       | ## Trigger and difficulty routing - Set node randomly assigns a difficulty value (easy, medium, hard).                              |
| Switch: triviaType     | Switch              | Routes flow to difficulty-specific API | Set: Choose Trivia Type | HTTP OpenTDB Easy / Medium / Hard | ## Trigger and difficulty routing - Switch node routes execution based on difficulty value.                                          |
| HTTP OpenTDB Easy      | HTTP Request        | Fetches easy difficulty trivia question| Switch: triviaType (easy output) | Set Message Easy         | ## Easy trivia branch - Calls OpenTDB API with difficulty=easy and returns one multiple-choice question.                             |
| Set Message Easy       | Set                 | Normalizes easy trivia API response    | HTTP OpenTDB Easy       | Merge                   | ## Easy trivia branch - Normalizes response to common trivia format for merging.                                                     |
| HTTP OpenTDB Medium    | HTTP Request        | Fetches medium difficulty trivia question| Switch: triviaType (medium output) | Set Message Medium       | ## Medium & hard trivia branches - Calls OpenTDB with difficulty=medium.                                                             |
| Set Message Medium     | Set                 | Normalizes medium trivia API response  | HTTP OpenTDB Medium     | Merge                   | ## Medium & hard trivia branches - Normalizes response for merging.                                                                   |
| HTTP OpenTDB Hard      | HTTP Request        | Fetches hard difficulty trivia question| Switch: triviaType (hard output) | Set Message Hard         | ## Medium & hard trivia branches - Calls OpenTDB with difficulty=hard.                                                                |
| Set Message Hard       | Set                 | Normalizes hard trivia API response    | HTTP OpenTDB Hard       | Merge                   | ## Medium & hard trivia branches - Normalizes response for merging.                                                                   |
| Merge                 | Merge               | Combines all difficulty branches       | Set Message Easy / Medium / Hard | Slack: Post Trivia       | ## Merge, Slack posting, and Sheets logging - Combines items into a single stream for posting and logging.                            |
| Slack: Post Trivia     | Slack               | Posts trivia question to Slack channel | Merge                   | Sheets: Append Trivia    | ## Merge, Slack posting, and Sheets logging - Posts formatted trivia to Slack #general channel.                                        |
| Sheets: Append Trivia  | Google Sheets       | Logs trivia question data to spreadsheet| Slack: Post Trivia      | None                    | ## Merge, Slack posting, and Sheets logging - Appends trivia data as new row to Google Sheets for archival.                           |
| Sticky: Template Overview | Sticky Note       | Documentation overview                  | None                    | None                    | ## Daily trivia to Slack + Google Sheets - Explains workflow purpose, audience, and process.                                           |
| Sticky: Trigger & Routing | Sticky Note       | Documentation for trigger and routing  | None                    | None                    | ## Trigger and difficulty routing - Describes schedule trigger, difficulty selection, and routing nodes.                              |
| Sticky: Easy Branch    | Sticky Note         | Documentation for easy branch           | None                    | None                    | ## Easy trivia branch - Details about easy difficulty API request and normalization.                                                  |
| Sticky: Medium & Hard Branches | Sticky Note  | Documentation for medium and hard branches | None                 | None                    | ## Medium & hard trivia branches - Details about medium and hard difficulty API requests and normalization.                          |
| Sticky: Merge, Slack & Sheets | Sticky Note    | Documentation for merge and output nodes | None                  | None                    | ## Merge, Slack posting, and Sheets logging - Explains merging, Slack message posting, and Google Sheets logging.                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Set to trigger once daily at 08:00 AM (Hour: 8).  
   - No credentials needed.

3. **Add a Set node named "Choose Trivia Type":**  
   - Add a string field named `difficulty`.  
   - Set its value with the expression:  
     `={{ ['easy','medium','hard'][Math.floor(Math.random() * 3)] }}`  
   - Connect Schedule Trigger output to this node’s input.

4. **Add a Switch node named "triviaType":**  
   - Set to check the string value of `difficulty`.  
   - Add three rules matching values: `easy`, `medium`, and `hard`.  
   - Connect output of "Choose Trivia Type" to this node.

5. **Add three HTTP Request nodes:**  

   - **HTTP OpenTDB Easy:**  
     - URL: `https://opentdb.com/api.php?amount=1&difficulty=easy&type=multiple`  
     - Method: GET (default)  
     - Connect Switch node’s `easy` output here.

   - **HTTP OpenTDB Medium:**  
     - URL: `https://opentdb.com/api.php?amount=1&difficulty=medium&type=multiple`  
     - Connect Switch node’s `medium` output here.

   - **HTTP OpenTDB Hard:**  
     - URL: `https://opentdb.com/api.php?amount=1&difficulty=hard&type=multiple`  
     - Connect Switch node’s `hard` output here.

6. **Add three Set nodes to normalize API responses:**  

   - Each Set node corresponds to one difficulty branch and is connected to the respective HTTP node.  
   - For each Set node, configure fields:

     - `timestamp`: `={{(new Date()).toISOString()}}`  
     - `date`: `={{(new Date()).toISOString().slice(0,10)}}`  
     - `difficulty`: `"easy"` / `"medium"` / `"hard"` as applicable  
     - `category`: `={{$json["results"][0]["category"]}}`  
     - `question`: `={{$json["results"][0]["question"]}}`  
     - `correct`: `={{$json["results"][0]["correct_answer"]}}`  
     - `incorrect`: `={{$json["results"][0]["incorrect_answers"].join(", ")}}`  
     - `messageTitle`: Set to emoji plus Japanese "今日のクイズ" plus difficulty, e.g.,  
       - Easy: `"🟢 今日のクイズ（EASY）"`  
       - Medium: `"🟡 今日のクイズ（MEDIUM）"`  
       - Hard: `"🔴 今日のクイズ（HARD）"`  
     - `messageBody`:  
       ```
       ={{"カテゴリ: " + $json["results"][0]["category"] + "\nQ: " + $json["results"][0]["question"] + "\n\n正解: " + $json["results"][0]["correct_answer"] + "\n誤答候補: " + $json["results"][0]["incorrect_answers"].join(", ")}}
       ```

7. **Add a Merge node:**  
   - Set "Mode" to "Number of Inputs" = 3.  
   - Connect the outputs of the three Set nodes (Easy, Medium, Hard) to the three inputs of Merge, respectively.

8. **Add a Slack node named "Post Trivia":**  
   - Authentication: OAuth2 with Slack credentials configured.  
   - Channel: Set to `#general` or your preferred Slack channel.  
   - Text:  
     ```
     ={{ $json["messageTitle"] + '\n\n' + $json["messageBody"] }}
     ```  
   - Connect Merge node output to Slack node input.

9. **Add a Google Sheets node named "Append Trivia":**  
   - Operation: Append  
   - Document ID: your target Google Sheets document ID.  
   - Sheet ID: the specific sheet/tab ID or name where trivia rows will be logged.  
   - Columns to map:  
     - `date` -> `date`  
     - `correct` -> `correct`  
     - `category` -> `category`  
     - `question` -> `question`  
     - `incorrect` -> `incorrect`  
     - `timestamp` -> `timestamp`  
     - `difficulty` -> `difficulty`  
   - Connect Slack node output to this node input.  
   - Credentials: Google Sheets OAuth2 with write access configured.

10. **Activate the workflow and test by manual run or wait for scheduled time.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow created to provide daily Slack icebreaker trivia questions using Open Trivia Database API and store questions in Google Sheets.| Purpose and usage context                                                                                     |
| Slack and Google Sheets credentials must be set up with appropriate OAuth2 authorization prior to activation.                            | Credential setup instructions                                                                                 |
| The trivia questions and answers are formatted with Japanese labels ("今日のクイズ", "カテゴリ", "正解", "誤答候補").                          | Localization detail for message formatting                                                                    |
| Open Trivia Database API documentation: https://opentdb.com/api_config.php                                                                | Reference for API parameters                                                                                   |
| Slack OAuth2 app setup guide: https://api.slack.com/authentication/oauth-v2                                                                | Setup instructions for Slack authentication                                                                   |
| Google Sheets API and OAuth2 setup: https://developers.google.com/sheets/api/quickstart/js                                                  | Setup instructions for Google Sheets API access                                                                |
| Adjust schedule time and Slack channel in respective nodes to customize workflow for your team’s timezone and preferred channel.          | Customization advice                                                                                           |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.