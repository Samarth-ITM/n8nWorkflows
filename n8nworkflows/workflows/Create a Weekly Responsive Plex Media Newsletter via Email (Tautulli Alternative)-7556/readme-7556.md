Create a Weekly Responsive Plex Media Newsletter via Email (Tautulli Alternative)

https://n8nworkflows.xyz/workflows/create-a-weekly-responsive-plex-media-newsletter-via-email--tautulli-alternative--7556


# Create a Weekly Responsive Plex Media Newsletter via Email (Tautulli Alternative)

### 1. Workflow Overview

This workflow automates the creation and distribution of a weekly Plex media newsletter, serving as an alternative to Tautulli notifications. It fetches recently added movies and TV shows from a Plex server via Tautulli API, merges this data, generates a styled HTML newsletter, and emails it to a list of recipients on a scheduled weekly basis.

Logical blocks:

- **1.1 Schedule Trigger**: Initiates the workflow weekly at a configured day and time.
- **1.2 Data Fetching from Tautulli API**: Retrieves recent movies and TV shows separately.
- **1.3 Data Merging**: Combines movie and TV show data into a single dataset.
- **1.4 HTML Newsletter Generation**: Processes combined data into a visually appealing HTML email template.
- **1.5 Recipient Preparation**: Expands the newsletter for multiple recipients.
- **1.6 Email Sending**: Sends the newsletter via SMTP.
- **1.7 Workflow Completion**: Summarizes the number of emails sent.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  This block triggers the entire workflow once a week on Friday at 08:00 AM (customizable).

- **Nodes Involved:**  
  - Note: Schedule (sticky note)  
  - Schedule Trigger (scheduleTrigger node)

- **Node Details:**  

  - **Note: Schedule**  
    - Type: Sticky Note  
    - Role: Documentation; explains the schedule timing and how to adjust it.  
    - Connections: None  
    - Sticky Note Content: "Runs weekly (Friday 08:00). Adjust day/hour to your liking."

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow automatically according to the defined schedule.  
    - Configuration: Weekly interval; triggers on Fridays (day 5 of the week), at hour 8.  
    - Key Config: `"interval":[{"field":"weeks","triggerAtDay":[5],"triggerAtHour":8}]`  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Fetch Recent Movies" node.  
    - Edge Cases: Workflow will not run if n8n instance is down at scheduled time; no retries on failure.  
    - Version: 1.2 (supports advanced interval config).

---

#### 2.2 Data Fetching from Tautulli API

- **Overview:**  
  Independently fetches the 10 most recent movies and 10 most recent TV shows added to the Plex library via Tautulli API calls.

- **Nodes Involved:**  
  - Note: Movies (sticky note)  
  - Fetch Recent Movies (httpRequest node)  
  - Note: TV Shows (sticky note)  
  - Fetch Recent TV Shows (httpRequest node)

- **Node Details:**  

  - **Note: Movies**  
    - Type: Sticky Note  
    - Role: Instruction to replace placeholders in the movie fetch URL.  
    - Content: "Replace placeholders in the URL: `YOUR_TAUTULLI_URL`, `YOUR_API_KEY`."  

  - **Fetch Recent Movies**  
    - Type: HTTP Request  
    - Role: Calls Tautulli API to get recent movies.  
    - Configuration:  
      - URL: `https://YOUR_TAUTULLI_URL/api/v2?apikey=YOUR_API_KEY&cmd=get_recently_added&count=10&media_type=movie`  
      - Method: GET (default)  
      - No additional options set.  
    - Inputs: Triggered by "Schedule Trigger"  
    - Outputs: Sent to "Combine Movie & TV Data" (input 1)  
    - Edge Cases:  
      - API unavailability or invalid API key results in HTTP error or empty data.  
      - Network timeouts or malformed response may cause failures.  
    - Requires manual replacement of placeholders.

  - **Note: TV Shows**  
    - Type: Sticky Note  
    - Role: Instruction to replace URL placeholders for TV shows fetch.  
    - Content: "Replace placeholders in the URL: `YOUR_TAUTULLI_URL`, `YOUR_API_KEY`."

  - **Fetch Recent TV Shows**  
    - Type: HTTP Request  
    - Role: Calls Tautulli API to get recent TV shows.  
    - Configuration:  
      - URL: `https://YOUR_TAUTULLI_URL/api/v2?apikey=YOUR_API_KEY&cmd=get_recently_added&count=10&media_type=show`  
      - Method: GET (default)  
    - Inputs: Triggered independently (not connected from schedule node directly)  
    - Outputs: Sent to "Combine Movie & TV Data" (input 2)  
    - Edge Cases: Same as "Fetch Recent Movies".  
    - Requires manual placeholder replacement.

---

#### 2.3 Data Merging

- **Overview:**  
  Merges the two data streams (movies and TV shows) into one dataset for unified processing downstream.

- **Nodes Involved:**  
  - Note: Merge (sticky note)  
  - Combine Movie & TV Data (merge node)

- **Node Details:**  

  - **Note: Merge**  
    - Type: Sticky Note  
    - Role: Explains the purpose of merging inputs.  
    - Content: "Just combines Movies (input 1) and TV (input 2) before building the email."

  - **Combine Movie & TV Data**  
    - Type: Merge  
    - Role: Joins two inputs (from movies and TV shows) into a single output stream.  
    - Configuration: Default (no special parameters) — merges inputs preserving original data structure.  
    - Inputs:  
      - Input 1: From "Fetch Recent Movies"  
      - Input 2: From "Fetch Recent TV Shows"  
    - Outputs: Connects to "Generate HTML Newsletter"  
    - Edge Cases:  
      - If either input is empty, output will still proceed with available data.  
      - If both inputs fail or produce no data, downstream will handle empty datasets.

---

#### 2.4 HTML Newsletter Generation

- **Overview:**  
  Converts combined movie and TV show data into a styled HTML email newsletter with images, descriptions, and Plex links.

- **Nodes Involved:**  
  - Note: HTML (sticky note)  
  - Generate HTML Newsletter (code node)

- **Node Details:**  

  - **Note: HTML**  
    - Type: Sticky Note  
    - Role: Instructions to replace key placeholders in the code.  
    - Content:  
      ```
      Edit the code and set:
      - YOUR_TAUTULLI_URL
      - YOUR_PLEX_TOKEN
      - YOUR_PLEX_SERVER_ID
      Everything else is template-safe.
      ```

  - **Generate HTML Newsletter**  
    - Type: Code (JavaScript)  
    - Role: Processes input JSON data to generate a complete HTML email body.  
    - Key Configuration:  
      - Defines helper functions for formatting duration, pill-style labels, section headers.  
      - Reads combined "recently_added" data from both movies and TV shows.  
      - Filters items by media_type to separate movies and TV shows internally.  
      - Generates HTML rows for movies and TV shows with images, links, ratings, summaries, and metadata.  
      - Builds a full HTML document with inline CSS optimized for email clients.  
      - Uses placeholders that must be replaced with user’s Tautulli URL, Plex API token, and Plex server ID for images and links.  
    - Inputs: Receives merged data from "Combine Movie & TV Data".  
    - Outputs: JSON with one field `htmlBody` containing the full HTML string.  
    - Edge Cases:  
      - Handles missing or undefined data gracefully with fallbacks (e.g., "No description provided", placeholder images).  
      - If no new movies or TV shows, inserts a message stating no new items.  
      - Requires careful replacement of placeholders or images/links will break.  
    - Version: 2

---

#### 2.5 Recipient Preparation

- **Overview:**  
  Expands the single newsletter HTML into multiple items, each addressed to a different recipient email.

- **Nodes Involved:**  
  - Note: Recipients (sticky note)  
  - Prepare Emails for Recipients (code node)

- **Node Details:**  

  - **Note: Recipients**  
    - Type: Sticky Note  
    - Role: Instruction to replace example emails with actual recipient list.  
    - Content: "Replace the example emails below with your list."

  - **Prepare Emails for Recipients**  
    - Type: Code (JavaScript)  
    - Role: Generates one output item per recipient from a static array.  
    - Key Configuration:  
      - Defines an array `recipients` containing emails (currently example placeholders).  
      - Copies the `htmlBody` from input to each output item.  
      - Outputs an array of JSON objects each with `htmlBody` and `recipientEmail`.  
    - Inputs: Receives newsletter HTML from "Generate HTML Newsletter".  
    - Outputs: One output item per recipient sent to "Send Newsletter Emails".  
    - Edge Cases:  
      - If recipient list is empty, no emails will be sent downstream.  
      - Static recipient list must be updated manually or replaced by dynamic data source if needed.

---

#### 2.6 Email Sending

- **Overview:**  
  Sends the generated HTML newsletter emails to each recipient using SMTP credentials.

- **Nodes Involved:**  
  - Note: Send (sticky note)  
  - Send Newsletter Emails (emailSend node)

- **Node Details:**  

  - **Note: Send**  
    - Type: Sticky Note  
    - Role: Instruction to configure sender email and SMTP credentials.  
    - Content: "Set `fromEmail` and attach your SMTP credentials in this node."

  - **Send Newsletter Emails**  
    - Type: Email Send  
    - Role: Sends the newsletter email to each recipient.  
    - Configuration:  
      - Subject: "New Plex Content"  
      - To Email: Expression `={{ $json.recipientEmail }}` (dynamic per item)  
      - From Email: Static string `"Your Newsletter <newsletter@example.com>"` (replace as needed)  
      - HTML Body: Expression `={{ $json.htmlBody }}`  
      - Options: Attribution disabled (no n8n footer)  
      - Credentials: SMTP credentials attached (user must configure)  
    - Inputs: From "Prepare Emails for Recipients"  
    - Outputs: Connects to "Finish" node  
    - Edge Cases:  
      - SMTP authentication failures, rate limits, or connection errors will fail email sending.  
      - Invalid recipient email format may cause delivery failures.  
      - Email size limits depend on SMTP provider.

---

#### 2.7 Workflow Completion

- **Overview:**  
  Summarizes the workflow execution by reporting how many emails were sent successfully.

- **Nodes Involved:**  
  - Note: Finish (sticky note)  
  - Finish (code node)

- **Node Details:**  

  - **Note: Finish**  
    - Type: Sticky Note  
    - Role: Provides a small summary message after sending emails.  
    - Content: "Tiny summary of how many emails were sent."

  - **Finish**  
    - Type: Code (JavaScript)  
    - Role: Counts the number of input items (emails sent) and returns a JSON status object.  
    - Configuration:  
      - Reads all input items (each corresponds to an email sent).  
      - Outputs `{ status: 'OK', sentCount: <number>, note: 'Newsletter sent successfully' }`.  
    - Inputs: From "Send Newsletter Emails"  
    - Outputs: Final output of the workflow.  
    - Edge Cases:  
      - If no email sent due to upstream issues, `sentCount` will be zero.

---

### 3. Summary Table

| Node Name                   | Node Type          | Functional Role                  | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                     |
|-----------------------------|--------------------|--------------------------------|------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| Note: Schedule              | Sticky Note        | Documentation for schedule      |                              |                              | Runs weekly (Friday 08:00). Adjust day/hour to your liking.                                   |
| Schedule Trigger            | Schedule Trigger   | Weekly trigger to start workflow|                              | Fetch Recent Movies            |                                                                                                |
| Note: Movies               | Sticky Note        | Instruction for Movies fetch URL|                              |                              | Replace placeholders in the URL: `YOUR_TAUTULLI_URL`, `YOUR_API_KEY`.                         |
| Fetch Recent Movies         | HTTP Request       | Fetch recent movies from Tautulli API| Schedule Trigger          | Combine Movie & TV Data (input 1) |                                                                                                |
| Note: TV Shows             | Sticky Note        | Instruction for TV shows fetch URL|                              |                              | Replace placeholders in the URL: `YOUR_TAUTULLI_URL`, `YOUR_API_KEY`.                         |
| Fetch Recent TV Shows       | HTTP Request       | Fetch recent TV shows from Tautulli API|                          | Combine Movie & TV Data (input 2) |                                                                                                |
| Note: Merge                | Sticky Note        | Explains merging of movie & TV data|                            |                              | Just combines Movies (input 1) and TV (input 2) before building the email.                    |
| Combine Movie & TV Data     | Merge              | Merges movie and TV datasets   | Fetch Recent Movies, Fetch Recent TV Shows | Generate HTML Newsletter    |                                                                                                |
| Note: HTML                 | Sticky Note        | Instructions for newsletter generation code|                       |                              | Edit the code and set:\n- `YOUR_TAUTULLI_URL`\n- `YOUR_PLEX_TOKEN`\n- `YOUR_PLEX_SERVER_ID`\nEverything else is template-safe. |
| Generate HTML Newsletter    | Code               | Generates HTML email body       | Combine Movie & TV Data       | Prepare Emails for Recipients  |                                                                                                |
| Note: Recipients           | Sticky Note        | Instructions to set recipient emails|                           |                              | Replace the example emails below with your list.                                              |
| Prepare Emails for Recipients| Code              | Creates one email per recipient | Generate HTML Newsletter      | Send Newsletter Emails         |                                                                                                |
| Note: Send                 | Sticky Note        | Instructions to configure SMTP send|                             |                              | Set `fromEmail` and attach your SMTP credentials in this node.                               |
| Send Newsletter Emails      | Email Send         | Sends newsletter emails         | Prepare Emails for Recipients | Finish                        |                                                                                                |
| Note: Finish               | Sticky Note        | Explains final summary step     |                              |                              | Tiny summary of how many emails were sent.                                                    |
| Finish                     | Code               | Reports number of emails sent   | Send Newsletter Emails        |                               |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Node Type: Schedule Trigger  
   - Parameters:  
     - Set Interval to weekly  
     - Trigger Day: Friday (day 5)  
     - Trigger Hour: 8 (08:00 AM)  
   - Connect no inputs; this is a trigger node.

2. **Add HTTP Request Node to Fetch Recent Movies**
   - Node Type: HTTP Request  
   - Parameters:  
     - URL: `https://YOUR_TAUTULLI_URL/api/v2?apikey=YOUR_API_KEY&cmd=get_recently_added&count=10&media_type=movie`  
     - Method: GET (default)  
     - Replace `YOUR_TAUTULLI_URL` and `YOUR_API_KEY` with your actual Tautulli URL and API key.  
   - Connect input from Schedule Trigger.

3. **Add HTTP Request Node to Fetch Recent TV Shows**
   - Node Type: HTTP Request  
   - Parameters:  
     - URL: `https://YOUR_TAUTULLI_URL/api/v2?apikey=YOUR_API_KEY&cmd=get_recently_added&count=10&media_type=show`  
     - Method: GET (default)  
     - Replace placeholders as above.  
   - Independent trigger is acceptable, but to synchronize, can add a dummy or manual trigger if needed.

4. **Add Merge Node to Combine Movie & TV Data**
   - Node Type: Merge  
   - Parameters: Default merge (no special config needed)  
   - Connect input 1 from "Fetch Recent Movies"  
   - Connect input 2 from "Fetch Recent TV Shows"

5. **Add Code Node to Generate HTML Newsletter**
   - Node Type: Code  
   - Parameters: Paste the provided JavaScript code for newsletter generation.  
   - Edit placeholders inside the code:  
     - `tautulliUrl` → Your Tautulli URL  
     - `plexApiToken` → Your Plex API Token  
     - `plexServerId` → Your Plex Server ID  
   - Connect input from Merge node.

6. **Add Code Node to Prepare Emails for Recipients**
   - Node Type: Code  
   - Parameters:  
     - Define `recipients` array with your actual recipient emails instead of example placeholders.  
     - Use the code to map the single newsletter HTML to multiple recipients.  
   - Connect input from HTML generation node.

7. **Add Email Send Node to Dispatch Emails**
   - Node Type: Email Send  
   - Parameters:  
     - Subject: "New Plex Content" (or your choice)  
     - To Email: Expression `{{$json["recipientEmail"]}}`  
     - From Email: Set your sender address and display name (e.g., "Your Newsletter <newsletter@example.com>")  
     - HTML: Expression `{{$json["htmlBody"]}}`  
     - Disable attribution/footer if desired.  
   - Credentials: Attach and configure your SMTP credentials for email sending.  
   - Connect input from recipient preparation node.

8. **Add Code Node for Workflow Completion Summary**
   - Node Type: Code  
   - Parameters: Use the provided JS code to count output items and return status JSON.  
   - Connect input from Email Send node.

9. **Optional: Add Sticky Notes for Documentation**
   - Add sticky notes near each block explaining their purpose and instructions, particularly for placeholder replacements and scheduling.

10. **Activate Workflow and Test**
    - Verify all placeholders are correctly replaced.  
    - Ensure SMTP credentials are valid and tested.  
    - Run the workflow manually or wait for scheduled trigger to validate end-to-end execution.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow is designed as a Tautulli alternative to provide weekly Plex newsletters automatically by email.   | Workflow Purpose                                                                                    |
| Replace all placeholder values (`YOUR_TAUTULLI_URL`, `YOUR_API_KEY`, `YOUR_PLEX_TOKEN`, `YOUR_PLEX_SERVER_ID`)    | Critical for API access and image/link generation                                                  |
| Use your own SMTP credentials for email sending; common providers include Gmail, Outlook, SendGrid, etc.          | See n8n documentation for SMTP node credential setup                                               |
| The HTML newsletter is responsive and styled for email clients with inline CSS and fallback images.              | Email client compatibility and styling best practices                                             |
| The workflow requires n8n version supporting scheduleTrigger 1.2 and code node version 2 or higher for best compatibility | Version requirements                                                                                |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.