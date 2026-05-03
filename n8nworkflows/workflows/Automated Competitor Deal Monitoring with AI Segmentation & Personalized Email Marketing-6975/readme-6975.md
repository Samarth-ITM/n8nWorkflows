Automated Competitor Deal Monitoring with AI Segmentation & Personalized Email Marketing

https://n8nworkflows.xyz/workflows/automated-competitor-deal-monitoring-with-ai-segmentation---personalized-email-marketing-6975


# Automated Competitor Deal Monitoring with AI Segmentation & Personalized Email Marketing

### 1. Workflow Overview

This workflow automates the monitoring of competitor deals and leverages AI-driven segmentation to generate personalized email marketing campaigns. It is designed for retail or e-commerce businesses seeking to maintain competitive intelligence, engage customers with targeted offers, and produce actionable analytics reports.  
The workflow runs daily at 8 AM and includes the following logical blocks:

- **1.1 Trigger & Data Collection**: Scheduled initiation and retrieval of competitor deals data from specified sources.  
- **1.2 Deal Analysis & Customer Segmentation**: Processing of deal data to assess market conditions and segment customers based on value and behavior.  
- **1.3 Offer Generation & Campaign Creation**: Crafting personalized discount offers tailored to each customer segment and generating corresponding email campaigns.  
- **1.4 Analytics & Reporting**: Compiling comprehensive reports summarizing deal insights, campaign performance, and strategic recommendations.  
- **1.5 Email Dispatch Setup**: Configuration for sending daily reports via email (disabled by default for safety).

---

### 2. Block-by-Block Analysis

---

#### 1.1 Trigger & Data Collection

**Overview:**  
This block initiates the workflow daily at 8 AM and fetches deal data from an external deals website, serving as the foundation for subsequent analysis.

**Nodes Involved:**  
- Daily Trigger  
- Fetch Deal Data  
- Data Collection Setup (Sticky Note)

**Node Details:**  

- **Daily Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow every day at 8:00 AM using a cron expression (`0 8 * * *`).  
  - Inputs: None  
  - Outputs: Triggers "Fetch Deal Data" node  
  - Edge Cases: Cron misconfiguration, system downtime at trigger time.

- **Fetch Deal Data**  
  - Type: HTTP Request  
  - Role: Retrieves current competitor deal listings from `https://www.retailmenot.com/coupons/electronics`.  
  - Configuration: 30-second timeout, GET request by default, no authentication included but noted as potentially needed.  
  - Inputs: Triggered by "Daily Trigger"  
  - Outputs: Sends fetched raw deal data to "Analyze Deals" node  
  - Edge Cases: HTTP errors, site rate limiting, changed URL structure, authentication failures, request timeouts.

- **Data Collection Setup** (Sticky Note)  
  - Provides setup instructions and best practices: replace URL as needed, add authentication, handle rate limiting, and use multiple sources for robustness.

---

#### 1.2 Deal Analysis & Customer Segmentation

**Overview:**  
Analyzes the fetched deal data to extract competitive intelligence and segments customers into actionable groups to personalize marketing efforts.

**Nodes Involved:**  
- Analyze Deals  
- Segment Customers  
- Intelligence Processing (Sticky Note)

**Node Details:**  

- **Analyze Deals**  
  - Type: Code (JavaScript)  
  - Role: Parses and analyzes competitor deals to summarize market metrics and classify threat levels.  
  - Configuration: Uses simulated mock data (due to scraping challenges) with predefined deals and categories.  
  - Key Logic: Computes total deals, average discount, top categories, threat level (High if any discount >35%), urgent deals.  
  - Inputs: Data from "Fetch Deal Data"  
  - Outputs: Analysis JSON to "Segment Customers" and "Generate Analytics Report"  
  - Edge Cases: Real data parsing failure, missing fields, logic misinterpretation.

- **Segment Customers**  
  - Type: Code (JavaScript)  
  - Role: Simulates customer database segmentation based on predefined segments: VIP, bargain hunter, at-risk, new prospect.  
  - Configuration: Hardcoded customer list with segments and values.  
  - Inputs: Analysis result from "Analyze Deals"  
  - Outputs: Segmented customer JSON for offer generation and analytics  
  - Edge Cases: Empty customer lists, segment misclassification.

- **Intelligence Processing** (Sticky Note)  
  - Summarizes analytical focus areas: discount rates, threat levels, category performance, customer segments.

---

#### 1.3 Offer Generation & Campaign Creation

**Overview:**  
Generates personalized discount offers for each customer segment and constructs email campaigns with dynamic content and prioritization.

**Nodes Involved:**  
- Generate Personalized Offers  
- Create Email Campaigns  
- Campaign Generation (Sticky Note)

**Node Details:**  

- **Generate Personalized Offers**  
  - Type: Code (JavaScript)  
  - Role: Creates segmented campaigns with tailored discount levels based on market threat and customer value.  
  - Configuration: Defines discount and priority per customer segment; generates unique discount codes per customer.  
  - Inputs: "Analyze Deals" and "Segment Customers" outputs (note: two inputs combined)  
  - Outputs: Detailed offer batches for each segment  
  - Edge Cases: Empty segments, discount calculation errors, random discount code collisions.

- **Create Email Campaigns**  
  - Type: Code (JavaScript)  
  - Role: Converts offers into individual email campaign objects with personalized subject lines, templates, and priorities.  
  - Configuration: Uses segment-specific templates and subject lines; constructs campaign IDs dynamically.  
  - Inputs: Offers from "Generate Personalized Offers"  
  - Outputs: Email campaign list for sending or reporting  
  - Edge Cases: Missing email addresses, template mapping errors.

- **Campaign Generation** (Sticky Note)  
  - Describes campaign creation process: dynamic subjects, unique discount codes, templates, and priority scheduling.

---

#### 1.4 Analytics & Reporting

**Overview:**  
Compiles an executive summary report with key insights and action recommendations based on deals, customer segments, offers, and campaign data.

**Nodes Involved:**  
- Generate Analytics Report  
- Send Daily Report (Disabled)  
- Analytics & Reporting (Sticky Note)

**Node Details:**  

- **Generate Analytics Report**  
  - Type: Code (JavaScript)  
  - Role: Aggregates all previous data into a structured report suitable for management review.  
  - Configuration: Summarizes deal metrics, customer segments, campaign statistics, and generates immediate action items based on threat level.  
  - Inputs: Outputs from "Analyze Deals", "Segment Customers", "Generate Personalized Offers", and "Create Email Campaigns"  
  - Outputs: Final analytics report JSON to "Send Daily Report"  
  - Edge Cases: Missing inputs, empty datasets, date formatting issues.

- **Send Daily Report**  
  - Type: HTTP Request  
  - Role: Sends the daily analytics report email via SendGrid API.  
  - Configuration: POST to SendGrid's mail send endpoint; uses generic HTTP header authentication; includes dynamic HTML content with key stats.  
  - Inputs: Analytics report from "Generate Analytics Report"  
  - Outputs: HTTP response from SendGrid  
  - Status: Disabled by default for safety; requires credential setup and email recipient update.  
  - Edge Cases: Authentication failures, SendGrid API limits, email formatting errors.

- **Analytics & Reporting** (Sticky Note)  
  - Highlights the contents and uses of daily reports, ideal for strategic meetings.

---

#### 1.5 Email Dispatch Setup

**Overview:**  
Instructions and precautions for configuring and enabling the email sending functionality.

**Nodes Involved:**  
- Email Configuration (Sticky Note)

**Node Details:**  

- **Email Configuration** (Sticky Note)  
  - Details required steps to activate email sending: enable node, set SendGrid credentials, update recipient email, test delivery.  
  - Warns that the email node is disabled by default for safety.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                               | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                             |
|-------------------------|---------------------|-----------------------------------------------|-------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow Overview       | Sticky Note         | Provides high-level workflow summary          | None                          | None                          | 🎯 **DEAL INTELLIGENCE & MARKETING AUTOMATION**... Runs every day at 8 AM, sends daily reports to management |
| Daily Trigger           | Schedule Trigger    | Triggers workflow daily at 8 AM                | None                          | Fetch Deal Data                |                                                                                                       |
| Data Collection Setup   | Sticky Note         | Setup instructions for data fetching           | None                          | None                          | 🔍 **DATA COLLECTION**... Replace URL, add authentication, consider rate limiting                       |
| Fetch Deal Data         | HTTP Request        | Retrieves competitor deal data                  | Daily Trigger                 | Analyze Deals                 |                                                                                                       |
| Analyze Deals           | Code                | Analyzes deals for market intelligence          | Fetch Deal Data               | Segment Customers, Generate Analytics Report |                                                                                                       |
| Segment Customers       | Code                | Segments customers based on behavior & value   | Analyze Deals                 | Generate Personalized Offers, Generate Analytics Report |                                                                                                       |
| Intelligence Processing | Sticky Note         | Explains deal analysis and segmentation logic  | None                          | None                          | 🧠 **INTELLIGENCE PROCESSING**... Deals analyzed for discount rates, threat levels, segmentation        |
| Generate Personalized Offers | Code           | Generates personalized offers per customer segment | Analyze Deals, Segment Customers | Create Email Campaigns       |                                                                                                       |
| Create Email Campaigns  | Code                | Creates email campaigns with personalized content | Generate Personalized Offers  | Generate Analytics Report     |                                                                                                       |
| Campaign Generation     | Sticky Note         | Describes campaign generation logic             | None                          | None                          | 📧 **CAMPAIGN GENERATION**... Personalized offers, dynamic subjects, unique codes, priority scheduling  |
| Generate Analytics Report | Code              | Compiles executive summary and insights         | Analyze Deals, Segment Customers, Generate Personalized Offers, Create Email Campaigns | Send Daily Report |                                                                                                       |
| Send Daily Report       | HTTP Request        | Sends daily report email via SendGrid            | Generate Analytics Report     | None                          | ⚙️ **EMAIL SETUP REQUIRED**... Disabled node, requires credentials and recipient update                  |
| Analytics & Reporting   | Sticky Note         | Describes report contents and strategic use      | None                          | None                          | 📊 **ANALYTICS & REPORTING**... Includes summary, insights, segment performance, revenue projections    |
| Email Configuration     | Sticky Note         | Instructions for enabling and configuring email | None                          | None                          | ⚙️ **EMAIL SETUP REQUIRED**... Enable node, setup SendGrid, update recipient, test delivery             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Daily Trigger" node**  
   - Type: Schedule Trigger  
   - Setup: Cron expression `0 8 * * *` to run at 8 AM daily.

2. **Create "Fetch Deal Data" node**  
   - Type: HTTP Request  
   - Connect input from "Daily Trigger"  
   - Configure URL to target deal website, e.g., `https://www.retailmenot.com/coupons/electronics`  
   - Set timeout to 30000 ms (30 seconds)  
   - Adjust authentication if required by the source site.

3. **Create "Analyze Deals" node**  
   - Type: Code  
   - Connect input from "Fetch Deal Data"  
   - Paste provided JavaScript to analyze deals, using mock data fallback if scraping fails.

4. **Create "Segment Customers" node**  
   - Type: Code  
   - Connect input from "Analyze Deals"  
   - Implement customer segmentation logic with predefined customer list and segments.

5. **Create "Generate Personalized Offers" node**  
   - Type: Code  
   - Connect two inputs: first from "Analyze Deals" and second from "Segment Customers" (ensure both inputs are attached properly to get combined inputs)  
   - Paste JavaScript that calculates discount levels and prepares campaign batches per segment.

6. **Create "Create Email Campaigns" node**  
   - Type: Code  
   - Connect input from "Generate Personalized Offers"  
   - Implement logic to create personalized email content per customer, including subject lines and templates.

7. **Create "Generate Analytics Report" node**  
   - Type: Code  
   - Connect four inputs: from "Analyze Deals", "Segment Customers", "Generate Personalized Offers", and "Create Email Campaigns"  
   - Use JavaScript to aggregate all inputs into a comprehensive report with executive summary, key insights, and immediate actions.

8. **Create "Send Daily Report" node** (Optional, initially disabled)  
   - Type: HTTP Request  
   - Connect input from "Generate Analytics Report"  
   - Configure as POST request to `https://api.sendgrid.com/v3/mail/send`  
   - Set HTTP header authentication with SendGrid API credentials  
   - Prepare JSON body with dynamic HTML content referencing report fields: threat level, campaigns created, projected revenue  
   - Disable node by default until credentials and recipient emails are set.

9. **Add Sticky Note nodes** for documentation at appropriate workflow positions:  
   - Workflow Overview  
   - Data Collection Setup  
   - Intelligence Processing  
   - Campaign Generation  
   - Analytics & Reporting  
   - Email Configuration

10. **Verify connections:**  
    - Daily Trigger → Fetch Deal Data → Analyze Deals → Segment Customers → Generate Personalized Offers → Create Email Campaigns → Generate Analytics Report → Send Daily Report (disabled)  
    - Analyze Deals → Generate Analytics Report (parallel input)  
    - Segment Customers → Generate Analytics Report (parallel input)

11. **Configure credentials:**  
    - For HTTP Request nodes requiring authentication (e.g., SendGrid), create and assign appropriate OAuth2 or API Key credentials in n8n.

12. **Test the workflow:**  
    - Run manually to verify data flows, output correctness, and error handling.  
    - Enable "Send Daily Report" only after confirming email delivery setup.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                                          |
|--------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Workflow is tagged under "Amazon Intelligence" indicating focus on e-commerce and retail competitor data. | Tag metadata in workflow                                                                                                 |
| Consider using multiple deal data sources for better deal coverage and reliability.                     | Data Collection Setup Sticky Note                                                                                        |
| Email node is disabled by default for safety; requires manual enablement and credential setup.          | Email Configuration Sticky Note                                                                                          |
| Daily report HTML content is minimal; for richer formatting customize the email body in "Send Daily Report" node. | Send Daily Report node configuration                                                                                     |
| Campaigns use dynamic discount codes generated randomly; consider collision risk in large datasets.     | Generate Personalized Offers node logic                                                                                  |
| Market threat level triggers immediate actions like aggressive pricing and increased email frequency.  | Generate Analytics Report node logic                                                                                      |

---

This completes the detailed reference documentation for the "Automated Competitor Deal Monitoring with AI Segmentation & Personalized Email Marketing" workflow. It enables users and AI agents to understand, reproduce, and maintain the workflow efficiently.