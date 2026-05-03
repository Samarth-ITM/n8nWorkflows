Fetch Squarespace Blog & Event Collections to Google Sheets

https://n8nworkflows.xyz/workflows/fetch-squarespace-blog---event-collections-to-google-sheets-3098


# Fetch Squarespace Blog & Event Collections to Google Sheets

### 1. Workflow Overview

This workflow automates the extraction of blog and event collection items from a Squarespace website and stores them into a Google Sheets spreadsheet. It is designed to handle pagination efficiently by fetching 20 items per request until all content is retrieved. The workflow supports both scheduled execution and manual triggering, making it suitable for keeping external content records up to date.

Logical blocks included:

- **1.1 Trigger Input**: Defines how the workflow is started (either scheduled or manual).
- **1.2 Data Retrieval from Squarespace**: Fetches paginated blog/event collection data via HTTP requests.
- **1.3 Data Processing**: Splits the retrieved collection items for individual processing.
- **1.4 Data Output to Google Sheets**: Inserts or updates the processed data into a Google Sheets document.
- **1.5 Informational Notes**: Sticky notes providing user guidance on configuration and setup.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger Input

- **Overview:** This block initiates the workflow either on a schedule or manually via a test trigger.
- **Nodes Involved:**  
  - Schedule Trigger  
  - When clicking ‘Test workflow’

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically starts the workflow at defined intervals.  
    - Configuration: Uses default interval (unspecified in JSON, likely every minute or as per user setup).  
    - Inputs: None  
    - Outputs: Connects to "Fetch Squarespace blog" node.  
    - Edge Cases: Misconfigured schedule may cause unexpected runs or no runs; ensure interval is set as desired.

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual execution for testing or on-demand runs.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Connects to "Fetch Squarespace blog" node.  
    - Edge Cases: None significant; manual trigger depends on user action.

---

#### 1.2 Data Retrieval from Squarespace

- **Overview:** Fetches blog and event collection data from Squarespace using paginated HTTP requests, retrieving 20 items per page until all pages are fetched.
- **Nodes Involved:**  
  - Fetch Squarespace blog

- **Node Details:**

  - **Fetch Squarespace blog**  
    - Type: HTTP Request  
    - Role: Sends GET requests to the Squarespace blog URL to retrieve collection data.  
    - Configuration:  
      - URL set to `https://beyondspace.studio/blog` (user must change to their own Squarespace blog URL).  
      - Pagination enabled with parameters:  
        - `offset` dynamically set from the previous response's `pagination.nextPageOffset`.  
        - `format=json` to request JSON responses.  
      - Pagination completes when `pagination.nextPage` is not true.  
      - Request interval set to 200ms between paginated requests to avoid rate limits.  
    - Key Expressions:  
      - `={{ $response.body.pagination.nextPageOffset }}` for offset parameter.  
      - `={{ $response.body.pagination.nextPage !== true }}` to detect pagination end.  
    - Inputs: From either Schedule Trigger or Manual Trigger.  
    - Outputs: Collection data passed to "Iterate the collection items".  
    - Edge Cases:  
      - Incorrect URL or network issues cause request failures.  
      - Pagination parameters missing or malformed response may break pagination logic.  
      - Rate limiting by Squarespace if requests are too frequent.  
      - Authentication is not required here but if the site is private, this node will fail.

---

#### 1.3 Data Processing

- **Overview:** Splits the array of collection items into individual items for separate processing and insertion into Google Sheets.
- **Nodes Involved:**  
  - Iterate the collection items

- **Node Details:**

  - **Iterate the collection items**  
    - Type: Split Out  
    - Role: Takes the array of `items` from the HTTP response and outputs each item as a separate data entry.  
    - Configuration:  
      - Field to split out: `items` (from the HTTP response JSON).  
    - Inputs: From "Fetch Squarespace blog".  
    - Outputs: Each item sent individually to the Google Sheets node.  
    - Edge Cases:  
      - If `items` field is missing or empty, no data will be processed downstream.  
      - Malformed JSON or unexpected data structure will cause errors.

---

#### 1.4 Data Output to Google Sheets

- **Overview:** Appends or updates each collection item into a Google Sheets spreadsheet, mapping relevant fields and handling duplicates by matching on the `id` column.
- **Nodes Involved:**  
  - Squarespace collection Spreadsheet

- **Node Details:**

  - **Squarespace collection Spreadsheet**  
    - Type: Google Sheets  
    - Role: Inserts or updates rows in a specified Google Sheets document with collection item data.  
    - Configuration:  
      - Operation: `appendOrUpdate` — adds new rows or updates existing rows based on matching `id`.  
      - Document ID: `1yf_RYZGFHpMyOvD3RKGSvIFY2vumvI4474Qm_1t4-jM` (user’s Google Sheet).  
      - Sheet Name: `gid=0` (first sheet/tab).  
      - Columns mapped:  
        - `id` (string)  
        - `tags` (joined string from array)  
        - `title` (string)  
        - `urlId` (string)  
        - `addedOn` (ISO date string, date part only)  
        - `fullUrl` (string)  
        - `assetUrl` (same as fullUrl)  
        - `publishOn` (ISO date string, date part only)  
        - `categories` (joined string from array)  
      - Matching column: `id` to avoid duplicates.  
      - Credentials: Uses OAuth2 Google Sheets API credentials.  
    - Inputs: Individual collection items from "Iterate the collection items".  
    - Outputs: None (end of data flow).  
    - Edge Cases:  
      - Authentication failures if credentials are invalid or expired.  
      - Rate limits or quota exceeded errors from Google Sheets API.  
      - Data type mismatches or invalid date formats could cause insertion errors.  
      - If the spreadsheet or sheet is renamed/moved, document ID or sheet name must be updated.

---

#### 1.5 Informational Notes

- **Overview:** Provides user guidance on configuration and setup.
- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Instructs user to change the HTTP Request URL to their Squarespace blog URL.  
    - Content:  
      ```
      ## Change this node
      Edit the HTTP Request URL to your Squarespace blog URL
      
      eg: https://beyondspace.studio/blog
      ```
    - Position: Near the HTTP Request node for visibility.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Provides a link to a sample Google Sheets template for user reference.  
    - Content:  
      ```
      ## Spreadsheet template
      Clone this spreadsheet as reference
      https://docs.google.com/spreadsheets/d/1HGc7o4mqMY1t9fXT6LBhmZixjJYr0eapSUosXMA9v8E/edit?gid=0#gid=0
      ```
    - Position: Near the Google Sheets node.

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                      | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                         |
|-----------------------------|---------------------|------------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger    | Starts workflow on schedule        | None                        | Fetch Squarespace blog       |                                                                                                   |
| When clicking ‘Test workflow’| Manual Trigger      | Starts workflow manually           | None                        | Fetch Squarespace blog       |                                                                                                   |
| Fetch Squarespace blog      | HTTP Request        | Fetches paginated Squarespace data | Schedule Trigger, Manual Trigger | Iterate the collection items | Change this node: Edit the HTTP Request URL to your Squarespace blog URL (e.g., https://beyondspace.studio/blog) |
| Iterate the collection items| Split Out           | Splits collection array into items | Fetch Squarespace blog       | Squarespace collection Spreadsheet |                                                                                                   |
| Squarespace collection Spreadsheet | Google Sheets     | Appends/updates data in Google Sheets | Iterate the collection items | None                        | Spreadsheet template: Clone this spreadsheet as reference https://docs.google.com/spreadsheets/d/1HGc7o4mqMY1t9fXT6LBhmZixjJYr0eapSUosXMA9v8E/edit?gid=0#gid=0 |
| Sticky Note                | Sticky Note         | User guidance on HTTP Request URL | None                        | None                        | Change this node: Edit the HTTP Request URL to your Squarespace blog URL (e.g., https://beyondspace.studio/blog) |
| Sticky Note1               | Sticky Note         | User guidance on Google Sheets template | None                        | None                        | Spreadsheet template: Clone this spreadsheet as reference https://docs.google.com/spreadsheets/d/1HGc7o4mqMY1t9fXT6LBhmZixjJYr0eapSUosXMA9v8E/edit?gid=0#gid=0 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Add a **Schedule Trigger** node:  
     - Set the interval as desired (e.g., every hour).  
     - No additional parameters needed.

   - Add a **Manual Trigger** node:  
     - Default settings, no parameters.

2. **Add HTTP Request Node to Fetch Squarespace Blog:**

   - Node Type: HTTP Request  
   - Name: `Fetch Squarespace blog`  
   - Method: GET  
   - URL: Set to your Squarespace blog collection URL, e.g., `https://yourdomain.com/blog`  
   - Query Parameters:  
     - `format` = `json`  
     - `offset` = dynamic, set via pagination (see below)  
   - Enable Pagination:  
     - Pagination type: Custom  
     - Parameters:  
       - Name: `offset`  
       - Value: `={{ $response.body.pagination.nextPageOffset }}`  
     - Request interval: 200 ms  
     - Pagination complete when: `={{ $response.body.pagination.nextPage !== true }}`  
   - Connect both Schedule Trigger and Manual Trigger nodes to this node.

3. **Add Split Out Node to Iterate Collection Items:**

   - Node Type: Split Out  
   - Name: `Iterate the collection items`  
   - Field to split out: `items`  
   - Connect output of `Fetch Squarespace blog` to this node.

4. **Add Google Sheets Node to Append or Update Data:**

   - Node Type: Google Sheets  
   - Name: `Squarespace collection Spreadsheet`  
   - Operation: `appendOrUpdate`  
   - Document ID: Use your Google Sheets document ID (e.g., `1yf_RYZGFHpMyOvD3RKGSvIFY2vumvI4474Qm_1t4-jM`)  
   - Sheet Name: Use the sheet/tab name or ID (e.g., `gid=0`)  
   - Columns to map:  
     - `id`: `={{ $json.id }}`  
     - `tags`: `={{ $json.tags.join(", ") }}`  
     - `title`: `={{ $json.title }}`  
     - `urlId`: `={{ $json.urlId }}`  
     - `addedOn`: `={{ new Date($json.addedOn).toISOString().split("T")[0] }}`  
     - `fullUrl`: `={{ $json.fullUrl }}`  
     - `assetUrl`: `={{ $json.fullUrl }}` (same as fullUrl)  
     - `publishOn`: `={{ new Date($json.publishOn).toISOString().split("T")[0] }}`  
     - `categories`: `={{ $json.categories.join(", ") }}`  
   - Matching Columns: `id` (to update existing rows)  
   - Connect output of `Iterate the collection items` to this node.  
   - Set up Google Sheets OAuth2 credentials (Google account with access to the spreadsheet).

5. **Add Sticky Notes for User Guidance (Optional):**

   - Add a Sticky Note near the HTTP Request node with content:  
     ```
     ## Change this node
     Edit the HTTP Request URL to your Squarespace blog URL

     eg: https://yourdomain.com/blog
     ```
   - Add a Sticky Note near the Google Sheets node with content:  
     ```
     ## Spreadsheet template
     Clone this spreadsheet as reference
     https://docs.google.com/spreadsheets/d/1HGc7o4mqMY1t9fXT6LBhmZixjJYr0eapSUosXMA9v8E/edit?gid=0#gid=0
     ```

6. **Connect Nodes:**

   - Connect Schedule Trigger → Fetch Squarespace blog  
   - Connect When clicking ‘Test workflow’ → Fetch Squarespace blog  
   - Connect Fetch Squarespace blog → Iterate the collection items  
   - Connect Iterate the collection items → Squarespace collection Spreadsheet

7. **Activate Workflow:**

   - Save and activate the workflow.  
   - Test manually via the Manual Trigger or wait for scheduled runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                             |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Use [this sample Google Sheets template](https://docs.google.com/spreadsheets/d/1HGc7o4mqMY1t9fXT6LBhmZixjJYr0eapSUosXMA9v8E/edit?gid=0#gid=0) to get started quickly. | Google Sheets setup reference for data insertion.                                                          |
| Check out other n8n templates by the creator: [n8n.io/creators/bangank36](https://n8n.io/creators/bangank36/)           | Additional workflow templates for automation inspiration.                                                  |
| Ensure your Squarespace collection URL is publicly accessible or does not require authentication, or else HTTP requests will fail. | Important for HTTP Request node to successfully fetch data.                                                |
| Pagination is set to request 20 items per page, controlled by Squarespace’s API defaults; adjust if API changes.       | Pagination behavior note.                                                                                   |
| Google Sheets OAuth2 credentials must have permission to edit the target spreadsheet.                                  | Credential setup requirement.                                                                               |

---

This document fully describes the workflow "Fetch Squarespace Blog & Event Collections to Google Sheets," enabling users and automation agents to understand, reproduce, and customize the workflow effectively.