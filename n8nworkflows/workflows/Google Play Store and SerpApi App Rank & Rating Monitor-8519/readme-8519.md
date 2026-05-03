Google Play Store and SerpApi App Rank & Rating Monitor

https://n8nworkflows.xyz/workflows/google-play-store-and-serpapi-app-rank---rating-monitor-8519


# Google Play Store and SerpApi App Rank & Rating Monitor

### 1. Workflow Overview

This workflow automates the monitoring of app ranks and ratings in the Google Play Store based on a list of specified keywords and app titles. It is intended for SEO professionals, app marketers, or developers who want to track how their apps rank for particular search terms on the Google Play Store. The workflow leverages SerpApi’s Google Play Store API to fetch search results, then parses the rank and rating of the target app, and logs these results into Google Sheets for historical tracking and dashboarding.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger:** Initiates the workflow daily at 10 AM UTC.
- **1.2 Input Reception and Loop Setup:** Reads keywords and app titles from a Google Sheet and loops over each row.
- **1.3 Search Execution:** Uses SerpApi to search Google Play Store with each keyword.
- **1.4 Data Parsing:** Extracts the rank and rating for the target app from the search results.
- **1.5 Data Logging:** Appends new results to a historical log sheet and updates a latest-run overview sheet.
- **1.6 Rate Limiting:** Introduces a delay between iterations to comply with Google Sheets API quotas.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  This block schedules the automation to run daily at 10 AM UTC. It is the entry point that triggers the entire workflow.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Sticky Note (Schedule description)

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Runs once per day at 10 AM UTC (hour 10).  
    - Inputs: None  
    - Outputs: Triggers "Get Keywords and Titles to Match" node.  
    - Edge Cases: Misconfigured trigger time or time zone issues can cause missed runs or wrong execution times.

  - **Sticky Note (Schedule)**  
    - Provides a summary of the scheduling configuration and instructions to adjust or trigger manually.

---

#### 2.2 Input Reception and Loop Setup

- **Overview:**  
  This block reads the list of keywords and corresponding app titles to be matched from a Google Sheet, then loops over each row to process them individually.

- **Nodes Involved:**  
  - Get Keywords and Titles to Match (Google Sheets)  
  - Loop Over Keywords (Split In Batches)  
  - Sticky Note (Get Keywords and Titles description)

- **Node Details:**  
  - **Get Keywords and Titles to Match**  
    - Type: Google Sheets Node (Read operation)  
    - Configuration: Reads rows from a configured Google Sheet document and sheet. The sheet must contain columns for keywords and app titles to match.  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Sends all rows fetched to "Loop Over Keywords" node.  
    - Edge Cases: Empty or malformed sheet, incorrect credentials, or missing data may cause errors or empty runs.

  - **Loop Over Keywords**  
    - Type: SplitInBatches  
    - Configuration: Splits input items into batches of 1 (default batch size), effectively looping over each keyword-app title pair sequentially.  
    - Inputs: Rows from Google Sheet  
    - Outputs: For each batch, triggers the search node.  
    - Edge Cases: Large input sets can lead to long run durations. No batch size specified means default 1, which is suitable for rate limiting.

  - **Sticky Note (Get Keywords and Titles)**  
    - Describes the purpose of reading keywords and titles for matching and looping over rows.

---

#### 2.3 Search Execution

- **Overview:**  
  Executes a search query on SerpApi’s Google Play Store API using the current keyword to retrieve search results.

- **Nodes Involved:**  
  - Search Google Play (SerpApi)  
  - Sticky Note (Search Google Play description)

- **Node Details:**  
  - **Search Google Play**  
    - Type: SerpApi Node  
    - Configuration:  
      - Query parameter `q` set to current keyword from the loop.  
      - Operation set to "google_play" to target Google Play Store API.  
      - No additional request options configured.  
    - Inputs: One keyword-app title pair from loop  
    - Outputs: Raw JSON with organic search results including apps and ratings.  
    - Credential: SerpApi API credentials required.  
    - Edge Cases: API key invalid/expired, rate limiting by SerpApi, network errors, or empty results.

  - **Sticky Note (Search Google Play)**  
    - Explains that this node searches the keyword in SerpApi's Google Play Store API.

---

#### 2.4 Data Parsing

- **Overview:**  
  Processes SerpApi’s JSON response to find the target app’s rank and rating based on the app title to match. If the app is not found, rank and rating are set to "N/A".

- **Nodes Involved:**  
  - Parse Rank & Rating for Target App (Code Node)  
  - Sticky Note (Parse Rank & Rating description)

- **Node Details:**  
  - **Parse Rank & Rating for Target App**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Searches the `organic_results[0].items` array for an item whose title includes the target app title.  
      - If found, rank is index + 1; rating is extracted from the matched item.  
      - If not found, rank and rating are both "N/A".  
      - Returns an object with `rank` and `rating`.  
    - Inputs: JSON from SerpApi search results and loop context for app title to match.  
    - Outputs: Parsed rank and rating.  
    - Edge Cases: Missing or malformed JSON, no organic results, no items array, or title mismatch.

  - **Sticky Note (Parse Rank & Rating)**  
    - Describes the parsing logic and fallback to "N/A" when app title is not found.

---

#### 2.5 Data Logging

- **Overview:**  
  Logs the parsed rank and rating results into two Google Sheets: one is an append-only historical log, and the other updates the latest run overview by matching on a composite key.

- **Nodes Involved:**  
  - Update Rank & Rating Log (Google Sheets Append)  
  - Update Latest Run (Google Sheets Update)  
  - Sticky Note (Update Google Sheet detailed instructions)

- **Node Details:**  
  - **Update Rank & Rating Log**  
    - Type: Google Sheets Node (Append operation)  
    - Configuration:  
      - Appends a new row with columns: rank, rating, keyword, searched_at (ISO timestamp), and app_title_to_match.  
      - Document ID and sheet name must be set to the user’s Google Sheet.  
    - Inputs: Parsed rank and rating with context from loop and search nodes.  
    - Outputs: Triggers "Update Latest Run" node.  
    - Edge Cases: Google Sheets API quota exceeded, permission errors, or misconfigured sheet.

  - **Update Latest Run**  
    - Type: Google Sheets Node (Update operation)  
    - Configuration:  
      - Updates existing rows matching on `title_keyword_pair` column.  
      - Updates rank, rating, keyword, searched_at, and app_title_to_match fields.  
      - Document ID and sheet name must be set accordingly.  
    - Inputs: Data from "Parse Rank & Rating" and loop context.  
    - Outputs: Triggers "Wait" node for rate limiting.  
    - Edge Cases: Matching row not found, permission issues, or API quota limits.

  - **Sticky Note (Update Google Sheet)**  
    - Provides detailed instructions on setting up Google Sheets nodes, including field mappings and matching expressions.

---

#### 2.6 Rate Limiting

- **Overview:**  
  Implements a delay of 4 seconds between processing each keyword-app pair to avoid hitting Google Sheets API quota limits.

- **Nodes Involved:**  
  - Wait  
  - Sticky Note (Delay description)

- **Node Details:**  
  - **Wait**  
    - Type: Wait  
    - Configuration: 4 seconds delay.  
    - Inputs: Triggered after updating the latest run sheet.  
    - Outputs: Continues looping to next keyword-app pair.  
    - Edge Cases: Excessive delay may slow workflow unnecessarily; insufficient delay may trigger API quota errors.

  - **Sticky Note (Delay)**  
    - Explains the reason for the delay and suggests adjusting or removing it based on quota limits.

---

### 3. Summary Table

| Node Name                     | Node Type              | Functional Role                         | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                                 |
|-------------------------------|------------------------|---------------------------------------|--------------------------|----------------------------|-------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger       | Starts workflow daily at 10 AM UTC    | None                     | Get Keywords and Titles to Match | Configured to run at 10 AM UTC every day. Adjust as needed or trigger it manually.                           |
| Sticky Note                   | Sticky Note            | Schedule info                         | None                     | None                       | Configured to run at 10 AM UTC every day. Adjust as needed or trigger it manually.                           |
| Get Keywords and Titles to Match | Google Sheets          | Reads keyword-app title pairs         | Schedule Trigger          | Loop Over Keywords          | Reads your Google Sheet to fetch your keywords and app titles to match. Then loops over each row.           |
| Loop Over Keywords            | SplitInBatches         | Iterates over each keyword-title row  | Get Keywords and Titles to Match | Search Google Play           | Reads your Google Sheet to fetch your keywords and app titles to match. Then loops over each row.           |
| Search Google Play            | SerpApi                | Searches keyword in Google Play Store | Loop Over Keywords        | Parse Rank & Rating for Target App | Searches keyword in SerpApi's Google Play Store API.                                                        |
| Sticky Note1                  | Sticky Note            | Search Google Play explanation        | None                     | None                       | Searches keyword in SerpApi's Google Play Store API.                                                        |
| Parse Rank & Rating for Target App | Code                   | Parses rank & rating from results     | Search Google Play        | Update Rank & Rating Log    | Code to find and parse target app's rank and rating. Assigns "N/A" if an app title is not found in results. |
| Sticky Note2                  | Sticky Note            | Parse Rank & Rating explanation       | None                     | None                       | Code to find and parse target app's rank and rating. Assigns "N/A" if an app title is not found in results. |
| Update Rank & Rating Log      | Google Sheets          | Appends results to historical log     | Parse Rank & Rating for Target App | Update Latest Run           | Logs results to results log and updates last run overview sheet. Add your own Google Sheet here.            |
| Update Latest Run             | Google Sheets          | Updates latest run overview            | Update Rank & Rating Log  | Wait                       | Logs results to results log and updates last run overview sheet. Add your own Google Sheet here.            |
| Sticky Note4                  | Sticky Note            | Update Google Sheet instructions       | None                     | None                       | Logs results to results log and updates last run overview sheet. Add your own Google Sheet here.            |
| Wait                         | Wait                   | Delays 4 seconds to respect API quotas | Update Latest Run         | Loop Over Keywords (continue) | Wait 4 seconds before going to next row to not hit Google Sheets API's per minute quota limit.              |
| Sticky Note5                  | Sticky Note            | Delay explanation                     | None                     | None                       | Wait 4 seconds before going to next row to not hit Google Sheets API's per minute quota limit.              |
| Sticky Note6                  | Sticky Note            | General project description and usage | None                     | None                       | Comprehensive project description with setup instructions and useful links.                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Add a **Schedule Trigger** node.  
   - Set it to trigger daily at 10 AM UTC (set "triggerAtHour" to 10).  
   - This node has no inputs. Output will trigger the next node.

2. **Add Google Sheets Node to Read Keywords and Titles**  
   - Add a **Google Sheets** node.  
   - Operation: Read rows from a Google Sheet document.  
   - Configure with your Google Sheets credentials.  
   - Set Document ID and Sheet Name to the sheet containing keywords and app titles.  
   - This node receives input from the Schedule Trigger node.

3. **Add SplitInBatches Node to Loop Over Each Keyword-Title Pair**  
   - Add a **SplitInBatches** node.  
   - No special batch size needed (defaults to 1).  
   - Connect input from the Google Sheets node above.  
   - This node outputs one item per execution, looping over all rows.

4. **Add SerpApi Node to Search Google Play Store**  
   - Add a **SerpApi** node.  
   - Operation: Set to `google_play`.  
   - Query (`q`): Set expression to current item's keyword: `={{ $json.keyword }}`.  
   - Add your SerpApi credentials (API key).  
   - Connect input from SplitInBatches node.

5. **Add Code Node to Parse Rank & Rating**  
   - Add a **Code** node (JavaScript).  
   - Paste the following logic (adapt as needed):  
     ```javascript
     index = $input.first().json.organic_results[0].items.findIndex(obj => obj.title.includes($('Loop Over Keywords').first().json.app_title_to_match));

     if (index >= 0) {
       rank = index + 1;
       rating = $input.first().json.organic_results[0].items[index].rating;
     } else {
       rank = "N/A";
       rating = "N/A";
     }

     return { rank, rating };
     ```  
   - Connect input from SerpApi node.

6. **Add Google Sheets Node to Append to Historical Log**  
   - Add a **Google Sheets** node.  
   - Operation: Append rows.  
   - Configure with your Google Sheets credentials.  
   - Set Document ID and Sheet Name to your historical log sheet.  
   - Define columns and map values:  
     - `rank`: `={{ $json.rank }}`  
     - `rating`: `={{ $json.rating }}`  
     - `keyword`: `={{ $('Search Google Play').item.json.search_parameters.q }}`  
     - `searched_at`: `={{ $now.toISO() }}`  
     - `app_title_to_match`: `={{ $('Loop Over Keywords').item.json.app_title_to_match }}`  
   - Connect input from Code node.

7. **Add Google Sheets Node to Update Latest Run Overview**  
   - Add a **Google Sheets** node.  
   - Operation: Update rows.  
   - Configure with your Google Sheets credentials.  
   - Set Document ID and Sheet Name to your latest run overview sheet.  
   - Use matching column `title_keyword_pair` with expression:  
     `={{ $('Loop Over Keywords').item.json.title_keyword_pair }}`  
   - Map columns:  
     - `rank`: from Code node parsed rank  
     - `rating`: from Code node parsed rating  
     - `keyword`: from loop keyword  
     - `searched_at`: current timestamp `={{ $now.toISO() }}`  
     - `app_title_to_match`: from loop  
   - Connect input from Append node.

8. **Add Wait Node for Rate Limiting**  
   - Add a **Wait** node.  
   - Set wait time to 4 seconds.  
   - Connect input from Update Latest Run node.

9. **Connect Wait Node Back to SplitInBatches Node**  
   - Connect Wait node output to SplitInBatches to continue looping over next keyword-title pair.

10. **Add Sticky Notes (Optional)**  
    - Add Sticky Note nodes with descriptions for clarity at various workflow points as per the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow is designed for daily automated monitoring of Google Play Store app ranks and ratings for SEO purposes. It uses SerpApi’s Google Play Store API and logs data in Google Sheets for historical tracking and dashboarding.                                                                                                                                                                                                                                                                                                                      | Project overview                                                                                          |
| To use this workflow, create a free SerpApi account at https://serpapi.com/ and add your API key credentials in n8n.                                                                                                                                                                                                                                                                                                                                                                                                                                      | https://serpapi.com/                                                                                      |
| Connect your Google Sheets account(s) to n8n with appropriate OAuth2 credentials for reading and writing data.                                                                                                                                                                                                                                                                                                                                                                                                                                           | https://n8n.io/integrations/google-sheets/                                                               |
| Copy the example Google Sheet provided here to your Google Drive and configure the nodes to point to your copy: https://docs.google.com/spreadsheets/d/1DiP6Zhe17tEblzKevtbPqIygH3dpPCW-NAprxup0VqA/edit?gid=1750873622#gid=1750873622                                                                                                                                                                                                                                                                                                                         | Example Google Sheet                                                                                        |
| The workflow includes a rate-limiting wait node (4 seconds delay) to avoid exceeding Google Sheets API quota limits; adjust or remove based on your quota.                                                                                                                                                                                                                                                                                                                                                                                             | API quota management                                                                                       |
| Documentation and guides for SerpApi’s Google Play API and n8n node usage can be found here: https://serpapi.com/google-play-api and https://serpapi.com/blog/boost-your-n8n-workflows-with-serpapis-verified-node/                                                                                                                                                                                                                                                                                                                                             | https://serpapi.com/google-play-api, https://serpapi.com/blog/boost-your-n8n-workflows-with-serpapis-verified-node/ |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.