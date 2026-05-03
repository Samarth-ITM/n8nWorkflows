Predict customer churn daily using ML or LLM models and notify via Slack/email

https://n8nworkflows.xyz/workflows/predict-customer-churn-daily-using-ml-or-llm-models-and-notify-via-slack-email-13726


# Predict customer churn daily using ML or LLM models and notify via Slack/email

This document provides a technical breakdown and reconstruction guide for the **AI Customer Churn Predictor** n8n workflow.

---

### 1. Workflow Overview
The **AI Customer Churn Predictor** is an automated pipeline designed to identify customers at risk of leaving a service. It operates on a daily schedule, aggregating raw data from multiple sources (profiles, logs, transactions), transforming that data into a feature vector, and utilizing a Machine Learning (ML) model to predict churn probability. Based on the resulting risk level (High, Medium, or Low), the workflow triggers specific retention campaigns, logs predictions to a database, and alerts stakeholders via Slack and Email with detailed HTML reports.

#### Functional Blocks:
*   **1.1 Data Collection:** Triggers daily and pulls data from three distinct API endpoints.
*   **1.2 Feature Engineering:** Merges data and calculates behavioral metrics (e.g., login frequency, spend trends).
*   **1.3 ML Prediction & Scoring:** Sends features to an external ML API and classifies the output into risk tiers.
*   **1.4 Action & Storage:** Executes retention tasks and persists predictions in a PostgreSQL database.
*   **1.5 Reporting & Notification:** Generates HTML reports and sends multi-channel alerts for high-risk accounts.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Collect Data
**Overview:** This block initiates the process every day at 2 AM and gathers the necessary raw data to evaluate customer behavior.
*   **Nodes Involved:** `Daily churn analysis at 2 AM`, `Fetch active customer profiles`, `Fetch customer activity logs (30 days)`, `Fetch transaction history (90 days)`.
*   **Node Details:**
    *   **Schedule Trigger:** Configured with a Cron expression `0 2 * * *`.
    *   **HTTP Request Nodes:** Three separate GET requests to `api.example.com`.
    *   **Edge Cases:** API timeouts or authentication failures will halt the workflow. Ensure the endpoints can handle the load of fetching all active customers simultaneously.

#### 2.2 Feature Engineering
**Overview:** Combines the disparate data sources into a single customer object and calculates mathematical indicators used for prediction.
*   **Nodes Involved:** `Merge customer and activity data`, `Engineer behavioral features`.
*   **Node Details:**
    *   **Merge Node:** Uses "Combine" mode to link profiles with logs and transactions.
    *   **Code Node (JavaScript):** Calculates `login_frequency`, `spend_trend`, and `adoption_score`. It also sets "Anomaly Flags" such as `flag_suspicious_login` (drop in logins) or `flag_no_purchase_60days`.
    *   **Key Logic:** If `login_last_7_days` is significantly lower than the previous period, it flags a login trend drop.

#### 2.3 ML Prediction & Scoring
**Overview:** Interfaces with a Machine Learning model and interprets the probabilistic output into actionable business logic.
*   **Nodes Involved:** `Call ML churn prediction model`, `Score and classify churn risk`.
*   **Node Details:**
    *   **HTTP Request (POST):** Sends a JSON payload containing the engineered features to `ml-api.example.com/predict/churn`.
    *   **Code Node:** Converts the 0.0–1.0 ML output into a 0–100 score. It assigns tiers: **HIGH_RISK** (>=70), **MEDIUM_RISK** (>=50), and **EARLY_RISK** (>=30).
    *   **Edge Cases:** If the ML API returns an error or a null prediction, the scoring node may fail unless default values are provided.

#### 2.4 Classify & Remediate
**Overview:** Directs the flow based on the severity of the risk and initiates CRM/Marketing actions.
*   **Nodes Involved:** `Route by risk level`, `Create retention campaign task`, `Store churn predictions in database`.
*   **Node Details:**
    *   **Switch Node:** Routes based on the `risk_level` string.
    *   **HTTP Request (POST):** For High and Medium risk, it calls a retention API to create "aggressive" or "engagement" campaigns.
    *   **Postgres Node:** Inserts the `customer_id`, `churn_probability`, and `risk_level` into the `churn_predictions` table for historical tracking.

#### 2.5 Report & Notify
**Overview:** Formats the findings into human-readable alerts and logs the final completion status.
*   **Nodes Involved:** `Generate churn analytics report`, `Filter at-risk customers`, `Post churn alert to Slack`, `Email report to customer success team`, `Log analysis completion`.
*   **Node Details:**
    *   **Code Node (HTML):** Builds a complete responsive HTML template with CSS, badges, and metric cards for the email body.
    *   **Filter Node:** Only allows customers with `action_required === true` (High/Medium risk) to proceed to notifications.
    *   **Slack/Email:** Sends the formatted markdown to Slack and the HTML report via SMTP.
    *   **Completion Log:** Updates `$getWorkflowStaticData` to keep track of cumulative stats (Total Analyzed, High Risk Count) within the workflow's memory.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Daily churn analysis at 2 AM | Schedule | Trigger | None | Fetch active profiles/logs/history | 1. Trigger & Collect Data |
| Fetch active customer profiles | HTTP Request | Data Retrieval | Schedule Trigger | Merge customer data | 1. Trigger & Collect Data |
| Fetch activity logs | HTTP Request | Data Retrieval | Schedule Trigger | Merge customer data | 1. Trigger & Collect Data |
| Fetch transaction history | HTTP Request | Data Retrieval | Schedule Trigger | Merge customer data | 1. Trigger & Collect Data |
| Merge customer and activity data | Merge | Data Synthesis | Profiles, Logs, History | Engineer behavioral features | 2. Feature Engineering |
| Engineer behavioral features | Code | Data Transformation | Merge Node | Call ML model | 2. Feature Engineering |
| Call ML churn prediction model | HTTP Request | ML Inference | Engineer features | Score and classify | 3. ML Prediction |
| Score and classify churn risk | Code | Logic/Business Rules | Call ML model | Route by risk level | 3. ML Prediction |
| Route by risk level | Switch | Conditional Logic | Score and classify | Campaign task, Database | 4. Classify & Remediate |
| Create retention campaign task | HTTP Request | Integration | Route (High/Med) | Generate report | 4. Classify & Remediate |
| Store churn predictions in DB | Postgres | Data Persistence | Route (All) | Generate report | 4. Classify & Remediate |
| Generate churn analytics report | Code | Formatting/UI | Campaign task, DB | Filter at-risk | 5. Report & Notify |
| Filter at-risk customers | Filter | Optimization | Generate report | Slack, Email | 5. Report & Notify |
| Post churn alert to Slack | HTTP Request | Notification | Filter | Log completion | 5. Report & Notify |
| Email report to team | Email Send | Notification | Filter | Log completion | 5. Report & Notify |
| Log analysis completion | Code | Observability | Slack, Email | None | 5. Report & Notify |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Schedule Trigger** node set to a daily interval (e.g., 02:00).
2.  **Data Ingestion:**
    *   Add three **HTTP Request** nodes. Configure them to fetch your Customer, Activity, and Transaction data.
    *   Add a **Merge** node set to "Combine" mode to join these three inputs into a single object per customer.
3.  **Feature Construction:**
    *   Add a **Code** node. Use JavaScript to calculate keys like `login_frequency` (logins/days), `spend_trend` (current/previous spend), and `days_since_last_purchase`.
4.  **AI/ML Integration:**
    *   Add an **HTTP Request** node (POST) to your ML model (or an OpenAI node with a prompt). Send the engineered features as the payload.
    *   Add another **Code** node to categorize the risk (High/Medium/Low) based on the score returned by the AI.
5.  **Logic Branching:**
    *   Add a **Switch** node using `{{ $json.risk_level }}` to branch the workflow.
6.  **Persistence & Actions:**
    *   Connect the Switch outputs to a **Postgres** node (Insert) to save every prediction.
    *   Connect High/Medium outputs to an **HTTP Request** node representing your CRM (HubSpot/Salesforce) to create tasks.
7.  **Communication:**
    *   Add a **Code** node to generate a `reportHtml` string and a `notificationMessage` string using template literals.
    *   Add a **Filter** node to check if `action_required` is true.
    *   Add a **Slack** node (via Webhook) and an **Email Send** node (SMTP) to deliver the results.
8.  **Credentials:** Ensure you have configured OAuth2 or API Keys for your Customer API, ML API, PostgreSQL, and SMTP server.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Model Optimization** | Accuracy depends on feature quality; ensure login trends and support interactions are updated daily. |
| **Alert Fatigue** | High-risk thresholds (>70%) should be tuned to avoid overwhelming the success team. |
| **CRM Integration** | This workflow assumes a REST API for campaigns; can be swapped for HubSpot or Salesforce nodes. |
| **Feedback Loop** | Consider adding a final step that stores "Actual Churn" later to retrain the ML model. |