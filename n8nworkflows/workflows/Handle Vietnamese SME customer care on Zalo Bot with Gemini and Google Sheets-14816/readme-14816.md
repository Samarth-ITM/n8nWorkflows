Handle Vietnamese SME customer care on Zalo Bot with Gemini and Google Sheets

https://n8nworkflows.xyz/workflows/handle-vietnamese-sme-customer-care-on-zalo-bot-with-gemini-and-google-sheets-14816


# Handle Vietnamese SME customer care on Zalo Bot with Gemini and Google Sheets

# Workflow Documentation: AI Customer Care Chatbot for Vietnamese SMEs (Zalo, Gemini, Google Sheets)

## 1. Workflow Overview

This workflow implements an automated customer service system tailored for Vietnamese Small and Medium Enterprises (SMEs) using the Zalo messaging platform. It combines a rule-based intent system for common business queries with a generative AI fallback for complex questions, all while maintaining a lightweight CRM via Google Sheets.

The logic is divided into the following functional blocks:

*   **1.1 Input Reception & Configuration:** Captures incoming Zalo messages and initializes business-specific variables (pricing, hotline, etc.).
*   **1.2 Customer Identification:** Validates if the sender exists in the Google Sheets CRM to personalize the experience.
*   **1.3 Onboarding Logic:** Handles first-time users by registering them in the CRM and sending a welcome sequence.
*   **1.4 Intent Detection:** A keyword-based router that categorizes messages into specific business paths (Menu, Products, Pricing, Human Support).
*   **1.5 Specialized Handlers:** Executes specific actions based on the detected intent (sending photos, text lists, or logging escalations).
*   **1.6 AI Fallback & Logging:** Uses Google Gemini to answer unrecognized queries and logs every interaction for audit and training.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Configuration
**Overview:** Acts as the entry point and the "Settings" panel for the entire workflow.
*   **Nodes Involved:** `Receive Zalo Message`, `Set Configuration`.
*   **Node Details:**
    *   `Receive Zalo Message` (Zalo Bot Trigger): Listens for `message.text.received` events. Requires Zalo Bot API credentials.
    *   `Set Configuration` (Set): Defines global constants such as `businessName`, `hotline`, `productPhotoUrl`, `pricingText`, and `googleSheetId`.
    *   **Failure Types:** Webhook delivery failure or incorrect Zalo Bot Token.

### 2.2 Customer Identification
**Overview:** Extracts user data and checks the CRM to determine if the user is a returning customer.
*   **Nodes Involved:** `Extract Customer Info`, `Lookup Customer in CRM`.
*   **Node Details:**
    *   `Extract Customer Info` (Set): Maps Zalo payload fields (sender ID, name, text) into clean variables. Converts text to lowercase for easier intent matching.
    *   `Lookup Customer in CRM` (Google Sheets): Performs a lookup in the "Customers" tab using the `senderId`.
    *   **Failure Types:** Google Sheets API quota limits or missing `senderId` in the payload.

### 2.3 Onboarding Logic
**Overview:** Differentiates between new and existing users to provide a tailored greeting.
*   **Nodes Involved:** `Is New Customer?`, `Append New Customer`, `Send Welcome Sticker`, `Send Welcome Message`, `Listen Welcome Reply`, `Normalize After Listen`.
*   **Node Details:**
    *   `Is New Customer?` (If): Checks if the lookup result for `senderId` is empty.
    *   `Append New Customer` (Google Sheets): Adds a new row to the "Customers" tab.
    *   `Send Welcome Sticker` & `Send Welcome Message` (Zalo Bot): Sends a visual sticker followed by a guided menu of options.
    *   `Listen Welcome Reply` (Zalo Bot): Uses `getUpdates` to wait for a response after the welcome message.
    *   `Normalize After Listen` (Set): Standardizes the response format before sending it to the intent detector.

### 2.4 Intent Detection
**Overview:** Routes the conversation based on Vietnamese keywords.
*   **Nodes Involved:** `Normalize Returning`, `Detect Intent`.
*   **Node Details:**
    *   `Normalize Returning` (Set): Ensures returning users follow the same data structure as new users.
    *   `Detect Intent` (Switch): Routes based on keywords:
        *   `menu`: (menu, dich vu) $\rightarrow$ Send Menu.
        *   `products`: (san pham, product, demo, giai phap) $\rightarrow$ Send Product Photo.
        *   `pricing`: (gia, price, bao gia, chi phi) $\rightarrow$ Send Pricing.
        *   `human`: (nguoi that, human, tu van, lien he, hotro) $\rightarrow$ Log Escalation.
        *   `fallback`: (All other cases) $\rightarrow$ Gemini AI.

### 2.5 Specialized Handlers
**Overview:** Provides immediate, structured answers for known business queries.
*   **Nodes Involved:** `Send Menu`, `Send Product Photo`, `Send Pricing`, `Log Escalation`, `Acknowledge Escalation`.
*   **Node Details:**
    *   `Send Menu` / `Send Pricing`: Sends pre-configured text from the Configuration node.
    *   `Send Product Photo`: Sends an image via URL with a descriptive caption.
    *   `Log Escalation` (Google Sheets): Appends a row to the "Conversations" tab with the intent `human_escalation`.
    *   `Acknowledge Escalation`: Notifies the user that a human agent will contact them.

### 2.6 AI Fallback & Logging
**Overview:** Handles complex queries using LLM and ensures a full audit trail.
*   **Nodes Involved:** `Gemini AI Reply`, `Send AI Reply`, `Log Conversation`.
*   **Node Details:**
    *   `Gemini AI Reply` (Google Gemini): Uses `gemini-2.0-flash`. The system prompt defines the persona (Professional Assistant for THE NEXOVA) and strict rules (Vietnamese, max 150 words, links to contact).
    *   `Send AI Reply` (Zalo Bot): Delivers the AI-generated text to the user.
    *   `Log Conversation` (Google Sheets): Appends the original message and AI response to the "Conversations" tab.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Zalo Message | Zalo Bot Trigger | Entry Point | - | Set Configuration | Receive and configure |
| Set Configuration | Set | Global Settings | Receive Zalo Message | Extract Customer Info | Receive and configure |
| Extract Customer Info | Set | Data Parsing | Set Configuration | Lookup Customer in CRM | Identify customer |
| Lookup Customer in CRM | Google Sheets | User Validation | Extract Customer Info | Is New Customer? | Identify customer |
| Is New Customer? | If | Logic Branching | Lookup Customer in CRM | Append New Customer / Normalize Returning | Welcome new customers |
| Append New Customer | Google Sheets | CRM Update | Is New Customer? | Send Welcome Sticker | Welcome new customers |
| Send Welcome Sticker | Zalo Bot | User Engagement | Append New Customer | Send Welcome Message | Welcome new customers |
| Send Welcome Message | Zalo Bot | Guided Onboarding | Send Welcome Sticker | Listen Welcome Reply | Welcome new customers |
| Listen Welcome Reply | Zalo Bot | Sync/Wait | Send Welcome Message | Normalize After Listen | Welcome new customers |
| Normalize After Listen | Set | Data Cleaning | Listen Welcome Reply | Detect Intent | Welcome new customers |
| Normalize Returning | Set | Data Cleaning | Is New Customer? | Detect Intent | Welcome new customers |
| Detect Intent | Switch | Request Routing | Norm. After Listen / Norm. Returning | Handlers / AI | Detect intent by keyword |
| Send Menu | Zalo Bot | Static Response | Detect Intent | - | Handle known intents |
| Send Product Photo | Zalo Bot | Media Response | Detect Intent | - | Handle known intents |
| Send Pricing | Zalo Bot | Static Response | Detect Intent | - | Handle known intents |
| Log Escalation | Google Sheets | CRM Logging | Detect Intent | Acknowledge Escalation | Handle known intents |
| Acknowledge Escalation | Zalo Bot | User Notification | Log Escalation | - | Handle known intents |
| Gemini AI Reply | Google Gemini | LLM Generation | Detect Intent | Send AI Reply | AI fallback with Gemini |
| Send AI Reply | Zalo Bot | AI Response | Gemini AI Reply | Log Conversation | Reply and log conversation |
| Log Conversation | Google Sheets | Audit Logging | Send AI Reply | - | Reply and log conversation |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Environment & Prerequisites
1.  **Host:** Use a self-hosted n8n instance.
2.  **Community Nodes:** Install `n8n-nodes-zalo-platform` via **Settings > Community Nodes**.
3.  **External Accounts:**
    *   Create a Zalo Bot via Zalo Bot Manager.
    *   Get a Google AI Studio API Key for Gemini.
    *   Set up a Google Cloud Service Account for Sheets access.
4.  **CRM Setup:** Create a Google Sheet with two tabs: `Customers` (columns: `senderId`, `name`, `firstSeen`, `lastSeen`, `messageCount`) and `Conversations` (columns: `timestamp`, `senderId`, `name`, `messageId`, `message`, `intent`, `reply`).

### Step 2: Configuration & Entry
1.  Create a **Zalo Bot Trigger** node; set event to `message.text.received`.
2.  Create a **Set** node (`Set Configuration`) to store:
    *   `googleSheetId`: Your spreadsheet ID.
    *   `businessName`, `hotline`, `pricingText`, `productPhotoUrl`, `welcomeStickerId`.
3.  Create a **Set** node (`Extract Customer Info`) to map:
    *   `senderId` $\leftarrow$ `{{ $json.message.from.id }}`
    *   `messageText` $\leftarrow$ `{{ $json.message.text.toLowerCase().trim() }}`

### Step 3: CRM Logic
1.  Add a **Google Sheets** node (`Lookup Customer in CRM`) using the `Get` operation, filtering by `senderId` in the `Customers` tab.
2.  Add an **If** node (`Is New Customer?`) to check if `senderId` is empty.
3.  **True Branch (New):** 
    *   **Google Sheets** (Append) $\rightarrow$ Add user to `Customers`.
    *   **Zalo Bot** (Send Sticker) $\rightarrow$ Use `welcomeStickerId`.
    *   **Zalo Bot** (Send Text) $\rightarrow$ Welcome message.
    *   **Zalo Bot** (Get Updates) $\rightarrow$ Listen for reply.
    *   **Set** (Normalize) $\rightarrow$ Clean incoming text for the Switch.
4.  **False Branch (Returning):**
    *   **Set** (Normalize) $\rightarrow$ Pass `messageText` directly to the Switch.

### Step 4: Intent & AI Routing
1.  Create a **Switch** node (`Detect Intent`) with 5 outputs:
    *   `menu`: contains 'menu', 'dich vu'.
    *   `products`: contains 'san pham', 'product'.
    *   `pricing`: contains 'gia', 'price'.
    *   `human`: contains 'nguoi that', 'tu van'.
    *   `ai` (Fallback): Default.
2.  **Route Outputs:**
    *   `menu/pricing` $\rightarrow$ **Zalo Bot** (Send Text).
    *   `products` $\rightarrow$ **Zalo Bot** (Send Photo).
    *   `human` $\rightarrow$ **Google Sheets** (Append to Conversations) $\rightarrow$ **Zalo Bot** (Text Acknowledgment).
    *   `ai` $\rightarrow$ **Google Gemini** (Model: `gemini-2.0-flash`). Provide the persona prompt $\rightarrow$ **Zalo Bot** (Send Text) $\rightarrow$ **Google Sheets** (Append to Conversations).

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheets CRM Template | [View Template](https://docs.google.com/spreadsheets/d/1e9155FKWikWTADXWssAdYvOq7g8l3N4NvhnxQXo9EFc/edit?usp=sharing) |
| Zalo Bot Node Setup Guide | [Nexova Setup Guide](https://thenexova.com/n8n-zalo-bot-node-complete-setup-and-operations-guide/) |
| Gemini API Access | [Google AI Studio](https://aistudio.google.com) |
| Limitation | Community nodes are not compatible with n8n Cloud. |