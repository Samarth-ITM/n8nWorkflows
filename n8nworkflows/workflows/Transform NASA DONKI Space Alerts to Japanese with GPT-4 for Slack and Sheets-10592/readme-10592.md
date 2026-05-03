Transform NASA DONKI Space Alerts to Japanese with GPT-4 for Slack and Sheets

https://n8nworkflows.xyz/workflows/transform-nasa-donki-space-alerts-to-japanese-with-gpt-4-for-slack-and-sheets-10592


# Transform NASA DONKI Space Alerts to Japanese with GPT-4 for Slack and Sheets

### 1. Workflow Overview

This workflow automates the monitoring of NASA's DONKI (Space Weather) alerts, classifies events by severity, translates and summarizes them in Japanese using GPT-4, and distributes notifications via Slack or logs them in Google Sheets for later review. It is designed for use cases involving space weather event monitoring, crisis communication, and record-keeping with multilingual support.

**Logical Blocks:**

- **1.1 Input Reception**: Periodic trigger and data retrieval from NASA’s DONKI API.
- **1.2 Event Analysis & Prioritization**: Deduplication, classification by event type, and severity assignment.
- **1.3 Severity-Based Routing**: Direct events to different processing branches according to severity: CRITICAL, HIGH, or other.
- **1.4 Critical Event Processing**: Translate and summarize critical alerts in Japanese and notify Slack channel.
- **1.5 High-Level Event Processing**: Summarize high-severity events in Japanese and notify Slack channel.
- **1.6 Informational Event Processing**: Summarize lower-severity events in Japanese and append structured data with summary to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Triggers the workflow every 30 minutes and fetches NASA DONKI space weather alerts from the past 24 hours.

- **Nodes Involved:**  
  - Cron  
  - NASA DONKI Notifications

- **Node Details:**  

  - **Cron**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution every 30 minutes automatically.  
    - Config: Interval set to 30 minutes using n8n’s schedule trigger parameters.  
    - Inputs: None (trigger only)  
    - Outputs: Connects to "NASA DONKI Notifications"  
    - Edge Cases: Cron misfires or system downtime could delay data fetching.

  - **NASA DONKI Notifications**  
    - Type: NASA API Node  
    - Role: Fetches notifications of space weather events from NASA DONKI API for the last 24 hours.  
    - Config: Uses dynamic date expressions for `startDate` (now minus 1 day) and `endDate` (now), formatted as 'yyyy-MM-dd'.  
    - Inputs: Trigger from Cron  
    - Outputs: Passes raw event data to next node  
    - Edge Cases: API unavailability, rate limiting, or malformed responses.

---

#### 2.2 Event Analysis & Prioritization

- **Overview:**  
Deduplicates events by ID, classifies event type and severity level (CRITICAL, HIGH, MEDIUM, INFO), and prepares data for routing.

- **Nodes Involved:**  
  - Analyze & Prioritize Events (Code)  
  - Switch

- **Node Details:**  

  - **Analyze & Prioritize Events**  
    - Type: Code Node (JavaScript)  
    - Role:  
      - Uses workflow static data to track previously seen events and avoid duplicates.  
      - Assigns severity based on event type and specific criteria (e.g., solar flare class, CME speed).  
      - Outputs enriched event objects including eventId, eventType, severity, time, link, and raw details.  
    - Key Expressions: Uses `staticData` for deduplication; derives severity with conditional logic on JSON fields like `messageType`, `classType`, and `speed`.  
    - Inputs: Raw events from NASA DONKI Notifications  
    - Outputs: Filtered and enriched event JSONs to Switch node  
    - Edge Cases: Static data storage overflow mitigated by limiting to last 1000 event IDs; missing fields in input JSON handled with defaults.

  - **Switch**  
    - Type: Switch Node  
    - Role: Routes each event based on computed severity: CRITICAL, HIGH, or other (MEDIUM/INFO).  
    - Configuration:  
      - Condition 1: severity equals "CRITICAL"  
      - Condition 2: severity equals "HIGH"  
      - Condition 3: severity not empty (catch-all for others)  
    - Inputs: From Analyze & Prioritize Events  
    - Outputs: Three output branches to respective AI Agents  
    - Edge Cases: Unrecognized severity values will fall to default branch; missing severity fields could cause routing errors.

---

#### 2.3 Critical Event Processing

- **Overview:**  
Processes CRITICAL events by generating a concise, urgent Japanese alert with GPT-4 and posting it to Slack.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - AI Agent (Critical)  
  - Slack Notify (Critical)

- **Node Details:**  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat  
    - Role: Provides GPT-4.1-mini model access for AI Agent (Critical).  
    - Config: Model set to "gpt-4.1-mini" with default options.  
    - Inputs: Passes data to AI Agent (Critical) via internal connection.  
    - Credentials: Requires OpenAI API credentials configured in n8n.  
    - Edge Cases: API rate limits, connectivity issues, or invalid prompts.

  - **AI Agent (Critical)**  
    - Type: LangChain Agent Node  
    - Role: Generates a powerful, jargon-free Japanese summary of the CRITICAL event focusing on potential impacts on Earth’s communication, power grids, and satellites.  
    - Config:  
      - Prompt instructs the agent to act as a space science crisis expert, output in Japanese only, using bullet points, and to avoid technical jargon.  
      - Uses event JSON messageBody as input.  
    - Inputs: Routed from Switch node on CRITICAL branch, uses OpenAI Chat Model as language model backend.  
    - Outputs: Passes summarized output to Slack Notify (Critical).  
    - Edge Cases: Failures in text generation or prompt errors.

  - **Slack Notify (Critical)**  
    - Type: Slack Node  
    - Role: Sends an @channel emergency notification to Slack #general with the AI-generated summary, localized event time, and official link.  
    - Config:  
      - Text includes explicit @channel mention and uses expressions to pull AI Agent output and event metadata.  
      - Channel is "#general" (selected by name).  
      - OAuth2 authentication with Slack account credentials.  
    - Inputs: From AI Agent (Critical) main output.  
    - Edge Cases: Slack API rate limits, authentication failures, or channel misconfiguration.

---

#### 2.4 High-Level Event Processing

- **Overview:**  
Processes HIGH severity events by creating a clear, informative Japanese summary and posting it to Slack with a less urgent tone.

- **Nodes Involved:**  
  - OpenAI Chat Model1  
  - AI Agent (High)  
  - Slack Notify (High)

- **Node Details:**  

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat  
    - Role: Provides GPT-4.1-mini model access for AI Agent (High).  
    - Credentials: OpenAI API credentials required.  
    - Inputs: Feeds output to AI Agent (High).  
    - Edge Cases: Same as OpenAI Chat Model.

  - **AI Agent (High)**  
    - Type: LangChain Agent Node  
    - Role: Generates a plain-language Japanese explanation of HIGH severity events, mentioning solar flare scale or CME speed when applicable.  
    - Config:  
      - Prompt instructs the agent to act as a space science expert explaining complex data simply.  
      - Output strictly in Japanese.  
    - Inputs: Routed from Switch node on HIGH branch; uses OpenAI Chat Model1.  
    - Outputs: To Slack Notify (High).  
    - Edge Cases: Text generation errors.

  - **Slack Notify (High)**  
    - Type: Slack Node  
    - Role: Posts the AI-generated summary to Slack #general channel without @channel mention but with event details and localized time.  
    - Config: OAuth2 authentication, channel "#general" by name.  
    - Inputs: From AI Agent (High).  
    - Edge Cases: Slack API or authentication issues.

---

#### 2.5 Informational Event Processing

- **Overview:**  
Summarizes non-urgent events into concise Japanese text for logging and appends structured event data plus summary to Google Sheets.

- **Nodes Involved:**  
  - OpenAI Chat Model2  
  - AI Agent (for Sheets)  
  - Merge Summary & Data (Code)  
  - Append row in sheet

- **Node Details:**  

  - **OpenAI Chat Model2**  
    - Type: LangChain OpenAI Chat  
    - Role: Provides GPT-4.1-mini model for AI Agent (for Sheets).  
    - Credentials: OpenAI API credentials required.  
    - Inputs: Feeds AI Agent (for Sheets).  
    - Edge Cases: Same as other OpenAI Chat models.

  - **AI Agent (for Sheets)**  
    - Type: LangChain Agent Node  
    - Role: Produces a 1-2 line Japanese summary focused on key facts, avoiding jargon, suitable for spreadsheet review.  
    - Config: Prompt tailored for concise, factual summaries intended for human review in sheets.  
    - Inputs: Routed from Switch node on "other" branch, uses OpenAI Chat Model2.  
    - Outputs: Passes summary to next code node.  
    - Edge Cases: Text generation errors.

  - **Merge Summary & Data (Code)**  
    - Type: Code Node (JavaScript)  
    - Role: Combines AI-generated summary with structured event fields (eventId, eventType, severity, time, link, details) into a single object for Google Sheets.  
    - Inputs: AI Agent (for Sheets) and Switch node data.  
    - Outputs: Passes merged data to Google Sheets node.  
    - Edge Cases: Missing fields or errors in JSON structure.

  - **Append row in sheet**  
    - Type: Google Sheets Node  
    - Role: Appends a new row to a specified Google Sheet ("nasa_news" tab) with columns: eventId, eventType, severity, time, link, details, and summary.  
    - Config:  
      - Document ID and Sheet Name use fixed IDs corresponding to the target spreadsheet.  
      - Columns mapped explicitly with defined schema.  
    - Credentials: Requires Google Sheets OAuth2 credentials.  
    - Inputs: Receives merged data object.  
    - Edge Cases: API quota, authentication failure, sheet access permissions, or malformed data.

---

### 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                                    | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                     |
|----------------------------|-----------------------------------|---------------------------------------------------|-----------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| Cron                       | Schedule Trigger                  | Triggers workflow every 30 minutes                | None                        | NASA DONKI Notifications       | ## Cron Triggers the workflow to run automatically every 30 minutes.                          |
| NASA DONKI Notifications   | NASA API Node                    | Fetches recent NASA DONKI space weather alerts   | Cron                        | Analyze & Prioritize Events     | ## NASA DONKI Notifications Fetches the latest space weather data from the NASA DONKI API for the past 24 hours. |
| Analyze & Prioritize Events| Code Node (JavaScript)            | Deduplicates and assigns severity to events       | NASA DONKI Notifications    | Switch                        | ## Analyze & Prioritize Events Runs custom JavaScript to deduplicate by event/message ID, classify the event type, and assign a severity label (e.g., CRITICAL, HIGH, or other) before routing. |
| Switch                     | Switch Node                      | Routes events based on severity                    | Analyze & Prioritize Events | AI Agent (Critical), AI Agent (High), AI Agent (for Sheets) | ## Switch Routes each processed event to the appropriate branch based on the computed severity: CRITICAL, HIGH, or “everything else.” |
| OpenAI Chat Model           | LangChain OpenAI Chat            | Provides GPT-4 model for critical event AI agent | AI Agent (Critical)         | AI Agent (Critical) (ai_languageModel) |                                                                                              |
| AI Agent (Critical)         | LangChain Agent                  | Creates urgent Japanese summary for critical events | Switch (CRITICAL branch), OpenAI Chat Model | Slack Notify (Critical)            | ## AI Agent (Critical) Takes CRITICAL-severity events and drafts a short, forceful Japanese alert that highlights the phenomenon, likely impact, and what to watch next. |
| Slack Notify (Critical)     | Slack Node                      | Posts critical alert to Slack #general with @channel | AI Agent (Critical)         | None                         | ## Slack Notify (Critical) Posts an @channel emergency message to Slack with the agent’s summary, the event time (localized), and the official reference link. |
| OpenAI Chat Model1          | LangChain OpenAI Chat            | Provides GPT-4 model for high event AI agent      | AI Agent (High)             | AI Agent (High) (ai_languageModel) |                                                                                              |
| AI Agent (High)             | LangChain Agent                  | Creates clear Japanese update for high severity   | Switch (HIGH branch), OpenAI Chat Model1 | Slack Notify (High)               | ## AI Agent (High) Converts HIGH-severity items into clear Japanese updates that are informative but less urgent in tone than the critical alerts. |
| Slack Notify (High)         | Slack Node                      | Posts high-level alert to Slack #general          | AI Agent (High)             | None                         | ## Slack Notify (High) Sends a Slack post for HIGH-priority events with the agent’s summary, localized time, and source link to keep teams informed. |
| OpenAI Chat Model2          | LangChain OpenAI Chat            | Provides GPT-4 for informational event AI agent   | AI Agent (for Sheets)       | AI Agent (for Sheets) (ai_languageModel) |                                                                                              |
| AI Agent (for Sheets)       | LangChain Agent                  | Creates concise Japanese summary for sheet logging | Switch ("other" branch), OpenAI Chat Model2 | Merge Summary & Data (Code)       | ## AI Agent (for Sheets) Produces a concise Japanese summary focusing on key facts only, suitable for spreadsheet rows and quick, future reference. |
| Merge Summary & Data (Code) | Code Node (JavaScript)            | Combines summary and event data for Google Sheets | AI Agent (for Sheets), Switch | Append row in sheet             | ## Merge Summary & Data (Code) Combines the Sheets-oriented summary with the event’s structured fields (ID, type, severity, time, link, details) into a single payload for the Google Sheets node. |
| Append row in sheet         | Google Sheets Node              | Appends event data and summary to Google Sheets   | Merge Summary & Data (Code) | None                         | ## Append row in sheet Appends non-urgent events to a specified Google Sheet (e.g., nasa_news) with columns such as eventType, eventId, severity, time, link, details, and a summary for later review. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Cron Trigger node**  
   - Name: `Cron`  
   - Set schedule to run every 30 minutes (minutes interval: 30).

3. **Add NASA DONKI Notifications node**  
   - Name: `NASA DONKI Notifications`  
   - Resource: `donkiNotifications`  
   - Additional Fields:  
     - `startDate`: Expression → `{{$now.minus(1, 'day').toFormat('yyyy-MM-dd')}}`  
     - `endDate`: Expression → `{{$now.toFormat('yyyy-MM-dd')}}`  
   - Connect `Cron` node output to this node.

4. **Add a Code node for event analysis and prioritization**  
   - Name: `Analyze & Prioritize Events`  
   - Paste supplied JavaScript code which:  
     - Deduplicates using workflow static data (keeps last 1000 IDs)  
     - Classifies event type and assigns severity ("CRITICAL", "HIGH", "MEDIUM", "INFO")  
   - Connect `NASA DONKI Notifications` output to this node.

5. **Add a Switch node**  
   - Name: `Switch`  
   - Add three rules on the field `severity`:  
     - Rule 1: equals "CRITICAL"  
     - Rule 2: equals "HIGH"  
     - Rule 3: not empty (default catch-all)  
   - Connect `Analyze & Prioritize Events` output to this node.

6. **Critical event branch:**

   - Add LangChain OpenAI Chat node  
     - Name: `OpenAI Chat Model`  
     - Model: `gpt-4.1-mini`  
     - Credentials: Assign OpenAI API credentials.  
   - Add LangChain Agent node  
     - Name: `AI Agent (Critical)`  
     - Prompt: Japanese crisis management expert; input event JSON; output detailed, jargon-free bullet points in Japanese.  
     - Connect `Switch` node's CRITICAL output to `AI Agent (Critical)` input.  
     - Connect `OpenAI Chat Model` node output to `AI Agent (Critical)` language model input.  
   - Add Slack node  
     - Name: `Slack Notify (Critical)`  
     - Authentication: OAuth2 with Slack account  
     - Channel: `#general` (select by name)  
     - Message: Include `@channel` mention, AI Agent output, formatted event time in Japanese locale, event type, and official link.  
     - Connect `AI Agent (Critical)` output to this Slack node.

7. **High event branch:**

   - Add LangChain OpenAI Chat node  
     - Name: `OpenAI Chat Model1`  
     - Model: `gpt-4.1-mini`  
     - Credentials: OpenAI API credentials.  
   - Add LangChain Agent node  
     - Name: `AI Agent (High)`  
     - Prompt: Japanese space science expert explaining complex data simply and including flare class or CME speed if applicable.  
     - Connect `Switch` node's HIGH output to `AI Agent (High)`.  
     - Connect `OpenAI Chat Model1` output to `AI Agent (High)` language model input.  
   - Add Slack node  
     - Name: `Slack Notify (High)`  
     - Authentication: OAuth2 Slack account  
     - Channel: `#general`  
     - Message: AI Agent output, localized time, event type, and link without @channel mention.  
     - Connect `AI Agent (High)` output to this Slack node.

8. **Informational event branch:**

   - Add LangChain OpenAI Chat node  
     - Name: `OpenAI Chat Model2`  
     - Model: `gpt-4.1-mini`  
     - Credentials: OpenAI API credentials.  
   - Add LangChain Agent node  
     - Name: `AI Agent (for Sheets)`  
     - Prompt: Japanese space science expert, creating 1-2 line non-technical summaries for spreadsheet review.  
     - Connect `Switch` node default output to `AI Agent (for Sheets)`.  
     - Connect `OpenAI Chat Model2` output to `AI Agent (for Sheets)` language model input.  
   - Add Code node  
     - Name: `Merge Summary & Data (Code)`  
     - JavaScript code merges AI summary with structured event data fields from the Switch node's corresponding event: eventId, eventType, severity, time, link, details, and summary_jp.  
     - Connect `AI Agent (for Sheets)` output and `Switch` node data to this code node.  
   - Add Google Sheets node  
     - Name: `Append row in sheet`  
     - Operation: Append  
     - Document ID: Use your Google Sheet ID (e.g., the one referenced in the workflow)  
     - Sheet Name: Use the appropriate sheet/tab name or ID (e.g., "nasa_news")  
     - Columns: Map eventId, eventType, severity, time, link, details, and summary_jp accordingly.  
     - Credentials: Google Sheets OAuth2 credentials.  
     - Connect `Merge Summary & Data (Code)` output to this node.

9. **Verify all connections:**  
   - Cron → NASA DONKI Notifications  
   - NASA DONKI Notifications → Analyze & Prioritize Events  
   - Analyze & Prioritize Events → Switch  
   - Switch CRITICAL → AI Agent (Critical) → Slack Notify (Critical)  
   - Switch HIGH → AI Agent (High) → Slack Notify (High)  
   - Switch Other → AI Agent (for Sheets) → Merge Summary & Data (Code) → Append row in sheet

10. **Set credentials:**  
    - OpenAI API credentials configured and assigned to all LangChain OpenAI Chat nodes.  
    - Slack OAuth2 credentials assigned to Slack nodes.  
    - Google Sheets OAuth2 credentials assigned to the Google Sheets node.

11. **Test execution:**  
    - Run manually or wait for Cron trigger.  
    - Confirm Slack notifications appear correctly in #general channel.  
    - Check Google Sheet rows append as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow monitors NASA space-weather events and uses GPT-4 to translate and summarize alerts into Japanese for Slack and Sheets. | Main project description.                                                                           |
| Slack messages include localized Japanese time formatting and official NASA event links.                                      | Slack Notify (Critical) and Slack Notify (High) nodes.                                             |
| Deduplication uses n8n's Static Data to prevent repeated alerts for the same event ID, limiting to 1000 stored IDs.            | Analyze & Prioritize Events node code.                                                             |
| AI prompts are strictly designed to produce output in Japanese, tailored for different severity levels and audiences.          | AI Agent nodes (Critical, High, for Sheets).                                                       |
| Google Sheets integration appends rows to a dedicated spreadsheet tab for non-urgent events to enable historical tracking.     | Append row in sheet node configuration.                                                            |
| For Slack OAuth2 authentication setup, ensure the app has permission to post messages in the target channel (#general).         | Slack Notify nodes authentication details.                                                         |
| OpenAI API usage requires proper API key management and adherence to quota limits, especially with GPT-4 model usage.          | All LangChain OpenAI Chat nodes.                                                                    |
| Localization uses JavaScript’s `toLocaleString('ja-JP')` for date/time formatting in Slack messages.                           | Slack Notify nodes message text expressions.                                                       |
| Sticky notes embedded in the canvas provide detailed documentation for each logical block and node.                             | Workflow sticky notes referenced throughout.                                                       |
| This workflow is inactive by default (`active: false`) — enable before use.                                                    | Workflow metadata.                                                                                  |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, strictly adhering to content policies and containing only legal, public data.