Track Telegram expenses with GPT-4 and Google Sheets (self-learning categories)

https://n8nworkflows.xyz/workflows/track-telegram-expenses-with-gpt-4-and-google-sheets--self-learning-categories--13667


# Track Telegram expenses with GPT-4 and Google Sheets (self-learning categories)

This technical document provides a detailed breakdown of the **AI Telegram Expense Tracker (Self-Learning Categories)** workflow. This system uses Large Language Models (LLMs) to identify, categorize, and record expenses sent via natural language messages on Telegram.

---

### 1. Workflow Overview
The workflow is designed to automate personal or shared finance tracking. It listens for Telegram messages, validates the sender, uses AI to determine if a message describes a financial transaction, and manages a dynamic list of expense categories stored in Google Sheets. If a new type of expense is detected, the AI suggests a new category, which the user can approve or edit. Finally, the structured data is saved to a Google Sheets database.

**Functional Blocks:**
*   **1.1 Input & Security:** Monitors Telegram for messages and filters them by approved Chat IDs.
*   **1.2 Expense Detection:** A lightweight AI check to determine if the message is actually an expense.
*   **1.3 Category Intelligence:** Retrieves existing categories from Google Sheets and determines if the expense fits an existing one or requires a new category.
*   **1.4 Data Extraction:** Uses GPT-4 to extract specific fields (date, amount, category, description, and shared status).
*   **1.5 Approval & Persistence:** Provides a final verification step for the user before appending the record to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Security
This block serves as the entry point and firewall for the workflow.
*   **Nodes Involved:** `Telegram - Receive Message`, `Security — Allow Approved Chat IDs`.
*   **Node Details:**
    *   **Telegram - Receive Message:** (Trigger v1.2) Triggers on every new message in the linked bot.
    *   **Security — Allow Approved Chat IDs:** (If v2.2) Compares `{{ $json.message.chat.id }}` against a hardcoded list of authorized numbers.
    *   **Edge Cases:** Unauthorized users will be silently ignored. Empty messages or non-text updates (like stickers) are handled in the subsequent AI block.

#### 2.2 AI Detection Layer
Determines if the text contains a description of money being spent.
*   **Nodes Involved:** `AI — Detect Expense Message`, `Parse — Expense Detection Output`, `Filter — Only Continue If Expense`.
*   **Node Details:**
    *   **AI — Detect Expense Message:** (OpenAI v2.1) Uses `gpt-4.1-nano` to return a boolean `is_expense`. It requires both a description and a numeric amount to be true.
    *   **Parse — Expense Detection Output:** (Set v3.4) Cleans the AI output string into a JSON object.
    *   **Filter — Only Continue If Expense:** (If v2.3) Stops the workflow if the AI determines the message is not an expense.

#### 2.3 Category Intelligence
The "self-learning" core of the workflow. It manages the classification taxonomy.
*   **Nodes Involved:** `Sheets — Load Existing Categories`, `Format — Build Category Prompt`, `AI — Classify Expense Category`, `Decision — Category Exists?`, `Telegram — Ask To Create New Category`, `Decision — Category Approved?`, `Telegram — Edit New Category Form`, `Sheets — Add Suggested/Edited Category`.
*   **Node Details:**
    *   **Sheets — Load Existing Categories:** Fetches the current list of categories, descriptions, and examples from a dedicated Sheet.
    *   **AI — Classify Expense Category:** (OpenAI v2.1) Compares the incoming message against the fetched list. If no match is found, it generates a proposal for a new category.
    *   **Telegram — Ask To Create New Category:** (Telegram v1.2) Uses `sendAndWait` with double-approval buttons.
    *   **Telegram — Edit New Category Form:** (Telegram v1.2) If the user rejects the AI suggestion, it opens a custom Telegram form to manually enter the category name and description.
    *   **Persistence Nodes:** `Sheets — Add Suggested Category` or `Sheets — Add Edited Category` append the new classification rules back to the Google Sheet for future use.

#### 2.4 Expense Extraction
Transforms natural language into structured data.
*   **Nodes Involved:** `Format — Rebuild Category Prompt`, `Merge — Expense + Categories`, `AI — Extract Structured Expense Data`, `Parse — Expense Extraction Output`.
*   **Node Details:**
    *   **AI — Extract Structured Expense Data:** (OpenAI v2.1) Uses GPT-4 to map the message to the following schema: `date` (YYYY-MM-DD), `amount` (float), `category`, `description`, and `common_expense` (boolean).
    *   **Logic:** It uses the updated category list (including any newly created ones) as context. It detects "shared" keywords (e.g., "we", "split") to toggle the `common_expense` flag.

#### 2.5 Approval & Persistence
The final human-in-the-loop safety check.
*   **Nodes Involved:** `Telegram — Confirm Expense Before Save`, `Prepare — Add Person Name`, `Filter — Only Approved Expenses`, `Sheets — Save Expense`, `Telegram — Expense Saved Confirmation`.
*   **Node Details:**
    *   **Telegram — Confirm Expense Before Save:** (Telegram v1.2) Sends the extracted data back to the user via `sendAndWait`. The user must click "YES" to proceed.
    *   **Prepare — Add Person Name:** (Set v3.4) Captures the Telegram `first_name` of the sender to identify who logged the expense.
    *   **Sheets — Save Expense:** (Google Sheets v4.7) Appends the final data to the main tracking sheet.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Telegram - Receive Message | telegramTrigger | Entry Point | (None) | Security — Allow... | Add your Telegram Credentials to the following Nodes: Receive Message |
| Security — Allow Approved Chat IDs | if | Security Filter | Telegram - Receive... | AI — Detect Expense... | Replace Telegram Chat IDs in following Security Nodes: Allow Approved Chat IDs |
| AI — Detect Expense Message | openAi | Intent Detection | Security — Allow... | Parse — Expense... | Add your AI Credentials to the following Node: Detect Expense Message |
| Parse — Expense Detection Output | set | Data Formatting | AI — Detect... | Filter — Only Continue... | (AI Detection Layer description) |
| Filter — Only Continue If Expense | if | Logic Gate | Parse — Expense... | Sheets — Load... | (AI Detection Layer description) |
| Sheets — Load Existing Categories | googleSheets | Data Fetching | Filter — Only Continue... | Format — Build... | Connect to the Sheet that contains the category data |
| AI — Classify Expense Category | openAi | Classification | Format — Build... | Parse — Category... | Add your AI Credentials to the following Node: Classify Expense Category |
| Telegram — Ask To Create New Category | telegram | User Interaction | Decision — Category... | Decision — Category... | Add your Telegram Credentials to the following Nodes: Ask To Create New Category |
| Telegram — Edit New Category Form | telegram | Manual Input | Decision — Category... | Sheets — Add Edited... | Add your Telegram Credentials to the following Nodes: Edit New Category Form |
| AI — Extract Structured Expense Data | openAi | Data Extraction | Merge — Expense... | Parse — Expense... | Add your AI Credentials to the following Node: Extract Structured Expense Data |
| Telegram — Confirm Expense Before Save | telegram | Final Approval | Parse — Expense... | Prepare — Add... | Add your Telegram Credentials to the following Nodes: Confirm Expense Before Save |
| Sheets — Save Expense | googleSheets | Data Persistence | Filter — Only Approved... | Telegram — Expense... | Connect to the Sheet that contains the category data in following Sheet Nodes: Save Expense |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:** Create a Google Sheet with two tabs:
    *   `EXPENSES`: Columns: `date`, `amount`, `category`, `description`, `common_expense`, `Person`.
    *   `EXPENSE_CATEGORIES`: Columns: `category`, `description`, `examples`.
2.  **Trigger Setup:** Add a **Telegram Trigger** node. Connect your Bot API Token and set the update to `message`.
3.  **Security Filter:** Add an **If** node to check if `{{ $json.message.chat.id }}` matches your Telegram ID.
4.  **Detection AI:** Add an **OpenAI** node (Model: GPT-4). Use a System Prompt to return JSON: `{"is_expense": boolean}` based on whether the message has an amount and description.
5.  **Category Logic:**
    *   Add a **Google Sheets** node to "Get Many" rows from the `EXPENSE_CATEGORIES` tab.
    *   Add a **Code** node to format these rows into a string for the AI prompt.
    *   Add an **OpenAI** node to classify the message. If no category matches, instruct it to return `match: false` and suggest a new one.
6.  **Human Feedback Loop:**
    *   Add a **Telegram** node (Operation: Send and Wait). Use "Double" approval type for "Create Category?".
    *   If "No", use another **Telegram** node with "Response Type: Custom Form" to let the user type the category details.
    *   Add **Google Sheets** nodes to append new categories to the `EXPENSE_CATEGORIES` sheet.
7.  **Data Extraction:**
    *   Add an **OpenAI** node (Model: GPT-4). Pass the original message and the current category list.
    *   Prompt the AI to extract `date`, `amount`, `category`, `description`, and `common_expense` in valid JSON.
8.  **Final Confirmation:**
    *   Add a **Telegram** node (Operation: Send and Wait) to show the user the extracted fields.
    *   If "YES", add a **Set** node to grab the user's name (`message.from.first_name`).
    *   Add a **Google Sheets** node (Operation: Append) to save all fields to the `EXPENSES` tab.
9.  **Completion:** End with a **Telegram** node sending a "Success" message.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Multi-User Capability** | By adding multiple Chat IDs to the Security node, several users can track expenses to the same sheet. |
| **Shared Expenses** | The AI looks for keywords like "we" or "split" to mark the `common_expense` column as `true`. |
| **Natural Language Support** | The system is optimized for German and English natural language processing. |
| **Learning Mechanism** | Every time a new category is confirmed, the Google Sheet is updated, making the "Category Intelligence" block smarter for the next execution. |