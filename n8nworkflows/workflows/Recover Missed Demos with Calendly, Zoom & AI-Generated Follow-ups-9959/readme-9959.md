Recover Missed Demos with Calendly, Zoom & AI-Generated Follow-ups

https://n8nworkflows.xyz/workflows/recover-missed-demos-with-calendly--zoom---ai-generated-follow-ups-9959


# Recover Missed Demos with Calendly, Zoom & AI-Generated Follow-ups

### 1. Workflow Overview

This workflow automates the process of tracking scheduled demo meetings via Calendly, validating attendance through Zoom, and managing follow-ups for no-show attendees using AI-generated messages. It is designed to improve demo recovery rates by integrating Calendly, Zoom, AI (OpenAI), Slack notifications, optional email outreach, and CRM updates.

The workflow is logically divided into two main paths:

- **1.1 Booking Path (Booking Tracking):**  
  Handles incoming Calendly webhook events for new demo bookings, extracts relevant Zoom meeting data, filters for specific demo event types, and saves booking details into a database.

- **1.2 Attendance Path (Attendance Check):**  
  Triggered by Zoom meeting end webhooks, this path validates the webhook, extracts the meeting ID, retrieves booking data from the database, fetches Zoom meeting participants, verifies attendance, updates the booking status accordingly, and initiates follow-up actions for no-shows.

Additional functional blocks include:

- **1.3 Setup and Configuration:**  
  One-time setup nodes to create Calendly webhook subscriptions, validate Zoom webhooks, and configure OAuth credentials.

- **1.4 AI Personalization & Follow-up Actions:**  
  Generates AI-based personalized follow-up messages for no-shows and triggers parallel actions: Slack notifications, optional recovery emails, and optional CRM updates.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Booking Path (Booking Tracking)

**Overview:**  
This block listens for new demo bookings via a Calendly webhook, extracts Zoom meeting details and attendee info, filters only demo event types, and stores booking data in an internal database for later reference.

**Nodes Involved:**  
- Calendly Booking Webhook  
- Extract Booking Data  
- Filter: Demo Events Only  
- Save Booking to Database  

**Node Details:**

- **Calendly Booking Webhook**  
  - Type: Webhook  
  - Role: Receives POST notifications from Calendly when a new event is scheduled.  
  - Config: HTTP POST, path `/cal-uri-get`.  
  - Inputs: External HTTP POST from Calendly webhooks.  
  - Outputs: Raw webhook payload.

- **Extract Booking Data**  
  - Type: Code  
  - Role: Parses the Calendly webhook payload to extract Zoom meeting ID, join URL, password, event details, and attendee info.  
  - Key logic: Throws error if no Zoom meeting ID found (ensures only Zoom meetings are processed).  
  - Inputs: JSON payload from Calendly Booking Webhook.  
  - Outputs: Structured booking data including meeting_id (integer), email, name, status ('pending'), timestamps.

- **Filter: Demo Events Only**  
  - Type: Filter  
  - Role: Allows only bookings matching a specific Calendly event type URI (configured by the user) to pass through.  
  - Key config: Replace `"YOUR_CALENDLY_EVENT_TYPE_URI"` with your actual demo event type URI.  
  - Inputs: Extracted booking data.  
  - Outputs: Booking data only for demo event types.

- **Save Booking to Database**  
  - Type: Data Table  
  - Role: Inserts booking record into n8n's internal Data Table with columns: meeting_id, email, status ('pending').  
  - Config: User must create and specify Data Table ID with appropriate schema.  
  - Inputs: Filtered booking data.  
  - Outputs: Confirmation of database insert.

**Potential Failures:**  
- Missing or invalid Zoom meeting ID in booking data (throws error in code node).  
- Incorrect Calendly event type URI filter configuration causing no bookings to pass.  
- Data Table misconfiguration or missing table causing insert failures.  
- Webhook delivery failures from Calendly.

---

#### 2.2 Attendance Path (Attendance Check)

**Overview:**  
On Zoom meeting end webhook, this block validates the webhook, extracts the meeting ID, retrieves stored booking info, fetches Zoom participants using Zoom API, verifies if the expected attendee joined, updates the attendance status in the database, and triggers follow-up steps if the attendee missed the demo.

**Nodes Involved:**  
- Zoom Meeting Ended Webhook  
- Filter: Meeting Ended Events  
- Extract Meeting ID from Zoom  
- Get Zoom Access Token  
- Get Booking from Database  
- Merge Access Token with Booking Data  
- Get Zoom Participants  
- Check if Attendee Showed Up  
- Update Attendance Status  
- Filter: No-Shows Only  
- AI Generate Follow-Up Messages  
- Notify Team in Slack  
- Send Recovery Email (disabled)  
- Update CRM Deal (disabled)  

**Node Details:**

- **Zoom Meeting Ended Webhook**  
  - Type: Webhook  
  - Role: Receives Zoom webhook calls when meetings end.  
  - Config: HTTP POST, path `/zoom-meeting-ended`.  
  - Inputs: Zoom event payloads.

- **Filter: Meeting Ended Events**  
  - Type: Filter  
  - Role: Ensures only events where `event == "meeting.ended"` proceed.  
  - Inputs: Zoom webhook JSON.  
  - Outputs: Meeting ended events only.

- **Extract Meeting ID from Zoom**  
  - Type: Code  
  - Role: Parses Zoom webhook payload to extract meeting ID (integer) and UUID, adds timestamp.  
  - Throws error if meeting ID missing.  
  - Inputs: Filtered Zoom webhook event.  
  - Outputs: JSON with meeting_id, meeting_uuid, ended_at.

- **Get Zoom Access Token**  
  - Type: HTTP Request  
  - Role: Obtains OAuth access token via Zoom Server-to-Server OAuth credentials.  
  - Config: POST to Zoom OAuth endpoint with account_id, client_id, client_secret.  
  - Inputs: Meeting ID JSON.  
  - Outputs: OAuth token JSON.  
  - Potential failure: Invalid credentials, expired tokens, network errors.

- **Get Booking from Database**  
  - Type: Data Table (Get operation)  
  - Role: Retrieves booking record matching the Zoom meeting ID.  
  - Inputs: Meeting ID.  
  - Outputs: Booking details JSON.

- **Merge Access Token with Booking Data**  
  - Type: Merge  
  - Role: Combines Zoom access token and booking data into one JSON object for next API call.  
  - Inputs: Zoom access token and booking data.  
  - Outputs: Combined JSON.

- **Get Zoom Participants**  
  - Type: HTTP Request  
  - Role: Calls Zoom API to retrieve participant list from past meeting.  
  - Config: GET `https://api.zoom.us/v2/past_meetings/{{meeting_id}}/participants` with Bearer token.  
  - Inputs: Merged JSON containing meeting_id and access token.  
  - Outputs: Participants array.

- **Check if Attendee Showed Up**  
  - Type: Code  
  - Role: Compares expected attendee email from booking with Zoom participants' emails (case-insensitive).  
  - Outputs: Adds `attended` boolean, participant count, and participant names list.  
  - Inputs: Participants list and booking data.

- **Update Attendance Status**  
  - Type: Data Table (Update operation)  
  - Role: Updates the booking record's status to `true` or `false` based on attendance.  
  - Inputs: JSON with meeting_id, email, attended status.

- **Filter: No-Shows Only**  
  - Type: Filter  
  - Role: Passes only records where `attended == false`.  
  - Inputs: Updated booking data with attendance.

- **AI Generate Follow-Up Messages**  
  - Type: OpenAI (LangChain) node  
  - Role: Generates personalized follow-up email subject, body, and LinkedIn message using GPT-4o.  
  - Config: User must connect OpenAI credentials and customize prompt as needed.  
  - Inputs: No-show booking data including email, meeting ID, participant info.  
  - Outputs: JSON with email_subject, email_body, linkedin_message.

- **Notify Team in Slack**  
  - Type: Slack  
  - Role: Sends a formatted message alerting the team about the no-show and suggested follow-ups.  
  - Config: OAuth credentials, Slack channel ID required.  
  - Inputs: AI-generated messages and booking data.

- **Send Recovery Email (Disabled)**  
  - Type: Email Send  
  - Role: Sends automated recovery email to no-show attendee.  
  - Config: User must replace sender email and configure email credentials. Disabled by default.  
  - Inputs: Email subject and body generated by AI.

- **Update CRM Deal (Disabled)**  
  - Type: HubSpot  
  - Role: Updates the CRM deal record for the no-show attendee (optional).  
  - Config: Requires HubSpot OAuth and deal ID mapping. Disabled by default.

**Potential Failures:**  
- Zoom webhook validation failure or timeout.  
- Missing or invalid meeting ID in Zoom webhook.  
- OAuth token retrieval failure (invalid credentials, rate limits).  
- Zoom API call failures or empty participant lists.  
- Email or Slack credentials misconfigured or rate-limited.  
- AI node failures (API key issues, prompt errors).  
- Data Table update errors (missing record or schema mismatch).

---

#### 2.3 Setup and Configuration

**Overview:**  
One-time setup nodes to initialize Calendly webhook subscriptions and validate Zoom webhook security, plus instructions and checklist for proper configuration.

**Nodes Involved:**  
- Manual Setup Trigger  
- Get Calendly Organization  
- Extract Organization URI  
- Create Calendly Webhook  
- Show Setup Success  
- Zoom Webhook Validator  
- Send Validation Response to Zoom  
- Zoom Validation Webhook (Duplicate - Disabled)  
- Various sticky notes providing setup instructions and guides  

**Node Details:**

- **Manual Setup Trigger**  
  - Type: Manual Trigger  
  - Role: Starts the one-time setup process.

- **Get Calendly Organization**  
  - Type: HTTP Request  
  - Role: Fetches current user info including organization URI from Calendly API using Personal Access Token.  
  - Config: Requires Authorization header with Bearer token.

- **Extract Organization URI**  
  - Type: Code  
  - Role: Parses Calendly user response to extract organization URI, user URI, name, and email.  
  - Throws error if no organization found (e.g., personal account).

- **Create Calendly Webhook**  
  - Type: HTTP Request  
  - Role: Creates webhook subscription for invitee.created event on Calendly organization.  
  - Config: POST JSON with webhook URL pointing to `/cal-uri-get` webhook node.  
  - Requires Calendly API token and n8n webhook URL.  
  - Outputs webhook subscription details.

- **Show Setup Success**  
  - Type: Code  
  - Role: Displays success message and webhook details or throws error if creation failed.

- **Zoom Webhook Validator**  
  - Type: Code  
  - Role: Responds to Zoom webhook validation challenge by computing HMAC SHA-256 hash of the plainToken using Zoom webhook secret.  
  - Must respond within 3 seconds to Zoom validation requests.

- **Send Validation Response to Zoom**  
  - Type: Respond To Webhook  
  - Role: Sends the encrypted token response to Zoom as required for webhook validation.

- **Zoom Validation Webhook (Duplicate)**  
  - Disabled duplicate webhook node for testing Zoom validation, recommended to delete after setup.

- **Sticky Notes**  
  - Provide detailed instructions for setup, Zoom validation, database schema, OAuth credential requirements, and overall checklist.

**Potential Failures:**  
- Incorrect Calendly API token or n8n URL causing webhook subscription failure.  
- Zoom webhook secret misconfigured causing validation failures.  
- Manual trigger not executed before scheduling live events, leading to missing webhook subscriptions.  
- Setup instructions not followed causing incomplete integrations.

---

#### 2.4 AI Personalization & Follow-Up Actions

**Overview:**  
Generates AI-driven personalized follow-up messages for no-show attendees and executes parallel actions: notifying the sales team via Slack, optionally sending recovery emails, and optionally updating CRM deals.

**Nodes Involved:**  
- AI Generate Follow-Up Messages  
- Notify Team in Slack  
- Send Recovery Email (disabled)  
- Update CRM Deal (disabled)  

**Node Details:**

- **AI Generate Follow-Up Messages**  
  - Type: OpenAI (LangChain)  
  - Role: Produces JSON-formatted follow-up email subject, email body, and LinkedIn message.  
  - Config: GPT-4o model, prompt includes attendee email, meeting info, participant details, and no-show status.  
  - Outputs structured JSON for downstream use.

- **Notify Team in Slack**  
  - Type: Slack  
  - Role: Sends notification with demo no-show details and AI-suggested messages.  
  - Requires Slack OAuth credentials and channel ID configuration.  
  - Supports markdown formatting.

- **Send Recovery Email (Disabled)**  
  - Type: Email Send  
  - Role: Sends automated follow-up email to the no-show attendee.  
  - Disabled by default; requires sender email and email provider credentials configuration.

- **Update CRM Deal (Disabled)**  
  - Type: HubSpot  
  - Role: Updates CRM deal record to log no-show activity or trigger workflows.  
  - Disabled by default; requires HubSpot OAuth and deal mapping.

**Potential Failures:**  
- AI API key missing or quota exceeded causing generation failure.  
- Slack OAuth misconfiguration or channel ID errors causing notification failure.  
- Email credentials missing causing email node failure if enabled.  
- CRM integration misconfigured or deal ID missing causing update failure.

---

### 3. Summary Table

| Node Name                      | Node Type                      | Functional Role                          | Input Node(s)                     | Output Node(s)                                | Sticky Note                                                                                                     |
|--------------------------------|--------------------------------|----------------------------------------|----------------------------------|-----------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Overview                       | Sticky Note                    | Workflow high-level description        |                                  |                                               | ## 📋 WORKFLOW OVERVIEW ... Two Main Paths: Booking and Attendance                                              |
| Setup Instructions             | Sticky Note                    | One-time setup instructions             |                                  |                                               | ## ⚙️ ONE-TIME SETUP (Run Once) ... Create Calendly webhook subscription                                         |
| Zoom Validation Guide          | Sticky Note                    | Zoom webhook validation instructions    |                                  |                                               | ## 🔴 CRITICAL: ZOOM VALIDATION ... Must respond within 3 seconds                                               |
| Path 1 Explanation             | Sticky Note                    | Explains Booking Path flow               |                                  |                                               | ## 📥 PATH 1: BOOKING TRACKING ... Save to database for later lookup                                            |
| Path 2 Explanation             | Sticky Note                    | Explains Attendance Path flow            |                                  |                                               | ## 🔍 PATH 2: ATTENDANCE CHECK ... Update database with attendance status                                       |
| AI Configuration              | Sticky Note                    | AI message generation customization     |                                  |                                               | ## 🤖 AI PERSONALIZATION ... Need to parse JSON fields after AI node                                            |
| Follow-up Actions             | Sticky Note                    | Describes follow-up parallel actions    |                                  |                                               | ## 📤 FOLLOW-UP ACTIONS ... Slack, email, CRM update; all run simultaneously                                    |
| Database Configuration        | Sticky Note                    | Database table schema and usage         |                                  |                                               | ## 💾 DATABASE SETUP ... meeting_id must match Calendly & Zoom                                                  |
| Zoom Auth Setup              | Sticky Note                    | Zoom Server-to-Server OAuth setup       |                                  |                                               | ## 🔐 ZOOM OAUTH SETUP ... Keep credentials secret                                                              |
| Setup Checklist              | Sticky Note                    | Pre-activation checklist                 |                                  |                                               | ## ⚠️ SETUP CHECKLIST ... Before activating workflow                                                            |
| Calendly Booking Webhook      | Webhook                       | Receives Calendly booking events        |                                  | Extract Booking Data                           | PATH 1: Receives booking data when a Calendly event is scheduled                                                |
| Extract Booking Data          | Code                          | Parses Calendly payload for Zoom details| Calendly Booking Webhook         | Filter: Demo Events Only                       | Parses Calendly webhook payload and extracts Zoom meeting details                                               |
| Filter: Demo Events Only      | Filter                        | Filters only demo event types            | Extract Booking Data             | Save Booking to Database                       | ⚠️ CONFIGURE: Replace 'YOUR_CALENDLY_EVENT_TYPE_URI'                                                             |
| Save Booking to Database      | Data Table                   | Saves booking record                     | Filter: Demo Events Only         |                                               | ⚠️ CONFIGURE: Create Data Table with columns meeting_id, email, status                                          |
| Zoom Meeting Ended Webhook    | Webhook                       | Receives Zoom meeting ended events      |                                  | Filter: Meeting Ended Events                   | PATH 2: Receives webhook when Zoom meeting ends                                                                 |
| Filter: Meeting Ended Events | Filter                        | Allows only 'meeting.ended' Zoom events | Zoom Meeting Ended Webhook       | Extract Meeting ID from Zoom                   | Ensures we only process meeting.ended events from Zoom                                                           |
| Extract Meeting ID from Zoom | Code                          | Extracts meeting ID from Zoom webhook   | Filter: Meeting Ended Events     | Get Zoom Access Token, Get Booking from Database | Parses Zoom webhook to get meeting ID                                                                           |
| Get Zoom Access Token         | HTTP Request                  | Retrieves Zoom OAuth access token       | Extract Meeting ID from Zoom     | Merge Access Token with Booking Data           | ⚠️ CONFIGURE: Replace with your Zoom OAuth credentials                                                           |
| Get Booking from Database     | Data Table                   | Retrieves booking by meeting ID          | Extract Meeting ID from Zoom     | Merge Access Token with Booking Data           | Retrieves original booking details using Zoom meeting ID                                                        |
| Merge Access Token with Booking Data | Merge                  | Combines token and booking data          | Get Zoom Access Token, Get Booking from Database | Get Zoom Participants                   | Combines Zoom token and booking data for Zoom API call                                                          |
| Get Zoom Participants         | HTTP Request                  | Gets participant list from Zoom API     | Merge Access Token with Booking Data | Check if Attendee Showed Up                   | Fetches list of participants from Zoom API                                                                      |
| Check if Attendee Showed Up   | Code                          | Checks if expected attendee joined      | Get Zoom Participants            | Update Attendance Status, Filter: No-Shows Only | Compares attendee email with Zoom participants                                                                  |
| Update Attendance Status      | Data Table                   | Updates attendance status in database   | Check if Attendee Showed Up      | Filter: No-Shows Only                          | Updates database with attendance status                                                                          |
| Filter: No-Shows Only         | Filter                        | Filters only no-show attendees           | Update Attendance Status         | AI Generate Follow-Up Messages                 | Only processes meetings where attendee did NOT show up                                                           |
| AI Generate Follow-Up Messages| OpenAI (LangChain)            | Generates personalized follow-up messages| Filter: No-Shows Only            | Notify Team in Slack, Send Recovery Email, Update CRM Deal | ⚠️ CONFIGURE: Connect OpenAI API, customize prompt                                                               |
| Notify Team in Slack          | Slack                         | Sends no-show alert to sales team        | AI Generate Follow-Up Messages   |                                               | ⚠️ CONFIGURE: Connect Slack OAuth, select channel, customize message                                             |
| Send Recovery Email           | Email Send                   | Sends follow-up email (disabled)         | AI Generate Follow-Up Messages   |                                               | ⚠️ CONFIGURE: Replace sender email, enable this node to send automated emails                                   |
| Update CRM Deal (Optional)    | HubSpot                      | Updates CRM deal record (disabled)       | AI Generate Follow-Up Messages   |                                               | ⚠️ OPTIONAL: Enable and configure for CRM updates                                                               |
| Manual Setup Trigger          | Manual Trigger               | One-time setup initiation                 |                                  | Get Calendly Organization                      | Run once to create Calendly webhook subscription                                                                 |
| Get Calendly Organization     | HTTP Request                  | Gets Calendly user and org info          | Manual Setup Trigger             | Extract Organization URI                        | ⚠️ CONFIGURE: Replace with Calendly API token                                                                    |
| Extract Organization URI      | Code                          | Extracts organization URI from Calendly response | Get Calendly Organization       | Create Calendly Webhook                         | Parses Calendly response to get organization URI                                                                 |
| Create Calendly Webhook       | HTTP Request                  | Creates webhook subscription in Calendly | Extract Organization URI         | Show Setup Success                              | ⚠️ CONFIGURE: Replace with n8n webhook URL and Calendly token                                                    |
| Show Setup Success            | Code                          | Confirms webhook creation success        | Create Calendly Webhook          |                                               | Displays webhook subscription details on success                                                                |
| Zoom Webhook Validator        | Code                          | Handles Zoom webhook validation challenge| Zoom Validation Webhook (Duplicate) | Send Validation Response to Zoom             | ⚠️ CONFIGURE: Insert Zoom webhook secret token                                                                   |
| Send Validation Response to Zoom | Respond To Webhook         | Replies to Zoom webhook validation        | Zoom Webhook Validator           |                                               | Responds with encrypted token to Zoom                                                                            |
| Zoom Validation Webhook (Duplicate) | Webhook                 | Duplicate Zoom validation webhook (disabled) |                               | Zoom Webhook Validator                         | DISABLED: used for testing, delete after validation                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Calendly Booking Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `/cal-uri-get`  
   - Purpose: Receive Calendly booking notifications.

2. **Add Extract Booking Data Code Node**  
   - Parse Calendly webhook payload to extract Zoom meeting ID, join URL, password, event details, attendee email, name.  
   - Throw error if Zoom meeting ID missing.  
   - Output a JSON object with booking and attendee info.

3. **Add Filter Node for Demo Events**  
   - Configure filter condition: `$json.event_type_uri` contains your specific Calendly demo event type URI.  
   - Only pass demo bookings.

4. **Create Data Table in n8n**  
   - Columns: `meeting_id` (string), `email` (string), `status` (string; e.g., 'pending')  
   - Create Data Table in n8n and note Data Table ID.

5. **Add Save Booking to Database Node**  
   - Type: Data Table (Insert)  
   - Map `meeting_id` and `email` from extracted data.  
   - Use your Data Table ID.

6. **Create Zoom Meeting Ended Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `/zoom-meeting-ended`  
   - Purpose: Receive Zoom meeting ended events.

7. **Add Filter Node for Zoom 'meeting.ended' Events**  
   - Filter where `$json.body.event` equals `'meeting.ended'`.

8. **Add Extract Meeting ID from Zoom Code Node**  
   - Extract meeting ID as integer from `$json.body.payload.object.id`.  
   - Add meeting UUID and current timestamp.

9. **Add Get Zoom Access Token HTTP Request Node**  
   - POST to Zoom OAuth endpoint `https://zoom.us/oauth/token`.  
   - Body parameters: grant_type=account_credentials, account_id, client_id, client_secret (from Zoom Server-to-Server OAuth app).  
   - Use form-urlencoded.

10. **Add Get Booking from Database Node**  
    - Type: Data Table (Get)  
    - Filter by `meeting_id` extracted from webhook.  
    - Use the same Data Table ID.

11. **Add Merge Node**  
    - Combine Zoom access token and booking data by position.  

12. **Add Get Zoom Participants HTTP Request Node**  
    - GET `https://api.zoom.us/v2/past_meetings/{{ $json.meeting_id }}/participants`  
    - Header Authorization: Bearer `{{access_token}}`.

13. **Add Check if Attendee Showed Up Code Node**  
    - Compare attendee email from booking with Zoom participants' emails (case-insensitive).  
    - Output fields: attended (boolean), participant_count, participants_list.

14. **Add Update Attendance Status Data Table Node**  
    - Update the booking record setting `status` to `true` or `false` based on attendance.  
    - Filter by `meeting_id`.

15. **Add Filter Node for No-Shows Only**  
    - Pass only if `attended == false`.

16. **Add AI Generate Follow-Up Messages Node**  
    - Use OpenAI GPT-4o via LangChain node.  
    - Prompt generates email subject, body, LinkedIn message in JSON format.  
    - Connect OpenAI credentials.

17. **Add Notify Team in Slack Node**  
    - Configure Slack OAuth2 credentials.  
    - Select channel ID.  
    - Customize message template to include booking and AI message details.

18. **(Optional) Add Send Recovery Email Node**  
    - Configure email provider credentials.  
    - Replace sender email with your sales email.  
    - Enable node to send emails automatically.

19. **(Optional) Add Update CRM Deal Node**  
    - Connect HubSpot OAuth credentials.  
    - Map deal ID and fields to update no-show status.

20. **Setup One-Time Initialization Path**  
    - Add Manual Trigger node.  
    - HTTP Request node to get Calendly user info with Personal Access Token.  
    - Code node to extract organization URI.  
    - HTTP Request node to create Calendly webhook subscription for invitee.created event.  
    - Code node to display webhook creation success or error.

21. **Add Zoom Webhook Validator Code Node**  
    - Implement HMAC SHA-256 encryption of Zoom validation plainToken using Zoom webhook secret.  
    - Respond to Zoom with encrypted token using Respond to Webhook node.

22. **Add Sticky Notes**  
    - Add sticky notes for instructions on setup, database schema, OAuth credentials, Zoom validation, AI customization, and follow-up actions.

23. **Connect all nodes as per workflow logic**  
    - Booking path: Calendly Booking Webhook → Extract Booking Data → Filter Demo Events → Save to DB  
    - Attendance path: Zoom Meeting Ended Webhook → Filter Meeting Ended → Extract Meeting ID → Get Zoom Access Token & Get Booking → Merge → Get Participants → Check Attendance → Update Status → Filter No-Shows → AI Generate Messages → Notify Slack + (optional email + CRM update)

24. **Test the workflow end-to-end**  
    - Perform test booking on Calendly with correct event type.  
    - Simulate or wait for Zoom meeting end webhook.  
    - Verify entries in Data Table and Slack notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Use your actual Calendly event type URI to filter demo bookings.                                              | Calendly event types URL: https://calendly.com/integrations/api_webhooks                                        |
| Zoom Server-to-Server OAuth app creation required for API token access.                                       | Zoom Marketplace: https://marketplace.zoom.us/                                                                 |
| Zoom webhook secret must be configured and kept confidential.                                                 | Zoom App Feature page for webhook secret                                                                         |
| Create n8n Data Table with columns: meeting_id, email, status to store booking and attendance data.           | n8n Data Table docs: https://docs.n8n.io/data-tables/                                                           |
| AI prompt can be customized to align with brand voice and message tone.                                       | OpenAI API docs: https://platform.openai.com/docs/                                                              |
| Slack OAuth credentials and channel ID required to send notifications.                                        | Slack App creation: https://api.slack.com/apps                                                                  |
| Email sending requires configuring SMTP or email provider credentials in n8n.                                 | n8n Email Send node docs: https://docs.n8n.io/nodes/n8n-nodes-base.emailSend/                                    |
| CRM update node is optional and requires appropriate OAuth and deal mapping.                                  | HubSpot API docs: https://developers.hubspot.com/docs/api/overview                                              |
| Zoom webhook validation requires responding to validation challenge within 3 seconds using HMAC SHA-256.      | Zoom webhook docs: https://marketplace.zoom.us/docs/api-reference/webhook-reference/webhook-event-reference       |
| Run the Manual Setup Trigger once before activating workflow to create Calendly webhook subscription.         | Ensures Calendly will send event notifications to your workflow                                                 |
| Sticky notes in the workflow provide detailed setup guidance and reminders for configuration steps.          |                                                                                                                 |

---

**Disclaimer:** The provided documentation is based exclusively on the n8n workflow automation JSON supplied. It complies strictly with applicable content policies and contains no illegal, offensive, or protected content. All processed data is legal and publicly available.