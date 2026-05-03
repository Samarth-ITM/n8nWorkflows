Scrape LinkedIn Post Comments & Reactions with Browserflow → Export to Google Sheets

https://n8nworkflows.xyz/workflows/scrape-linkedin-post-comments---reactions-with-browserflow---export-to-google-sheets-10787


# Scrape LinkedIn Post Comments & Reactions with Browserflow → Export to Google Sheets

### 1. Workflow Overview

This workflow automates scraping LinkedIn post comments and reactions using Browserflow and exports the extracted data into Google Sheets for lead generation and sales automation purposes. It is designed for manual execution but can be customized for scheduling and filtering. The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Fetches LinkedIn post URLs from a Google Sheets document that contains the posts to be scraped.
- **1.2 Scraping Execution:** Uses the Browserflow node to scrape comments and reactions from the specified LinkedIn post URL.
- **1.3 Data Processing and Export:** Splits the scraped data into comments and reactions subsets and appends each subset to the corresponding Google Sheets tabs ("Comments" and "Reactions"). It also updates the original post record to mark it as processed with a timestamp.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block retrieves the LinkedIn post URL that needs to be scraped from a Google Sheets document. It acts as the starting point by fetching the post URL and associated metadata like row number for later updating.

- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’`  
  - `Get Post Url`  
  - `Update row in sheet`

- **Node Details:**

  1. **When clicking ‘Execute workflow’**  
     - Type: Manual Trigger  
     - Role: Entry point of the workflow, manually initiated by the user.  
     - Configuration: No parameters; simply waits for manual execution.  
     - Input/Output: No inputs; outputs trigger the next node.  
     - Edge Cases: User forgetting to run manually; no posts available in sheet.  

  2. **Get Post Url**  
     - Type: Google Sheets (Read)  
     - Role: Reads post URLs from the "Posts" sheet.  
     - Configuration: Reads data from Google Sheet with ID `1huUaOdCXtioJgUpJHbK8cm-LLTtw-R7B8N4RKPkbois`, sheet by GID `1217532733`. Filters by `scraped_at` column (filter details not explicit, likely to pick unprocessed rows).  
     - Key Expressions: Accessed later as `$('Get Post Url').item.json.url` and `row_number`.  
     - Input: Trigger from manual node  
     - Output: Emits post URL and metadata for next step.  
     - Edge Cases: Empty sheet; multiple rows returned (only first processed here); API credential failure.  

  3. **Update row in sheet**  
     - Type: Google Sheets (Update)  
     - Role: Marks the post as scraped by updating the `scraped_at` timestamp on the same row.  
     - Configuration: Updates the row identified by `row_number` from `Get Post Url` with current timestamp `$now`.  
     - Key Expressions: `row_number` and `$now` for timestamp.  
     - Input: From `Get Post Url`  
     - Output: Triggers scraping node after update completes.  
     - Edge Cases: Row mismatch; write permission errors; timestamp formatting issues.

---

#### 1.2 Scraping Execution

- **Overview:**  
  This block performs the actual scraping of LinkedIn post comments and reactions using the Browserflow integration. It processes the post URL and extracts detailed profile information including comments and reactions.

- **Nodes Involved:**  
  - `Scrape profiles from a linkedin post`

- **Node Details:**

  1. **Scrape profiles from a linkedin post**  
     - Type: Browserflow (Community Node)  
     - Role: Scrapes LinkedIn post comments and reactions using Browserflow API.  
     - Configuration:  
       - Operation: `scrapeProfilesFromPostComments`  
       - Post URL: dynamically assigned from `Get Post Url` node (`={{ $('Get Post Url').item.json.url }}`)  
       - Options enabled: `addComments` = true, `addReactions` = true  
     - Credentials: Requires Browserflow API key configured.  
     - Input: Receives post URL after marking post scraped.  
     - Output: Produces JSON arrays for `comments` and `reactions`.  
     - Edge Cases: API rate limits; invalid or private post URLs; network timeouts; partial data scraping; Browserflow service errors.

---

#### 1.3 Data Processing and Export

- **Overview:**  
  This block splits the scraped data into separate streams for comments and reactions, then appends each to their respective Google Sheets tabs. It finalizes the data pipeline by storing all scraped leads for further analysis or outreach.

- **Nodes Involved:**  
  - `Split Out Reactions`  
  - `Split Out Comments`  
  - `Append to Comments Sheet`  
  - `Append to Reactions Sheet`

- **Node Details:**

  1. **Split Out Reactions**  
     - Type: Split Out  
     - Role: Extracts the `comments` array from the scraped data into individual items for processing.  
     - Configuration: Splits on field `comments` (note: naming here is inverted; see connections).  
     - Input: Output from scraping node.  
     - Output: Feeds into `Append to Reactions Sheet`.  
     - Edge Cases: Empty or null comments field; malformed data structure.  

  2. **Split Out Comments**  
     - Type: Split Out  
     - Role: Extracts the `reactions` array from the scraped data into individual items for processing.  
     - Configuration: Splits on field `reactions`.  
     - Input: Output from scraping node.  
     - Output: Feeds into `Append to Comments Sheet`.  
     - Edge Cases: Empty or null reactions field; malformed data structure.  

  3. **Append to Comments Sheet**  
     - Type: Google Sheets (Append)  
     - Role: Appends individual comment data to the "Comments" tab of the Google Sheets document.  
     - Configuration:  
       - Document ID: `1huUaOdCXtioJgUpJHbK8cm-LLTtw-R7B8N4RKPkbois`  
       - Sheet Name: `gid=0` referring to "Comments" tab.  
       - Columns: name, likes, comment, image_url (rendered as image formula), replies, tagline, relation, linkedin_url.  
     - Input: From `Split Out Comments` node.  
     - Output: Workflow ends here for this branch.  
     - Edge Cases: Google Sheets API limits; formula rendering issues; missing fields in input data.  

  4. **Append to Reactions Sheet**  
     - Type: Google Sheets (Append)  
     - Role: Appends individual reaction data to the "Reactions" tab of the Google Sheets document.  
     - Configuration:  
       - Document ID: Same as above  
       - Sheet Name: `385427295` (gid for "Reactions" tab)  
       - Columns: name, image_url (image formula), tagline, relation, linkedin_url, reaction_type.  
     - Input: From `Split Out Reactions` node.  
     - Output: Workflow ends here for this branch.  
     - Edge Cases: Same as comment append node; missing reaction type; API write errors.

---

### 3. Summary Table

| Node Name                   | Node Type                | Functional Role                           | Input Node(s)              | Output Node(s)                  | Sticky Note                                                                                                              |
|-----------------------------|--------------------------|-----------------------------------------|----------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger           | Start workflow manually                  | -                          | Get Post Url                   |                                                                                                                          |
| Get Post Url                | Google Sheets            | Fetch LinkedIn post URL to scrape       | When clicking ‘Execute workflow’ | Update row in sheet            | ## Fetch posts to scrape                                                                                                 |
| Update row in sheet         | Google Sheets            | Mark post as scraped with timestamp     | Get Post Url                | Scrape profiles from a linkedin post |                                                                                                                          |
| Scrape profiles from a linkedin post | Browserflow              | Scrape comments and reactions from post | Update row in sheet         | Split Out Reactions, Split Out Comments | ##  Scrape the post                                                                                                      |
| Split Out Reactions         | Split Out                | Split comments array into individual items | Scrape profiles from a linkedin post | Append to Comments Sheet       | ##  Store the results                                                                                                    |
| Split Out Comments          | Split Out                | Split reactions array into individual items | Scrape profiles from a linkedin post | Append to Reactions Sheet      | ##  Store the results                                                                                                    |
| Append to Comments Sheet    | Google Sheets            | Append comment data to "Comments" sheet | Split Out Comments          | -                              |                                                                                                                          |
| Append to Reactions Sheet   | Google Sheets            | Append reaction data to "Reactions" sheet | Split Out Reactions         | -                              |                                                                                                                          |
| Sticky Note                 | Sticky Note              | Overview and setup instructions         | -                          | -                              | ## Scrape LinkedIn Post Comments & Reactions  \n\nManually run the workflow to scrape any LinkedIn post.  \nIt reads your **Posts** sheet, marks each URL as processed, scrapes comments + reactions via Browserflow, then appends everything into the **Comments** and **Reactions** tabs.\n\n### How to set up\n1. Install **Browserflow for LinkedIn** (community node)  \n2. Add your Browserflow API key  \n3. Copy the Google Sheets template (Posts, Comments, Reactions)  \n4. Add post URLs to the **Posts** tab  \n5. Click **Execute workflow**\n\n### Customization (optional)\n- Filter reaction types or keywords  \n- Schedule with Cron for automatic scraping\n- Use additional Browserflow actions to scrape profiles or send invites |
| Sticky Note1                | Sticky Note              | Label for scraping block                | -                          | -                              | ##  Scrape the post                                                                                                      |
| Sticky Note2                | Sticky Note              | Label for data storing block            | -                          | -                              | ##  Store the results                                                                                                    |
| Sticky Note3                | Sticky Note              | Label for data fetching block           | -                          | -                              | ##  Fetch posts to scrape                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Google Sheets Node to Fetch Post URL**  
   - Name: `Get Post Url`  
   - Type: Google Sheets  
   - Operation: Read (default)  
   - Credentials: Configure your Google Sheets OAuth2 credentials.  
   - Document ID: Use your Google Sheets document ID where posts are stored.  
   - Sheet Name: Use the GID or sheet name for the "Posts" sheet (e.g., `1217532733`).  
   - Filters: Optionally filter rows by `scraped_at` column to get unprocessed posts.  
   - Connect output from Manual Trigger node.

3. **Create Google Sheets Node to Update Post Row**  
   - Name: `Update row in sheet`  
   - Type: Google Sheets  
   - Operation: Update  
   - Credentials: Use the same Google Sheets OAuth2 credentials.  
   - Document ID and Sheet Name: Same as `Get Post Url` node.  
   - Mapping:  
     - `row_number`: Map from `Get Post Url` → `row_number`.  
     - `scraped_at`: Use expression `$now` to update the scrape timestamp.  
   - Connect input from `Get Post Url` node output.

4. **Create Browserflow Node to Scrape LinkedIn Post**  
   - Name: `Scrape profiles from a linkedin post`  
   - Type: Browserflow  
   - Credentials: Setup Browserflow API key credentials.  
   - Operation: `scrapeProfilesFromPostComments`  
   - Parameters:  
     - `postUrl`: Set expression to pull from `Get Post Url` node → `url` field.  
     - Enable `addComments` and `addReactions` options.  
   - Connect input from `Update row in sheet` node output.

5. **Create Split Out Node for Reactions**  
   - Name: `Split Out Reactions`  
   - Type: Split Out  
   - Parameters: Set `fieldToSplitOut` to `comments` (note: due to data structure, comments are split here).  
   - Connect input from Browserflow node output.

6. **Create Split Out Node for Comments**  
   - Name: `Split Out Comments`  
   - Type: Split Out  
   - Parameters: Set `fieldToSplitOut` to `reactions` (similarly, reactions are split here).  
   - Connect input from Browserflow node output.

7. **Create Google Sheets Node to Append to Comments Sheet**  
   - Name: `Append to Comments Sheet`  
   - Type: Google Sheets  
   - Operation: Append  
   - Credentials: Use Google Sheets OAuth2 credentials.  
   - Document ID: Same document as before.  
   - Sheet Name: Use GID or name for "Comments" tab (e.g., `gid=0`).  
   - Columns to map:  
     - `name`, `likes`, `comment`, `img_url` (use IMAGE formula), `replies`, `tagline`, `relation`, `linkedin_url`.  
   - Connect input from `Split Out Comments`.

8. **Create Google Sheets Node to Append to Reactions Sheet**  
   - Name: `Append to Reactions Sheet`  
   - Type: Google Sheets  
   - Operation: Append  
   - Credentials: Google Sheets OAuth2 credentials.  
   - Document ID: Same as above.  
   - Sheet Name: Use GID for "Reactions" tab (e.g., `385427295`).  
   - Columns to map:  
     - `name`, `img_url` (image formula), `tagline`, `relation`, `linkedin_url`, `reaction_type`.  
   - Connect input from `Split Out Reactions`.

9. **Add Sticky Notes** (optional for clarity)  
   - Create sticky notes with content describing each block: input, scraping, data storage, and overview instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                                |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Manually run the workflow to scrape any LinkedIn post. It reads your **Posts** sheet, marks each URL as processed, scrapes comments + reactions via Browserflow, then appends everything into the **Comments** and **Reactions** tabs.                                                                                                                                         | Sticky Note at workflow start                                                                                                 |
| Setup requires installing **Browserflow for LinkedIn** community node and adding your Browserflow API key. Copy the Google Sheets template containing "Posts", "Comments", and "Reactions" tabs. Add post URLs to the **Posts** tab before running the workflow.                                                                                                            | Sticky Note at workflow start                                                                                                 |
| Customization options include filtering reaction types or keywords, scheduling with Cron for automatic scraping, or adding more Browserflow actions like scraping profiles or sending LinkedIn invites automatically.                                                                                                                                                            | Sticky Note at workflow start                                                                                                 |
| Google Sheets columns for Comments include rendering profile images using the `IMAGE()` formula to display images directly in the sheet. Similar approach is used for Reactions.                                                                                                                                                                                              | Mapping configuration in Append nodes                                                                                        |
| The split field names in Split Out nodes appear inverted (`Split Out Reactions` splits `comments` and vice versa), indicating possibly swapped naming or data structure nuances; verify data output from Browserflow to adjust accordingly.                                                                                                                                      | Node configuration detail                                                                                                    |
| Browserflow API usage is subject to rate limits and may experience failures due to private posts or network issues; implement error handling or retries if needed for production use.                                                                                                                                                                                          | General operational note                                                                                                     |

---

This documentation provides a detailed, stepwise understanding and reproduction plan for the LinkedIn post scraping workflow integrating Browserflow and Google Sheets within n8n. It covers the full data flow, node configurations, and usage notes to support both advanced users and automation agents.