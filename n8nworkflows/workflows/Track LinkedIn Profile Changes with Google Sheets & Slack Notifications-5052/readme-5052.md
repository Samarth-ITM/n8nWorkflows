Track LinkedIn Profile Changes with Google Sheets & Slack Notifications

https://n8nworkflows.xyz/workflows/track-linkedin-profile-changes-with-google-sheets---slack-notifications-5052


# Track LinkedIn Profile Changes with Google Sheets & Slack Notifications

### 1. Workflow Overview

This workflow automates the tracking of LinkedIn profile changes for a list of profiles stored in a Google Sheets document. It scrapes LinkedIn profile data via an external API, compares it with previously stored data, and sends Slack notifications for any detected changes. The workflow updates the Google Sheet accordingly, ensuring the stored profile data remains current.

**Target Use Cases:**  
- Monitoring professional profile updates for recruitment or sales intelligence teams.  
- Automating alerts to Slack channels when key profile information changes.  
- Maintaining an up-to-date centralized Google Sheet tracker of LinkedIn profiles.

**Logical Blocks:**

- **1.1 Trigger & Profile List Loading:** Scheduled daily trigger fetches the list of LinkedIn profiles from Google Sheets.
- **1.2 New Profile Detection and Scraping:** Identifies profiles not yet scraped, scrapes their data and posts, and updates the sheet.
- **1.3 Existing Profile Scraping:** Scrapes current data and posts for existing profiles.
- **1.4 Change Detection:** Compares newly scraped data with existing sheet data to detect changes across multiple profile fields.
- **1.5 Alerting & Updating:** For each changed field, sends a Slack alert and updates the Google Sheet accordingly.
- **1.6 No Change Handling:** Handles cases where no changes are detected.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Profile List Loading

- **Overview:**  
  Initiates the workflow daily at 8 AM and loads the full list of LinkedIn profiles from a Google Sheet for processing.

- **Nodes Involved:**  
  - Daily Trigger  
  - Read Profiles List  
  - Filter New Profiles  
  - Scrape New Profiles Posts  
  - Wait

- **Node Details:**

  - **Daily Trigger**  
    - Type: Schedule Trigger  
    - Configured to run daily at 08:00.  
    - Starts the workflow.

  - **Read Profiles List**  
    - Type: Google Sheets  
    - Reads "Profiles" sheet (gid=0) from a specific Google Sheet document containing LinkedIn profile URLs and metadata.  
    - Uses OAuth2 credentials for Google Sheets.  
    - Always outputs data even if empty.  
    - Potential failures: Google Sheets API authentication, quota limits.

  - **Filter New Profiles**  
    - Type: Filter  
    - Filters profiles where the "ID" field is empty (meaning new profiles without scraped data).  
    - Outputs two branches: new profiles and existing profiles.

  - **Scrape New Profiles Posts**  
    - Type: HTTP Request  
    - Calls an external API (Ghost Genius) to scrape LinkedIn posts for new profiles.  
    - Sends LinkedIn URL and page=1 as query parameters.  
    - Uses HTTP header authentication with Ghost Genius credentials.  
    - Batch settings: 1 request every 2 seconds to respect rate limits.  
    - On error: continues regular output.  
    - Potential failures: API key issues, rate limiting, invalid URLs.

  - **Wait**  
    - Type: Wait  
    - Waits 2 seconds before proceeding to next node to avoid API rate limits or concurrency issues.

---

#### 2.2 New Profile Scraping and Sheet Update

- **Overview:**  
  Scrapes detailed profile data for new profiles and updates the Google Sheet with this data, including posts.

- **Nodes Involved:**  
  - Scrape New Profiles  
  - If No New Profile  
  - Update New Profiles  
  - Read Existing Data

- **Node Details:**

  - **Scrape New Profiles**  
    - Type: HTTP Request  
    - Calls Ghost Genius API to scrape profile data for new profiles.  
    - Query parameter: LinkedIn URL from filtered new profiles.  
    - Uses HTTP header authentication with Ghost Genius credentials.  
    - Batch: 1 request every 2.5 seconds.  
    - On error: continues regular output.  
    - Potential issues: API errors, invalid URLs, network failures.

  - **If No New Profile**  
    - Type: If  
    - Checks if an error exists in the scraped profile data (indicating no new profile or error in scraping).  
    - Routes to read existing data or update new profiles accordingly.

  - **Update New Profiles**  
    - Type: Google Sheets  
    - Updates the sheet with new profile data, mapping fields such as ID, Hiring status, Summary, Tagline, Last name, LinkedIn URL, First name, Latest Post, Open to work, Latest experience.  
    - Uses LinkedIn URL to match rows.  
    - Requires Google Sheets OAuth2 credentials.  
    - On error: continues regular output.  
    - Potential issues: Sheet access, data mapping errors.

  - **Read Existing Data**  
    - Type: Google Sheets  
    - Reads existing profiles data from the same sheet to be used later for change detection.  
    - Executes once per workflow run.

---

#### 2.3 Existing Profile Scraping

- **Overview:**  
  Scrapes current profile data and posts for profiles already present in the Google Sheet.

- **Nodes Involved:**  
  - Scrape Current Posts  
  - Scrape Current Data  
  - Detect Changes

- **Node Details:**

  - **Scrape Current Posts**  
    - Type: HTTP Request  
    - Calls Ghost Genius API to scrape latest posts for existing profiles.  
    - Query params: LinkedIn URL and page=1.  
    - Uses HTTP header authentication.  
    - Batch: 1 request every 2 seconds.  
    - Always outputs data.  
    - Potential failures: API issues, rate limits.

  - **Scrape Current Data**  
    - Type: HTTP Request  
    - Calls Ghost Genius API to scrape current profile data for comparison.  
    - Query: LinkedIn URLs from existing data.  
    - Batch: 1 request every 2.5 seconds.  
    - Always outputs data.  
    - Potential failures: similar to new profile scraping.

  - **Detect Changes**  
    - Type: Switch  
    - Compares fields of scraped current data with existing sheet data to detect differences: Firstname, Lastname, Tagline, Summary, Latest experience, Open to work?, Hiring?, Latest Post.  
    - Uses loose type validation and string inequality checks.  
    - Outputs multiple branches, one per changed field detected.  
    - Potential issues: Expression evaluation errors if data missing or malformed.

---

#### 2.4 Alerting & Updating Changes

- **Overview:**  
  For each detected change, sends a Slack notification alerting the change and updates the Google Sheet with new data.

- **Nodes Involved:**  
  - Alert First Name, Update First Name  
  - Alert Last Name, Update Last Name  
  - Alert Tagline, Update Tagline  
  - Alert Summary, Update Summary  
  - Alert Experience, Update Experience  
  - Alert Open to Work, Update Open to Work  
  - Alert Hiring Status, Update Hiring Status  
  - Alert Post, Update Post  
  - No Change

- **Node Details:**  
  Each alert node:  
  - Type: Slack  
  - Sends a formatted text message to a specific Slack channel (ID: C08RP1DL69M).  
  - Uses OAuth2 Slack credentials.  
  - Messages include old and new values, LinkedIn URL, and row number for context.  
  - Does not include a link back to the workflow.

  Each update node:  
  - Type: Google Sheets  
  - Updates only the changed field in the sheet row matching by ID.  
  - Uses OAuth2 Google Sheets credentials.  
  - Maps appropriate columns for the field updated.  
  - Potential failures: Sheet API errors, concurrency conflicts.

  - **No Change**  
    - Type: No Operation (NoOp)  
    - Used when no changes are detected to end that branch cleanly.

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                     | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                                          |
|------------------------|----------------------|-----------------------------------|------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Daily Trigger          | Schedule Trigger     | Starts workflow daily at 8 AM      |                              | Read Profiles List             |                                                                                                                      |
| Read Profiles List     | Google Sheets        | Loads LinkedIn profiles list       | Daily Trigger                | Filter New Profiles            |                                                                                                                      |
| Filter New Profiles    | Filter               | Filters profiles without ID        | Read Profiles List           | Scrape New Profiles Posts      |                                                                                                                      |
| Scrape New Profiles Posts | HTTP Request         | Scrapes LinkedIn posts for new profiles | Filter New Profiles          | Wait                          |                                                                                                                      |
| Wait                   | Wait                 | Delay for API rate limiting        | Scrape New Profiles Posts    | Scrape New Profiles            |                                                                                                                      |
| Scrape New Profiles    | HTTP Request         | Scrapes detailed data for new profiles | Wait                        | If No New Profile             |                                                                                                                      |
| If No New Profile      | If                   | Checks if scraping new profile failed | Scrape New Profiles          | Read Existing Data, Update New Profiles |                                                                                                                      |
| Update New Profiles    | Google Sheets        | Updates sheet with new profile data | If No New Profile           | Read Existing Data             |                                                                                                                      |
| Read Existing Data     | Google Sheets        | Reads current stored profile data  | If No New Profile, Update New Profiles | Scrape Current Posts          |                                                                                                                      |
| Scrape Current Posts   | HTTP Request         | Scrapes latest posts for existing profiles | Read Existing Data           | Scrape Current Data            |                                                                                                                      |
| Scrape Current Data    | HTTP Request         | Scrapes current profile data       | Scrape Current Posts         | Detect Changes                |                                                                                                                      |
| Detect Changes         | Switch               | Detects field changes comparing current vs stored | Scrape Current Data          | Alert First Name, Alert Last Name, Alert Tagline, Alert Summary, Alert Experience, Alert Open to Work, Alert Hiring Status, Alert Post |                                                                                                                      |
| Alert First Name       | Slack                | Alerts first name change            | Detect Changes              | Update First Name             |                                                                                                                      |
| Update First Name      | Google Sheets        | Updates first name in sheet         | Alert First Name            |                               |                                                                                                                      |
| Alert Last Name        | Slack                | Alerts last name change             | Detect Changes              | Update Last Name              |                                                                                                                      |
| Update Last Name       | Google Sheets        | Updates last name in sheet          | Alert Last Name             |                               |                                                                                                                      |
| Alert Tagline          | Slack                | Alerts tagline change               | Detect Changes              | Update Tagline                |                                                                                                                      |
| Update Tagline         | Google Sheets        | Updates tagline in sheet            | Alert Tagline               |                               |                                                                                                                      |
| Alert Summary          | Slack                | Alerts summary change               | Detect Changes              | Update Summary                |                                                                                                                      |
| Update Summary         | Google Sheets        | Updates summary in sheet            | Alert Summary               |                               |                                                                                                                      |
| Alert Experience       | Slack                | Alerts latest experience change     | Detect Changes              | Update Experience             |                                                                                                                      |
| Update Experience      | Google Sheets        | Updates latest experience in sheet | Alert Experience            |                               |                                                                                                                      |
| Alert Open to Work     | Slack                | Alerts open to work status change   | Detect Changes              | Update Open to Work           |                                                                                                                      |
| Update Open to Work    | Google Sheets        | Updates open to work status in sheet | Alert Open to Work          |                               |                                                                                                                      |
| Alert Hiring Status    | Slack                | Alerts hiring status change         | Detect Changes              | Update Hiring Status          |                                                                                                                      |
| Update Hiring Status   | Google Sheets        | Updates hiring status in sheet      | Alert Hiring Status         |                               |                                                                                                                      |
| Alert Post             | Slack                | Alerts new post detected            | Detect Changes              | Update Post                  |                                                                                                                      |
| Update Post            | Google Sheets        | Updates latest post URL in sheet    | Alert Post                  |                               |                                                                                                                      |
| No Change              | NoOp                 | Handles no change scenario          | If                         |                               |                                                                                                                      |
| Sticky Note            | Sticky Note          | Notes indicating block purpose     |                              |                               | "## Trigger & Check New Profiles "                                                                                   |
| Sticky Note1           | Sticky Note          | Notes indicating scrape current data |                              |                               | "## Scrape Current Data"                                                                                              |
| Sticky Note2           | Sticky Note          | Notes indicating track changes block |                              |                               | "## Track if there is a change, Alert and Update"                                                                    |
| Sticky Note3           | Sticky Note          | Useful resource links and descriptions |                              |                               | "## Resources\n[Video Setup](https://youtu.be/RIMS4Xu8-Eg)\n\nScraper LinkedIn (cookieless): [Ghost Genius API](https://www.ghostgenius.fr/)\n\nGoogle Sheet: [Make a copy here](https://docs.google.com/spreadsheets/d/1fGlKRGQrY0rW41DQ7GdtBh0QBR4P8Npbyr_k00DF_5Q/edit?usp=sharing)\n\nGoogle Sheet Credential Setup: [Video Tutorial](https://www.youtube.com/watch?v=pWGXlZBGu4k)\n\nSlack Credential Setup: [Video Tutorial](https://www.youtube.com/watch?v=9s4D36wDBwY&t) (Follow the instructions for OAuth2 only)" |
| Sticky Note4           | Sticky Note          | Link to video setup                |                              |                               | "# [Video Setup](https://youtu.be/RIMS4Xu8-Eg)"                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: "Daily Trigger"  
   - Set to trigger daily at 08:00.

3. **Add a Google Sheets node:**  
   - Name: "Read Profiles List"  
   - Operation: Read rows  
   - Document ID: your Google Sheet ID containing profiles  
   - Sheet Name: "Profiles" (gid=0)  
   - Credentials: OAuth2 Google Sheets  
   - Always output data.

4. **Add a Filter node:**  
   - Name: "Filter New Profiles"  
   - Condition: Check if "ID" field is empty (string empty).  
   - This separates new profiles from existing ones.

5. **Add HTTP Request node:**  
   - Name: "Scrape New Profiles Posts"  
   - URL: https://api.ghostgenius.fr/v2/profile/posts  
   - Query parameters: "url" = LinkedIn URL of profile, "page"=1  
   - Authentication: HTTP Header Auth with Ghost Genius API key  
   - Batching: batch size 1, interval 2000ms  
   - Always output data.

6. **Add Wait node:**  
   - Name: "Wait"  
   - Duration: 2 seconds  
   - Connect from "Scrape New Profiles Posts".

7. **Add HTTP Request node:**  
   - Name: "Scrape New Profiles"  
   - URL: https://api.ghostgenius.fr/v2/profile  
   - Query parameter: "url" = LinkedIn URL  
   - Authentication: HTTP Header Auth with Ghost Genius API key  
   - Batching: batch size 1, interval 2500ms  
   - Always output data.

8. **Add If node:**  
   - Name: "If No New Profile"  
   - Condition: Check if "error" property exists in JSON response (object exists).  
   - True branch: connect to "Read Existing Data"  
   - False branch: connect to "Update New Profiles".

9. **Add Google Sheets node:**  
   - Name: "Update New Profiles"  
   - Operation: Update row  
   - Match by "LinkedIn" URL  
   - Columns to update include ID, Hiring?, Summary, Tagline, Lastname, LinkedIn, Firstname, Latest Post, Open to work?, Latest experience  
   - Credentials: Google Sheets OAuth2.

10. **Add Google Sheets node:**  
    - Name: "Read Existing Data"  
    - Operation: Read rows  
    - Document and sheet same as before  
    - Execute once per run.

11. **Add HTTP Request node:**  
    - Name: "Scrape Current Posts"  
    - URL: https://api.ghostgenius.fr/v2/profile/posts  
    - Query: "url" = LinkedIn URL from existing data, "page"=1  
    - Authentication: Ghost Genius API key  
    - Batching: 1 request every 2000ms  
    - Always output data.

12. **Add HTTP Request node:**  
    - Name: "Scrape Current Data"  
    - URL: https://api.ghostgenius.fr/v2/profile  
    - Query: "url" = LinkedIn URL from existing data  
    - Authentication: Ghost Genius API key  
    - Batching: 1 request every 2500ms  
    - Always output data.

13. **Add Switch node:**  
    - Name: "Detect Changes"  
    - Add multiple rules comparing each field (Firstname, Lastname, Tagline, Summary, Latest experience, Open to work?, Hiring?, Latest Post) between scraped current data and existing sheet data using string inequality.  
    - Enable "All matching outputs" to detect multiple changes.

14. **For each field that can change, add two nodes:**  
    - **Slack node** to send alert:  
      - Message includes old value, new value, LinkedIn URL, row number.  
      - Channel: Use specific Slack channel ID.  
      - Authentication: Slack OAuth2 credentials.  
    - **Google Sheets update node** to update the corresponding field in the sheet matching by ID.

    Specifically create these pairs for:  
    - Firstname  
    - Lastname  
    - Tagline  
    - Summary  
    - Latest experience  
    - Open to work?  
    - Hiring?  
    - Latest Post

15. **Add No Operation (NoOp) node:**  
    - Name: "No Change"  
    - Connect as fallback for when no changes detected.

16. **Connect nodes appropriately:**  
    - Trigger → Read Profiles List → Filter New Profiles → Scrape New Profiles Posts → Wait → Scrape New Profiles → If No New Profile → Update New Profiles/Read Existing Data → Scrape Current Posts → Scrape Current Data → Detect Changes → (Alerts & Updates / No Change).

17. **Set all credentials:**  
    - Google Sheets OAuth2 for all Google Sheets nodes.  
    - Ghost Genius API HTTP Header Auth for all HTTP Request nodes accessing profile/post scraping.  
    - Slack OAuth2 for all Slack alert nodes.

18. **Set batching and delays carefully:**  
    - Respect API rate limits with wait nodes and batching settings as described.

19. **Test workflow end to end with a small profile list.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Video setup guide for this workflow                                                                                                                                                                                                      | https://youtu.be/RIMS4Xu8-Eg                                                                                |
| LinkedIn scraper API used (cookieless)                                                                                                                                                                                                   | https://www.ghostgenius.fr/                                                                                 |
| Google Sheet template used for profile tracking (make a copy to use)                                                                                                                                                                      | https://docs.google.com/spreadsheets/d/1fGlKRGQrY0rW41DQ7GdtBh0QBR4P8Npbyr_k00DF_5Q/edit?usp=sharing         |
| Google Sheets credential setup tutorial                                                                                                                                                                                                   | https://www.youtube.com/watch?v=pWGXlZBGu4k                                                                |
| Slack OAuth2 credential setup tutorial (follow OAuth2 instructions only)                                                                                                                                                                  | https://www.youtube.com/watch?v=9s4D36wDBwY&t                                                               |

---

This document fully describes the workflow’s structure, logic, node configurations, and integration details to allow both human developers and AI agents to understand, reproduce, or modify it effectively.