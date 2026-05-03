Manage attendee registrations and send emails

https://n8nworkflows.xyz/workflows/manage-attendee-registrations-and-send-emails-730


# Manage attendee registrations and send emails

### 1. Workflow Overview

This workflow automates the management of attendee registrations for n8nConf, a no-code automation conference. It receives registration data via a Typeform form, records attendee information into a Google Sheet, creates user accounts in Mattermost (a team chat platform), adds attendees to appropriate teams and channels based on their session choices, updates Google Calendar events with attendee emails, and finally, sends personalized welcome emails confirming registration details.

The workflow can be logically divided into the following blocks:

- **1.1 Input Reception:** Captures attendee registration data from Typeform.
- **1.2 Data Storage:** Appends attendee data to a Google Sheet for record-keeping.
- **1.3 User Account Management:** Creates a Mattermost account for the attendee and invites them to the team.
- **1.4 Session Processing:** Transforms session selections into rows, fetches detailed session info from Google Sheets, and merges data.
- **1.5 Channel and Calendar Updates:** Adds the attendee to session-specific Mattermost channels and updates Google Calendar events with their attendance.
- **1.6 Communication:** Sends a personalized welcome email with login credentials and session info.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for new submissions on the Typeform registration form and triggers the workflow.
- **Nodes Involved:**  
  - Attendee Registrations (Typeform Trigger)
- **Node Details:**

| Node Name           | Details                                                                                                                                    |
|---------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Attendee Registrations | - Type: Typeform Trigger<br>- Configured to listen to form submissions for form ID "RknoIFsl".<br>- Uses Typeform Burner Account credentials.<br>- Outputs JSON containing answers keyed by question texts.<br>- Inputs: Webhook trigger only.<br>- Outputs: JSON with all questions and answers.<br>- Potential failure: Webhook connectivity issues, API rate limits, form ID changes.<br>- Version: 1 |

---

#### 2.2 Data Storage

- **Overview:** Stores the attendee’s raw registration data into a Google Sheets spreadsheet for attendance tracking.
- **Nodes Involved:**  
  - Add to Sheets (Google Sheets Append)
- **Node Details:**

| Node Name    | Details                                                                                                                                          |
|--------------|--------------------------------------------------------------------------------------------------------------------------------------------------|
| Add to Sheets | - Type: Google Sheets Node<br>- Operation: Append<br>- Sheet: "Attendees" range A:F in specified spreadsheet.<br>- Authentication: OAuth2 with Google Sheets.<br>- Inputs: Data from "Attendee Registrations".<br>- Outputs: Confirmation of append operation.<br>- Potential failure: OAuth token expiry, sheet permission errors, invalid range.<br>- Version: 1 |

---

#### 2.3 User Account Management

- **Overview:** Creates a new user account in Mattermost for each attendee, then invites them to the main team.
- **Nodes Involved:**  
  - Create Account (Mattermost User Create)  
  - Add to team (Mattermost User Invite)
- **Node Details:**

| Node Name     | Details                                                                                                                                                                                                                                                                                                       |
|---------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Create Account | - Type: Mattermost Node, User resource, 'create' operation.<br>- Email: Extracted from the attendee’s answer to "And what's your email address?"<br>- Password: Constructed as `P!_` + email with spaces removed + current hour + current date (dynamic password generation).<br>- Username: Full name with spaces removed + current hour.<br>- Additional: Stores first_name as full name.<br>- Credentials: Mattermost API.<br>- Inputs: From "Add to Sheets".<br>- Outputs: JSON with created user details, including ID and username.<br>- Edge cases: Account duplication, invalid email, API auth failure.<br>- Version: 1 |
| Add to team   | - Type: Mattermost Node, User resource, 'invite' operation.<br>- Emails: Uses email from "Attendee Registrations" node.<br>- Team ID: Fixed team identifier.<br>- Credentials: Mattermost API.<br>- Inputs: From "Create Account".<br>- Outputs: Confirmation of invitation sent.<br>- Edge cases: User already invited, invalid team ID, auth failure.<br>- Version: 1 |

---

#### 2.4 Session Processing

- **Overview:** Processes the sessions selected by the attendee, expands the array into individual rows, fetches session details from Google Sheets, and merges these details with the session selections.
- **Nodes Involved:**  
  - Array to Rows (Function)  
  - Get Session Details (Google Sheets Read)  
  - Merge Data (Merge Node)
- **Node Details:**

| Node Name         | Details                                                                                                                                                                                                                                     |
|-------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Array to Rows     | - Type: Function Node<br>- Purpose: Iterates over the "Which sessions would you like to attend?" array from the registration JSON and creates a new item for each session.<br>- Returns an array of JSON objects with a single "Session" property.<br>- Inputs: From "Add to team".<br>- Outputs: Array of session rows.<br>- Edge cases: Empty session selection, malformed input.<br>- Version: 1 |
| Get Session Details | - Type: Google Sheets Node<br>- Operation: Read from "Sessions" sheet, range A:F.<br>- Credentials: Google Sheets OAuth2.<br>- Inputs: None directly connected (runs in parallel).<br>- Outputs: Full list of session details.<br>- Edge cases: Sheet access issues, invalid range.<br>- Version: 1 |
| Merge Data        | - Type: Merge Node (mergeByKey mode)<br>- Merges session selections (from "Array to Rows") with session details (from "Get Session Details") on the "Session" property.<br>- Inputs: Two inputs, one from each previous node.<br>- Outputs: Combined session data enriched with details.<br>- Edge cases: Missing session keys, no matching sessions.<br>- Version: 1 |

---

#### 2.5 Channel and Calendar Updates

- **Overview:** Adds the attendee to Mattermost channels corresponding to their sessions and updates Google Calendar events by adding the attendee as an event attendee.
- **Nodes Involved:**  
  - Add to channels (Mattermost Channel Add User)  
  - Add to Event (Google Calendar Update)
- **Node Details:**

| Node Name       | Details                                                                                                                                                                                                                                                                                 |
|-----------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Add to channels | - Type: Mattermost Node, Channel resource, 'addUser' operation.<br>- User ID: Uses the Mattermost user ID from "Create Account".<br>- Channel ID: Extracted from merged session data ("Mattermost Channel ID").<br>- Credentials: Mattermost API.<br>- Inputs: From "Merge Data".<br>- Outputs: Confirmation of user addition.<br>- Edge cases: Invalid channel ID, user already in channel, API errors.<br>- Version: 1 |
| Add to Event    | - Type: Google Calendar Node, 'update' operation.<br>- Event ID: From merged session data ("Google Calendar Event ID").<br>- Calendar: Fixed calendar address.<br>- Adds the attendee's email as an event attendee.<br>- Credentials: Google Calendar OAuth2.<br>- Inputs: From "Add to channels".<br>- Outputs: Confirmation of event update.<br>- Edge cases: Event not found, calendar access denied, invalid email.<br>- Version: 1 |

---

#### 2.6 Communication

- **Overview:** Sends a personalized welcome email to the attendee confirming their registration and providing login credentials and session information.
- **Nodes Involved:**  
  - Welcome Email (Gmail Send)
- **Node Details:**

| Node Name    | Details                                                                                                                                                                                                                                                                                                                                                                                     |
|--------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Welcome Email | - Type: Gmail Node, message resource.<br>- To: Attendee email.<br>- Subject: "Welcome to n8nConf".<br>- Message: Personalized greeting including attendee full name, session list, Mattermost login URL, username, and password.<br>- Credentials: Gmail OAuth2.<br>- Inputs: From "Add to Event".<br>- Outputs: Confirmation of email sent.<br>- Edge cases: Email delivery failure, OAuth token expiry.<br>- Version: 1 |

---

### 3. Summary Table

| Node Name             | Node Type                 | Functional Role                              | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                 |
|-----------------------|---------------------------|----------------------------------------------|------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------|
| Attendee Registrations | Typeform Trigger          | Capture registration data                     | Webhook trigger         | Add to Sheets           |                                                                                                             |
| Add to Sheets         | Google Sheets             | Append attendee data to spreadsheet           | Attendee Registrations  | Create Account          |                                                                                                             |
| Create Account        | Mattermost                | Create user account in Mattermost             | Add to Sheets           | Add to team             |                                                                                                             |
| Add to team           | Mattermost                | Invite user to Mattermost team                 | Create Account          | Array to Rows           |                                                                                                             |
| Array to Rows         | Function                  | Expand session array into individual items    | Add to team             | Merge Data              |                                                                                                             |
| Get Session Details   | Google Sheets             | Fetch session details from spreadsheet        | (No direct input)       | Merge Data              |                                                                                                             |
| Merge Data            | Merge                     | Merge session selections with session details | Array to Rows, Get Session Details | Add to channels        |                                                                                                             |
| Add to channels       | Mattermost                | Add user to session-specific channels         | Merge Data              | Add to Event            |                                                                                                             |
| Add to Event          | Google Calendar           | Update Google Calendar events with attendee   | Add to channels         | Welcome Email           |                                                                                                             |
| Welcome Email         | Gmail                     | Send confirmation email with login info       | Add to Event            | (End)                   |                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Typeform Trigger Node:**
   - Type: Typeform Trigger
   - Configure with your Typeform account and specify form ID (e.g., "RknoIFsl").
   - Set webhook to trigger on form submission.

2. **Create Google Sheets Append Node ("Add to Sheets"):**
   - Type: Google Sheets
   - Operation: Append
   - Sheet ID: Use your Google Sheet ID.
   - Range: "Attendees!A:F"
   - Authentication: OAuth2 with Google Sheets credentials.
   - Connect input from "Attendee Registrations."

3. **Create Mattermost User Create Node ("Create Account"):**
   - Type: Mattermost
   - Resource: User
   - Operation: Create
   - Email: Expression pulling email answer from Typeform JSON.
   - Password: Construct dynamically using `P!_` + email (spaces removed) + current hour + date.
   - Username: Full name (spaces removed) + current hour.
   - Additional field: first_name set to full name.
   - Credentials: Mattermost API.
   - Connect input from "Add to Sheets."

4. **Create Mattermost User Invite Node ("Add to team"):**
   - Type: Mattermost
   - Resource: User
   - Operation: Invite
   - Emails: Use email from Typeform JSON.
   - Team ID: Fixed team identifier.
   - Credentials: Mattermost API.
   - Connect input from "Create Account."

5. **Create Function Node ("Array to Rows"):**
   - Type: Function
   - Code: Iterate over attendee's session selections array and output each session as a separate JSON item with property "Session".
   - Connect input from "Add to team."

6. **Create Google Sheets Read Node ("Get Session Details"):**
   - Type: Google Sheets
   - Operation: Read
   - Sheet ID: Same as "Add to Sheets".
   - Range: "Sessions!A:F"
   - Authentication: OAuth2 with Google Sheets credentials.
   - No direct input connection (runs in parallel).

7. **Create Merge Node ("Merge Data"):**
   - Type: Merge
   - Mode: mergeByKey
   - Property Name 1: "Session" (from "Array to Rows")
   - Property Name 2: "Session" (from "Get Session Details")
   - Connect inputs from "Array to Rows" and "Get Session Details."

8. **Create Mattermost Channel Add User Node ("Add to channels"):**
   - Type: Mattermost
   - Resource: Channel
   - Operation: Add User
   - User ID: From "Create Account" node output ("id").
   - Channel ID: From merged session data ("Mattermost Channel ID").
   - Credentials: Mattermost API.
   - Connect input from "Merge Data."

9. **Create Google Calendar Update Node ("Add to Event"):**
   - Type: Google Calendar
   - Operation: Update
   - Calendar: Fixed calendar email address.
   - Event ID: From merged session data ("Google Calendar Event ID").
   - Update attendees: Add attendee email from Typeform JSON.
   - Credentials: Google Calendar OAuth2.
   - Connect input from "Add to channels."

10. **Create Gmail Send Node ("Welcome Email"):**
    - Type: Gmail
    - Resource: Message
    - To: Attendee email from Typeform JSON.
    - Subject: "Welcome to n8nConf"
    - Message: Personalized email including full name, sessions list, Mattermost login URL, username, and password.
    - Credentials: Gmail OAuth2.
    - Connect input from "Add to Event."

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                                                            |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------|
| This workflow is the official n8nConf Companion workflow designed to automate attendee registrations and communications for the n8n conference.                                                                                                    | Workflow Screenshot: ![workflow-screenshot](https://f000.backblazeb2.com/file/n8n-website-images/b68150cbd1e64ff0bfe4aad63d91387f.png) |
| Mattermost login URL used in emails: https://mm.failedmachine.com/                                                                                                                                                                                   | Included in Welcome Email node                                                            |
| Ensure OAuth2 credentials for Google Sheets, Google Calendar, Gmail, and Mattermost API are correctly configured with appropriate scopes and permissions.                                                                                             | Credential Setup Requirement                                                              |
| Password generation logic uses current hour and date, which means passwords will be time-sensitive and unique per registration but should be communicated clearly to attendees for first login.                                                     | Security consideration                                                                    |
| To add additional sessions or update session details, edit the "Sessions" sheet within the Google Sheet linked to the workflow.                                                                                                                    | Session management via Google Sheets                                                      |