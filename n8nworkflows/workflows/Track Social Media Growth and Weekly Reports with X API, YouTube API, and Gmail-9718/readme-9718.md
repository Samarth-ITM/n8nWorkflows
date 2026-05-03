Track Social Media Growth and Weekly Reports with X API, YouTube API, and Gmail

https://n8nworkflows.xyz/workflows/track-social-media-growth-and-weekly-reports-with-x-api--youtube-api--and-gmail-9718


# Track Social Media Growth and Weekly Reports with X API, YouTube API, and Gmail

### 1. Workflow Overview

This workflow automates the tracking of social media growth metrics from X (formerly Twitter) and YouTube, storing daily follower and subscriber counts in a centralized Data Table within n8n. It also sends a weekly summary report via Gmail every Sunday comparing the current metrics to those from the previous Monday. The workflow is designed for social media managers or growth analysts who want hands-free, scheduled monitoring and reporting.

The workflow logic is divided into the following blocks:

- **1.1 Input Configuration:** Setting target usernames/handles for X and YouTube channels.
- **1.2 Fetching Social Media Metrics:** Retrieving follower/subscriber counts from X API and YouTube API.
- **1.3 Storing Daily Metrics:** Upserting the retrieved daily counts into a Data Table.
- **1.4 Scheduling and Conditional Logic:** Triggering the workflow daily and checking if today is Sunday to generate weekly reports.
- **1.5 Weekly Report Generation and Sending:** Fetching data for today and last Monday, calculating growth, and sending a summary email via Gmail.
- **1.6 Documentation and Setup Notes:** Sticky notes providing setup instructions and helpful resources.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Configuration

**Overview:**  
Defines the social media accounts to track by setting usernames/handles for X and YouTube. This block initializes variables used in API requests.

**Nodes Involved:**  
- Set X Username  
- Set YT Channel Username  
- Sticky Note (target usernames instructions)

**Node Details:**

- **Set X Username**  
  - Type: Set node  
  - Role: Assigns the X (Twitter) username to the variable `xUsername`.  
  - Configuration: Sets `xUsername` to `"n8n_io"` as default.  
  - Input: Triggered by Schedule Trigger node.  
  - Output: Connected to Fetch X Profile Metrics node.  
  - Edge cases: Ensure valid username strings; missing or invalid usernames will cause API errors downstream.

- **Set YT Channel Username**  
  - Type: Set node  
  - Role: Assigns the YouTube channel handle to the variable `ytUsername`.  
  - Configuration: Sets `ytUsername` to `"n8n-io"` as default.  
  - Input: Triggered by Schedule Trigger node.  
  - Output: Connected to Fetch YT Channel Stats node.  
  - Edge cases: Incorrect usernames or handles may result in empty API responses or errors.

- **Sticky Note** (Positioned near input nodes)  
  - Role: Provides instructions to users on how to set their own target usernames for X and YouTube.  
  - Content emphasizes accuracy of these usernames since they drive API calls.

---

#### 1.2 Fetching Social Media Metrics

**Overview:**  
Calls external APIs to retrieve current follower counts from X and subscriber counts from YouTube.

**Nodes Involved:**  
- Fetch X Profile Metrics  
- Fetch YT Channel Stats  
- Sticky Note1 (YouTube Query Auth setup)  
- Sticky Note3 (X API Bearer Token setup)

**Node Details:**

- **Fetch X Profile Metrics**  
  - Type: HTTP Request node  
  - Role: Calls X API endpoint to fetch public metrics for the username set in `xUsername`.  
  - Configuration:  
    - URL: `https://api.twitter.com/2/users/by/username/{{ $json.xUsername }}`  
    - Query Parameter: `user.fields=public_metrics` to get follower count.  
    - Authentication: Bearer token via predefined credentials (token stored securely).  
  - Inputs: From Set X Username node.  
  - Output: Passes data to Save X Followers Count node.  
  - Edge cases: Authentication failure, rate limiting by X API, invalid usernames causing 404 or empty response.

- **Fetch YT Channel Stats**  
  - Type: HTTP Request node  
  - Role: Calls YouTube Data API v3 to get channel statistics for the handle in `ytUsername`.  
  - Configuration:  
    - URL: `https://www.googleapis.com/youtube/v3/channels`  
    - Query Parameters: `part=statistics`, `forHandle={{ $json.ytUsername }}`  
    - Authentication: Query Auth with API key from Google Cloud Console (configured in credentials).  
  - Inputs: From Set YT Channel Username node.  
  - Output: Passes data to Save YT Subscriber Count node.  
  - Edge cases: Invalid API key, quota exceeded, invalid channel handle, network errors.

- **Sticky Note1**  
  - Provides detailed step-by-step instructions on setting up Query Auth for the YouTube API HTTP Request node. Includes a reference image link for visual aid.

- **Sticky Note3**  
  - Details how to configure Bearer token authentication for the X API HTTP Request node, including where to find the Bearer token in the X Developer Portal.

---

#### 1.3 Storing Daily Metrics

**Overview:**  
Saves or updates follower and subscriber counts in a centralized Data Table within n8n, keyed by the current UTC date.

**Nodes Involved:**  
- Save X Followers Count  
- Save YT Subscriber Count  
- Sticky Note2 (Data Table creation instructions)

**Node Details:**

- **Save X Followers Count**  
  - Type: Data Table node  
  - Role: Upserts today's follower count for X into the Data Table.  
  - Configuration:  
    - Columns:  
      - `date`: Current date at UTC midnight (new Date with UTC components)  
      - `xFollowersCount`: Extracted from X API response at `data.public_metrics.followers_count`  
    - Filter: Upsert based on `date` matching today to avoid duplicates.  
    - Data Table: Connected to a Data Table with fields `date` (DateTime) and `xFollowersCount` (Number).  
  - Inputs: From Fetch X Profile Metrics node.  
  - Output: None further in this branch.  
  - Edge cases: Date calculation errors, data type mismatches, Data Table connectivity issues.

- **Save YT Subscriber Count**  
  - Type: Data Table node  
  - Role: Upserts today's subscriber count for YouTube into the same Data Table.  
  - Configuration:  
    - Columns:  
      - `date`: Current UTC midnight date  
      - `ytSubscriberCount`: From YouTube API response at `items[0].statistics.subscriberCount`  
    - Filter: Upsert using `date` to avoid duplicates.  
    - Data Table: Same as above, with additional field `ytSubscriberCount` (Number).  
  - Inputs: From Fetch YT Channel Stats node.  
  - Output: None further downstream.  
  - Edge cases: Missing or malformed API responses, subscriber counts as strings requiring conversion, Data Table write errors.

- **Sticky Note2**  
  - Instructs users to create a Data Table in n8n with the appropriate fields (`date`, `xFollowersCount`, `ytSubscriberCount`) and data types.  
  - Emphasizes importance of correct field types to prevent later data errors.

---

#### 1.4 Scheduling and Conditional Logic

**Overview:**  
Runs the workflow daily, sets usernames, fetches metrics, and conditionally triggers weekly report generation only on Sundays.

**Nodes Involved:**  
- Schedule Trigger  
- If Today Is Sunday  
- Sticky Note6 (weekly report condition instructions)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger node  
  - Role: Starts the workflow on a daily schedule.  
  - Configuration: Interval set to 1 day (default daily trigger).  
  - Outputs: Sends trigger to Set X Username, Set YT Channel Username, and If Today Is Sunday nodes.  
  - Edge cases: Scheduling misconfigurations or time zone mismatches may cause missed or multiple runs.

- **If Today Is Sunday**  
  - Type: If node  
  - Role: Checks if the current day (UTC) is Sunday (represented by 0 in JavaScript Date `getUTCDay()`).  
  - Configuration:  
    - Condition: `new Date(new Date().toISOString().split('T')[0] + 'T00:00:00.000Z').getUTCDay() === 0`  
  - Input: From Schedule Trigger node.  
  - Output: If true, proceeds to Get Today's Data; otherwise, no further action for the weekly report branch.  
  - Edge cases: Date calculation errors or time zone issues could cause incorrect branching.

- **Sticky Note6**  
  - Brief note explaining that the weekly report is generated and sent only on Sundays, which clarifies the purpose of the If node.

---

#### 1.5 Weekly Report Generation and Sending

**Overview:**  
Retrieves saved data for today and the previous Monday, computes follower and subscriber changes, and sends a formatted summary email via Gmail.

**Nodes Involved:**  
- Get Today's Data  
- Get Last Monday's Data  
- Send a message  
- Sticky Note5 (Gmail OAuth2 setup)

**Node Details:**

- **Get Today's Data**  
  - Type: Data Table node  
  - Role: Retrieves today's stored data (followers and subscribers) from the Data Table.  
  - Configuration: Filters on `date` equal to current UTC midnight date.  
  - Input: From If Today Is Sunday node (true branch).  
  - Output: Passes data to Get Last Monday's Data node.  
  - Edge cases: Missing data for today may cause null errors downstream.

- **Get Last Monday's Data**  
  - Type: Data Table node  
  - Role: Retrieves last Monday's stored data for comparison.  
  - Configuration:  
    - Date filter computes last Monday's UTC midnight date using:  
      ```js
      new Date(Date.UTC(
        new Date().getUTCFullYear(),
        new Date().getUTCMonth(),
        new Date().getUTCDate() - ((new Date().getUTCDay() + 6) % 7)
      ))
      ```  
  - Input: From Get Today's Data node.  
  - Output: Passes data to Send a message node.  
  - Edge cases: Missing or incomplete data for last Monday will affect comparison accuracy.

- **Send a message**  
  - Type: Gmail node  
  - Role: Sends a weekly summary email with follower/subscriber growth stats.  
  - Configuration:  
    - Recipient: `example@mail.com` (placeholder, replace with actual email)  
    - Subject: "Weekly Social Media Growth Report 📊"  
    - Message body: Uses expressions to calculate difference and percentage growth between last Monday and today for both X and YouTube metrics.  
    - Authentication: Google OAuth2 credential configured with Gmail API scope (`gmail.send`).  
  - Inputs: From Get Last Monday's Data node.  
  - Output: None (final node).  
  - Edge cases: Email sending failures due to OAuth token issues, invalid recipient, Gmail API quota limits.

- **Sticky Note5**  
  - Detailed instructions for setting up Gmail node authentication with Google OAuth2, including enabling the Gmail API, creating OAuth credentials, and setting redirect URIs. Also explains how to specify recipient email addresses.

---

#### 1.6 Documentation and Setup Notes

**Overview:**  
Sticky notes that provide users with guidance on configuration, credentials, and support contact information.

**Nodes Involved:**  
- Sticky Note (target usernames)  
- Sticky Note1 (YouTube Query Auth setup)  
- Sticky Note2 (Data Table creation)  
- Sticky Note3 (X API Bearer token)  
- Sticky Note4 (Support contacts)  
- Sticky Note5 (Gmail OAuth2 setup)  
- Sticky Note6 (Weekly report timing)

**Node Details:**

- **Sticky Note4**  
  - Provides multiple contact points for support related to this workflow: email, X (Twitter), Discord, WhatsApp, Reddit, and encourages users to ask any questions.

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                           | Input Node(s)               | Output Node(s)             | Sticky Note                                            |
|-------------------------|-----------------------|-----------------------------------------|-----------------------------|----------------------------|--------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger      | Starts workflow daily                    |                             | Set X Username, Set YT Channel Username, If Today Is Sunday |                                                        |
| Set X Username           | Set                   | Sets X username variable                 | Schedule Trigger             | Fetch X Profile Metrics     |                                                        |
| Fetch X Profile Metrics  | HTTP Request          | Fetches follower data from X API         | Set X Username              | Save X Followers Count      | Sticky Note3 (X API Bearer token setup)                 |
| Save X Followers Count   | Data Table            | Stores X followers count for today       | Fetch X Profile Metrics      |                            | Sticky Note2 (Data Table creation instructions)         |
| Set YT Channel Username  | Set                   | Sets YouTube channel username            | Schedule Trigger             | Fetch YT Channel Stats      |                                                        |
| Fetch YT Channel Stats   | HTTP Request          | Fetches subscriber data from YouTube API | Set YT Channel Username      | Save YT Subscriber Count    | Sticky Note1 (YouTube Query Auth setup)                  |
| Save YT Subscriber Count | Data Table            | Stores YouTube subscriber count for today| Fetch YT Channel Stats       |                            | Sticky Note2 (Data Table creation instructions)         |
| If Today Is Sunday       | If                    | Checks if current day is Sunday          | Schedule Trigger             | Get Today's Data (if true)  | Sticky Note6 (Weekly report condition explanation)      |
| Get Today's Data         | Data Table            | Retrieves today's metrics from Data Table| If Today Is Sunday           | Get Last Monday's Data      |                                                        |
| Get Last Monday's Data   | Data Table            | Retrieves last Monday's metrics          | Get Today's Data             | Send a message             |                                                        |
| Send a message           | Gmail                 | Sends weekly growth report email         | Get Last Monday's Data       |                            | Sticky Note5 (Gmail OAuth2 setup)                        |
| Sticky Note              | Sticky Note           | Instructions for setting target usernames|                             |                            | Sticky Note (target usernames instructions)              |
| Sticky Note1             | Sticky Note           | YouTube API Query Auth setup guide       |                             |                            | Sticky Note1                                             |
| Sticky Note2             | Sticky Note           | Data Table creation guidance              |                             |                            | Sticky Note2                                             |
| Sticky Note3             | Sticky Note           | X API Bearer token setup guide            |                             |                            | Sticky Note3                                             |
| Sticky Note4             | Sticky Note           | Support and contact information           |                             |                            | Sticky Note4                                             |
| Sticky Note5             | Sticky Note           | Gmail OAuth2 credential setup instructions|                             |                            | Sticky Note5                                             |
| Sticky Note6             | Sticky Note           | Weekly report generation timing note      |                             |                            | Sticky Note6                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to daily (every 1 day).

2. **Create Set X Username node**  
   - Type: Set  
   - Add a string field named `xUsername` with value set to your X handle (e.g., `"n8n_io"`).  
   - Connect Schedule Trigger output to this node.

3. **Create Fetch X Profile Metrics node**  
   - Type: HTTP Request  
   - URL: `https://api.twitter.com/2/users/by/username/{{ $json.xUsername }}`  
   - Query Parameters: `user.fields=public_metrics`  
   - Authentication: Select Bearer Token authentication.  
   - Create new credential with your X API Bearer token (from X Developer Portal).  
   - Connect Set X Username output to this node.

4. **Create Save X Followers Count node**  
   - Type: Data Table  
   - Create or select a Data Table with fields:  
     - `date` (DateTime)  
     - `xFollowersCount` (Number)  
     - `ytSubscriberCount` (Number) - optional here, can remain empty.  
   - Configure columns to upsert:  
     - `date`: `={{ new Date(Date.UTC(new Date().getUTCFullYear(), new Date().getUTCMonth(), new Date().getUTCDate())) }}`  
     - `xFollowersCount`: `={{ $json.data.public_metrics.followers_count }}`  
   - Filter: Upsert on `date` matching today’s date.  
   - Connect Fetch X Profile Metrics output to this node.

5. **Create Set YT Channel Username node**  
   - Type: Set  
   - Add string field `ytUsername` with your YouTube channel handle (e.g., `"n8n-io"`).  
   - Connect Schedule Trigger output to this node.

6. **Create Fetch YT Channel Stats node**  
   - Type: HTTP Request  
   - URL: `https://www.googleapis.com/youtube/v3/channels`  
   - Query Parameters:  
     - `part=statistics`  
     - `forHandle={{ $json.ytUsername }}`  
   - Authentication: Use Query Auth with your Google Cloud API key (YouTube Data API enabled).  
   - Connect Set YT Channel Username output to this node.

7. **Create Save YT Subscriber Count node**  
   - Type: Data Table  
   - Use the same Data Table as Save X Followers Count.  
   - Configure columns:  
     - `date`: same as above (UTC midnight)  
     - `ytSubscriberCount`: `={{ $json.items[0].statistics.subscriberCount }}`  
   - Filter: Upsert on `date`.  
   - Connect Fetch YT Channel Stats output to this node.

8. **Create If Today Is Sunday node**  
   - Type: If  
   - Condition: Check if the current UTC day is Sunday (0).  
   - Expression:  
     ```js
     new Date(new Date().toISOString().split('T')[0] + 'T00:00:00.000Z').getUTCDay() === 0
     ```  
   - Connect Schedule Trigger output to this node.

9. **Create Get Today's Data node**  
   - Type: Data Table  
   - Operation: Get  
   - Filter on `date` equal to today’s UTC midnight date.  
   - Data Table: Same as before.  
   - Connect If Today Is Sunday (true output) to this node.

10. **Create Get Last Monday's Data node**  
    - Type: Data Table  
    - Operation: Get  
    - Filter on `date` equal to last Monday’s UTC midnight date, calculated as:  
      ```js
      new Date(Date.UTC(
        new Date().getUTCFullYear(),
        new Date().getUTCMonth(),
        new Date().getUTCDate() - ((new Date().getUTCDay() + 6) % 7)
      ))
      ```  
    - Data Table: Same as above.  
    - Connect Get Today's Data output to this node.

11. **Create Send a message node**  
    - Type: Gmail  
    - Authentication: Google OAuth2 with Gmail API enabled and `gmail.send` scope.  
    - Configure OAuth2 credentials accordingly.  
    - Set recipient email in "To Email" field (e.g., your email).  
    - Subject: `"Weekly Social Media Growth Report 📊"`  
    - Message body (text type) with this expression (adjust as needed):  
      ```
      Hey there,

      Here’s your social media performance summary for this week:

      - X Followers Change: {{ $('Get Today\'s Data').item.json.xFollowersCount - $json.xFollowersCount }} ({{ (( $('Get Today\'s Data').item.json.xFollowersCount - $json.xFollowersCount ) / $json.xFollowersCount * 100).toFixed(2) }}%)
      - YouTube Subscribers Change: {{ $('Get Today\'s Data').item.json.ytSubscriberCount - $json.ytSubscriberCount }} ({{ (( $('Get Today\'s Data').item.json.ytSubscriberCount - $json.ytSubscriberCount ) / $json.ytSubscriberCount * 100).toFixed(2) }}%)

      Summary:
      Started with {{ $json.xFollowersCount }} X followers and {{ $json.ytSubscriberCount }} YouTube subscribers.
      Now at {{ $('Get Today\'s Data').item.json.xFollowersCount }} X followers and {{ $('Get Today\'s Data').item.json.ytSubscriberCount }} subscribers.

      Keep up the great work 🚀
      ```
    - Connect Get Last Monday's Data output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Make sure to create a Data Table in n8n with fields `date` (DateTime), `xFollowersCount` (Number), and `ytSubscriberCount` (Number) to store data. | Sticky Note2                                                                                                    |
| For X API authentication, create Bearer token credentials using your token from X Developer Portal.                                                | Sticky Note3                                                                                                    |
| For YouTube API, use Query Auth with your Google Cloud API key. Enable YouTube Data API in GCP console.                                            | Sticky Note1                                                                                                    |
| Gmail node requires Google OAuth2 credentials with Gmail API `gmail.send` scope enabled. Set redirect URI to your n8n instance URI.               | Sticky Note5                                                                                                    |
| Weekly report email is triggered only on Sundays based on UTC time.                                                                                 | Sticky Note6                                                                                                    |
| Support contact: Email hello@scoutnow.app, X (Twitter) @ScoutNowApp and @encryptman, Discord encryptman#4196, WhatsApp https://wa.me/923161262192 | Sticky Note4                                                                                                    |
| Reference image for YouTube Query Auth setup: [Query Auth Setup in HTTP Node](https://res.cloudinary.com/datbbikfe/image/upload/v1760540335/n8n/Screenshot_2025-10-15_at_7.58.40_PM_pxkbbg.png) | Sticky Note1                                                                                                    |

---

*Disclaimer:* The provided text is exclusively derived from an n8n automated workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.