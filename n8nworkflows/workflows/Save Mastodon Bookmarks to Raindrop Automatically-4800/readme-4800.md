Save Mastodon Bookmarks to Raindrop Automatically

https://n8nworkflows.xyz/workflows/save-mastodon-bookmarks-to-raindrop-automatically-4800


# Save Mastodon Bookmarks to Raindrop Automatically

### 1. Workflow Overview

This workflow automates the synchronization of Mastodon bookmarks into Raindrop.io collections. It periodically fetches new bookmarks from a Mastodon instance, filters and validates the data, and saves each bookmark as a Raindrop bookmark for convenient external access and organization. It also maintains pagination state to avoid duplicate processing.

Logical blocks:

- **1.1 Input Reception:** Triggering the workflow manually or on a scheduled interval.
- **1.2 State Initialization:** Retrieving the last processed `min_id` to ensure incremental bookmark fetching.
- **1.3 Mastodon API Request:** Making authenticated requests to Mastodon's bookmarks API with pagination.
- **1.4 Response Validation & Pagination:** Checking for bookmarks in the response, then extracting pagination info from HTTP headers.
- **1.5 Pagination State Update:** Saving the new `min_id` for future incremental requests.
- **1.6 Bookmark Processing:** Splitting the array of bookmarks, filtering valid data, and saving bookmarks into Raindrop using two possible data paths depending on data completeness.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block provides two entry points: manual trigger for testing and a scheduled trigger that runs the workflow every minute.

**Nodes Involved:**  
- Allows manual testing of the workflow  
- Triggers the workflow on a set interval

**Node Details:**

- **Allows manual testing of the workflow**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow manually for testing or on-demand execution.  
  - Configuration: Default manual trigger with no parameters.  
  - Inputs: None  
  - Outputs: Connected to “Retrieves the last processed min_id to avoid duplicates”  
  - Edge Cases: None significant, just manual invocation.

- **Triggers the workflow on a set interval**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow every minute to keep bookmarks in sync.  
  - Configuration: Interval set to every 1 minute.  
  - Inputs: None  
  - Outputs: Connected to “Retrieves the last processed min_id to avoid duplicates”  
  - Edge Cases: Network or system downtime may cause missed runs.

---

#### 1.2 State Initialization

**Overview:**  
Reads the last saved `min_id` from global static data to fetch only new bookmarks and avoid duplicates.

**Nodes Involved:**  
- Retrieves the last processed min_id to avoid duplicates

**Node Details:**

- **Retrieves the last processed min_id to avoid duplicates**  
  - Type: Code (JavaScript)  
  - Role: Reads global static data to get the last `min_id` used for pagination; initializes it to 0 if not present.  
  - Configuration: Checks if `min_id` exists in static data; if not, sets to 0, then returns it in output JSON.  
  - Key Expression: Accesses `$getWorkflowStaticData('global')`  
  - Inputs: From manual or schedule trigger  
  - Outputs: To Mastodon API request node  
  - Edge Cases: Static data corruption or resets cause re-fetching from the start (`min_id=0`).

---

#### 1.3 Mastodon API Request

**Overview:**  
Makes an authenticated HTTP GET request to Mastodon’s `/api/v1/bookmarks` endpoint with the `min_id` query parameter to fetch bookmarks newer than the last processed.

**Nodes Involved:**  
- Makes authenticated request to Mastodon’s bookmarks API

**Node Details:**

- **Makes authenticated request to Mastodon’s bookmarks API**  
  - Type: HTTP Request  
  - Role: Fetches bookmarks from Mastodon server using bearer token authentication.  
  - Configuration:  
    - URL: `{VOTRE SERVEUR MASTODON}/api/v1/bookmarks` (user must replace `{VOTRE SERVEUR MASTODON}` with actual Mastodon server URL)  
    - Authentication: HTTP Bearer Token (credential stored under “Bearer Auth account”)  
    - Query Parameter: `min_id` dynamically set from previous node output to fetch only new bookmarks  
    - Response: Set to receive full HTTP response for header parsing  
  - Inputs: `min_id` from static data node  
  - Outputs: To the “Ensures the response has bookmarks before continuing” node  
  - Edge Cases:  
    - Authentication failure (invalid token)  
    - Network timeouts or server errors  
    - Empty or malformed responses

---

#### 1.4 Response Validation & Pagination

**Overview:**  
Verifies that the API response contains bookmarks before continuing, then extracts pagination data (`min_id` for next fetch) from HTTP response headers.

**Nodes Involved:**  
- Ensures the response has bookmarks before continuing  
- Extracts pagination data (like next min_id) from HTTP headers

**Node Details:**

- **Ensures the response has bookmarks before continuing**  
  - Type: If  
  - Role: Checks if the `body` array in the HTTP response contains at least one bookmark.  
  - Configuration: Condition checks if `$json.body.length > 0`  
  - Inputs: HTTP response node  
  - Outputs: If true, proceeds to pagination extraction; if false, workflow ends here (no bookmarks).  
  - Edge Cases: Empty bookmark list halts processing.

- **Extracts pagination data (like next min_id) from HTTP headers**  
  - Type: Code (JavaScript)  
  - Role: Parses the HTTP `Link` header to extract query parameters like `min_id` for pagination.  
  - Configuration:  
    - Parses header string with regex to find key-value pairs  
    - Adds extracted keys to JSON for downstream nodes  
  - Inputs: HTTP response with headers  
  - Outputs: Two branches — splitting bookmarks and saving new `min_id`  
  - Edge Cases:  
    - Missing or malformed `Link` header  
    - Regex parsing errors  
    - Pagination parameters not present (no next page)

---

#### 1.5 Pagination State Update

**Overview:**  
Updates the stored `min_id` with the new value extracted from headers so future runs fetch only new bookmarks.

**Nodes Involved:**  
- Saves the new min_id to workflow static data

**Node Details:**

- **Saves the new min_id to workflow static data**  
  - Type: Code (JavaScript)  
  - Role: Writes the new `min_id` value into global static data for persistent storage.  
  - Configuration: Updates `workflowStaticData.min_id` with the new `min_id` from previous node.  
  - Inputs: Pagination extraction node  
  - Outputs: None (end of this branch)  
  - Edge Cases: Static data write failures or concurrency issues.

---

#### 1.6 Bookmark Processing

**Overview:**  
Splits the array of bookmarks into individual items, filters out invalid or incomplete bookmarks, and saves valid bookmarks into Raindrop.io. Uses two different Raindrop nodes depending on the availability of card metadata.

**Nodes Involved:**  
- Splits API response into individual bookmark items  
- Filters out invalid or incomplete bookmark data  
- Saves valid bookmark using post metadata (e.g., title, card.url)  
- Saves bookmark using alternate fields if card data is missing

**Node Details:**

- **Splits API response into individual bookmark items**  
  - Type: Split Out  
  - Role: Converts the array of bookmarks in `body` to individual workflow items for processing.  
  - Configuration: Splits on the `body` field of the HTTP response JSON.  
  - Inputs: Pagination extraction node  
  - Outputs: To the filter node  
  - Edge Cases: Empty array results in no outputs.

- **Filters out invalid or incomplete bookmark data**  
  - Type: If  
  - Role: Checks if the bookmark item has a non-empty `card.url` field to determine validity.  
  - Configuration: Condition checks if `$json.card.url` is not empty.  
  - Inputs: Split bookmark items  
  - Outputs:  
    - True: to "Saves valid bookmark using post metadata"  
    - False: to "Saves bookmark using alternate fields"  
  - Edge Cases: Missing or malformed bookmark data.

- **Saves valid bookmark using post metadata (e.g., title, card.url)**  
  - Type: Raindrop.io node  
  - Role: Creates a new bookmark in Raindrop using the bookmark’s metadata (`card.url` and `card.title`).  
  - Configuration:  
    - Operation: Create bookmark  
    - Link: From `$json.card.url`  
    - Title: From `$json.card.title`  
    - Collection ID: `-1` (default collection or user-specific)  
    - Option: `pleaseParse` set true to let Raindrop parse metadata  
    - Credential: Raindrop OAuth2 credential named “Raindrop account”  
  - Inputs: Filter node (true branch)  
  - Outputs: None (end)  
  - Edge Cases: Raindrop API errors, invalid URLs.

- **Saves bookmark using alternate fields if card data is missing**  
  - Type: Raindrop.io node  
  - Role: Fallback to save bookmark using the top-level `url` and account username as title if `card` data is missing.  
  - Configuration:  
    - Operation: Create bookmark  
    - Link: From `$json.url`  
    - Title: From `$json.account.username`  
    - Collection ID: `-1`  
    - Option: `pleaseParse` true  
    - Credential: Same Raindrop OAuth2 credential  
  - Inputs: Filter node (false branch)  
  - Outputs: None (end)  
  - Edge Cases: Missing `url` or `account.username`, Raindrop API errors.

---

### 3. Summary Table

| Node Name                                               | Node Type            | Functional Role                                           | Input Node(s)                                   | Output Node(s)                                         | Sticky Note                                                                                                                                                                                                                              |
|---------------------------------------------------------|----------------------|-----------------------------------------------------------|------------------------------------------------|--------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Allows manual testing of the workflow                    | Manual Trigger       | Manual workflow initiation                                | None                                           | Retrieves the last processed min_id to avoid duplicates |                                                                                                                                                                                                                                          |
| Triggers the workflow on a set interval                  | Schedule Trigger     | Scheduled workflow initiation every minute                | None                                           | Retrieves the last processed min_id to avoid duplicates |                                                                                                                                                                                                                                          |
| Retrieves the last processed min_id to avoid duplicates | Code                 | Reads stored `min_id` from static data for incremental fetch | Allows manual testing of the workflow, Triggers the workflow on a set interval | Makes authenticated request to Mastodon’s bookmarks API |                                                                                                                                                                                                                                          |
| Makes authenticated request to Mastodon’s bookmarks API | HTTP Request         | Fetch bookmarks from Mastodon using bearer token          | Retrieves the last processed min_id to avoid duplicates | Ensures the response has bookmarks before continuing    |                                                                                                                                                                                                                                          |
| Ensures the response has bookmarks before continuing     | If                   | Validates response has bookmarks before processing        | Makes authenticated request to Mastodon’s bookmarks API | Extracts pagination data (like next min_id) from HTTP headers |                                                                                                                                                                                                                                          |
| Extracts pagination data (like next min_id) from HTTP headers | Code                 | Parses HTTP headers to get pagination parameters           | Ensures the response has bookmarks before continuing | Splits API response into individual bookmark items, Saves the new min_id to workflow static data |                                                                                                                                                                                                                                          |
| Saves the new min_id to workflow static data             | Code                 | Updates stored `min_id` for next incremental fetch         | Extracts pagination data (like next min_id) from HTTP headers | None                                                   |                                                                                                                                                                                                                                          |
| Splits API response into individual bookmark items       | Split Out            | Splits bookmarks array into individual items               | Extracts pagination data (like next min_id) from HTTP headers | Filters out invalid or incomplete bookmark data          |                                                                                                                                                                                                                                          |
| Filters out invalid or incomplete bookmark data          | If                   | Filters bookmarks by checking for non-empty `card.url`    | Splits API response into individual bookmark items | Saves valid bookmark using post metadata, Saves bookmark using alternate fields if card data is missing |                                                                                                                                                                                                                                          |
| Saves valid bookmark using post metadata (e.g., title, card.url) | Raindrop.io           | Saves bookmark to Raindrop using card metadata             | Filters out invalid or incomplete bookmark data  | None                                                   |                                                                                                                                                                                                                                          |
| Saves bookmark using alternate fields if card data is missing | Raindrop.io           | Saves bookmark to Raindrop using fallback data             | Filters out invalid or incomplete bookmark data  | None                                                   |                                                                                                                                                                                                                                          |
| Sticky Note                                              | Sticky Note          | Workflow description and requirements                      | None                                           | None                                                   | ## 🗂️ Sync Mastodon Bookmarks to Raindrop  This workflow fetches your latest bookmarks from Mastodon and saves them to Raindrop.io, allowing you to organize and access saved posts outside of Mastodon.  🔧 Requirements:  A Mastodon access token with permission to read bookmarks  A Raindrop.io OAuth2 credential  Replace {VOTRE SERVEUR MASTODON} with your Mastodon server base URL (e.g., https://mastodon.social)  First run will default to min_id = 0, then it updates to fetch only new bookmarks  🕒 Triggered on schedule or manually  💡 Pagination data is extracted from HTTP headers to support future syncs without duplication. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: "Allows manual testing of the workflow"  
   - Purpose: Manual start for testing.

2. **Create a Schedule Trigger node**  
   - Name: "Triggers the workflow on a set interval"  
   - Purpose: Runs every 1 minute.  
   - Parameters: Set interval to 1 minute.

3. **Create a Code node**  
   - Name: "Retrieves the last processed min_id to avoid duplicates"  
   - Purpose: Read or initialize `min_id` from global static data.  
   - Code:
   ```javascript
   const workflowStaticData = $getWorkflowStaticData('global');
   if (!workflowStaticData.hasOwnProperty('min_id')) {
     workflowStaticData.min_id = 0;
   }
   return [{ min_id: workflowStaticData.min_id }];
   ```
   - Connect both triggers to this node.

4. **Create an HTTP Request node**  
   - Name: "Makes authenticated request to Mastodon’s bookmarks API"  
   - Purpose: Fetch bookmarks from Mastodon.  
   - Parameters:  
     - Method: GET  
     - URL: Replace `{VOTRE SERVEUR MASTODON}` with your Mastodon server URL, e.g., `https://mastodon.social/api/v1/bookmarks`  
     - Authentication: HTTP Bearer Token  
     - Credential: Create and select an HTTP Bearer Auth credential with your Mastodon access token  
     - Query Parameters: Add `min_id` with value `={{ $json.min_id }}` (from previous node)  
     - Response: Enable full response to access headers.

5. **Create an If node**  
   - Name: "Ensures the response has bookmarks before continuing"  
   - Purpose: Proceed only if bookmarks are returned.  
   - Condition: Check if the length of `$json.body` is greater than 0.

6. **Create a Code node**  
   - Name: "Extracts pagination data (like next min_id) from HTTP headers"  
   - Purpose: Parse the `Link` HTTP header to extract `min_id`.  
   - Code:
   ```javascript
   const links = $input.first().json.headers.link;
   let regexp = /(?:\\?|\\&)(?<key>[\\w]+)=(?<value>[\\w+,.-]+)/g;
   let results = links.matchAll(regexp);
   for (let match of results) {
     let key = match.groups.key;
     let value = match.groups.value;
     $input.last().json[key] = value;
   }
   return $input.all();
   ```
   - Connect the If node's true branch to this node.

7. **Create two nodes connected from the pagination extraction node:**

   - **Split Out node**  
     - Name: "Splits API response into individual bookmark items"  
     - Purpose: Split the bookmarks array into individual items.  
     - Field to split out: `body`

   - **Code node**  
     - Name: "Saves the new min_id to workflow static data"  
     - Purpose: Update stored `min_id`.  
     - Code:
     ```javascript
     const workflowStaticData = $getWorkflowStaticData('global');
     workflowStaticData.min_id = $input.first().json.min_id;
     return [{ accessToken: workflowStaticData.min_id }];
     ```

8. **Create an If node**  
   - Name: "Filters out invalid or incomplete bookmark data"  
   - Purpose: Validate if `card.url` exists.  
   - Condition: Check that `$json.card.url` is not empty.

   - Connect the Split Out node output to this node.

9. **Create two Raindrop nodes:**

   - **Node 1:**  
     - Name: "Saves valid bookmark using post metadata (e.g., title, card.url)"  
     - Resource: Bookmark  
     - Operation: Create  
     - Link: `={{ $json.card.url }}`  
     - Title: `={{ $json.card.title }}`  
     - Collection ID: `-1` (default or user collection)  
     - Additional Fields: Set `pleaseParse` to true  
     - Credential: Raindrop OAuth2 (create and connect your Raindrop account credential)  
     - Connect If node’s true branch here.

   - **Node 2:**  
     - Name: "Saves bookmark using alternate fields if card data is missing"  
     - Resource: Bookmark  
     - Operation: Create  
     - Link: `={{ $json.url }}`  
     - Title: `={{ $json.account.username }}`  
     - Collection ID: `-1`  
     - Additional Fields: `pleaseParse` true  
     - Credential: Same Raindrop OAuth2 credential  
     - Connect If node’s false branch here.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| ## 🗂️ Sync Mastodon Bookmarks to Raindrop  This workflow fetches your latest bookmarks from Mastodon and saves them to Raindrop.io, allowing you to organize and access saved posts outside of Mastodon.  🔧 Requirements:  - A Mastodon access token with permission to read bookmarks  - A Raindrop.io OAuth2 credential  - Replace {VOTRE SERVEUR MASTODON} with your Mastodon server base URL (e.g., https://mastodon.social)  - First run will default to min_id = 0, then it updates to fetch only new bookmarks  🕒 Triggered on schedule or manually  💡 Pagination data is extracted from HTTP headers to support future syncs without duplication. | Sticky Note node content from workflow description                                                       |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automated workflow. It respects all content policies and contains no illegal or protected elements. All data handled is legal and public.