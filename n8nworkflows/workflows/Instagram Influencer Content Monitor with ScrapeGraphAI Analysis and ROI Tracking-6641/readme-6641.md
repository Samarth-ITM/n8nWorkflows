Instagram Influencer Content Monitor with ScrapeGraphAI Analysis and ROI Tracking

https://n8nworkflows.xyz/workflows/instagram-influencer-content-monitor-with-scrapegraphai-analysis-and-roi-tracking-6641


# Instagram Influencer Content Monitor with ScrapeGraphAI Analysis and ROI Tracking

### 1. Workflow Overview

This workflow automates daily monitoring and analysis of Instagram influencer content to support influencer marketing campaigns. It is designed to extract influencer profile data and recent posts, analyze engagement and content quality, detect brand mentions (including sponsored posts), track campaign performance, and calculate marketing ROI. The workflow is structured into six logical blocks reflecting a stepwise process from data acquisition to business intelligence output.

**Logical Blocks:**

- **1.1 Daily Trigger:** Initiates the workflow every day at 9:00 AM.
- **1.2 Data Extraction:** Scrapes Instagram influencer profiles and recent posts using ScrapeGraphAI.
- **1.3 Content Analysis:** Processes scraped posts to evaluate engagement, content quality, and performance tiers.
- **1.4 Brand Mention Detection:** Identifies brand mentions and sponsored content within posts.
- **1.5 Campaign Performance Tracking:** Aggregates metrics to assess campaign reach, engagement, and performance scores.
- **1.6 Marketing ROI Calculation:** Calculates return on investment and provides investment efficiency insights and recommendations.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Trigger

- **Overview:**  
  Automatically triggers the entire workflow daily at 9:00 AM UTC to ensure up-to-date monitoring.

- **Nodes Involved:**  
  - Daily Schedule Trigger

- **Node Details:**  
  - **Node Name:** Daily Schedule Trigger  
  - **Type:** Schedule Trigger  
  - **Configuration:** Cron expression set to run at 09:00 AM daily (`0 9 * * *`) in UTC timezone.  
  - **Inputs:** None (trigger node)  
  - **Outputs:** Connected to "ScrapeGraphAI - Influencer Profiles" node.  
  - **Edge Cases:** Misconfigured cron expressions may cause missed runs; time zone differences may affect scheduling; ensure server time sync.  
  - **Sticky Note:** Provides overview of schedule and use cases (daily campaign monitoring, reporting).

#### 2.2 Data Extraction

- **Overview:**  
  Uses ScrapeGraphAI to scrape Instagram profiles dynamically based on influencer usernames, retrieving profile statistics and recent posts with metadata.

- **Nodes Involved:**  
  - ScrapeGraphAI - Influencer Profiles

- **Node Details:**  
  - **Node Name:** ScrapeGraphAI - Influencer Profiles  
  - **Type:** ScrapeGraphAI node (custom integration)  
  - **Configuration:**  
    - `userPrompt`: Structured JSON schema instructing extraction of profile info (username, followers, bio, verification) and recent posts (url, caption, likes, comments, hashtags, mentions).  
    - `websiteUrl`: Dynamic URL constructed as `https://www.instagram.com/{{ $json.influencer_username }}`, expecting `influencer_username` input from trigger or previous node (not explicitly in this workflow, so must be set upstream in actual usage).  
  - **Inputs:** Trigger node output (schedule trigger)  
  - **Outputs:** To "Content Analyzer"  
  - **Edge Cases:**  
    - Missing or incorrect influencer usernames leading to failed scraping.  
    - Rate limiting or blocking by Instagram or ScrapeGraphAI service.  
    - Schema mismatch or unexpected data formats causing parsing errors.  
  - **Sticky Note:** Explains extraction details and supported platforms (Instagram, TikTok, YouTube).

#### 2.3 Content Analysis

- **Overview:**  
  Analyzes recent posts to compute engagement rates, quality scores, and categorize posts into performance tiers, providing a quantitative content benchmark.

- **Nodes Involved:**  
  - Content Analyzer

- **Node Details:**  
  - **Node Name:** Content Analyzer  
  - **Type:** Code Node (JavaScript)  
  - **Configuration:**  
    - Parses scraped post data.  
    - Calculates engagement rate using formula: ((likes + comments) / likes * 100).  
    - Scores content quality based on caption length, number of hashtags, and mentions.  
    - Assigns performance tiers: High (>5%), Medium (>2%), Low (<=2%).  
    - Computes average engagement across analyzed posts.  
  - **Inputs:** From "ScrapeGraphAI - Influencer Profiles"  
  - **Outputs:** To "Brand Mention Detector"  
  - **Edge Cases:**  
    - Posts with zero likes to avoid division by zero (handled by conditional checks).  
    - Missing or malformed data fields (likes, comments, captions).  
    - Non-numeric values in engagement fields.  
  - **Sticky Note:** Details analysis features and output expectations.

#### 2.4 Brand Mention Detection

- **Overview:**  
  Detects brand mentions and sponsored content indicators within the analyzed posts to identify promotional or partnership content.

- **Nodes Involved:**  
  - Brand Mention Detector

- **Node Details:**  
  - **Node Name:** Brand Mention Detector  
  - **Type:** Code Node (JavaScript)  
  - **Configuration:**  
    - Uses a predefined list of brand keywords and sponsored content hashtags (`#ad`, `#sponsored`, etc.).  
    - Searches captions, hashtags, and @mentions for brand indicators.  
    - Flags posts as sponsored content if indicators found.  
    - Aggregates brand mention counts and lists top brands mentioned.  
  - **Inputs:** From "Content Analyzer"  
  - **Outputs:** To "Campaign Performance Tracker"  
  - **Edge Cases:**  
    - Case sensitivity handled by converting to lowercase.  
    - False positives/negatives if brand keywords overlap with common words.  
    - Posts without captions or mentions handled gracefully.  
  - **Sticky Note:** Explains detection logic and metrics tracked.

#### 2.5 Campaign Performance Tracking

- **Overview:**  
  Aggregates influencer profile data and post metrics to compute overall campaign performance, including reach, engagement, sponsored content impact, and a composite performance score.

- **Nodes Involved:**  
  - Campaign Performance Tracker

- **Node Details:**  
  - **Node Name:** Campaign Performance Tracker  
  - **Type:** Code Node (JavaScript)  
  - **Configuration:**  
    - Calculates total reach from follower count.  
    - Computes total and sponsored post engagement (likes + comments).  
    - Calculates average engagements per post and engagement rate relative to reach.  
    - Generates a campaign performance score (0-100) based on engagement rate, sponsored posts count, and total brand mentions.  
    - Identifies top 3 performing sponsored posts.  
  - **Inputs:** From "Brand Mention Detector"  
  - **Outputs:** To "Marketing ROI Calculator"  
  - **Edge Cases:**  
    - Profile follower count missing or malformed.  
    - Posts with missing engagement data.  
    - No sponsored posts scenario handled.  
  - **Sticky Note:** Describes key metrics and performance analysis.

#### 2.6 Marketing ROI Calculation

- **Overview:**  
  Calculates the financial return on investment of influencer campaigns based on estimated costs, engagement, reach, and campaign performance data, offering business insights and recommendations.

- **Nodes Involved:**  
  - Marketing ROI Calculator

- **Node Details:**  
  - **Node Name:** Marketing ROI Calculator  
  - **Type:** Code Node (JavaScript)  
  - **Configuration:**  
    - Uses fixed estimated costs for sponsored posts, management, and content creation.  
    - Calculates total investment and estimated value based on engagement and reach valuations.  
    - Computes ROI percentage, cost per engagement, and CPM (cost per thousand reach).  
    - Produces investment efficiency status and payback period.  
    - Generates recommendations for campaign scaling and optimization.  
  - **Inputs:** From "Campaign Performance Tracker"  
  - **Outputs:** Final report JSON output.  
  - **Edge Cases:**  
    - Zero or missing investment or engagement values.  
    - Negative ROI handled with specific recommendations.  
  - **Sticky Note:** Explains ROI calculation details and business intelligence usage.

---

### 3. Summary Table

| Node Name                        | Node Type                | Functional Role                      | Input Node(s)                       | Output Node(s)                     | Sticky Note                                                                                              |
|---------------------------------|--------------------------|------------------------------------|-----------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------|
| Daily Schedule Trigger           | Schedule Trigger         | Starts workflow daily at 9:00 AM   | None                              | ScrapeGraphAI - Influencer Profiles | Step 1: Daily Schedule Trigger ⏰ - Describes schedule and use cases                                    |
| ScrapeGraphAI - Influencer Profiles | ScrapeGraphAI Node       | Extracts influencer profile and posts | Daily Schedule Trigger            | Content Analyzer                 | Step 2: ScrapeGraphAI - Influencer Profiles 🤖 - Details extraction schema and supported platforms     |
| Content Analyzer                | Code Node (JavaScript)   | Analyzes posts for engagement and quality | ScrapeGraphAI - Influencer Profiles | Brand Mention Detector          | Step 3: Content Analyzer 📊 - Explains engagement metrics and quality scoring                           |
| Brand Mention Detector          | Code Node (JavaScript)   | Detects brand mentions and sponsorship | Content Analyzer                 | Campaign Performance Tracker     | Step 4: Brand Mention Detector 🏷️ - Lists detection capabilities and tracked metrics                    |
| Campaign Performance Tracker    | Code Node (JavaScript)   | Aggregates campaign KPIs and scores | Brand Mention Detector           | Marketing ROI Calculator         | Step 5: Campaign Performance Tracker 📈 - Summarizes campaign metrics and performance analysis          |
| Marketing ROI Calculator        | Code Node (JavaScript)   | Calculates ROI and investment efficiency | Campaign Performance Tracker     | None (final output)              | Step 6: Marketing ROI Calculator 💰 - Details ROI calculations and business intelligence insights       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Daily Schedule Trigger" node**  
   - Type: Schedule Trigger  
   - Set cron expression to `0 9 * * *` (9:00 AM daily UTC)  
   - No inputs, output connects to next node.

2. **Create "ScrapeGraphAI - Influencer Profiles" node**  
   - Type: ScrapeGraphAI node (custom integration)  
   - Configure `userPrompt` with JSON schema to extract:  
     - Profile: username, followers, following, posts_count, bio, verified  
     - Recent posts: post_url, caption, likes, comments, date, hashtags, mentions  
   - Set `websiteUrl` to `https://www.instagram.com/{{ $json.influencer_username }}` (ensure upstream input provides `influencer_username`)  
   - Connect input from "Daily Schedule Trigger".

3. **Create "Content Analyzer" node**  
   - Type: Code Node (JavaScript)  
   - Paste JavaScript code for engagement and quality analysis (as described)  
   - Connect input from "ScrapeGraphAI - Influencer Profiles".

4. **Create "Brand Mention Detector" node**  
   - Type: Code Node (JavaScript)  
   - Paste JavaScript code to detect brand mentions and sponsored indicators  
   - Use predefined brand keywords and sponsored hashtags list  
   - Connect input from "Content Analyzer".

5. **Create "Campaign Performance Tracker" node**  
   - Type: Code Node (JavaScript)  
   - Paste JavaScript code to calculate campaign reach, engagement, performance score, and top campaigns  
   - Connect input from "Brand Mention Detector".

6. **Create "Marketing ROI Calculator" node**  
   - Type: Code Node (JavaScript)  
   - Paste JavaScript code to calculate ROI, cost efficiency, and recommendations based on campaign data  
   - Use fixed estimated cost parameters (cost per sponsored post, management cost, content creation cost)  
   - Connect input from "Campaign Performance Tracker".

7. **Configure execution order:**  
   - Daily Schedule Trigger → ScrapeGraphAI - Influencer Profiles → Content Analyzer → Brand Mention Detector → Campaign Performance Tracker → Marketing ROI Calculator

8. **Credentials:**  
   - For ScrapeGraphAI node, configure API credentials as required by the integration.  
   - No OAuth or external credentials needed for code nodes.

9. **Parameters and defaults:**  
   - Ensure influencer usernames are fed as input JSON with key `influencer_username`.  
   - Cost parameters in ROI Calculator can be adjusted as per actual campaign data.  
   - Timezone assumptions for schedule trigger should align with operational needs.

10. **Testing:**  
    - Run manually first with test influencer usernames.  
    - Validate output at each node.  
    - Handle errors in scraping or parsing by adding error workflows or fallback nodes if needed.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                             |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Workflow designed for daily automated monitoring of Instagram influencer marketing campaigns.       | Workflow purpose and use case overview.                                                                    |
| ScrapeGraphAI supports multiple social platforms including Instagram, TikTok, and YouTube.         | See Scraper Info sticky note.                                                                               |
| Content quality scoring combines caption length, hashtag count, and brand mentions for robust analysis. | Content Analyzer sticky note.                                                                                |
| Brand detection uses a predefined list of brands and sponsored content keywords for accuracy.       | Brand Detector Info sticky note.                                                                            |
| Campaign performance scoring normalizes engagement and sponsorship data into a single score (0-100). | Performance Tracker Info sticky note.                                                                       |
| ROI calculation uses estimated costs and values; adjust parameters to reflect actual campaign data. | ROI Calculator Info sticky note.                                                                             |
| Scheduling uses UTC timezone by default; adjust cron or timezone settings as needed for local time. | Daily Schedule Trigger sticky note.                                                                          |

---

*Disclaimer:* The provided content originates solely from an automated n8n workflow. It adheres strictly to content policies and contains no illegal or protected elements. All data processed is lawful and public.