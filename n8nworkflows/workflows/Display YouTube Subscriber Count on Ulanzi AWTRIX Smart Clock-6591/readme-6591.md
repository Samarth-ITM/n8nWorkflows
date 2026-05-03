Display YouTube Subscriber Count on Ulanzi AWTRIX Smart Clock

https://n8nworkflows.xyz/workflows/display-youtube-subscriber-count-on-ulanzi-awtrix-smart-clock-6591


# Display YouTube Subscriber Count on Ulanzi AWTRIX Smart Clock

### 1. Workflow Overview

This n8n workflow is designed to display the current YouTube subscriber count on an Ulanzi AWTRIX Smart Clock, an IoT dashboard device. It targets users who want to monitor their YouTube channel growth in real-time on a dedicated smart display. The workflow runs on an hourly schedule and consists of three main logical blocks:

- **1.1 Scheduled Trigger:** Periodically initiates the workflow every hour.
- **1.2 YouTube Data Retrieval:** Fetches the subscriber count from the YouTube Data API.
- **1.3 AWTRIX Display Update:** Sends the formatted subscriber count to the AWTRIX device for display.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow once every hour, ensuring that the displayed subscriber count is updated regularly.

- **Nodes Involved:**  
  - Every Hour

- **Node Details:**

  - **Node Name:** Every Hour  
    - **Type:** Schedule Trigger  
    - **Technical Role:** Initiates workflow execution on a recurring time-based schedule.  
    - **Configuration:** Set to trigger every hour using the interval field set to "hours".  
    - **Key Expressions/Variables:** None.  
    - **Input Connections:** None (trigger node).  
    - **Output Connections:** Outputs to "Fetch YT Subscriber Count" node.  
    - **Version Requirements:** Compatible with n8n version supporting Schedule Trigger node v1.  
    - **Potential Failures:** None typical; possible runtime engine failure or time drift if system clock is incorrect.  
    - **Sub-workflow:** None.

#### 2.2 YouTube Data Retrieval

- **Overview:**  
  This block requests the YouTube Data API to obtain the latest subscriber count for a specified channel, using API credentials and channel ID.

- **Nodes Involved:**  
  - Fetch YT Subscriber Count

- **Node Details:**

  - **Node Name:** Fetch YT Subscriber Count  
    - **Type:** HTTP Request  
    - **Technical Role:** Performs a GET request to the YouTube Data API v3 channels endpoint to retrieve channel statistics.  
    - **Configuration:**  
      - URL: `https://youtube.googleapis.com/youtube/v3/channels`  
      - Query Parameters:  
        - `id`: YouTube Channel ID (to be replaced by the user)  
        - `key`: YouTube API key (to be replaced by the user)  
        - `part`: "statistics" (to fetch subscriber count and other stats)  
      - Method: GET (default for HTTP Request when no method specified)  
    - **Key Expressions/Variables:** None in the node config; data extracted downstream.  
    - **Input Connections:** From "Every Hour" node.  
    - **Output Connections:** Outputs to "Send to AWTRIX" node.  
    - **Version Requirements:** Requires n8n version supporting HTTP Request node v3.  
    - **Potential Failures:**  
      - API authentication failure (invalid or missing API key)  
      - Channel ID invalid or private channel  
      - API quota exceeded  
      - Network timeouts or connection errors  
      - Unexpected API response structure changes  
    - **Sub-workflow:** None.

#### 2.3 AWTRIX Display Update

- **Overview:**  
  This block sends the formatted subscriber count to the AWTRIX Smart Clock’s custom API endpoint to update the display with the latest subscriber number and an icon.

- **Nodes Involved:**  
  - Send to AWTRIX

- **Node Details:**

  - **Node Name:** Send to AWTRIX  
    - **Type:** HTTP Request  
    - **Technical Role:** Sends a POST request with subscriber count data to AWTRIX device API.  
    - **Configuration:**  
      - URL: `http://YOUR_AWTRIX_IP/api/custom` (replace with actual AWTRIX IP)  
      - Method: POST  
      - Body Parameters:  
        - `text`: Formatted subscriber count extracted from previous node JSON data, using JavaScript expression:  
          ```js
          Intl.NumberFormat('en-US', {
            notation: "compact",
            maximumFractionDigits: 2
          }).format($json.items[0].statistics.subscriberCount)
          ```  
          This converts raw subscriber count to a compact, human-readable format (e.g., "1.2M").  
        - `icon`: A user-defined icon ID for display on AWTRIX (to be replaced).  
      - Query Parameters:  
        - `name`: "ytsubs" (identifies the custom widget on AWTRIX)  
      - Retry on Fail: Enabled, so request will retry in case of transient network errors.  
    - **Key Expressions/Variables:** See above for `text` expression.  
    - **Input Connections:** From "Fetch YT Subscriber Count" node.  
    - **Output Connections:** None (last node).  
    - **Version Requirements:** Requires n8n HTTP Request node v3 supporting body and query parameter sending.  
    - **Potential Failures:**  
      - Network failures reaching AWTRIX device (wrong IP, device offline)  
      - AWTRIX API errors or changes in API schema  
      - Malformed body parameters causing display errors  
      - Incorrect icon ID causing missing/incorrect icon display  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role              | Input Node(s)      | Output Node(s)           | Sticky Note                             |
|-----------------------|--------------------|-----------------------------|--------------------|--------------------------|---------------------------------------|
| Every Hour            | Schedule Trigger   | Initiates workflow hourly    | -                  | Fetch YT Subscriber Count |                                       |
| Fetch YT Subscriber Count | HTTP Request       | Retrieves YouTube subscriber count | Every Hour         | Send to AWTRIX             |                                       |
| Send to AWTRIX         | HTTP Request       | Sends formatted count to AWTRIX device | Fetch YT Subscriber Count | -                        |                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Every Hour" Node**  
   - Add node: **Schedule Trigger**  
   - Name it: "Every Hour"  
   - Set trigger rule: Interval every 1 hour  
   - No credentials needed  
   - This node triggers the workflow periodically.

2. **Create "Fetch YT Subscriber Count" Node**  
   - Add node: **HTTP Request**  
   - Name it: "Fetch YT Subscriber Count"  
   - Set URL to: `https://youtube.googleapis.com/youtube/v3/channels`  
   - Method: GET (default)  
   - Under Query Parameters, add:  
     - `id`: Your YouTube Channel ID (replace `"YOUR_YOUTUBE_CHANNEL_ID"`)  
     - `key`: Your YouTube API Key (replace `"YOUR_YOUTUBE_API_KEY"`)  
     - `part`: `statistics`  
   - Connect output of "Every Hour" to input of this node.  
   - No credentials node needed if API Key is passed as query param.

3. **Create "Send to AWTRIX" Node**  
   - Add node: **HTTP Request**  
   - Name it: "Send to AWTRIX"  
   - Set URL to: `http://YOUR_AWTRIX_IP/api/custom` (replace with your AWTRIX device IP)  
   - Set Method to POST  
   - Under Body Parameters (send body as parameters):  
     - `text`: Use expression editor and insert:  
       ```js
       {{ Intl.NumberFormat('en-US', { notation: "compact", maximumFractionDigits: 2 }).format($json.items[0].statistics.subscriberCount) }}
       ```  
     - `icon`: Replace with your desired AWTRIX icon ID (e.g., `"YOUR_ICON_ID"`)  
   - Under Query Parameters:  
     - `name`: `ytsubs`  
   - Enable "Retry on Fail" option for network resilience.  
   - Connect output of "Fetch YT Subscriber Count" node to input of this node.

4. **Activate the Workflow**  
   - Save and activate the workflow.  
   - Ensure your AWTRIX device is reachable at the specified IP and configured to accept custom API calls.  
   - Make sure the YouTube API key has permissions and quota to access channel data.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Replace placeholders `"YOUR_YOUTUBE_CHANNEL_ID"`, `"YOUR_YOUTUBE_API_KEY"`, `"YOUR_AWTRIX_IP"`, and `"YOUR_ICON_ID"` accordingly before activating the workflow. | Critical for correct API calls and device integration.                                              |
| The YouTube Data API v3 documentation for channels endpoint: https://developers.google.com/youtube/v3/docs/channels/list | Official reference for parameter usage and quotas.                                                 |
| AWTRIX custom API documentation (if available) should be consulted for icon IDs and payload formats.                      | Ensures compatibility with your specific AWTRIX firmware and customization options.                 |
| The subscriber count is formatted in a compact notation (e.g., 1.2M) for readability on small device displays.            | This uses JavaScript Intl.NumberFormat with notation "compact".                                     |

---

This document fully describes the n8n workflow "AWTRIX YouTube Subscriber Count" and enables both human and automated agents to understand, reproduce, and maintain the integration reliably.