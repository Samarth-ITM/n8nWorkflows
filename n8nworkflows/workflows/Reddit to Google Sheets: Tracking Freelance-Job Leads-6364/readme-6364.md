Reddit to Google Sheets: Tracking Freelance/Job Leads

https://n8nworkflows.xyz/workflows/reddit-to-google-sheets--tracking-freelance-job-leads-6364


# Reddit to Google Sheets: Tracking Freelance/Job Leads

### 1. Workflow Overview

This workflow automates the process of tracking freelance or job leads from Reddit posts and comments, saving relevant data into a Google Sheets document for easy management. It targets users such as freelancers or job seekers who want to monitor specific Reddit communities for hiring opportunities.

The workflow is structured into these logical blocks:

- **1.1 Scheduled Reddit Data Retrieval:** Periodically (via a schedule trigger) sends multiple HTTP requests to Reddit APIs to fetch posts from various subreddits or search queries.
- **1.2 Post Metadata Extraction and Splitting:** Processes the raw Reddit API responses to extract useful metadata and splits the aggregated post arrays into individual posts for easier handling.
- **1.3 Comments Retrieval and Filtering:** For each post, retrieves related comments via the Reddit node, then removes moderator comments and filters for those likely related to hiring.
- **1.4 Lead Management:** Fetches existing leads from Google Sheets, filters out duplicates with the new leads, then appends unique new leads to the sheet for tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Reddit Data Retrieval

- **Overview:**  
  This block initiates the workflow on a schedule and performs multiple parallel HTTP requests to Reddit APIs to retrieve posts data from different sources or queries.

- **Nodes Involved:**  
  - Schedule Trigger  
  - HTTP Request (8 instances: HTTP Request, HTTP Request1, HTTP Request2, HTTP Request3, HTTP Request4, HTTP Request5, HTTP Request6, HTTP Request7)

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Trigger node  
    - Role: Starts the workflow on a defined schedule (e.g., every hour/day)  
    - Configuration: Default or user-defined schedule (not explicitly shown)  
    - Inputs: None (trigger only)  
    - Outputs: Connects to eight HTTP Request nodes in parallel  
    - Failure modes: Schedule misconfiguration or workflow disabled  

  - **HTTP Request (multiple instances)**  
    - Type: HTTP Request node  
    - Role: Calls Reddit API endpoints to fetch posts data (likely from different subreddits or search queries)  
    - Configuration: Each node uses distinct Reddit API URLs and parameters (not detailed here)  
    - Inputs: Trigger output from Schedule Trigger  
    - Outputs: Each connects to a corresponding "Extract Post Metadata" node  
    - Failure modes: API rate limits, network errors, invalid credentials, response format changes  

#### 2.2 Post Metadata Extraction and Splitting

- **Overview:**  
  Converts the raw JSON responses from Reddit into structured metadata objects, then splits each array of posts into individual post items for downstream processing.

- **Nodes Involved:**  
  - Extract Post Metadata (8 instances: Extract Post Metadata, Extract Post Metadata1, ..., Extract Post Metadata7)  
  - Seperate array into individual posts (8 instances: Seperate array into individual posts, Seperate array into individual posts1, ..., Seperate array into individual posts7)

- **Node Details:**  

  - **Extract Post Metadata (multiple instances)**  
    - Type: Set node  
    - Role: Extracts and formats necessary post metadata fields (e.g., title, author, URL, timestamp) from the HTTP response  
    - Configuration: Extracts and maps specific JSON properties into flat fields for each post  
    - Inputs: From corresponding HTTP Request node  
    - Outputs: To corresponding SplitOut node  
    - Execution: Configured to run once per incoming data batch  
    - Failure modes: Missing expected JSON paths, empty data fields  

  - **Seperate array into individual posts (multiple instances)**  
    - Type: SplitOut node  
    - Role: Splits arrays of posts into individual items for processing downstream  
    - Configuration: Default splitting on arrays  
    - Inputs: From corresponding Extract Post Metadata node  
    - Outputs: All connect to the Merge node, with each SplitOut connected to a separate input index  
    - Failure modes: Input data not an array, empty arrays  

#### 2.3 Comments Retrieval and Filtering

- **Overview:**  
  This block aggregates all individual posts, retrieves their comments, then filters these comments to isolate those related to hiring and exclude moderator inputs.

- **Nodes Involved:**  
  - Merge  
  - Get many comments FROM multiple POSTS (Reddit node)  
  - Remove Mod Comments (If node)  
  - Hiring-related Comments (Filter node)

- **Node Details:**  

  - **Merge**  
    - Type: Merge node  
    - Role: Combines all streams of individual posts from the split nodes into a single stream for comment processing  
    - Configuration: Default merge mode (likely “Merge by index” or “Append”)  
    - Inputs: From all eight SplitOut nodes  
    - Outputs: To Reddit node for comment retrieval  
    - Failure modes: Data type mismatches, empty inputs  

  - **Get many comments FROM multiple POSTS**  
    - Type: Reddit node  
    - Role: Retrieves multiple comments for each post using Reddit API  
    - Configuration: Configured to get comments for each post ID or URL  
    - Inputs: From Merge node  
    - Outputs: To Remove Mod Comments node  
    - Failure modes: API rate limits, authentication issues, unavailable comments  

  - **Remove Mod Comments**  
    - Type: If node  
    - Role: Filters out comments made by subreddit moderators to avoid noise  
    - Configuration: Conditional expression checking if comment author is a moderator  
    - Inputs: From Reddit comments node  
    - Outputs: True branch to Hiring-related Comments node, false branch ignored or discarded  
    - Failure modes: Incorrect moderator identification, missing author data  

  - **Hiring-related Comments**  
    - Type: Filter node  
    - Role: Further filters comments to keep only those relevant to hiring (keywords or criteria)  
    - Configuration: Filter expressions or rules matching hiring-related keywords  
    - Inputs: From Remove Mod Comments node  
    - Outputs: To "Get present leads" node  
    - Failure modes: Overly strict/lenient filters, missing keywords  

#### 2.4 Lead Management

- **Overview:**  
  Retrieves existing leads from Google Sheets, filters duplicates against the new hires found, and appends only unique new leads back to the sheet.

- **Nodes Involved:**  
  - Get present leads (Google Sheets node)  
  - Filter Unique Leads (Code node)  
  - Add Leads to Google Sheet (Google Sheets node)

- **Node Details:**  

  - **Get present leads**  
    - Type: Google Sheets node  
    - Role: Reads existing leads data from a configured Google Sheet to avoid duplicates  
    - Configuration: Reads specific worksheet and range, configured to always output data  
    - Inputs: From Hiring-related Comments node  
    - Outputs: To Filter Unique Leads node  
    - Failure modes: Google API auth errors, sheet permission issues, empty sheet  

  - **Filter Unique Leads**  
    - Type: Code node (JavaScript)  
    - Role: Compares new leads with existing leads to filter out duplicates based on unique identifiers (e.g., post URL or comment ID)  
    - Configuration: Custom code logic implementing filtering  
    - Inputs: From Get present leads node (existing leads) and Hiring-related Comments node (new leads)  
    - Outputs: To Add Leads to Google Sheet node  
    - Failure modes: Code errors, unexpected data formats  

  - **Add Leads to Google Sheet**  
    - Type: Google Sheets node  
    - Role: Appends filtered unique leads into the Google Sheet for tracking  
    - Configuration: Appends rows to the configured worksheet  
    - Inputs: From Filter Unique Leads node  
    - Outputs: None (workflow end)  
    - Failure modes: Google API quota issues, write permission errors  

---

### 3. Summary Table

| Node Name                          | Node Type            | Functional Role                        | Input Node(s)                                    | Output Node(s)                          | Sticky Note |
|-----------------------------------|----------------------|-------------------------------------|-------------------------------------------------|---------------------------------------|-------------|
| Schedule Trigger                  | Schedule Trigger     | Starts workflow on schedule         | -                                               | HTTP Request, HTTP Request1, ..., HTTP Request7 |             |
| HTTP Request                     | HTTP Request         | Fetch Reddit posts data             | Schedule Trigger                                | Extract Post Metadata                  |             |
| Extract Post Metadata            | Set                  | Extract post metadata from response | HTTP Request                                    | Seperate array into individual posts  |             |
| Seperate array into individual posts | SplitOut          | Split posts array into individual posts | Extract Post Metadata                         | Merge                                 |             |
| HTTP Request1                    | HTTP Request         | Fetch Reddit posts data (source 2) | Schedule Trigger                                | Extract Post Metadata1                 |             |
| Extract Post Metadata1           | Set                  | Extract post metadata from response | HTTP Request1                                   | Seperate array into individual posts1 |             |
| Seperate array into individual posts1 | SplitOut          | Split posts array into individual posts | Extract Post Metadata1                        | Merge                                 |             |
| HTTP Request2                    | HTTP Request         | Fetch Reddit posts data (source 3) | Schedule Trigger                                | Extract Post Metadata2                 |             |
| Extract Post Metadata2           | Set                  | Extract post metadata from response | HTTP Request2                                   | Seperate array into individual posts2 |             |
| Seperate array into individual posts2 | SplitOut          | Split posts array into individual posts | Extract Post Metadata2                        | Merge                                 |             |
| HTTP Request3                    | HTTP Request         | Fetch Reddit posts data (source 4) | Schedule Trigger                                | Extract Post Metadata3                 |             |
| Extract Post Metadata3           | Set                  | Extract post metadata from response | HTTP Request3                                   | Seperate array into individual posts3 |             |
| Seperate array into individual posts3 | SplitOut          | Split posts array into individual posts | Extract Post Metadata3                        | Merge                                 |             |
| HTTP Request4                    | HTTP Request         | Fetch Reddit posts data (source 5) | Schedule Trigger                                | Extract Post Metadata4                 |             |
| Extract Post Metadata4           | Set                  | Extract post metadata from response | HTTP Request4                                   | Seperate array into individual posts4 |             |
| Seperate array into individual posts4 | SplitOut          | Split posts array into individual posts | Extract Post Metadata4                        | Merge                                 |             |
| HTTP Request5                    | HTTP Request         | Fetch Reddit posts data (source 6) | Schedule Trigger                                | Extract Post Metadata5                 |             |
| Extract Post Metadata5           | Set                  | Extract post metadata from response | HTTP Request5                                   | Seperate array into individual posts5 |             |
| Seperate array into individual posts5 | SplitOut          | Split posts array into individual posts | Extract Post Metadata5                        | Merge                                 |             |
| HTTP Request6                    | HTTP Request         | Fetch Reddit posts data (source 7) | Schedule Trigger                                | Extract Post Metadata6                 |             |
| Extract Post Metadata6           | Set                  | Extract post metadata from response | HTTP Request6                                   | Seperate array into individual posts6 |             |
| Seperate array into individual posts6 | SplitOut          | Split posts array into individual posts | Extract Post Metadata6                        | Merge                                 |             |
| HTTP Request7                    | HTTP Request         | Fetch Reddit posts data (source 8) | Schedule Trigger                                | Extract Post Metadata7                 |             |
| Extract Post Metadata7           | Set                  | Extract post metadata from response | HTTP Request7                                   | Seperate array into individual posts7 |             |
| Seperate array into individual posts7 | SplitOut          | Split posts array into individual posts | Extract Post Metadata7                        | Merge                                 |             |
| Merge                           | Merge                | Combine all individual posts streams | All Seperate array into individual posts nodes | Get many comments FROM multiple POSTS |             |
| Get many comments FROM multiple POSTS | Reddit            | Retrieve comments for posts         | Merge                                           | Remove Mod Comments                   |             |
| Remove Mod Comments             | If                   | Filter out moderator comments       | Get many comments FROM multiple POSTS           | Hiring-related Comments               |             |
| Hiring-related Comments         | Filter               | Filter comments related to hiring   | Remove Mod Comments                              | Get present leads                    |             |
| Get present leads              | Google Sheets         | Read existing leads from sheet      | Hiring-related Comments                          | Filter Unique Leads                  |             |
| Filter Unique Leads            | Code                  | Filter out duplicate leads           | Get present leads                               | Add Leads to Google Sheet            |             |
| Add Leads to Google Sheet      | Google Sheets         | Append unique new leads to sheet    | Filter Unique Leads                             | -                                   |             |
| Sticky Note                   | Sticky Note           | No content                         | -                                               | -                                   |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node**  
   - Set the schedule to desired frequency (e.g., every hour or every day).

2. **Create eight HTTP Request nodes (HTTP Request through HTTP Request7)**  
   - Configure each node with a unique Reddit API endpoint URL to fetch posts (e.g., different subreddits, search queries).  
   - Use GET method.  
   - Set any required headers (e.g., User-Agent).  
   - Connect the Schedule Trigger node output to each HTTP Request node input in parallel.

3. **Create eight Set nodes to Extract Post Metadata (Extract Post Metadata through Extract Post Metadata7)**  
   - For each node, map fields from the HTTP Request response JSON to structured fields (e.g., post title, author, permalink, created_utc).  
   - Configure 'Execute Once' to true to process the full response at once.  
   - Connect each HTTP Request node output to its respective Extract Post Metadata node.

4. **Create eight SplitOut nodes (Seperate array into individual posts through Seperate array into individual posts7)**  
   - Configure to split the array field containing posts into individual items.  
   - Connect each Extract Post Metadata node output to its corresponding SplitOut node.

5. **Create a Merge node**  
   - Configure to append inputs, merging all eight SplitOut outputs into a single stream.  
   - Connect all eight SplitOut nodes outputs to the Merge node inputs (each on a separate input index).

6. **Create a Reddit node named 'Get many comments FROM multiple POSTS'**  
   - Configure to retrieve comments for each post using identifiers from the merged stream (e.g., post ID or permalink).  
   - Connect Merge node output to this node input.  
   - Set Reddit credentials and OAuth2 authentication.

7. **Create an If node named 'Remove Mod Comments'**  
   - Set condition to check if comment author is a moderator (e.g., via comment metadata).  
   - Connect Reddit node output to this If node input.  
   - The true branch should be discarded or ignored; the false branch proceeds.

8. **Create a Filter node named 'Hiring-related Comments'**  
   - Configure filter rules to keep only comments containing hiring-related keywords (like "hiring", "job", "freelance", "contract").  
   - Connect the false output of the If node to this Filter node.

9. **Create a Google Sheets node named 'Get present leads'**  
   - Configure to read all existing leads from the target Google Sheet and worksheet.  
   - Set Google Sheets OAuth2 credentials.  
   - Connect Hiring-related Comments node output to this node.

10. **Create a Code node named 'Filter Unique Leads'**  
    - Write JavaScript code to compare new leads against existing leads, filtering out duplicates based on a unique key such as post URL or comment ID.  
    - Connect Get present leads node output and Hiring-related Comments output as inputs to this node (set multiple inputs if needed).

11. **Create a Google Sheets node named 'Add Leads to Google Sheet'**  
    - Configure to append rows to the Google Sheet worksheet for new unique leads.  
    - Connect Filter Unique Leads node output to this node.  
    - Use the same Google Sheets credentials as above.

12. **Validate all connections and credentials before activating the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                |
|------------------------------------------------------------------------------|---------------------------------------------------------------|
| The workflow is designed to track freelance or job leads automatically from Reddit posts and comments. | Workflow description                                           |
| Ensure Reddit API usage complies with Reddit’s API terms and rate limits.    | Reddit API documentation: https://www.reddit.com/dev/api/     |
| Google Sheets credentials require OAuth2 setup with permission for reading and writing sheets. | Google Sheets API documentation: https://developers.google.com/sheets/api |
| Use robust keyword filters in the 'Hiring-related Comments' node to minimize false positives. | Filtering strategy                                             |
| Moderator comments are excluded to reduce noise and irrelevant data.         | Node 'Remove Mod Comments' functionality                       |

---

**Disclaimer:** The provided text is exclusively generated from an automated workflow made with n8n, a tool for integration and automation. This process strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.