Track SDK Documentation Drift with GitHub, Notion, Google Sheets, and Slack

https://n8nworkflows.xyz/workflows/track-sdk-documentation-drift-with-github--notion--google-sheets--and-slack-10337


# Track SDK Documentation Drift with GitHub, Notion, Google Sheets, and Slack

### 1. Workflow Overview

This workflow automates tracking of SDK releases and documentation freshness by integrating GitHub, Notion, Google Sheets, and Slack. Its primary purpose is to detect when the SDK documentation (specifically FAQs stored in Notion) lags behind the latest SDK release versions tracked on GitHub, and to alert the team through Slack if the documentation is overdue by more than 30 days.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Trigger on GitHub repository events related to releases.
- **1.2 Data Retrieval:** Fetch all SDK releases from GitHub and retrieve FAQ documentation pages from Notion.
- **1.3 Data Transformation and Logging:** Extract and transform release data, log releases to Google Sheets, and merge release and FAQ data.
- **1.4 Drift Calculation:** Compute the number of days documentation is behind the latest SDK release.
- **1.5 Result Logging and Alerting:** Update Google Sheets with drift status, filter overdue items, and send Slack notifications for overdue documentation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens to GitHub repository events to detect any changes, such as new releases or tags, which trigger the workflow.
- **Nodes Involved:**  
  - GitHub Trigger

- **Node Details:**

  - **GitHub Trigger**  
    - *Type:* Trigger node listening to GitHub repository events  
    - *Configuration:*  
      - OAuth2 authentication for GitHub  
      - Monitors "repository" event type, specifically for releases, tags, and repository updates  
      - Owner set via URL expression to organization GitHub URL  
      - Repository selected from list (placeholder "YOUR_REPOSITORY_NAME")  
    - *Inputs:* None (trigger)  
    - *Outputs:* Emits event data when repository events occur  
    - *Edge cases:*  
      - Authentication errors if OAuth2 token is invalid or expired  
      - Missing or incorrect repository name causing no triggers  
      - Rate limits on GitHub API calls

#### 2.2 Data Retrieval

- **Overview:** Fetches full release history from GitHub and all FAQ pages from Notion to prepare for drift analysis.
- **Nodes Involved:**  
  - GitHub Fetch Releases  
  - Notion Fetch FAQ Data

- **Node Details:**

  - **GitHub Fetch Releases**  
    - *Type:* GitHub API node  
    - *Configuration:*  
      - Uses OAuth2 GitHub credentials  
      - Retrieves all releases for the repository triggered  
      - Returns full release history (returnAll=true)  
      - Owner URL dynamically taken from trigger event data  
    - *Inputs:* Trigger data from GitHub Trigger  
    - *Outputs:* JSON array of release metadata (tags, dates, URLs)  
    - *Edge cases:*  
      - API rate limits  
      - Repository access permissions  
      - Empty or no releases available

  - **Notion Fetch FAQ Data**  
    - *Type:* Notion API node  
    - *Configuration:*  
      - Uses Notion API credentials  
      - Retrieves all pages from the specified FAQ database  
      - Return all pages for complete FAQ dataset  
      - Database ID must be replaced with user’s own FAQ database  
    - *Inputs:* Triggered after Google Sheets logging (in parallel merge)  
    - *Outputs:* JSON array of FAQ pages with metadata including last edited dates  
    - *Edge cases:*  
      - API permission errors  
      - Empty or missing FAQ pages  
      - Rate limits on Notion API

#### 2.3 Data Transformation and Logging

- **Overview:** Processes raw release data into structured fields, logs SDK releases into Google Sheets, and merges release and FAQ data streams for comparison.
- **Nodes Involved:**  
  - Edit Fields  
  - Google Sheets Log Release Data  
  - Merge

- **Node Details:**

  - **Edit Fields**  
    - *Type:* Set node (data transformation)  
    - *Configuration:*  
      - Extracts and assigns fields: SDK Name, Release Tag, Release Title, Release Date, Release URL, Days Since Release (calculated as days from published_at to now)  
      - Adds placeholders for FAQ update date, days since FAQ update, and overdue status initialized as "Pending"  
      - SDK Name dynamically taken from GitHub Trigger repository name  
    - *Inputs:* Release data from GitHub Fetch Releases  
    - *Outputs:* Structured release metadata JSON  
    - *Edge cases:*  
      - Expression failures if date fields missing or malformed  
      - Empty release data causing no assignments

  - **Google Sheets Log Release Data**  
    - *Type:* Google Sheets node  
    - *Configuration:*  
      - Appends release data rows into specified sheet and tab  
      - Uses OAuth2 Google Sheets credentials  
      - Sheet ID and GID placeholders to be replaced by user  
      - Auto-maps input fields to sheet columns  
    - *Inputs:* Output from Edit Fields  
    - *Outputs:* Confirmation of append operation  
    - *Edge cases:*  
      - Authentication or permission errors  
      - Invalid Sheet ID or GID causing operation failure  
      - API rate limits

  - **Merge**  
    - *Type:* Merge node  
    - *Configuration:*  
      - Merges two input streams by default (no special join condition)  
      - Inputs:  
        1. Release data after logging to Google Sheets  
        2. FAQ data from Notion  
      - Produces combined dataset for downstream drift calculation  
    - *Inputs:* From Google Sheets Log Release Data and Notion Fetch FAQ Data  
    - *Outputs:* Combined data stream with both release and FAQ info  
    - *Edge cases:*  
      - Data mismatch if one input is empty  
      - Potential need for join logic if data alignment required (currently raw merge)

#### 2.4 Drift Calculation

- **Overview:** Compares release dates with the last FAQ update date to compute how many days documentation lags behind the SDK release.
- **Nodes Involved:**  
  - Function Compute Drift

- **Node Details:**

  - **Function Compute Drift**  
    - *Type:* Code node (JavaScript)  
    - *Configuration:*  
      - Receives merged data of SDK releases and FAQ updates  
      - For each SDK release:  
        - Parses release date, last FAQ update date (if available)  
        - Calculates days since release and days since FAQ update  
        - Marks documentation as "OVERDUE" if FAQ update is more than 30 days behind release  
      - Outputs an array of objects with calculated drift metrics and overdue status  
    - *Inputs:* Merged data stream  
    - *Outputs:* Array of drift calculation results including overdue status  
    - *Edge cases:*  
      - Missing or invalid dates causing calculation errors  
      - FAQ update date missing defaults to release date (assumes no update)  
      - Timezone considerations for date calculations  
      - Large datasets impacting performance

#### 2.5 Result Logging and Alerting

- **Overview:** Appends drift status results to Google Sheets, filters overdue SDKs, and sends Slack alerts for overdue documentation.
- **Nodes Involved:**  
  - Google Sheets Update Drift Status  
  - Filter Overdue SDKs  
  - Slack Post Alerts

- **Node Details:**

  - **Google Sheets Update Drift Status**  
    - *Type:* Google Sheets node  
    - *Configuration:*  
      - Appends new rows containing drift analysis results to the same Google Sheet  
      - Uses OAuth2 Google Sheets credentials  
      - Sheet ID and GID placeholders must be replaced  
      - Defines exact column mappings for drift fields (SDK Name, Release Tag, Overdue Status, etc.)  
    - *Inputs:* Drift calculation output from Function Compute Drift  
    - *Outputs:* Confirmation of append operation  
    - *Edge cases:*  
      - Authentication or API errors  
      - Invalid sheet or tab reference  
      - Rate limits

  - **Filter Overdue SDKs**  
    - *Type:* If node (filter)  
    - *Configuration:*  
      - Checks if "Overdue Status" equals "OVERDUE"  
      - Filters only overdue SDK documentation entries for alerting  
    - *Inputs:* Output from Google Sheets Update Drift Status  
    - *Outputs:* Passes overdue entries to next node, discards others  
    - *Edge cases:*  
      - Case sensitivity in string comparison  
      - Missing or empty status field

  - **Slack Post Alerts**  
    - *Type:* Slack node  
    - *Configuration:*  
      - Sends message to specified Slack channel  
      - Message includes SDK name, days since last FAQ update, latest release date, and release URL  
      - Uses Slack API credentials  
      - Slack Channel ID placeholder must be replaced  
    - *Inputs:* Filtered overdue SDK entries  
    - *Outputs:* Slack message sending confirmation  
    - *Edge cases:*  
      - Slack API token invalid or lacking permissions  
      - Invalid channel ID  
      - Message formatting issues

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                         | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                                                         |
|-------------------------------|-----------------------|--------------------------------------|-------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| GitHub Trigger                | githubTrigger         | Trigger workflow on repo events      | None                          | GitHub Fetch Releases              | 📌 GitHub Trigger Setup: Monitors repository events, releases, tags. Replace with your own repo.                                     |
| GitHub Fetch Releases         | github                | Fetch all releases from GitHub       | GitHub Trigger                | Edit Fields                       | 📥 Fetch All Releases: Retrieves full release history including tag, name, date, URL.                                                |
| Edit Fields                  | set                   | Transform release data fields        | GitHub Fetch Releases          | Google Sheets Log Release Data, Notion Fetch FAQ Data | 🔄 Transform Release Data: Extracts SDK name, version, dates, calculates days since release.                                         |
| Google Sheets Log Release Data | googleSheets           | Log release info to Google Sheets    | Edit Fields                   | Merge                            | 📊 Log to Google Sheets: Append new rows with release data. Replace Sheet ID.                                                        |
| Notion Fetch FAQ Data         | notion                 | Fetch FAQ pages from Notion          | Google Sheets Log Release Data | Merge                            | 📚 Fetch FAQ Data: Retrieves all FAQ pages from your Notion database. Replace Database ID.                                           |
| Merge                        | merge                  | Combine release and FAQ data         | Google Sheets Log Release Data, Notion Fetch FAQ Data | Function Compute Drift           | 🔀 Merge Data Streams: Combines release and FAQ data for drift calculation.                                                          |
| Function Compute Drift        | code                   | Calculate documentation drift        | Merge                         | Google Sheets Update Drift Status | 🧮 Calculate Documentation Drift: Computes days documentation lags behind release, marks overdue if >30 days.                        |
| Google Sheets Update Drift Status | googleSheets           | Append drift status to Google Sheets | Function Compute Drift         | Filter Overdue SDKs               | 💾 Update Drift Status: Logs drift results. Replace Sheet ID.                                                                        |
| Filter Overdue SDKs           | if                     | Filter SDKs overdue in documentation | Google Sheets Update Drift Status | Slack Post Alerts                | 🚦 Filter Overdue Items: Passes only entries where documentation is overdue (>30 days).                                              |
| Slack Post Alerts             | slack                  | Send Slack alert for overdue docs   | Filter Overdue SDKs            | None                            | 🔔 Slack Alert: Notifies team about overdue documentation with SDK name, days since FAQ update, latest release date, and URL.        |
| Sticky Note (multiple)       | stickyNote             | Documentation and instructions       | None                         | None                             | Multiple sticky notes provide detailed purpose and setup instructions for each stage of the workflow.                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create GitHub Trigger Node**  
   - Type: GitHub Trigger  
   - Credentials: GitHub OAuth2 with appropriate scopes  
   - Configure to trigger on "repository" events for your target organization and SDK repository  
   - Ensure webhook is registered and active  

2. **Create GitHub Fetch Releases Node**  
   - Type: GitHub  
   - Credentials: Same GitHub OAuth2  
   - Operation: Get All Releases  
   - Owner: Use expression to dynamically get owner URL from GitHub Trigger event (`{{$json.body.repository.owner.html_url}}`)  
   - Repository: Set your SDK repository name  
   - Connect input from GitHub Trigger output  

3. **Create Edit Fields Node (Set)**  
   - Type: Set  
   - Add fields:  
     - SDK Name: Extract from GitHub Trigger repository name  
     - Release Tag: From fetched release tag_name  
     - Release Title: release name  
     - Release Date: release published_at  
     - Release URL: release html_url  
     - Days Since Release: Calculate difference between now and published_at in days  
     - Last FAQ Update(Notion), Days Since FAQ Update: Initialize as empty strings  
     - Overdue Status: Initialize as "Pending"  
   - Connect input from GitHub Fetch Releases output  

4. **Create Google Sheets Log Release Data Node**  
   - Type: Google Sheets  
   - Credentials: Google Sheets OAuth2  
   - Operation: Append  
   - Document ID: Enter your Google Sheet ID  
   - Sheet Name (GID): Enter your sheet tab ID  
   - Mapping Mode: Auto map input data  
   - Connect input from Edit Fields output  

5. **Create Notion Fetch FAQ Data Node**  
   - Type: Notion  
   - Credentials: Notion API credentials with database read access  
   - Resource: Database Page  
   - Operation: Get All  
   - Database ID: Enter your Notion FAQ database ID  
   - Connect input from Google Sheets Log Release Data output  

6. **Create Merge Node**  
   - Type: Merge  
   - Connect two inputs:  
     - Input 1: Google Sheets Log Release Data output  
     - Input 2: Notion Fetch FAQ Data output  
   - Leave default merge mode (combine inputs)  

7. **Create Function Compute Drift Node**  
   - Type: Code (JavaScript)  
   - Paste code that:  
     - Iterates merged data  
     - Parses release and FAQ update dates  
     - Calculates days since release and since FAQ update  
     - Flags overdue if FAQ update older than 30 days  
     - Outputs array of objects with drift metrics and overdue status  
   - Connect input from Merge output  

8. **Create Google Sheets Update Drift Status Node**  
   - Type: Google Sheets  
   - Credentials: Google Sheets OAuth2  
   - Operation: Append  
   - Document ID and Sheet Name: Same as previous Google Sheets node  
   - Mapping Mode: Define below with exact field mappings for drift data fields  
   - Connect input from Function Compute Drift output  

9. **Create Filter Overdue SDKs Node**  
   - Type: If  
   - Condition: Check if field "Overdue Status" equals "OVERDUE" (case sensitive)  
   - Connect input from Google Sheets Update Drift Status output  

10. **Create Slack Post Alerts Node**  
    - Type: Slack  
    - Credentials: Slack API token with chat:write scope  
    - Channel ID: Your Slack channel for alerts  
    - Message: Template with SDK name, days since FAQ update, latest release date, and release URL  
    - Connect input from Filter Overdue SDKs “true” output  

11. **Activate the workflow** and test by pushing a new release or triggering a repository event on GitHub.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Replace placeholders for Google Sheet ID, Notion Database ID, Slack Channel ID, and GitHub repo | Critical for correct operation; user must configure these values specific to their environment              |
| Workflow designed for SDK release tracking and documentation freshness monitoring              | Useful for teams maintaining SDKs with linked FAQs in Notion                                                |
| Slack message format includes direct call to action for documentation updates                 | Helps ensure timely updates by alerting responsible team members                                            |
| OAuth2 credentials required for GitHub, Google Sheets, Notion, and Slack APIs                 | Ensure tokens have appropriate scopes and are kept secure                                                   |
| The drift threshold of 30 days is configurable in the Function Compute Drift node code         | Adjust as needed to fit organizational SLA or documentation update policies                                 |

---

*Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.*