Manage creator sponsorship and brand deals with Gmail, OpenAI, Notion and Telegram

https://n8nworkflows.xyz/workflows/manage-creator-sponsorship-and-brand-deals-with-gmail--openai--notion-and-telegram-13837


# Manage creator sponsorship and brand deals with Gmail, OpenAI, Notion and Telegram

### 1. Workflow Overview

The **Creator Sponsorship & Brand Deal OS** is a comprehensive automation suite designed for content creators and talent managers. It streamlines the lifecycle of a brand deal—from the initial inquiry to daily relationship management and monthly financial performance analysis.

The workflow is divided into three distinct functional flows:
*   **1.1 Flow A: Inbound Email Detection:** Monitors Gmail for incoming sponsorship inquiries, uses AI to extract structured data (budget, platform, campaign type), logs the inquiry into a Notion CRM, and prepares a draft response.
*   **1.2 Flow B: Daily Follow-up System:** A proactive system that checks the Notion CRM every morning for deals requiring follow-up, generates personalized AI reminders, and sends them to Telegram.
*   **1.3 Flow C: Monthly Revenue Report:** A financial reporting module that aggregates closed deals at the end of each month, calculates performance metrics, and uses AI to provide strategic business coaching insights.

---

### 2. Block-by-Block Analysis

#### 2.1 Flow A — Inbound Email Detection
This block acts as the entry point for new business opportunities. It filters noise from the inbox and ensures every lead is captured and categorized automatically.

*   **Nodes Involved:** `📧 Gmail Trigger`, `🔧 Extract Email Data`, `🧠 LLM - Classify and Extract`, `🤖 OpenAI - Classifier`, `⚙️ Parse AI Response`, `🔍 Is Sponsorship?`, `🔎 Notion - Search Deal`, `📋 Deal Already Exists?`, `✅ Notion - Create Deal`, `🔄 Notion - Update Deal`, `🔀 Merge After Save`, `✍️ LLM - Draft Reply`, `🤖 OpenAI - Reply Drafter`, `📱 Telegram - New Deal Alert`.
*   **Node Details:**
    *   **📧 Gmail Trigger:** Polls the inbox every minute. Requires OAuth2 credentials. Potential for rate limits if the inbox is extremely high-volume.
    *   **🧠 LLM - Classify and Extract:** A Chain LLM node using `gpt-4o` (via `🤖 OpenAI - Classifier`) to transform unstructured email text into a structured JSON object. It identifies if the email is a sponsorship and extracts details like "Proposed Budget" and "Campaign Type".
    *   **⚙️ Parse AI Response (Code):** A Javascript node that sanitizes the AI output, handling potential formatting errors (like markdown code blocks) and merging it with the original email metadata.
    *   **🔍 Is Sponsorship? (IF):** Filters out non-business emails. Only `true` results proceed to the CRM logging phase.
    *   **🔎 Notion - Search Deal:** Queries the Notion database using the Company Name to prevent duplicate entries.
    *   **✍️ LLM - Draft Reply:** Generates a professional first-contact email. It is configured with a temperature of 0.7 to ensure the tone remains friendly and human-like.
    *   **📱 Telegram - New Deal Alert:** Sends a formatted Markdown message to the creator with the deal summary and the AI-generated draft ready for copying.

#### 2.2 Flow B — Daily Follow-up System
This block ensures that no lead goes cold by automating the "check-in" process for active negotiations.

*   **Nodes Involved:** `⏰ Daily 9AM Trigger`, `📋 Notion - Get Follow-ups`, `📌 Has Follow-ups Today?`, `🔁 Loop Each Deal`, `🔧 Extract Deal Info`, `✍️ LLM - Follow-up Draft`, `🤖 OpenAI - Follow-up`, `📱 Telegram - Follow-up Alert`.
*   **Node Details:**
    *   **⏰ Daily 9AM Trigger:** A Schedule node set to fire every 24 hours.
    *   **📋 Notion - Get Follow-ups:** Retrieves all pages where the `Follow-up Date` property equals the current date.
    *   **🔁 Loop Each Deal (Split in Batches):** Processes each deal individually to avoid context mixing in the AI node.
    *   **✍️ LLM - Follow-up Draft:** Uses the previous deal notes and stage to write a context-aware follow-up (max 80 words).
    *   **📱 Telegram - Follow-up Alert:** Notifies the user on Telegram with the specific contact email and the follow-up text.

#### 2.3 Flow C — Monthly Revenue Report
This block provides a high-level executive summary of the creator's business health.

*   **Nodes Involved:** `📅 Monthly Report Trigger`, `💰 Notion - Get Paid Deals`, `📊 Calculate Revenue Stats`, `✍️ LLM - Monthly Summary`, `🤖 OpenAI - Report Writer`, `📱 Telegram - Monthly Report`, `📧 Gmail - Monthly Report Email`.
*   **Node Details:**
    *   **📊 Calculate Revenue Stats (Code):** A Javascript node that filters Notion data for the previous calendar month. It calculates Total Revenue, Deal Count, Average Deal Value, and creates breakdowns by Channel (e.g., YouTube vs. Instagram).
    *   **✍️ LLM - Monthly Summary:** Acts as a "Business Coach." It analyzes the raw stats and writes a 3-4 sentence narrative celebrating wins or suggesting improvements.
    *   **📧 Gmail - Monthly Report Email:** Sends a formal HTML report containing a summary table and the AI insights to the user’s email address.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 📧 Gmail Trigger | n8n-nodes-base.gmailTrigger | Entry Point (Inbound) | None | 🔧 Extract Email Data | Polls Gmail every minute for new emails |
| 🔧 Extract Email Data | n8n-nodes-base.set | Data Prep | 📧 Gmail Trigger | 🧠 LLM - Classify and Extract | Extracts company name, contact, budget... |
| 🧠 LLM - Classify and Extract | @n8n/n8n-nodes-langchain.chainLlm | AI Classification | 🔧 Extract Email Data | ⚙️ Parse AI Response | AI classifies email + extracts structured data. |
| ⚙️ Parse AI Response | n8n-nodes-base.code | Data Sanitization | 🧠 LLM - Classify and Extract | 🔍 Is Sponsorship? | AI classifies email + extracts structured data. |
| 🔍 Is Sponsorship? | n8n-nodes-base.if | Logic Filter | ⚙️ Parse AI Response | 🔎 Notion - Search Deal | AI classifies email + extracts structured data. |
| 🔎 Notion - Search Deal | n8n-nodes-base.notion | Duplicate Check | 🔍 Is Sponsorship? | 📋 Deal Already Exists? | Searches for existing deal. |
| 📋 Deal Already Exists? | n8n-nodes-base.if | Routing | 🔎 Notion - Search Deal | ✅ Notion - Create / 🔄 Notion - Update | Searches for existing deal. |
| ✅ Notion - Create Deal | n8n-nodes-base.notion | Data Storage | 📋 Deal Already Exists? | 🔀 Merge After Save | Creates new or updates existing entry. |
| 🔄 Notion - Update Deal | n8n-nodes-base.notion | Data Storage | 📋 Deal Already Exists? | 🔀 Merge After Save | Creates new or updates existing entry. |
| ✍️ LLM - Draft Reply | @n8n/n8n-nodes-langchain.chainLlm | Content Generation | 🔀 Merge After Save | 📱 Telegram - New Deal Alert | Drafts a professional reply... |
| ⏰ Daily 9AM Trigger | n8n-nodes-base.scheduleTrigger | Entry Point (Daily) | None | 📋 Notion - Get Follow-ups | Runs automatically every day at 9 AM |
| 📋 Notion - Get Follow-ups | n8n-nodes-base.notion | Data Retrieval | ⏰ Daily 9AM Trigger | 📌 Has Follow-ups Today? | Fetches all Notion deals where Follow-up Date = today |
| 🔁 Loop Each Deal | n8n-nodes-base.splitInBatches | Iterator | 📌 Has Follow-ups Today? | 🔧 Extract Deal Info | Loops through each deal... |
| ✍️ LLM - Follow-up Draft | @n8n/n8n-nodes-langchain.chainLlm | Content Generation | 🔧 Extract Deal Info | 📱 Telegram - Follow-up Alert | Generates personalized follow-up draft per deal. |
| 📅 Monthly Report Trigger | n8n-nodes-base.scheduleTrigger | Entry Point (Monthly) | None | 💰 Notion - Get Paid Deals | Runs on the 1st of every month at 8 AM |
| 📊 Calculate Revenue Stats | n8n-nodes-base.code | Data Aggregation | 💰 Notion - Get Paid Deals | ✍️ LLM - Monthly Summary | Calculates total revenue, deal count, avg deal value... |
| ✍️ LLM - Monthly Summary | @n8n/n8n-nodes-langchain.chainLlm | AI Analysis | 📊 Calculate Revenue Stats | 📱 Telegram - Monthly Report | Writes monthly revenue narrative with insights and tips. |

---

### 4. Reproducing the Workflow from Scratch

#### Prerequisites
1.  **Notion Database:** Create a database with the following properties:
    *   `Company Name` (Title)
    *   `Contact Person` (Rich Text)
    *   `Contact Email` (Email)
    *   `Budget` (Rich Text)
    *   `Campaign Type` (Select: Integration, Dedicated Video, Newsletter Shoutout, UGC, Affiliate)
    *   `Channel` (Select: YouTube, Newsletter, X, Instagram, Multiple)
    *   `Deal Stage` (Select: New Lead, New Activity, Negotiating, Paid)
    *   `Last Contact` (Date)
    *   `Follow-up Date` (Date)
    *   `Revenue` (Number)
    *   `Notes` (Rich Text)
2.  **Telegram Bot:** Create a bot via `@BotFather` and obtain your `Chat ID`.

#### Step-by-Step Instructions

**1. Set up Flow A (Inbound):**
*   Add a **Gmail Trigger** node; set to "Every Minute".
*   Add a **Set** node to map `from.address`, `subject`, and `text` from Gmail.
*   Add a **Basic LLM Chain** node. Connect an **OpenAI Chat Model** node. Use the prompt to extract JSON containing `is_sponsorship`, `company_name`, and `proposed_budget`.
*   Add a **Code** node to parse the JSON output using `JSON.parse()`.
*   Add an **IF** node to check if `is_sponsorship` is true.
*   Add a **Notion** node (Get Many) to search by `Company Name`. Use an **IF** node to check if an ID was returned.
*   Add **Notion Create** and **Notion Update** nodes respectively. Map the extracted fields to the Notion properties.
*   Add another **Basic LLM Chain** to draft the reply.
*   Add a **Telegram** node to send the summary and draft.

**2. Set up Flow B (Follow-ups):**
*   Add a **Schedule Trigger**; set to 09:00 daily.
*   Add a **Notion** node (Get Many); filter where `Follow-up Date` equals `{{ $today }}`.
*   Add a **Split In Batches** node to handle multiple follow-ups.
*   Add a **Basic LLM Chain** to write the follow-up text based on `Notes` and `Deal Stage`.
*   Add a **Telegram** node to send the alert.

**3. Set up Flow C (Reports):**
*   Add a **Schedule Trigger**; set to the 1st of every month.
*   Add a **Notion** node (Get Many); filter where `Deal Stage` is "Paid".
*   Add a **Code** node. Use Javascript to calculate the previous month's date range and sum the `Revenue` field.
*   Add a **Basic LLM Chain** with a prompt to act as a business coach summarizing the stats.
*   Add a **Telegram** node and a **Gmail (Send)** node to distribute the report.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Database ID Setup | You must replace `YOUR_NOTION_DATABASE_ID` in all three Notion nodes. |
| Chat ID Setup | You must replace `YOUR_TELEGRAM_CHAT_ID` in all three Telegram nodes. |
| Token Usage | Using `gpt-4o` for extraction and `gpt-3.5-turbo` (or similar) for drafting can optimize costs. |
| Gmail API Scopes | Ensure your Gmail credential has `https://www.googleapis.com/auth/gmail.readonly` and `https://www.googleapis.com/auth/gmail.send`. |