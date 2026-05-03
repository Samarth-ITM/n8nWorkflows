Run email outreach campaigns from Telegram with Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/run-email-outreach-campaigns-from-telegram-with-gmail-and-google-sheets-13732


# Run email outreach campaigns from Telegram with Gmail and Google Sheets

This document provides a technical breakdown of the **Automated Email Outreach** workflow, designed to manage cold email campaigns using Telegram as an interface, Google Sheets as a database, and Gmail for delivery.

---

### 1. Workflow Overview
The workflow serves as a lightweight, self-hosted CRM and outreach tool. It allows users to initiate campaigns via a Telegram bot, which then triggers a sequence that pulls leads from a Google Sheet, sends personalized HTML emails, and tracks engagement (opens) and opt-outs (unsubscribes).

**Logical Blocks:**
*   **1.1 Campaign Initiation (Telegram):** Receives the `/outreach` command, collects campaign details via a form, and records the new campaign.
*   **1.2 Campaign Processing & Delivery:** Monitors the spreadsheet for new "draft" campaigns, fetches active leads, and sends emails one-by-one with randomized delays to mimic human behavior.
*   **1.3 Engagement Tracking:** A webhook-based system that captures email opens via a tracking pixel and updates campaign statistics.
*   **1.4 Subscription Management:** A webhook-based system that processes unsubscribe requests to ensure GDPR/CAN-SPAM compliance.

---

### 2. Block-by-Block Analysis

#### 2.1 Launch Campaign from Telegram
This block provides the user interface for starting a campaign.
*   **Nodes Involved:** Listen for Telegram Commands, If Outreach Command, Prompt Campaign via Telegram, Map Campaign Fields, Create Campaign Row.
*   **Details:**
    *   **Listen for Telegram Commands:** A Telegram Trigger looking for message updates.
    *   **If Outreach Command:** Validates that the message is `/outreach` and checks the `chat.id` (1750928632) to prevent unauthorized use.
    *   **Prompt Campaign via Telegram:** Uses the `sendAndWait` operation. It presents a custom form with fields: Sender Email (Dropdown), Subject, Content (Textarea), and Cap (Limit).
    *   **Create Campaign Row:** Appends the form data to the "Dashboard" sheet with a default status of "draft".

#### 2.2 Send Email Campaign
This is the engine of the workflow, triggered by changes in the spreadsheet.
*   **Nodes Involved:** Watch for New Campaigns, If Status Is Draft, Route by Sender Email, Fetch Subscribed Leads, Cap Lead Count, Map Lead Fields, Loop Over Leads, Build HTML Template, Send Email via Gmail, Update Send Stats, Wait Random Delay.
*   **Details:**
    *   **Watch for New Campaigns:** Polls the Google Sheet every minute for new rows.
    *   **Route by Sender Email:** A Switch node that branches based on the chosen sender address to maintain correct sender identity.
    *   **Fetch Subscribed Leads:** Queries the "Leads" sheet for rows where `Subscribe == "yes"`.
    *   **Loop Over Leads:** A `Split in Batches` node processing leads one by one.
    *   **Build HTML Template:** Takes the raw content and wraps it in a professional, responsive HTML structure.
    *   **Send Email via Gmail:** Sends the message. It includes an `<img>` tag at the bottom (tracking pixel) and an unsubscribe link pointing to the workflow's webhook URLs.
    *   **Wait Random Delay:** Executes a Code node to generate a 1–5 minute delay between sends to protect Gmail sender reputation.

#### 2.3 Track Email Opens
Handles incoming "hits" from the tracking pixel.
*   **Nodes Involved:** Track Email Open (Webhook), Extract Tracking Params, Lookup Campaign Row, Lookup Open Logs, Check If Already Opened, If Not Already Logged, Increment Open Count, Log Open Event.
*   **Details:**
    *   **Tracking Logic:** When the webhook is triggered, it extracts the Campaign ID and Lead Email.
    *   **Duplicate Prevention:** A Code node (`Check If Already Opened`) looks through a "logs" sheet. If that specific lead has already opened that specific campaign, it stops to avoid inflated stats.
    *   **Analytics Update:** Increments the `Opened` column in the Dashboard for the respective campaign.

#### 2.4 Handle Unsubscribe Requests
Manages the opt-out process.
*   **Nodes Involved:** Handle Unsubscribe (Webhook), Extract Unsubscribe ID, Lookup Lead Record, Mark as Unsubscribed.
*   **Details:**
    *   **Mark as Unsubscribed:** Updates the "Leads" sheet, setting the `Subscribe` column to "no" for the specific Lead ID.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Listen for Telegram Commands | Telegram Trigger | Entry Point | - | If Outreach Command | 1. Launch Campaign from Telegram |
| If Outreach Command | If | Authorization | Listen for Telegram Commands | Prompt Campaign via Telegram | 1. Launch Campaign from Telegram |
| Prompt Campaign via Telegram | Telegram | User Input Form | If Outreach Command | Map Campaign Fields | 1. Launch Campaign from Telegram |
| Map Campaign Fields | Set | Data Formatting | Prompt Campaign via Telegram | Create Campaign Row | 1. Launch Campaign from Telegram |
| Create Campaign Row | Google Sheets | Data Storage | Map Campaign Fields | - | 1. Launch Campaign from Telegram |
| Watch for New Campaigns | Google Sheets Trigger | Polling Trigger | - | If Status Is Draft | 2. Send Email Campaign |
| If Status Is Draft | If | Logic Filter | Watch for New Campaigns | Route by Sender Email | 2. Send Email Campaign |
| Route by Sender Email | Switch | Routing | If Status Is Draft | Fetch Subscribed Leads | 2. Send Email Campaign |
| Fetch Subscribed Leads | Google Sheets | Data Retrieval | Route by Sender Email | Cap Lead Count | 2. Send Email Campaign |
| Cap Lead Count | Limit | Safety Throttle | Fetch Subscribed Leads | Map Lead Fields | 2. Send Email Campaign |
| Map Lead Fields | Set | Mapping | Cap Lead Count | Loop Over Leads | 2. Send Email Campaign |
| Loop Over Leads | Split In Batches | Iterator | Map Lead Fields, Wait | Build HTML / Mark Done | 2. Send Email Campaign |
| Build HTML Template | HTML | Content Creation | Loop Over Leads | Send Email via Gmail | 2. Send Email Campaign |
| Send Email via Gmail | Gmail | Delivery | Build HTML Template | Fetch Current Stats | 2. Send Email Campaign |
| Update Send Stats | Google Sheets | Progress Tracking | Fetch Current Stats | Generate Random Delay | 2. Send Email Campaign |
| Generate Random Delay | Code | Anti-Spam | Update Send Stats | Wait Random Delay | 2. Send Email Campaign |
| Wait Random Delay | Wait | Delivery Pacing | Generate Random Delay | Loop Over Leads | 2. Send Email Campaign |
| Track Email Open | Webhook | Tracking Trigger | - | Extract Tracking Params | 3. Track Email Opens |
| Extract Tracking Params | Set | Data Extraction | Track Email Open | Lookup Campaign Row | 3. Track Email Opens |
| Lookup Open Logs | Google Sheets | Data Retrieval | Lookup Campaign Row | Check If Already Opened | 3. Track Email Opens |
| Check If Already Opened | Code | Logic | Lookup Open Logs | If Not Already Logged | 3. Track Email Opens |
| Increment Open Count | Google Sheets | Analytics | If Not Already Logged | Log Open Event | 3. Track Email Opens |
| Handle Unsubscribe | Webhook | Opt-out Trigger | - | Extract Unsubscribe ID | 4. Handle Unsubscribe Requests |
| Mark as Unsubscribed | Google Sheets | Compliance | Lookup Lead Record | - | 4. Handle Unsubscribe Requests |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:** Create a Google Sheet with three tabs:
    *   `Dashboard`: Columns: `Sn`, `Email`, `Subject`, `Content`, `Cap`, `Status`, `Sent`, `Failed`, `Opened`.
    *   `Leads`: Columns: `Id`, `Email`, `Name`, `Subscribe`.
    *   `logs`: Columns: `id`, `email`, `date`, `seen`.
2.  **Telegram Bot:** Create a bot via BotFather, get the API Token, and find your Chat ID.
3.  **Telegram Trigger:** Set up the "Listen for Telegram Commands" node. Connect an "If" node to filter `text` for `/outreach` and `chat.id` for your ID.
4.  **Telegram Form:** Add "Prompt Campaign via Telegram" with `operation: sendAndWait`. Create the form fields.
5.  **Campaign Storage:** Connect a Google Sheets node to append the form results to the `Dashboard` tab.
6.  **Polling:** Create the "Watch for New Campaigns" node (polling the `Dashboard` sheet). Filter for `Status == draft`.
7.  **Leads Logic:** Use a Google Sheets node to "Get Many Rows" from the `Leads` tab where `Subscribe == yes`.
8.  **Batching:** Use "Split in Batches" (Batch Size: 1). Inside the loop:
    *   **HTML:** Use an HTML node to wrap `{{ $json.content }}`.
    *   **Gmail:** Configure Gmail OAuth2. In the body, append a hidden image: `<img src="WEB_HOOK_URL/open?id={{CAMPAIGN_ID}}&email={{LEAD_EMAIL}}"/>`.
    *   **Stats:** After sending, use Google Sheets "Update" to increment the `Sent` count in the Dashboard.
    *   **Delay:** Use a Code node with `Math.random()` to generate a delay, followed by a Wait node.
9.  **Tracking Webhooks:** 
    *   Create a Webhook node (GET) for `/open`.
    *   Use Google Sheets nodes to check the `logs` tab for existing `id` + `email` combinations.
    *   Update the `Dashboard` sheet's `Opened` column if the log is new.
10. **Unsubscribe Webhook:**
    *   Create a Webhook node (GET) for `/unsubscribe`.
    *   Update the `Leads` sheet to set `Subscribe = no` for the matching ID.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Feedback & Consulting Form | [Start the conversation here](https://tally.so/r/EkKGgB) |
| Author LinkedIn | [Milo Bravo](https://linkedin.com/in/MiloBravo/) |
| Organizational Credit | BRaiA Labs |
| Anti-Spam Tip | Ensure your Gmail daily send limits are respected (usually 2,000 for Workspace, 500 for personal). |