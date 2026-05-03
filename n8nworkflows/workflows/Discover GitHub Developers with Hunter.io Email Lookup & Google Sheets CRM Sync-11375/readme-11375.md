Discover GitHub Developers with Hunter.io Email Lookup & Google Sheets CRM Sync

https://n8nworkflows.xyz/workflows/discover-github-developers-with-hunter-io-email-lookup---google-sheets-crm-sync-11375


# Discover GitHub Developers with Hunter.io Email Lookup & Google Sheets CRM Sync

### 1. Workflow Overview

This workflow automates the discovery and enrichment of GitHub developers as potential leads, syncing the curated data into a Google Sheets CRM database. It is designed for recruiting, sales, or marketing teams targeting developers based on location, programming language, and follower count on GitHub.

The workflow consists of three main logical blocks:

- **1.1 Find Developers on GitHub:** Periodically triggers the process, defines search criteria, queries GitHub users, and fetches detailed user profiles.

- **1.2 Find Developers’ Emails Using Hunter.io:** Checks if email information is missing; if so, enriches developer data by querying Hunter.io for domain-based email addresses.

- **1.3 Remove Duplicates and Save to CRM Database:** Checks for existing leads in Google Sheets to avoid duplicates, formats the data, and appends new leads to the CRM sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Find Developers on GitHub

**Overview:**  
This block initiates the workflow on a scheduled weekly interval, defines developer search parameters (location, language, follower count), executes GitHub API queries to find matching users, and retrieves detailed user profiles to prepare for enrichment.

**Nodes Involved:**  
- Schedule Trigger  
- Define Developer Searches  
- Search GitHub Users  
- Parse Users  
- Get User Details  
- Extract Developer Data  

**Node Details:**

- **Schedule Trigger**  
  - Type: `scheduleTrigger`  
  - Role: Starts the workflow on a weekly interval.  
  - Configuration: Interval set to every week.  
  - Inputs: None (trigger node).  
  - Outputs: Connects to Define Developer Searches.  
  - Edge Cases: Workflow will not run if n8n instance is offline at scheduled time; no retries configured here.

- **Define Developer Searches**  
  - Type: `code`  
  - Role: Defines an array of developer search criteria objects (location, language, minFollowers).  
  - Configuration: Hardcoded criteria for New Jersey (JavaScript, 50 followers), Texas (Python, 100 followers), New York (React, 200 followers).  
  - Key Expression: Returns array of JSON objects, each representing one search criterion.  
  - Inputs: From Schedule Trigger.  
  - Outputs: Array of search objects feeding Search GitHub Users.  
  - Edge Cases: Modifications to criteria require code edits; no dynamic input.  
  - Version: Uses version 2 of code node.

- **Search GitHub Users**  
  - Type: `httpRequest`  
  - Role: Queries GitHub Search Users API with criteria from previous node.  
  - Configuration:  
    - URL: `https://api.github.com/search/users`  
    - Query Parameters: `q` constructed dynamically using location, followers, and language from input JSON; `per_page` set to 30.  
    - Headers: Accepts GitHub API v3; Authorization header uses personal GitHub token credential (`GitHub_API_Token`).  
  - Credentials: Requires GitHub personal access token with appropriate scopes.  
  - Inputs: Search parameters from Define Developer Searches node.  
  - Outputs: JSON response with users matching criteria.  
  - Edge Cases:  
    - API rate limits from GitHub can cause 403 errors.  
    - Token invalid or expired causes auth errors.  
    - Query construction failures if input malformed.  
  - Notes: Sticky note reminds to replace `YOUR_GITHUB_TOKEN` with actual token.

- **Parse Users**  
  - Type: `code`  
  - Role: Extracts and normalizes user list from GitHub API responses, preparing simplified developer objects for next step.  
  - Configuration: Iterates over all input items, extracts username, profile URL, avatar URL, and user API URL.  
  - Inputs: Receives multiple GitHub user search results (array).  
  - Outputs: One item per developer with normalized minimal data.  
  - Edge Cases: Handles empty or missing `items` gracefully by returning empty arrays.  
  - Always outputs data for consistent downstream processing.

- **Get User Details**  
  - Type: `httpRequest`  
  - Role: Fetches detailed GitHub user profile data for each parsed developer.  
  - Configuration:  
    - URL dynamically set to `userApiUrl` from Parse Users output.  
    - Headers include Accept v3 and Authorization with GitHub token.  
  - Credentials: Uses same GitHub API token credential.  
  - Inputs: One developer per item.  
  - Outputs: Full user profile JSON.  
  - Edge Cases:  
    - API rate limits or invalid token issues.  
    - Missing or malformed URLs cause request failures.

- **Extract Developer Data**  
  - Type: `code`  
  - Role: Cleans and organizes user profile data; extracts website and domain; calculates lead score and quality based on followers, repos, and contact info; formats output lead data.  
  - Configuration:  
    - Normalizes website URL and extracts domain.  
    - Scoring weighted by followers, public repos, and presence of contact info (email, website, company).  
    - Lead quality classified as Cold, Warm, or Hot based on score thresholds.  
  - Inputs: Detailed user profile JSON.  
  - Outputs: Single enriched lead JSON object with calculated fields like leadScore and leadQuality.  
  - Edge Cases: Handles malformed website URLs with try-catch; defaults scores and quality if data missing.

---

#### 1.2 Find Developers’ Emails Using Hunter.io

**Overview:**  
This block conditionally enriches developer data by looking up emails via Hunter.io if the email field is missing or marked as “Check website” and a domain is available.

**Nodes Involved:**  
- Need Email Enrichment? (IF node)  
- Hunter.io Email Lookup  
- Update With Email  

**Node Details:**

- **Need Email Enrichment?**  
  - Type: `if`  
  - Role: Checks if email enrichment via Hunter.io is needed.  
  - Configuration:  
    - Condition 1: domain must not be empty.  
    - Condition 2: email field must equal "Check website".  
    - Logical AND between conditions.  
  - Inputs: Developer data from Extract Developer Data.  
  - Outputs: Two branches:  
    - True: triggers Hunter.io lookup.  
    - False: bypasses Hunter.io and continues to duplicate check.  
  - Edge Cases:  
    - Domain field empty or malformed disables lookup.  
    - Email already present skips Hunter.io to save credits.

- **Hunter.io Email Lookup**  
  - Type: `httpRequest`  
  - Role: Queries Hunter.io domain-search API to find emails related to domain.  
  - Configuration:  
    - URL: `https://api.hunter.io/v2/domain-search`  
    - Query parameters: domain from input JSON, API key credential, limit set to 3 results.  
  - Credentials: Requires Hunter.io API key credential (`Hunter_API_Key`).  
  - Inputs: Domain string from previous step.  
  - Outputs: Hunter.io response data with emails and confidence scores.  
  - Notes: Optional step depending on Hunter.io API credits.  
  - Edge Cases:  
    - API rate limits or invalid API key cause errors.  
    - No emails found returns empty arrays.

- **Update With Email**  
  - Type: `code`  
  - Role: Merges Hunter.io email results with existing developer data; selects highest confidence email if available.  
  - Configuration:  
    - Extracts emails array from Hunter.io data.  
    - Sorts by confidence and picks top email.  
    - Updates email and adds `emailConfidence` field.  
  - Inputs: Hunter.io response plus original developer data.  
  - Outputs: Enriched lead JSON with updated email field.  
  - Edge Cases: No emails found leaves original email unchanged.

---

#### 1.3 Remove Duplicates and Save to CRM Database

**Overview:**  
This block checks the Google Sheets CRM database for existing leads to avoid duplicates, formats the new lead data for the CRM schema, and appends new leads to the database sheet.

**Nodes Involved:**  
- Check For Duplicates (Google Sheets)  
- Filter Duplicates (Code)  
- Merge  
- Format For Database (Code)  
- Add to Lead Database (Google Sheets)  

**Node Details:**

- **Check For Duplicates**  
  - Type: `googleSheets`  
  - Role: Reads all existing leads from the Google Sheets database for duplicate checking.  
  - Configuration:  
    - Document ID set to the specific Google Sheets lead database.  
    - Sheet Name: Sheet1 (gid=0).  
  - Credentials: Requires Google Sheets OAuth2 credentials.  
  - Inputs: Developer data from Update With Email or directly from Extract Developer Data if no enrichment needed.  
  - Outputs: Array of existing leads rows from sheet.  
  - Edge Cases: Access permission errors, empty sheet returns empty arrays.

- **Filter Duplicates**  
  - Type: `code`  
  - Role: Compares current lead against existing leads to detect duplicates by business name, email, or username in notes.  
  - Configuration:  
    - Uses multiple conditions to flag duplicates.  
    - Returns empty output if duplicate found to stop processing.  
  - Inputs: Current lead data and existing leads data.  
  - Outputs: Passes through only unique leads.  
  - Edge Cases: Case sensitivity or partial matches might cause false negatives.

- **Merge**  
  - Type: `merge`  
  - Role: Combines two input branches: one with filtered unique leads and one with leads that did not require email enrichment (direct from Check For Duplicates).  
  - Configuration: Mode set to "chooseBranch" to select branch 2 data when available.  
  - Inputs: Two branches from Filter Duplicates and Check For Duplicates.  
  - Outputs: Single stream for further processing.  
  - Edge Cases: Misconfiguration could cause data loss or duplication.

- **Format For Database**  
  - Type: `code`  
  - Role: Maps lead fields to CRM database schema, generates unique Lead ID, and formats all required columns for Google Sheets.  
  - Configuration:  
    - Generates Lead ID with prefix `GITHUB-` plus timestamp and random number.  
    - Sets default values for job title, phone, rating, status, contacted, etc.  
  - Inputs: Lead JSON from Merge node.  
  - Outputs: Formatted lead record matching Google Sheets columns.  
  - Edge Cases: Random number collisions extremely unlikely but possible; missing fields default to placeholders.

- **Add to Lead Database**  
  - Type: `googleSheets`  
  - Role: Appends new lead record to Google Sheets CRM database.  
  - Configuration:  
    - Document ID and sheet name set as in Check For Duplicates.  
    - Append operation with mapping defined to match CRM columns.  
  - Credentials: Uses same Google Sheets OAuth2 credentials.  
  - Inputs: Formatted lead record.  
  - Outputs: None (end node).  
  - Edge Cases: API quota limits, permission errors, or connection failures.

---

### 3. Summary Table

| Node Name               | Node Type          | Functional Role                             | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                 |
|-------------------------|--------------------|---------------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | scheduleTrigger    | Starts workflow periodically (weekly)       | -                           | Define Developer Searches    |                                                                                                             |
| Define Developer Searches| code               | Defines search criteria for developers       | Schedule Trigger             | Search GitHub Users          |                                                                                                             |
| Search GitHub Users     | httpRequest        | Queries GitHub users API with criteria       | Define Developer Searches    | Parse Users                 | Replace YOUR_GITHUB_TOKEN with your personal access token                                                  |
| Parse Users             | code               | Parses GitHub user search results             | Search GitHub Users          | Get User Details             |                                                                                                             |
| Get User Details        | httpRequest        | Fetches detailed user profiles from GitHub   | Parse Users                 | Extract Developer Data       | Replace YOUR_GITHUB_TOKEN                                                                                   |
| Extract Developer Data  | code               | Cleans and scores developer data              | Get User Details             | Need Email Enrichment?       |                                                                                                             |
| Need Email Enrichment?  | if                 | Checks if email enrichment is needed          | Extract Developer Data       | Hunter.io Email Lookup, Filter Duplicates |                                                                                                             |
| Hunter.io Email Lookup  | httpRequest        | Enriches email data via Hunter.io             | Need Email Enrichment?       | Update With Email            | Optional - uses Hunter.io credits                                                                            |
| Update With Email       | code               | Merges Hunter.io emails with existing data    | Hunter.io Email Lookup       | Check For Duplicates, Merge  |                                                                                                             |
| Check For Duplicates    | googleSheets       | Reads CRM sheet to check for duplicate leads  | Update With Email, Need Email Enrichment? | Filter Duplicates, Merge    |                                                                                                             |
| Filter Duplicates       | code               | Filters out existing duplicate leads           | Check For Duplicates         | Format For Database          |                                                                                                             |
| Merge                   | merge              | Combines branches for leads with and without enrichment | Update With Email, Check For Duplicates | Filter Duplicates           |                                                                                                             |
| Format For Database     | code               | Formats lead data to match Google Sheets CRM  | Filter Duplicates            | Add to Lead Database         |                                                                                                             |
| Add to Lead Database    | googleSheets       | Appends new leads to Google Sheets CRM         | Format For Database          | -                           |                                                                                                             |
| Sticky Note             | stickyNote         | Section header: Find Developers on GitHub      | -                           | -                           | ## 1. Find Developers on GitHub                                                                             |
| Sticky Note1            | stickyNote         | Section header: Find Developers emails with Hunter.io | -                           | -                           | ## 2. Find Developers emails using Hunter.io                                                                |
| Sticky Note2            | stickyNote         | Section header: Remove Duplicates and Save To CRM | -                           | -                           | ## 3. Remove Duplicates and Save To CRM Database                                                            |
| Sticky Note3            | stickyNote         | Full workflow summary and setup instructions   | -                           | -                           | ## GitHub Developer Scraper With Email Lookup and Lead Database\n\n## How it works\n1. Schedule Trigger: start trigger at a certain interval\n2. Define developer searches Code nodes: find a developer at a certain location and demographic\n3. Search GitHub users: Run code through GitHub to collect details\n4. Parse users: Analyze users' data\n5. Get users' details: Return users' data\n6. Extract developer data: clean and organize the returned users' data\n7. Need enrichment: check if no email\n8. Hunter.io email lookup: find and verify user email\n9. Update with email: organize data\n10. Check for duplicate data, remove duplicates, organize for the database and store to Google Sheets accordingly. \n\n## Setup\n* Schedule trigger to start\n* Get the GitHub API and insert it in the \"Search GitHub users\" HTTP request node. Also, insert it in the \"Get user details\" HTTP request node\n* Get Hunter.io API and insert it in the Hunter.io Email Lookup HTTP request\n* Use \"Get the template\" Google sheet\n\n## Get the template sheet: https://docs.google.com/spreadsheets/d/1HyoSj2HncMgap96eNxkzLEMxt1DjO6J4uOj8CHrYkFc/edit?usp=sharing |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger** node:  
   - Type: `Schedule Trigger`  
   - Configure to run every 1 week (interval: weeks = 1).

2. **Add a Code node named "Define Developer Searches":**  
   - Use version 2.  
   - Paste JS code to define developer searches as an array of objects with fields: location, language, minFollowers.  
   - Example criteria:  
     ```js
     const searches = [
       { location: "New Jersey", language: "javascript", minFollowers: 50 },
       { location: "Texas", language: "python", minFollowers: 100 },
       { location: "New York", language: "react", minFollowers: 200 }
     ];
     return searches.map(search => ({ json: search }));
     ```
   - Connect Schedule Trigger output to this node.

3. **Add an HTTP Request node named "Search GitHub Users":**  
   - URL: `https://api.github.com/search/users`  
   - Method: GET  
   - Query Parameters:  
     - `q`: `location:{{$json.location}} followers:>{{$json.minFollowers}} language:{{$json.language}}`  
     - `per_page`: 30  
   - Headers:  
     - Accept: `application/vnd.github.v3+json`  
     - Authorization: Use HTTP Header Auth with GitHub personal access token credential (name it e.g. `GitHub_API_Token`).  
   - Connect "Define Developer Searches" output to this node.

4. **Add a Code node named "Parse Users":**  
   - Version 2.  
   - JS code to iterate all input items, extract user login, profile url, avatar url, and user API url:  
     ```js
     const allInputs = $input.all();
     const developers = [];
     allInputs.forEach(input => {
       const users = input.json.items || [];
       users.forEach(user => {
         developers.push({
           json: {
             username: user.login,
             profileUrl: user.html_url,
             avatarUrl: user.avatar_url,
             userApiUrl: user.url,
           }
         });
       });
     });
     return developers;
     ```
   - Connect "Search GitHub Users" output to this node.

5. **Add an HTTP Request node named "Get User Details":**  
   - URL: `={{ $json.userApiUrl }}` (dynamic)  
   - Method: GET  
   - Headers:  
     - Accept: `application/vnd.github.v3+json`  
     - Authorization: Use same GitHub token credential.  
   - Connect "Parse Users" output to this node.

6. **Add a Code node named "Extract Developer Data":**  
   - Version 2.  
   - Paste JS code that:  
     - Extracts website and domain with URL parsing fallback.  
     - Calculates lead score based on followers, repos, and contact info.  
     - Determines lead quality (Cold, Warm, Hot).  
     - Formats final JSON with enriched fields (name, username, email, bio, company, location, website, domain, twitter, followers, publicRepos, leadScore, leadQuality, scrapedDate, notes).  
   - Connect "Get User Details" output to this node.

7. **Add an If node named "Need Email Enrichment?":**  
   - Configure conditions:  
     - `domain` not equal to empty string  
     - `email` equal to "Check website"  
   - Logical AND between conditions.  
   - Connect "Extract Developer Data" output to this node.

8. **Add an HTTP Request node named "Hunter.io Email Lookup":**  
   - URL: `https://api.hunter.io/v2/domain-search`  
   - Method: GET  
   - Query Parameters:  
     - `domain`: `={{ $json.domain }}`  
     - `api_key`: Use Hunter.io API key credential (name e.g. `Hunter_API_Key`)  
     - `limit`: 3  
   - Connect If node "true" output to this node.

9. **Add a Code node named "Update With Email":**  
   - Version 2.  
   - JS code merges Hunter.io emails, selects best confidence email, updates lead email field, adds emailConfidence:  
     ```js
     const hunterData = $input.item.json.data || {};
     const emails = hunterData.emails || [];
     const previousData = $('Extract Developer Data').item.json;

     let bestEmail = previousData.email;
     if (emails.length > 0) {
       const sortedEmails = emails.sort((a,b) => b.confidence - a.confidence);
       bestEmail = sortedEmails[0].value;
     }

     return {
       json: {
         ...previousData,
         email: bestEmail,
         emailConfidence: emails[0]?.confidence || 0
       }
     };
     ```
   - Connect "Hunter.io Email Lookup" output to this node.

10. **Add a Google Sheets node named "Check For Duplicates":**  
    - Operation: Read rows from Google Sheets.  
    - Spreadsheet ID: Use your CRM Google Sheets document ID.  
    - Sheet Name: "Sheet1" (or your sheet).  
    - Credentials: Google Sheets OAuth2 credentials.  
    - Connect "Update With Email" output (and also connect If node "false" output) to this node.

11. **Add a Code node named "Filter Duplicates":**  
    - Version 2.  
    - JS code to compare current lead against existing leads checking for duplicates by business name, email, or username in notes:  
      ```js
      const existingLeads = $('Check For Duplicates').all();
      const currentLead = $input.item.json;

      const isDuplicate = existingLeads.some(item => {
        const row = item.json;
        return row['Business Name'] === currentLead.name ||
               row.Email === currentLead.email ||
               (row.Notes && row.Notes.includes(currentLead.username));
      });

      if (isDuplicate) {
        return [];
      }
      return [{ json: currentLead }];
      ```
    - Connect "Check For Duplicates" output to this node.

12. **Add a Merge node named "Merge":**  
    - Mode: "chooseBranch"  
    - Use data of input 2.  
    - Connect "Check For Duplicates" output to Merge input 1.  
    - Connect "Filter Duplicates" output to Merge input 2.

13. **Add a Code node named "Format For Database":**  
    - Version 2.  
    - JS code formats lead into CRM schema and generates `leadId`:  
      ```js
      const data = $input.item.json;
      const timestamp = Date.now();
      const randomNum = Math.floor(Math.random() * 1000);
      const leadId = `GITHUB-${timestamp}-${randomNum}`;

      return {
        json: {
          leadId: leadId,
          businessName: data.name,
          contactName: data.name,
          jobTitle: "Developer",
          email: data.email,
          phone: "Not available",
          website: data.website,
          address: data.location,
          industry: "Software Development",
          companySize: data.company || "Freelancer",
          rating: 0,
          reviews: 0,
          leadScore: data.leadScore,
          leadQuality: data.leadQuality,
          scrapedDate: data.scrapedDate,
          status: "New - Scraped",
          contacted: "No",
          notes: data.notes,
          leadSource: data.leadSource
        }
      };
      ```
    - Connect "Merge" output to this node.

14. **Add a Google Sheets node named "Add to Lead Database":**  
    - Operation: Append row.  
    - Spreadsheet ID and Sheet Name same as "Check For Duplicates".  
    - Map columns to the fields output by "Format For Database".  
    - Credentials: Google Sheets OAuth2 credentials.  
    - Connect "Format For Database" output to this node.

15. **Optional:** Add Sticky Notes for organization and documentation with titles:  
    - "1. Find Developers on GitHub"  
    - "2. Find Developers emails using Hunter.io"  
    - "3. Remove Duplicates and Save To CRM Database"  
    - Full workflow overview and setup instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| GitHub Developer Scraper With Email Lookup and Lead Database Sync: Automates searching GitHub developers by location and skill, enriches emails using Hunter.io, deduplicates entries, and syncs to Google Sheets CRM. Setup requires GitHub API token, Hunter.io API key, and Google Sheets OAuth2 credentials.                                                                                                                                                                                           | Full workflow description in Sticky Note3                                                                                      |
| GitHub API token must have permission to access user data; beware of API rate limits (https://docs.github.com/en/rest/overview/resources-in-the-rest-api#rate-limiting).                                                                                                                                                                                                                                                                                                                               | GitHub API documentation                                                                                                       |
| Hunter.io API key is required for email enrichment; usage is credit-limited depending on your Hunter.io plan.                                                                                                                                                                                                                                                                                                                                                                                       | Hunter.io API documentation: https://hunter.io/api-docs                                                                         |
| Google Sheets document used for the CRM database should have columns matching the mapped fields, including Lead ID, Business Name, Email, Notes, etc. Template sheet link: https://docs.google.com/spreadsheets/d/1HyoSj2HncMgap96eNxkzLEMxt1DjO6J4uOj8CHrYkFc/edit?usp=sharing                                                                                                                                                                                                                      | Template Google Sheet                                                                                                           |
| To avoid hitting GitHub API rate limits, consider caching results or adjusting schedule trigger frequency.                                                                                                                                                                                                                                                                                                                                                                                          |                                                                                                                                |
| In case of malformed website URLs from GitHub profiles, extraction falls back to raw string to prevent workflow failure.                                                                                                                                                                                                                                                                                                                                                                            |                                                                                                                                |
| The lead scoring system is customizable by adjusting weights in the Extract Developer Data code node.                                                                                                                                                                                                                                                                                                                                                                                               |                                                                                                                                |

---

**Disclaimer:** The provided content originates exclusively from an automated n8n workflow. It complies fully with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.