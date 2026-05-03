Monitor & Alert on Social Media Ad Performance with Bright Data MCP

https://n8nworkflows.xyz/workflows/monitor---alert-on-social-media-ad-performance-with-bright-data-mcp-5954


# Monitor & Alert on Social Media Ad Performance with Bright Data MCP

### 1. Workflow Overview

This workflow automates the monitoring of social media advertising performance—specifically Facebook Ads—by scraping ad data from the Facebook Ads Library using Bright Data MCP (Managed Proxy Client) and AI models, analyzing key performance indicators (KPIs), and sending alert emails if any ad is underperforming. It is designed for marketers or analysts who want automated, regular checks on ad effectiveness without manual data collection.

The workflow is logically divided into four main blocks:

- **1.1 Trigger & Input Setup:** Automatically initiates the workflow on a schedule and sets the brand or query URL to monitor.
- **1.2 Ad Scraping via AI Agent (Bright Data MCP):** Uses an AI-powered agent combining OpenAI and Bright Data MCP to simulate user browsing and scrape structured ad data from Facebook.
- **1.3 Single Ad Processing & Performance Evaluation:** Processes one ad at a time from the scraped list, checking if it is underperforming based on defined thresholds (CTR and CPA).
- **1.4 Alerting & No-Action Handling:** Sends an alert email for underperforming ads or silently ends the workflow if the ad is performing satisfactorily.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Setup

- **Overview:**  
  This block sets up the recurring execution of the workflow and defines the specific Facebook Ads dashboard URL related to the brand or keyword of interest (e.g., “Nike”). It prepares the input that drives subsequent ad scraping.

- **Nodes Involved:**  
  - 🔁 Check Ads Every Hour  
  - 📥 Set facebook ad dashboard URL

- **Node Details:**

  - **🔁 Check Ads Every Hour**  
    - *Type:* Schedule Trigger  
    - *Role:* Automatically triggers the workflow every hour to ensure regular monitoring.  
    - *Configuration:* Interval set to 1 hour.  
    - *Inputs:* None (start node).  
    - *Outputs:* Connects to “📥 Set facebook ad dashboard URL”.  
    - *Edge Cases:* Trigger failures could occur due to n8n service downtime or misconfiguration.  
    - *Version:* 1.2

  - **📥 Set facebook ad dashboard URL**  
    - *Type:* Set  
    - *Role:* Assigns the URL string of the Facebook Ads library filtered for the targeted brand/keyword.  
    - *Configuration:* Static URL pointing to Facebook Ads Library filtered by “Nike” in the US market.  
    - *Inputs:* From schedule trigger.  
    - *Outputs:* Connects to “🧠 Scrape Ads via MCP Agent”.  
    - *Edge Cases:* If the URL format changes or the Facebook Ads Library interface is updated, scraping may fail downstream.  
    - *Version:* 3.4

#### 2.2 Ad Scraping via AI Agent (Bright Data MCP)

- **Overview:**  
  This block leverages an AI agent that combines OpenAI language models with Bright Data MCP to simulate a real user browsing Facebook Ads Library and scrape detailed ad performance data. The scraped data is parsed and structured as JSON for further processing.

- **Nodes Involved:**  
  - 🧠 Scrape Ads via MCP Agent (Agent node)  
  - MCP Client  
  - OpenAI Chat Model  
  - Auto-fixing Output Parser  
  - OpenAI Chat Model1  
  - Structured Output Parser1

- **Node Details:**

  - **🧠 Scrape Ads via MCP Agent**  
    - *Type:* AI Agent (Langchain agent node)  
    - *Role:* Coordinates the scraping operation, using sub-nodes for language modeling, MCP tool invocation, and output parsing.  
    - *Configuration:* Prompt to scrape Facebook Ads dashboard data from the URL set in previous block. Uses structured output parser to clean results.  
    - *Inputs:* Receives URL from previous node.  
    - *Outputs:* Passes structured ad data to “🔄 Return One Ad at a Time”.  
    - *Edge Cases:*  
      - MCP client authentication or connection errors.  
      - Changes in Facebook Ads Library page structure causing scraping inaccuracies.  
      - OpenAI API rate limits or failures.  
    - *Version:* 2

  - **MCP Client**  
    - *Type:* MCP Client Tool  
    - *Role:* Executes the tool to scrape data using Bright Data’s proxy infrastructure.  
    - *Configuration:* Uses “scrape_as_markdown” tool with parameters dynamically injected from AI prompt override.  
    - *Inputs:* Called by AI Agent.  
    - *Outputs:* Returns raw scraped markdown data.  
    - *Edge Cases:* Proxy connection failures, blocked requests, or data extraction issues.  
    - *Version:* 1

  - **OpenAI Chat Model**  
    - *Type:* Language Model (OpenAI Chat)  
    - *Role:* Processes scraping instructions and helps parse/structure data.  
    - *Configuration:* Uses GPT-4o-mini model.  
    - *Inputs:* Connected as AI language model in agent flow.  
    - *Outputs:* Intermediate parsed text for further processing.  
    - *Edge Cases:* API quota limits, network errors.  
    - *Version:* 1.2

  - **Auto-fixing Output Parser**  
    - *Type:* Langchain Output Parser (auto-fixing)  
    - *Role:* Automatically fixes and normalizes the output JSON for consistency.  
    - *Inputs:* Receives parsed data from OpenAI Chat Model1.  
    - *Outputs:* Cleaned structured JSON passed back to agent.  
    - *Edge Cases:* Parsing errors if schema changes or unexpected data formats appear.  
    - *Version:* 1

  - **OpenAI Chat Model1**  
    - *Type:* Language Model (OpenAI Chat)  
    - *Role:* Further processes structured output for validation and enhancement.  
    - *Configuration:* Uses GPT-4o-mini model.  
    - *Inputs:* From Auto-fixing Output Parser.  
    - *Outputs:* Feeds into Structured Output Parser1.  
    - *Edge Cases:* Same as other OpenAI nodes.  
    - *Version:* 1.2

  - **Structured Output Parser1**  
    - *Type:* Structured Output Parser  
    - *Role:* Parses final JSON output according to a predefined schema example describing ads and their KPIs.  
    - *Configuration:* JSON schema example includes fields like ad_id, advertiser, ad_title, platforms, status, start_date, ad_text, media_type, media_url, impressions, engagement (likes, shares, comments), estimated_ctr, estimated_cpa.  
    - *Inputs:* From OpenAI Chat Model1.  
    - *Outputs:* Sends cleaned structured data back to Auto-fixing Output Parser and then agent output.  
    - *Edge Cases:* If output does not match schema, parsing errors occur.  
    - *Version:* 1.2

#### 2.3 Single Ad Processing & Performance Evaluation

- **Overview:**  
  This block takes the full list of ads scraped and processes one ad at a time, checking performance metrics against thresholds to decide if the ad is underperforming.

- **Nodes Involved:**  
  - 🔄 Return One Ad at a Time (Code Node)  
  - 🚨 Is Ad Underperforming? (If Node)

- **Node Details:**

  - **🔄 Return One Ad at a Time**  
    - *Type:* Code (JavaScript)  
    - *Role:* Extracts only the first ad from the ads array to process one ad per workflow execution.  
    - *Configuration:* Custom JS code reads `$json.ads` and returns the first ad or empty array if none.  
    - *Inputs:* Receives full ads array from “🧠 Scrape Ads via MCP Agent”.  
    - *Outputs:* Sends one ad object to the If node.  
    - *Edge Cases:* Empty ads array leads to no output; code errors if ads property missing.  
    - *Version:* 2

  - **🚨 Is Ad Underperforming?**  
    - *Type:* If (Conditional)  
    - *Role:* Checks if the ad's estimated CTR is less than 1% and estimated CPA greater than $10 to flag underperformance.  
    - *Configuration:* Two numeric conditions combined with AND:  
      - `$json.estimated_ctr < 1`  
      - `$json.estimated_cpa > 10`  
    - *Inputs:* Single ad object from Code node.  
    - *Outputs:*  
      - True branch: underperforming → “📬 Send Alert Email”.  
      - False branch: performing well → “✅ Do Nothing (Ad is OK)”.  
    - *Edge Cases:* Missing or malformed CTR/CPA values cause evaluation errors.  
    - *Version:* 2.2

#### 2.4 Alerting & No-Action Handling

- **Overview:**  
  This block handles the alerting mechanism: sending an email notification if the ad is underperforming or doing nothing if the ad is satisfactory.

- **Nodes Involved:**  
  - 📬 Send Alert Email (Gmail Node)  
  - ✅ Do Nothing (Ad is OK) (No Operation Node)

- **Node Details:**

  - **📬 Send Alert Email**  
    - *Type:* Gmail  
    - *Role:* Sends an HTML-formatted email alert detailing the underperforming ad’s key info.  
    - *Configuration:*  
      - Recipient: `shahkar.genai@gmail.com`  
      - Subject: "🚨 Underperforming Ad Detected"  
      - Message body includes ad title, status, estimated CTR and CPA, start date, platforms, ad text, and a media URL link.  
    - *Inputs:* From If node “True” branch.  
    - *Outputs:* None (end node).  
    - *Credentials:* Gmail OAuth2 configured.  
    - *Edge Cases:* Email delivery failures, credential expiration, or message formatting issues.  
    - *Version:* 2.1

  - **✅ Do Nothing (Ad is OK)**  
    - *Type:* No Operation  
    - *Role:* Ends workflow silently if the ad passes performance checks.  
    - *Inputs:* From If node “False” branch.  
    - *Outputs:* None (end node).  
    - *Edge Cases:* None.  
    - *Version:* 1

---

### 3. Summary Table

| Node Name                     | Node Type                            | Functional Role                         | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                               |
|-------------------------------|------------------------------------|---------------------------------------|---------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------|
| 🔁 Check Ads Every Hour        | Schedule Trigger                   | Trigger workflow every hour           | None                            | 📥 Set facebook ad dashboard URL | Section 1: Trigger & Input Setup - automatic hourly schedule; sets monitoring frequency                   |
| 📥 Set facebook ad dashboard URL | Set                                | Defines Facebook Ads Library URL      | 🔁 Check Ads Every Hour          | 🧠 Scrape Ads via MCP Agent      | Section 1: Sets keyword/brand URL for scraping                                                           |
| 🧠 Scrape Ads via MCP Agent    | AI Agent (Langchain Agent)         | Scrapes ads data via Bright Data MCP  | 📥 Set facebook ad dashboard URL | 🔄 Return One Ad at a Time       | Section 2: AI Agent scraping ads using MCP and OpenAI                                                    |
| MCP Client                    | MCP Client Tool                    | Executes scraping tool via Bright Data| AI Agent sub-node                | AI Agent sub-node                | Part of AI Agent sub-nodes                                                                                |
| OpenAI Chat Model             | Language Model (OpenAI Chat)       | Processes scraping instructions       | AI Agent sub-node                | AI Agent sub-node                | Part of AI Agent sub-nodes                                                                                |
| Auto-fixing Output Parser     | Langchain Output Parser (Auto)     | Cleans and fixes output JSON           | OpenAI Chat Model1               | AI Agent sub-node                | Part of AI Agent sub-nodes                                                                                |
| OpenAI Chat Model1            | Language Model (OpenAI Chat)       | Further refines structured output      | Auto-fixing Output Parser        | Structured Output Parser1        | Part of AI Agent sub-nodes                                                                                |
| Structured Output Parser1     | Structured Output Parser           | Parses JSON output against schema      | OpenAI Chat Model1               | Auto-fixing Output Parser        | Part of AI Agent sub-nodes                                                                                |
| 🔄 Return One Ad at a Time     | Code                              | Returns one ad from ads list per run   | 🧠 Scrape Ads via MCP Agent      | 🚨 Is Ad Underperforming?        | Section 3: Processes single ad per execution                                                             |
| 🚨 Is Ad Underperforming?      | If                                | Checks if ad performance is below threshold | 🔄 Return One Ad at a Time       | 📬 Send Alert Email / ✅ Do Nothing (Ad is OK) | Section 3: Evaluates ad CTR and CPA against thresholds                                                    |
| 📬 Send Alert Email            | Gmail                             | Sends alert email for underperforming ad | 🚨 Is Ad Underperforming? (true) | None                            | Section 4: Sends alert email with ad details                                                             |
| ✅ Do Nothing (Ad is OK)       | No Operation                      | Ends workflow silently for performing ads | 🚨 Is Ad Underperforming? (false) | None                            | Section 4: No action required if ad is performing well                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to execute every 1 hour.  
   - Name it “🔁 Check Ads Every Hour”.

2. **Create a Set node:**  
   - Type: Set  
   - Add a string field named `url`.  
   - Set value to the Facebook Ads Library URL filtered for your brand/keyword, e.g.:  
     `https://www.facebook.com/ads/library/?active_status=all&ad_type=all&country=US&q=nike&sort_data[direction]=desc&sort_data[mode]=relevancy_monthly_grouped`  
   - Name it “📥 Set facebook ad dashboard URL”.  
   - Connect the Schedule Trigger node output to this node input.

3. **Create an AI Agent node (Langchain Agent):**  
   - Name: “🧠 Scrape Ads via MCP Agent”  
   - Configure prompt text:  
     `"scrape the data from the facebook ads dashboard:\n{{ $json.url }}"`  
   - Enable output parsing.  
   - Add sub-nodes within the agent:  
     - MCP Client tool:  
       - Tool Name: `scrape_as_markdown`  
       - Operation: `executeTool`  
       - Tool Parameters: dynamically injected from AI prompt override.  
       - Credentials: Bright Data MCP Client API.  
     - OpenAI Chat Model node:  
       - Model: `gpt-4o-mini`  
       - Credentials: OpenAI API.  
     - Auto-fixing Output Parser node (no special config).  
     - Another OpenAI Chat Model node (same model and credentials).  
     - Structured Output Parser node:  
       - Provide JSON schema example representing ads data (fields like ad_id, ad_title, estimated_ctr, estimated_cpa, media_url, etc.).  
   - Connect nodes internally as per the agent’s workflow: OpenAI Chat → Structured Output Parser → Auto-fixing Parser → MCP Client → back to agent output.  
   - Connect output of “📥 Set facebook ad dashboard URL” to this agent node.

4. **Create a Code node:**  
   - Name: “🔄 Return One Ad at a Time”  
   - Paste JavaScript code to extract the first ad from the array:  
     ```js
     const ads = $json.ads;
     if (ads && ads.length > 0) {
       return [ads[0]];
     } else {
       return [];
     }
     ```  
   - Connect the AI Agent output to this Code node.

5. **Create an If node:**  
   - Name: “🚨 Is Ad Underperforming?”  
   - Set conditions:  
     - Condition 1: `$json.estimated_ctr < 1` (Number)  
     - Condition 2: `$json.estimated_cpa > 10` (Number)  
     - Both combined with AND.  
   - Connect Code node output to this If node.

6. **Create a Gmail node:**  
   - Name: “📬 Send Alert Email”  
   - Credentials: Gmail OAuth2 account.  
   - Configure recipient email (e.g., `shahkar.genai@gmail.com`).  
   - Subject: “🚨 Underperforming Ad Detected”  
   - Message (HTML): Include dynamic fields from ad JSON like ad_title, status, estimated_ctr, estimated_cpa, start_date, platforms (joined), ad_text, and media_url as link.  
   - Connect If node “true” branch to this Gmail node.

7. **Create a No Operation node:**  
   - Name: “✅ Do Nothing (Ad is OK)”  
   - Connect If node “false” branch to this node.

8. **Verify connections:**  
   - Schedule Trigger → Set URL → AI Agent → Code (One Ad) → If (Check Underperformance) → True → Gmail Alert  
   - → False → No Operation

9. **Set credentials:**  
   - Configure Bright Data MCP Client credentials on MCP Client node.  
   - Configure OpenAI API credentials on OpenAI Chat Model nodes.  
   - Configure Gmail OAuth2 credentials on Gmail node.

10. **Test the workflow:**  
    - Run manually or wait for scheduled trigger.  
    - Verify scraping, parsing, evaluation, and email sending.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| For support or questions, contact Yaron at Yaron@nofluff.online                                                                          | Workflow Assistance contact                             |
| Explore additional tips and tutorials by Yaron Been on YouTube and LinkedIn                                                               | YouTube: https://www.youtube.com/@YaronBeen/videos     |
| LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                                                          |                                                        |
| Bright Data affiliate link for supporting content creation                                                                                | https://get.brightdata.com/1tndi4600b25                 |
| Workflow designed to keep inbox clean by alerting only on actionable underperforming ads                                                  | Workflow design note                                   |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It complies fully with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.