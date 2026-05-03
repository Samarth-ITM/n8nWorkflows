Monitor Instagram Competitor Trends with Claude 3.5 & Multi-Channel Alerts

https://n8nworkflows.xyz/workflows/monitor-instagram-competitor-trends-with-claude-3-5---multi-channel-alerts-10341


# Monitor Instagram Competitor Trends with Claude 3.5 & Multi-Channel Alerts

### 1. Workflow Overview

This workflow automates monitoring of Instagram competitors’ performance and trends using AI-powered analysis and multi-channel alerting. It is designed for marketing, social media managers, or competitive intelligence teams aiming to track competitor Instagram engagement metrics and derive actionable strategic insights automatically.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Input Trigger & Competitor Loading**: Periodically triggers the workflow and loads the competitor list from a Google Sheet.
- **1.2 Competitor Data Retrieval Loop**: Iterates over each competitor to fetch their recent Instagram posts via the Instagram Graph API.
- **1.3 Performance Metric Calculation**: Computes average engagement and classifies engagement trends from the retrieved posts.
- **1.4 AI-Powered Strategic Insight Generation**: Sends the engagement data to Claude 3.5 AI to generate three strategic bullet-point insights.
- **1.5 Multi-Channel Alerts & Logging**: Sends alerts via email and WhatsApp, and logs the alert details into a Google Sheet.
- **1.6 Workflow Termination**: Ends the execution cleanly.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Input Trigger & Competitor Loading

**Overview:**  
This block initiates the workflow at scheduled intervals and loads the list of Instagram competitors from a Google Sheet for processing.

**Nodes Involved:**  
- Schedule Trigger  
- Get Competitor List  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Cron Trigger  
  - Role: Initiates workflow daily at 10 AM (implied by sticky note) or via webhook `/competitor-alert` (not explicitly shown but mentioned in sticky note)  
  - Configuration: Default cron, no parameters shown (assumed daily schedule)  
  - Input: None (trigger node)  
  - Output: Triggers "Get Competitor List"  
  - Edge Cases: Cron misfires, time zone mismatches  
  - Sticky Note: “Runs daily at 10 AM or via webhook (/competitor-alert)”  

- **Get Competitor List**  
  - Type: Google Sheets node  
  - Role: Reads competitors’ data (columns A to C) from a Google Sheet named “your-competitors-sheet-id”  
  - Configuration: Reads range `A:C` to get competitor names and Instagram user IDs  
  - Credentials: Google API OAuth configured  
  - Input: Trigger output  
  - Output: Sends data to "Loop Over Competitors"  
  - Edge Cases: Invalid sheet ID, API rate limits, empty or malformed data  
  - Sticky Note: “Loads competitors from Competitors sheet”  

#### 2.2 Competitor Data Retrieval Loop

**Overview:**  
Processes each competitor one by one (batch size 1) to avoid API rate limits. For each competitor, retrieves the last 10 Instagram posts.

**Nodes Involved:**  
- Loop Over Competitors  
- Get Competitor Posts  

**Node Details:**  

- **Loop Over Competitors**  
  - Type: SplitInBatches  
  - Role: Processes competitors individually in batches of 1 to throttle API calls  
  - Configuration: Batch size = 1  
  - Input: Competitor list from previous node  
  - Output: Feeds single competitor item to "Get Competitor Posts"  
  - Edge Cases: Batch size misconfiguration could cause performance issues  
  - Sticky Note: “Processes each competitor to avoid API limits”  

- **Get Competitor Posts**  
  - Type: HTTP Request  
  - Role: Calls Instagram Graph API to fetch last 10 media posts for the competitor’s Instagram user ID  
  - Configuration:  
    - URL constructed dynamically using `{{ $json["competitorUserId"] }}`  
    - Fields retrieved: id, media_type, media_url, like_count, comments_count, caption, timestamp  
    - Access token must be replaced with a valid Instagram Access Token  
    - Limit set to 10 posts  
  - Input: Single competitor data from SplitInBatches  
  - Output: JSON array of posts to "Calculate Performance Metrics"  
  - Edge Cases: Invalid or expired access token, API rate limits, private or invalid user ID, no posts found  
  - Sticky Note: “Fetches last 10 posts via Instagram Graph API”  

#### 2.3 Performance Metric Calculation

**Overview:**  
Calculates the average engagement (likes + comments per post) and determines a simple engagement trend ("Rising" if avg engagement > 50, else "Stable").

**Nodes Involved:**  
- Calculate Performance Metrics  

**Node Details:**  

- **Calculate Performance Metrics**  
  - Type: Code (JavaScript)  
  - Role: Processes the Instagram posts array to compute average engagement and classify trend  
  - Configuration:  
    - Sums likes and comments over all posts  
    - Average engagement = (totalLikes + totalComments) / number of posts  
    - Trend determination: threshold at 50 average engagement  
    - Also forwards competitor name for downstream use  
  - Input: Instagram posts data from HTTP Request node  
  - Output: JSON with `avgEngagement`, `trend`, and `competitorName` to AI insights node  
  - Edge Cases: No posts (division by zero handled), missing data fields, malformed JSON  
  - Sticky Note: “Computes avg engagement and trend using Code node”  

#### 2.4 AI-Powered Strategic Insight Generation

**Overview:**  
Sends the engagement data and competitor info to Anthropic Claude 3.5 API to generate strategic insights as bullet points, focusing on content strategy, posting frequency, and engagement tactics.

**Nodes Involved:**  
- Generate AI Insights (Claude AI)  

**Node Details:**  

- **Generate AI Insights (Claude AI)**  
  - Type: HTTP Request  
  - Role: Calls Claude 3.5 AI endpoint for natural language analysis and insight generation  
  - Configuration:  
    - POST request to `https://api.anthropic.com/v1/messages`  
    - Model: `claude-3-5-sonnet-20241022`  
    - Max tokens: 1024  
    - Message content includes serialized JSON data of competitor metrics  
    - Prompt requests 3 strategic insights as bullet points focusing on content strategy, posting frequency, and engagement tactics  
  - Input: Metrics JSON from Code node  
  - Output: AI-generated text insights to multi-channel alert nodes  
  - Edge Cases: API authentication errors, rate limits, malformed prompt, incomplete AI response  
  - Sticky Note: “Analyzes data for 3 strategic bullet-point insights”  

#### 2.5 Multi-Channel Alerts & Logging

**Overview:**  
Distributes the AI insights and engagement metrics via email and WhatsApp alerts, and logs the alert details into a Google Sheet for record-keeping.

**Nodes Involved:**  
- Send Email Alert  
- Send WhatsApp Alert (Twilio)  
- Log Alert  

**Node Details:**  

- **Send Email Alert**  
  - Type: Email Send  
  - Role: Sends a detailed email report to the internal team  
  - Configuration:  
    - Subject dynamically includes competitor name and engagement trend  
    - Recipient: `alerts@yourcompany.com`  
    - Sender: `your-email@domain.com`  
    - SMTP credentials required (not provided)  
  - Input: AI insights and metrics JSON  
  - Output: Terminates at "End Workflow"  
  - Edge Cases: SMTP authentication failure, invalid email addresses, email sending delays  
  - Sticky Note: “Emails detailed report to team” (also applies to WhatsApp and Log Alert nodes)  

- **Send WhatsApp Alert (Twilio)**  
  - Type: Twilio node  
  - Role: Sends a concise WhatsApp message alert with competitor engagement and a snippet of AI insights  
  - Configuration:  
    - To and From phone numbers provided (must be Twilio verified)  
    - Message includes competitor name, average engagement, trend, and first 100 characters of AI insights  
    - Twilio API credentials configured  
  - Input: AI insights and metrics JSON  
  - Output: Terminates at "End Workflow"  
  - Edge Cases: Twilio API quota limits, invalid phone numbers, message truncation issues  
  - Sticky Note: “Sends concise alert via WhatsApp”  

- **Log Alert**  
  - Type: Google Sheets node  
  - Role: Appends alert data (metrics, insights, timestamps) to a Google Sheet log for historical tracking  
  - Configuration:  
    - Target sheet range: `your-alerts-log-sheet-id!A:E`  
    - Operation: Append row  
    - Uses Google API credentials  
  - Input: AI insights and metrics JSON  
  - Output: Terminates at "End Workflow"  
  - Edge Cases: Google Sheets API quota exceeded, malformed data, network issues  
  - Sticky Note: “Records metrics and insights in AlertsLog sheet”  

#### 2.6 Workflow Termination

**Overview:**  
A Set node with no data used to explicitly end the workflow after all alerting and logging tasks complete.

**Nodes Involved:**  
- End Workflow  

**Node Details:**  

- **End Workflow**  
  - Type: Set node  
  - Role: Terminates workflow execution cleanly  
  - Configuration: Keeps no data, acts as a final node  
  - Input: From any alert/log node  
  - Output: None  
  - Edge Cases: None (safe termination)  
  - Sticky Note: “Terminates execution”  

---

### 3. Summary Table

| Node Name                 | Node Type         | Functional Role                         | Input Node(s)            | Output Node(s)                   | Sticky Note                                                                                  |
|---------------------------|-------------------|---------------------------------------|--------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger          | Cron Trigger      | Initiates workflow daily               | None                     | Get Competitor List              | Runs daily at 10 AM or via webhook (/competitor-alert)                                       |
| Get Competitor List       | Google Sheets     | Loads competitors from Google Sheet   | Schedule Trigger         | Loop Over Competitors            | Loads competitors from Competitors sheet                                                     |
| Loop Over Competitors     | SplitInBatches    | Processes competitors one by one      | Get Competitor List      | Get Competitor Posts             | Processes each competitor to avoid API limits                                               |
| Get Competitor Posts      | HTTP Request      | Fetches last 10 Instagram posts       | Loop Over Competitors    | Calculate Performance Metrics    | Fetches last 10 posts via Instagram Graph API                                               |
| Calculate Performance Metrics | Code           | Calculates avg engagement and trend   | Get Competitor Posts     | Generate AI Insights (Claude AI) | Computes avg engagement and trend using Code node                                           |
| Generate AI Insights (Claude AI) | HTTP Request | Generates strategic insights via AI   | Calculate Performance Metrics | Send Email Alert, Send WhatsApp Alert (Twilio), Log Alert | Analyzes data for 3 strategic bullet-point insights                                         |
| Send Email Alert          | Email Send        | Sends detailed email alert             | Generate AI Insights     | End Workflow                    | Emails detailed report to team                                                               |
| Send WhatsApp Alert (Twilio) | Twilio         | Sends concise WhatsApp alert           | Generate AI Insights     | End Workflow                    | Sends concise alert via WhatsApp                                                            |
| Log Alert                 | Google Sheets     | Logs alert data into Google Sheet      | Generate AI Insights     | End Workflow                    | Records metrics and insights in AlertsLog sheet                                             |
| End Workflow              | Set               | Terminates workflow                    | Send Email Alert, Send WhatsApp Alert, Log Alert | None                          | Terminates execution                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Cron Trigger  
   - Set to run daily at 10 AM or configure webhook `/competitor-alert` (if applicable)  

2. **Create Google Sheets Node "Get Competitor List":**  
   - Operation: Read rows  
   - Sheet ID: Set your competitors Google Sheet ID and range `A:C`  
   - Credentials: Connect Google API OAuth credentials  

3. **Create SplitInBatches Node "Loop Over Competitors":**  
   - Set batch size to 1 to process competitors sequentially  
   - Connect input from "Get Competitor List"  

4. **Create HTTP Request Node "Get Competitor Posts":**  
   - Method: GET  
   - URL:  
     ```
     https://graph.facebook.com/v12.0/{{ $json["competitorUserId"] }}/media?fields=id,media_type,media_url,like_count,comments_count,caption,timestamp&access_token=YOUR_INSTAGRAM_ACCESS_TOKEN&limit=10
     ```  
   - Replace `YOUR_INSTAGRAM_ACCESS_TOKEN` with a valid token  
   - Connect input from "Loop Over Competitors"  

5. **Create Code Node "Calculate Performance Metrics":**  
   - Language: JavaScript  
   - Paste code:  
     ```js
     const posts = $json['data'] || [];
     let totalLikes = 0, totalComments = 0, totalPosts = posts.length;

     posts.forEach(post => {
       totalLikes += post.like_count || 0;
       totalComments += post.comments_count || 0;
     });

     const avgEngagement = totalPosts > 0 ? (totalLikes + totalComments) / totalPosts : 0;
     const trend = avgEngagement > 50 ? 'Rising' : 'Stable';

     return {
       json: {
         ...$json,
         avgEngagement,
         trend,
         competitorName: $input.item.json.competitorName
       }
     };
     ```  
   - Connect input from "Get Competitor Posts"  

6. **Create HTTP Request Node "Generate AI Insights (Claude AI)":**  
   - Method: POST  
   - URL: `https://api.anthropic.com/v1/messages`  
   - Headers: Add API key authentication as required by Anthropic API  
   - Body (JSON):  
     ```json
     {
       "model": "claude-3-5-sonnet-20241022",
       "max_tokens": 1024,
       "messages": [{
         "role": "user",
         "content": "Analyze these competitor Instagram trends: {{ JSON.stringify($json) }}. Provide 3 strategic insights to stay ahead, focusing on content strategy, posting frequency, and engagement tactics. Output as bullet points."
       }]
     }
     ```  
   - Enable "JSON Parameters"  
   - Connect input from "Calculate Performance Metrics"  

7. **Create Email Send Node "Send Email Alert":**  
   - Subject: `Competitor Trends Alert: {{ $node["Loop Over Competitors"].json["competitorName"] }} - {{ $json["trend"] }} Engagement`  
   - To: `alerts@yourcompany.com`  
   - From: Your verified sender email  
   - Credentials: Configure SMTP credentials  
   - Connect input from "Generate AI Insights (Claude AI)"  

8. **Create Twilio Node "Send WhatsApp Alert (Twilio)":**  
   - To: Recipient WhatsApp number (e.g., `+919282772672`)  
   - From: Your Twilio WhatsApp-enabled number (e.g., `+919988776677`)  
   - Message:  
     ```
     🚨 Competitor Alert: {{ $node["Loop Over Competitors"].json["competitorName"] }} | Engagement: {{ $json["avgEngagement"] }} | Trend: {{ $json["trend"] }}

     Insights:
     {{ $json["content"][0]["text"].substring(0, 100) }}...
     ```  
   - Credentials: Twilio API credentials  
   - Connect input from "Generate AI Insights (Claude AI)"  

9. **Create Google Sheets Node "Log Alert":**  
   - Operation: Append row  
   - Sheet ID: Your Alerts Log Google Sheet ID with range `A:E`  
   - Credentials: Google API OAuth credentials  
   - Connect input from "Generate AI Insights (Claude AI)"  

10. **Create Set Node "End Workflow":**  
    - No parameters needed, just a clean termination point  
    - Connect outputs from "Send Email Alert", "Send WhatsApp Alert (Twilio)", and "Log Alert" to this node  

11. **Connect all nodes as per the flow:**  
    - Schedule Trigger → Get Competitor List → Loop Over Competitors → Get Competitor Posts → Calculate Performance Metrics → Generate AI Insights → [Send Email Alert, Send WhatsApp Alert, Log Alert] → End Workflow  

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                 |
|----------------------------------------------------------------------------------------------------------|------------------------------------------------|
| Workflow runs daily at 10 AM or can be triggered manually via webhook `/competitor-alert`.                | Sticky Note near Schedule Trigger node          |
| Designed to process competitors one by one to avoid Instagram Graph API rate limits.                      | Sticky Note near Loop Over Competitors node     |
| Instagram Graph API requires a valid access token with appropriate permissions to fetch media data.      | Instagram Graph API documentation                |
| Claude 3.5 AI model used for advanced natural language insights generation—requires API key and quota.   | Anthropic API docs: https://www.anthropic.com/  |
| Alerts sent via email and WhatsApp; ensure SMTP and Twilio credentials are correctly configured.          | SMTP & Twilio official docs                       |
| Logs all alerts into Google Sheets for historical tracking and audit purposes.                            | Google Sheets API docs                            |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.