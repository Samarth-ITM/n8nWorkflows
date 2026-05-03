Track and analyze Google Sheets expenses with Gemini and Gmail alerts

https://n8nworkflows.xyz/workflows/track-and-analyze-google-sheets-expenses-with-gemini-and-gmail-alerts-14901


# Track and analyze Google Sheets expenses with Gemini and Gmail alerts

# Workflow Documentation: Track and analyze Google Sheets expenses with Gemini and Gmail alerts

## 1. Workflow Overview

This workflow is an automated financial monitoring system designed to ingest expense data from a Google Sheet, categorize spending using keyword matching, and generate AI-powered financial advisory reports via Google Gemini. Based on a configurable budget threshold, the system saves the analysis back to a spreadsheet and triggers differentiated Gmail alerts (Urgent High Expense vs. Normal Summary).

The logic is divided into three primary functional blocks:
- **1.1 Data Ingestion:** Extracts raw data, applies a categorization engine based on keywords, and aggregates spending per category.
- **1.2 AI Analysis and Sheet Output:** Formats data for a Large Language Model (LLM), generates a detailed advisory report using Gemini, and archives the result.
- **1.3 Email Notifications:** Evaluates the spending level against the budget and sends a tailored email notification to the user.

---

## 2. Block-by-Block Analysis

### 2.1 Data Ingestion
**Overview:** This block handles the retrieval of raw expense data and transforms it into categorized totals.
- **Nodes Involved:** `Click Here to Run`, `Read Expenses from Google Sheet`, `Settings — Change These Before Running`, `Clean and Categorise Each Expense Row`, `Add Up Total Spent per Category`.
- **Node Details:**
    - **Click Here to Run (Manual Trigger):** Entry point to start the workflow manually.
    - **Read Expenses from Google Sheet (Google Sheets):** Fetches rows from "Sheet1" of the specified spreadsheet. Requires `Date`, `Description`, and `Amount` columns.
    - **Settings — Change These Before Running (Set):** A configuration hub. Defines constants like `Budget Limit` (1000), `Currency Symbol` (Rs.), `Recipient Email`, and keyword lists for categories (Food, Transport, etc.).
    - **Clean and Categorise Each Expense Row (Set):** Uses complex ternary expressions to check if the `Description` contains specific keywords (e.g., "zomato" $\rightarrow$ Food). Normalizes descriptions to lowercase for consistency.
    - **Add Up Total Spent per Category (Summarize):** Groups the data by the `Category` field and calculates the sum of `Expense Amount`.

### 2.2 AI Analysis and Sheet Output
**Overview:** This block turns the aggregated financial data into actionable advice and stores it for historical tracking.
- **Nodes Involved:** `Build AI Prompt Fields`, `Ask Gemini to Write Expense Report`, `Collect Report Fields for Sheet and Email`, `Save Report to Sheet2`.
- **Node Details:**
    - **Build AI Prompt Fields (Set):** Prepares the payload for Gemini. It determines the `Expense Status` (High vs. Normal) by comparing the total spent against the Budget Limit from the Settings node.
    - **Ask Gemini to Write Expense Report (Google Gemini):** Uses a sophisticated prompt to act as "ARTHA," a financial advisor. It generates a report with sections: Why it's draining money, Honest Verdict, Top 3 ways to cut expense, Alternatives, Health Score, and Action for the week.
    - **Collect Report Fields for Sheet and Email (Set):** Extracts the text output from Gemini's JSON response and maps it alongside other metadata (Date, Status, etc.) for downstream use.
    - **Save Report to Sheet2 (Google Sheets):** Appends the finalized AI report and category totals to "Sheet2".

### 2.3 Email Notifications
**Overview:** This block manages the communication layer, ensuring the user is alerted based on the severity of their spending.
- **Nodes Involved:** `Keep Data Alive After Sheet Write`, `Check Expense Level — High or Normal?`, `Send High Expense Alert Email`, `Send Normal Expense Summary Email`.
- **Node Details:**
    - **Keep Data Alive After Sheet Write (Set):** Because the Google Sheets node often outputs the written row rather than the input data, this node manually recovers all necessary variables from the `Collect Report Fields` node to ensure the email has the full context.
    - **Check Expense Level (If):** A boolean check on the `Is High Expense` variable.
    - **Send High Expense Alert Email (Gmail):** Sends an urgent email with a "High Expense Detected" subject line.
    - **Send Normal Expense Summary Email (Gmail):** Sends a routine summary email.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Click Here to Run | Manual Trigger | Entry Point | None | Read Expenses... | |
| Read Expenses... | Google Sheets | Data Extraction | Click Here to Run | Settings... | Data Ingestion: Fetches raw rows from Sheet1. |
| Settings... | Set | Global Config | Read Expenses... | Clean and Cat... | Data Ingestion: Settings node holds all configurable values. |
| Clean and Categorise... | Set | Data Cleaning | Settings... | Add Up Total... | Data Ingestion: Assigns a category per row by matching keywords. |
| Add Up Total Spent... | Summarize | Aggregation | Clean and Cat... | Build AI Prompt... | Data Ingestion: Summarise adds up totals per category. |
| Build AI Prompt Fields | Set | Prompt Prep | Add Up Total... | Ask Gemini... | AI Analysis: Prepares category, amount and status for the prompt. |
| Ask Gemini... | Google Gemini | Content Generation | Build AI Prompt... | Collect Report... | AI Analysis: Gemini generates a full advisory report per category. |
| Collect Report Fields... | Set | Data Formatting | Ask Gemini... | Save Report... | AI Analysis: Gathers everything needed before writing to Sheet2. |
| Save Report to Sheet2 | Google Sheets | Data Archiving | Collect Report... | Keep Data Alive... | AI Analysis: Saves results to Sheet2. |
| Keep Data Alive... | Set | State Recovery | Save Report... | Check Expense... | Email Alerts: Recovers all fields lost after the Sheets write node. |
| Check Expense Level... | If | Routing Logic | Keep Data Alive... | Send High/Normal... | Email Alerts: Splits the flow based on budget threshold. |
| Send High Expense... | Gmail | Urgent Alert | Check Expense... | None | Email Alerts: High Expense goes to an urgent alert email. |
| Send Normal Expense... | Gmail | Routine Summary | Check Expense... | None | Email Alerts: Normal goes to a friendly summary email. |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Google Sheets Setup
1. Create a Spreadsheet with two tabs: **Sheet1** (Columns: `Date`, `Description`, `Amount`) and **Sheet2** (Columns: `Date`, `Category`, `Total Spent`, `AI Report`, `Status`, `Reviewed On`).
2. Add a **Google Sheets Node** (Read operation) to fetch data from Sheet1.

### Step 2: Configuration & Cleaning
3. Add a **Set Node** ("Settings") to define:
    - `Budget Limit` (Number)
    - `Send Report To Email` (String)
    - `Currency Symbol` (String)
    - `Keywords` for Food, Transport, Shopping, etc.
4. Add a **Set Node** ("Clean and Categorise") using an expression to check if the `Description` contains any of the keywords defined in the Settings node, assigning the corresponding category name.
5. Add a **Summarize Node** to group by `Category` and sum the `Expense Amount`.

### Step 3: AI Integration
6. Add a **Set Node** ("Build AI Prompt Fields") to calculate if `sum_Expense_Amount` > `Budget Limit` and assign a "High" or "Normal" label.
7. Add a **Google Gemini Node**. Configure the prompt to act as "ARTHA" and use the variables from the previous node (Category, Amount, Status).
8. Add a **Set Node** ("Collect Report Fields") to extract the Gemini response: `{{ $json.content.parts[0].text }}`.

### Step 4: Output & Alerting
9. Add a **Google Sheets Node** (Append operation) to write the collected fields into **Sheet2**.
10. Add a **Set Node** ("Keep Data Alive") to re-map the AI Report and Email fields (since the Sheet node output is limited).
11. Add an **If Node** to check the boolean `Is High Expense`.
12. Create two **Gmail Nodes**:
    - **True Path:** High Expense Alert (Urgent subject).
    - **False Path:** Normal Summary (Friendly subject).

### Required Credentials
- **Google Sheets OAuth2 API**
- **Google Gemini (PaLM) API Key**
- **Gmail OAuth2 API**

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Workflow Purpose | Automate financial auditing and AI-driven budgeting for Indian users. |
| AI Personality | "ARTHA" — sharp, caring, and honest financial advisor. |
| Critical Dependency | Sheet1 must have exact headers: Date, Description, Amount. |