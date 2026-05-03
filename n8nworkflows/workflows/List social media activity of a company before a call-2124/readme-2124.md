List social media activity of a company before a call

https://n8nworkflows.xyz/workflows/list-social-media-activity-of-a-company-before-a-call-2124


# List social media activity of a company before a call

### 1. Workflow Overview

This workflow automates the process of gathering recent social media activity for companies involved in your scheduled meetings, preparing you thoroughly for sales or business calls. By scanning your daily Google Calendar meetings, it identifies companies associated with attendee email domains, fetches their latest LinkedIn and Twitter (X) posts via RapidAPI, and compiles this information into personalized emails sent to you or your team.

The logic is grouped into these functional blocks:

- **1.1 Scheduled Trigger & Setup**: Initiates the workflow every morning at 7 AM and sets up necessary API keys and recipient emails.
- **1.2 Calendar Meeting Extraction & Domain Parsing**: Retrieves today's meetings from Google Calendar and extracts attendee email domains for company identification.
- **1.3 Company Enrichment & Social Media Fetching**: Uses Clearbit to enrich company data by domain, then conditionally fetches recent LinkedIn posts and Twitter tweets via RapidAPI.
- **1.4 Data Formatting & Merging**: Formats social media posts into HTML snippets, merges LinkedIn and Twitter data per company, and prepares personalized email content.
- **1.5 Email Delivery**: Sends out the compiled social media activity emails via Gmail to configured recipients.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Setup

**Overview:**  
This block triggers the workflow daily at 7 AM, initializing the API keys and email recipients used throughout the workflow.

**Nodes Involved:**  
- Every morning @ 7  
- Setup  
- Sticky Note (instructions)

**Node Details:**  
- **Every morning @ 7**  
  - Type: Schedule Trigger  
  - Config: Triggers once daily at 7:00 AM local time.  
  - Input: None (time-based trigger)  
  - Output: Triggers “Setup” node  
  - Edge cases: Ensure server timezone matches expected schedule; possible trigger failure if n8n instance is down at trigger time.

- **Setup**  
  - Type: Set  
  - Config: Defines three fields for credentials and notification: `linkedInAPIKey`, `twitterAPIKey`, `emails` (recipient list).  
  - These values must be manually configured with valid API keys and email addresses.  
  - Inputs: Trigger from schedule  
  - Outputs: Feeds into “Get meetings for today”  
  - Edge cases: Missing or invalid API keys/emails will cause downstream HTTP requests or email sends to fail.  
  - Sticky Note attached explains how to register on RapidAPI, subscribe to required APIs, and where to set keys and emails.

- **Sticky Note**  
  - Provides setup instructions with links to RapidAPI and required APIs.  
  - Ensures users configure keys and emails before running workflow.

---

#### 2.2 Calendar Meeting Extraction & Domain Parsing

**Overview:**  
Fetches all meetings scheduled for the current day from Google Calendar, extracts attendee email domains to identify companies, and prepares data for enrichment.

**Nodes Involved:**  
- Get meetings for today  
- Get attendees email domains  
- Split Out  
- Keep only ones with the domain

**Node Details:**  
- **Get meetings for today**  
  - Type: Google Calendar  
  - Config: Retrieves all events between start and end of current day on configured Google account.  
  - Key Parameters: `timeMin` = start of day, `timeMax` = end of day, `singleEvents`=true (expands recurring events).  
  - Input: Setup node  
  - Output: List of meetings with attendees  
  - Edge cases: Calendar API quota limits; no meetings today results in empty data.

- **Get attendees email domains**  
  - Type: Set  
  - Config: Extracts domains from attendee emails (excluding organizers) and lists attendee emails.  
  - Expressions:  
    - `domain` = array of attendee email domains from all non-organizer attendees  
    - `attendeeEmails` = array of all attendee emails  
  - Input: Meetings data  
  - Output: Passes enriched meeting data  
  - Edge cases: Meetings without attendees or missing emails could cause empty domain arrays.

- **Split Out**  
  - Type: Split Out  
  - Config: Splits the array of domains into individual items for processing one domain at a time. Also passes attendeeEmails and meeting start time alongside.  
  - Input: Domains array from previous node  
  - Output: Multiple items, each with one domain and meeting info  
  - Edge cases: Empty domain list results in no output items.

- **Keep only ones with the domain**  
  - Type: Filter  
  - Config: Filters out items where the `domain` field does not exist or is empty, ensuring only valid domains proceed.  
  - Input: Split domains  
  - Output: Valid domains only  
  - Edge cases: Domains missing or null filtered out; if all filtered, downstream nodes receive no input.

---

#### 2.3 Company Enrichment & Social Media Fetching

**Overview:**  
Enriches each domain to get company details using Clearbit, then branches based on availability of LinkedIn and Twitter info to fetch recent posts.

**Nodes Involved:**  
- Enrich attendee company  
- Switch  
- Get recent LinkedIn posts  
- Get recetn tweets (typo in node name)  

**Node Details:**  
- **Enrich attendee company**  
  - Type: Clearbit  
  - Config: Uses Clearbit API to enrich company info by domain, retrieving LinkedIn and Twitter identifiers.  
  - Input: Filtered domains  
  - Output: Enriched company data (including `linkedin.handle` and `twitter.id`)  
  - Edge cases: Clearbit API rate limits or missing data for domains; null handles cause downstream skips.

- **Switch**  
  - Type: Switch  
  - Config: Routes based on presence of LinkedIn handle and Twitter user ID in company data.  
    - Branch “linkedin” if `linkedin.handle` exists and is not null.  
    - Branch “twitter” if `twitter.id` exists and is not null.  
  - Input: Enriched company data  
  - Output: One or both branches triggered for parallel fetching.  
  - Edge cases: Both missing leads to no further data fetch; logic assumes these fields are reliable.

- **Get recent LinkedIn posts**  
  - Type: HTTP Request  
  - Config: Calls Fresh LinkedIn Profile Data API via RapidAPI to fetch recent company posts.  
  - Query: `linkedin_url` constructed as `https://www.linkedin.com/{handle}`  
  - Headers: RapidAPI key and host for LinkedIn API.  
  - Input: Switch output “linkedin”  
  - Output: JSON data with recent LinkedIn posts  
  - Edge cases: API quota limits, invalid handle, network errors.

- **Get recetn tweets** (typo, should be “Get recent tweets”)  
  - Type: HTTP Request  
  - Config: Calls Twitter API via RapidAPI to fetch up to 10 recent tweets by user ID, excluding replies and pinned tweets.  
  - Headers: RapidAPI key and host for Twitter API.  
  - Input: Switch output “twitter”  
  - Output: JSON data with recent tweets  
  - Edge cases: API quota limits, invalid user ID, network errors.

---

#### 2.4 Data Formatting & Merging

**Overview:**  
Formats LinkedIn posts and tweets into styled HTML tables, merges the two datasets per company, and prepares final email content with meeting details.

**Nodes Involved:**  
- Format LinkedIn Posts  
- Format Tweets  
- Combine all activity for a company  
- Extract data for email  
- Prepare email template  
- Sticky Note (email example)  
- Sticky Note1 (general tip on adding more social media)

**Node Details:**  
- **Format LinkedIn Posts**  
  - Type: Code (JavaScript)  
  - Config: Runs once per LinkedIn response, generates HTML snippet listing up to 10 LinkedIn posts with clickable links and engagement stats.  
  - Input: LinkedIn API response  
  - Output: JSON object with `html_linkedin`, company name, meeting info  
  - Edge cases: Empty or missing posts handled by empty table; potential null references if API response changes.

- **Format Tweets**  
  - Type: Code (JavaScript)  
  - Config: Runs once per Twitter response, generates HTML snippet listing tweets with links and engagement stats.  
  - Input: Twitter API response  
  - Output: JSON object with `html_twitter`, company name, meeting info  
  - Edge cases: Empty tweets handled gracefully; potential mismatch if API response format changes.

- **Combine all activity for a company**  
  - Type: Merge  
  - Config: Combines LinkedIn and Twitter formatted data by matching on `name` field (company name), preferring Twitter data in case of field clashes.  
  - Input: Two branches from formatted LinkedIn and Twitter posts  
  - Output: Combined JSON object with both `html_linkedin` and `html_twitter` fields  
  - Edge cases: Missing one social media channel results in partial data; ensures no data loss in merge.

- **Extract data for email**  
  - Type: Set  
  - Config: Extracts attendee email matching the company domain, meeting start hour and minute, and passes along HTML snippets and company name for email template.  
  - Expressions: Finds attendee email ending with the company domain; parses start time from ISO string.  
  - Input: Merged social media data  
  - Output: JSON with all data needed for email  
  - Edge cases: No matching attendee email could cause missing recipient info; ISO parsing errors if date format changes.

- **Prepare email template**  
  - Type: HTML  
  - Config: Uses HTML template embedding meeting info and social media activity HTML snippets. Formats meeting time and displays company contact email.  
  - Input: Extracted data for email  
  - Output: Full HTML email content and subject line including company name  
  - Edge cases: Missing LinkedIn or Twitter HTML parts handled by optional chaining (empty string fallback).

- **Sticky Note1**  
  - Contains a tip about adding more social media accounts from Clearbit if desired, with a reminder to process them properly in switch node branches.

- **Sticky Note2**  
  - Shows example screenshot of the resulting email format users will receive, helping visualize output.

---

#### 2.5 Email Delivery

**Overview:**  
Sends the compiled social media activity email to configured recipients via Gmail OAuth2 credentials.

**Nodes Involved:**  
- Gmail

**Node Details:**  
- **Gmail**  
  - Type: Gmail  
  - Config: Sends email to addresses set in Setup node; uses OAuth2 credential configured in n8n credentials manager.  
  - Subject: “Latest social activity for: [Company Name]”  
  - Message: HTML content from email template node  
  - Input: Prepared email template  
  - Output: Email sent confirmation  
  - Edge cases: OAuth2 token expiration, API quota limits, invalid recipient emails cause delivery failures.

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                               | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                           |
|----------------------------|--------------------|-----------------------------------------------|-------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------|
| Every morning @ 7           | Schedule Trigger   | Triggers workflow daily at 7 AM                | None                          | Setup                           |                                                                                                     |
| Setup                      | Set                | Defines API keys and recipient emails          | Every morning @ 7             | Get meetings for today           | Register on RapidAPI and subscribe to required APIs; set API keys and emails here                   |
| Get meetings for today     | Google Calendar    | Fetches today’s meetings                        | Setup                        | Get attendees email domains      |                                                                                                     |
| Get attendees email domains | Set                | Extracts domains and emails from attendees     | Get meetings for today        | Split Out                      |                                                                                                     |
| Split Out                  | Split Out          | Splits domain array into single items          | Get attendees email domains   | Keep only ones with the domain   |                                                                                                     |
| Keep only ones with the domain | Filter           | Filters out invalid or missing domains          | Split Out                    | Enrich attendee company          |                                                                                                     |
| Enrich attendee company    | Clearbit           | Enriches company info by domain                 | Keep only ones with the domain | Switch                        |                                                                                                     |
| Switch                     | Switch             | Routes to LinkedIn and/or Twitter fetch branches | Enrich attendee company       | Get recent LinkedIn posts, Get recetn tweets |                                                                                                     |
| Get recent LinkedIn posts  | HTTP Request       | Fetches recent LinkedIn posts from RapidAPI    | Switch (linkedin branch)      | Format LinkedIn Posts            |                                                                                                     |
| Get recetn tweets          | HTTP Request       | Fetches recent tweets from Twitter API         | Switch (twitter branch)       | Format Tweets                   |                                                                                                     |
| Format LinkedIn Posts      | Code               | Formats LinkedIn posts into HTML                | Get recent LinkedIn posts     | Combine all activity for a company |                                                                                                     |
| Format Tweets              | Code               | Formats tweets into HTML                         | Get recetn tweets             | Combine all activity for a company |                                                                                                     |
| Combine all activity for a company | Merge        | Merges LinkedIn and Twitter HTML data           | Format LinkedIn Posts, Format Tweets | Extract data for email       |                                                                                                     |
| Extract data for email     | Set                | Extracts meeting time and attendee email for email | Combine all activity for a company | Prepare email template       |                                                                                                     |
| Prepare email template     | HTML               | Creates final HTML email body                    | Extract data for email        | Gmail                          | Shows example email format (Sticky Note2)                                                           |
| Gmail                      | Gmail              | Sends email to recipients                        | Prepare email template        | None                           |                                                                                                     |
| Sticky Note                | Sticky Note        | Setup instructions                              | None                         | None                           | Register on RapidAPI, set API keys and recipient emails                                             |
| Sticky Note1               | Sticky Note        | Tip about adding more social media sources      | None                         | None                           | Suggests adding more social media accounts from Clearbit with proper switch branch processing       |
| Sticky Note2               | Sticky Note        | Email example screenshot                         | None                         | None                           | Visual example of email output                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure: Trigger once daily at 7:00 AM local time.

2. **Create Set node named “Setup”:**  
   - Fields to set:  
     - `linkedInAPIKey` (string) – Your RapidAPI key for LinkedIn API  
     - `twitterAPIKey` (string) – Your RapidAPI key for Twitter API  
     - `emails` (string or array) – Recipient email addresses (comma-separated or array)  
   - Connect the Schedule Trigger output to this node.

3. **Create Google Calendar node named “Get meetings for today”:**  
   - Operation: Get All Events  
   - Calendar: Select your Google Calendar account  
   - Options:  
     - `timeMin`: Expression `{{$now.beginningOf('day')}}`  
     - `timeMax`: Expression `{{$now.endOf('day')}}`  
     - `singleEvents`: true  
   - Connect “Setup” output to this node.

4. **Create Set node named “Get attendees email domains”:**  
   - Fields:  
     - `domain` (array): Expression to extract domains:  
       `{{$json.attendees.filter(a => !a.organizer).map(a => a.email.split('@').pop())}}`  
     - `attendeeEmails` (array): Expression:  
       `{{$json.attendees.filter(a => !a.organizer).map(a => a.email)}}`  
   - Connect “Get meetings for today” output to this node.

5. **Create Split Out node named “Split Out”:**  
   - Field to split out: `domain`  
   - Fields to include: `attendeeEmails`, `start`  
   - Connect “Get attendees email domains” output to this node.

6. **Create Filter node named “Keep only ones with the domain”:**  
   - Condition: Field `domain` exists (string operation: exists)  
   - Connect “Split Out” output to this node.

7. **Create Clearbit node named “Enrich attendee company”:**  
   - Operation: Company enrichment by domain  
   - Domain: Expression `{{$json.domain}}`  
   - Connect “Keep only ones with the domain” output to this node.  
   - Configure Clearbit credentials in n8n before use.

8. **Create Switch node named “Switch”:**  
   - Add two rules:  
     - Rule “linkedin”: Condition `{{$json.linkedin.handle !== null}}` is true  
     - Rule “twitter”: Condition `{{$json.twitter.id !== null}}` is true  
   - Connect “Enrich attendee company” output to this node.

9. **Create HTTP Request node named “Get recent LinkedIn posts”:**  
   - URL: `https://fresh-linkedin-profile-data.p.rapidapi.com/get-company-posts`  
   - Query parameters:  
     - `linkedin_url`: Expression: `="https://www.linkedin.com/" + $json.linkedin.handle`  
     - `sort_by`: `recent`  
   - Headers:  
     - `X-RapidAPI-Key`: Expression: `{{$node["Setup"].json.linkedInAPIKey}}`  
     - `X-RapidAPI-Host`: `fresh-linkedin-profile-data.p.rapidapi.com`  
   - Connect “Switch” output “linkedin” branch to this node.

10. **Create HTTP Request node named “Get recent tweets”:**  
    - URL: `https://twitter154.p.rapidapi.com/user/tweets`  
    - Query parameters:  
      - `limit`: 10  
      - `user_id`: Expression: `{{$json.twitter.id}}`  
      - `include_replies`: false  
      - `include_pinned`: false  
    - Headers:  
      - `X-RapidAPI-Key`: Expression: `{{$node["Setup"].json.twitterAPIKey}}`  
      - `X-RapidAPI-Host`: `twitter154.p.rapidapi.com`  
    - Connect “Switch” output “twitter” branch to this node.

11. **Create Code node named “Format LinkedIn Posts”:**  
    - Mode: Run Once For Each Item  
    - JavaScript code: Format LinkedIn posts into styled HTML table with clickable links and engagement stats (use the logic from original node).  
    - Connect “Get recent LinkedIn posts” output to this node.

12. **Create Code node named “Format Tweets”:**  
    - Mode: Run Once For Each Item  
    - JavaScript code: Format tweets similarly into HTML with links and engagement metrics.  
    - Connect “Get recent tweets” output to this node.

13. **Create Merge node named “Combine all activity for a company”:**  
    - Mode: Combine, Join Mode: Keep Everything  
    - Merge by field: `name` (company name)  
    - Connect “Format LinkedIn Posts” and “Format Tweets” outputs to this node.

14. **Create Set node named “Extract data for email”:**  
    - Fields:  
      - `attendeeEmail`: Expression to find attendee email matching domain:  
        `{{$json.meeting.attendeeEmails.find(a => a.endsWith($json.meeting.domain))}}`  
      - `startHour`: Expression: `DateTime.fromISO($json.meeting.start.dateTime).hour`  
      - `startMinute`: Expression: `DateTime.fromISO($json.meeting.start.dateTime).minute`  
    - Include fields: `name`, `html_twitter`, `html_linkedin`  
    - Connect “Combine all activity for a company” output to this node.

15. **Create HTML node named “Prepare email template”:**  
    - Configure HTML with embedded variables for meeting info and social media HTML snippets (copy from original node).  
    - Connect “Extract data for email” output to this node.

16. **Create Gmail node named “Gmail”:**  
    - Configure to send email:  
      - To: Expression `{{$node["Setup"].json.emails}}`  
      - Subject: `Latest social activity for: {{$json.name}}`  
      - Message: Use HTML from previous node  
    - Connect “Prepare email template” output to this node.  
    - Configure Gmail OAuth2 credentials in n8n credentials manager before using.

17. **Add Sticky Notes:**  
    - One near “Setup” node with instructions on API registration and key setup.  
    - One near formatting nodes with tip on adding more social media sources from Clearbit.  
    - One near email delivery node with example email screenshot.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Register on [RapidAPI](https://rapidapi.com) and subscribe to these APIs: Fresh LinkedIn Profile Data and Twitter (X).    | Setup step, required for API access.                                                               |
| Email example screenshot shows the final email format with meeting info and social media posts.                           | https://i.imgur.com/7T8XIX3.png                                                                     |
| Tip: To add more social media accounts found via Clearbit, add separate branches in the Switch node and respective fetch logic. | Workflow extensibility note.                                                                        |
| Gmail OAuth2 credential must be configured in n8n for sending emails securely.                                            | Gmail node credential configuration requirement.                                                    |

---

This documentation provides a complete, detailed reference to understand, reproduce, and maintain the "List social media activity of a company before a call" workflow using n8n. It covers all nodes, logic flows, configuration details, and contextual guidance for error handling and extensibility.