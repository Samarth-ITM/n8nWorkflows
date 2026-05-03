Track n8n Community Topics with Keywords and Save to Google Sheets

https://n8nworkflows.xyz/workflows/track-n8n-community-topics-with-keywords-and-save-to-google-sheets-3152


# Track n8n Community Topics with Keywords and Save to Google Sheets

### 1. Workflow Overview

This workflow is designed to help community managers, developers, and enthusiasts monitor the n8n community forum for new discussion topics containing specific keywords. It automates the process of tracking relevant topics, extracting key information, and logging this data into a Google Sheet for organized record-keeping. Optionally, it can notify users via Slack or email whenever new topics are added.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Keyword Monitoring:** Periodically queries the n8n community forum for new topics matching a user-defined keyword.
- **1.2 Data Extraction and Processing:** Parses the API response to extract individual topic details.
- **1.3 Google Sheets Logging:** Appends or updates the extracted topic data into a configured Google Sheet.
- **1.4 Optional Alerting:** Listens for new rows added to the Google Sheet and sends notifications via Slack and/or email.
- **1.5 Configuration Guidance:** Sticky notes provide instructions for customizing the keyword query and Google Sheets setup.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Keyword Monitoring

- **Overview:**  
  This block triggers the workflow at regular intervals (hourly) and performs an HTTP request to the n8n community forum search API to retrieve the latest topics matching a specified keyword.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get latest topics (HTTP Request)  
  - Sticky Note (Keyword modification guidance)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow every hour.  
    - Configuration: Interval set to 1 hour.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Get latest topics" node.  
    - Edge Cases: If n8n instance is down or paused, triggers will not fire; no direct error handling here.

  - **Get latest topics**  
    - Type: HTTP Request  
    - Role: Queries the n8n community forum search endpoint for topics matching the keyword.  
    - Configuration:  
      - URL: `https://community.n8n.io/search`  
      - Query Parameters:  
        - `q`: Keyword to search for (default placeholder "ADD-YOUR-KEYWORD-HERE")  
        - `order`: "latest" to get newest topics first  
      - Response Format: JSON  
    - Inputs: Triggered by Schedule Trigger  
    - Outputs: Passes response JSON to "Get topics" node  
    - Expressions: Keyword is hardcoded in query parameters but intended to be edited by user.  
    - Edge Cases:  
      - Network errors or API downtime could cause request failure.  
      - Keyword not set or misspelled results in no or irrelevant data.  
      - Rate limiting by the forum API is possible but not handled explicitly.

  - **Sticky Note (Keyword modification guidance)**  
    - Type: Sticky Note  
    - Role: Provides instructions to user on how to modify the keyword parameter in the HTTP Request node.  
    - Content: Explains to double-click the HTTP Request node and change the "q" parameter to desired keyword.

---

#### 2.2 Data Extraction and Processing

- **Overview:**  
  This block processes the JSON response from the forum API, extracting the list of topics for further handling.

- **Nodes Involved:**  
  - Get topics (Split Out)

- **Node Details:**

  - **Get topics**  
    - Type: Split Out  
    - Role: Extracts the array of topics from the JSON response, splitting them into individual items for processing.  
    - Configuration:  
      - Field to split out: `topics` (the key in the JSON response containing the list of topic objects)  
    - Inputs: Receives JSON from "Get latest topics" node  
    - Outputs: Each topic is output as a separate item to the "Google Sheets" node  
    - Edge Cases:  
      - If the `topics` field is missing or empty, no data will be passed forward.  
      - Malformed JSON or unexpected API response structure could cause failure.

---

#### 2.3 Google Sheets Logging

- **Overview:**  
  This block appends or updates rows in a Google Sheet with the extracted topic details, maintaining an organized log of tracked community discussions.

- **Nodes Involved:**  
  - Google Sheets  
  - Sticky Note (Google Sheets setup guidance)

- **Node Details:**

  - **Google Sheets**  
    - Type: Google Sheets (Append or Update)  
    - Role: Writes topic data into a specific Google Sheet, either appending new rows or updating existing ones based on topic ID.  
    - Configuration:  
      - Operation: Append or Update  
      - Sheet Name: Sheet with `gid=0` (default first sheet)  
      - Document ID: User must specify the Google Sheet document ID  
      - Columns mapped:  
        - `id`: topic id from JSON  
        - `url`: constructed URL using topic slug (`https://community.n8n.io/t/{{ $json.slug }}`)  
        - `date`: topic creation date  
        - `title`: topic title  
        - `has_solution`: boolean indicating if topic has accepted answer  
      - Matching Column: `id` to avoid duplicates  
    - Inputs: Receives individual topic items from "Get topics" node  
    - Outputs: None connected downstream  
    - Credentials: Requires Google Sheets OAuth2 credentials  
    - Edge Cases:  
      - Missing or incorrect document ID or sheet name will cause errors.  
      - Credential expiration or permission issues can block access.  
      - Data type mismatches or missing fields in JSON may cause partial failures.

  - **Sticky Note (Google Sheets setup guidance)**  
    - Type: Sticky Note  
    - Role: Instructs user to add Google Sheets API credentials and configure columns in the sheet accordingly.  
    - Content: Advises to double-click the Google Sheets node, select the document, and ensure columns `id`, `date`, `title`, `url`, `has_solution` exist.

---

#### 2.4 Optional Alerting

- **Overview:**  
  This block listens for new rows added to the Google Sheet and sends notifications via Slack and/or email to alert users about new topics.

- **Nodes Involved:**  
  - Google Sheets Trigger  
  - Slack  
  - Send Email  
  - Sticky Note (Alerting guidance)

- **Node Details:**

  - **Google Sheets Trigger**  
    - Type: Google Sheets Trigger  
    - Role: Watches the configured Google Sheet for new rows added or updated.  
    - Configuration:  
      - Poll interval: Every minute  
      - Document ID and Sheet Name: Same as the Google Sheets node  
    - Inputs: None (trigger node)  
    - Outputs: Connected to Slack and Send Email nodes  
    - Credentials: Requires Google Sheets OAuth2 credentials (can be same or different from logging node)  
    - Edge Cases:  
      - Polling frequency may cause delays or missed updates if too infrequent.  
      - Credential or permission issues may prevent trigger from firing.

  - **Slack**  
    - Type: Slack  
    - Role: Sends a notification message to a configured Slack webhook/channel when new topics are detected.  
    - Configuration:  
      - Message text: "New topics are available in n8n community"  
      - Webhook ID: Configured webhook for Slack channel  
    - Inputs: Triggered by Google Sheets Trigger  
    - Outputs: None  
    - Edge Cases:  
      - Slack webhook misconfiguration or network issues can cause message delivery failure.

  - **Send Email**  
    - Type: Email Send  
    - Role: Sends an email notification about new topics.  
    - Configuration:  
      - Email text: "New topics are available in n8n community."  
      - SMTP credentials configured  
      - Email format: plain text  
    - Inputs: Triggered by Google Sheets Trigger  
    - Outputs: None  
    - Edge Cases:  
      - SMTP server issues or credential errors can cause email failures.

  - **Sticky Note (Alerting guidance)**  
    - Type: Sticky Note  
    - Role: Explains that alerting nodes are optional and can be deleted if not needed.  
    - Content: "Send a message when Sheet is updated (Optional). Delete these nodes if you don't want to be alerted. You can configure channels you want to connect, when Google Sheet is updated, so that you receive a message instantly."

---

### 3. Summary Table

| Node Name          | Node Type              | Functional Role                          | Input Node(s)         | Output Node(s)           | Sticky Note                                                                                          |
|--------------------|------------------------|----------------------------------------|-----------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger    | Schedule Trigger       | Triggers workflow hourly                | None                  | Get latest topics        |                                                                                                    |
| Get latest topics   | HTTP Request           | Queries n8n community forum for topics | Schedule Trigger      | Get topics               |                                                                                                    |
| Get topics          | Split Out              | Extracts individual topics from response | Get latest topics    | Google Sheets            |                                                                                                    |
| Google Sheets       | Google Sheets          | Appends/updates topic data in sheet    | Get topics             | None                     | Add your Google Sheets API credentials. Select document and ensure columns id, date, title, url, has_solution exist |
| Google Sheets Trigger | Google Sheets Trigger | Watches sheet for new rows              | None                   | Slack, Send Email        |                                                                                                    |
| Slack              | Slack                  | Sends Slack notification                | Google Sheets Trigger  | None                     |                                                                                                    |
| Send Email         | Email Send             | Sends email notification                | Google Sheets Trigger  | None                     |                                                                                                    |
| Sticky Note        | Sticky Note            | Keyword modification guidance           | None                   | None                     | Double-click the HTTP Request node to edit the "q" parameter for your keyword                      |
| Sticky Note1       | Sticky Note            | Google Sheets setup guidance            | None                   | None                     | Double-click the Google Sheets node to add credentials and configure columns                       |
| Sticky Note2       | Sticky Note            | Alerting optional guidance               | None                   | None                     | Send a message when Sheet is updated (Optional). Delete these nodes if you don't want to be alerted |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set the interval to trigger every 1 hour.

2. **Add an HTTP Request node named "Get latest topics":**  
   - URL: `https://community.n8n.io/search`  
   - Method: GET  
   - Query Parameters:  
     - `q`: Set to the keyword you want to monitor (e.g., "n8n")  
     - `order`: Set to "latest"  
   - Response Format: JSON  
   - Connect the Schedule Trigger output to this node input.

3. **Add a Split Out node named "Get topics":**  
   - Field to split out: `topics`  
   - Connect "Get latest topics" output to this node input.

4. **Add a Google Sheets node:**  
   - Operation: Append or Update  
   - Document ID: Select or enter your Google Sheet document ID  
   - Sheet Name: Use the sheet with `gid=0` or specify your target sheet  
   - Columns to map (define below):  
     - `id` → `{{$json["id"]}}`  
     - `url` → `https://community.n8n.io/t/{{$json["slug"]}}` (constructed URL)  
     - `date` → `{{$json["created_at"]}}`  
     - `title` → `{{$json["title"]}}`  
     - `has_solution` → `{{$json["has_accepted_answer"]}}`  
   - Matching Columns: `id` (to avoid duplicates)  
   - Connect "Get topics" output to this node input.  
   - Configure Google Sheets OAuth2 credentials.

5. **(Optional) Add a Google Sheets Trigger node:**  
   - Document ID and Sheet Name: same as Google Sheets node  
   - Poll interval: every minute  
   - Configure Google Sheets OAuth2 credentials.  

6. **(Optional) Add a Slack node:**  
   - Configure Slack webhook credentials.  
   - Message text: "New topics are available in n8n community"  
   - Connect Google Sheets Trigger output to Slack node input.

7. **(Optional) Add an Email Send node:**  
   - Configure SMTP credentials.  
   - Email text: "New topics are available in n8n community."  
   - Email format: plain text  
   - Connect Google Sheets Trigger output to Email Send node input.

8. **Add Sticky Notes for user guidance:**  
   - One near the HTTP Request node explaining how to modify the keyword query parameter.  
   - One near the Google Sheets node explaining how to add credentials and set up columns.  
   - One near the alerting nodes explaining that alerting is optional and can be removed.

9. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow template helps track n8n community forum topics by keyword and logs them in Sheets. | Workflow purpose and use case description                                                        |
| Modify the HTTP Request node's "q" parameter to set your desired keyword for monitoring.         | Keyword customization instruction                                                               |
| Ensure your Google Sheet has columns: `id`, `date`, `title`, `url`, `has_solution` before use.   | Google Sheets setup requirement                                                                 |
| Optional alerting via Slack or Email can be configured to receive instant notifications.          | Alerting feature explanation                                                                    |
| Google Sheets OAuth2 credentials must be configured for both reading and writing nodes.          | Credential setup requirement                                                                    |
| Slack webhook and SMTP credentials must be properly configured for notifications to work.        | Credential setup for alerting nodes                                                             |
| The community forum API endpoint used: https://community.n8n.io/search                           | API reference                                                                                   |

---

This comprehensive documentation enables users and automation agents to understand, reproduce, and modify the workflow effectively, while anticipating common issues and configuration needs.