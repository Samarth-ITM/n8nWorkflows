Instagram Influencer Finder with Bright Data (Auto-Filter & Save to Sheets)

https://n8nworkflows.xyz/workflows/instagram-influencer-finder-with-bright-data--auto-filter---save-to-sheets--6621


# Instagram Influencer Finder with Bright Data (Auto-Filter & Save to Sheets)

### 1. Workflow Overview

This workflow is designed to automate the process of identifying valuable Instagram influencer profiles using Bright Data’s scraping capabilities and saving qualified leads into a Google Sheet. It targets marketing professionals, influencer outreach teams, and brand researchers who want to filter Instagram profiles based on specific business criteria without manual checking.

The workflow is logically divided into three main blocks:

- **1.1 User Input Setup**  
  Starts the workflow manually and sets the Instagram profile URL to analyze.

- **1.2 Profile Scraping and Smart Filtering**  
  Uses Bright Data to scrape Instagram profile data and applies business rules to filter qualified profiles.

- **1.3 Smart Output Handling**  
  Saves qualified profiles to Google Sheets or skips unqualified ones, ensuring clean data output.

---

### 2. Block-by-Block Analysis

#### 2.1 User Input Setup

**Overview:**  
This block initiates the workflow manually and provides a way to input the Instagram profile URL for analysis. It is the entry point for the workflow and can be adapted to accept URLs from other sources such as Google Sheets or webhooks.

**Nodes Involved:**  
- Trigger - Start Manually  
- Set Instagram Profile URL

**Node Details:**

- **Trigger - Start Manually**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow on user demand; useful for testing or one-off executions.  
  - *Configuration:* No parameters set; simply triggers the workflow.  
  - *Input/Output:* No input; outputs a trigger signal to "Set Instagram Profile URL".  
  - *Failures:* None expected; user-dependent.  
  - *Sub-workflow:* None.

- **Set Instagram Profile URL**  
  - *Type:* Set Node  
  - *Role:* Defines or sets the Instagram profile URL to be analyzed, here hardcoded as "https://www.instagram.com/cristiano/".  
  - *Configuration:* Assigns static string "post URL" with the Instagram profile URL.  
  - *Key Expressions:* None dynamic; static assignment.  
  - *Input:* Trigger from manual start.  
  - *Output:* Passes the profile URL as JSON field "post URL" to the next node.  
  - *Failures:* Misconfiguration can lead to invalid URLs; input can be modified to dynamic sources.  
  - *Sub-workflow:* None.

---

#### 2.2 Profile Scraping and Smart Filtering

**Overview:**  
This block interacts with Bright Data to scrape detailed Instagram profile data and then filters profiles based on four business criteria: verified account, follower count > 10,000, engagement rate > 0.5%, and professional account status. Only profiles meeting all criteria proceed.

**Nodes Involved:**  
- Bright Data - Scrape IG Profile  
- Filter - Qualified Profile?

**Node Details:**

- **Bright Data - Scrape IG Profile**  
  - *Type:* Bright Data API Node (webScrapper resource)  
  - *Role:* Sends the Instagram profile URL to Bright Data’s Instagram profile scraping service and retrieves structured profile data.  
  - *Configuration:*  
    - URL parameter is dynamically assigned from the incoming “post URL” field.  
    - Uses Bright Data dataset ID "gd_l1vikfch901nx3by4" for Instagram Profiles.  
    - Credentials: Bright Data API credentials configured.  
  - *Key Expressions:* URL: `={{ $json["post URL"] }}`  
  - *Input:* Profile URL from "Set Instagram Profile URL" node.  
  - *Output:* Instagram profile data fields including followers, avg_engagement, is_verified, is_professional_account, biography, posts_count, account, etc.  
  - *Failures:*  
    - API errors or rate limits from Bright Data.  
    - Invalid or private Instagram profiles returning incomplete data.  
    - Network timeouts or auth failures.  
  - *Sub-workflow:* None.

- **Filter - Qualified Profile?**  
  - *Type:* IF Node  
  - *Role:* Applies business logic filters to determine if the profile is valuable enough to save.  
  - *Configuration:* Checks all these conditions:  
    - followers > 10,000  
    - avg_engagement > 0.005 (0.5%)  
    - is_verified is true  
    - is_professional_account is true  
  - *Key Expressions:*  
    - `{{$json.followers}} > 10000`  
    - `{{$json.avg_engagement}} > 0.005`  
    - `{{$json.is_verified}} === true`  
    - `{{$json.is_professional_account}} === true`  
  - *Input:* Instagram profile JSON from Bright Data.  
  - *Output:*  
    - True branch: passes data to "Save to Google Sheet" node.  
    - False branch: passes data to "Skip - Unqualified Profile" node.  
  - *Failures:* Expression evaluation errors if fields are missing or malformed.  
  - *Sub-workflow:* None.

---

#### 2.3 Smart Output Handling

**Overview:**  
This block handles saving the qualified profiles to a Google Sheet and quietly skipping those that do not meet the criteria to keep the output clean and relevant.

**Nodes Involved:**  
- Save to Google Sheet  
- Skip - Unqualified Profile

**Node Details:**

- **Save to Google Sheet**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends profile data to a specified Google Sheet for lead tracking and further analysis.  
  - *Configuration:*  
    - Operation: Append  
    - Document ID and Sheet Name are pre-configured to a specific Google Sheet (ID: `1z0aqvzLogALkYK4NBcT49uqO7xGaEf_twqCg2K3vj4k`, Sheet: `gid=0`)  
    - Columns mapped: Biography, Followers, Total posts, Account name, Average engagement  
    - Credentials: Google Sheets OAuth2 credentials configured.  
  - *Input:* Profile JSON from filter node (qualified only).  
  - *Output:* None (end of qualified branch).  
  - *Failures:*  
    - Auth errors if Google credentials expire.  
    - API rate limits or quota exceeded on Google Sheets.  
    - Data mapping errors if fields are missing.  
  - *Sub-workflow:* None.

- **Skip - Unqualified Profile**  
  - *Type:* No Operation (NoOp) Node  
  - *Role:* Ends the workflow silently for profiles failing qualification checks.  
  - *Configuration:* None; acts as a sink node.  
  - *Input:* Profiles from the false branch of the filter.  
  - *Output:* None.  
  - *Failures:* None expected.  
  - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name                 | Node Type                | Functional Role                      | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                                                      |
|---------------------------|--------------------------|------------------------------------|---------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Trigger - Start Manually   | Manual Trigger           | Workflow starter                   | -                         | Set Instagram Profile URL   | Section 1: User Input Setup — Starts the workflow manually, useful for tests or one-off runs.                                    |
| Set Instagram Profile URL  | Set Node                 | Defines Instagram URL input        | Trigger - Start Manually   | Bright Data - Scrape IG Profile | Section 1: User Input Setup — Sets the Instagram profile URL to analyze; can be replaced by dynamic inputs.                      |
| Bright Data - Scrape IG Profile | Bright Data API Node   | Scrapes Instagram profile data    | Set Instagram Profile URL  | Filter - Qualified Profile? | Section 2: Profile Scraping + Smart Filtering — Retrieves profile data with Bright Data’s Instagram scraper.                     |
| Filter - Qualified Profile?| IF Node                  | Applies qualification filters      | Bright Data - Scrape IG Profile | Save to Google Sheet, Skip - Unqualified Profile | Section 2: Profile Scraping + Smart Filtering — Filters profiles by verified, followers >10K, engagement >0.5%, professional account. |
| Save to Google Sheet       | Google Sheets Node       | Saves qualified profiles           | Filter - Qualified Profile? (true) | -                          | Section 3: Smart Output Handling — Appends qualified profiles to Google Sheets for lead tracking.                                |
| Skip - Unqualified Profile| NoOp Node                | Ends workflow for unqualified profiles | Filter - Qualified Profile? (false) | -                          | Section 3: Smart Output Handling — Silently skips unqualified profiles to keep outputs clean.                                     |
| Sticky Note                | Sticky Note              | Documentation and explanation      | -                         | -                          | Covers Section 1: User Input Setup                                                                                              |
| Sticky Note1               | Sticky Note              | Documentation and explanation      | -                         | -                          | Covers Section 2: Profile Scraping + Smart Filtering                                                                           |
| Sticky Note2               | Sticky Note              | Documentation and explanation      | -                         | -                          | Covers Section 3: Smart Output Handling                                                                                         |
| Sticky Note9               | Sticky Note              | Workflow assistance contact info   | -                         | -                          | Provides support contact and resource links                                                                                    |
| Sticky Note4               | Sticky Note              | Full workflow description          | -                         | -                          | Comprehensive workflow goal and section explanations                                                                           |
| Sticky Note5               | Sticky Note              | Affiliate link for Bright Data     | -                         | -                          | Bright Data referral link and commission notice                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually for testing or ad-hoc runs.  
   - No parameters needed.

2. **Add a Set Node to Define Instagram Profile URL**  
   - Type: Set  
   - Add a new field: Name = `post URL`, Type = String, Value = `"https://www.instagram.com/cristiano/"` (or replace with any desired Instagram URL).  
   - Connect the Manual Trigger output to this Set node.

3. **Add Bright Data Node to Scrape Instagram Profile**  
   - Type: Bright Data API Node (resource: webScrapper)  
   - Parameters:  
     - URLs: Use expression `=[{"url":"{{ $json["post URL"] }}" }]` to pass the URL dynamically.  
     - Dataset ID: Set to `"gd_l1vikfch901nx3by4"` (Instagram Profiles dataset).  
   - Credentials: Configure Bright Data API credentials with valid API key.  
   - Connect output of Set node to this Bright Data node.

4. **Add an IF Node to Filter Qualified Profiles**  
   - Type: IF  
   - Add conditions (AND combinator):  
     - `$json.followers` > 10000  
     - `$json.avg_engagement` > 0.005  
     - `$json.is_verified` == true  
     - `$json.is_professional_account` == true  
   - Connect the Bright Data node’s output to the IF node.

5. **Add Google Sheets Node to Save Qualified Profiles**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: `"1z0aqvzLogALkYK4NBcT49uqO7xGaEf_twqCg2K3vj4k"` (replace with your own sheet ID).  
   - Sheet Name: `"gid=0"` (or your target sheet tab name).  
   - Columns to map:  
     - Account name: `{{$json.account}}`  
     - Followers: `{{$json.followers}}`  
     - Total posts: `{{$json.posts_count}}`  
     - Average engagement: `{{$json.avg_engagement}}`  
     - Biography: `{{$json.biography}}`  
   - Credentials: Set up Google Sheets OAuth2 credentials.  
   - Connect the IF node’s true output to this node.

6. **Add No Operation Node to Skip Unqualified Profiles**  
   - Type: NoOp  
   - Connect the IF node’s false output to this node.

7. **Validate and Save Your Workflow**  
   - Test manually from the trigger node.  
   - Adjust Instagram URL in the Set node to test various profiles.  
   - Check your Google Sheet for appended rows matching the filter criteria.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| For support or questions, contact Yaron at Yaron@nofluff.online. Explore tutorials and tips on YouTube and LinkedIn.                      | YouTube: https://www.youtube.com/@YaronBeen/videos, LinkedIn: https://www.linkedin.com/in/yaronbeen/ |
| Copy of example Google Sheet used for testing: https://docs.google.com/spreadsheets/d/1hZFawprfr_a6JC26-0LCQUoz-VuX58A2YxlphQ9Kn_0/edit  | Spreadsheet template for leads                                                                       |
| Bright Data referral link for free credits and support: https://get.brightdata.com/1tndi4600b25                                              | Affiliate link                                                                                        |
| This workflow saves hours of manual Instagram profile checking, ideal for influencer marketing, brand outreach, and automation enthusiasts.| Workflow benefit summary                                                                              |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.