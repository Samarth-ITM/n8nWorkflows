Turn WhatsApp into an AI shopping and price comparison assistant using Wati, OpenAI, Serper, and Google Sheets

https://n8nworkflows.xyz/workflows/turn-whatsapp-into-an-ai-shopping-and-price-comparison-assistant-using-wati--openai--serper--and-google-sheets-13722


# Turn WhatsApp into an AI shopping and price comparison assistant using Wati, OpenAI, Serper, and Google Sheets

This document provides a comprehensive technical analysis of the n8n workflow: **Turn WhatsApp into an AI shopping and price comparison assistant.**

---

### 1. Workflow Overview
This workflow transforms WhatsApp into a personal shopping assistant. It enables users to manage shopping lists, compare real-time prices across the web using AI, and receive automated alerts when significant discounts are found.

**The logic is divided into three core pipelines:**
*   **Pipeline A (Intake):** Receives natural language shopping lists via WhatsApp, parses them using OpenAI, and stores structured items in Google Sheets.
*   **Pipeline B (Price Engine):** Triggered by the keyword "compare." It searches the live web for every item on the user's list, identifies the best deals via AI, and returns a formatted comparison card.
*   **Pipeline C (Daily Alerts):** A scheduled routine that checks for saved deals with at least 10% savings and sends a proactive notification to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Intent Routing
This block acts as the entry point for all user interactions via WhatsApp.
*   **Nodes Involved:** `Wati Trigger`, `Intent Router1`
*   **Node Details:**
    *   **Wati Trigger:** A Webhook-based trigger that listens for the `messageReceived` event from the Wati platform.
    *   **Intent Router1 (Switch Node):** Analyzes the incoming text to route the flow to specific logic branches:
        *   Starts with `list:` → Pipeline A
        *   Matches `compare` → Pipeline B
        *   Matches `mylist` → View current list
        *   Matches `deals` → View price history
        *   Matches `clear` → List maintenance

#### 2.2 Pipeline A: List Intake & Parsing
Processes natural language text into a structured database format.
*   **Nodes Involved:** `OpenAI – Parse List1`, `Structure List Items1`, `Google Sheets – Save Shopping List1`, `Send a text message5`
*   **Node Details:**
    *   **OpenAI – Parse List1:** Uses `gpt-4o-mini` to convert text (e.g., "milk 2L and eggs") into a JSON array with keys: `item`, `qty`, `unit`, `category`.
    *   **Structure List Items1 (Code):** Assigns a unique `listId`, formats the data for Google Sheets, and generates a user-friendly confirmation message with emojis.
    *   **Google Sheets Node:** Appends rows to the "Shopping List" sheet.
    *   **Send a text message5 (Wati):** Sends the structured confirmation back to the user.

#### 2.3 Pipeline B: Price Comparison Engine
The most complex block, performing live web searches and AI evaluation.
*   **Nodes Involved:** `Google Sheets – Read List for Compare1`, `Prepare Search Queries1`, `Serper – Search Prices1`, `OpenAI – Extract Best Deal1`, `Process Deal Result1`, `Google Sheets – Save Deals1`, `Aggregate Comparison Card1`, `Send a text message6`
*   **Node Details:**
    *   **Prepare Search Queries1:** Filters active items and creates optimized search strings (e.g., "best price milk 2L buy online India").
    *   **Serper – Search Prices1:** Performs a Google Shopping search via the Serper.dev API.
    *   **OpenAI – Extract Best Deal1:** Analyzes the raw search results to find the lowest price from a reputable store and calculates potential savings.
    *   **Aggregate Comparison Card1:** Uses `$input.all()` to wait for all items to be processed, then calculates the grand total and builds a single comprehensive WhatsApp message.

#### 2.4 Pipeline C: Daily Deal Alerts
Automated background task for proactive user engagement.
*   **Nodes Involved:** `Schedule Trigger – Daily 8AM Alerts1`, `Google Sheets – Read Deals for Alert1`, `Build Alert Messages1`, `Send a text message`
*   **Node Details:**
    *   **Schedule Trigger:** Set to run every day at 08:00 AM.
    *   **Build Alert Messages1:** Filters the "Deals" sheet for rows where `alertSent = 'No'` and `savingPct >= 10`. It groups these by phone number to send one message per user.
    *   **Edge Case:** The JSON indicates a manual "To-do": an Update node should be added after the final message to set `alertSent` to 'Yes' to prevent duplicate alerts.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Wati Trigger | Wati Trigger | Entry Point | - | Intent Router1 | Captures incoming WhatsApp messages via Wati. |
| Intent Router1 | Switch | Routing | Wati Trigger | Various | The Switch node parses text to determine what the user wants (compare, deals, mylist, clear). |
| OpenAI – Parse List1 | HTTP Request | AI Extraction | Intent Router1 | Structure List Items1 | OpenAI parses every item with qty/unit/category. |
| Structure List Items1 | Code | Data Formatting | OpenAI – Parse List1 | Google Sheets – Save... | Generates one row per item + ackMessage. |
| Google Sheets – Save Shopping List1 | Google Sheets | Database | Structure List Items1 | Send a text message5 | Appends items to Shopping List tab. |
| Google Sheets – Read List for Compare1 | Google Sheets | Data Fetching | Intent Router1 | Prepare Search Queries1 | Reads the Shopping List for the current user. |
| Serper – Search Prices1 | HTTP Request | Web Search | Prepare Search Queries1 | OpenAI – Extract... | Serper searches live web prices. |
| OpenAI – Extract Best Deal1 | HTTP Request | AI Analysis | Serper – Search Prices1 | Process Deal Result1 | OpenAI picks best deal from search results. |
| Aggregate Comparison Card1 | Code | Data Aggregation | Google Sheets – Save Deals1 | Send a text message6 | Collects ALL items, calculates total, builds full card. |
| Schedule Trigger – Daily 8AM Alerts1 | Schedule | Automation | - | Google Sheets – Read Deals... | Runs automatically every day at 8:00 AM. |
| Build Alert Messages1 | Code | Logic/Filtering | Google Sheets – Read Deals... | Send a text message | Filters: alertSent = No AND savingPct ≥ 10. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Create a Google Sheet with two tabs: `Shopping List` (columns: listid, phone, username, item, qty, unit, category, addedAt, Status) and `Deals` (columns: listid, phone, item, storeName, price, originalPrice, savingPct, Currency, url, summary, confidence, alternatives, foundAt, alertSent).
    *   Obtain API Keys for: **Wati**, **OpenAI**, and **Serper.dev**.

2.  **Setup Trigger & Router:**
    *   Add a **Wati Trigger** node (Event: `messageReceived`).
    *   Connect it to a **Switch Node**. Create rules for string starts with `list:` and exact matches for `compare`, `deals`, `mylist`, and `clear`.

3.  **Build Pipeline A (Intake):**
    *   Add an **HTTP Request** node to call OpenAI. Use a System Prompt to extract JSON from the user message.
    *   Add a **Code Node** to generate a `listId` and map the JSON array into individual n8n items.
    *   Add a **Google Sheets** node to `Append` the data.
    *   Add a **Wati Node** to send a "List Saved" confirmation.

4.  **Build Pipeline B (Comparison):**
    *   Add a **Google Sheets** node to `Read` rows where the phone number matches the trigger.
    *   Add a **Code Node** to transform list items into search strings.
    *   Add an **HTTP Request** node for Serper.dev (Google Shopping endpoint).
    *   Add an **HTTP Request** node (OpenAI) to interpret the Serper JSON and find the cheapest reputable offer.
    *   Add a **Code Node** to format the specific item's "Deal Card."
    *   Add a **Google Sheets** node to log the deal in the `Deals` tab.
    *   Add a **Code Node** using `$input.all()` to combine all items into one summary message.
    *   Add a **Wati Node** to send the final report.

5.  **Build Pipeline C (Alerts):**
    *   Add a **Schedule Trigger** (Cron: `0 8 * * *`).
    *   Add a **Google Sheets** node to read the `Deals` tab.
    *   Add a **Code Node** to filter for `savingPct > 10` and `alertSent == 'No'`.
    *   Add a **Wati Node** to send the alert.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Serper API** | Free tier available at [serper.dev](https://serper.dev) (2500 searches/month). |
| **Credential Requirements** | Requires WATI API, OpenAI Header Auth, Serper Header Auth, and Google Sheets OAuth2. |
| **Aggregation Logic** | Pipeline B uses `Aggregate` to ensure the user receives ONE message for the whole list, rather than one message per item. |
| **Maintenance** | To avoid duplicate alerts, users should add a Google Sheets 'Update' node after the Daily Alert to mark deals as sent. |