Daily Competitor Tweet Summarizer with X API, GPT-5-Nano, and Gmail Delivery

https://n8nworkflows.xyz/workflows/daily-competitor-tweet-summarizer-with-x-api--gpt-5-nano--and-gmail-delivery-9355


# Daily Competitor Tweet Summarizer with X API, GPT-5-Nano, and Gmail Delivery

---

### 1. Workflow Overview

This workflow automates the daily summarization of a competitor's recent tweets on X (formerly Twitter) using the X API, leverages GPT-5-Nano via OpenAI for AI-driven content analysis, and sends the resulting summary via Gmail email delivery. It is designed for digital marketing analysts, social media managers, or competitive intelligence teams who want automated, daily insights into competitor activity on social media.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow automatically based on a daily schedule.
- **1.2 Competitor User Data Retrieval:** Fetches the competitor’s user profile data from X to obtain the user ID.
- **1.3 Recent Tweets Fetching:** Retrieves the competitor’s most recent tweets using the user ID.
- **1.4 AI-Powered Analysis:** Sends tweet content and engagement metrics to GPT-5-Nano to create a summarized competitor analysis.
- **1.5 Email Delivery:** Sends the generated summary via Gmail to a specified email address.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block initiates the workflow automatically on a daily schedule to keep the competitor analysis up-to-date without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Node Name:** Schedule Trigger  
  - **Type:** `n8n-nodes-base.scheduleTrigger`  
  - **Role:** Triggers the workflow execution on a recurring schedule.  
  - **Configuration:**  
    - Interval configured to run daily at midnight (default daily interval).  
  - **Input/Output:**  
    - No input; output connects to the "Get User" node.  
  - **Edge Cases / Failures:**  
    - Misconfigured scheduling can cause missed runs or excessive triggering.  
    - Workflow inactive status disables triggering.  
  - **Sticky Note Content:**  
    - "Step 1: Set your preferred schedule. The task is currently set to run every day at midnight."

---

#### 1.2 Competitor User Data Retrieval

- **Overview:**  
  This block retrieves the competitor’s user profile from X by username to extract the user ID, which is necessary for fetching tweets.

- **Nodes Involved:**  
  - Get User

- **Node Details:**  
  - **Node Name:** Get User  
  - **Type:** `n8n-nodes-base.twitter` (Twitter OAuth2 API)  
  - **Role:** Fetches basic profile information, including user ID by username.  
  - **Configuration:**  
    - Resource: "user"  
    - Mode: "username"  
    - Username: hardcoded competitor’s X username (configured in parameters).  
    - OAuth2 credentials: linked to an X account for authenticated access.  
  - **Input/Output:**  
    - Input: triggered by Schedule Trigger.  
    - Output: user profile JSON object passed to "Fetch Recent Posts."  
  - **Edge Cases / Failures:**  
    - Invalid username or suspended account results in empty or error response.  
    - API rate limits or authentication failures.  
  - **Sticky Note Content:**  
    - "Step 2: Hardcode your competitor's X (formerly Twitter) username in this node. This node will fetch basic profile data to extract the corresponding user ID."

---

#### 1.3 Recent Tweets Fetching

- **Overview:**  
  Fetches the five latest tweets from the competitor’s timeline using the user ID obtained previously, including engagement metrics.

- **Nodes Involved:**  
  - Fetch Recent Posts

- **Node Details:**  
  - **Node Name:** Fetch Recent Posts  
  - **Type:** `n8n-nodes-base.httpRequest`  
  - **Role:** Calls the X API’s user tweets endpoint to fetch recent tweets with public metrics.  
  - **Configuration:**  
    - HTTP Method: GET  
    - URL: `https://api.twitter.com/2/users/{{ $json.id }}/tweets` (dynamic user ID insertion)  
    - Query Parameters:  
      - `max_results=5` (number of tweets to fetch)  
      - `tweet.fields=public_metrics,created_at` (includes likes, impressions, creation time)  
    - Authentication: Bearer Token OAuth2 (predefined credentials)  
  - **Input/Output:**  
    - Input: receives user ID JSON from "Get User" node.  
    - Output: tweet data array passed to "Message a model."  
  - **Edge Cases / Failures:**  
    - API rate limits or authorization errors.  
    - User with no tweets or protected tweets results in empty data.  
    - Network timeouts or malformed responses.  
  - **Sticky Note Content:**  
    - "Step 3: This node will fetch the 5 most recent posts from your competitor. You can change this number by modifying the `max_results` parameter in this node."

---

#### 1.4 AI-Powered Analysis

- **Overview:**  
  Sends the recent tweets with their engagement metrics to OpenAI's GPT-5-Nano model to generate a concise competitor analysis summary.

- **Nodes Involved:**  
  - Message a model

- **Node Details:**  
  - **Node Name:** Message a model  
  - **Type:** `@n8n/n8n-nodes-langchain.openAi`  
  - **Role:** Uses OpenAI API to process tweet data and generate a textual summary.  
  - **Configuration:**  
    - Model ID: `gpt-5-nano`  
    - Prompt:  
      - Dynamically constructs a prompt by mapping over the tweet data array to list each tweet’s text, likes, and impressions.  
      - Prompt instructs the model to analyze recent competitor tweets considering updates and engagement rate.  
    - No additional OpenAI options specified.  
    - Credentials: OpenAI API key stored securely and referenced.  
  - **Input/Output:**  
    - Input: tweet data JSON from "Fetch Recent Posts."  
    - Output: AI-generated summary message JSON passed to "Send a message."  
  - **Edge Cases / Failures:**  
    - OpenAI API quota exhaustion or authentication failure.  
    - Prompt construction errors if input data is malformed.  
    - Latency or timeout issues.  
  - **Sticky Note Content:**  
    - "Step 4: This OpenAI node uses the `gpt-5-nano` model to generate a competitor analysis summary based on recent posts. You can tailor the summary to your needs by modifying the prompt."

---

#### 1.5 Email Delivery

- **Overview:**  
  Sends the AI-generated competitor analysis summary to the user’s email address via Gmail.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**  
  - **Node Name:** Send a message  
  - **Type:** `n8n-nodes-base.gmail`  
  - **Role:** Sends an email with the generated summary as the message body.  
  - **Configuration:**  
    - Recipient email: hardcoded (e.g., `your-email@example.com`) – must be updated by the user.  
    - Subject: "Latest Competitor Analysis"  
    - Message body: populated dynamically from the AI-generated summary content.  
    - Email type: Plain text.  
    - Credentials: OAuth2 linked Gmail account for secure sending.  
  - **Input/Output:**  
    - Input: receives AI model output with summary content.  
    - Output: terminal node (no further output).  
  - **Edge Cases / Failures:**  
    - Gmail OAuth2 authentication failures or revoked permissions.  
    - Email delivery issues such as invalid recipient address.  
    - API rate limits or quota exceeded.  
  - **Sticky Note Content:**  
    - "Final Step: Send an email to yourself or your team by updating the email address in this node."

---

### 3. Summary Table

| Node Name         | Node Type                        | Functional Role                  | Input Node(s)       | Output Node(s)       | Sticky Note                                             |
|-------------------|---------------------------------|---------------------------------|---------------------|----------------------|---------------------------------------------------------|
| Schedule Trigger  | n8n-nodes-base.scheduleTrigger  | Workflow initiation by schedule | None                | Get User             | Step 1: Set your preferred schedule. The task is currently set to run every day at midnight. |
| Get User          | n8n-nodes-base.twitter          | Fetch competitor user profile   | Schedule Trigger    | Fetch Recent Posts    | Step 2: Hardcode your competitor's X username in this node. This node will fetch basic profile data to extract the corresponding user ID. |
| Fetch Recent Posts| n8n-nodes-base.httpRequest      | Fetch competitor recent tweets  | Get User            | Message a model       | Step 3: Fetches 5 most recent posts. Change `max_results` to modify count.                 |
| Message a model   | @n8n/n8n-nodes-langchain.openAi| Generate AI summary of tweets   | Fetch Recent Posts  | Send a message        | Step 4: Uses GPT-5-Nano to generate competitor analysis summary; customize prompt as needed. |
| Send a message    | n8n-nodes-base.gmail            | Email the AI-generated summary  | Message a model     | None                 | Final Step: Send email by updating recipient address.                                 |
| Sticky Note       | n8n-nodes-base.stickyNote       | Documentation / notes            | None                | None                 | Multiple notes accompany steps 1–4 and final step.                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: `Schedule Trigger`  
   - Configure interval to run daily at midnight (default).  
   - No credentials needed.  

2. **Create Twitter "Get User" Node**  
   - Type: `Twitter` node (OAuth2)  
   - Set resource to "user" and mode to "username."  
   - Hardcode competitor’s X username in the username field.  
   - Connect output from Schedule Trigger to this node's input.  
   - Set Twitter OAuth2 credentials linked to an X developer account.  

3. **Create HTTP Request Node "Fetch Recent Posts"**  
   - Type: `HTTP Request`  
   - Method: GET  
   - URL: `https://api.twitter.com/2/users/{{ $json.id }}/tweets` (dynamic user ID from previous node)  
   - Query parameters:  
     - `max_results=5` (or desired number of tweets)  
     - `tweet.fields=public_metrics,created_at`  
   - Authentication: Bearer Token (OAuth2) with valid token.  
   - Connect output of "Get User" node to this node.  

4. **Create OpenAI Node "Message a model"**  
   - Type: `OpenAI` from Langchain nodes  
   - Model: `gpt-5-nano`  
   - Message content:  
     ```
     Analyze the recent competitor tweets on X, considering their recent updates and engagement rate.

     {{ $json.data.map(t => `Tweet: ${t.text}\nLikes: ${t.public_metrics.like_count}, Impressions: ${t.public_metrics.impression_count}`).join('\n\n') }}
     ```  
   - Connect output of "Fetch Recent Posts" to this node.  
   - Set OpenAI API credentials.  

5. **Create Gmail Node "Send a message"**  
   - Type: `Gmail` node  
   - Set recipient email address (e.g., your personal/work email).  
   - Subject: "Latest Competitor Analysis"  
   - Message body: `={{ $json.message.content }}` (dynamic content from AI output)  
   - Connect output of OpenAI node to this node.  
   - Configure Gmail OAuth2 credentials for sending email.  

6. **Verify Connections & Activate Workflow**  
   - Ensure nodes are connected in this order: Schedule Trigger → Get User → Fetch Recent Posts → Message a model → Send a message.  
   - Activate the workflow to enable scheduled runs.  

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow is designed for daily execution but scheduling can be adjusted as needed in Schedule Trigger. | Scheduling flexibility for different business needs.                                            |
| Prompt in OpenAI node is customizable to tailor the analysis focus or style.                          | Modify prompt content to change AI summary tone or details.                                     |
| Credentials must be set up before activating the workflow: Twitter OAuth2, OpenAI API key, Gmail OAuth2. | OAuth2 credentials require proper app registration and consent for API access.                   |
| The workflow uses X API v2 endpoint for tweets; ensure API access and permissions are granted.        | Twitter/X developer account required with appropriate API access level.                         |
| Consider API rate limits for Twitter and OpenAI when scaling or increasing frequency.                 | Monitor API usage to avoid throttling or service interruptions.                                |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---