Search and download torrents using transmission-daemon

https://n8nworkflows.xyz/workflows/search-and-download-torrents-using-transmission-daemon-1381


# Search and download torrents using transmission-daemon

### 1. Workflow Overview

This workflow automates the process of searching for movie torrents by name and initiating their download using the Transmission daemon. It is designed for media-center enthusiasts who want to streamline torrent handling through voice commands (e.g., via Google Assistant and IFTTT) or other HTTP POST triggers.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the movie name via a webhook triggered by an external source (e.g., IFTTT).
- **1.2 Torrent Search:** Uses the `torrent-search-api` library to find torrents matching the movie title.
- **1.3 Decision Making:** Checks if torrents were found.
- **1.4 Torrent Download Initiation:** Sends the download request to the Transmission daemon, handling Transmission’s CSRF protection by repeating the request with a session token.
- **1.5 Notification:** Sends Telegram messages to notify the user whether the torrent was found and download started, or not found.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Receives incoming POST requests with a movie title payload to trigger the workflow.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook  
    - Role: Entry point for the workflow, receives HTTP POST requests with raw body enabled.  
    - Configuration:  
      - Path: Unique webhook ID path (e.g., `6be952e8-e30f-4dd7-90b3-bc202ae9f174`)  
      - HTTP Method: POST  
      - Options: Raw body enabled (to parse JSON payload)  
    - Input: External HTTP POST trigger, typically from IFTTT or other integrations.  
    - Output: JSON containing the request body, expected to include a `title` field with the movie name.  
    - Failure modes: Invalid or missing payload, webhook not triggered due to network issues.

#### 2.2 Torrent Search

- **Overview:**  
  Searches for torrent files matching the movie title using enabled torrent providers.

- **Nodes Involved:**  
  - SearchTorrent (Function Item)

- **Node Details:**  
  - **SearchTorrent**  
    - Type: FunctionItem (JavaScript code execution per item)  
    - Role: Executes JavaScript code to use the `torrent-search-api` npm package for searching torrents.  
    - Configuration:  
      - Enables providers: KickassTorrents, Rarbg  
      - Reads movie title from webhook JSON body, trims whitespace  
      - Searches for top 5 torrents in all categories  
      - Attaches found torrents array and a `found` boolean flag to output  
    - Key expressions:  
      - `item.title = $node["Webhook"].json["body"].title.trim()`  
      - `const torrents = await TorrentSearchApi.search(item.title, 'All', 5);`  
      - `item.found = torrents.length > 0`  
    - Input: JSON from Webhook node  
    - Output: JSON containing `title`, `torrents` array, and `found` boolean  
    - Failure modes: Package not installed or misconfigured, no torrents found, network/API errors.

#### 2.3 Decision Making: Torrent Found?

- **Overview:**  
  Determines if any torrents were found and routes workflow accordingly.

- **Nodes Involved:**  
  - IF

- **Node Details:**  
  - **IF**  
    - Type: If  
    - Role: Checks the boolean `found` flag from SearchTorrent node.  
    - Configuration:  
      - Condition: `found == true`  
    - Input: Output of SearchTorrent node  
    - Output:  
      - True branch: Proceed to start the download  
      - False branch: Send Telegram notification that torrent was not found  
    - Failure modes: Expression evaluation failure if `found` is missing.

#### 2.4 Torrent Download Initiation

- **Overview:**  
  Sends a POST request to the Transmission RPC interface to add the torrent and start downloading. Handles Transmission’s CSRF token protection by retrying with updated session ID.

- **Nodes Involved:**  
  - Start download (HTTP Request)  
  - IF2 (If)  
  - Start download new token (HTTP Request)

- **Node Details:**  
  - **Start download**  
    - Type: HTTP Request  
    - Role: Sends the initial torrent-add request to Transmission daemon.  
    - Configuration:  
      - URL: `http://localhost:9091/transmission/rpc`  
      - Method: POST  
      - Authentication: Basic Auth (configured with Transmission credentials)  
      - Headers: Includes a placeholder `X-Transmission-Session-Id` (initially invalid)  
      - Body: JSON specifying method `torrent-add` with arguments: not paused, download directory `/media/FILM/TORRENT`, filename set to the magnet link of the first found torrent  
      - Continue On Fail: Enabled to allow workflow continuation on error  
    - Input: JSON from IF node (true branch)  
    - Output: Success or error with potential CSRF token in error response headers  
    - Failure modes: Authentication errors, network issues, invalid session token causing 409 Conflict response.

  - **IF2**  
    - Type: If  
    - Role: Checks if the response from Start download node is a 409 Conflict error indicating missing or invalid `X-Transmission-Session-Id`.  
    - Configuration:  
      - Condition: `error.statusCode == 409` (string comparison)  
    - Input: Output of Start download node  
    - Output:  
      - True branch: Retry with updated session token  
      - False branch: Proceed to Telegram notification of download start  
    - Failure modes: Missing error property, unexpected error codes.

  - **Start download new token**  
    - Type: HTTP Request  
    - Role: Retries the torrent-add request including the valid `X-Transmission-Session-Id` header obtained from previous error response.  
    - Configuration:  
      - Same URL and method as Start download  
      - Authentication: Basic Auth  
      - Header: Reads session ID dynamically from error response headers of Start download node  
      - Body: Same as Start download node (torrent-add arguments)  
    - Input: True branch of IF2 node  
    - Output: Result of adding torrent with valid session ID  
    - Failure modes: Authentication issues, network failure, invalid session ID retrieval.

#### 2.5 Notification

- **Overview:**  
  Sends Telegram messages informing user about torrent search and download status.

- **Nodes Involved:**  
  - Torrent not found (Telegram)  
  - Telegram1 (Telegram)

- **Node Details:**  
  - **Torrent not found**  
    - Type: Telegram  
    - Role: Sends message to user if no torrent was found.  
    - Configuration:  
      - Text: `"Film {movie title} non trovato."` (Italian for "Movie not found")  
      - Chat ID: Configured with user chat ID or username  
      - Credentials: Telegram bot API credentials  
    - Input: False branch of IF node  
    - Output: Telegram message sent confirmation  
    - Failure modes: Invalid bot credentials, network issues, invalid chat ID.

  - **Telegram1**  
    - Type: Telegram  
    - Role: Sends message confirming that download started.  
    - Configuration:  
      - Text: `"Scarico {movie title}!\nTitolo: {torrent title}"` (Italian for "Downloading {movie title}! Title: {torrent title}")  
      - Chat ID: Configured accordingly  
      - Credentials: Telegram bot API credentials  
    - Input: False branch of IF2 node (successful initial download request) and True branch of Start download new token node (retry success)  
    - Output: Telegram message sent confirmation  
    - Failure modes: Same as above.

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                             | Input Node(s)           | Output Node(s)              | Sticky Note                                                             |
|-----------------------|---------------------|--------------------------------------------|-------------------------|-----------------------------|-------------------------------------------------------------------------|
| Webhook               | Webhook             | Receives POST request with movie title     | External HTTP           | SearchTorrent               |                                                                         |
| SearchTorrent         | FunctionItem        | Searches torrents using torrent-search-api | Webhook                 | IF                          |                                                                         |
| IF                    | If                  | Checks if torrents were found               | SearchTorrent           | Start download, Torrent not found |                                                                         |
| Start download        | HTTP Request        | Initiates torrent download on Transmission | IF (true branch)        | IF2                         | Requires Transmission basic auth credentials; first call to get session token |
| IF2                   | If                  | Checks if Transmission returned 409 error  | Start download          | Start download new token, Telegram1 |                                                                         |
| Start download new token | HTTP Request        | Retries download with valid session token  | IF2 (true branch)       | Telegram1                   | Requires Transmission basic auth credentials                            |
| Torrent not found     | Telegram             | Notifies user torrent not found             | IF (false branch)       |                             | Requires Telegram bot credentials; configured chat ID                  |
| Telegram1             | Telegram             | Notifies user download started              | IF2 (false branch), Start download new token |                             | Requires Telegram bot credentials; configured chat ID                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique string (e.g., `6be952e8-e30f-4dd7-90b3-bc202ae9f174`)  
   - Options: Enable Raw Body to accept JSON payload  
   - This node will receive external requests with JSON body containing a `title` field for the movie.

2. **Create Function Item Node (SearchTorrent)**  
   - Type: Function Item  
   - Connect input from Webhook node  
   - Paste JavaScript code to:  
     - Import and enable providers in `torrent-search-api` (`KickassTorrents` and `Rarbg`)  
     - Extract movie title from webhook payload (`item.title = $node["Webhook"].json["body"].title.trim()`)  
     - Run asynchronous `TorrentSearchApi.search()` for top 5 torrents  
     - Set `item.torrents` to results and `item.found` boolean accordingly  
     - Return the modified item  
   - Ensure that the `torrent-search-api` npm package is installed and accessible via n8n’s external node allowance.

3. **Create If Node (IF)**  
   - Connect input from SearchTorrent node  
   - Condition: Boolean check if `found` equal to `true`  
   - True branch: Proceed to Start download node  
   - False branch: Proceed to Torrent not found node

4. **Create HTTP Request Node (Start download)**  
   - Connect input from IF node (true branch)  
   - Configure:  
     - URL: `http://localhost:9091/transmission/rpc` (adjust if Transmission runs elsewhere)  
     - Method: POST  
     - Authentication: Basic Auth with Transmission credentials (username/password)  
     - Body Parameters (JSON):  
       ```json
       {
         "method": "torrent-add",
         "arguments": {
           "paused": false,
           "download-dir": "/media/FILM/TORRENT",
           "filename": "{{$node[\"SearchTorrent\"].json[\"torrents\"][0][\"magnet\"]}}"
         }
       }
       ```  
     - Header Parameters (JSON):  
       ```json
       {
         "X-Transmission-Session-Id": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
       }
       ```  
       (Use placeholder, this will trigger 409 response to get session ID)  
     - Enable “Continue On Fail” to handle retry logic.

5. **Create If Node (IF2)**  
   - Connect input from Start download node  
   - Condition: Check if error status code equals `"409"` (string match on `error.statusCode`)  
   - True branch: Proceed to Start download new token node  
   - False branch: Proceed to Telegram1 node (download started notification)

6. **Create HTTP Request Node (Start download new token)**  
   - Connect input from IF2 node (true branch)  
   - Configure similarly to Start download node but with dynamic header:  
     - Header Parameters (JSON):  
       ```json
       {
         "X-Transmission-Session-Id": "{{$node[\"Start download\"].json[\"error\"][\"response\"][\"headers\"][\"x-transmission-session-id\"]}}"
       }
       ```  
     - Other parameters same as Start download node  
     - Authentication: Basic Auth with Transmission credentials.

7. **Create Telegram Node (Torrent not found)**  
   - Connect input from IF node (false branch)  
   - Configure:  
     - Text: `Film {{$node["Webhook"].json["body"].title}} non trovato.`  
     - Chat ID: Your Telegram chat ID or username  
     - Credentials: Telegram bot API credentials.

8. **Create Telegram Node (Telegram1)**  
   - Connect input from IF2 node (false branch) and Start download new token node  
   - Configure:  
     - Text: `Scarico {{$node["Webhook"].json["body"].title}}!\nTitolo: {{$node["SearchTorrent"].json["torrents"][0]["title"]}}`  
     - Chat ID: Your Telegram chat ID or username  
     - Credentials: Telegram bot API credentials.

**Additional Setup:**

- **Install `torrent-search-api` npm package:**  
  - Navigate to your n8n directory (where `docker-compose.yaml` is located).  
  - Run `npm i torrent-search-api`.  
  - Ensure your `docker-compose.yaml` mounts all required node modules for this package to work inside the container (as per detailed volume configuration in the description).  
  - Enable external node execution in n8n environment variable:  
    `NODE_FUNCTION_ALLOW_EXTERNAL=torrent-search-api`.

- **Configure Transmission Credentials:**  
  - Create HTTP Basic Auth credentials in n8n with Transmission daemon username and password.  
  - Assign these credentials to both HTTP Request nodes.

- **Configure Telegram Credentials:**  
  - Create Telegram API credentials with your bot token.  
  - Assign credentials to both Telegram nodes.  
  - Set proper chat IDs (can be your user ID obtained from `useridinfobot` or username).

- **External Trigger:**  
  - To trigger the webhook externally, configure IFTTT or any HTTP client with POST method to your n8n webhook URL with JSON body containing `{ "title": "movie name" }`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Transmission daemon requires two-step request due to CSRF protection (X-Transmission-Session-Id header). This workflow handles it by retrying the request with the session ID obtained from the initial 409 Conflict response.               | [Cross-site request forgery - Wikipedia](https://en.wikipedia.org/wiki/Cross-site_request_forgery) |
| Telegram notification nodes require you to create a Telegram bot and configure credentials plus chat ID. Use `useridinfobot` Telegram bot to get your user ID easily.                                                                     | [Telegram node docs](https://docs.n8n.io/nodes/n8n-nodes-base.telegram/)                            |
| The workflow is designed to integrate with Google Assistant via IFTTT, using phrases like “Ok Google, download [movie name]” that trigger a webhook POST to n8n.                                                                            | User guide in description with IFTTT screenshots                                                    |
| Security concern: The webhook endpoint could be exposed. It is recommended to add a shared token or authentication parameter in the POST body to prevent unauthorized triggering and potential malware downloads.                            | Security best practices                                                                              |
| The `torrent-search-api` library and dependencies have known vulnerabilities; use with caution on production media centers.                                                                                                              | npm package page warnings                                                                           |
| The search results rely on torrent providers KickassTorrents and Rarbg; their availability and reliability may vary.                                                                                                                    | Runtime dependency                                                                                  |

---

This document fully describes the workflow's logic, node details, configuration steps, and external dependencies to enable reproduction, modification, and troubleshooting.