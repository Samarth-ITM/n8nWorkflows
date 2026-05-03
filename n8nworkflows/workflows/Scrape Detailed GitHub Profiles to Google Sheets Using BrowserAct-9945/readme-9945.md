Scrape Detailed GitHub Profiles to Google Sheets Using BrowserAct

https://n8nworkflows.xyz/workflows/scrape-detailed-github-profiles-to-google-sheets-using-browseract-9945


# Scrape Detailed GitHub Profiles to Google Sheets Using BrowserAct

---

### 1. Workflow Overview

This workflow automates detailed scraping of GitHub user profiles and organizes the enriched data into structured Google Sheets reports. It is designed for data analysts, recruiters, or researchers who need comprehensive GitHub user activity data aggregated efficiently.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception & Looping:** Manual trigger or scheduled start, fetches a list of GitHub profile URLs from a Google Sheet, and iterates over each user one-by-one.
- **1.2 Scraping & Data Consolidation:** For each GitHub user, sends a Slack notification, runs a BrowserAct scraping task on the user’s GitHub page, retrieves and fixes the scraped JSON data, merging it into a single structured object.
- **1.3 Dynamic Google Sheet Management:** Dynamically creates and clears a dedicated Google Sheet tab named after each GitHub user, preparing the sheet for data insertion.
- **1.4 Data Splitting & Output:** Splits the consolidated user data into profile info, social links, and repositories, then appends each category into the corresponding section of the user’s Google Sheet tab.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Looping

- **Overview:**  
  This block initiates the workflow via manual trigger, retrieves the master list of GitHub profile URLs from a Google Sheet, and processes each profile individually using batching.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get row(s) in sheet (Google Sheets)  
  - Loop Over Items (SplitInBatches)

- **Node Details:**

  1. **When clicking ‘Execute workflow’**  
     - Type: Manual Trigger  
     - Role: Starts the workflow on manual execution  
     - Config: No parameters, simple trigger node  
     - Input: None  
     - Output: Triggers next node in flow  
     - Edge cases: No failures expected but no automatic scheduling

  2. **Get row(s) in sheet**  
     - Type: Google Sheets (Read)  
     - Role: Reads rows from a specified Google Sheet containing GitHub profile URLs  
     - Config: Reads from sheet "Source Top GitHub Contributors by Language & Location" in document ID "18sw7io0yJOTDzvcknGmjBBqtK154CLk3k0FoWJZbfI0"  
     - Credentials: Google Sheets OAuth2  
     - Input: Trigger from manual node  
     - Output: List of rows with GitHub profile URLs  
     - Edge cases: Sheet access permission errors, empty sheet, malformed data (missing URL column)

  3. **Loop Over Items**  
     - Type: SplitInBatches  
     - Role: Processes each GitHub profile individually to avoid overload and ensure clean execution  
     - Config: Default batch size (1) to process users one at a time  
     - Input: Rows array from Google Sheets  
     - Output: Individual GitHub user data for downstream processing  
     - Edge cases: Batch processing errors, empty batches

---

#### 2.2 Scraping & Data Consolidation

- **Overview:**  
  This block handles scraping of each GitHub user profile via BrowserAct, sends Slack notifications when scraping starts, retrieves scraping results, and consolidates raw scraped JSON into a clean structured object.

- **Nodes Involved:**  
  - Send a message (Slack)  
  - Run a workflow task (BrowserAct)  
  - Get details of a workflow task (BrowserAct)  
  - Code in JavaScript (Code)

- **Node Details:**

  1. **Send a message**  
     - Type: Slack  
     - Role: Sends notification message to Slack channel announcing start of scraping for each user  
     - Config: OAuth2 authentication with Slack, predefined channel "all-browseract-workflow-test"  
     - Input: Current GitHub user data from Loop Over Items  
     - Output: Passes data downstream without modification  
     - Edge cases: Slack auth failure, network issues, invalid channel ID

  2. **Run a workflow task**  
     - Type: BrowserAct Workflow Execution Node  
     - Role: Executes a BrowserAct scraping workflow on the user’s GitHub profile URL  
     - Config: Uses workflow ID "57481883835648378", input parameter "Target_Page" set dynamically to current user’s URL  
     - Credentials: BrowserAct API  
     - Input: Current user data from Loop  
     - Output: Returns a task ID for scraping job  
     - Edge cases: BrowserAct API errors, invalid URL, workflow task creation failure

  3. **Get details of a workflow task**  
     - Type: BrowserAct Workflow Task Retrieval  
     - Role: Polls until the scraping task finishes, retrieves scraping results  
     - Config: Waits up to 900 seconds, polls every 20 seconds, uses task ID dynamically from previous node  
     - Credentials: BrowserAct API  
     - Input: Task ID from previous node  
     - Output: Raw scraped data including text output  
     - Edge cases: Timeout if scraping takes too long, task failure, malformed output

  4. **Code in JavaScript**  
     - Type: Code  
     - Role: Fixes and parses broken JSON strings from scraping output, merges multiple JSON objects into one, parses nested JSON strings recursively  
     - Configuration details:  
        - Uses regex to add missing quotes around keys  
        - Parses JSON array and merges objects  
        - Parses nested stringified JSON values  
        - Throws error if JSON parsing fails after attempted fix  
     - Input: Raw scraping output string  
     - Output: Single clean JSON object with consolidated user data  
     - Edge cases: Unexpected string formats, parsing failures, malformed JSON beyond regex fix

---

#### 2.3 Dynamic Google Sheet Management

- **Overview:**  
  This block dynamically creates a new sheet tab named after the GitHub user, clears any existing data, and prepares the sheet with clean headers for data insertion.

- **Nodes Involved:**  
  - Create sheet (Google Sheets)  
  - Edit Fields (Set)  
  - Clear sheet (Google Sheets)  
  - Append row in sheet (Google Sheets)

- **Node Details:**

  1. **Create sheet**  
     - Type: Google Sheets (Create sheet)  
     - Role: Creates a new tab within the output spreadsheet, named after the scraped user's name  
     - Config: Spreadsheet ID "1OPN-GHxA1jhulioo0v-x63ZiVhGjl-ed-QBIxH8KlVs"  
     - Input: User’s Name from parsed JSON object  
     - Output: Passes sheet name downstream  
     - Edge cases: Sheet name conflicts, permissions, spreadsheet access errors

  2. **Edit Fields**  
     - Type: Set  
     - Role: Initializes and resets key user profile fields with empty strings for clean header row setup  
     - Config: Sets fields like Name, Username, Location, Site, link, Title, Summary, Date, Programming Language, Stars to empty strings  
     - Input: User name for sheet creation  
     - Output: Prepared empty fields object for clearing and appending  
     - Execute Once: True (runs only once per user)  
     - Edge cases: Field naming mismatches

  3. **Clear sheet**  
     - Type: Google Sheets (Clear)  
     - Role: Clears all existing data in the newly created user-named sheet tab to start fresh  
     - Config: Uses same spreadsheet ID and dynamic sheet name from "Code in JavaScript" node output  
     - Execute Once: True  
     - Edge cases: Sheet not found, permission errors, clearing failures

  4. **Append row in sheet**  
     - Type: Google Sheets (Append)  
     - Role: Adds the empty header row or initial data row to the cleared sheet tab using fields set previously  
     - Config: Maps key fields (Name, Username, Summary) from "Edit Fields" node, uses dynamic sheet name  
     - Execute Once: True  
     - Edge cases: Mapping errors, append failures, spreadsheet limits

---

#### 2.4 Data Splitting & Output

- **Overview:**  
  This block splits the consolidated user data into three logical parts—main profile info, social links, and repositories—and appends each to the appropriate section of the user’s Google Sheet tab, producing a clear, organized report.

- **Nodes Involved:**  
  - Merge (chooseBranch)  
  - Split Out (for Links)  
  - Split Out1 (for main profile info)  
  - Split Out2 (for Repositories)  
  - User Data (Google Sheets Append)  
  - User Links (Google Sheets Append)  
  - User Repositories (Google Sheets Append)  
  - Merge1 (merges append results for synchronization)

- **Node Details:**

  1. **Merge**  
     - Type: Merge (chooseBranch mode)  
     - Role: Takes output from code node and routes to three split nodes for Links, Profile, and Repositories data  
     - Input: Parsed user JSON object, output from "Code in JavaScript"  
     - Output: Three branches to split nodes  
     - Edge cases: Branch routing errors

  2. **Split Out (Links)**  
     - Type: SplitOut  
     - Role: Extracts the "Links" array/field from consolidated user data  
     - Input: From Merge node branch 0  
     - Output: User Links data for appending  
     - Edge cases: Missing or empty Links field

  3. **Split Out1 (Profile info)**  
     - Type: SplitOut  
     - Role: Extracts main profile fields: Name, Username, Summary, Location  
     - Input: From Merge node branch 1  
     - Output: User Data for appending  
     - Edge cases: Missing profile fields

  4. **Split Out2 (Repositories)**  
     - Type: SplitOut  
     - Role: Extracts "Repositories" data array for appending  
     - Input: From Merge node branch 2  
     - Output: User Repositories data for appending  
     - Edge cases: Missing repositories, empty arrays

  5. **User Data (Google Sheets Append)**  
     - Type: Google Sheets (Append)  
     - Role: Appends main user profile info into the user-named sheet tab  
     - Config: Maps Name, Summary, Location, Username fields dynamically  
     - Input: From Split Out1  
     - Output: Confirmation of append  
     - Edge cases: Append errors, mapping mismatches

  6. **User Links (Google Sheets Append)**  
     - Type: Google Sheets (Append)  
     - Role: Appends social links data into user sheet  
     - Config: Maps Site and link fields dynamically  
     - Input: From Split Out  
     - Output: Confirmation of append  
     - Edge cases: Missing link data, append failures

  7. **User Repositories (Google Sheets Append)**  
     - Type: Google Sheets (Append)  
     - Role: Appends repository details (Date, Stars, Programming Language, Repo Name) into user sheet  
     - Config: Maps relevant fields dynamically  
     - Input: From Split Out2  
     - Output: Confirmation of append  
     - Edge cases: Missing repo data, append errors

  8. **Merge1**  
     - Type: Merge (numberInputs=3)  
     - Role: Synchronizes completion of the three append operations before processing next user  
     - Input: Outputs from User Data, User Links, User Repositories nodes  
     - Output: Feeds back to Loop Over Items to continue processing  
     - Edge cases: Synchronization failure, partial append completion

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                                       | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                                                       |
|-----------------------------|---------------------------------|------------------------------------------------------|----------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                 | Starts workflow on manual execution                   | None                             | Get row(s) in sheet             |                                                                                                                                |
| Get row(s) in sheet          | Google Sheets (Read)             | Reads GitHub profile URLs from master Google Sheet   | When clicking ‘Execute workflow’ | Loop Over Items                 |                                                                                                                                |
| Loop Over Items              | SplitInBatches                  | Processes each GitHub profile one-by-one              | Get row(s) in sheet              | Send a message, Run a workflow task | Sticky Note - Input & Loop: Explains input fetching and batch processing                                                      |
| Send a message              | Slack                          | Sends Slack notification when scraping starts        | Loop Over Items                  | None                           | Sticky Note - Scrape & Process: Describes Slack notification in scrape step                                                     |
| Run a workflow task          | BrowserAct Workflow             | Starts scraping task on GitHub profile URL            | Loop Over Items                  | Get details of a workflow task | Sticky Note - Scrape & Process: Describes BrowserAct scraping workflow execution                                                |
| Get details of a workflow task | BrowserAct Workflow Retrieval | Polls and retrieves scraping results                   | Run a workflow task              | Code in JavaScript             | Sticky Note - Scrape & Process                                                                                                  |
| Code in JavaScript           | Code                           | Parses and merges scraped JSON data                    | Get details of a workflow task   | Create sheet, Merge            | Sticky Note - Scrape & Process                                                                                                  |
| Create sheet                | Google Sheets (Create)          | Dynamically creates new sheet tab named after user    | Code in JavaScript               | Edit Fields                   | Sticky Note - Dynamic Sheet Creation: Describes dynamic sheet creation and management                                           |
| Edit Fields                 | Set                            | Initializes empty header fields for sheet preparation | Create sheet                    | Clear sheet                   | Sticky Note - Dynamic Sheet Creation                                                                                           |
| Clear sheet                 | Google Sheets (Clear)           | Clears existing data in the user’s sheet tab          | Edit Fields                    | Append row in sheet           | Sticky Note - Dynamic Sheet Creation                                                                                           |
| Append row in sheet         | Google Sheets (Append)          | Appends header/initial row to cleared sheet           | Clear sheet                    | Merge                        | Sticky Note - Dynamic Sheet Creation                                                                                           |
| Merge                      | Merge (chooseBranch)             | Routes consolidated data into Links, Profile, Repos branches | Code in JavaScript             | Split Out, Split Out1, Split Out2 | Sticky Note - Data Output: Explains data splitting for output                                                                  |
| Split Out                  | SplitOut                        | Extracts Links data                                    | Merge                         | User Links                   | Sticky Note - Data Output                                                                                                       |
| Split Out1                 | SplitOut                        | Extracts main profile data                             | Merge                         | User Data                    | Sticky Note - Data Output                                                                                                       |
| Split Out2                 | SplitOut                        | Extracts repositories data                             | Merge                         | User Repositories            | Sticky Note - Data Output                                                                                                       |
| User Data                  | Google Sheets (Append)          | Appends profile info to user’s sheet                   | Split Out1                    | Merge1                      | Sticky Note - Data Output                                                                                                       |
| User Links                 | Google Sheets (Append)          | Appends social links to user’s sheet                    | Split Out                     | Merge1                      | Sticky Note - Data Output                                                                                                       |
| User Repositories          | Google Sheets (Append)          | Appends repository details to user’s sheet             | Split Out2                    | Merge1                      | Sticky Note - Data Output                                                                                                       |
| Merge1                     | Merge (numberInputs=3)           | Synchronizes completion of all append operations       | User Data, User Links, User Repositories | Loop Over Items       |                                                                                                                                |
| Sticky Note - Intro         | Sticky Note                    | Provides workflow introduction and summary             | None                         | None                        | Detailed template overview, requirements, and help links                                                                      |
| Sticky Note - How to Use    | Sticky Note                    | Instructions on setup and usage                         | None                         | None                        | Setup instructions for credentials, BrowserAct template, input sheet, and Slack channel                                        |
| Sticky Note - Need Help     | Sticky Note                    | Help links to videos and blog                            | None                         | None                        | List of helpful video guides and blog posts on BrowserAct and GitHub scraping                                                  |
| Sticky Note - Input & Loop  | Sticky Note                    | Describes Input Reception & Loop block                  | None                         | None                        | Covers trigger, Google Sheets input, and batching process                                                                     |
| Sticky Note - Scrape & Process | Sticky Note                  | Describes Scraping & Data Consolidation block           | None                         | None                        | Covers Slack notification, BrowserAct scraping, and JSON parsing                                                             |
| Sticky Note - Dynamic Sheet Creation | Sticky Note             | Describes dynamic sheet creation and prep               | None                         | None                        | Covers sheet creation, clearing, and initial row append                                                                       |
| Sticky Note - Data Output   | Sticky Note                    | Describes data splitting and output block               | None                         | None                        | Covers splitting of consolidated data and appending each category into sheets                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - Purpose: To start workflow manually

2. **Add Google Sheets Node to Read Input:**  
   - Type: Google Sheets (Read Rows)  
   - Connect from Manual Trigger  
   - Configure Spreadsheet ID: `18sw7io0yJOTDzvcknGmjBBqtK154CLk3k0FoWJZbfI0`  
   - Sheet name: `"Source Top GitHub Contributors by Language & Location"` (use sheet ID or name)  
   - Credentials: Setup Google Sheets OAuth2 API credentials

3. **Add SplitInBatches Node:**  
   - Type: SplitInBatches  
   - Connect from Google Sheets Read node  
   - Default batch size (1) to process users one at a time

4. **Add Slack Node to Send Notification:**  
   - Type: Slack  
   - Connect from SplitInBatches node (first output)  
   - Configure OAuth2 Slack credentials  
   - Set Channel ID to your Slack channel (e.g., "all-browseract-workflow-test")  
   - Message Text: "Users Data Scrapped from Github" or dynamic message

5. **Add BrowserAct Node to Run Scraping Workflow:**  
   - Type: BrowserAct Workflow Node  
   - Connect from SplitInBatches node (second output)  
   - Set workflow ID: `57481883835648378` (BrowserAct workflow for GitHub scraping)  
   - Input Parameters: Set "Target_Page" to the current item’s `URL` field (`={{ $json.URL }}`)  
   - Credentials: BrowserAct API credentials

6. **Add BrowserAct Node to Get Scraper Task Results:**  
   - Type: BrowserAct Workflow Task Retrieval  
   - Connect from Run BrowserAct node  
   - Set "taskId" parameter dynamically to `={{ $json.id }}`  
   - Operation: `getTask`  
   - Max wait time: 900 seconds  
   - Polling interval: 20 seconds  
   - Credentials: BrowserAct API credentials

7. **Add Code Node to Fix and Parse JSON:**  
   - Type: Code (JavaScript)  
   - Connect from BrowserAct getTask node  
   - Paste the provided JS code to fix malformed JSON, merge objects, and parse nested JSON strings  
   - Input: raw scraping output string from previous node

8. **Add Google Sheets Node to Create New Sheet Tab:**  
   - Type: Google Sheets (Create Sheet)  
   - Connect from Code node  
   - Spreadsheet ID: `1OPN-GHxA1jhulioo0v-x63ZiVhGjl-ed-QBIxH8KlVs`  
   - Title: Dynamic sheet name from user’s Name field (`={{ $json.Name }}`)  
   - Credentials: Google Sheets OAuth2

9. **Add Set Node to Initialize Header Fields:**  
   - Type: Set  
   - Connect from Create Sheet node  
   - Add fields: Name, Username, Location, Site, link, Title, Summary, Date, Programing Language, Stars - all set to empty string  
   - Execute Once: Enabled

10. **Add Google Sheets Node to Clear Sheet:**  
    - Type: Google Sheets (Clear)  
    - Connect from Set node  
    - Spreadsheet: Same as above  
    - Sheet name: Dynamic from Code node’s user Name field  
    - Execute Once: Enabled  
    - Credentials: Google Sheets OAuth2

11. **Add Google Sheets Node to Append Header Row:**  
    - Type: Google Sheets (Append Row)  
    - Connect from Clear Sheet node  
    - Spreadsheet and sheet same as above  
    - Map columns Name, Username, Summary from Set node fields  
    - Execute Once: Enabled  
    - Credentials: Google Sheets OAuth2

12. **Add Merge Node (chooseBranch mode):**  
    - Type: Merge  
    - Connect from Code node (second output)  
    - Configure to route data into three branches: Links, Profile, Repositories

13. **Add Three SplitOut Nodes:**  
    - Split Out (Links): Extract "Links" field  
    - Split Out1 (Profile): Extract Name, Username, Summary, Location  
    - Split Out2 (Repositories): Extract "Repositories" field

14. **Add Three Google Sheets Append Nodes:**  
    - User Data: Append profile info (Name, Summary, Location, Username)  
    - User Links: Append social links (Site, link)  
    - User Repositories: Append repo info (Date, Stars, Programming Lang, Repositories_Name)  
    - Each uses spreadsheet ID and dynamic sheet name from Code node user Name field  
    - Credentials: Google Sheets OAuth2

15. **Add Merge Node (numberInputs=3):**  
    - Merge1: Synchronizes completion of the three append nodes  
    - Connect from the three Google Sheets append nodes  
    - Output connects back to Loop Over Items node to continue processing

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow requires a BrowserAct API account and the BrowserAct n8n Community Node ([n8n Nodes BrowserAct](https://www.npmjs.com/package/n8n-nodes-browseract-workflows))                                                  | BrowserAct API and n8n integration                                                              |
| A BrowserAct scraping template named "Scraping GitHub Users Activity & Data (Based on Source Top GitHub Contributors by Language & Location)" must be set up and used in BrowserAct                                          | BrowserAct template setup                                                                       |
| Google Sheets credentials need to be configured for both input (reading GitHub URLs) and output (writing enriched data)                                                                                                       | Google Sheets OAuth2 credentials                                                                |
| Slack OAuth2 credentials are required to send notifications to a specific Slack channel                                                                                                                                        | Slack OAuth2 setup                                                                             |
| Helpful video tutorial links and blog are available for setup and customization:                                                                                                                                             | [Discord](https://discord.com/invite/UpnCKd7GaU), [Blog](https://www.browseract.com/blog)         |
| YouTube help videos for BrowserAct key, workflow ID, n8n connection, and usage:                                                                                                                                               | [How to Find Your BrowseAct API Key & Workflow ID](https://www.youtube.com/watch?v=pDjoZWEsZlE) |
|                                                                                                                                                                                                                               | [How to Connect n8n to Browseract](https://www.youtube.com/watch?v=RoYMdJaRdcQ)                 |
|                                                                                                                                                                                                                               | [How to Use & Customize BrowserAct Templates](https://www.youtube.com/watch?v=CPZHFUASncY)      |
|                                                                                                                                                                                                                               | [How to Use the BrowserAct N8N Community Node](https://youtu.be/j0Nlba2pRLU)                    |
|                                                                                                                                                                                                                               | [GitHub Data Mining: Extracting User Profiles & Repositories with N8N](https://youtu.be/YjINoZgqx0M) |

---

**Disclaimer:** The provided content originates solely from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.