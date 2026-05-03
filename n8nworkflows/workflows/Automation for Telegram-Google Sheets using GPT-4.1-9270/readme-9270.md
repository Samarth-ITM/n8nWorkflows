Automation for Telegram/Google Sheets using GPT-4.1

https://n8nworkflows.xyz/workflows/automation-for-telegram-google-sheets-using-gpt-4-1-9270


# Automation for Telegram/Google Sheets using GPT-4.1

### 1. Workflow Overview

This workflow automates the process of discovering freelance job postings from Freelancer.com, analyzing their quality and suitability using GPT-4.1 AI models, generating tailored proposals, logging results in Google Sheets, and sending alerts through Telegram. It targets freelance professionals who want to automate job discovery, evaluation, and outreach to increase efficiency and reduce manual filtering.

The workflow is organized into the following logical blocks:

- **1.1 Scheduling & Initialization:** Periodic trigger and setting up keywords and wishlists.
- **1.2 Data Retrieval & Deduplication:** Fetch RSS feed jobs, load previously seen job links, and filter duplicates.
- **1.3 Job Quality Filtering:** Apply manual filters on job descriptions and posting freshness.
- **1.4 AI Job Analysis:** Use GPT-4.1 to score, classify, and analyze jobs.
- **1.5 Score Filtering:** Gate jobs by AI-generated score threshold.
- **1.6 Proposal Generation:** Generate personalized proposals via AI, enriched by optional Sheets tool.
- **1.7 Logging & Notifications:** Append job and analysis data to Google Sheets and send Telegram alerts.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Initialization

**Overview:**  
Triggers the workflow every 5 minutes, then sets the keywords and wishlist criteria for job searching.

**Nodes Involved:**  
- Schedule: Every 5 Minutes1  
- Settings (Keyword & Wishlist)1

**Node Details:**

- **Schedule: Every 5 Minutes1**  
  - Type: Schedule Trigger  
  - Configuration: Runs the workflow every 5 minutes to maintain a regular check for new jobs. The frequency can be customized (e.g., 3 min for competitive niches or 10–15 min for cost savings).  
  - Inputs: None (trigger node)  
  - Outputs: Settings (Keyword & Wishlist)1  
  - Edge cases: Missed triggers if n8n server downtime; frequency trade-off between API cost and responsiveness.

- **Settings (Keyword & Wishlist)1**  
  - Type: Set  
  - Configuration: Defines main search keywords (e.g., shopify, react) and wishlist criteria (e.g., budget > $500, clear scope). These variables are used downstream to customize job fetching and filtering.  
  - Inputs: From Schedule trigger  
  - Outputs: Load Seen Links (Google Sheets)1  
  - Edge cases: Incorrect or empty keywords can lead to no results or irrelevant jobs.

---

#### 2.2 Data Retrieval & Deduplication

**Overview:**  
Loads previously processed job links from Google Sheets to prevent duplicate alerts, fetches new jobs via RSS, and filters out duplicates.

**Nodes Involved:**  
- Load Seen Links (Google Sheets)1  
- Collect Seen Links1  
- Fetch Freelancer.com RSS  
- De-duplicate by Link1

**Node Details:**

- **Load Seen Links (Google Sheets)1**  
  - Type: Google Sheets  
  - Configuration: Reads the "Link" column from the logging sheet to get all previously seen job URLs. Uses the same spreadsheet as the logger.  
  - Inputs: Settings (Keyword & Wishlist)1  
  - Outputs: Collect Seen Links1  
  - Edge cases: Sheets API quota limits; missing "Link" column causes failure; network issues.

- **Collect Seen Links1**  
  - Type: Aggregate  
  - Configuration: Aggregates loaded links into a quick lookup structure for deduplication (likely a list or hash map).  
  - Inputs: Load Seen Links (Google Sheets)1  
  - Outputs: Fetch Freelancer.com RSS  
  - Edge cases: Empty input if no prior links; malformed data.

- **Fetch Freelancer.com RSS**  
  - Type: RSS Feed Read  
  - Configuration: Fetches latest jobs from Freelancer.com RSS feed. The URL is customizable with parameters like keyword, min_price, max_price. Example URL includes filter for keyword=wordpress and min_price=500. Multiple RSS nodes can be configured for multiple keywords.  
  - Inputs: Collect Seen Links1  
  - Outputs: De-duplicate by Link1  
  - Edge cases: RSS feed downtime; malformed or empty feed; rate limiting.

- **De-duplicate by Link1**  
  - Type: Filter  
  - Configuration: Passes only jobs whose link is not already in the seen links list. If links are unstable, alternative deduplication strategies such as hashing title + date are recommended.  
  - Inputs: Fetch Freelancer.com RSS  
  - Outputs: Filter Job Quality  
  - Edge cases: Link instability; false negatives or positives in deduplication.

---

#### 2.3 Job Quality Filtering

**Overview:**  
Applies manual filters to weed out low-quality or irrelevant jobs based on description length, posting recency, and optionally budget or keywords.

**Nodes Involved:**  
- Filter Job Quality

**Node Details:**

- **Filter Job Quality**  
  - Type: Filter  
  - Configuration: Current rules include description longer than 100 characters and posted within the last 2 hours. Optional filters could include budget thresholds or keyword inclusion/exclusion.  
  - Inputs: De-duplicate by Link1  
  - Outputs: AI Job Analyzer  
  - Edge cases: Jobs with incomplete metadata; time zone inconsistencies; overly strict filters may exclude good jobs.

---

#### 2.4 AI Job Analysis

**Overview:**  
Runs GPT-4.1 based AI analysis to score each job (1–10), explain reasoning, flag risks, and classify client type.

**Nodes Involved:**  
- AI Job Analyzer  
- Structured Output Parser  
- LLM (OpenAI)1

**Node Details:**

- **AI Job Analyzer**  
  - Type: Langchain Agent  
  - Configuration: AI agent configured to evaluate jobs according to a specific wishlist, outputting score, explanations, risk flags, and client classification. The wishlist should be specific to reduce noise.  
  - Inputs: Filter Job Quality (main), Structured Output Parser (ai_outputParser), LLM (OpenAI)1 (ai_languageModel)  
  - Outputs: Gate: Score ≥  
  - Edge cases: AI model errors; request timeouts; inconsistent or ambiguous job descriptions.

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Configuration: Enforces strict JSON formatting on AI responses to prevent downstream parsing errors.  
  - Inputs: AI Job Analyzer (ai_outputParser)  
  - Outputs: AI Job Analyzer  
  - Edge cases: Improperly formatted AI output; parser failures.

- **LLM (OpenAI)1**  
  - Type: Langchain OpenAI Chat Model  
  - Configuration: Backend language model (GPT-4.1 or gpt-4o-mini recommended for speed/cost). Used by AI Job Analyzer and Proposal Generator.  
  - Inputs: None (credential node), connected to AI Job Analyzer and AI Proposal Generator as LLM backend  
  - Edge cases: API quota, latency, and authentication errors.

---

#### 2.5 Score Filtering

**Overview:**  
Allows only jobs with AI score ≥ 7 to proceed for proposal generation and alerting.

**Nodes Involved:**  
- Gate: Score ≥

**Node Details:**

- **Gate: Score ≥**  
  - Type: If (conditional filter)  
  - Configuration: Filters jobs based on AI score, passing only those with score ≥ 7. Threshold can be adjusted (6 for more alerts, 8 for premium only).  
  - Inputs: AI Job Analyzer  
  - Outputs: AI Proposal Generator  
  - Edge cases: Missing or malformed score data; threshold tuning affects alert volume.

---

#### 2.6 Proposal Generation

**Overview:**  
Generates a personalized proposal text using AI, optionally augmented with portfolio snippets from Google Sheets, preparing for outreach.

**Nodes Involved:**  
- AI Proposal Generator  
- CV (Sheets Tool)1

**Node Details:**

- **AI Proposal Generator**  
  - Type: Langchain OpenAI Node  
  - Configuration: Uses AI to write a job proposal based on job data and personal skills/experience. Users should customize "My Skills" and include relevant portfolio links. AI completes ~90%, user adds 10% for personal touch.  
  - Inputs: Gate: Score ≥ (main), CV (Sheets Tool)1 (ai_tool)  
  - Outputs: Log to Google Sheets  
  - Edge cases: AI output variability; incomplete portfolio data; API errors.

- **CV (Sheets Tool)1**  
  - Type: Google Sheets Tool  
  - Configuration: Optional node that pulls portfolio snippets or other relevant data from Google Sheets to enrich proposals. Can be disabled initially without breaking workflow.  
  - Inputs: None (credential node), connected as ai_tool input to AI Proposal Generator  
  - Outputs: AI Proposal Generator  
  - Edge cases: Missing portfolio data; Sheets API errors.

---

#### 2.7 Logging & Notifications

**Overview:**  
Stores job data, AI analysis, and proposals in Google Sheets and sends Telegram alerts with buttons for interactions.

**Nodes Involved:**  
- Log to Google Sheets  
- Send Telegram Alert

**Node Details:**

- **Log to Google Sheets**  
  - Type: Google Sheets  
  - Configuration: Appends or updates rows keyed by job link to avoid duplicates. Recommended columns include Timestamp, Job, Description, Link, AI Score, Reasoning, Red Flags, Client Type, AI Proposal, and Alert Timestamp.  
  - Inputs: AI Proposal Generator  
  - Outputs: Send Telegram Alert  
  - Edge cases: API quota; update conflicts; missing columns.

- **Send Telegram Alert**  
  - Type: Telegram  
  - Configuration: Sends message alerts via Telegram bot. Requires bot token (created via @BotFather) and chat ID (obtained via @userinfobot). Supports interactive buttons through custom webhooks for "Skip" or "Applied" actions.  
  - Inputs: Log to Google Sheets  
  - Outputs: None (end node)  
  - Edge cases: Bot token or chat ID misconfiguration; Telegram API limits; user interaction webhook failures.

---

### 3. Summary Table

| Node Name                  | Node Type                   | Functional Role               | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                                 |
|----------------------------|-----------------------------|------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Schedule: Every 5 Minutes1  | Schedule Trigger            | Trigger workflow periodically | None                         | Settings (Keyword & Wishlist)1 | ⏰ TRIGGER — Runs workflow every 5 minutes. Customize frequency to balance competitiveness and cost.                         |
| Settings (Keyword & Wishlist)1 | Set                      | Define keywords & wishlist    | Schedule: Every 5 Minutes1    | Load Seen Links (Google Sheets)1 | ⚙️ SETTINGS — Customize job keywords and wishlist criteria. Clone for multiple keywords.                                    |
| Load Seen Links (Google Sheets)1 | Google Sheets          | Load previously seen job links | Settings (Keyword & Wishlist)1 | Collect Seen Links1          | 📥 DEDUPE SOURCE — Loads links from sheet to prevent duplicates.                                                            |
| Collect Seen Links1         | Aggregate                   | Aggregate links for lookup   | Load Seen Links (Google Sheets)1 | Fetch Freelancer.com RSS    | 🧮 AGGREGATE — Builds lookup for dedupe filter.                                                                             |
| Fetch Freelancer.com RSS    | RSS Feed Read               | Fetch new jobs from Freelancer.com RSS | Collect Seen Links1           | De-duplicate by Link1        | 🌐 RSS FETCH — Fetch latest jobs; customize URL with keywords, price. Use multiple RSS nodes for multiple keywords.          |
| De-duplicate by Link1       | Filter                      | Filter out duplicate jobs    | Fetch Freelancer.com RSS      | Filter Job Quality           | 🚫 DEDUPE — Pass only jobs with unseen links. Use hash of title+date if links unstable.                                      |
| Filter Job Quality          | Filter                      | Apply quality filters        | De-duplicate by Link1         | AI Job Analyzer             | ✅ QUALITY FILTER — Filters jobs by description length, posting time, optional budget and keywords.                         |
| AI Job Analyzer             | Langchain Agent             | AI job scoring & classification | Filter Job Quality            | Gate: Score ≥                | 🧠 AI ANALYZER — Scores jobs 1–10, explains reasoning, flags risks, classifies client.                                       |
| Structured Output Parser    | Langchain Output Parser     | Enforce strict JSON parsing  | AI Job Analyzer (ai_outputParser) | AI Job Analyzer             | 🧩 OUTPUT PARSER — Forces strict JSON to avoid downstream errors.                                                           |
| LLM (OpenAI)1              | Langchain OpenAI Chat Model | AI language model backend    | None                         | AI Job Analyzer, AI Proposal Generator | ⚙️ LLM BACKEND — Use GPT-4.1 or gpt-4o-mini for speed/cost.                                                                |
| Gate: Score ≥               | If                          | Filter jobs by AI score      | AI Job Analyzer              | AI Proposal Generator        | 🎯 SCORE GATE — Only pass jobs with score ≥ 7. Threshold adjustable.                                                        |
| AI Proposal Generator       | Langchain OpenAI Node       | AI proposal writing          | Gate: Score ≥, CV (Sheets Tool)1 | Log to Google Sheets         | ✍️ PROPOSAL WRITER — Customize skills, experience, portfolio link. AI writes 90%, add 10% personal touch.                   |
| CV (Sheets Tool)1           | Google Sheets Tool          | Optional portfolio data pull | None                         | AI Proposal Generator        | 🛠️ SHEETS TOOL — Optional for advanced automations. Safe to disable initially.                                              |
| Log to Google Sheets        | Google Sheets               | Log jobs & AI data           | AI Proposal Generator         | Send Telegram Alert          | 📊 JOB LOGGER — Append/update by link; recommended columns include job details, AI scores, proposal, timestamps.             |
| Send Telegram Alert         | Telegram                    | Send alerts                  | Log to Google Sheets          | None                        | 📱 TELEGRAM — Setup bot token, get chat ID. Supports interactive buttons for Skip/Applied via webhook.                      |
| Sticky: SETUP START         | Sticky Note                 | Workflow start note          | None                         | None                        |                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to run every 5 minutes (customize as needed).

2. **Create a Set node for keywords and wishlist**  
   - Type: Set  
   - Define variables for job keywords (e.g., "shopify, react") and wishlist criteria (e.g., "Budget > $500, clear scope").  
   - Connect Schedule Trigger output to this node.

3. **Create a Google Sheets node to load seen job links**  
   - Type: Google Sheets  
   - Configure credentials for your Google account.  
   - Read from your logging spreadsheet, selecting the column "Link".  
   - Connect Set node output to this node.

4. **Create an Aggregate node**  
   - Type: Aggregate  
   - Configure to aggregate all links into a list or lookup structure.  
   - Connect Google Sheets load node output to this node.

5. **Create an RSS Feed Read node**  
   - Type: RSS Feed Read  
   - Set the RSS URL for Freelancer.com, e.g., `https://www.freelancer.com/rss.xml?keyword=your_keyword&min_price=500`  
   - For multiple keywords, clone and customize multiple RSS nodes and merge outputs accordingly.  
   - Connect Aggregate node output to this node.

6. **Create a Filter node for deduplication**  
   - Type: Filter  
   - Configure to only pass items whose link is not in the aggregated seen links list.  
   - Connect RSS node output to this node.

7. **Create a Filter node for job quality**  
   - Type: Filter  
   - Set rules: description length > 100 chars, posted within last 2 hours, optionally budget and keywords.  
   - Connect dedupe filter output to this node.

8. **Create an OpenAI Chat Model node for LLM backend**  
   - Type: Langchain OpenAI Chat Model  
   - Configure OpenAI API credentials with GPT-4.1 or gpt-4o-mini.  
   - No input connections; will be connected as backend for AI nodes.

9. **Create a Langchain Agent node for AI job analysis**  
   - Type: Langchain Agent  
   - Configure prompt to score job 1–10, explain reasoning, flag risks, classify client.  
   - Connect job quality filter output to main input.  
   - Connect LLM node as language model backend.  
   - Add a Structured Output Parser node linked to AI outputParser input.

10. **Create a Structured Output Parser node**  
    - Type: Langchain Output Parser  
    - Configure to enforce strict JSON output for AI responses.  
    - Connect its output to AI Job Analyzer node ai_outputParser input.

11. **Create an If node for score gating**  
    - Type: If  
    - Configure condition: AI score ≥ 7.  
    - Connect AI Job Analyzer main output to this node.

12. **Create a Google Sheets Tool node (optional)**  
    - Type: Google Sheets Tool  
    - Configure credentials and spreadsheet to pull portfolio snippets or other relevant data.  
    - No input connections; connect as ai_tool input for proposal generator.

13. **Create a Langchain OpenAI node for proposal generation**  
    - Type: Langchain OpenAI Node  
    - Configure prompt with personal skills and portfolio link placeholders.  
    - Connect If node output (true branch) to main input.  
    - Connect Google Sheets Tool node as ai_tool input.  
    - Connect LLM node as language model backend.

14. **Create a Google Sheets node for logging**  
    - Type: Google Sheets  
    - Configure credentials and spreadsheet with recommended columns: Timestamp, Job, Description, Link, AI Score, Reasoning, Red Flags, Client Type, AI Proposal, Alert Timestamp.  
    - Set to append or update rows keyed by Link.  
    - Connect proposal generator output to this node.

15. **Create a Telegram node for alerting**  
    - Type: Telegram  
    - Configure bot token (via @BotFather) and chat ID (via @userinfobot).  
    - Optionally configure buttons for Skip/Applied via webhook.  
    - Connect Google Sheets logging output to this node.

16. **Add sticky notes or comments at key nodes for documentation and user guidance.**

17. **Test workflow end-to-end, tuning filters and thresholds as needed.**

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Use multiple RSS Feed nodes for different keywords in parallel, then merge outputs if needed.                     | Workflow scalability and coverage improvement.                                                        |
| Customize Telegram bot buttons via your own webhook to implement Skip or Applied actions for job alerts.          | Telegram integration enhancement.                                                                     |
| Recommended Google Sheets columns: Timestamp, Job, Description, Link, AI Score, Reasoning, Red Flags, Client Type, AI Proposal, Alert Timestamp | Standardized logging for tracking and reporting.                                                       |
| Start with loose filters and 5-min schedule; tighten filters and adjust frequency after 1–2 days based on volume. | Optimization advice for balance between cost and alert relevance.                                     |
| Use GPT-4o-mini for cost-effective AI requests unless richer reasoning is required.                               | Cost and performance balance for AI usage.                                                            |
| To get Telegram Chat ID: DM your bot, then use @userinfobot to retrieve your chat ID.                             | Telegram setup instructions.                                                                           |
| Workflow designed with n8n version supporting Langchain nodes and Google Sheets integration (version 1.1+).       | Version compatibility note.                                                                            |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.