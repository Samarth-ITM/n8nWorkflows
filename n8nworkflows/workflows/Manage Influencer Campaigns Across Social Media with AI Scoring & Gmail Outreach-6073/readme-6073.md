Manage Influencer Campaigns Across Social Media with AI Scoring & Gmail Outreach

https://n8nworkflows.xyz/workflows/manage-influencer-campaigns-across-social-media-with-ai-scoring---gmail-outreach-6073


# Manage Influencer Campaigns Across Social Media with AI Scoring & Gmail Outreach

### 1. Workflow Overview

This workflow automates the management of influencer marketing campaigns across multiple social media platforms by leveraging AI-powered scoring and Gmail-based outreach. It is designed to intake campaign parameters, search and qualify influencers from Instagram, TikTok, and YouTube based on configurable criteria, generate personalized outreach emails, and manage follow-ups and tracking.

The workflow is logically divided into the following blocks:

- **1.1 Campaign Input & Configuration:** Receives campaign briefs via webhook and sets campaign parameters.
- **1.2 Influencer Discovery:** Searches for influencers on Instagram, TikTok, and YouTube using the target audience keywords.
- **1.3 Influencer Scoring & Qualification:** Aggregates influencer data, applies a scoring algorithm based on followers, engagement, platform preference, and bio relevance, and filters the top candidates.
- **1.4 Personalized Outreach Preparation:** Generates personalized email content and extracts or predicts influencer contact information.
- **1.5 Email Outreach & Follow-up:** Sends outreach emails, waits, and sends follow-up emails to influencers.
- **1.6 Campaign Tracking:** Logs influencer outreach and status to a Google Sheet for monitoring.

---

### 2. Block-by-Block Analysis

#### 2.1 Campaign Input & Configuration

**Overview:**  
This block receives the campaign brief via an HTTP webhook, then sets up internal campaign parameters such as budget, follower thresholds, engagement rate minimums, campaign name, and contact email.

**Nodes Involved:**  
- Campaign Brief Webhook  
- Campaign Settings  
- Sticky Note (Informational)

**Node Details:**

- **Campaign Brief Webhook**  
  - *Type:* Webhook (HTTP POST)  
  - *Role:* Entry point receiving campaign briefs with parameters like target audience, budget, and campaign name.  
  - *Configuration:* HTTP POST on path `/campaign-brief`. No authentication or additional options specified.  
  - *Inputs:* External HTTP POST request with JSON payload.  
  - *Outputs:* Passes received JSON to next node.  
  - *Edge Cases:* Missing or malformed JSON can cause failures; lacks validation.  

- **Campaign Settings**  
  - *Type:* Set  
  - *Role:* Extracts and sets campaign parameters into workflow variables for reuse.  
  - *Configuration:*  
    - `totalBudget`: from webhook `budget` field.  
    - `minFollowers`: 10,000 (hardcoded).  
    - `maxFollowers`: 500,000 (hardcoded).  
    - `minEngagementRate`: 3% (hardcoded).  
    - `campaignName`: from webhook `campaign_name`.  
    - `targetAudience`: from webhook `target_audience`.  
    - `brandEmail`: static email `partnerships@yourbrand.com`.  
  - *Inputs:* JSON from webhook.  
  - *Outputs:* JSON with campaign parameters for downstream nodes.  
  - *Edge Cases:* Assumes webhook fields exist; no fallback defaults if missing.

- **Sticky Note**  
  - *Type:* Sticky Note  
  - *Role:* Documentation for campaign setup parameters.  
  - *Content:* Lists key campaign config points: demographics, budget, content, timeline.

---

#### 2.2 Influencer Discovery

**Overview:**  
This block searches for influencers related to the target audience on Instagram, TikTok, and YouTube by invoking their respective platform APIs.

**Nodes Involved:**  
- Search Instagram Influencers  
- Search TikTok Influencers  
- Search YouTube Influencers

**Node Details:**

- **Search Instagram Influencers**  
  - *Type:* HTTP Request  
  - *Role:* Queries Instagram API user search endpoint with `targetAudience` as query.  
  - *Configuration:*  
    - URL: `https://api.instagram.com/v1/users/search`  
    - Query param `q`: target audience keywords  
    - Count: 50 results  
    - Authorization header uses Instagram OAuth token credential.  
  - *Inputs:* Campaign parameters including `targetAudience`.  
  - *Outputs:* JSON list of Instagram users matching search.  
  - *Edge Cases:*  
    - API rate limits or token expiration can cause failures.  
    - Search results may not include follower or engagement data; may require additional calls.  

- **Search TikTok Influencers**  
  - *Type:* HTTP Request  
  - *Role:* Queries TikTok API user search with target keywords.  
  - *Configuration:*  
    - URL: `https://api.tiktok.com/v1/users/search`  
    - Query params: `limit=50`, `keyword=targetAudience`  
    - Authorization header uses TikTok OAuth token credential.  
  - *Inputs:* Campaign target audience.  
  - *Outputs:* JSON list of TikTok users matching query.  
  - *Edge Cases:*  
    - TikTok API restrictions, possible changes in endpoints or tokens.  
    - Data completeness varies; no follower/engagement guarantees.  

- **Search YouTube Influencers**  
  - *Type:* HTTP Request  
  - *Role:* Uses YouTube Data API v3 to search channels relevant to the target audience.  
  - *Configuration:*  
    - URL: `https://api.youtube.com/v3/search`  
    - Query params: `q=targetAudience`, `type=channel`, `maxResults=50`  
    - Authorization header uses YouTube OAuth token credential.  
  - *Inputs:* Campaign target audience.  
  - *Outputs:* JSON list of channels with snippets and partial statistics.  
  - *Edge Cases:*  
    - Statistics might be incomplete or missing; requires additional API calls for full stats (not shown here).  
    - API quota limits and token expiration.

---

#### 2.3 Influencer Scoring & Qualification

**Overview:**  
Aggregates influencers from all platforms, normalizes data fields, applies a multi-factor scoring algorithm, filters by campaign thresholds, and selects the top 20 influencers.

**Nodes Involved:**  
- Score & Qualify Influencers  

**Node Details:**

- **Score & Qualify Influencers**  
  - *Type:* Code (JavaScript)  
  - *Role:* Combines influencer data from all platform searches, enriches with scoring based on followers (30%), engagement rate (40%), platform preference (20%), and bio keyword relevance (10%).  
  - *Configuration:*  
    - Accesses JSON outputs from Instagram, TikTok, and YouTube search nodes.  
    - Uses campaign parameters for thresholds and scoring.  
    - Calculates estimated cost per post differently by platform ($10, $8, $25 per 1k followers/subscribers).  
    - Includes cost per engagement metric.  
    - Assigns a unique campaign ID and timestamp.  
    - Returns top 20 scored influencers with full enriched data.  
  - *Inputs:* JSON arrays from search nodes and campaign settings.  
  - *Outputs:* Filtered and scored influencer array.  
  - *Edge Cases:*  
    - Missing follower or engagement data handled with defaults or zero, which may impact scoring accuracy.  
    - Date formatting and math operations assume valid numeric data.  
    - Risk of division by zero in cost per engagement calculation if engagement rate or follower count is zero.  
    - Assumes all API responses follow expected schema; otherwise may cause runtime errors.

---

#### 2.4 Personalized Outreach Preparation

**Overview:**  
Filters influencers with score ≥ 70, generates personalized outreach email content including subject lines, deliverables, campaign timeline, extracts or predicts email addresses for outreach.

**Nodes Involved:**  
- Filter Top Influencers  
- Generate Outreach Content  
- Extract Contact Info  

**Node Details:**

- **Filter Top Influencers**  
  - *Type:* If  
  - *Role:* Allows only influencers with score ≥ 70 to proceed.  
  - *Configuration:* Numeric condition on `$json.score >= 70`.  
  - *Inputs:* Scored influencer data array.  
  - *Outputs:* Qualified influencers only.  
  - *Edge Cases:* Influencers with missing or non-numeric scores will fail filter.  

- **Generate Outreach Content**  
  - *Type:* Code (JavaScript)  
  - *Role:* Creates personalized email subject lines, determines deliverables based on platform, calculates campaign start and end dates (2 weeks duration), and timestamps proposal send time.  
  - *Configuration:*  
    - Randomly selects from 4 subject line templates incorporating campaign and influencer data.  
    - Deliverables vary by platform (Instagram: posts, stories, reels; TikTok: video post, story updates; YouTube: sponsored video, community post).  
    - Sets campaign start as now, end 14 days later.  
  - *Inputs:* Influencer JSON, campaign settings.  
  - *Outputs:* JSON enriched with outreach subject, deliverables, campaign dates.  
  - *Edge Cases:* Date calculations assume valid Date object creation; no timezone handling.  

- **Extract Contact Info**  
  - *Type:* Code (JavaScript)  
  - *Role:* Attempts to find an email in influencer bio via regex; if none found, generates probable email addresses using common patterns and username.  
  - *Configuration:*  
    - Email regex to parse bio text.  
    - Generates fallback emails using username and domain heuristics (gmail, outlook, yahoo, custom domains).  
    - Marks `email_verified` flag true if found in bio.  
  - *Inputs:* Influencer JSON with bio and username.  
  - *Outputs:* Influencer JSON with email address and verification flag.  
  - *Edge Cases:*  
    - Regex may fail if bio is missing or malformed.  
    - Generated emails are guesses and may lead to invalid outreach attempts.  
    - No external email verification is performed.

---

#### 2.5 Email Outreach & Follow-up

**Overview:**  
Sends personalized outreach emails via Gmail, waits 3 days, then sends a follow-up email to non-responding influencers.

**Nodes Involved:**  
- Send Outreach Email  
- Wait 3 Days  
- Send Follow-up Email  
- Sticky Note1 (Informational)

**Node Details:**

- **Send Outreach Email**  
  - *Type:* Gmail (Send Email)  
  - *Role:* Sends the initial personalized outreach email to influencer's email address.  
  - *Configuration:*  
    - Sends to `{{ $json.email }}`  
    - HTML email body with styled content including campaign details, engagement stats, deliverables, compensation, and CTA mailto link to brand email.  
    - Subject uses dynamically generated outreach subject.  
    - Requires Gmail OAuth2 credentials with send email permissions.  
  - *Inputs:* Influencer JSON with email and outreach content.  
  - *Outputs:* Proceeds to wait node, also triggers campaign tracking.  
  - *Edge Cases:*  
    - Gmail API quota limits or auth errors may block sending.  
    - Invalid or guessed emails can cause bounce.  

- **Wait 3 Days**  
  - *Type:* Wait  
  - *Role:* Pauses workflow for 3 days after initial outreach before follow-up.  
  - *Configuration:* 3 days delay.  
  - *Inputs:* After sending initial email.  
  - *Outputs:* Passes to follow-up email node.  
  - *Edge Cases:* Workflow interruptions or restarts could affect timing.  

- **Send Follow-up Email**  
  - *Type:* Gmail (Send Email)  
  - *Role:* Sends a polite follow-up email reminding influencer about the partnership proposal.  
  - *Configuration:*  
    - Sends to same email as initial outreach.  
    - Styled HTML email with recap of campaign details and CTA for scheduling a call.  
    - Fixed subject line "Following up on our collaboration opportunity".  
  - *Inputs:* Influencer JSON from wait node.  
  - *Outputs:* None further downstream.  
  - *Edge Cases:* Same as initial email node.

- **Sticky Note1**  
  - *Type:* Sticky Note  
  - *Role:* Documentation for campaign tracking focus areas: response rates, cost per acquisition, engagement, and ROI.

---

#### 2.6 Campaign Tracking

**Overview:**  
Appends influencer outreach data to a Google Sheet for tracking campaign progress and responses.

**Nodes Involved:**  
- Track Campaign Progress  

**Node Details:**

- **Track Campaign Progress**  
  - *Type:* Google Sheets (Append Row)  
  - *Role:* Logs outreach data including campaign ID, influencer name, platform, followers, engagement, score, estimated cost, email, proposal send timestamp, and current status ("pending_response").  
  - *Configuration:*  
    - Spreadsheet ID and sheet name specified.  
    - Values dynamically extracted from influencer JSON.  
    - Requires Google Sheets OAuth2 credentials with write access.  
  - *Inputs:* Influencer JSON after email extraction.  
  - *Outputs:* None further downstream.  
  - *Edge Cases:*  
    - Spreadsheet ID must be valid and accessible.  
    - API rate limits and credential expiry may cause failure.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                                | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                                          |
|----------------------------|---------------------|-----------------------------------------------|--------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Campaign Brief Webhook      | Webhook             | Receives campaign brief input                  | -                              | Campaign Settings                  |                                                                                                                      |
| Campaign Settings           | Set                 | Configures campaign parameters                  | Campaign Brief Webhook          | Search Instagram/TikTok/YouTube   |                                                                                                                      |
| Sticky Note                | Sticky Note          | Notes on campaign setup                         | -                              | -                                 | ## Influencer Campaign Setup ⚙️ **Configure campaign parameters:** - Target audience demographics - Budget allocation per platform - Content requirements - Timeline and deliverables |
| Search Instagram Influencers| HTTP Request        | Searches Instagram for influencers              | Campaign Settings              | Score & Qualify Influencers       |                                                                                                                      |
| Search TikTok Influencers   | HTTP Request        | Searches TikTok for influencers                  | Campaign Settings              | Score & Qualify Influencers       |                                                                                                                      |
| Search YouTube Influencers  | HTTP Request        | Searches YouTube for influencer channels        | Campaign Settings              | Score & Qualify Influencers       |                                                                                                                      |
| Score & Qualify Influencers | Code                | Scores and filters influencers                   | Search Instagram/TikTok/YouTube| Filter Top Influencers             |                                                                                                                      |
| Filter Top Influencers      | If                  | Filters influencers by score ≥ 70                | Score & Qualify Influencers    | Generate Outreach Content          |                                                                                                                      |
| Generate Outreach Content   | Code                | Creates personalized email content and timeline | Filter Top Influencers         | Extract Contact Info               |                                                                                                                      |
| Extract Contact Info        | Code                | Extracts or guesses influencer email addresses  | Generate Outreach Content      | Send Outreach Email, Track Campaign Progress |                                                                                                                      |
| Send Outreach Email         | Gmail               | Sends personalized outreach email                | Extract Contact Info           | Wait 3 Days                      |                                                                                                                      |
| Wait 3 Days                | Wait                 | Delays 3 days before follow-up                    | Send Outreach Email            | Send Follow-up Email              |                                                                                                                      |
| Send Follow-up Email        | Gmail               | Sends follow-up outreach email                    | Wait 3 Days                   | -                                 |                                                                                                                      |
| Sticky Note1               | Sticky Note          | Notes on campaign tracking                        | -                              | -                                 | ## Campaign Tracking 📊 **Monitor progress:** - Response rates by platform - Cost per acquisition - Engagement analytics - ROI optimization |
| Track Campaign Progress     | Google Sheets       | Logs outreach data to Google Sheet                | Extract Contact Info           | -                                 |                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `campaign-brief`  
   - Purpose: Receive campaign brief JSON payload.  

2. **Add Set Node (Campaign Settings)**  
   - Type: Set  
   - Parameters:  
     - `totalBudget` = `{{ $json.budget }}`  
     - `minFollowers` = 10000  
     - `maxFollowers` = 500000  
     - `minEngagementRate` = 3  
     - `campaignName` = `{{ $json.campaign_name }}`  
     - `targetAudience` = `{{ $json.target_audience }}`  
     - `brandEmail` = `partnerships@yourbrand.com`  
   - Connect Webhook output to this node.  

3. **Create Three HTTP Request Nodes for Influencer Search:**  
   - **Instagram Search:**  
     - URL: `https://api.instagram.com/v1/users/search`  
     - Method: GET  
     - Query Params: `q={{ $json.targetAudience }}`, `count=50`  
     - Headers: Authorization `Bearer {{ $credentials.instagram.accessToken }}`  
     - Connect Campaign Settings output here.  
   - **TikTok Search:**  
     - URL: `https://api.tiktok.com/v1/users/search`  
     - Method: GET  
     - Query Params: `keyword={{ $json.targetAudience }}`, `limit=50`  
     - Headers: Authorization `Bearer {{ $credentials.tiktok.accessToken }}`  
     - Connect Campaign Settings output here.  
   - **YouTube Search:**  
     - URL: `https://api.youtube.com/v3/search`  
     - Method: GET  
     - Query Params: `q={{ $json.targetAudience }}`, `type=channel`, `maxResults=50`  
     - Headers: Authorization `Bearer {{ $credentials.youtube.accessToken }}`  
     - Connect Campaign Settings output here.  

4. **Add Code Node (Score & Qualify Influencers):**  
   - Copy the JavaScript logic that:  
     - Reads all three search results, normalizes data.  
     - Applies scoring based on follower count, engagement rate, platform, bio relevance.  
     - Filters by min/max followers and engagement thresholds.  
     - Sorts and slices top 20.  
     - Adds estimated costs and campaign metadata.  
   - Connect outputs of all three search nodes to this node.  

5. **Add If Node (Filter Top Influencers):**  
   - Condition: Numeric, `{{ $json.score }} >= 70`  
   - Connect Score & Qualify Influencers output here.  

6. **Add Code Node (Generate Outreach Content):**  
   - Copy JS code to generate:  
     - Randomized personalized email subject lines.  
     - Deliverables list by platform.  
     - Campaign start and end dates (now + 14 days).  
     - Proposal sent timestamp.  
   - Connect If node “true” output here.  

7. **Add Code Node (Extract Contact Info):**  
   - Copy JS code to extract email from bio or generate fallback emails.  
   - Connect Generate Outreach Content output here.  

8. **Add Gmail Node (Send Outreach Email):**  
   - Configure to send email:  
     - To: `{{ $json.email }}`  
     - Subject: `{{ $json.outreach_subject }}`  
     - Message: Paste the provided styled HTML template with variables replaced accordingly.  
   - Connect Extract Contact Info output here.  
   - Configure Gmail OAuth2 credentials with send permission.  

9. **Add Google Sheets Node (Track Campaign Progress):**  
   - Operation: Append Row  
   - Spreadsheet ID: Your Google Sheets document ID  
   - Sheet Name: `Influencer Outreach Tracking`  
   - Values: Map fields like campaign ID, username, platform, followers, engagement, score, estimated cost, email, proposal sent timestamp, and set status as "pending_response".  
   - Connect Extract Contact Info output here (parallel to Send Outreach Email).  
   - Configure Google Sheets OAuth2 credentials.  

10. **Add Wait Node (Wait 3 Days):**  
    - Duration: 3 days  
    - Connect Send Outreach Email output here.  

11. **Add Gmail Node (Send Follow-up Email):**  
    - Configure email:  
      - To: `{{ $json.email }}`  
      - Subject: `Following up on our collaboration opportunity`  
      - Message: Paste the provided styled HTML template for follow-up with variables.  
    - Connect Wait node output here.  
    - Use same Gmail OAuth2 credentials.  

12. **Optional: Add Sticky Note Nodes** for documentation inside the workflow, replicating the content for campaign setup and tracking.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                               |
|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| The workflow requires active OAuth2 credentials for Instagram, TikTok, YouTube APIs, Gmail, and Google Sheets. | Credential setup in n8n must be preconfigured.|
| API rate limits and token expiration must be monitored for robust operation.                                   | Platform API documentation and n8n credential management. |
| The influencer scoring algorithm weights followers (30%), engagement (40%), platform preference (20%), and bio relevance (10%). | Customizable in the JavaScript scoring node.  |
| Email extraction uses regex and heuristic generation; consider integrating an email verification service for better accuracy. | Email validation services like Hunter.io, NeverBounce, or ZeroBounce (not included). |
| HTML email templates use inline CSS styling for consistent rendering across clients.                            | Templates included in Gmail nodes.             |
| Google Sheet must have columns matching the appended values for proper tracking.                               | Ensure column order matches mapping in Google Sheets node. |
| Campaign start and end dates are calculated as current date plus 14 days, no timezone adjustments.             | Adjust if necessary to localize campaign timing. |
| Sticky Notes provide helpful documentation within the workflow for maintainers.                                | Useful for team collaboration and onboarding. |

---

*Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.*