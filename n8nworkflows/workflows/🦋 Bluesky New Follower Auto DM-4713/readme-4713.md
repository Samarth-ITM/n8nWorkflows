🦋 Bluesky New Follower Auto DM

https://n8nworkflows.xyz/workflows/---bluesky-new-follower-auto-dm-4713


# 🦋 Bluesky New Follower Auto DM

### 1. Workflow Overview

The **"🦋 Bluesky New Follower Auto DM"** workflow automates sending direct messages (DMs) to new followers on the Bluesky social platform. It periodically checks for new follower notifications, filters out followers who have already been messaged, initiates conversations with new followers, and sends them a welcome DM.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger and Setup:** Periodically triggers the workflow and prepares initial parameters.
- **1.2 Session Creation:** Authenticates and establishes a session with Bluesky API.
- **1.3 Fetching Notifications:** Retrieves follower notifications from Bluesky.
- **1.4 Filtering New Followers:** Identifies which followers are new and have not been contacted yet.
- **1.5 Initiating Conversations:** Starts a conversation thread with each new follower.
- **1.6 Sending Welcome DM:** Sends a personalized direct message to each new follower.

Several sticky notes provide internal documentation and guidance, though their content is empty in this export.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Setup

- **Overview:**  
  This block triggers the workflow on a schedule and prepares any necessary static or dynamic setup parameters.

- **Nodes Involved:**  
  - Cron Trigger  
  - Setup

- **Node Details:**  

  - **Cron Trigger**  
    - Type: Cron Trigger  
    - Role: Periodically initiates the workflow run (default schedule, unspecified here).  
    - Config: No parameters set, so default trigger timing applies (likely every minute or hour).  
    - Inputs: None (trigger node).  
    - Outputs: Connects to Setup node.  
    - Edge Cases: Cron misconfiguration may cause unexpected run intervals.

  - **Setup**  
    - Type: Set  
    - Role: Defines static or initial variables for downstream nodes.  
    - Config: No parameters explicitly set (empty), possibly a placeholder or preparing context.  
    - Inputs: From Cron Trigger.  
    - Outputs: Connects to Create Session node.  
    - Edge Cases: Empty setup may mean hardcoded values are in subsequent HTTP nodes.

---

#### 2.2 Session Creation

- **Overview:**  
  Authenticates to Bluesky API to establish a session for authorized requests.

- **Nodes Involved:**  
  - Create Session

- **Node Details:**  

  - **Create Session**  
    - Type: HTTP Request  
    - Role: Sends an HTTP request to Bluesky API to create or refresh authentication session.  
    - Config: Details not provided here, but likely includes OAuth tokens or API keys in headers or body.  
    - Inputs: From Setup node.  
    - Outputs: Connects to Get Follow Notifications node.  
    - Edge Cases: Possible authentication failures (expired or invalid credentials), HTTP timeouts, rate limiting.

---

#### 2.3 Fetching Notifications

- **Overview:**  
  Retrieves notifications from Bluesky, specifically focusing on new follower events.

- **Nodes Involved:**  
  - Get Follow Notifications

- **Node Details:**  

  - **Get Follow Notifications**  
    - Type: HTTP Request  
    - Role: Queries Bluesky API endpoint for notifications related to new followers.  
    - Config: Likely uses session tokens from the previous node for auth, targets specific notification API endpoint.  
    - Inputs: From Create Session node.  
    - Outputs: Connects to Filter New Follows node.  
    - Edge Cases: API response errors, empty notification list, pagination issues.

---

#### 2.4 Filtering New Followers

- **Overview:**  
  Processes notification data to isolate new followers that have not yet been sent a DM.

- **Nodes Involved:**  
  - Filter New Follows

- **Node Details:**  

  - **Filter New Follows**  
    - Type: Code  
    - Role: Custom JavaScript logic to filter out followers already contacted, keeping only new ones.  
    - Config: Contains a code snippet (not visible here) that checks follower IDs against a stored list or database.  
    - Inputs: From Get Follow Notifications node.  
    - Outputs: Connects to Initiate Convo with Follower node.  
    - Edge Cases: Code errors, missing data fields, state persistence issues (e.g., tracking already messaged followers).

---

#### 2.5 Initiating Conversations

- **Overview:**  
  Starts a new conversation thread with each new follower before sending a message.

- **Nodes Involved:**  
  - Initiate Convo with Follower  
  - Filter

- **Node Details:**  

  - **Initiate Convo with Follower**  
    - Type: HTTP Request (version 4.2)  
    - Role: Sends an API call to Bluesky to open a conversation channel with a follower.  
    - Config: Uses follower ID from previous node output to target the right user.  
    - Inputs: From Filter New Follows node.  
    - Outputs: Connects to Filter node.  
    - Edge Cases: API failures, follower blocking, rate limits.

  - **Filter**  
    - Type: Filter  
    - Role: Conditional node that likely verifies if the conversation initiation was successful before proceeding.  
    - Config: Conditions not detailed; presumably checks HTTP response status or presence of conversation ID.  
    - Inputs: From Initiate Convo with Follower.  
    - Outputs: Connects to Send DM node if condition passes.  
    - Edge Cases: Improper filtering conditions, false negatives blocking message sends.

---

#### 2.6 Sending Welcome DM

- **Overview:**  
  Sends a predefined or dynamically generated welcome direct message to the follower once the conversation is established.

- **Nodes Involved:**  
  - Send DM

- **Node Details:**  

  - **Send DM**  
    - Type: HTTP Request  
    - Role: Posts a direct message to the Bluesky conversation thread initiated previously.  
    - Config: Message content likely hardcoded or templated; uses conversation or user ID from Filter node output.  
    - Inputs: From Filter node.  
    - Outputs: None (end of workflow chain).  
    - Edge Cases: Message delivery failures, API rate limits, blocked users.

---

#### 2.7 Sticky Notes

The workflow contains multiple sticky notes named:

- 📝 Setup Instructions  
- 🔑 Create Session  
- 📬 Get Notifications  
- 🔍 Filter Follows  
- 💬 Send Welcome DM  
- Sticky Note (general)  
- Sticky Note1  
- Sticky Note2  

All sticky notes have empty content in this export, so no additional annotations are available.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                               | Input Node(s)           | Output Node(s)               | Sticky Note              |
|----------------------------|---------------------|-----------------------------------------------|-------------------------|-----------------------------|--------------------------|
| Cron Trigger               | Cron Trigger        | Periodic trigger to start the workflow        | —                       | Setup                       |                          |
| Setup                     | Set                 | Prepare initial parameters (empty here)       | Cron Trigger            | Create Session              |                          |
| Create Session            | HTTP Request        | Authenticate and create Bluesky session       | Setup                   | Get Follow Notifications    | 🔑 Create Session        |
| Get Follow Notifications  | HTTP Request        | Retrieve follower notifications from Bluesky  | Create Session          | Filter New Follows          | 📬 Get Notifications     |
| Filter New Follows        | Code                | Filter new followers not yet messaged          | Get Follow Notifications| Initiate Convo with Follower| 🔍 Filter Follows        |
| Initiate Convo with Follower | HTTP Request      | Start conversation thread with new follower   | Filter New Follows      | Filter                      | Sticky Note1             |
| Filter                   | Filter               | Verify successful conversation initiation      | Initiate Convo with Follower | Send DM                 | Sticky Note2             |
| Send DM                  | HTTP Request         | Send welcome DM to new follower                 | Filter                  | —                           | 💬 Send Welcome DM       |
| 📝 Setup Instructions     | Sticky Note          | Instruction placeholder (empty)                 | —                       | —                           |                          |
| 🔑 Create Session         | Sticky Note          | Instruction placeholder (empty)                 | —                       | —                           |                          |
| 📬 Get Notifications      | Sticky Note          | Instruction placeholder (empty)                 | —                       | —                           |                          |
| 🔍 Filter Follows         | Sticky Note          | Instruction placeholder (empty)                 | —                       | —                           |                          |
| 💬 Send Welcome DM        | Sticky Note          | Instruction placeholder (empty)                 | —                       | —                           |                          |
| Sticky Note               | Sticky Note          | Empty content                                   | —                       | —                           |                          |
| Sticky Note1              | Sticky Note          | Empty content                                   | —                       | —                           |                          |
| Sticky Note2              | Sticky Note          | Empty content                                   | —                       | —                           |                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Trigger node:**  
   - Type: Cron Trigger  
   - Set the frequency for checking new followers (e.g., every 15 minutes or as desired).

2. **Add a Set node (named "Setup"):**  
   - No parameters to set initially; can be used later if required for static variables.

3. **Add an HTTP Request node (named "Create Session"):**  
   - Configure to authenticate with Bluesky API:  
     - Method: POST (likely) to the session or login endpoint.  
     - Authentication: Use necessary credentials (API key, OAuth token).  
     - Save the session token or cookies from the response for subsequent requests.

   - Connect "Setup" node output to this node input.

4. **Add an HTTP Request node (named "Get Follow Notifications"):**  
   - Configure to GET notifications from Bluesky API endpoint for new followers.  
   - Use the session token from "Create Session" for authentication headers.  
   - Connect "Create Session" output to this node input.

5. **Add a Code node (named "Filter New Follows"):**  
   - Write JavaScript code to:  
     - Parse the notifications payload.  
     - Identify new follower events.  
     - Filter out followers already messaged (requires persistence, e.g., external DB or workflow variable).  
   - Connect "Get Follow Notifications" output to this node.

6. **Add an HTTP Request node (named "Initiate Convo with Follower"):**  
   - Configure POST request to Bluesky API to start conversation with each new follower ID from previous node.  
   - Include necessary authentication headers.  
   - Connect "Filter New Follows" output to this node.

7. **Add a Filter node (named "Filter"):**  
   - Configure conditions to check if the conversation initiation response was successful, e.g., HTTP status 200 or presence of conversation ID in response.  
   - Connect "Initiate Convo with Follower" output to this node.

8. **Add an HTTP Request node (named "Send DM"):**  
   - Configure POST request to send a direct message to the newly initiated conversation:  
     - Include message content (e.g., welcome message).  
     - Use conversation ID or follower ID from the Filter node output.  
     - Set authentication headers.  
   - Connect "Filter" node output to this node.

9. **Connect nodes according to the sequence:**  
   - Cron Trigger → Setup → Create Session → Get Follow Notifications → Filter New Follows → Initiate Convo with Follower → Filter → Send DM.

10. **Credential Setup:**  
    - Create and configure credentials for Bluesky API access (likely OAuth2 or API Key).  
    - Attach credentials to all HTTP Request nodes.

11. **Persistence for Filter New Follows:**  
    - Implement a mechanism outside the workflow or via n8n’s data storage to track which followers have already been messaged, to avoid duplicate messages.

12. **Optional: Add Sticky Notes in the n8n editor** to document each block as per naming conventions in this workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                        |
|------------------------------------------------------------------------------------------------------------------------------|--------------------------------------|
| The workflow requires valid Bluesky API credentials and permission to send DMs and read notifications.                       | Bluesky API Documentation (not linked) |
| Consider implementing error handling and retry logic for HTTP requests to handle rate limiting and transient failures.       | Standard API best practices           |
| Persistence of messaged follower IDs is crucial to avoid repetitive DMs; use external DB or n8n’s built-in data stores.       | Data management best practices        |
| For scheduling, adjust Cron Trigger frequency to balance API limits and timely messaging.                                     | n8n Cron Trigger documentation        |

---

**Disclaimer:** The provided content is derived solely from an automated workflow created with n8n, respecting all content policies and containing no illegal or offensive materials. All data handled are legal and public.