Predict Customer Churn Risk & Create Interventions with GPT-4, Zendesk & HubSpot

https://n8nworkflows.xyz/workflows/predict-customer-churn-risk---create-interventions-with-gpt-4--zendesk---hubspot-7344


# Predict Customer Churn Risk & Create Interventions with GPT-4, Zendesk & HubSpot

### 1. Workflow Overview

This workflow is designed to predict customer churn risk 30 to 90 days in advance by aggregating and analyzing multiple data signals using AI, then automatically triggering personalized intervention actions to reduce churn. It is tailored for Customer Success (CS) teams aiming to shift from reactive to predictive customer management.

**Target Use Cases:**  
- Early identification of customers at risk of churn  
- Automated generation of personalized outreach content  
- Multi-source data integration: product usage, support tickets, billing, email engagement  
- Automated notifications and CRM updates based on risk level  

**Logical Blocks:**

- **1.1 Setup & Documentation:** Sticky notes with overview, purpose, setup instructions, and AI logic details.  
- **1.2 Data Collection:** Scheduled daily trigger fetching product usage, support tickets, and billing data.  
- **1.3 Data Merging & AI Risk Scoring:** Merge diverse data streams and run AI prediction of churn risk score.  
- **1.4 Risk-Based Routing:** Branch workflow based on risk levels (Critical, High, Medium, Low).  
- **1.5 Intervention Generation & Notifications:** Generate personalized interventions via AI, send emails, update CRM, and send Slack alerts for critical risks.  

---

### 2. Block-by-Block Analysis

#### 2.1 Setup & Documentation

**Overview:**  
This block consists of sticky notes providing high-level context, setup instructions, and explanation of AI logic. It ensures users understand the workflow's purpose, required credentials, environment variables, and the AI scoring methodology.

**Nodes Involved:**  
- Overview & Purpose (Sticky Note)  
- Setup Instructions (Sticky Note)  
- AI Analysis Logic (Sticky Note)  

**Node Details:**  

- **Overview & Purpose**  
  - Type: Sticky Note  
  - Purpose: Describes the workflow’s goal to predict churn risk using AI and outlines its innovative approach.  
  - Position: Top-left  
  - No inputs or outputs  
  - No version-specific requirements  
  - Edge cases: None  
  - Content covers security notes about API keys stored in environment variables.  

- **Setup Instructions**  
  - Type: Sticky Note  
  - Purpose: Lists environment variables and credentials needed to run the workflow, plus estimated setup time.  
  - Position: Below Overview & Purpose  
  - No inputs or outputs  
  - Edge cases: Users must ensure all credentials and environment variables are configured correctly to avoid failures downstream.  

- **AI Analysis Logic**  
  - Type: Sticky Note  
  - Purpose: Explains the AI scoring logic combining product usage, support sentiment, billing, email engagement, and contract dates to assign risk levels.  
  - Position: Top-right  
  - No inputs or outputs  

#### 2.2 Data Collection

**Overview:**  
Triggered daily, this block fetches raw data from three sources: product usage analytics, support tickets, and billing info. These feeds provide the input signals for AI risk analysis.

**Nodes Involved:**  
- Daily Risk Analysis (Cron)  
- Fetch Product Usage Data (HTTP Request)  
- Fetch Support Tickets (Zendesk)  
- Fetch Billing Data (Stripe)  

**Node Details:**  

- **Daily Risk Analysis**  
  - Type: Cron Trigger  
  - Purpose: Initiates the workflow daily to refresh data and re-assess risk.  
  - Configuration: Runs on a daily schedule (default timing not specified, assumed daily).  
  - Output: Triggers three parallel fetch nodes.  
  - Edge cases: Cron misconfiguration could delay risk assessments.  

- **Fetch Product Usage Data**  
  - Type: HTTP Request  
  - Purpose: Retrieves product usage events (login, feature usage, session end) from an analytics API (e.g., Mixpanel).  
  - Configuration:  
    - URL uses environment variable `ANALYTICS_API_URL`.  
    - Query parameters specify a 30-day window from today minus 30 days to today.  
    - Auth: HTTP header with API key from environment variable `ANALYTICS_API_KEY`.  
  - Inputs: Trigger from Cron node  
  - Outputs: JSON data of product usage events  
  - Edge cases: API downtime, auth failures, or rate limits could cause incomplete data.  

- **Fetch Support Tickets**  
  - Type: Zendesk node  
  - Purpose: Retrieves all support tickets to analyze sentiment and frequency.  
  - Configuration: Operation set to getAll tickets (default options).  
  - Inputs: From Cron node  
  - Outputs: JSON array of tickets  
  - Edge cases: Zendesk API quota or auth errors.  

- **Fetch Billing Data**  
  - Type: Stripe node  
  - Purpose: Lists customer billing data to detect payment delays or issues.  
  - Configuration: Resource set to customer, operation list.  
  - Inputs: From Cron node  
  - Outputs: Customer billing information  
  - Edge cases: Stripe API errors, auth issues.  

#### 2.3 Data Merging & AI Risk Scoring

**Overview:**  
This block consolidates data from all sources and applies AI to compute a risk score representing the likelihood of customer churn.

**Nodes Involved:**  
- Merge All Data Sources (Merge)  
- AI Risk Prediction (AI Transform)  

**Node Details:**  

- **Merge All Data Sources**  
  - Type: Merge node  
  - Purpose: Combines product usage, support tickets, and billing data into a single dataset for AI analysis.  
  - Inputs:  
    - Product Usage Data (index 0)  
    - Support Tickets (index 1)  
    - Billing Data (index 2) (Note: In workflow JSON only 2 inputs shown, billing data is connected from Fetch Billing Data to Merge All Data Sources without explicit index - assumed as index 2 or merged later)  
  - Outputs: Combined dataset JSON  
  - Edge cases: Missing or incomplete data from any source could bias AI predictions.  

- **AI Risk Prediction**  
  - Type: AI Transform (likely connected to OpenAI GPT-4 or similar)  
  - Purpose: Processes merged data with AI model to generate a churn risk score per customer.  
  - Inputs: Merged dataset  
  - Outputs: JSON with risk scores and possibly additional metadata (e.g., risk level)  
  - Edge cases: API rate limits, malformed input data causing AI errors, or unexpected model outputs.  
  - Version: Configured to use environment variable `OPENAI_MODEL_VERSION` (implied from setup notes).  

#### 2.4 Risk-Based Routing

**Overview:**  
Branches the workflow based on the AI risk score into categories: Critical, High, Medium, and Low risk, triggering different actions accordingly.

**Nodes Involved:**  
- Route by Risk Level (If node)  
- Check High Risk (If node)  

**Node Details:**  

- **Route by Risk Level**  
  - Type: If node  
  - Purpose: Checks if risk score is >= 90 (Critical) to route accordingly.  
  - Conditions: `$json.risk_score >= 90` for critical risk branch  
  - Inputs: AI Risk Prediction output  
  - Outputs:  
    - True: Critical risk branch (Generate Critical Intervention, Slack alert)  
    - False: Proceeds to Check High Risk node  

- **Check High Risk**  
  - Type: If node  
  - Purpose: Checks if risk score is >= 70 for high risk branch.  
  - Conditions: `$json.risk_score >= 70`  
  - Inputs: Output of Route by Risk Level false branch  
  - Outputs:  
    - True: High risk intervention generation  
    - False: Medium risk intervention generation (implicitly low risk is handled as medium or below)  

#### 2.5 Intervention Generation & Notifications

**Overview:**  
Generates personalized customer success interventions via AI, sends emails, updates CRM records, and sends Slack alerts for critical cases.

**Nodes Involved:**  
- Generate Critical Intervention (AI Transform)  
- Critical Risk Slack Alert (Slack)  
- Generate High Risk Intervention (AI Transform)  
- Generate Medium Risk Check-in (AI Transform)  
- Send Personalized Email (Email Send)  
- Update CRM with Risk Data (HubSpot)  

**Node Details:**  

- **Generate Critical Intervention**  
  - Type: AI Transform  
  - Purpose: Creates tailored intervention content for critical risk customers.  
  - Inputs: Critical risk data from Route by Risk Level  
  - Outputs: Intervention content for email and CRM update  

- **Critical Risk Slack Alert**  
  - Type: Slack node  
  - Purpose: Sends an alert message to a high-risk Slack channel notifying CS team of critical risk customers.  
  - Configuration:  
    - Text: "🚨 CRITICAL CUSTOMER RISK ALERT 🚨"  
    - Channel ID from environment variable `HIGH_RISK_SLACK_CHANNEL`  
  - Inputs: Critical risk branch from Route by Risk Level  
  - Edge cases: Slack API errors, invalid channel ID.  

- **Generate High Risk Intervention**  
  - Type: AI Transform  
  - Purpose: Generates personalized intervention for high-risk customers.  
  - Inputs: From Check High Risk true branch  
  - Outputs: Content for email and CRM update  

- **Generate Medium Risk Check-in**  
  - Type: AI Transform  
  - Purpose: Generates a less urgent, check-in style personalized message for medium risk customers.  
  - Inputs: From Check High Risk false branch  
  - Outputs: Content for email and CRM update  

- **Send Personalized Email**  
  - Type: Email Send  
  - Purpose: Sends the generated intervention message to the customer.  
  - Configuration:  
    - Subject line: From AI generated JSON field `subject_line`  
    - Recipient: Customer email from `customer_data.email`  
    - Sender: Customer Success team email from environment variable `CS_TEAM_EMAIL`  
  - Inputs: From all intervention generation nodes  
  - Edge cases: Email server issues, invalid email addresses  

- **Update CRM with Risk Data**  
  - Type: HubSpot node  
  - Purpose: Updates customer records with risk score and intervention details.  
  - Configuration: Operation set to update (specific fields not detailed)  
  - Inputs: From all intervention generation nodes  
  - Edge cases: API errors, permission issues, field mapping errors  

---

### 3. Summary Table

| Node Name                   | Node Type          | Functional Role                       | Input Node(s)                 | Output Node(s)                        | Sticky Note                                                                                                  |
|-----------------------------|--------------------|------------------------------------|------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Overview & Purpose          | Sticky Note        | Workflow purpose & context          | None                         | None                                | ## 🔮 AI Customer Success Risk Prediction Workflow... (full content as in 2.1)                                |
| Setup Instructions          | Sticky Note        | Setup and configuration notes       | None                         | None                                | ## 🎯 Setup & Configuration... (full content as in 2.1)                                                      |
| AI Analysis Logic           | Sticky Note        | Explanation of AI scoring logic     | None                         | None                                | ## 🧠 AI Risk Analysis Logic... (full content as in 2.1)                                                     |
| Daily Risk Analysis         | Cron               | Daily trigger to start risk analysis| None                         | Fetch Product Usage Data, Fetch Support Tickets, Fetch Billing Data |                                                                                                              |
| Fetch Product Usage Data    | HTTP Request       | Retrieve product usage events       | Daily Risk Analysis           | Merge All Data Sources               |                                                                                                              |
| Fetch Support Tickets       | Zendesk            | Retrieve support tickets            | Daily Risk Analysis           | Merge All Data Sources               |                                                                                                              |
| Fetch Billing Data          | Stripe             | Retrieve billing/customer data      | Daily Risk Analysis           | Merge All Data Sources               |                                                                                                              |
| Merge All Data Sources      | Merge              | Combine fetched data for AI         | Fetch Product Usage Data, Fetch Support Tickets, Fetch Billing Data | AI Risk Prediction                    |                                                                                                              |
| AI Risk Prediction          | AI Transform       | Compute churn risk score             | Merge All Data Sources        | Route by Risk Level                  |                                                                                                              |
| Route by Risk Level         | If                 | Route based on critical risk score  | AI Risk Prediction           | Generate Critical Intervention, Critical Risk Slack Alert, Check High Risk |                                                                                                              |
| Check High Risk             | If                 | Route based on high risk score       | Route by Risk Level          | Generate High Risk Intervention, Generate Medium Risk Check-in |                                                                                                              |
| Generate Critical Intervention | AI Transform    | Create intervention for critical risk| Route by Risk Level (critical branch) | Send Personalized Email, Update CRM with Risk Data |                                                                                                              |
| Critical Risk Slack Alert   | Slack              | Notify Slack channel for critical risk| Route by Risk Level (critical branch) | None                               |                                                                                                              |
| Generate High Risk Intervention| AI Transform    | Create intervention for high risk   | Check High Risk (true branch)| Send Personalized Email, Update CRM with Risk Data |                                                                                                              |
| Generate Medium Risk Check-in| AI Transform       | Create intervention for medium risk | Check High Risk (false branch)| Send Personalized Email, Update CRM with Risk Data |                                                                                                              |
| Send Personalized Email     | Email Send         | Email personalized intervention     | Generate Critical/High/Medium Interventions | None                               |                                                                                                              |
| Update CRM with Risk Data   | HubSpot            | Update customer record with risk    | Generate Critical/High/Medium Interventions | None                               |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Documentation:**  
   - Create three sticky notes labeled "Overview & Purpose", "Setup Instructions", and "AI Analysis Logic."  
   - Fill content with workflow purpose, setup variables and credentials, and AI logic details as described in section 2.1.

2. **Add Cron Node (Daily Risk Analysis):**  
   - Type: Cron  
   - Configure to run once daily at a chosen time (e.g., 00:00).  
   - This node triggers the data collection nodes.

3. **Add HTTP Request Node (Fetch Product Usage Data):**  
   - Set URL to `{{$env.ANALYTICS_API_URL}}/api/export`.  
   - Use GET method.  
   - Add query parameters:  
     - `from_date` = `{{$now.minus({days:30}).toISODate()}}`  
     - `to_date` = `{{$now.toISODate()}}`  
     - `event` = `["login","feature_used","session_end"]`  
   - Headers: Authorization Bearer token from env variable `ANALYTICS_API_KEY`.  
   - Authentication: HTTP Header Auth.  
   - Connect input from Cron node.

4. **Add Zendesk Node (Fetch Support Tickets):**  
   - Operation: Get All tickets.  
   - Credentials: Configure Zendesk credentials.  
   - Connect input from Cron node.

5. **Add Stripe Node (Fetch Billing Data):**  
   - Resource: Customer  
   - Operation: List  
   - Credentials: Configure Stripe credentials.  
   - Connect input from Cron node.

6. **Add Merge Node (Merge All Data Sources):**  
   - Mode: Merge by index (default).  
   - Connect inputs from:  
     - Fetch Product Usage Data (index 0)  
     - Fetch Support Tickets (index 1)  
     - Fetch Billing Data (index 2)  
   - Output combined data.

7. **Add AI Transform Node (AI Risk Prediction):**  
   - Connect input from Merge node.  
   - Configure to use OpenAI or Anthropic API with environment variable `OPENAI_MODEL_VERSION`.  
   - Prompt or model should score churn risk based on combined data.  
   - Output JSON must include `risk_score`.

8. **Add If Node (Route by Risk Level):**  
   - Input: AI Risk Prediction output.  
   - Condition: `$json.risk_score >= 90` for Critical risk branch.  
   - True branch: Connect to critical risk nodes.  
   - False branch: Connect to next If node.

9. **Add If Node (Check High Risk):**  
   - Condition: `$json.risk_score >= 70` for High risk branch.  
   - True branch: Connect to high risk intervention.  
   - False branch: Connect to medium risk intervention.

10. **Add AI Transform Node (Generate Critical Intervention):**  
    - Connect input from Route by Risk Level true branch.  
    - Configure prompt to generate intervention content tailored to critical risk customers.

11. **Add Slack Node (Critical Risk Slack Alert):**  
    - Configure Slack credentials.  
    - Text: "🚨 CRITICAL CUSTOMER RISK ALERT 🚨"  
    - Channel: From env variable `HIGH_RISK_SLACK_CHANNEL`.  
    - Connect input from Route by Risk Level true branch.

12. **Add AI Transform Node (Generate High Risk Intervention):**  
    - Connect input from Check High Risk true branch.  
    - Configure prompt for high risk intervention.

13. **Add AI Transform Node (Generate Medium Risk Check-in):**  
    - Connect input from Check High Risk false branch.  
    - Configure prompt for medium risk check-in message.

14. **Add Email Send Node (Send Personalized Email):**  
    - Connect inputs from all three intervention generation nodes.  
    - Subject: `{{$json.subject_line}}` from AI output.  
    - To: `{{$json.customer_data.email}}`  
    - From: Environment variable `CS_TEAM_EMAIL`.  
    - Configure SMTP or email credentials.

15. **Add HubSpot Node (Update CRM with Risk Data):**  
    - Operation: Update customer record.  
    - Connect inputs from all intervention generation nodes.  
    - Configure HubSpot credentials and specify fields to update with risk and intervention details.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                                 |
|------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| All API keys and sensitive credentials are stored securely as environment variables and n8n credentials.               | Security best practices                                                                                         |
| Workflow reduces churn by 15-30% through early AI-driven interventions.                                                | Business impact                                                                                                |
| Setup time estimated at ~45 minutes; ensure all external services (Mixpanel, Zendesk, Stripe, HubSpot, Slack, Email) are properly credentialed. | Setup and Configuration Notes                                                                                   |
| AI Model version is configurable via environment variable `OPENAI_MODEL_VERSION` to allow flexibility in AI upgrades.  | AI model management                                                                                            |
| Slack alerts notify CS teams in designated high-risk channels for urgent intervention.                                 | Incident response integration                                                                                   |
| Uses multi-source data fusion to improve risk prediction accuracy beyond single data signals.                          | Data science approach                                                                                           |

---

**Disclaimer:** The provided content is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.