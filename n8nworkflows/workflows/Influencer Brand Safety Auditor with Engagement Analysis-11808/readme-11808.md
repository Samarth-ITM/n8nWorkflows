Influencer Brand Safety Auditor with Engagement Analysis

https://n8nworkflows.xyz/workflows/influencer-brand-safety-auditor-with-engagement-analysis-11808


# Influencer Brand Safety Auditor with Engagement Analysis

### 1. Workflow Overview

This workflow, titled **Influencer Brand Safety Auditor with Engagement Analysis**, is designed for vetting Instagram influencers to protect brand reputation and advertising budgets. It targets influencer marketing managers, agencies, and brand managers who need to assess potential partners for suspicious engagement patterns (like fake followers or bots) and brand safety risks in content.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Collect influencer Instagram username and optional competitor brand names via a user form.
- **1.2 Configuration Setup:** Define API tokens, limits, and threshold parameters for engagement analysis.
- **1.3 Data Acquisition:** Scrape Instagram profile data and recent posts using Apify Instagram Scraper API.
- **1.4 Engagement Analysis:** Calculate average engagement rate and classify profile health.
- **1.5 Content Aggregation:** Aggregate post captions and media URLs for AI evaluation.
- **1.6 AI Content Safety Audit:** Use OpenAI to analyze aggregated content for risk flags, competitor mentions, and generate safety scores.
- **1.7 Reporting and Storage:** Send detailed audit reports to Slack and log requests and results in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Collects the Instagram username and optional competitor brand names from the user via a web form.

**Nodes Involved:**  
- Influencer Audit Form

**Node Details:**

- **Influencer Audit Form**  
  - Type: Form Trigger node  
  - Role: Initiates workflow with user input  
  - Configuration:  
    - Form title: "Influencer Brand Safety Audit"  
    - Fields: Mandatory "Instagram Username", optional "Competitor Brand Names" (comma-separated)  
    - No automatic attribution appended  
  - Connections: Outputs to "Workflow Configuration"  
  - Edge cases: Invalid or missing username input; form submission timeout or user cancellation  

---

#### 2.2 Configuration Setup

**Overview:**  
Sets workflow parameters such as API tokens and thresholds used throughout the workflow.

**Nodes Involved:**  
- Workflow Configuration

**Node Details:**

- **Workflow Configuration**  
  - Type: Set node  
  - Role: Defines key parameters for API access and thresholds  
  - Configuration:  
    - `apifyApiToken` (string): Placeholder for Apify API token needed for Instagram scraping  
    - `resultsLimit` (number): Maximum posts to fetch (default 30)  
    - `engagementThresholdLow` (number): Lower engagement rate threshold (default 1%)  
    - `engagementThresholdHigh` (number): Upper engagement rate threshold (default 10%)  
  - Connections: Outputs to "Store Audit Request"  
  - Edge cases: Missing or invalid API token causing authentication failure  

---

#### 2.3 Data Acquisition

**Overview:**  
Stores the audit request in Google Sheets and fetches Instagram profile data and posts using Apify.

**Nodes Involved:**  
- Store Audit Request  
- Apify Instagram Scraper

**Node Details:**

- **Store Audit Request**  
  - Type: Google Sheets node  
  - Role: Logs incoming audit requests with username into "Audit Requests" sheet  
  - Configuration:  
    - Operation: Append or update, matching by username  
    - Document ID and Sheet name configured (placeholders)  
    - Credential: Google Sheets OAuth2  
  - Connections: Outputs to "Apify Instagram Scraper"  
  - Edge cases: Google Sheets API quota exceeded, permission errors  

- **Apify Instagram Scraper**  
  - Type: HTTP Request node  
  - Role: Calls Apify Instagram Scraper API to fetch user posts  
  - Configuration:  
    - Method: POST  
    - URL: Dynamic with Apify token from configuration  
    - Body JSON: Direct URL of Instagram user profile, post limit, results type = posts, user search type, Apify proxy enabled  
  - Connections: Outputs to "Calculate Engagement Rate"  
  - Edge cases: API timeout (waitForFinish=300 seconds), token invalid, no posts found, network errors  

---

#### 2.4 Engagement Analysis

**Overview:**  
Calculates average engagement rate from fetched posts and classifies account health status.

**Nodes Involved:**  
- Calculate Engagement Rate

**Node Details:**

- **Calculate Engagement Rate**  
  - Type: Code node (JavaScript)  
  - Role: Processes Apify data to compute engagement metrics and flags suspicious behavior  
  - Logic:  
    - Extracts posts array; if empty returns error object  
    - Retrieves follower count and username from first post  
    - Calculates total likes and comments across posts  
    - Computes average engagement and engagement rate (%) relative to followers  
    - Flags "Suspicious" if engagement rate below low threshold (fake followers) or above high threshold (possible bots, for accounts with >10k followers)  
  - Connections: Outputs enriched post data to "Aggregate Captions"  
  - Edge cases: No posts returned, missing follower counts, division by zero handled by defaulting followers to 1, malformed API response  

---

#### 2.5 Content Aggregation

**Overview:**  
Aggregates captions and media URLs of fetched posts for AI content safety evaluation.

**Nodes Involved:**  
- Aggregate Captions

**Node Details:**

- **Aggregate Captions**  
  - Type: Aggregate node  
  - Role: Combines post captions and display URLs into a single dataset  
  - Configuration:  
    - Aggregates on fields: caption, displayUrl  
  - Connections: Outputs to "AI Content Safety Audit"  
  - Edge cases: Posts with empty captions or no media URLs, malformed data fields  

---

#### 2.6 AI Content Safety Audit

**Overview:**  
Sends aggregated content to OpenAI via LangChain node to analyze brand safety risks and competitor mentions.

**Nodes Involved:**  
- AI Content Safety Audit

**Node Details:**

- **AI Content Safety Audit**  
  - Type: LangChain OpenAI node  
  - Role: AI-powered content evaluation  
  - Configuration:  
    - Operation: Message-based interaction  
    - Inputs: Aggregated captions and URLs from previous node  
  - Outputs: JSON fields including safety_score, risk_flags array, competitor_check, engagement_assessment, content_summary, recommendation  
  - Connections: Outputs to "Send Audit Report to Slack"  
  - Edge cases: OpenAI API quota limits, latency or timeout, malformed input, incomplete analysis results  
  - Version-specific: Requires n8n with LangChain OpenAI integration support  

---

#### 2.7 Reporting and Storage

**Overview:**  
Delivers the audit results to Slack and logs detailed audit results in Google Sheets.

**Nodes Involved:**  
- Send Audit Report to Slack  
- Store Audit Results

**Node Details:**

- **Send Audit Report to Slack**  
  - Type: Slack node  
  - Role: Sends formatted audit report message  
  - Configuration:  
    - Message text includes influencer username, numerical engagement data, AI safety assessment, content summary, and recommendation  
    - Sends to specified Slack channel (ID placeholder)  
    - Authenticated via Slack OAuth2 credentials  
  - Connections: Outputs to "Store Audit Results"  
  - Edge cases: Slack API rate limits, invalid channel ID, auth token expiration  

- **Store Audit Results**  
  - Type: Google Sheets node  
  - Role: Logs full audit results in "Audit Results" sheet  
  - Configuration:  
    - Operation: Append all data fields automatically mapped  
    - Document ID and sheet name configured (placeholders)  
    - Credential: Google Sheets OAuth2  
  - Edge cases: Google Sheets API quota or permission issues  

---

### 3. Summary Table

| Node Name                | Node Type                   | Functional Role                      | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                                           |
|--------------------------|-----------------------------|------------------------------------|----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Influencer Audit Form     | Form Trigger                | Input reception                    |                            | Workflow Configuration       | 1.  **Input:** Takes an Instagram username and optional competitor names via an **n8n Form**.                         |
| Workflow Configuration    | Set                         | Setup parameters                   | Influencer Audit Form       | Store Audit Request          |                                                                                                                       |
| Store Audit Request       | Google Sheets               | Log audit request                  | Workflow Configuration      | Apify Instagram Scraper      |                                                                                                                       |
| Apify Instagram Scraper   | HTTP Request                | Fetch Instagram data               | Store Audit Request         | Calculate Engagement Rate    | 2.  **Scraping:** Uses **Apify** to fetch the influencer's profile details and their most recent 30 posts.            |
| Calculate Engagement Rate | Code                        | Analyze engagement rate            | Apify Instagram Scraper     | Aggregate Captions           | 3.  ** engagement Analysis:** Calculates the average engagement rate. It flags the account as \"Suspicious\" if...     |
| Aggregate Captions        | Aggregate                   | Aggregate post captions & URLs    | Calculate Engagement Rate   | AI Content Safety Audit      |                                                                                                                       |
| AI Content Safety Audit   | LangChain OpenAI            | AI analysis of content safety     | Aggregate Captions          | Send Audit Report to Slack   | 4.  **AI Safety Check:** Aggregates recent post captions and sends them to **OpenAI** for risk and competitor analysis. |
| Send Audit Report to Slack| Slack                       | Send audit report to Slack        | AI Content Safety Audit     | Store Audit Results          | 5.  **Reporting:** Sends a detailed audit report to **Slack** and logs the results in **Google Sheets**.              |
| Store Audit Results       | Google Sheets               | Log audit results                 | Send Audit Report to Slack  |                             |                                                                                                                       |
| Sticky Note              | Sticky Note                 | Overview and user guidance         |                            |                             | ## Overview\nProtect your brand reputation and budget by automatically vetting potential influencer partners...       |
| Sticky Note1             | Sticky Note                 | Input explanation                  |                            |                             | 1.  **Input:** Takes an Instagram username and optional competitor names via an **n8n Form**.                         |
| Sticky Note2             | Sticky Note                 | Scraping explanation              |                            |                             | 2.  **Scraping:** Uses **Apify** to fetch the influencer's profile details and their most recent 30 posts.            |
| Sticky Note3             | Sticky Note                 | Engagement analysis explanation   |                            |                             | 3.  ** engagement Analysis:** Calculates the average engagement rate...                                              |
| Sticky Note4             | Sticky Note                 | AI safety check explanation       |                            |                             | 4.  **AI Safety Check:** Aggregates recent post captions and sends them to **OpenAI**...                               |
| Sticky Note5             | Sticky Note                 | Reporting explanation             |                            |                             | 5.  **Reporting:** Sends a detailed audit report to **Slack** and logs the results in **Google Sheets**.              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named **Influencer Audit Form**:  
   - Set form title: "Influencer Brand Safety Audit"  
   - Add two fields:  
     - "Instagram Username" (required)  
     - "Competitor Brand Names (optional, comma-separated)"  
   - No form attribution appended  
   - Position it as the workflow start  

2. **Add a Set node** named **Workflow Configuration**:  
   - Define variables:  
     - `apifyApiToken` (string) with your Apify API token  
     - `resultsLimit` (number), default 30  
     - `engagementThresholdLow` (number), default 1  
     - `engagementThresholdHigh` (number), default 10  
   - Connect **Influencer Audit Form** to this node  

3. **Add a Google Sheets node** named **Store Audit Request**:  
   - Operation: Append or Update, match by "username" column  
   - Set target Google Sheet document ID and sheet name "Audit Requests"  
   - Use OAuth2 credentials for Google Sheets  
   - Connect **Workflow Configuration** to this node  

4. **Add an HTTP Request node** named **Apify Instagram Scraper**:  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/apify~instagram-scraper/runs?token={{ $json.apifyApiToken }}&waitForFinish=300`  
   - Body (JSON):  
     ```json
     {
       "directUrls": ["https://www.instagram.com/{{ $json.username }}/"],
       "resultsLimit": {{ $json.resultsLimit }},
       "resultsType": "posts",
       "searchType": "user",
       "proxy": { "useApifyProxy": true }
     }
     ```  
   - Connect **Store Audit Request** to this node  

5. **Add a Code node** named **Calculate Engagement Rate**:  
   - Paste JavaScript code that:  
     - Extracts posts data  
     - Calculates total/average engagement (likes + comments)  
     - Computes engagement rate relative to followers  
     - Flags status as "Healthy", "Suspicious (Low Engagement)", or "Suspicious (Unusually High)" based on thresholds  
   - Connect **Apify Instagram Scraper** to this node  

6. **Add an Aggregate node** named **Aggregate Captions**:  
   - Aggregate fields: "caption" and "displayUrl" from posts  
   - Connect **Calculate Engagement Rate** to this node  

7. **Add an OpenAI LangChain node** named **AI Content Safety Audit**:  
   - Operation: Message  
   - Input the aggregated captions and media URLs  
   - Requires OpenAI API credentials configured in n8n  
   - Connect **Aggregate Captions** to this node  

8. **Add a Slack node** named **Send Audit Report to Slack**:  
   - Compose a detailed message with:  
     - Influencer username, followers, engagement rate, average engagement, health status  
     - AI safety score, risk flags, competitor mentions, engagement assessment, content summary, and recommendation  
   - Set Slack channel ID and use OAuth2 Slack credentials  
   - Connect **AI Content Safety Audit** to this node  

9. **Add a Google Sheets node** named **Store Audit Results**:  
   - Operation: Append all audit result data to sheet "Audit Results"  
   - Use Google Sheets OAuth2 credentials  
   - Connect **Send Audit Report to Slack** to this node  

10. **(Optional) Add Sticky Note nodes** at key points to document the workflow for users.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Protect your brand reputation and budget by automatically vetting potential influencer partners using this workflow. | Overview sticky note at workflow start                                                                           |
| Workflow uses Apify Instagram Scraper API: https://apify.com/instagram-scraper                                  | Apify API documentation                                                                                           |
| OpenAI integration via LangChain node requires OpenAI API key and n8n LangChain node support                   | https://platform.openai.com/docs/api-reference                                                                    |
| Slack OAuth2 authentication setup required to send messages to workspace channels                              | https://api.slack.com/authentication/oauth-v2                                                                     |
| Google Sheets OAuth2 credentials must have write permissions for configured spreadsheets                       |                                                                                                                  |
| Engagement thresholds can be adjusted in Workflow Configuration node for tuning sensitivity                    |                                                                                                                  |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. It adheres strictly to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.