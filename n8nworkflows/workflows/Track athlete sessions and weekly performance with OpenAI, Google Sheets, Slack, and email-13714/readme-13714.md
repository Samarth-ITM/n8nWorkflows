Track athlete sessions and weekly performance with OpenAI, Google Sheets, Slack, and email

https://n8nworkflows.xyz/workflows/track-athlete-sessions-and-weekly-performance-with-openai--google-sheets--slack--and-email-13714


# Track athlete sessions and weekly performance with OpenAI, Google Sheets, Slack, and email

This technical document provides a comprehensive breakdown of the **Smart Athlete Performance Tracker** workflow. This system automates the monitoring of athlete training sessions, providing real-time AI-driven analysis and scheduled weekly summaries.

---

### 1. Workflow Overview

The workflow is designed for coaches and athletic trainers to track progress, detect performance plateaus, and receive automated alerts. It operates via two distinct logical pipelines:

1.  **Real-Time Session Analysis:** Triggered by a form submission, it records the session, compares it with historical data using AI, updates a central database, and alerts staff if performance falls below a specific threshold.
2.  **Weekly Performance Reporting:** Triggered every Monday morning, it aggregates all training data from the previous week, groups it by athlete, and generates a comprehensive progress report via AI.

#### Logical Blocks:
*   **1.1 Data Intake & Storage:** Captures form inputs and commits them to Google Sheets.
*   **1.2 Historical Contextualization:** Retrieves previous sessions to provide the AI with longitudinal data.
*   **1.3 AI Performance Engine:** Uses LLMs to interpret trends (improvement, plateau, regression).
*   **1.4 Automated Alerting:** Evaluates AI scores against business logic to send Slack/Email notifications.
*   **1.5 Weekly Batch Processing:** Schedules and groups data for high-level management reporting.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Intake & Configuration
This block handles the entry point for new training data and initializes environment variables.
*   **Nodes Involved:** `Training Session Form`, `Workflow Configuration`, `Store Training Record`.
*   **Node Details:**
    *   **Training Session Form:** A native n8n form collecting fields like Skater Name, Training Date, Duration, Session Type, Laps, Speed, Stamina, and Notes.
    *   **Workflow Configuration:** A "Set" node defining static IDs for Google Sheets, Slack channels, and the `performanceThreshold` (default: 5).
    *   **Store Training Record:** Appends the form data as a new row in the "Training Records" Google Sheet.

#### 2.2 Historical Context & Session Analysis
This block prepares the data for the AI Agent to ensure the analysis is not based on a single isolated event.
*   **Nodes Involved:** `Fetch Historical Data`, `Combine Current and Historical Data`, `Performance Analysis Agent`, `OpenAI Model - Analysis`, `Analysis Output Parser`.
*   **Node Details:**
    *   **Fetch Historical Data:** Queries Google Sheets for all previous records matching the current Skater's Name.
    *   **Combine Current and Historical Data:** Merges the new session data with historical records into a single JSON object.
    *   **Performance Analysis Agent:** A LangChain Agent that receives the combined data. It is instructed to identify trends across metrics like jump consistency and speed.
    *   **Analysis Output Parser:** Forces the AI to return a structured JSON object containing an `overallTrend`, `performanceScore` (0-10), and a boolean `alertRequired`.

#### 2.3 Result Persistence & Alerting
This block saves the AI insights back to the database and handles conditional notifications.
*   **Nodes Involved:** `Update Record with Insights`, `Check Performance Threshold`, `Send Slack Alert`, `Send Email Alert`.
*   **Node Details:**
    *   **Update Record with Insights:** Updates the original row in Google Sheets with the AI's trend analysis and score.
    *   **Check Performance Threshold:** An "If" node. It triggers alerts if the `performanceScore` is less than the threshold (5) OR if the AI explicitly flagged `alertRequired` as true.
    *   **Send Slack/Email Alert:** Dispatches formatted messages containing recommendations and concern areas to the coaching staff.

#### 2.4 Weekly Batch Processing
A separate pipeline that provides a "big picture" view of athlete development.
*   **Nodes Involved:** `Weekly Summary Schedule`, `Weekly Config`, `Fetch Weekly Data`, `Group by Athlete`, `Weekly Summary Agent`, `Summary Output Parser`.
*   **Node Details:**
    *   **Weekly Summary Schedule:** Triggers every Monday at 9:00 AM.
    *   **Group by Athlete:** A "Code" node that takes all sessions from the last 7 days and organizes them into arrays keyed by the athlete's name.
    *   **Weekly Summary Agent:** Analyzes the aggregate weekly statistics (total minutes, average speed) to generate a motivational and technical summary.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Training Session Form | Form Trigger | Entry Point | None | Workflow Configuration | How It Works: Automates athlete monitoring... |
| Workflow Configuration | Set | Config Management | Training Session Form | Store Training Record | Fetch & Combine Historical Data: Retrieves prior records... |
| Store Training Record | Google Sheets | Data Storage | Workflow Config | Fetch Hist. Data, Combine Data | Fetch & Combine Historical Data: Retrieves prior records... |
| Fetch Historical Data | Google Sheets | Data Retrieval | Store Training Record | Combine Data | Fetch & Combine Historical Data: Retrieves prior records... |
| Combine Current... | Merge | Data Prep | Store Record, Fetch Hist. | Perf. Analysis Agent | Fetch & Combine Historical Data: Retrieves prior records... |
| Performance Analysis Agent | AI Agent | Insight Generation | Combine Data | Update Record | Performance Analysis Agent: Analyses data using OpenAI... |
| Update Record... | Google Sheets | Data Persistence | Perf. Analysis Agent | Check Threshold | Performance Analysis Agent: Analyses data using OpenAI... |
| Check Performance Threshold | If | Logical Filter | Update Record | Slack Alert, Email Alert | Check Performance Threshold: Evaluates metric breaches. |
| Send Slack Alert | Slack | Notification | Check Threshold | None | Check Performance Threshold: Alerts coaches. |
| Weekly Summary Schedule | Schedule | Batch Trigger | None | Weekly Config | Weekly Summary Schedule & Grouping: Organises data. |
| Group by Athlete | Code | Data Transformation | Fetch Weekly Data | Weekly Summary Agent | Weekly Summary Schedule & Grouping: Organises data. |
| Weekly Summary Agent | AI Agent | Weekly Reporting | Group by Athlete | Slack Summary, Email Summary | Weekly Summary Agent: Generates summaries via OpenAI. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Database Setup:** Create a Google Sheet with a tab named "Training Records". Headers should include: `timestamp`, `Skater Name`, `Training Date`, `Training Duration`, `Session Type`, `Coach Name`, `Laps Completed`, `Average Speed`, `Stamina Score`, `Jump Consistency`, `Notes`, and `AI Analysis`.
2.  **Initial Trigger:** Add a **Form Trigger** node. Define fields matching the headers above.
3.  **Variable Initialization:** Add a **Set** node. Create string variables for `googleSheetId`, `slackChannel`, and `emailApiUrl`. Add a number variable for `performanceThreshold`.
4.  **Storage Logic:** 
    *   Add a **Google Sheets (Append)** node to save the form data.
    *   Add a **Google Sheets (Get Many)** node to filter records where `Skater Name` equals the current input.
5.  **AI Integration:**
    *   Add a **Merge** node (Mode: Combine) to join the new entry with the list of historical entries.
    *   Add an **AI Agent** node. Connect an **OpenAI Chat Model** (GPT-4o or GPT-4o-mini).
    *   Add a **Structured Output Parser**. Define a JSON schema including `performanceScore` (number) and `alertRequired` (boolean).
6.  **Alerting Logic:**
    *   Add a **Google Sheets (Update)** node to write the AI output back to the specific row.
    *   Add an **If** node to check if the score is below the threshold.
    *   Connect **Slack** and **HTTP Request** nodes to the "True" branch for coaching alerts.
7.  **Weekly Automation:**
    *   Add a **Schedule Trigger** (Every Monday).
    *   Add a **Google Sheets (Get Many)** node to fetch rows where `Training Date` is within the last 7 days.
    *   Add a **Code** node to iterate through the list and group sessions by `athleteName`.
    *   Add a second **AI Agent** specialized in "Weekly Summaries" and connect to **Slack/Email** outputs.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Prerequisites** | Google Sheets Service Account, Slack Bot Token, OpenAI API Key. |
| **Use Case** | Real-time performance threshold alerts for elite athlete training programs. |
| **Customization** | The OpenAI model can be replaced with Anthropic Claude for different analysis nuances. |
| **Data Flow** | Eliminates manual data aggregation for athletic performance analysts. |