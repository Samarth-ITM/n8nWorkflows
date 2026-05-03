Detect influencer fraud and fake followers with Instagram, X, TikTok and Claude

https://n8nworkflows.xyz/workflows/detect-influencer-fraud-and-fake-followers-with-instagram--x--tiktok-and-claude-13708


# Detect influencer fraud and fake followers with Instagram, X, TikTok and Claude

This document provides a technical overview and reconstruction guide for the **AI Influencer Fraud & Fake Follower Detector** n8n workflow.

---

### 1. Workflow Overview

The purpose of this workflow is to automate the vetting process for social media influencers before brand partnerships. It evaluates account authenticity by cross-referencing raw metrics from social platforms with algorithmic scoring and AI-powered behavioral analysis.

The logic is organized into three primary functional blocks:
*   **1.1 Data Acquisition & Algorithmic Scoring:** Receives a request via Webhook, fetches profile data from Instagram, X (Twitter), or TikTok, and calculates a base fraud score using JavaScript.
*   **1.2 AI Verification & Reporting:** Uses Claude (Anthropic) to perform a deep qualitative analysis of the metrics and generates a comprehensive structured report.
*   **1.3 Notification & Archiving:** Sends the final report to the partnership team via email and logs the results into a PostgreSQL database for historical tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Acquisition & Algorithmic Scoring
**Overview:** This block acts as the entry point, retrieving real-time data from social APIs and performing the initial quantitative "Red Flag" check.

*   **Nodes Involved:** `Influencer Profile Input`, `Fetch Influencer Data`, `Analyze Follower Patterns`.
*   **Node Details:**
    *   **Influencer Profile Input (Webhook):** Receives a POST request with `username` and `platform`. It triggers the entire process.
    *   **Fetch Influencer Data (HTTP Request):** A dynamic node that selects the API endpoint and authentication method based on the platform (Instagram Graph API, Twitter API v2, or TikTok API).
    *   **Analyze Follower Patterns (Code):** A complex JavaScript node that calculates metrics:
        *   *Follower-to-Following Ratio:* Penalizes accounts that follow too many people relative to their followers.
        *   *Engagement Rate:* Flags accounts with high followers but low likes/comments.
        *   *Growth Velocity:* Identifies suspicious "spikes" in follower growth.
        *   *Consistency:* Measures posting frequency to detect inactive or bot-managed accounts.

#### 2.2 AI Verification & Reporting
**Overview:** This block adds a layer of intelligence by using LLMs to interpret the data and synthesize a final decision.

*   **Nodes Involved:** `AI Authenticity Analysis`, `Generate Fraud Report`.
*   **Node Details:**
    *   **AI Authenticity Analysis (HTTP Request):** Sends the calculated metrics and red flags to Claude (`claude-sonnet-4-20250514`). It asks for a verdict (Authentic/Fraudulent) and a confidence score.
    *   **Generate Fraud Report (Code):** Combines the algorithmic score (70% weight) and AI confidence (30% weight) into a final "Authenticity Score." It maps the results into a professional report structure including "Next Steps" and "Estimated Fake Follower Percentage."

#### 2.3 Notification & Archiving
**Overview:** Ensures the results are delivered to human stakeholders and stored for long-term auditing.

*   **Nodes Involved:** `Send Report & Notification`, `Save to Database`.
*   **Node Details:**
    *   **Send Report & Notification (Email Send):** Dispatches an HTML email to the brand team. The subject line dynamically changes based on the risk level (e.g., "REJECTED - HIGH FRAUD RISK").
    *   **Save to Database (Postgres):** Maps the report data into a `partnerships.influencer_fraud_reports` table. This prevents duplicate analysis in the future and tracks influencer trends.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Influencer Profile Input** | Webhook | Entry Point | (None) | Fetch Influencer Data | Input → Fetch Profile → Analyze Patterns |
| **Fetch Influencer Data** | HTTP Request | Data Retrieval | Influencer Profile Input | Analyze Follower Patterns | Input → Fetch Profile → Analyze Patterns |
| **Analyze Follower Patterns** | Code | Algorithmic Scoring | Fetch Influencer Data | AI Authenticity Analysis | Input → Fetch Profile → Analyze Patterns |
| **AI Authenticity Analysis** | HTTP Request | Qualitative Analysis | Analyze Follower Patterns | Generate Fraud Report | IAI Verification → Generate Report → Notify Team |
| **Generate Fraud Report** | Code | Data Synthesis | AI Authenticity Analysis | Send Report & Notification | IAI Verification → Generate Report → Notify Team |
| **Send Report & Notification** | Email Send | Communication | Generate Fraud Report | Save to Database | IAI Verification → Generate Report → Notify Team |
| **Save to Database** | Postgres | Persistence | Send Report & Notification | (None) | Database Log |

---

### 4. Reproducing the Workflow from Scratch

1.  **Webhook Setup:** Create a Webhook node named `Influencer Profile Input`. Set method to `POST` and path to `influencer-fraud-check`.
2.  **Social API Integration:**
    *   Add an HTTP Request node. Use a ternary expression in the URL field to switch between Instagram, Twitter, and TikTok endpoints based on `$json.platform`.
    *   Configure credentials for each platform (OAuth2 for Instagram/Twitter).
3.  **Algorithmic Logic:**
    *   Add a Code node. Implement logic to calculate `followerRatio`, `engagementRate`, and `growthVelocity`.
    *   Define "Red Flags" (e.g., if followers > 10k and posts < 10).
    *   Return a JSON object containing a `fraudAssessment` object.
4.  **AI Integration:**
    *   Add an HTTP Request node for Anthropic. 
    *   Target `https://api.anthropic.com/v1/messages`.
    *   Prompt the AI to analyze the metrics provided by the previous node and return a JSON verdict.
5.  **Final Report Synthesis:**
    *   Add a Code node to merge the AI's response with the automated metrics. 
    *   Calculate the weighted `finalScore` (70% automated, 30% AI).
6.  **Outputs:**
    *   Add an Email node. Set the recipient and use expressions to include the `executiveSummary` in the body.
    *   Add a Postgres node. Configure it to insert data into the `influencer_fraud_reports` table (refer to the schema in the general notes).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Database Schema** | Requires a table with columns: `report_id`, `username`, `platform`, `authenticity_score`, `risk_level`, `red_flags` (JSONB), etc. |
| **API Keys Required** | Meta for Developers (Instagram), Twitter Developer Portal, Anthropic (Claude), and SMTP server. |
| **Usage Example** | `curl -X POST https://your-n8n.com/webhook/influencer-fraud-check -d '{"username":"example_user","platform":"instagram"}'` |
| **Confidence Scoring** | The methodology uses a 70/30 split between hard data (ratios) and soft data (AI behavioral analysis). |