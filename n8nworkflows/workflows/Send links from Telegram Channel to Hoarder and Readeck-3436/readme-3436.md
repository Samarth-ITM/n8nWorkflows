Send links from Telegram Channel to Hoarder and Readeck

https://n8nworkflows.xyz/workflows/send-links-from-telegram-channel-to-hoarder-and-readeck-3436


# Send links from Telegram Channel to Hoarder and Readeck

### 1. Workflow Overview

This n8n workflow automates the process of synchronizing links posted in a personal Telegram channel with two external services: Hoarder and Readeck. It periodically checks the Telegram channel for new hyperlinks, filters out any links already saved in these services, and then posts the new links to each service’s respective API.

The workflow is logically divided into the following blocks:

- **1.1 Telegram Channel Polling**  
  Periodically triggers and fetches updates from the Telegram channel using the Telegram Bot API.

- **1.2 Link Extraction and Filtering**  
  Extracts only valid hyperlinks from the Telegram messages and filters out non-link content.

- **1.3 Readeck Synchronization Block**  
  Retrieves saved links from Readeck, filters out already saved links, and posts new links to Readeck.

- **1.4 Hoarder Synchronization Block**  
  Retrieves saved links from Hoarder, filters out already saved links, and posts new links to Hoarder.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Channel Polling

- **Overview:**  
  This block periodically triggers the workflow and fetches updates (messages) from the specified Telegram channel using the Telegram Bot API.

- **Nodes Involved:**  
  - Schedule Trigger  
  - channel_items_tg  
  - channel_links_tg

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow on a fixed interval (every hour).  
    - Configuration: Interval set to trigger every 1 hour.  
    - Inputs: None (trigger node).  
    - Outputs: Triggers `channel_items_tg`.  
    - Edge cases: If the server is down during trigger time, the trigger will miss that cycle. Also, if interval is changed to a very short period, Telegram API rate limits might be hit.

  - **channel_items_tg**  
    - Type: HTTP Request  
    - Role: Calls the Telegram API endpoint `getUpdates` to retrieve recent updates/messages.  
    - Configuration:  
      - URL: `https://api.telegram.org/bot{{TG_SHERLINK_BOT_TOKEN}}/getUpdates` (uses Telegram bot token from environment variables).  
      - Method: GET  
      - Query Parameters: None specifically set, default behavior to fetch updates.  
    - Inputs: Trigger from Schedule Trigger.  
    - Outputs: JSON response with updates to `channel_links_tg`.  
    - Edge cases:  
      - Network or Telegram API downtime causes failure.  
      - Telegram API may paginate updates or require offset handling (not explicitly handled here).  
      - If bot token is invalid or revoked, authentication error occurs.

  - **channel_links_tg**  
    - Type: Code (JavaScript)  
    - Role: Filters the Telegram updates to extract only channel posts from the target channel that contain hyperlinks.  
    - Configuration/Logic:  
      - Uses environment variable `TG_SHERLINK_ID` for Telegram channel ID.  
      - Processes the `result` array from previous node.  
      - Filters messages by matching channel ID and checks if text contains a hyperlink via regex `/https?:\/\/[^\s]+/`.  
      - Outputs items with fields: `id` (message_id), and `url` (text content).  
    - Inputs: Output from `channel_items_tg`.  
    - Outputs: An array of JSON items representing Telegram links, passed to the next block.  
    - Edge cases:  
      - If no updates or no matching messages, outputs empty array.  
      - If text does not contain a URL, message is excluded.  
      - Regex may fail if URLs are malformed or complex.

---

#### 2.2 Link Extraction and Filtering

- **Overview:**  
  This block splits the workflow to handle Hoarder and Readeck independently, each receiving the Telegram links to compare against their saved links.

- **Nodes Involved:**  
  - get_links_rd  
  - get_links_hd  
  - Split Out

- **Node Details:**

  - **get_links_rd**  
    - Type: HTTP Request  
    - Role: Fetches all saved bookmarks from Readeck API.  
    - Configuration:  
      - URL: `{{READECK_SERVER}}/api/bookmarks`  
      - Method: GET  
      - Headers: `accept: application/json`, `authorization: Bearer {{READECK_API_KEY}}`.  
    - Inputs: Output from `channel_links_tg`.  
    - Outputs: List of saved bookmarks to `saved_links_rd`.  
    - Edge cases:  
      - API credentials invalid or expired causes auth error.  
      - Network/API downtime.  
      - Large bookmark lists might cause performance issues.

  - **get_links_hd**  
    - Type: HTTP Request  
    - Role: Fetches all saved bookmarks from Hoarder API.  
    - Configuration:  
      - URL: `{{HOARDER_SERVER}}/api/v1/bookmarks`  
      - Method: GET  
      - Headers: `Authorization: Bearer {{HOARDER_API_KEY}}`.  
    - Inputs: Output from `channel_links_tg`.  
    - Outputs: List of saved bookmarks to `saved_links_hd`.  
    - Edge cases:  
      - Same as `get_links_rd`.

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits array of bookmarks from Hoarder’s saved links into individual items.  
    - Configuration: Field to split out is `bookmarks`.  
    - Inputs: Output from `get_links_hd`.  
    - Outputs: Individual bookmark items to `saved_links_hd`.  
    - Edge cases:  
      - If `bookmarks` field is undefined or empty, no output items.

---

#### 2.3 Readeck Synchronization Block

- **Overview:**  
  Filters Telegram links to exclude those already saved in Readeck and then posts new links to Readeck.

- **Nodes Involved:**  
  - saved_links_rd  
  - not_saved_links_rd  
  - save_link_rd

- **Node Details:**

  - **saved_links_rd**  
    - Type: Set  
    - Role: Simplifies and maps Readeck’s fetched bookmarks to a format with `id` and `url` properties for filtering.  
    - Configuration: Maps `id` and `url` fields from API response items.  
    - Inputs: From `get_links_rd`.  
    - Outputs: To `not_saved_links_rd`.  
    - Edge cases:  
      - If bookmarks have inconsistent structure, mapping might fail.

  - **not_saved_links_rd**  
    - Type: Code (JavaScript)  
    - Role: Filters Telegram channel links to only those not already saved in Readeck.  
    - Configuration:  
      - Uses input from `channel_links_tg` and saved links from `saved_links_rd`.  
      - Compares URLs to exclude duplicates.  
    - Inputs: From `channel_links_tg` and `saved_links_rd`.  
    - Outputs: Filtered new links to `save_link_rd`.  
    - Edge cases:  
      - URL normalization is not handled; URLs with minor differences may cause duplicates.  
      - Empty inputs lead to empty outputs.

  - **save_link_rd**  
    - Type: HTTP Request  
    - Role: Posts new link to Readeck’s API to save it as a bookmark.  
    - Configuration:  
      - URL: `{{READECK_SERVER}}/api/bookmarks`  
      - Method: POST  
      - Body: JSON with `url` field from filtered links.  
      - Headers: `accept: application/json`, `authorization: Bearer {{READECK_API_KEY}}`.  
    - Inputs: From `not_saved_links_rd`.  
    - Outputs: None (endpoint call only).  
    - Edge cases:  
      - API errors (validation, rate limits, downtime).  
      - Duplicate link race conditions if run concurrently.

---

#### 2.4 Hoarder Synchronization Block

- **Overview:**  
  Filters Telegram links to exclude those already saved in Hoarder and then posts new links to Hoarder.

- **Nodes Involved:**  
  - saved_links_hd  
  - not_saved_links_hd  
  - save_link_hd

- **Node Details:**

  - **saved_links_hd**  
    - Type: Set  
    - Role: Extracts URLs from Hoarder’s saved bookmarks to prepare for filtering.  
    - Configuration: Maps `content.url` from API response items to `content.url` property.  
    - Inputs: From `Split Out`.  
    - Outputs: To `not_saved_links_hd`.  
    - Edge cases:  
      - If `content.url` missing or malformed, filtering may fail.

  - **not_saved_links_hd**  
    - Type: Code (JavaScript)  
    - Role: Filters Telegram channel links to only those not already saved in Hoarder.  
    - Configuration:  
      - Uses input from `channel_links_tg` and saved links from `saved_links_hd`.  
      - Compares URLs to exclude duplicates.  
    - Inputs: From `channel_links_tg` and `saved_links_hd`.  
    - Outputs: Filtered new links to `save_link_hd`.  
    - Edge cases:  
      - Same URL normalization caveats as Readeck block.

  - **save_link_hd**  
    - Type: HTTP Request  
    - Role: Posts new link to Hoarder’s API to save it as a bookmark.  
    - Configuration:  
      - URL: `{{HOARDER_SERVER}}/api/v1/bookmarks`  
      - Method: POST  
      - Body: JSON with fields `type: link` and `url`.  
      - Headers: `Authorization: Bearer {{HOARDER_API_KEY}}`.  
    - Inputs: From `not_saved_links_hd`.  
    - Outputs: None (endpoint call only).  
    - Edge cases:  
      - Same API error scenarios as Readeck.

---

### 3. Summary Table

| Node Name         | Node Type           | Functional Role                                  | Input Node(s)                | Output Node(s)              | Sticky Note                             |
|-------------------|---------------------|-------------------------------------------------|------------------------------|-----------------------------|---------------------------------------|
| Schedule Trigger  | Schedule Trigger     | Periodically triggers the workflow               | None                         | channel_items_tg             | ## Telegram                           |
| channel_items_tg  | HTTP Request        | Fetches updates from Telegram Bot API             | Schedule Trigger             | channel_links_tg             | ## Telegram                           |
| channel_links_tg  | Code                | Extracts and filters messages with hyperlinks     | channel_items_tg             | get_links_rd, get_links_hd   | ## Telegram                           |
| get_links_rd      | HTTP Request        | Fetches saved bookmarks from Readeck API          | channel_links_tg             | saved_links_rd               | ## Readeck                           |
| saved_links_rd    | Set                 | Maps Readeck bookmarks for filtering               | get_links_rd                 | not_saved_links_rd           | ## Readeck                           |
| not_saved_links_rd| Code                | Filters out links already saved in Readeck         | saved_links_rd, channel_links_tg | save_link_rd             | ## Readeck                           |
| save_link_rd      | HTTP Request        | Posts new links to Readeck API                      | not_saved_links_rd           | None                        | ## Readeck                           |
| get_links_hd      | HTTP Request        | Fetches saved bookmarks from Hoarder API           | channel_links_tg             | Split Out                   | ## Hoarder                          |
| Split Out         | Split Out            | Splits Hoarder bookmarks array into individual items| get_links_hd                 | saved_links_hd              | ## Hoarder                          |
| saved_links_hd    | Set                 | Maps Hoarder bookmarks for filtering                | Split Out                   | not_saved_links_hd           | ## Hoarder                          |
| not_saved_links_hd| Code                | Filters out links already saved in Hoarder          | saved_links_hd, channel_links_tg | save_link_hd             | ## Hoarder                          |
| save_link_hd      | HTTP Request        | Posts new links to Hoarder API                       | not_saved_links_hd           | None                        | ## Hoarder                          |
| Split Out         | Split Out            | Splits Hoarder bookmarks array into individual items| get_links_hd                 | saved_links_hd              | ## Hoarder                          |
| Sticky Note       | Sticky Note          | Label for Readeck block                              | None                        | None                        | ## Readeck                          |
| Sticky Note1      | Sticky Note          | Label for Telegram block                             | None                        | None                        | ## Telegram                        |
| Sticky Note2      | Sticky Note          | Label for Hoarder block                              | None                        | None                        | ## Hoarder                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Schedule Trigger`  
   - Set to trigger every 1 hour (Interval: 1 hour).

3. **Add an HTTP Request node:**  
   - Name: `channel_items_tg`  
   - Method: GET  
   - URL: `https://api.telegram.org/bot{{$env.TG_SHERLINK_BOT_TOKEN}}/getUpdates`  
   - No query parameters required.  
   - Connect `Schedule Trigger` output to this node.

4. **Add a Code node:**  
   - Name: `channel_links_tg`  
   - Language: JavaScript  
   - Code Logic:  
     - Parse `$node["channel_items_tg"].json["result"]`.  
     - Filter for messages with `channel_post.chat.id` matching `parseInt($env.TG_SHERLINK_ID,10)`.  
     - Extract messages containing hyperlinks matching regex `/https?:\/\/[^\s]+/`.  
     - Output items with `id` (message_id) and `url` (text).  
   - Connect `channel_items_tg` output to this node.

5. **Add two HTTP Request nodes in parallel to fetch saved bookmarks:**

   - **Readeck bookmarks:**  
     - Name: `get_links_rd`  
     - Method: GET  
     - URL: `{{$env.READECK_SERVER}}/api/bookmarks`  
     - Headers:  
       - `accept: application/json`  
       - `authorization: Bearer {{$env.READECK_API_KEY}}`  
     - Connect `channel_links_tg` output to this node.

   - **Hoarder bookmarks:**  
     - Name: `get_links_hd`  
     - Method: GET  
     - URL: `{{$env.HOARDER_SERVER}}/api/v1/bookmarks`  
     - Headers:  
       - `Authorization: Bearer {{$env.HOARDER_API_KEY}}`  
     - Connect `channel_links_tg` output to this node.

6. **Add a Split Out node:**  
   - Name: `Split Out`  
   - Field to split out: `bookmarks`  
   - Connect output of `get_links_hd` to this node.

7. **Add two Set nodes to normalize saved links:**

   - **Readeck Set node:**  
     - Name: `saved_links_rd`  
     - Assignments:  
       - `id` = `={{ $json.id }}`  
       - `url` = `={{ $json.url }}`  
     - Connect `get_links_rd` output to this node.

   - **Hoarder Set node:**  
     - Name: `saved_links_hd`  
     - Assignments:  
       - `content.url` = `={{ $json.content.url }}`  
     - Connect `Split Out` output to this node.

8. **Add two Code nodes to filter out already saved links:**

   - **Readeck filter:**  
     - Name: `not_saved_links_rd`  
     - JavaScript code:  
       - Compare Telegram links (`channel_links_tg`) and Readeck saved links (`saved_links_rd`) by URL.  
       - Output only links not present in saved links.  
     - Connect both `channel_links_tg` and `saved_links_rd` to this node (use multiple inputs or expression referencing).

   - **Hoarder filter:**  
     - Name: `not_saved_links_hd`  
     - JavaScript code:  
       - Compare Telegram links (`channel_links_tg`) and Hoarder saved links (`saved_links_hd`) by URL.  
       - Output only links not present in saved links.  
     - Connect both `channel_links_tg` and `saved_links_hd` to this node.

9. **Add two HTTP Request nodes to save new links:**

   - **Readeck save link:**  
     - Name: `save_link_rd`  
     - Method: POST  
     - URL: `{{$env.READECK_SERVER}}/api/bookmarks`  
     - Headers:  
       - `accept: application/json`  
       - `authorization: Bearer {{$env.READECK_API_KEY}}`  
     - Body Parameters (JSON):  
       - `url`: `={{ $json.url }}`  
     - Connect `not_saved_links_rd` output to this node.

   - **Hoarder save link:**  
     - Name: `save_link_hd`  
     - Method: POST  
     - URL: `{{$env.HOARDER_SERVER}}/api/v1/bookmarks`  
     - Headers:  
       - `Authorization: Bearer {{$env.HOARDER_API_KEY}}`  
     - Body Parameters (JSON):  
       - `type`: `"link"`  
       - `url`: `={{ $json.url }}`  
     - Connect `not_saved_links_hd` output to this node.

10. **Add Sticky Notes for clarity:**  
    - Label blocks: Telegram, Readeck, Hoarder.

11. **Configure environment variables in n8n runtime or docker-compose:**  
    - `TG_SHERLINK_BOT_TOKEN`  
    - `TG_SHERLINK_ID`  
    - `READECK_SERVER`  
    - `READECK_API_KEY`  
    - `HOARDER_SERVER`  
    - `HOARDER_API_KEY`

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                            |
|----------------------------------------------------------------------------------------------|--------------------------------------------|
| The workflow relies on environment variables configured externally, typically in `.env` file. | Declared in `docker-compose.yml` for n8n. |
| Telegram Bot API `getUpdates` method requires careful offset handling for production usage.  | See Telegram API docs for `getUpdates`.    |
| URL matching uses regex `/https?:\/\/[^\s]+/` which may not cover all edge cases for URLs.   | Consider improving regex for URL validation.|
| API tokens must have appropriate permissions on Readeck and Hoarder servers.                  | Ensure API keys are valid and not expired. |
| Workflow created with n8n version 1.85.4.                                                    | Version-specific features may vary.        |
| For further reading on n8n HTTP Request node and environment variables, see n8n documentation. | https://docs.n8n.io/nodes/n8n-nodes-base.httpRequest/ |

---

This document provides a detailed, structured understanding of the workflow’s logic, node configurations, dependencies, and reproduction steps, enabling advanced users and AI agents to maintain, extend, or automate the workflow confidently.