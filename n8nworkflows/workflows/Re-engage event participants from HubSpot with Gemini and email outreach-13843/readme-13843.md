Re-engage event participants from HubSpot with Gemini and email outreach

https://n8nworkflows.xyz/workflows/re-engage-event-participants-from-hubspot-with-gemini-and-email-outreach-13843


# Re-engage event participants from HubSpot with Gemini and email outreach

This technical reference document provides a comprehensive breakdown of the "Participant Re-engager for Event Registration" n8n workflow.

---

### 1. Workflow Overview

The purpose of this workflow is to automate a re-engagement campaign targeting past event attendees stored in HubSpot. It identifies eligible "alumni," segments them based on historical engagement (frequency of attendance and deal activity), and uses Google Gemini AI to generate highly personalized outreach emails.

**The logic is organized into the following functional blocks:**
*   **1.1 Trigger & Configuration:** Initializes the campaign variables and starts the process.
*   **1.2 Data Retrieval & Filtering:** Queries HubSpot for past participants and excludes those already registered for the upcoming event.
*   **1.3 Segmentation Logic:** Categorizes contacts into tiers (Champion, Returning, One-timer) to inform the AI's tone.
*   **1.4 AI Personalization:** Uses LLM extraction to generate structured email content.
*   **1.5 Outreach & Reporting:** Sends the emails, logs the activity, and notifies the team via Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Configuration
*   **Overview:** Sets the global variables for the specific event being promoted.
*   **Nodes Involved:** `Start Alumni Campaign`, `Set Campaign Config`.
*   **Node Details:**
    *   **Start Alumni Campaign (Manual Trigger):** Used for on-demand execution. Can be replaced by a Schedule Trigger for automated "8-12 weeks out" campaigns.
    *   **Set Campaign Config (Set Node):** Defines `event_id`, `event_name`, `event_date`, and `reg_url`. These are used throughout the workflow for filtering and email content.

#### 2.2 Data Retrieval & Filtering
*   **Overview:** Fetches historical data from HubSpot and cleans it for the re-engagement logic.
*   **Nodes Involved:** `Fetch Past Attendees from HubSpot`, `Filter Already Registered`, `Has Eligible Alumni?`.
*   **Node Details:**
    *   **Fetch Past Attendees (HTTP Request):** Performs a POST to HubSpot’s CRM Search API. It filters for contacts where the custom property `event_registration` is known.
    *   **Filter Already Registered (Code Node):** Compares the HubSpot `event_registration` list against the current `event_id`. It also normalizes the HubSpot object into a flat structure (e.g., extracting `firstname`, `jobtitle`, etc.).
    *   **Has Eligible Alumni? (If Node):** A safety check to ensure the workflow stops if no contacts meet the criteria, preventing errors in downstream AI nodes.

#### 2.3 Segmentation Logic
*   **Overview:** Determines the "relationship strength" of the contact.
*   **Nodes Involved:** `Segment Alumni by Engagement`.
*   **Node Details:**
    *   **Segment Alumni (Code Node):** Calculates `events_count`.
        *   **Champion:** 3+ events or 2+ associated deals.
        *   **Returning:** 2 events.
        *   **One-timer:** 1 event.
    *   This node also injects the campaign config data (event name, date) into every item for the AI node's context.

#### 2.4 AI Personalization
*   **Overview:** Generates personalized HTML email bodies and subject lines.
*   **Nodes Involved:** `Generate Personalized Email with Gemini`, `Google Gemini for Alumni`.
*   **Node Details:**
    *   **Generate Personalized Email (Information Extractor):** Takes the contact’s profile and segment to request a specific JSON schema: `email_body` and `subject_line`.
    *   **Google Gemini for Alumni (Chat Model):** Uses `gemini-2.5-flash-lite`. Configured with a temperature of 0.7 to allow for creative yet professional writing. It adapts the tone based on the segment (VIP language for Champions vs. value-propositions for One-timers).

#### 2.5 Outreach & Reporting
*   **Overview:** Executes the communication and provides visibility to the user.
*   **Nodes Involved:** `Send Alumni Email`, `Log Alumni Outreach`, `Post Campaign Summary to Slack`.
*   **Node Details:**
    *   **Send Alumni Email (Email Send):** Sends the AI-generated HTML content. It uses `continueOnFail: true` to ensure the workflow proceeds even if a specific email address bounces or has a delivery error.
    *   **Log Alumni Outreach (Code Node):** Formats a summary object containing the `hubspot_id`, `segment`, and `sent_at` timestamp.
    *   **Post Campaign Summary (Slack):** Sends a single summary message to a specific channel detailing how many alumni were contacted and for which event.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Start Alumni Campaign | Manual Trigger | Entry Point | - | Set Campaign Config | Set your event details here before running. |
| Set Campaign Config | Set | Variable Init | Start Alumni Campaign | Fetch Past Attendees | Set your event details here before running. |
| Fetch Past Attendees | HTTP Request | Data Fetching | Set Campaign Config | Filter Already Registered | Searches HubSpot CRM via the Search API for contacts where event_registration is set. |
| Filter Already Registered | Code | Data Cleaning | Fetch Past Attendees | Has Eligible Alumni? | Excludes contacts already registered for the current event_id. |
| Has Eligible Alumni? | If | Logic Gate | Filter Already Registered | Segment Alumni | Excludes contacts already registered for the current event_id. |
| Segment Alumni | Code | Classification | Has Eligible Alumni? | Generate Email | Categorizes alumni into three tiers: Champion, Returning, One-timer. |
| Generate Email | Info Extractor | AI Processing | Segment Alumni | Send Alumni Email | Gemini Flash Lite generates unique email copy for each alumnus. |
| Google Gemini | Gemini Chat | AI Engine | - | Generate Email | Gemini Flash Lite generates unique email copy for each alumnus. |
| Send Alumni Email | Email Send | Outreach | Generate Email | Log Alumni Outreach | HTML email with alumni-exclusive CTA. Uses continueOnFail. |
| Log Alumni Outreach | Code | Reporting | Send Alumni Email | Post Summary | Records HubSpot ID, segment, event, timestamp. |
| Post Summary | Slack | Notification | Log Alumni Outreach | - | Posts campaign stats to your registrations channel. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Ensure HubSpot has a custom property `event_registration` (text).
    *   Gather credentials for HubSpot (API Key/Header Auth), Google Gemini (API Key), SMTP (for email), and Slack (OAuth2).

2.  **Step 1: Setup Config**
    *   Add a **Manual Trigger**.
    *   Connect a **Set Node** (`Set Campaign Config`). Create four string assignments: `event_id`, `event_name`, `event_date`, and `reg_url`.

3.  **Step 2: HubSpot Integration**
    *   Add an **HTTP Request Node**. Set method to `POST`, URL to `https://api.hubapi.com/crm/v3/objects/contacts/search`.
    *   Authentication: Header Auth (`Authorization: Bearer [API_KEY]`).
    *   Body: Use a JSON filter searching for contacts where `event_registration` is `HAS_PROPERTY`.

4.  **Step 3: Filtering & Segmentation**
    *   Add a **Code Node** to filter out users where `event_registration` already contains the `event_id` from Step 1.
    *   Add an **If Node** to check if the array length is > 0.
    *   Add a **Code Node** to assign segments. Logic: If `events_attended.length >= 3` → 'champion'; `length == 2` → 'returning'; else 'one-timer'.

5.  **Step 4: AI Configuration**
    *   Add the **Information Extractor Node** (LangChain).
    *   Set the schema to include `email_body` (string) and `subject_line` (string).
    *   Attach a **Google Gemini Chat Model Node** to the Information Extractor. Use `gemini-2.5-flash-lite`.

6.  **Step 5: Execution & Logging**
    *   Add the **Email Send Node**. Map the subject and body from the AI node’s output. Set "Continue on Fail" to true.
    *   Add a **Code Node** to create a log entry (ID, email, segment, timestamp).
    *   Add a **Slack Node**. Configure it to send a message to your desired channel ID using expressions to count the total items processed.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Strategy Tip** | Alumni convert at 3-5x the rate of cold prospects according to Julius Solaris. |
| **Pagination Warning** | For lists >100, implement HubSpot pagination with the `after` cursor. |
| **Feedback/Consulting** | Get in touch with Milo Bravo for custom workflows. [Tally Form](https://tally.so/r/EkKGgB) |
| **Companion Template** | Works best with the "Event Registration + Auto-Enrichment Intelligence" workflow. |
| **LinkedIn Profile** | Connect with the author. [Milo Bravo LinkedIn](https://linkedin.com/in/MiloBravo/) |