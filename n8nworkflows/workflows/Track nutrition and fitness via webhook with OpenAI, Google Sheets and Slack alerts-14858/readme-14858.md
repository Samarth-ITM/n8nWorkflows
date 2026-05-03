Track nutrition and fitness via webhook with OpenAI, Google Sheets and Slack alerts

https://n8nworkflows.xyz/workflows/track-nutrition-and-fitness-via-webhook-with-openai--google-sheets-and-slack-alerts-14858


# Track nutrition and fitness via webhook with OpenAI, Google Sheets and Slack alerts

# Technical Documentation: AI Fitness Coach Workflow

## 1. Workflow Overview
The **AI Fitness Coach** is an automated nutrition and fitness tracking system designed to process real-time user inputs (via webhooks), analyze them using Artificial Intelligence, and maintain a persistent health record in Google Sheets. The workflow monitors daily caloric targets and health risks, triggering alerts via Slack when specific thresholds are breached.

The logic is organized into the following functional blocks:
- **1.1 Daily Reset Logic:** A scheduled process to clear daily metrics.
- **1.2 Input Reception & Validation:** Webhook entry and user verification via Google Sheets.
- **1.3 AI Nutritional Analysis:** Natural language processing to extract nutritional data.
- **1.4 Metrics Processing:** Computational logic to update total daily intake.
- **1.5 Monitoring & Alerting:** Multi-stage checks for calorie limits and health risks.
- **1.6 Response & Persistence:** Returning the result to the user and updating the database.

---

## 2. Block-by-Block Analysis

### 1.1 Daily Reset Logic
**Overview:** This block ensures that users start every day with zeroed-out metrics (calories, protein, and risk counts) to maintain accurate daily tracking.
- **Nodes Involved:** `Daily Reset Trigger`, `Fetch All Users`, `Reset Daily Stats`.
- **Node Details:**
    - **Daily Reset Trigger** (Schedule Trigger): Triggers the workflow on a predefined interval.
    - **Fetch All Users** (Google Sheets): Retrieves all rows from the "Fittness tracker" sheet to identify all registered users.
    - **Reset Daily Stats** (Google Sheets): Performs an `update` operation matching by `Phone`. It sets `Consumed_Calories`, `Protein`, and `Risk_Count` to `0` and `Status` to `NEW_DAY`.

### 1.2 Input Reception & Validation
**Overview:** Acts as the gateway for the workflow, ensuring that only recognized users can trigger AI processing.
- **Nodes Involved:** `Receive User Input`, `Fetch User Data`, `Validate User Exists`.
- **Node Details:**
    - **Receive User Input** (Webhook): Listens for POST requests on the path `diet-input`. It expects a JSON body containing `phone` and `message`.
    - **Fetch User Data** (Google Sheets): Looks up the user's profile in the "Fittness tracker" sheet using the `Phone` column.
    - **Validate User Exists** (If): Checks if the Google Sheets lookup returned a result. If no user is found, the workflow stops.

### 1.3 AI Nutritional Analysis
**Overview:** Transforms unstructured text (e.g., "I ate two eggs and a toast") into structured JSON data.
- **Nodes Involved:** `AI Nutrition Analyzer`, `AI Model (GPT)`.
- **Node Details:**
    - **AI Nutrition Analyzer** (AI Agent): Uses a detailed system prompt to categorize input as `FOOD`, `ACTIVITY`, or `UNKNOWN`. It extracts calories, protein, meal type, and identifies risks (e.g., `OVER_EATING`).
    - **AI Model (GPT)** (OpenAI Chat Model): The LLM engine (configured as `gpt-4.1-mini`) that powers the analyzer.
    - **Failure Type:** AI might return invalid JSON or "UNKNOWN" types if the input is ambiguous.

### 1.4 Metrics Processing
**Overview:** Calculates the new totals by combining the AI's findings with the user's current daily stats.
- **Nodes Involved:** `Process Nutrition Data`.
- **Node Details:**
    - **Process Nutrition Data** (Code): A JavaScript node that:
        1. Parses the AI JSON output.
        2. Adds calories/protein if type is `FOOD`.
        3. Subtracts calories if type is `ACTIVITY`.
        4. Increments `Risk_Count` if the AI detected a health risk.
        5. Merges these results with the original user profile.

### 1.5 Monitoring & Alerting
**Overview:** A three-tier verification system to notify administrators or users of health deviations via Slack.
- **Nodes Involved:** `Check Calorie Threshold`, `Alert: Calorie Limit Exceeded`, `Check Health Risk`, `Alert: Health Risk`, `Check Critical Risk`, `Alert: Critical Health Issue`.
- **Node Details:**
    - **Check Calorie Threshold** (If): Compares `Consumed_Calories` against `Daily_Calorie_Target`.
    - **Alert: Calorie Limit Exceeded** (Slack): Sends a warning to a specific channel if calories are exceeded.
    - **Check Health Risk** (If): Checks if the AI `risk` variable is not equal to `NONE`.
    - **Alert: Health Risk** (Slack): Sends a warning about the specific risk detected.
    - **Check Critical Risk** (If): Checks if `Risk_Count` is $\ge 3$.
    - **Alert: Critical Health Issue** (Slack): Sends a high-priority alert indicating a repeated pattern of unhealthy behavior.

### 1.6 Response & Persistence
**Overview:** Closes the loop by notifying the user and saving the updated state to the database.
- **Nodes Involved:** `Send API Response`, `Update User Data`.
- **Node Details:**
    - **Send API Response** (Respond to Webhook): Returns a JSON response to the calling app containing the AI's friendly reply and the updated calorie totals.
    - **Update User Data** (Google Sheets): Updates the user's row in the spreadsheet, persisting the new `Consumed_Calories`, `Protein`, `Risk_Count`, and `Status`.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Daily Reset Trigger | Schedule Trigger | Trigger daily reset | None | Fetch All Users | Reset Daily Nutrition Data |
| Fetch All Users | Google Sheets | Get all user list | Daily Reset Trigger | Reset Daily Stats | Reset Daily Nutrition Data |
| Reset Daily Stats | Google Sheets | Zero out metrics | Fetch All Users | None | Reset Daily Nutrition Data |
| Receive User Input | Webhook | Entry point | None | Fetch User Data | Receive and Validate User Input |
| Fetch User Data | Google Sheets | Retrieve user profile | Receive User Input | Validate User Exists | Receive and Validate User Input |
| Validate User Exists | If | User verification | Fetch User Data | AI Nutrition Analyzer | Receive and Validate User Input |
| AI Nutrition Analyzer | AI Agent | Extract nutrients | Validate User Exists | Process Nutrition Data | Analyze Food or Activity with AI |
| AI Model (GPT) | OpenAI Model | LLM Engine | None | AI Nutrition Analyzer | Analyze Food or Activity with AI |
| Process Nutrition Data | Code | Math & Logic | AI Nutrition Analyzer | Check Calorie Threshold | Calculate Calories and Update Metrics |
| Check Calorie Threshold| If | Limit check | Process Nutrition Data | Alert: Calorie Limit Exceeded | Monitor Daily Calorie Limit |
| Alert: Calorie Limit Exceeded | Slack | Notify excess cals | Check Calorie Threshold | Check Health Risk | Monitor Daily Calorie Limit |
| Check Health Risk | If | Risk detection | Alert: Calorie Limit Exceeded | Alert: Health Risk | Detect Health Risks and Escalate Alerts |
| Alert: Health Risk | Slack | Notify risk | Check Health Risk | Check Critical Risk | Detect Health Risks and Escalate Alerts |
| Check Critical Risk | If | Frequency check | Alert: Health Risk | Alert: Critical Health Issue | Detect Health Risks and Escalate Alerts |
| Alert: Critical Health Issue | Slack | High priority alert | Check Critical Risk | Send API Response | Detect Health Risks and Escalate Alerts |
| Send API Response | Webhook Resp. | User feedback | Alert: Critical Health Issue | Update User Data | Response & Data Storage |
| Update User Data | Google Sheets | Save state | Send API Response | None | Response & Data Storage |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Database Setup (Google Sheets)
Create a Google Sheet named "Fittness tracker" with the following columns:
`Name`, `Phone`, `Goal`, `Daily_Calorie_Target`, `Consumed_Calories`, `Protein`, `Last_Meal`, `Status`, `Risk_Count`.

### Step 2: Daily Reset Path
1. Add a **Schedule Trigger** (Interval: Daily).
2. Connect to a **Google Sheets** node (Operation: `Get Many` / `Read` all rows).
3. Connect to a **Google Sheets** node (Operation: `Update`). Match by `Phone`. Set `Consumed_Calories`, `Protein`, and `Risk_Count` to `0`.

### Step 3: Input & Validation Path
1. Add a **Webhook** node. Path: `diet-input`. Response Mode: `Response Node`.
2. Connect to a **Google Sheets** node (Operation: `Get`). Filter by `Phone` = `{{ $json.body.phone }}`.
3. Connect to an **If** node. Condition: Check if the result from the previous node exists.

### Step 4: AI Integration
1. Add an **AI Agent** node.
2. Attach an **OpenAI Chat Model** node (Model: `gpt-4.1-mini`).
3. Set the Agent's prompt to include the User Goal and Calorie Target from the Sheet, and the message from the Webhook. Instruct it to return a strict JSON format with `type`, `calories`, `protein`, `risk`, `suggestion`, and `reply`.

### Step 5: Data Processing
1. Add a **Code** node. Implement the JavaScript logic to:
   - Parse AI JSON.
   - Update `Consumed_Calories` (Add if FOOD, Subtract if ACTIVITY).
   - Increment `Risk_Count` if `risk != 'NONE'`.

### Step 6: Alerting Logic
1. **If Node 1**: Compare `Consumed_Calories` > `Daily_Calorie_Target`. $\rightarrow$ **Slack Node** (Alert channel).
2. **If Node 2**: Compare `risk` != `NONE`. $\rightarrow$ **Slack Node** (Warning channel).
3. **If Node 3**: Compare `Risk_Count` $\ge 3$. $\rightarrow$ **Slack Node** (Critical channel).

### Step 7: Closing the Loop
1. Add a **Respond to Webhook** node. Use expressions to return the AI's `reply`, `calories`, and `total_today`.
2. Add a final **Google Sheets** node (Operation: `Update`). Match by `Phone`. Update all nutritional and risk columns with the values calculated in the Code node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Required Credentials | Google Sheets OAuth2, OpenAI API Key, Slack OAuth2 |
| Critical Columns | `Phone` is used as the Unique Identifier (Primary Key) |
| AI Model | Uses `gpt-4.1-mini` for cost-efficient, fast extraction |
| System Logic | The workflow utilizes a "waterfall" alerting system where the response is only sent after all potential alert checks are processed |