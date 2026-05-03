Personalized Customer Retention Campaigns with GPT-4o Analytics & Gmail

https://n8nworkflows.xyz/workflows/personalized-customer-retention-campaigns-with-gpt-4o-analytics---gmail-11857


# Personalized Customer Retention Campaigns with GPT-4o Analytics & Gmail

### 1. Workflow Overview

This workflow automates personalized customer retention and win-back campaigns by leveraging advanced AI analytics (GPT-4o) combined with customer data and email engagement via Gmail. It targets businesses seeking to proactively identify customers with churn risk and deliver tailored win-back offers to maximize retention and revenue.

The workflow’s logical blocks are:

- **1.1 Scheduled Trigger & Configuration:** Initiates daily churn risk check and loads key configuration parameters.
- **1.2 Customer Data Retrieval & Churn Signal Calculation:** Fetches detailed customer activity and calculates churn risk scores.
- **1.3 Risk Segmentation & Intelligence Enrichment:** Segments customers by risk level and enriches data with sentiment and usage analytics.
- **1.4 AI-Driven Strategy Agents:** For each risk segment (High Risk, Low Risk), AI agents create personalized win-back strategies and emails.
- **1.5 Email Engagement Monitoring & Follow-up:** Tracks email responses to initial outreach and triggers follow-up strategies if needed.
- **1.6 Campaign Aggregation & Performance Analysis:** Aggregates campaign results, calculates Customer Lifetime Value (CLV) impact, and provides strategy optimization recommendations.
- **1.7 Strategy Parameter Update:** Updates external strategy API with insights from campaign performance analysis.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Configuration

**Overview:**  
Triggers the workflow daily at 2 AM and sets static configuration parameters for API endpoints, thresholds, and campaign timing.

**Nodes Involved:**  
- Daily Churn Check  
- Workflow Configuration

**Node Details:**

- **Daily Churn Check**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow daily at 2:00 AM.  
  - Configuration: Trigger time set to 2 AM daily.  
  - Input: None  
  - Output: Connects to Workflow Configuration  
  - Failures: Scheduling misconfigurations or n8n runtime downtime.

- **Workflow Configuration**  
  - Type: Set Node  
  - Role: Defines workflow parameters like API URLs, churn thresholds (e.g., 0.7), risk thresholds (high: 85, medium: 70), and email wait time (48 hours).  
  - Configuration: Static values with placeholders for URLs and numeric thresholds.  
  - Input: From Daily Churn Check  
  - Output: Connects to Fetch Customer Data  
  - Failures: Misconfigured parameters can cause API call failures downstream.

---

#### 1.2 Customer Data Retrieval & Churn Signal Calculation

**Overview:**  
Fetches comprehensive customer data including activity, purchases, and subscriptions, then calculates churn risk signals based on recency, engagement, and subscription status.

**Nodes Involved:**  
- Fetch Customer Data  
- Calculate Churn Signals  
- Check Churn Risk

**Node Details:**

- **Fetch Customer Data**  
  - Type: HTTP Request  
  - Role: Calls external customer API endpoint with query params to include activity, purchases, and subscriptions.  
  - Configuration: URL from Workflow Configuration, query parameters fixed.  
  - Input: From Workflow Configuration  
  - Output: Connects to Calculate Churn Signals  
  - Failures: HTTP errors (timeout, 4xx/5xx), incorrect API URL or authentication issues.

- **Calculate Churn Signals**  
  - Type: Code (JavaScript)  
  - Role: Calculates days since last activity and purchase, subscription status, engagement score (based on logins, email opens, feature usage), and churn risk score (0-100).  
  - Configuration: Custom JS logic using input JSON data, with weighted scoring for churn factors.  
  - Input: From Fetch Customer Data  
  - Output: Connects to Check Churn Risk  
  - Failures: Data format issues, null or missing fields, calculation errors.

- **Check Churn Risk**  
  - Type: If Node  
  - Role: Filters customers with churn risk score >= configured churnThreshold (0.7).  
  - Configuration: Numeric comparison of churn risk score against threshold.  
  - Input: From Calculate Churn Signals  
  - Output: On true, connects to Fetch Sentiment Data and Fetch Product Usage Analytics  
  - Failures: Expression evaluation errors with missing data.

---

#### 1.3 Risk Segmentation & Intelligence Enrichment

**Overview:**  
Segments customers into High Risk, Medium Risk, and Low Risk groups based on churn risk score, and enriches customer intelligence with sentiment analysis and product usage analytics.

**Nodes Involved:**  
- Fetch Sentiment Data  
- Fetch Product Usage Analytics  
- Merge Customer Intelligence  
- Segment by Risk Level

**Node Details:**

- **Fetch Sentiment Data**  
  - Type: HTTP Request  
  - Role: Retrieves sentiment data including social sentiment for each customer.  
  - Configuration: URL from Workflow Configuration, query params include customer_id and include_social=true.  
  - Input: From Check Churn Risk (true branch)  
  - Output: Connects to Merge Customer Intelligence  
  - Failures: HTTP errors, invalid customer_id, API downtime.

- **Fetch Product Usage Analytics**  
  - Type: HTTP Request  
  - Role: Retrieves last 90 days product usage analytics per customer.  
  - Configuration: URL from Workflow Configuration, query params customer_id and period_days=90.  
  - Input: From Check Churn Risk (true branch)  
  - Output: Connects to Merge Customer Intelligence  
  - Failures: Similar to Fetch Sentiment Data.

- **Merge Customer Intelligence**  
  - Type: Merge Node  
  - Role: Combines outputs from Fetch Sentiment Data, Fetch Product Usage Analytics, and customer data into a single enriched JSON object per customer.  
  - Configuration: Combine by position (align by index).  
  - Input: From Fetch Sentiment Data, Fetch Product Usage Analytics, and Check Churn Risk node output.  
  - Output: Connects to Segment by Risk Level  
  - Failures: Mismatched input counts leading to incomplete merges.

- **Segment by Risk Level**  
  - Type: Switch Node  
  - Role: Categorizes customers into “High Risk” if churn_risk_score >= highRiskThreshold (85), “Medium Risk” if >= mediumRiskThreshold (70), else Low Risk.  
  - Configuration: Numeric comparisons to thresholds from Workflow Configuration.  
  - Input: From Merge Customer Intelligence  
  - Output:  
    - High Risk → High Risk Strategy Agent  
    - Medium Risk → Wait for Email Response then Low Risk Strategy Agent  
    - Low Risk → Wait for Email Response then Low Risk Strategy Agent  
  - Failures: Misclassification if churn_risk_score missing or invalid.

---

#### 1.4 AI-Driven Strategy Agents

**Overview:**  
Separate AI agents generate personalized win-back strategies and email content for High Risk and Low Risk customers using GPT-4o and GPT-4o-mini models, integrated with various tools for enriched context.

**Nodes Involved:**  
- High Risk Strategy Agent  
- Low Risk Strategy Agent  
- Pricing API Tool  
- MCP Client Tool  
- Gmail Tool  
- OpenAI Chat Models (gpt-4o and gpt-4o-mini)  
- Structured Output Parsers (for each strategy agent)

**Node Details:**

- **High Risk Strategy Agent**  
  - Type: Langchain Agent Node  
  - Role: Uses detailed customer intelligence to create aggressive win-back offers with premium incentives (30-50% discounts, extended trials, VIP support).  
  - Configuration: System message instructs empathetic tone, urgency, and personal touches, using MCP Client Tool and Gmail Tool.  
  - Input: From Segment by Risk Level (High Risk branch)  
  - Output: Connects to Wait for Email Response and Merge All Campaign Paths  
  - Failures: AI model errors, input JSON size limits, tool API failures.

- **Low Risk Strategy Agent**  
  - Type: Langchain Agent Node  
  - Role: Designs gentle, proactive engagement strategies with modest offers (5-15% discounts, bonus features), focusing on relationship building.  
  - Configuration: Uses Pricing API Tool, MCP Client Tool, and Gmail Tool with a system message emphasizing light tone and feedback requests.  
  - Input: From Segment by Risk Level (Medium and Low Risk branches via Wait for Email Response)  
  - Output: Connects to Wait for Email Response and Merge All Campaign Paths  
  - Failures: Same as High Risk Agent.

- **Pricing API Tool**  
  - Type: HTTP Request Tool  
  - Role: Provides current pricing and promotion info to Low Risk Strategy Agent.  
  - Configuration: Endpoint URL placeholder, expects customer pricing data.  
  - Input: Used as AI Tool by Low Risk Strategy Agent  
  - Output: N/A (used as tool in AI prompt)  
  - Failures: API downtime or incorrect pricing data.

- **MCP Client Tool**  
  - Type: Langchain MCP Client Tool  
  - Role: Supplies detailed customer history and preferences to AI agents for context.  
  - Configuration: Endpoint URL placeholder for MCP Server.  
  - Input: Used by both High and Low Risk Strategy Agents  
  - Output: N/A  
  - Failures: Connectivity or auth failures.

- **Gmail Tool**  
  - Type: Gmail Node  
  - Role: Sends personalized email messages crafted by AI agents to customers.  
  - Configuration: Uses OAuth2 Gmail credentials; parameters dynamically set from AI output fields (customerEmail, emailSubject, emailBody).  
  - Input: Invoked inside AI agents as a tool  
  - Output: N/A  
  - Failures: Gmail quota limits, OAuth token expiry, invalid email addresses.

- **OpenAI Chat Models**  
  - Types: Langchain OpenAI Chat (gpt-4o and gpt-4o-mini)  
  - Role: Provide language model capabilities for AI agents and follow-up agent.  
  - Configuration: OpenAI API credentials configured, model versions specified.  
  - Input: Within AI agent nodes  
  - Output: AI-generated structured offers and email text.  
  - Failures: API request limits, model unavailability.

- **Structured Output Parsers (1, 3)**  
  - Type: Langchain Output Parser Structured  
  - Role: Parse AI agent JSON responses into structured fields: customerEmail, offerType, offerValue, emailSubject, emailBody, reasoning.  
  - Configuration: Manual JSON schema for expected output format.  
  - Input: AI agent output  
  - Output: Connected back to AI agent nodes (parsing step)  
  - Failures: Parsing errors if AI response does not conform to schema.

---

#### 1.5 Email Engagement Monitoring & Follow-up

**Overview:**  
Monitors email open/click activity and triggers follow-up AI agent if the customer does not engage with the initial email.

**Nodes Involved:**  
- Wait for Email Response  
- Check Email Engagement  
- Check Response Status  
- Follow-up Strategy Agent  
- Gmail Tool (via Follow-up Agent)  
- OpenAI Chat Model (gpt-4o)  
- Structured Output Parser (for follow-up)

**Node Details:**

- **Wait for Email Response**  
  - Type: Wait Node with Webhook Resume  
  - Role: Pauses workflow to wait for email engagement event (up to 48 hours configured).  
  - Configuration: Resume via webhook, no timeout specified here.  
  - Input: From High Risk and Low Risk Strategy Agents  
  - Output: Connects to Check Email Engagement  
  - Failures: Webhook not triggered if no engagement event received.

- **Check Email Engagement**  
  - Type: HTTP Request  
  - Role: Queries email engagement API to check if email was opened or clicked.  
  - Configuration: Uses customerEmail and campaign_id from input JSON, URL from Workflow Configuration.  
  - Input: From Wait for Email Response  
  - Output: Connects to Check Response Status  
  - Failures: API errors, missing engagement data.

- **Check Response Status**  
  - Type: If Node  
  - Role: Checks if email_opened or email_clicked is false; if no engagement, triggers Follow-up Strategy Agent; else proceeds to Merge All Campaign Paths.  
  - Configuration: OR condition on email_opened and email_clicked boolean fields.  
  - Input: From Check Email Engagement  
  - Output:  
    - No engagement → Follow-up Strategy Agent  
    - Engagement → Merge All Campaign Paths  
  - Failures: Logic errors if data fields missing.

- **Follow-up Strategy Agent**  
  - Type: Langchain Agent Node  
  - Role: Creates alternative messaging and offers for customers who did not engage initially, and sends follow-up email.  
  - Configuration: System message instructs different subject line, tone, and incentives. Uses Gmail Tool.  
  - Input: From Check Response Status (no engagement path)  
  - Output: Connects to Aggregate Campaign Results  
  - Failures: AI errors, email send failures.

- **Gmail Tool (within Follow-up Agent)**  
  - Same as above; sends follow-up email.

- **OpenAI Chat Model (gpt-4o)**  
  - Used internally by Follow-up Strategy Agent.

- **Structured Output Parser (3)**  
  - Parses follow-up agent output JSON.

---

#### 1.6 Campaign Aggregation & Performance Analysis

**Overview:**  
Aggregates all campaign results, analyzes overall performance metrics, calculates Customer Lifetime Value impact and ROI, and generates strategic recommendations.

**Nodes Involved:**  
- Merge All Campaign Paths  
- Aggregate Campaign Results  
- Calculate CLV Impact  
- Analyze Performance Data

**Node Details:**

- **Merge All Campaign Paths**  
  - Type: Merge Node  
  - Role: Combines outputs from High Risk, Low Risk, and Follow-up Strategy Agents into a unified dataset.  
  - Configuration: Combine by position.  
  - Input: From Wait for Email Response (Low Risk), High Risk Strategy Agent, and Follow-up Strategy Agent  
  - Output: Connects to Aggregate Campaign Results  
  - Failures: Input count mismatches cause incomplete merges.

- **Aggregate Campaign Results**  
  - Type: Aggregate Node  
  - Role: Aggregates campaign results, specifically grouping by offerType.  
  - Configuration: Aggregates on field "offerType".  
  - Input: From Merge All Campaign Paths  
  - Output: Connects to Calculate CLV Impact  
  - Failures: Aggregation errors if fields missing or malformed.

- **Calculate CLV Impact**  
  - Type: Code Node  
  - Role: Calculates retention metrics, saved revenue, campaign cost, ROI, and segment-level analysis by churn risk.  
  - Configuration: Custom JS with assumptions on average monthly revenue ($100), customer lifetime (24 months), baseline retention (20%).  
  - Input: From Aggregate Campaign Results  
  - Output: Connects to Analyze Performance Data  
  - Failures: Data inconsistencies, division by zero, or missing fields.

- **Analyze Performance Data**  
  - Type: Code Node  
  - Role: Analyzes overall campaign performance, conversion and response rates, identifies best performing offers and patterns, and provides recommendations to optimize strategy.  
  - Configuration: Custom JS analyzing conversion metrics and generating prioritized suggestions.  
  - Input: From Calculate CLV Impact  
  - Output: Connects to Update Strategy Parameters  
  - Failures: JS errors, empty input data.

---

#### 1.7 Strategy Parameter Update

**Overview:**  
Updates external strategy API endpoint with refined churn thresholds, best performing offers, and timing recommendations based on analyzed campaign data.

**Nodes Involved:**  
- Update Strategy Parameters

**Node Details:**

- **Update Strategy Parameters**  
  - Type: HTTP Request  
  - Role: Sends PUT request to external strategy API with updated parameters derived from performance analysis.  
  - Configuration: URL from Workflow Configuration; body parameters for optimalChurnThreshold, bestPerformingOfferTypes, and timingRecommendations pulled from Analyze Performance Data node output.  
  - Input: From Analyze Performance Data  
  - Output: None (end of workflow)  
  - Failures: HTTP request failures, invalid payloads, API authentication failures.

---

### 3. Summary Table

| Node Name                 | Node Type                               | Functional Role                                    | Input Node(s)                      | Output Node(s)                                    | Sticky Note                                                      |
|---------------------------|---------------------------------------|---------------------------------------------------|----------------------------------|--------------------------------------------------|-----------------------------------------------------------------|
| Daily Churn Check          | Schedule Trigger                      | Triggers workflow daily at 2 AM                    | None                             | Workflow Configuration                            | ## Data Ingestion & Validation: Ensures clean input             |
| Workflow Configuration     | Set                                  | Defines API URLs, thresholds, and campaign params | Daily Churn Check                | Fetch Customer Data                              |                                                                 |
| Fetch Customer Data        | HTTP Request                         | Retrieves customer data with activity and purchases| Workflow Configuration           | Calculate Churn Signals                           |                                                                 |
| Calculate Churn Signals    | Code                                | Calculates churn risk and engagement scores        | Fetch Customer Data               | Check Churn Risk                                 |                                                                 |
| Check Churn Risk           | If                                  | Filters customers above churn risk threshold        | Calculate Churn Signals           | Fetch Sentiment Data, Fetch Product Usage Analytics |                                                                 |
| Fetch Sentiment Data       | HTTP Request                        | Gets customer sentiment and social data             | Check Churn Risk                 | Merge Customer Intelligence                      |                                                                 |
| Fetch Product Usage Analytics | HTTP Request                     | Gets product usage analytics data                    | Check Churn Risk                 | Merge Customer Intelligence                      |                                                                 |
| Merge Customer Intelligence| Merge                              | Combines enriched customer data                      | Fetch Sentiment Data, Fetch Product Usage Analytics, Check Churn Risk | Segment by Risk Level                              |                                                                 |
| Segment by Risk Level      | Switch                             | Segments customers into High, Medium, Low risk      | Merge Customer Intelligence       | High Risk Strategy Agent, Wait for Email Response |                                                                 |
| High Risk Strategy Agent   | Langchain Agent                    | Creates aggressive win-back offers for high risk    | Segment by Risk Level             | Wait for Email Response, Merge All Campaign Paths| ## AI-Powered Enrichment: Applies GPT-4o for intelligence       |
| Low Risk Strategy Agent    | Langchain Agent                    | Creates gentle engagement offers for low risk       | Segment by Risk Level via Wait for Email Response | Wait for Email Response, Merge All Campaign Paths|                                                                 |
| MCP Client Tool            | Langchain MCP Client Tool           | Provides detailed customer preferences               | Used by AI Agents                | N/A                                              |                                                                 |
| Pricing API Tool           | HTTP Request Tool                   | Supplies pricing and promotion info                   | Used by Low Risk Strategy Agent  | N/A                                              |                                                                 |
| Gmail Tool                 | Gmail Node                        | Sends personalized emails                             | Used inside AI Agents            | N/A                                              |                                                                 |
| Wait for Email Response    | Wait Node                         | Pauses workflow awaiting email engagement            | High Risk Strategy Agent, Low Risk Strategy Agent | Check Email Engagement                            |                                                                 |
| Check Email Engagement     | HTTP Request                     | Checks if email was opened or clicked                 | Wait for Email Response          | Check Response Status                            |                                                                 |
| Check Response Status      | If                              | Determines if follow-up email needed                  | Check Email Engagement           | Follow-up Strategy Agent, Merge All Campaign Paths |                                                                 |
| Follow-up Strategy Agent   | Langchain Agent                 | Creates alternative follow-up offers and sends email | Check Response Status (no engagement) | Aggregate Campaign Results                      |                                                                 |
| OpenAI Chat Model          | Langchain LM Chat OpenAI         | AI model used by Follow-up Strategy Agent             | Internal to AI nodes             | Internal                                         |                                                                 |
| Structured Output Parser (3)| Langchain Output Parser Structured | Parses follow-up AI output into structured fields    | Follow-up Strategy Agent         | Follow-up Strategy Agent                          |                                                                 |
| Merge All Campaign Paths   | Merge                           | Combines all campaign paths outputs                   | High Risk Strategy Agent, Low Risk Strategy Agent, Follow-up Strategy Agent | Aggregate Campaign Results                      |                                                                 |
| Aggregate Campaign Results | Aggregate                      | Aggregates campaign data by offer type                 | Merge All Campaign Paths         | Calculate CLV Impact                             |                                                                 |
| Calculate CLV Impact       | Code                         | Calculates financial impact and retention metrics     | Aggregate Campaign Results       | Analyze Performance Data                         |                                                                 |
| Analyze Performance Data   | Code                         | Analyzes campaign success and generates recommendations| Calculate CLV Impact             | Update Strategy Parameters                       |                                                                 |
| Update Strategy Parameters | HTTP Request                 | Sends updated strategy parameters to external API     | Analyze Performance Data         | None                                             |                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node** named **Daily Churn Check**:  
   - Set to trigger daily at 2:00 AM UTC.

2. **Add a Set node** named **Workflow Configuration**:  
   - Add string parameters for all API URLs with placeholders: customerApiUrl, trackingApiUrl, strategyApiUrl, sentimentApiUrl, usageAnalyticsApiUrl, emailEngagementApiUrl.  
   - Add number parameters: churnThreshold (0.7), inactiveDays (30), highRiskThreshold (85), mediumRiskThreshold (70), emailWaitHours (48).  
   - Connect Daily Churn Check → Workflow Configuration.

3. **Add an HTTP Request node** named **Fetch Customer Data**:  
   - URL parameter set to `{{$node["Workflow Configuration"].json.customerApiUrl}}`  
   - Query parameters: include_activity=true, include_purchases=true, include_subscriptions=true.  
   - Connect Workflow Configuration → Fetch Customer Data.

4. **Add a Code node** named **Calculate Churn Signals**:  
   - Paste provided JavaScript to calculate churn risk score and enrich customers.  
   - Connect Fetch Customer Data → Calculate Churn Signals.

5. **Add an If node** named **Check Churn Risk**:  
   - Condition: churn_risk_score >= churnThreshold (from Workflow Configuration).  
   - Connect Calculate Churn Signals → Check Churn Risk.

6. **Add two HTTP Request nodes** named **Fetch Sentiment Data** and **Fetch Product Usage Analytics**:  
   - Sentiment Data URL from Workflow Configuration’s sentimentApiUrl, query with customer_id and include_social=true.  
   - Usage Analytics URL from usageAnalyticsApiUrl, query with customer_id and period_days=90.  
   - Connect Check Churn Risk (true path) → both nodes.

7. **Add a Merge node** named **Merge Customer Intelligence**:  
   - Combine mode: combine by position, inputs: outputs of Fetch Sentiment Data, Fetch Product Usage Analytics, and Check Churn Risk output.  
   - Connect both Fetch nodes → Merge Customer Intelligence.

8. **Add a Switch node** named **Segment by Risk Level**:  
   - Rules for churn_risk_score:  
     - High Risk: >= highRiskThreshold  
     - Medium Risk: >= mediumRiskThreshold (and < highRiskThreshold)  
     - Else Low Risk  
   - Connect Merge Customer Intelligence → Segment by Risk Level.

9. **Create Langchain agent nodes**:  
   - **High Risk Strategy Agent**: use GPT-4o model, system message instructing aggressive retention strategy with premium offers, using MCP Client Tool and Gmail Tool.  
   - **Low Risk Strategy Agent**: use GPT-4o-mini model, system message instructing gentle engagement, using Pricing API Tool, MCP Client Tool, and Gmail Tool.  
   - Connect High Risk output from Segment by Risk Level → High Risk Strategy Agent.  
   - Connect Medium and Low Risk outputs → Wait for Email Response node (next step).

10. **Create a Wait node** named **Wait for Email Response**:  
    - Resume via webhook, wait period configured externally (48 hours from config).  
    - Connect from Low Risk branch and Medium Risk branch after segmentation and Low Risk Strategy Agent.

11. **Add an HTTP Request node** named **Check Email Engagement**:  
    - URL from emailEngagementApiUrl, query with customerEmail and campaign_id from previous outputs.  
    - Connect Wait for Email Response → Check Email Engagement.

12. **Add an If node** named **Check Response Status**:  
    - Condition: email_opened OR email_clicked is false → trigger Follow-up Strategy Agent, else → Merge All Campaign Paths.  
    - Connect Check Email Engagement → Check Response Status.

13. **Add a Langchain agent node** named **Follow-up Strategy Agent**:  
    - Use GPT-4o model, system message instructing alternate messaging and offers for non-engaged customers, using Gmail Tool.  
    - Connect Check Response Status (no engagement path) → Follow-up Strategy Agent.

14. **Add a Merge node** named **Merge All Campaign Paths**:  
    - Combine mode: combine by position, inputs from High Risk Strategy Agent, Low Risk Strategy Agent, and Follow-up Strategy Agent.  
    - Connect Follow-up Strategy Agent and Strategy Agents outputs → Merge All Campaign Paths.

15. **Add an Aggregate node** named **Aggregate Campaign Results**:  
    - Aggregate by field "offerType".  
    - Connect Merge All Campaign Paths → Aggregate Campaign Results.

16. **Add a Code node** named **Calculate CLV Impact**:  
    - Paste provided JavaScript for retention metrics and ROI calculations.  
    - Connect Aggregate Campaign Results → Calculate CLV Impact.

17. **Add a Code node** named **Analyze Performance Data**:  
    - Paste provided JavaScript analyzing conversion rates, offer effectiveness, and generating recommendations.  
    - Connect Calculate CLV Impact → Analyze Performance Data.

18. **Add an HTTP Request node** named **Update Strategy Parameters**:  
    - PUT method to strategyApiUrl from Workflow Configuration.  
    - Body parameters: optimalChurnThreshold, bestPerformingOfferTypes, timingRecommendations from Analyze Performance Data output.  
    - Connect Analyze Performance Data → Update Strategy Parameters.

19. **Configure credentials**:  
    - OpenAI API key for GPT-4o models.  
    - Gmail OAuth2 credentials for sending emails.  
    - Ensure HTTP Request nodes have valid API keys or authentication as needed.

20. **Add Structured Output Parser nodes** after each AI agent to parse the JSON output according to the defined schema: customerEmail, offerType, offerValue, emailSubject, emailBody, reasoning.

21. **Test the workflow** incrementally, verifying data flows, API responses, and email delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                                                                                  |
|------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow automates end-to-end customer retention using GPT-4o and Gmail integration.            | Title: Personalized Customer Retention Campaigns with GPT-4o Analytics & Gmail                                                                                |
| Prerequisites: OpenAI API key, Gmail account (OAuth2), configured external APIs for customer data, sentiment, usage, pricing, and strategy endpoints. | Sticky Notes in workflow outline prerequisites and setup steps.                                                                                                |
| Customization options: Swap AI models (Claude, Gemini, Llama), modify analysis branches or output destinations. | Sticky Note1 in workflow.                                                                                                                                      |
| Benefits include 80% time savings by eliminating manual processing and enabling multi-perspective AI analysis. | Sticky Note1.                                                                                                                                                  |
| Workflow designed for business analysts, data scientists, and marketing operations teams.       | Sticky Note3 describes target users and workflow purpose.                                                                                                     |
| AI agents use tools integration to enhance context: MCP Client Tool for customer data, Pricing API for promotions, Gmail Tool for emails. | Detailed in AI Strategy Agents block.                                                                                                                         |
| Email engagement tracking enables dynamic follow-up strategies.                                 | Follow-up Strategy Agent adjusts messaging based on real-time email engagement analytics.                                                                     |
| Performance analysis includes ROI and CLV impact calculations to inform strategic decisions.    | Code nodes Calculate CLV Impact and Analyze Performance Data provide deep financial insights.                                                                 |

---

This comprehensive reference document covers the entire workflow structure, logic, node configurations, and integration points to enable advanced users and AI systems to fully understand, reproduce, and maintain the workflow.