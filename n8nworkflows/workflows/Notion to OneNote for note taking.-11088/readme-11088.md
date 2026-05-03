Notion to OneNote for note taking.

https://n8nworkflows.xyz/workflows/notion-to-onenote-for-note-taking--11088


# Notion to OneNote for note taking.

---

### 1. Workflow Overview

This workflow automates the synchronization of note-taking content from Notion to OneNote by creating or updating OneNote files stored on Nextcloud. It is designed to facilitate seamless note management between Notion pages and OneNote notebooks, leveraging Redis for concurrency locking to prevent duplicate processing.

The workflow is divided into the following logical blocks:

- **1.1 Input Reception and Locking:** Receives webhook notifications of changes in Notion, checks for processing locks via Redis to avoid redundant updates.
- **1.2 Notion Page Retrieval and Validation:** Fetches the updated Notion page data, filters out irrelevant updates, and switches logic based on the presence or absence of note content.
- **1.3 OneNote File Mapping and Existence Check:** Maps Notion page metadata to OneNote file paths and URLs, then checks if the corresponding OneNote file exists on Nextcloud.
- **1.4 OneNote File Creation or Update:** Depending on file existence, either copies a template OneNote file as a new note or proceeds to update the existing note, updating the Notion page with the OneNote URL.
- **1.5 Redis Lock Management:** Sets or renews Redis locks post-successful updates to prevent race conditions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Locking

- **Overview:**  
  This block receives webhook events from Notion and checks Redis to determine if the Notion page is currently locked (being processed). It prevents duplicate or concurrent processing of the same Notion page.

- **Nodes Involved:**  
  - Webhook  
  - Redis (get)  
  - Filter  
  - Sticky Note (visual comment)

- **Node Details:**

  - **Webhook**  
    - Type: Webhook (HTTP POST listener)  
    - Configuration: Listens for POST requests at path `09aff775-83ef-404f-9b05-1bc806f4dbcd`  
    - Key Expressions: Uses `body.entity.id` from webhook JSON payload as Notion page identifier  
    - Output: Passes webhook JSON to Redis node  
    - Errors: Potential webhook misconfiguration or authentication failure if external service not connected properly

  - **Redis (get)**  
    - Type: Redis key-value get operation  
    - Configuration: Retrieves key `lock_{{ $json.body.entity.id }}` to check if page is locked  
    - Input: Receives JSON from webhook node  
    - Output: Passes Redis value `item` to Filter node  
    - Credentials: Redis account configured  
    - Edge Cases: Redis connection failures, missing keys (interpreted as no lock)  

  - **Filter**  
    - Type: Filter node  
    - Configuration: Passes workflow only if Redis value does not equal `"Update"` (i.e., not locked for update)  
    - Input: Redis node output  
    - Output: Continues only if page is not locked  
    - Edge Cases: Expression failure if Redis returns unexpected data type

  - **Sticky Note**  
    - Purpose: Visual note to remind to check if the Notion page ID is locked  
    - No functional role  

---

#### 2.2 Notion Page Retrieval and Validation

- **Overview:**  
  Retrieves the full Notion page data using the Notion API and filters for relevant updates by checking if the `property_notes` field is empty or not.

- **Nodes Involved:**  
  - Get Notion page  
  - Switch  

- **Node Details:**

  - **Get Notion page**  
    - Type: Notion API node (databasePage get)  
    - Configuration: Uses Notion page ID from webhook payload (`body.entity.id`) to retrieve page details  
    - Credentials: Notion API account  
    - Output: Notion page JSON data  
    - Edge Cases: API authentication failure, invalid page ID, rate limits

  - **Switch**  
    - Type: Switch node  
    - Configuration: Checks if `property_notes` property of Notion page is empty  
    - Logic: If empty, routes differently (though only one route is defined here)  
    - Edge Cases: Missing or null property causes routing ambiguity

---

#### 2.3 OneNote File Mapping and Existence Check

- **Overview:**  
  Maps Notion page name and subject to OneNote file paths and URLs on Nextcloud, then performs a PROPFIND HTTP request to check if the OneNote note file already exists.

- **Nodes Involved:**  
  - Mapping (Code node)  
  - HTTP Request1 (PROPFIND on Nextcloud)  
  - Match titiles (Code node)  

- **Node Details:**

  - **Mapping**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Extracts `name` and `property_subject` from Notion page data  
      - Constructs Nextcloud destination path for OneNote file as `OneNote_Notes/{subject}/{name}.one`  
      - Builds a local URL for accessing the file via a local server proxy  
      - Outputs properties: `destination`, `url`, `name`, `subject`  
    - Input: Notion page JSON  
    - Output: JSON with mapping info  
    - Edge Cases: Missing properties cause undefined paths

  - **HTTP Request1**  
    - Type: HTTP Request node  
    - Configuration:  
      - PROPFIND method against Nextcloud URL for the subject folder  
      - Sends XML body requesting DAV properties (e.g. displayname)  
      - Authenticated via Nextcloud credentials  
      - Retries on failure enabled  
    - Input: Mapping output (subject)  
    - Output: XML response containing file listings  
    - Edge Cases: Connection timeout, auth failure, XML parsing issues

  - **Match titiles**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Parses PROPFIND XML to extract hrefs of files  
      - Filters for `.one` files directly under the subject folder (no nested paths)  
      - Normalizes and compares file basenames to the Notion page name  
      - Outputs boolean `match` indicating if target OneNote file exists  
    - Input: HTTP Request1 XML data and mapping data  
    - Output: JSON with boolean `match`  
    - Edge Cases: Malformed XML, decoding errors, character normalization mismatches

---

#### 2.4 OneNote File Creation or Update

- **Overview:**  
  Based on the existence check, either copies a template OneNote file to create a new note or proceeds to update the existing note, then updates the Notion page with the OneNote file URL.

- **Nodes Involved:**  
  - If  
  - HTTP Request (COPY template to destination)  
  - Update task in Notion  
  - Update task in Notion1  
  - Aggregate  
  - Store (Redis set)  
  - Sticky Note1 (visual comment)

- **Node Details:**

  - **If**  
    - Type: If node  
    - Configuration: Checks if `match` is false (file does not exist)  
    - Routes:  
      - True branch: File does not exist → copy template  
      - False branch: File exists → aggregate and update existing note  
    - Edge Cases: Boolean expression failure

  - **HTTP Request**  
    - Type: HTTP Request node  
    - Configuration:  
      - COPY method from Nextcloud template file `empty.one` to the mapped destination path  
      - Sets `Destination` header to target OneNote file path  
      - Authenticated with Nextcloud credentials  
      - Retries on fail enabled  
    - Input: If node (true branch)  
    - Output: Passes to Update task in Notion  
    - Edge Cases: Copy failure, permission denied, network issues

  - **Update task in Notion**  
    - Type: HTTP Request node (PATCH)  
    - Configuration:  
      - Updates Notion page property `Notes.url` with the OneNote URL from Mapping node  
      - Uses Notion API credentials  
    - Input: HTTP Request (copy) output  
    - Output: Passes to Store (Redis set)  
    - Edge Cases: API authentication failure, invalid property update

  - **Aggregate**  
    - Type: Aggregate node (empty configuration)  
    - Purpose: Aggregates data for the false branch (file exists) path before updating Notion  
    - Input: If node (false branch)  
    - Output: Passes to Update task in Notion1  
    - Edge Cases: Misconfiguration if no fields specified

  - **Update task in Notion1**  
    - Type: HTTP Request node (PATCH)  
    - Same as Update task in Notion, updates the Notion page URL property  
    - Input: Aggregate node output  
    - Output: Passes to Store (Redis set)  
    - Edge Cases: Same as above

  - **Store**  
    - Type: Redis key-value set operation  
    - Configuration:  
      - Sets key `lock_{{ $json.id }}` with value `"Updated"`  
      - TTL (time to live) of 80 seconds to temporarily lock Notion page post-update  
    - Input: Both Update task in Notion and Update task in Notion1 outputs converge here  
    - Credentials: Redis account  
    - Edge Cases: Redis connection issues, key conflicts

  - **Sticky Note1**  
    - Purpose: Large visual note, empty content, possibly a placeholder or design artifact  

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                          | Input Node(s)             | Output Node(s)          | Sticky Note                                |
|---------------------|---------------------|----------------------------------------|---------------------------|-------------------------|--------------------------------------------|
| Webhook             | Webhook             | Receives Notion webhook events         | —                         | Redis                   |                                            |
| Redis               | Redis               | Checks if page is locked in Redis      | Webhook                   | Filter                  | Check if id not locked.                     |
| Filter              | Filter              | Filters out locked pages                | Redis                     | Get Notion page         | Check if id not locked.                     |
| Get Notion page     | Notion              | Retrieves Notion page details           | Filter                    | Switch                  |                                            |
| Switch              | Switch              | Routes based on presence of notes       | Get Notion page           | Mapping                 |                                            |
| Mapping             | Code                | Maps Notion page to OneNote file paths | Switch                    | HTTP Request1           |                                            |
| HTTP Request1       | HTTP Request        | PROPFIND to check OneNote file existence| Mapping                   | Match titiles           |                                            |
| Match titiles       | Code                | Matches OneNote file names to Notion   | HTTP Request1             | If                      |                                            |
| If                  | If                  | Chooses file creation or update path   | Match titiles             | HTTP Request / Aggregate |                                            |
| HTTP Request        | HTTP Request        | Copies OneNote template to create file | If (true branch)          | Update task in Notion   |                                            |
| Update task in Notion| HTTP Request       | Updates Notion page with OneNote URL   | HTTP Request              | Store                   |                                            |
| Aggregate           | Aggregate           | Prepares data for update on existing file | If (false branch)       | Update task in Notion1  |                                            |
| Update task in Notion1| HTTP Request      | Updates Notion page with OneNote URL   | Aggregate                 | Store                   |                                            |
| Store               | Redis               | Sets lock key in Redis post-update     | Update task in Notion, Update task in Notion1 | —           |                                            |
| Sticky Note         | Sticky Note         | Visual reminder about locking           | —                         | —                       | Check if id not locked.                     |
| Sticky Note1        | Sticky Note         | Large visual note (empty content)       | —                         | —                       |                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Webhook`  
   - Type: Webhook (HTTP POST)  
   - Path: `09aff775-83ef-404f-9b05-1bc806f4dbcd`  
   - Purpose: Receive Notion webhook events  
   - No credentials needed  

2. **Create Redis Node (Get)**  
   - Name: `Redis`  
   - Type: Redis  
   - Operation: `get`  
   - Key: `lock_{{ $json.body.entity.id }}`  
   - Property Name: `item`  
   - Credentials: Link your Redis account  
   - Connect input from `Webhook` output  

3. **Create Filter Node**  
   - Name: `Filter`  
   - Type: Filter  
   - Condition: Pass only if `{{$json.item}}` not equals `"Update"`  
   - Connect input from `Redis` output  

4. **Create Notion Node (Get Notion page)**  
   - Name: `Get Notion page`  
   - Type: Notion  
   - Resource: `databasePage`  
   - Operation: `get`  
   - Page ID: `{{$json.body.entity.id}}` (expression)  
   - Credentials: Connect your Notion API account  
   - Connect input from `Filter` output  

5. **Create Switch Node**  
   - Name: `Switch`  
   - Type: Switch  
   - Condition: Check if `{{$json.property_notes}}` is empty (string empty operator)  
   - Connect input from `Get Notion page` output  

6. **Create Code Node (Mapping)**  
   - Name: `Mapping`  
   - Type: Code  
   - JavaScript:  
     ```js
     let output = {};
     const name = $('Get Notion page').first().json.name;
     const sub = $('Get Notion page').first().json.property_subject;
     output.destination = `https://nextcloud.yuvrajsingh.icu/remote.php/dav/files/yuvrajsingh/Documents/Notes/OneNote_Notes/${sub}/${name}.one`;
     output.url = 'http://localhost:3069/open?file=C%3A%5CUsers%5CYuvrajSingh%5CNextcloud%5CDocuments%5CNotes%5COneNote_Notes%5C' + encodeURIComponent(`${sub}\\\\${name}.one`);
     output.name = name;
     output.subject = sub;
     return { json: output };
     ```  
   - Connect input from `Switch` output  

7. **Create HTTP Request Node (PROPFIND)**  
   - Name: `HTTP Request1`  
   - Type: HTTP Request  
   - Method: `PROPFIND`  
   - URL: `https://nextcloud.yuvrajsingh.icu/remote.php/dav/files/yuvrajsingh/Documents/Notes/OneNote_Notes/{{$json.subject}}`  
   - Body (raw XML):  
     ```xml
     <?xml version="1.0" encoding="UTF-8"?>
     <d:propfind xmlns:d="DAV:" xmlns:oc="http://owncloud.org/ns" xmlns:nc="http://nextcloud.org/ns">
       <d:prop>
         <d:displayname/>
       </d:prop>
     </d:propfind>
     ```  
   - Content Type: `application/xml` (raw)  
   - Send Headers: true  
   - Authentication: Predefined credential for Nextcloud  
   - Retry on Fail: enabled  
   - Connect input from `Mapping` output  

8. **Create Code Node (Match titiles)**  
   - Name: `Match titiles`  
   - Type: Code  
   - JavaScript:  
     (Use the provided robust XML parsing and matching code from workflow)  
   - Connect input from `HTTP Request1` output  

9. **Create If Node**  
   - Name: `If`  
   - Type: If  
   - Condition: Check if `{{$json.match}}` not equals `true` (boolean)  
   - Connect input from `Match titiles` output  

10. **Create HTTP Request Node (COPY template)**  
    - Name: `HTTP Request`  
    - Type: HTTP Request  
    - Method: `COPY`  
    - URL: `https://nextcloud.yuvrajsingh.icu/remote.php/dav/files/yuvrajsingh/Documents/Notes/OneNote_Notes/Templates/empty.one`  
    - Headers: `Destination: {{$json.destination}}`  
    - Authentication: Predefined Nextcloud credential  
    - Retry on Fail: enabled  
    - Connect input from `If` node, True branch  

11. **Create HTTP Request Node (Update task in Notion)**  
    - Name: `Update task in Notion`  
    - Type: HTTP Request  
    - Method: `PATCH`  
    - URL: `https://api.notion.com/v1/pages/{{$('Get Notion page').item.json.id}}`  
    - Body Parameters (JSON):  
      - `properties.Notes.url`: `{{$('Mapping').item.json.url}}`  
    - Authentication: Predefined Notion API credential  
    - Connect input from `HTTP Request` output  

12. **Create Aggregate Node**  
    - Name: `Aggregate`  
    - Type: Aggregate  
    - No specific aggregation fields configured (default)  
    - Connect input from `If` node, False branch  

13. **Create HTTP Request Node (Update task in Notion1)**  
    - Name: `Update task in Notion1`  
    - Type: HTTP Request  
    - Same configuration as `Update task in Notion` node  
    - Connect input from `Aggregate` output  

14. **Create Redis Node (Store)**  
    - Name: `Store`  
    - Type: Redis  
    - Operation: `set`  
    - Key: `lock_{{ $json.id }}`  
    - Value: `"Updated"`  
    - TTL: 80 seconds  
    - Expire: true  
    - Credentials: Redis account  
    - Connect input from both `Update task in Notion` and `Update task in Notion1` outputs  

15. **(Optional) Add Sticky Notes**  
    - Add visual Sticky Note nodes with content:  
      - "Check if id not locked." near Redis and Filter nodes  
      - An empty Sticky Note covering broad area as per original layout (optional)  

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                        |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| Workflow integrates Notion pages with OneNote files stored on Nextcloud using webdav API.       | General project context                                |
| Uses Redis keys as locks to prevent duplicate updates during concurrent webhook events.          | Concurrency control mechanism                          |
| Nextcloud PROPFIND method is used to list existing OneNote files to avoid duplication.           | Nextcloud webdav API docs                              |
| OneNote template file `empty.one` is copied to create new note files dynamically.                | Template-based note creation                           |
| Notion page property updated with direct URL to OneNote file for easy access and reference.      | Notion API patch update                                |
| Robust XML parsing and normalization logic to handle filename matching edge cases.               | Custom code for file detection                          |
| Local URL construction uses URL encoding and escaped backslashes for path compatibility.         | Local server proxy for OneNote file access             |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---