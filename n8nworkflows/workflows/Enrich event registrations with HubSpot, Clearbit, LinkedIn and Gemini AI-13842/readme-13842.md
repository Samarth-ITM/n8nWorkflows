Enrich event registrations with HubSpot, Clearbit, LinkedIn and Gemini AI

https://n8nworkflows.xyz/workflows/enrich-event-registrations-with-hubspot--clearbit--linkedin-and-gemini-ai-13842


# Enrich event registrations with HubSpot, Clearbit, LinkedIn and Gemini AI

This document provides a comprehensive technical analysis of the **Event Registration with Auto-Enrichment Intelligence** n8n workflow.

### 1. Workflow Overview
This workflow manages an end-to-end event registration process designed to maximize conversion by minimizing form fields. It captures basic user data via a high-speed webhook and immediately confirms the registration. Post-response, it triggers an asynchronous **Enrichment Waterfall** to gather deep professional insights (LinkedIn, Company size, Role) using multiple APIs and AI. It also handles abandoned cart tracking (via beacons) and real-time promo code validation.

#### Logical Blocks:
*   **1.1 Abandoned Cart Beacon:** Tracks when a user starts a form but hasn't submitted yet.
*   **1.2 Registration & CRM Sync:** Receives form data, validates it, and upserts the contact in HubSpot.
*   **1.3 Enrichment Waterfall (Async):** A multi-step process (Cache -> Clearbit -> Proxycurl -> Google/Gemini) to enrich the lead data.
*   **1.4 Notification & Analytics:** Sends confirmation emails, Slack alerts, and logs enriched data for reporting.
*   **1.5 Promo Code Validation:** A standalone endpoint to check code validity and usage limits.

---

### 2. Block-by-Block Analysis

#### 2.1 Abandoned Cart Beacon
*   **Overview:** Captures "form_started" events to track potential registrants who drop off.
*   **Nodes Involved:** `Receive Registration Beacon`, `Log Beacon to Data Table`, `Respond OK to Beacon`.
*   **Node Details:**
    *   **Log Beacon to Data Table:** Records `session_id`, `event_id`, and `user_agent`. Uses `continueOnFail` to ensure the beacon never breaks the UI.
    *   **Respond OK to Beacon:** Returns a 200 OK immediately to the browser.

#### 2.2 Registration & CRM Sync
*   **Overview:** The primary entry point. Validates input and ensures the lead exists in the CRM.
*   **Nodes Involved:** `Receive Registration`, `Validate Registration Data`, `Check Validation Result`, `Respond Validation Error`, `Create or Update HubSpot Contact`, `Respond Registration Success`.
*   **Node Details:**
    *   **Validate Registration Data (Code):** Normalizes names and emails; checks for required fields. Returns a `valid: true/false` flag.
    *   **Create or Update HubSpot Contact:** Uses the email as a unique identifier. Sets `leadStatus` to "NEW".
    *   **Respond Registration Success:** Sends a 200 JSON response to the user *before* enrichment starts to eliminate latency.

#### 2.3 Enrichment Waterfall
*   **Overview:** An intelligent, cost-optimized sequence that stops as soon as sufficient data is found.
*   **Nodes Involved:** `Initialize Enrichment State`, `Check Enrichment Cache`, `Cache Hit with Good Score?`, `Enrich via Clearbit`, `Parse Clearbit Results`, `Enough Data After Clearbit?`, `Enrich via Proxycurl LinkedIn`, `Parse Proxycurl Results`, `Enough Data After LinkedIn?`, `Search Google for Person`, `Extract Intelligence with Gemini`, `Merge AI Extraction Results`, `Score Data Completeness`, `Cache Enriched Profile`.
*   **Node Details:**
    *   **Check Enrichment Cache:** Hits an internal n8n Data Table to see if the email was enriched recently (score ≥ 50).
    *   **Enrich via Clearbit:** Standard Enrichment (Person + Company).
    *   **Enrich via Proxycurl:** Specifically resolves LinkedIn profiles from work emails.
    *   **Extract Intelligence with Gemini:** If APIs fail, it uses Google Search results and Gemini AI to extract structured data (Role, Industry, Location) from search snippets.
    *   **Score Data Completeness:** Calculates a 0-100 score based on field weights (e.g., Role = 20 pts).

#### 2.4 Notification & Analytics
*   **Overview:** Merges all data and executes outbound actions.
*   **Nodes Involved:** `Merge Registration and Enrichment`, `Send Confirmation Email`, `Post to Slack Registrations`, `Has Promo Code?`, `Log Promo Code Usage`, `Prepare Analytics Row`, `Log Registration Analytics`.
*   **Node Details:**
    *   **Post to Slack:** Formats a rich message with emojis including the "Enrichment Score".
    *   **Log Registration Analytics:** Stores the final "Direct Registration" record in a Data Table for BI tools.

#### 2.5 Promo Code Validation
*   **Overview:** A separate utility path for real-time frontend validation.
*   **Nodes Involved:** `Receive Promo Code Validation`, `Parse Promo Code`, `Lookup Promo Code`, `Validate Promo Code`, `Log Promo Code to Data Table`, `Respond with Promo Validation`.
*   **Node Details:**
    *   **Validate Promo Code:** Checks `expires_at`, `max_uses`, and `current_uses` from a Data Table. Returns the `discount_percent`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Registration Beacon | Webhook | Input (Beacon) | - | Log Beacon to Data Table | 1. Abandoned Cart Beacon |
| Log Beacon to Data Table | Data Table | Logging | Receive Registration Beacon | Respond OK to Beacon | 1. Abandoned Cart Beacon |
| Receive Registration | Webhook | Input (Main) | - | Validate Registration Data | 2. Validate & Create HubSpot Contact |
| Validate Registration Data | Code | Validation | Receive Registration | Check Validation Result | 2. Validate & Create HubSpot Contact |
| Create or Update HubSpot | HubSpot | CRM Sync | Check Validation Result | Respond Registration Success | 2. Validate & Create HubSpot Contact |
| Respond Registration Success| Respond to Webhook| Response | Create or Update HubSpot | - | 2. Validate & Create HubSpot Contact |
| Initialize Enrichment State | Code | Logic Init | Create or Update HubSpot | Check Enrichment Cache | 3. Enrichment Waterfall (Async) |
| Check Enrichment Cache | HTTP Request | Cache Lookup | Initialize Enrichment State | Cache Hit with Good Score? | 3. Enrichment Waterfall (Async) |
| Enrich via Clearbit | HTTP Request | Enrichment | Cache Hit... | Parse Clearbit Results | 3. Enrichment Waterfall (Async) |
| Enrich via Proxycurl | HTTP Request | Enrichment | Enough Data... | Parse Proxycurl Results | 3. Enrichment Waterfall (Async) |
| Extract Intelligence w/ Gemini| Information Extractor| AI Extraction | Search Google | Merge AI Results | 3. Enrichment Waterfall (Async) |
| Score Data Completeness | Code | Scoring | Waterfall Steps | Cache Enriched Profile | 3. Enrichment Waterfall (Async) |
| Post to Slack Registrations | Slack | Notification | Merge Reg and Enrich | - | 4. Notify & Log |
| Receive Promo Validation | Webhook | Input (Utility) | - | Parse Promo Code | 5. Promo Code Validation |
| Validate Promo Code | Code | Logic | Lookup Promo Code | Log Promo Code | 5. Promo Code Validation |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:** Create three n8n Data Tables: `enriched_profiles`, `reg_analytics`, and `promo_codes`.
2.  **Webhooks:**
    *   Create a Webhook node for `/event-registration` (POST).
    *   Create a Webhook node for `/reg-beacon` (POST).
    *   Create a Webhook node for `/validate-promo` (POST).
3.  **Validation Logic:** Add a **Code Node** after the registration webhook to validate the body. If invalid, use **Respond to Webhook** with a 400 status.
4.  **CRM Integration:** Add a **HubSpot Node**. Select "Upsert" using `Email`. Map the `First Name`, `Last Name`, and `Company` from the validation node.
5.  **Immediate Response:** Place a **Respond to Webhook** node (200 OK) immediately after HubSpot to ensure the user isn't waiting for enrichment.
6.  **The Waterfall:**
    *   **State:** Use a **Code Node** to create a JSON object containing all empty enrichment fields.
    *   **HTTP Steps:** Sequence Clearbit and Proxycurl. After each, use an **If Node** to check if `role`, `industry`, and `linkedin_url` are filled. If yes, skip to Scoring.
    *   **AI Fallback:** If fields are still missing, use an **HTTP Request** (SerpAPI) to search Google, followed by the **AI Information Extractor** node using **Gemini 2.5 Flash**.
7.  **Closing the Loop:**
    *   Use a **Code Node** to calculate the completeness score.
    *   Use a **Data Table Node** to cache the result.
    *   Use a **Merge Node** (or Code) to combine the original registration data with the new enrichment data.
8.  **Notifications:** Add **Slack**, **Email**, and **Data Table (Analytics)** nodes to record the final enriched lead.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Cost Management** | Clearbit is the most expensive; AI + Search is the cheapest fallback. |
| **Performance** | Webhook response is sent *before* the waterfall to prevent frontend timeouts. |
| **Contact / Support** | Milo Bravo | [LinkedIn Profile](https://linkedin.com/in/MiloBravo/) |
| **Feedback & Consulting** | Braia Labs | [Start Conversation](https://tally.so/r/EkKGgB) |
| **Error Handling** | `continueOnFail` is active on enrichment nodes to ensure registration flow never breaks. |