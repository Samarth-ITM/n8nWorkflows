Subscription Revenue Optimizer with Stripe, Postgres & Gmail - Predictive Retention

https://n8nworkflows.xyz/workflows/subscription-revenue-optimizer-with-stripe--postgres---gmail---predictive-retention-6072


# Subscription Revenue Optimizer with Stripe, Postgres & Gmail - Predictive Retention

### 1. Workflow Overview

This workflow, titled **Subscription Revenue Optimizer with Stripe, Postgres & Gmail - Predictive Retention**, is designed to analyze subscription revenue data daily and automate targeted customer retention, upsell, and re-engagement campaigns based on predictive analytics. It integrates Stripe for customer details, Postgres for analytics data storage, and Gmail for sending personalized emails.

**Target Use Cases:**  
- SaaS businesses or subscription services aiming to optimize recurring revenue by identifying customers at risk of churn, candidates for upselling, or needing re-engagement.  
- Automating personalized email campaigns to improve customer retention and increase revenue.  
- Leveraging customer analytics data to make data-driven marketing decisions.

**Logical Blocks:**  
- **1.1 Scheduled Trigger & Configuration**: Initiates the workflow daily and sets key parameters for churn risk, upselling, and retention discount.  
- **1.2 Data Retrieval**: Queries customer analytics data from Postgres.  
- **1.3 Revenue Opportunity Analysis**: Processes customer data to score churn risk, upsell potential, and recommends marketing actions.  
- **1.4 Customer Segmentation & Campaign Dispatch**: Filters customers by recommended action and sends tailored email campaigns via Gmail, sourcing customer details from Stripe.  
- **1.5 Documentation & Notes**: Sticky notes provide contextual guidance on configuration and campaign logic.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Configuration

- **Overview:**  
  This block triggers the workflow daily at 6:00 AM and defines configurable parameters for churn risk threshold, upsell threshold, and retention discount rate.

- **Nodes Involved:**  
  - Daily Revenue Analysis (Schedule Trigger)  
  - Sticky Note (Revenue Optimization guidance)  
  - Revenue Settings (Set node with parameters)

- **Node Details:**

  - **Daily Revenue Analysis**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution daily at 6:00 AM using a cron expression `"0 6 * * *"`.  
    - Inputs: None  
    - Outputs: Triggers downstream nodes  
    - Edge Cases: Misconfigured cron can cause missed runs; no direct error handling needed here.

  - **Sticky Note (Revenue Optimization)**  
    - Type: Sticky Note  
    - Role: Provides instructions on configuring key parameters such as churn prediction thresholds, upselling triggers, pricing strategies, and retention campaigns.  
    - Inputs/Outputs: Visual aid only.

  - **Revenue Settings**  
    - Type: Set node  
    - Role: Defines reusable parameters for thresholds and discount values:
      - `churnRiskThreshold`: 0.7 (70%)  
      - `upsellThreshold`: 0.8 (80%)  
      - `retentionDiscount`: 25 (% discount)  
    - Inputs: Trigger from Schedule node  
    - Outputs: Passes parameters downstream for use in analytics  
    - Edge Cases: Parameter values must be validated externally; no built-in validation.

---

#### 1.2 Data Retrieval

- **Overview:**  
  Retrieves current customer analytics data from a Postgres database for active customers to feed into the revenue analysis.

- **Nodes Involved:**  
  - Get Customer Analytics (Postgres node)

- **Node Details:**

  - **Get Customer Analytics**  
    - Type: Postgres node  
    - Role: Executes SQL query to fetch active customers’ metrics including usage, engagement, support tickets, payment history, plan type, and monthly recurring revenue (MRR).  
    - Query:  
      ```sql
      SELECT customer_id, usage_score, engagement_score, support_tickets, payment_history, plan_type, mrr FROM customer_analytics WHERE active = true
      ```  
    - Inputs: Receives parameters from Revenue Settings (indirectly via connection)  
    - Outputs: Emits rows of customer data as JSON  
    - Edge Cases: Database connection errors, query timeouts, data inconsistencies, or missing fields could disrupt downstream processing.

---

#### 1.3 Revenue Opportunity Analysis

- **Overview:**  
  Processes each customer record to calculate churn risk, upsell potential, and assign recommended marketing actions with priority and projected revenue impact.

- **Nodes Involved:**  
  - Analyze Revenue Opportunities (Code node)

- **Node Details:**

  - **Analyze Revenue Opportunities**  
    - Type: Code node (JavaScript)  
    - Role: Applies weighted rules to customer metrics to score churn risk and upsell potential:
      - Churn risk weights: usage decline (40%), engagement decline (30%), support tickets (20%), payment issues (10%)  
      - Upsell potential thresholds based on usage and engagement scores  
    - Determines one of three recommendations per customer:
      - `retention_campaign` (high churn risk)  
      - `upsell_campaign` (high upsell potential)  
      - `engagement_campaign` (medium churn risk)  
      - Default: `monitor` (no immediate action)  
    - Calculates potential revenue impact based on action type and current MRR.  
    - Adds metadata: priority, analyzed timestamp, plan type, scores.  
    - Inputs: Customer data array, Revenue Settings parameters  
    - Outputs: Annotated customer records with actionable insights  
    - Edge Cases: Missing or malformed input data, JSON parsing errors, or referencing undefined parameters. Defensive coding needed to avoid runtime exceptions.

---

#### 1.4 Customer Segmentation & Campaign Dispatch

- **Overview:**  
  Filters customers by recommended action and sends personalized email campaigns accordingly. Customer details are enriched by fetching Stripe customer data before email dispatch.

- **Nodes Involved:**  
  - Filter High Risk Customers (If node)  
  - Get Customer Details (HTTP Request node)  
  - Send Retention Campaign (Gmail node)  
  - Filter Upsell Opportunities (If node)  
  - Get Customer Details1 (HTTP Request node)  
  - Send Upsell Campaign (Gmail node)  
  - Filter Re-engagement Needed (If node)  
  - Get Customer Details2 (HTTP Request node)  
  - Send Re-engagement Campaign (Gmail node)  
  - Sticky Note1 (Revenue Intelligence guidance)

- **Node Details:**

  - **Filter High Risk Customers**  
    - Type: If node  
    - Role: Filters customers where `recommended_action` equals `retention_campaign`.  
    - Input: Annotated customer list from analysis.  
    - Output: Routes matching customers to retention campaign branch.  
    - Edge Cases: Case sensitivity or missing field might cause false negatives.

  - **Get Customer Details**  
    - Type: HTTP Request  
    - Role: Fetches detailed customer info from Stripe API using customer_id.  
    - Method: GET  
    - URL: `https://api.stripe.com/v1/customers/{{ $json.customer_id }}`  
    - Authentication: OAuth2 or API key in Authorization header from Stripe credentials.  
    - Output: Enriched customer info including email and name.  
    - Edge Cases: API rate limits, invalid customer IDs, network errors, expired credentials.

  - **Send Retention Campaign**  
    - Type: Gmail node  
    - Role: Sends a highly personalized HTML email offering retention incentives with a 25% discount for 3 months.  
    - To: `{{ $json.email }}` from Stripe details  
    - Subject: "Special Offer - We Value Your Business 💎"  
    - Content: HTML with embedded variables such as customer name, discount percentage, and expiry date calculated at send time.  
    - Edge Cases: Invalid email, Gmail API rate limits, authentication failures.

  - **Filter Upsell Opportunities**  
    - Type: If node  
    - Role: Filters customers where `recommended_action` equals `upsell_campaign`.  
    - Output: Routes to upsell campaign branch.

  - **Get Customer Details1**  
    - Same as Get Customer Details but for upsell branch.

  - **Send Upsell Campaign**  
    - Type: Gmail node  
    - Role: Sends HTML email encouraging plan upgrade highlighting usage stats and benefits.  
    - Subject: "Ready to Unlock More Value? 🚀"  
    - Includes usage scores and plan type dynamically.  
    - Edge Cases similar to retention campaign.

  - **Filter Re-engagement Needed**  
    - Type: If node  
    - Role: Filters customers where `recommended_action` equals `engagement_campaign`.  
    - Output: Routes to re-engagement branch.

  - **Get Customer Details2**  
    - Same as prior Get Customer Details nodes but for re-engagement branch.

  - **Send Re-engagement Campaign**  
    - Type: Gmail node  
    - Role: Sends HTML email aimed at re-activating low engagement customers with personalized offers and resource links.  
    - Subject: "We Miss You! Let's Get You Back on Track 🎯"  
    - Edge Cases similar to other Gmail nodes.

  - **Sticky Note1 (Revenue Intelligence)**  
    - Provides summary of automated campaign logic for internal reference.

---

### 3. Summary Table

| Node Name                 | Node Type         | Functional Role                        | Input Node(s)                | Output Node(s)                        | Sticky Note                                                                                           |
|---------------------------|-------------------|-------------------------------------|-----------------------------|-------------------------------------|-----------------------------------------------------------------------------------------------------|
| Daily Revenue Analysis     | Schedule Trigger  | Triggers workflow daily at 6 AM      | None                        | Revenue Settings                    |                                                                                                     |
| Sticky Note               | Sticky Note       | Configuration guidance               | None                        | None                               | ## Revenue Optimization ⚙️ **Configure parameters:** - Churn prediction thresholds - Upselling triggers - Pricing strategies - Retention campaigns |
| Revenue Settings           | Set               | Defines churn and upsell parameters  | Daily Revenue Analysis       | Get Customer Analytics             |                                                                                                     |
| Get Customer Analytics     | Postgres          | Retrieves active customer analytics  | Revenue Settings             | Analyze Revenue Opportunities      |                                                                                                     |
| Analyze Revenue Opportunities | Code          | Scores churn risk, upsell potential, recommends actions | Get Customer Analytics       | Filter High Risk Customers, Filter Upsell Opportunities, Filter Re-engagement Needed |                                                                                                     |
| Filter High Risk Customers | If                | Filters customers for retention      | Analyze Revenue Opportunities | Get Customer Details               |                                                                                                     |
| Get Customer Details       | HTTP Request      | Fetches Stripe customer details      | Filter High Risk Customers   | Send Retention Campaign            |                                                                                                     |
| Send Retention Campaign    | Gmail             | Sends retention offer email          | Get Customer Details         | None                              |                                                                                                     |
| Filter Upsell Opportunities| If                | Filters customers for upsell         | Analyze Revenue Opportunities | Get Customer Details1              |                                                                                                     |
| Get Customer Details1      | HTTP Request      | Fetches Stripe customer details      | Filter Upsell Opportunities  | Send Upsell Campaign               |                                                                                                     |
| Send Upsell Campaign       | Gmail             | Sends upsell campaign email          | Get Customer Details1        | None                              |                                                                                                     |
| Filter Re-engagement Needed| If                | Filters customers for re-engagement  | Analyze Revenue Opportunities | Get Customer Details2              |                                                                                                     |
| Get Customer Details2      | HTTP Request      | Fetches Stripe customer details      | Filter Re-engagement Needed  | Send Re-engagement Campaign        |                                                                                                     |
| Send Re-engagement Campaign| Gmail             | Sends re-engagement email            | Get Customer Details2        | None                              |                                                                                                     |
| Sticky Note1              | Sticky Note       | Summary of campaign logic            | None                        | None                              | ## Revenue Intelligence 📊 **Automated campaigns:** - High churn risk: Retention offers - High usage: Upsell campaigns - Low engagement: Re-activation - Healthy: Success stories |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node named "Daily Revenue Analysis":**  
   - Set trigger type to Cron  
   - Cron expression: `0 6 * * *` (runs daily at 6 AM)

2. **Add a Sticky Note node named "Sticky Note":**  
   - Content: Guidance on revenue optimization parameters (churn thresholds, upselling triggers, etc.)  
   - Position near the trigger node for visibility

3. **Create a Set node named "Revenue Settings":**  
   - Define three number parameters:  
     - `churnRiskThreshold` = 0.7  
     - `upsellThreshold` = 0.8  
     - `retentionDiscount` = 25  
   - Connect output of "Daily Revenue Analysis" to this node

4. **Create a Postgres node named "Get Customer Analytics":**  
   - Configure Postgres credentials  
   - SQL Query:  
     ```sql
     SELECT customer_id, usage_score, engagement_score, support_tickets, payment_history, plan_type, mrr FROM customer_analytics WHERE active = true
     ```  
   - Connect output of "Revenue Settings" to this node

5. **Add a Code node named "Analyze Revenue Opportunities":**  
   - Paste the provided JavaScript code that:  
     - Iterates over customers  
     - Calculates churn risk and upsell potential  
     - Assigns recommended action and revenue impact  
   - Connect output of "Get Customer Analytics" to this node

6. **Add three If nodes for filtering by recommended action:**  
   - "Filter High Risk Customers": condition `recommended_action == 'retention_campaign'`  
   - "Filter Upsell Opportunities": condition `recommended_action == 'upsell_campaign'`  
   - "Filter Re-engagement Needed": condition `recommended_action == 'engagement_campaign'`  
   - Connect output of "Analyze Revenue Opportunities" to all three If nodes, each on a separate output branch

7. **For each filtered branch, add an HTTP Request node to get detailed customer info from Stripe:**  
   - Name them respectively: "Get Customer Details", "Get Customer Details1", "Get Customer Details2"  
   - Configure with Stripe API credentials using Bearer token (API key)  
   - Method: GET  
   - URL: `https://api.stripe.com/v1/customers/{{ $json.customer_id }}`  
   - Connect each If node’s true output to its respective HTTP Request node

8. **Add Gmail nodes to send personalized campaigns:**  
   - "Send Retention Campaign": Linked from "Get Customer Details"  
     - To: `{{ $json.email }}`  
     - Subject: "Special Offer - We Value Your Business 💎"  
     - Content: Use provided retention HTML template, dynamically injecting customer name and retention discount parameter  
   - "Send Upsell Campaign": Linked from "Get Customer Details1"  
     - To: `{{ $json.email }}`  
     - Subject: "Ready to Unlock More Value? 🚀"  
     - Content: Use upsell HTML template with usage stats and plan type injected  
   - "Send Re-engagement Campaign": Linked from "Get Customer Details2"  
     - To: `{{ $json.email }}`  
     - Subject: "We Miss You! Let's Get You Back on Track 🎯"  
     - Content: Use re-engagement HTML template with personalized offers  
   - Configure Gmail credentials (OAuth2 or app password)  
   - Connect each HTTP Request node to corresponding Gmail node

9. **Add a Sticky Note named "Sticky Note1":**  
   - Content: Summary of automated campaign logic for internal documentation

10. **Final connections:**  
    - "Daily Revenue Analysis" → "Revenue Settings" → "Get Customer Analytics" → "Analyze Revenue Opportunities"  
    - "Analyze Revenue Opportunities" → three If nodes for filtering  
    - Each If node (true branch) → respective "Get Customer Details" HTTP Request node → respective Gmail node

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow automates revenue optimization by combining predictive analytics with targeted email campaigns leveraging Stripe and Postgres data. | Core project purpose                                                                                 |
| Sticky notes provide parameter configuration and campaign logic summaries for maintainers and operators.                  | Internal visual documentation                                                                       |
| Gmail nodes use HTML templates with inline CSS for well-formatted, branded emails.                                          | Email content best practice                                                                          |
| Stripe API calls require valid API key credentials with appropriate permissions for customer data access.                  | Stripe API documentation: https://stripe.com/docs/api/customers/retrieve                            |
| Postgres node requires a properly configured database with customer analytics data structured as per SQL query.            | Database schema must include fields: customer_id, usage_score, engagement_score, support_tickets, payment_history, plan_type, mrr, active flag |
| Email templates include dynamic expressions referencing node outputs and parameters, ensure expression syntax matches n8n version used. | n8n expression syntax documentation: https://docs.n8n.io/nodes/expressions/                         |

---

This completes the comprehensive reference documentation of the Subscription Revenue Optimizer workflow, enabling advanced users or AI agents to fully understand, reproduce, and maintain the automation.