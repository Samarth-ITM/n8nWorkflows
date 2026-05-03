Automated Upwork Job Alerts with MongoDB & Slack

https://n8nworkflows.xyz/workflows/automated-upwork-job-alerts-with-mongodb---slack-2834


# Automated Upwork Job Alerts with MongoDB & Slack

### 1. Workflow Overview

This workflow automates the retrieval, filtering, storage, and notification of Upwork job postings. It is designed to run every 20 minutes during working hours (9 AM to 5 PM), fetching job listings from Upwork via the Apify API, removing duplicates by checking against a MongoDB collection, and sending notifications about new jobs to a Slack channel.

**Logical Blocks:**

- **1.1 Schedule and Time Filtering:** Controls when the workflow runs and restricts execution to specified working hours.
- **1.2 Job Retrieval from Upwork:** Queries the Apify Upwork scraper API to fetch job postings based on predefined search URLs.
- **1.3 Duplicate Detection:** Checks MongoDB for existing job entries to avoid duplicates.
- **1.4 Filtering and Storing New Jobs:** Filters out duplicates and inserts only new job postings into MongoDB.
- **1.5 Slack Notification:** Sends Slack messages for newly discovered jobs.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule and Time Filtering

- **Overview:**  
  This block triggers the workflow periodically and ensures it only runs during defined working hours (9 AM to 5 PM).

- **Nodes Involved:**  
  - Schedule Trigger  
  - If Working Hours

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow every 10 minutes (configured interval)  
    - Configuration: Interval set to every 10 minutes (note: description says 20 minutes, but node is set to 10)  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "If Working Hours" node  
    - Edge Cases: If the workflow is paused or n8n is offline, triggers will be missed.  
    - Version: 1.2

  - **If Working Hours**  
    - Type: If Node  
    - Role: Filters execution to only proceed if current hour is between 3 and 15 (UTC assumed)  
    - Configuration:  
      - Condition 1: Hour > 2  
      - Condition 2: Hour < 15  
      - Both conditions combined with AND  
      - Uses `$json.Hour` (assumed to be injected or derived from system time)  
    - Inputs: From Schedule Trigger  
    - Outputs: True branch connects to "Assign parameters" node; false branch stops workflow  
    - Edge Cases: Timezone mismatch may cause incorrect filtering; `$json.Hour` must be correctly set or derived.  
    - Version: 2.2

#### 2.2 Job Retrieval from Upwork

- **Overview:**  
  Fetches job postings from Upwork using the Apify API with predefined search URLs and proxy country code.

- **Nodes Involved:**  
  - Assign parameters  
  - Query For Upwork Job Posts

- **Node Details:**

  - **Assign parameters**  
    - Type: Set Node  
    - Role: Defines parameters for the API call, including Upwork search URLs and proxy country code  
    - Configuration:  
      - `startUrls`: Array of Upwork search URLs with GET method, e.g., for "python" and "java" keywords  
      - `proxyCountryCode`: Set to "FR" (France)  
    - Inputs: From "If Working Hours" (true branch)  
    - Outputs: To "Query For Upwork Job Posts"  
    - Edge Cases: URLs must be valid Upwork search URLs; proxy country code must be supported by Apify  
    - Version: 3.4

  - **Query For Upwork Job Posts**  
    - Type: HTTP Request  
    - Role: Calls Apify API to run the Upwork scraper and retrieve job posts synchronously  
    - Configuration:  
      - Method: POST  
      - URL: Apify endpoint for Upwork scraper dataset items  
      - Body: Sends `startUrls` and `proxyCountryCode` from previous node  
      - Authentication: HTTP Query Auth with token (Apify token)  
    - Inputs: From "Assign parameters"  
    - Outputs: Two outputs:  
      - Main output: To "Find Existing Entries" (for jobs found)  
      - Secondary output: To "Output New Entries" (for fallback or additional processing)  
    - Edge Cases:  
      - API token invalid or expired  
      - API rate limits or downtime  
      - Network timeouts  
      - Unexpected response format  
    - Version: 4.2  
    - Credentials: Requires Apify token configured as HTTP Query Auth with key='token'

#### 2.3 Duplicate Detection

- **Overview:**  
  Checks MongoDB to determine if each fetched job already exists based on title and budget.

- **Nodes Involved:**  
  - Find Existing Entries

- **Node Details:**

  - **Find Existing Entries**  
    - Type: MongoDB Node  
    - Role: Queries MongoDB collection "n8n" to find jobs matching title and budget of each fetched job  
    - Configuration:  
      - Query uses dynamic expressions:  
        ```json
        {
          "title": "{{ $json.title }}",
          "budget": "{{ $json.budget }}"
        }
        ```  
      - Collection: "n8n"  
      - Operation: Find  
    - Inputs: From "Query For Upwork Job Posts"  
    - Outputs: To "Output New Entries"  
    - Edge Cases:  
      - MongoDB connection failure  
      - Query syntax errors if fields missing  
      - Large result sets causing performance issues  
    - Version: 1.1  
    - Credentials: MongoDB connection required

#### 2.4 Filtering and Storing New Jobs

- **Overview:**  
  Compares fetched jobs with existing MongoDB entries, filters out duplicates, and inserts only new jobs into MongoDB.

- **Nodes Involved:**  
  - Output New Entries (Merge)  
  - Add New Entries To MongoDB

- **Node Details:**

  - **Output New Entries**  
    - Type: Merge Node  
    - Role: Combines data from fetched jobs and MongoDB query results, keeping only jobs not found in MongoDB (non-matches)  
    - Configuration:  
      - Mode: Combine  
      - Join Mode: Keep Non Matches (i.e., keep jobs not matched in MongoDB)  
      - Fields to match: "title, budget"  
    - Inputs:  
      - From "Find Existing Entries" (existing jobs)  
      - From "Query For Upwork Job Posts" (fetched jobs)  
    - Outputs: To "Add New Entries To MongoDB" and "Send message in #general"  
    - Edge Cases:  
      - Matching fields must be consistent and non-null  
      - Large datasets may affect performance  
    - Version: 3

  - **Add New Entries To MongoDB**  
    - Type: MongoDB Node  
    - Role: Inserts new job postings into MongoDB collection "n8n"  
    - Configuration:  
      - Operation: Insert  
      - Fields: title, link, paymentType, budget, projectLength, shortBio, skills, publishedDate, normalizedDate, searchUrl  
      - Collection: "n8n"  
    - Inputs: From "Output New Entries"  
    - Outputs: None (end node for DB insertion)  
    - Edge Cases:  
      - MongoDB write failures  
      - Duplicate insertion if concurrency issues occur  
    - Version: 1.1  
    - Credentials: MongoDB connection required

#### 2.5 Slack Notification

- **Overview:**  
  Sends a Slack message to a specified channel for each new job inserted into MongoDB.

- **Nodes Involved:**  
  - Send message in #general

- **Node Details:**

  - **Send message in #general**  
    - Type: Slack Node  
    - Role: Posts a formatted message to Slack channel "#general" with job details  
    - Configuration:  
      - Channel: #general (by name)  
      - Message text includes:  
        - Job Title  
        - Published Date  
        - Link  
        - Payment Type  
        - Budget  
        - Skills  
        - Short Bio  
    - Inputs: From "Output New Entries" (only new jobs)  
    - Outputs: None (terminal node)  
    - Edge Cases:  
      - Slack API token invalid or expired  
      - Channel ID incorrect or missing permissions  
      - Message formatting errors  
    - Version: 2.3  
    - Credentials: Slack API token with chat:write permissions

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                         | Input Node(s)               | Output Node(s)                         | Sticky Note                                                                                   |
|---------------------------|---------------------|---------------------------------------|-----------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger    | Triggers workflow every 10 minutes    | None                        | If Working Hours                      |                                                                                               |
| If Working Hours          | If Node            | Filters execution to working hours    | Schedule Trigger            | Assign parameters                     |                                                                                               |
| Assign parameters         | Set Node           | Sets Upwork URLs and proxy code       | If Working Hours            | Query For Upwork Job Posts            |                                                                                               |
| Query For Upwork Job Posts | HTTP Request       | Calls Apify API to fetch job posts    | Assign parameters           | Find Existing Entries, Output New Entries |                                                                                               |
| Find Existing Entries     | MongoDB Node       | Checks MongoDB for existing jobs      | Query For Upwork Job Posts  | Output New Entries                    |                                                                                               |
| Output New Entries        | Merge Node         | Filters out duplicates, keeps new jobs| Find Existing Entries, Query For Upwork Job Posts | Add New Entries To MongoDB, Send message in #general |                                                                                               |
| Add New Entries To MongoDB | MongoDB Node       | Inserts new jobs into MongoDB          | Output New Entries          | None                                  |                                                                                               |
| Send message in #general  | Slack Node         | Sends Slack notification for new jobs | Output New Entries          | None                                  |                                                                                               |
| Sticky Note               | Sticky Note        | Setup instructions                    | None                        | None                                  | 1. Add MongoDB, Slack credentials 2. Add a query auth credential with key='token' and your Apify token 3. Modify 'Assign parameters' node URLs |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to every 10 minutes (or 20 minutes as desired)  
   - No credentials needed

2. **Create If Node (If Working Hours)**  
   - Type: If  
   - Add two conditions:  
     - `$json.Hour` > 2  
     - `$json.Hour` < 15  
   - Combine with AND  
   - This node filters execution to working hours (adjust hours/timezone as needed)  
   - Connect Schedule Trigger output to this node

3. **Create Set Node (Assign parameters)**  
   - Type: Set  
   - Add two fields:  
     - `startUrls` (type: array) with Upwork search URLs, e.g.:  
       ```json
       [
         {"url": "https://www.upwork.com/nx/search/jobs/?nbs=1&q=python", "method": "GET"},
         {"url": "https://www.upwork.com/nx/search/jobs/?nbs=1&q=java", "method": "GET"}
       ]
       ```  
     - `proxyCountryCode` (string), e.g., "FR"  
   - Connect If Working Hours (true output) to this node

4. **Create HTTP Request Node (Query For Upwork Job Posts)**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/arlusm~upwork-scraper-with-fresh-job-posts/run-sync-get-dataset-items`  
   - Send body with parameters:  
     - `startUrls` from previous node (`={{ $json.startUrls }}`)  
     - `proxyCountryCode` from previous node (`={{ $json.proxyCountryCode }}`)  
   - Authentication: HTTP Query Auth  
     - Credential setup: key = "token", value = your Apify API token  
   - Connect Assign parameters node to this node

5. **Create MongoDB Node (Find Existing Entries)**  
   - Type: MongoDB  
   - Operation: Find  
   - Collection: "n8n"  
   - Query:  
     ```json
     {
       "title": "{{ $json.title }}",
       "budget": "{{ $json.budget }}"
     }
     ```  
   - Connect HTTP Request node output to this node

6. **Create Merge Node (Output New Entries)**  
   - Type: Merge  
   - Mode: Combine  
   - Join Mode: Keep Non Matches  
   - Fields to match: "title, budget"  
   - Connect MongoDB Find node output to first input  
   - Connect HTTP Request node output to second input

7. **Create MongoDB Node (Add New Entries To MongoDB)**  
   - Type: MongoDB  
   - Operation: Insert  
   - Collection: "n8n"  
   - Fields to insert:  
     - title, link, paymentType, budget, projectLength, shortBio, skills, publishedDate, normalizedDate, searchUrl  
   - Connect Merge node output to this node

8. **Create Slack Node (Send message in #general)**  
   - Type: Slack  
   - Channel: #general (by name)  
   - Message Text:  
     ```
     Job Title : {{ $json.title }}
     Published : {{ $json.publishedDate }}
     Link : {{ $json.link }}
     Payment Type: {{ $json.paymentType }}
     Budget: {{ $json.budget }}
     Skills: {{ $json.skills }}
     Bio: {{ $json.shortBio }}
     ```  
   - Credentials: Slack API token with chat:write permission  
   - Connect Merge node output to this node

9. **Add Sticky Note (Optional)**  
   - Add a sticky note with setup instructions:  
     - Add MongoDB and Slack credentials  
     - Add Apify token as HTTP Query Auth credential (key='token')  
     - Modify 'Assign parameters' node URLs as needed

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow requires an active Apify subscription to use the Upwork scraper API.                              | Apify API documentation: https://docs.apify.com/api/v2/acts/arlusm~upwork-scraper-with-fresh-job-posts |
| Modify the 'If Working Hours' node to match your timezone and working hours or remove it to run continuously.   | Timezone considerations are critical for correct filtering                                      |
| Slack API token must have chat:write permission and be authorized for the target channel.                       | Slack API docs: https://api.slack.com/messaging/sending                                         |
| MongoDB connection string must be configured in n8n credentials for MongoDB nodes to work properly.             | MongoDB Atlas or local MongoDB setup                                                            |
| The workflow can be customized by changing search keywords in the 'Assign parameters' node.                     | Add or remove URLs to track different job categories                                            |
| Consider expanding notifications to other platforms like Telegram or Email for enhanced alerts.                 |                                                                                                 |

---

This documentation provides a comprehensive understanding of the "Automated Upwork Job Alerts with MongoDB & Slack" workflow, enabling reproduction, modification, and troubleshooting.