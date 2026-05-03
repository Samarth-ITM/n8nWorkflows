Shopify Multi-Module Automation with GPT-4o, Langchain Agents & Integrations

https://n8nworkflows.xyz/workflows/shopify-multi-module-automation-with-gpt-4o--langchain-agents---integrations-4455


# Shopify Multi-Module Automation with GPT-4o, Langchain Agents & Integrations

### 1. Workflow Overview

This workflow titled **"Shopify AI Agent Suite: Full E-commerce Automation (Support, Sales, Ops)"** is a comprehensive automation pipeline for Shopify e-commerce stores. It integrates multiple AI-powered Langchain agents (using GPT-4o and OpenAI chat models) with Shopify APIs, Google Sheets, Slack, Twilio, Notion, and email services to automate customer support, product inquiries, abandoned cart recovery, inventory stock alerts, post-purchase review requests, and marketing email campaigns.

The workflow is logically divided into six main functional blocks:

- **1.1 Customer Support AI (FAQ + Order Help)**: Handles incoming customer messages, fetches FAQ data, performs order status lookups, and uses AI to generate responses or escalate to human agents.
  
- **1.2 Product Inquiry & Selection Assistance**: Manages user product inquiries, searches product inventory, applies filters and sorting, refines product selections with AI, and responds with interactive product carousels.
  
- **1.3 Abandoned Cart Recovery**: Detects abandoned carts, waits a period, fetches customer/cart data, uses AI to analyze and generate recovery offers (discounts), and sends recovery emails and SMS reminders.
  
- **1.4 Inventory Monitoring & Low Stock Alerts**: Runs hourly inventory checks, filters low stock items, generates AI-based stock reports, and sends alerts to Slack and email, also supporting manual restock notifications.
  
- **1.5 Post-Purchase Review Requests & Monitoring**: After order delivery, waits 3 days, fetches customer data to send AI-generated review request emails; listens for incoming reviews, flags negative ones, notifies support, and logs reviews.
  
- **1.6 Email Campaign Automation**: Scheduled campaign execution fetching target audiences, segmenting them, generating AI-created campaign content, sending emails, tracking performance, and adjusting campaigns dynamically.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Customer Support AI (FAQ + Order Help)

- **Overview:**  
Receives incoming customer support messages via webhook, retrieves FAQ data from Google Sheets, looks up order information, preprocesses input, and uses an AI agent to generate responses. If the AI determines escalation is needed, it notifies a human agent via Slack; otherwise, it sends an AI response back to the customer and logs the interaction in Notion.

- **Nodes Involved:**  
`Incoming Message`, `Get FAQs Data`, `Lookup Order API`, `Preprocess Input for Agent`, `AI Agent`, `OpenAI Chat Model for Agent`, `Check if Escalation Needed`, `Notify Human Agent`, `Send AI Response to Customer`, `Log Interaction`, `Store to Notion`, plus Langchain tool nodes `SearchFAQs` and `LookupOrderStatus`.

- **Node Details:**  
  - **Incoming Message**  
    - Type: Webhook  
    - Role: Entry point for incoming customer messages (e.g., chat or support form)  
    - Config: Uses a unique webhook URL for external systems to post messages  
    - Outputs: To `Get FAQs Data` and `Preprocess Input for Agent`  
    - Potential failures: webhook not triggered if URL misconfigured; payload format errors

  - **Get FAQs Data**  
    - Type: Google Sheets  
    - Role: Retrieves FAQ content for AI reference  
    - Config: Reads from a predefined FAQ sheet to provide structured knowledge  
    - Output: To `Preprocess Input for Agent`  
    - Failures: API quota limits, sheet access permissions

  - **Lookup Order API**  
    - Type: HTTP Request  
    - Role: Queries Shopify order API to fetch order status/details  
    - Config: Uses authenticated Shopify API credentials  
    - Output: To `Preprocess Input for Agent`  
    - Failures: Auth errors, API rate limits, invalid order IDs

  - **Preprocess Input for Agent**  
    - Type: Function  
    - Role: Normalizes and structures input data for AI consumption  
    - Uses expressions to combine FAQ and order data with user message  
    - Outputs: To `AI Agent`

  - **AI Agent**  
    - Type: Langchain Agent (v1.9)  
    - Role: Core AI processing of support queries using GPT-4o-powered OpenAI chat model  
    - Uses linked OpenAI Chat Model node (`OpenAI Chat Model for Agent`)  
    - Calls Langchain tools: `SearchFAQs` and `LookupOrderStatus` to augment responses  
    - Output: To `Check if Escalation Needed`

  - **Check if Escalation Needed**  
    - Type: IF Node  
    - Role: Determines if the AI response requires human intervention  
    - True branch: `Notify Human Agent`  
    - False branch: `Send AI Response to Customer`

  - **Notify Human Agent**  
    - Type: Slack  
    - Role: Sends alert to support team channel with conversation context  
    - Output: To `Log Interaction`

  - **Send AI Response to Customer**  
    - Type: HTTP Request  
    - Role: Sends AI-generated reply back to customer via external messaging API  
    - Output: To `Log Interaction`

  - **Log Interaction**  
    - Type: Function  
    - Role: Prepares interaction data for record keeping  
    - Output: To `Store to Notion`

  - **Store to Notion**  
    - Type: Notion  
    - Role: Logs support interaction in a Notion database for traceability  
    - Failures: Auth errors, API limits

  - **Langchain Tool Nodes (SearchFAQs, LookupOrderStatus)**  
    - Type: ToolCode  
    - Role: Provide contextual AI tools for the agent to query FAQs and order status  
    - Essential for modular AI reasoning

---

#### 2.2 Product Inquiry & Selection Assistance

- **Overview:**  
Handles product inquiry webhook triggers, summarizes user requests, searches Shopify product catalog, filters results (stock availability, price), sorts by rating, and refines selections through an AI agent. The final product list is formatted into a carousel and sent back to the user.

- **Nodes Involved:**  
`Product Inquiry Webhook`, `Summarize Request & Get Filters`, `Search Products API`, `Filter Out of Stock`, `Sort by Rating`, `Filter by Price`, `AI Agent1 (Product Selection Refinement)`, `OpenAI Chat Model for Agent1`, `Build Product Carousel`, `Respond to User with Carousel`, `Log Product Suggestion`, plus Langchain tool node `RefineProductSelection`.

- **Node Details:**  
  - **Product Inquiry Webhook**  
    - Type: Webhook  
    - Role: Entry point for product inquiry requests  
    - Output: To `Summarize Request & Get Filters`

  - **Summarize Request & Get Filters**  
    - Type: Function  
    - Role: Analyzes and extracts filter criteria (e.g., price range, preferences) from user input  
    - Output: To `Search Products API` and `AI Agent1`

  - **Search Products API**  
    - Type: HTTP Request  
    - Role: Queries Shopify product API with filters  
    - Output: To `Filter Out of Stock`

  - **Filter Out of Stock**  
    - Type: IF Node  
    - Role: Removes products with zero stock  
    - True: Excluded; False: Passes on to `Sort by Rating`

  - **Sort by Rating**  
    - Type: Function  
    - Role: Sorts available products based on customer ratings  
    - Output: To `Filter by Price`

  - **Filter by Price**  
    - Type: IF Node  
    - Role: Filters products within user-specified price limits  
    - Output: To `AI Agent1 (Product Selection Refinement)`

  - **AI Agent1 (Product Selection Refinement)**  
    - Type: Langchain Agent  
    - Role: Refines product list based on complex user preferences using AI  
    - Uses `OpenAI Chat Model for Agent1` and tool `RefineProductSelection`  
    - Output: To `Build Product Carousel`

  - **Build Product Carousel**  
    - Type: Function  
    - Role: Formats product data into a carousel UI format for messaging platforms  
    - Output: To `Respond to User with Carousel`

  - **Respond to User with Carousel**  
    - Type: HTTP Request  
    - Role: Sends product carousel back to user through messaging API  
    - Output: To `Log Product Suggestion`

  - **Log Product Suggestion**  
    - Type: Function  
    - Role: Logs product suggestion details for analytics or auditing

---

#### 2.3 Abandoned Cart Recovery

- **Overview:**  
Triggered by Shopify abandoned cart webhook, waits a configured period, fetches customer and cart details, analyzes data with AI to decide on discounts, sends recovery emails and SMS reminders, then waits and checks if recovery succeeded, escalating or terminating workflow accordingly.

- **Nodes Involved:**  
`Detect Abandoned Cart`, `Wait Period`, `Fetch Customer Info`, `Analyze Cart & Customer Data`, `AI Agent2 (Cart Recovery)`, `OpenAI Chat Model for Agent2`, `Create Discount (if applicable)`, `Send Recovery Email`, `Send SMS Reminder`, `Follow-Up Wait`, `Check if Still Not Recovered`, plus Langchain tool `DetermineDiscount`.

- **Node Details:**  
  - **Detect Abandoned Cart**  
    - Type: Shopify Trigger  
    - Role: Detects when a cart is abandoned  
    - Output: To `Wait Period`

  - **Wait Period**  
    - Type: Wait  
    - Role: Delays processing to allow customer time before recovery attempts  
    - Output: To `Fetch Customer Info`

  - **Fetch Customer Info**  
    - Type: HTTP Request  
    - Role: Retrieves customer and cart data from Shopify  
    - Output: To `Analyze Cart & Customer Data`

  - **Analyze Cart & Customer Data**  
    - Type: Function  
    - Role: Prepares data for AI analysis  
    - Output: To `AI Agent2 (Cart Recovery)`

  - **AI Agent2 (Cart Recovery)**  
    - Type: Langchain Agent  
    - Role: AI generates personalized recovery strategy and discount offers  
    - Uses `OpenAI Chat Model for Agent2` and tool `DetermineDiscount`  
    - Output: To `Create Discount (if applicable)`

  - **Create Discount (if applicable)**  
    - Type: Shopify node  
    - Role: Creates a discount code if AI recommends it  
    - Output: To `Send Recovery Email`

  - **Send Recovery Email**  
    - Type: HTTP Request  
    - Role: Sends recovery email to customer with discount and cart info  
    - Output: To `Send SMS Reminder`

  - **Send SMS Reminder**  
    - Type: Twilio  
    - Role: Sends SMS reminder to customer about abandoned cart  
    - Output: To `Follow-Up Wait`

  - **Follow-Up Wait**  
    - Type: Wait  
    - Role: Waits additional time for customer response  
    - Output: To `Check if Still Not Recovered`

  - **Check if Still Not Recovered**  
    - Type: IF Node  
    - Role: Checks if cart was recovered (order placed)  
    - True: Ends or logs success  
    - False: Could escalate or retry (not detailed)

---

#### 2.4 Inventory Monitoring & Low Stock Alerts

- **Overview:**  
An hourly cron triggers inventory fetch from Shopify, filters low stock items (<10 units), uses AI to generate a formatted low stock report, sends alerts to Slack and email, and exports logs to Google Sheets. Also supports manual restock alerts via webhook and SMS.

- **Nodes Involved:**  
`Hourly Check`, `Fetch Inventory`, `Filter Low Stock (<10)`, `AI Agent3 (Low Stock Reporting)`, `OpenAI Chat Model for Agent3`, `Notify Slack (Low Stock)`, `Generate Email Report (Low Stock)`, `Export to Sheets (Low Stock Log)`, `Trigger Restock Webhook`, `Send SMS Alert (Restock)`, plus Langchain tool `FormatLowStockReport`.

- **Node Details:**  
  - **Hourly Check**  
    - Type: Cron  
    - Role: Scheduled hourly trigger  
    - Output: To `Fetch Inventory`

  - **Fetch Inventory**  
    - Type: Shopify  
    - Role: Retrieves current inventory data  
    - Output: To `Filter Low Stock (<10)`

  - **Filter Low Stock (<10)**  
    - Type: IF Node  
    - Role: Filters products with stock less than 10  
    - True: To `AI Agent3 (Low Stock Reporting)`

  - **AI Agent3 (Low Stock Reporting)**  
    - Type: Langchain Agent  
    - Role: Uses AI to format low stock data into human-readable report  
    - Uses `OpenAI Chat Model for Agent3` and tool `FormatLowStockReport`  
    - Output: To `Notify Slack (Low Stock)`

  - **Notify Slack (Low Stock)**  
    - Type: Slack  
    - Role: Sends alert message with low stock report to Slack channel  
    - Output: To `Generate Email Report (Low Stock)`

  - **Generate Email Report (Low Stock)**  
    - Type: Email Send  
    - Role: Emails low stock report to stakeholders  
    - Output: To `Export to Sheets (Low Stock Log)`

  - **Export to Sheets (Low Stock Log)**  
    - Type: Google Sheets  
    - Role: Logs low stock data for historical tracking

  - **Trigger Restock Webhook**  
    - Type: Webhook  
    - Role: Manual trigger for restock alert workflow  
    - Output: To `Send SMS Alert (Restock)`

  - **Send SMS Alert (Restock)**  
    - Type: Twilio  
    - Role: Sends SMS notification for restock alerts  
    - Output: To `Basic LLM Chain` (for follow-up email)

  - **Basic LLM Chain** and **Send restock Request Email1**  
    - Supporting nodes for composing and sending restock request emails

---

#### 2.5 Post-Purchase Review Requests & Monitoring

- **Overview:**  
Triggered when an order is delivered, waits 3 days, fetches customer data, and uses AI to generate personalized review request emails. Also listens for incoming reviews via webhook, flags negative reviews, notifies support via Slack, and logs all reviews in Google Sheets.

- **Nodes Involved:**  
`Order Delivered Trigger`, `Wait 3 Days`, `Fetch Customer Data (for Review)`, `AI Agent4 (Review Request Email)`, `OpenAI Chat Model for Agent4`, `Send Review Request Email`, `Listen for Review Webhook`, `Flag Negative Review`, `Notify Support Team (Negative Review)`, `Add Review to Database`, plus Langchain tool `DraftReviewRequestEmail`.

- **Node Details:**  
  - **Order Delivered Trigger**  
    - Type: Shopify Trigger  
    - Role: Activated when order delivery is confirmed  
    - Output: To `Wait 3 Days`

  - **Wait 3 Days**  
    - Type: Wait  
    - Role: Delay to allow customer experience before requesting review  
    - Output: To `Fetch Customer Data (for Review)`

  - **Fetch Customer Data (for Review)**  
    - Type: HTTP Request  
    - Role: Gets customer info for personalized email  
    - Output: To `AI Agent4 (Review Request Email)`

  - **AI Agent4 (Review Request Email)**  
    - Type: Langchain Agent  
    - Role: Generates customized review request email content  
    - Uses `OpenAI Chat Model for Agent4` and tool `DraftReviewRequestEmail`  
    - Output: To `Send Review Request Email`

  - **Send Review Request Email**  
    - Type: Email Send  
    - Role: Sends email to customer requesting review

  - **Listen for Review Webhook**  
    - Type: Webhook  
    - Role: Listens for inbound review notifications or submissions  
    - Output: To `OpenAI` for sentiment analysis and `Flag Negative Review`

  - **Flag Negative Review**  
    - Type: IF Node  
    - Role: Determines if review is negative  
    - True: To `Notify Support Team (Negative Review)`  
    - False: To `Add Review to Database`

  - **Notify Support Team (Negative Review)**  
    - Type: Slack  
    - Role: Alerts support team for follow-up on negative reviews  
    - Output: To `Add Review to Database`

  - **Add Review to Database**  
    - Type: Google Sheets  
    - Role: Saves review details for record-keeping and analytics

---

#### 2.6 Email Campaign Automation

- **Overview:**  
Runs scheduled marketing campaigns by fetching target audience, segmenting them, generating AI-crafted campaign content, sending emails, tracking campaign metrics, and dynamically adjusting campaigns via AI agents based on performance data.

- **Nodes Involved:**  
`Campaign Schedule (e.g., Daily at 9 AM)`, `Fetch Target Audience API`, `Segment Audience Logic`, `AI Agent5 (Campaign Content Generation)`, `OpenAI Chat Model for Agent5`, `GenerateCampaignEmailVariant`, `Prepare Campaign Email`, `Send Campaign Email`, `Track Campaign Metrics API`, `AI Agent6 (Campaign Adjustment)`, `OpenAI Chat Model for Agent6`, `SuggestCampaignAdjustments`, `Adjust Campaign (Implement Suggestions)`, `Log Campaign Outcome & Adjustments`, plus Langchain tools `GenerateCampaignEmailVariant`, `AnalyzeCampaignPerformance`, `SuggestCampaignAdjustments`.

- **Node Details:**  
  - **Campaign Schedule**  
    - Type: Cron  
    - Role: Triggers campaigns on schedule (e.g., daily at 9 AM)  
    - Output: To `Fetch Target Audience API`

  - **Fetch Target Audience API**  
    - Type: HTTP Request  
    - Role: Retrieves customer segments for campaign targeting  
    - Output: To `Segment Audience Logic`

  - **Segment Audience Logic**  
    - Type: Function  
    - Role: Applies segmentation logic to audience data  
    - Output: To `AI Agent5 (Campaign Content Generation)`

  - **AI Agent5 (Campaign Content Generation)**  
    - Type: Langchain Agent  
    - Role: Generates email content variants using AI  
    - Uses `OpenAI Chat Model for Agent5` and tool `GenerateCampaignEmailVariant`  
    - Output: To `Prepare Campaign Email`

  - **Prepare Campaign Email**  
    - Type: Function  
    - Role: Formats email content for sending  
    - Output: To `Send Campaign Email`

  - **Send Campaign Email**  
    - Type: Email Send  
    - Role: Sends campaign email to segmented audience  
    - Output: To `Track Campaign Metrics API`

  - **Track Campaign Metrics API**  
    - Type: HTTP Request  
    - Role: Fetches campaign performance data (opens, clicks, conversions)  
    - Output: To `AI Agent6 (Campaign Adjustment)`

  - **AI Agent6 (Campaign Adjustment)**  
    - Type: Langchain Agent  
    - Role: Analyzes campaign data and suggests optimizations  
    - Uses `OpenAI Chat Model for Agent6` and tools `AnalyzeCampaignPerformance` and `SuggestCampaignAdjustments`  
    - Output: To `Adjust Campaign (Implement Suggestions)`

  - **Adjust Campaign (Implement Suggestions)**  
    - Type: Function  
    - Role: Applies AI-suggested campaign changes  
    - Output: To `Log Campaign Outcome & Adjustments`

  - **Log Campaign Outcome & Adjustments**  
    - Type: Google Sheets  
    - Role: Records campaign results and adjustment history for review

---

### 3. Summary Table

| Node Name                         | Node Type                       | Functional Role                             | Input Node(s)                      | Output Node(s)                            | Sticky Note                                   |
|----------------------------------|--------------------------------|--------------------------------------------|----------------------------------|-------------------------------------------|-----------------------------------------------|
| Incoming Message                 | Webhook                        | Receives incoming customer support messages| None                             | Preprocess Input for Agent, Get FAQs Data|                                               |
| Get FAQs Data                   | Google Sheets                  | Fetches FAQ data for AI context             | Incoming Message                 | Preprocess Input for Agent                 |                                               |
| Lookup Order API                | HTTP Request                  | Retrieves order status/details               | Incoming Message                 | Preprocess Input for Agent                 |                                               |
| Preprocess Input for Agent      | Function                      | Normalizes input for AI agent                | Incoming Message, Get FAQs Data, Lookup Order API | AI Agent                                |                                               |
| AI Agent                       | Langchain Agent               | AI processing of support queries             | Preprocess Input for Agent       | Check if Escalation Needed                 |                                               |
| OpenAI Chat Model for Agent    | Langchain LM Chat Model       | Provides GPT-4o chat model for AI Agent     | AI Agent                       | AI Agent                                 |                                               |
| Check if Escalation Needed     | IF Node                       | Determines if human escalation needed       | AI Agent                       | Notify Human Agent, Send AI Response to Customer |                                               |
| Notify Human Agent             | Slack                        | Sends alert to human support team            | Check if Escalation Needed       | Log Interaction                           |                                               |
| Send AI Response to Customer  | HTTP Request                 | Sends AI reply to customer                    | Check if Escalation Needed       | Log Interaction                           |                                               |
| Log Interaction               | Function                     | Logs interaction data                         | Notify Human Agent, Send AI Response to Customer | Store to Notion                        |                                               |
| Store to Notion               | Notion                       | Stores interaction logs                       | Log Interaction                 | None                                      |                                               |
| SearchFAQs                   | Langchain ToolCode            | Provides FAQ search tool for AI               | AI Agent                       | AI Agent                                 |                                               |
| LookupOrderStatus            | Langchain ToolCode            | Provides order lookup tool for AI             | AI Agent                       | AI Agent                                 |                                               |
| Product Inquiry Webhook       | Webhook                      | Receives product inquiry requests             | None                             | Summarize Request & Get Filters           |                                               |
| Summarize Request & Get Filters| Function                    | Extracts filters from user input              | Product Inquiry Webhook         | Search Products API, AI Agent1             |                                               |
| Search Products API           | HTTP Request                 | Queries Shopify for products                   | Summarize Request & Get Filters | Filter Out of Stock                        |                                               |
| Filter Out of Stock           | IF Node                      | Filters out-of-stock products                  | Search Products API             | Sort by Rating                            |                                               |
| Sort by Rating               | Function                      | Sorts products by rating                       | Filter Out of Stock             | Filter by Price                           |                                               |
| Filter by Price              | IF Node                      | Filters products by price                       | Sort by Rating                 | AI Agent1 (Product Selection Refinement) |                                               |
| AI Agent1 (Product Selection Refinement)| Langchain Agent       | Refines product list with AI                    | Filter by Price                | Build Product Carousel                    |                                               |
| OpenAI Chat Model for Agent1 | Langchain LM Chat Model       | GPT-4o chat model for product selection AI    | AI Agent1                     | AI Agent1                                |                                               |
| Build Product Carousel       | Function                     | Formats product list into carousel              | AI Agent1                     | Respond to User with Carousel             |                                               |
| Respond to User with Carousel| HTTP Request                 | Sends product carousel to user                  | Build Product Carousel         | Log Product Suggestion                    |                                               |
| Log Product Suggestion       | Function                     | Logs product suggestions                         | Respond to User with Carousel  | None                                      |                                               |
| Detect Abandoned Cart        | Shopify Trigger              | Detects abandoned cart event                     | None                             | Wait Period                              |                                               |
| Wait Period                 | Wait                         | Waits before recovery attempt                    | Detect Abandoned Cart          | Fetch Customer Info                      |                                               |
| Fetch Customer Info          | HTTP Request                 | Gets customer and cart data                       | Wait Period                   | Analyze Cart & Customer Data             |                                               |
| Analyze Cart & Customer Data | Function                     | Prepares data for AI analysis                      | Fetch Customer Info            | AI Agent2 (Cart Recovery)                 |                                               |
| AI Agent2 (Cart Recovery)    | Langchain Agent              | AI generates recovery strategy                    | Analyze Cart & Customer Data   | Create Discount (if applicable)           |                                               |
| OpenAI Chat Model for Agent2 | Langchain LM Chat Model       | GPT-4o chat model for cart recovery AI           | AI Agent2                     | AI Agent2                                |                                               |
| Create Discount (if applicable)| Shopify                    | Creates discount codes                             | AI Agent2                     | Send Recovery Email                      |                                               |
| Send Recovery Email          | HTTP Request                 | Sends recovery email                               | Create Discount               | Send SMS Reminder                       |                                               |
| Send SMS Reminder            | Twilio                       | Sends SMS recovery reminder                        | Send Recovery Email           | Follow-Up Wait                          |                                               |
| Follow-Up Wait              | Wait                         | Waits before rechecking recovery                   | Send SMS Reminder             | Check if Still Not Recovered            |                                               |
| Check if Still Not Recovered | IF Node                      | Checks if cart was recovered                         | Follow-Up Wait                | (End or escalate)                        |                                               |
| Hourly Check                | Cron                         | Hourly scheduled trigger                            | None                         | Fetch Inventory                         |                                               |
| Fetch Inventory             | Shopify                      | Gets current inventory data                          | Hourly Check                 | Filter Low Stock (<10)                   |                                               |
| Filter Low Stock (<10)      | IF Node                      | Filters products with less than 10 stock             | Fetch Inventory              | AI Agent3 (Low Stock Reporting)            |                                               |
| AI Agent3 (Low Stock Reporting)| Langchain Agent            | AI formats low stock report                           | Filter Low Stock (<10)        | Notify Slack (Low Stock)                 |                                               |
| OpenAI Chat Model for Agent3| Langchain LM Chat Model       | GPT-4o chat model for low stock AI                   | AI Agent3                    | AI Agent3                                |                                               |
| Notify Slack (Low Stock)    | Slack                        | Sends low stock alert to Slack channel                 | AI Agent3                    | Generate Email Report (Low Stock)        |                                               |
| Generate Email Report (Low Stock)| Email Send              | Emails low stock report                                | Notify Slack                 | Export to Sheets (Low Stock Log)         |                                               |
| Export to Sheets (Low Stock Log)| Google Sheets             | Logs low stock data                                    | Generate Email Report         | None                                      |                                               |
| Trigger Restock Webhook     | Webhook                      | Manual restock alert trigger                            | None                         | Send SMS Alert (Restock)                |                                               |
| Send SMS Alert (Restock)    | Twilio                       | Sends SMS alert for restock                             | Trigger Restock Webhook       | Basic LLM Chain                        |                                               |
| Basic LLM Chain             | Langchain Chain LLM           | Supports restock email generation                       | Send SMS Alert               | Send restock Request Email1            |                                               |
| Send restock Request Email1 | Email Send                   | Sends restock request email                             | Basic LLM Chain              | None                                      |                                               |
| Order Delivered Trigger     | Shopify Trigger              | Trigger when order is delivered                          | None                         | Wait 3 Days                            |                                               |
| Wait 3 Days                 | Wait                         | Waits 3 days post-delivery                               | Order Delivered Trigger      | Fetch Customer Data (for Review)        |                                               |
| Fetch Customer Data (for Review)| HTTP Request             | Gets customer info for review request                    | Wait 3 Days                 | AI Agent4 (Review Request Email)        |                                               |
| AI Agent4 (Review Request Email)| Langchain Agent           | Generates review request email content                   | Fetch Customer Data          | Send Review Request Email               |                                               |
| OpenAI Chat Model for Agent4| Langchain LM Chat Model       | GPT-4o chat model for review request AI                   | AI Agent4                   | AI Agent4                                |                                               |
| Send Review Request Email   | Email Send                   | Sends review request email                               | AI Agent4                   | None                                      |                                               |
| Listen for Review Webhook  | Webhook                      | Receives incoming product reviews                        | None                         | OpenAI, Flag Negative Review            |                                               |
| Flag Negative Review        | IF Node                      | Flags negative reviews                                    | Listen for Review Webhook    | Notify Support Team, Add Review to Database |                                               |
| Notify Support Team (Negative Review)| Slack                | Alerts support team of negative review                     | Flag Negative Review         | Add Review to Database                  |                                               |
| Add Review to Database      | Google Sheets                | Logs all reviews                                         | Flag Negative Review, Notify Support Team | None                              |                                               |
| Campaign Schedule (e.g., Daily at 9 AM)| Cron                | Schedules marketing campaigns                             | None                         | Fetch Target Audience API              |                                               |
| Fetch Target Audience API   | HTTP Request                 | Retrieves target audience data for campaigns              | Campaign Schedule            | Segment Audience Logic                 |                                               |
| Segment Audience Logic      | Function                     | Segments audience based on criteria                        | Fetch Target Audience API    | AI Agent5 (Campaign Content Generation) |                                               |
| AI Agent5 (Campaign Content Generation)| Langchain Agent       | Generates marketing email content                          | Segment Audience Logic       | Prepare Campaign Email                 |                                               |
| OpenAI Chat Model for Agent5| Langchain LM Chat Model       | GPT-4o chat model for campaign content generation          | AI Agent5                   | AI Agent5                                |                                               |
| GenerateCampaignEmailVariant| Langchain ToolCode            | Tool to generate email variants                            | AI Agent5                   | AI Agent5                                |                                               |
| Prepare Campaign Email      | Function                     | Prepares email content for sending                          | AI Agent5                   | Send Campaign Email                   |                                               |
| Send Campaign Email         | Email Send                   | Sends campaign email                                       | Prepare Campaign Email       | Track Campaign Metrics API           |                                               |
| Track Campaign Metrics API  | HTTP Request                 | Fetches campaign performance data                           | Send Campaign Email          | AI Agent6 (Campaign Adjustment)       |                                               |
| AI Agent6 (Campaign Adjustment)| Langchain Agent            | Analyzes campaign performance and suggests adjustments      | Track Campaign Metrics API   | Adjust Campaign (Implement Suggestions) |                                               |
| OpenAI Chat Model for Agent6| Langchain LM Chat Model       | GPT-4o chat model for campaign adjustment AI                 | AI Agent6                   | AI Agent6                                |                                               |
| SuggestCampaignAdjustments  | Langchain ToolCode            | Tool to suggest campaign improvements                       | AI Agent6                   | AI Agent6                                |                                               |
| AnalyzeCampaignPerformance  | Langchain ToolCode            | Tool to analyze campaign metrics                             | AI Agent6                   | AI Agent6                                |                                               |
| Adjust Campaign (Implement Suggestions)| Function               | Implements AI-suggested campaign changes                     | AI Agent6                   | Log Campaign Outcome & Adjustments    |                                               |
| Log Campaign Outcome & Adjustments| Google Sheets             | Logs campaign results and adjustments                        | Adjust Campaign             | None                                      |                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes:**
   - Create `Incoming Message` webhook for customer support messages.
   - Create `Product Inquiry Webhook` for product inquiries.
   - Create `Detect Abandoned Cart` Shopify trigger webhook.
   - Create `Order Delivered Trigger` Shopify trigger webhook.
   - Create `Listen for Review Webhook` for incoming reviews.
   - Create `Trigger Restock Webhook` for manual restock alerts.

2. **Set Up FAQ Retrieval:**
   - Add `Get FAQs Data` Google Sheets node configured to read FAQs.
   - Connect it from `Incoming Message`.

3. **Order Lookup:**
   - Add `Lookup Order API` node (HTTP Request) configured with Shopify API credentials to retrieve order info.
   - Connect it from `Incoming Message`.

4. **Preprocessing Function:**
   - Create `Preprocess Input for Agent` function node to merge message, FAQ, and order info.
   - Connect from `Incoming Message`, `Get FAQs Data`, and `Lookup Order API`.

5. **Customer Support AI Agent:**
   - Add `AI Agent` Langchain agent node, connect `Preprocess Input for Agent` to it.
   - Add `OpenAI Chat Model for Agent` (GPT-4o) linked to the agent.
   - Add Langchain tool nodes `SearchFAQs` and `LookupOrderStatus` linked as AI tools.

6. **Escalation Decision:**
   - Add `Check if Escalation Needed` IF node based on AI output.
   - True branch: Add `Notify Human Agent` Slack node configured with Slack credentials.
   - False branch: Add `Send AI Response to Customer` HTTP Request node configured for sending replies.
   - Both branches connect to `Log Interaction` function node.

7. **Logging Support Interactions:**
   - Add `Log Interaction` function node to prepare log data.
   - Connect to `Store to Notion` node with Notion credentials to save logs.

8. **Product Inquiry Flow:**
   - From `Product Inquiry Webhook`, connect to `Summarize Request & Get Filters` function to extract filters.
   - Connect to `Search Products API` HTTP Request node calling Shopify products API.
   - Connect to `Filter Out of Stock` IF node filtering zero stock.
   - Pass to `Sort by Rating` function node.
   - Connect to `Filter by Price` IF node.
   - Connect to `AI Agent1 (Product Selection Refinement)` Langchain agent node.
   - Add `OpenAI Chat Model for Agent1` GPT-4o linked to agent.
   - Add `RefineProductSelection` Langchain tool for product refinement.
   - Connect to `Build Product Carousel` function node.
   - Connect to `Respond to User with Carousel` HTTP Request node.
   - Connect to `Log Product Suggestion` function node.

9. **Abandoned Cart Recovery:**
   - From `Detect Abandoned Cart` Shopify trigger, connect to `Wait Period` node.
   - Connect to `Fetch Customer Info` HTTP Request node.
   - Connect to `Analyze Cart & Customer Data` function node.
   - Connect to `AI Agent2 (Cart Recovery)` Langchain agent node.
   - Add `OpenAI Chat Model for Agent2` GPT-4o linked to agent.
   - Add `DetermineDiscount` Langchain tool node.
   - Connect to `Create Discount (if applicable)` Shopify node.
   - Connect to `Send Recovery Email` HTTP Request node.
   - Connect to `Send SMS Reminder` Twilio node.
   - Connect to `Follow-Up Wait` wait node.
   - Connect to `Check if Still Not Recovered` IF node for final status.

10. **Inventory Monitoring:**
    - Add `Hourly Check` cron node scheduled hourly.
    - Connect to `Fetch Inventory` Shopify node.
    - Connect to `Filter Low Stock (<10)` IF node.
    - Connect to `AI Agent3 (Low Stock Reporting)` Langchain agent node.
    - Add `OpenAI Chat Model for Agent3` GPT-4o linked to agent.
    - Add `FormatLowStockReport` Langchain tool.
    - Connect to `Notify Slack (Low Stock)` Slack node.
    - Connect to `Generate Email Report (Low Stock)` email send node.
    - Connect to `Export to Sheets (Low Stock Log)` Google Sheets node.

11. **Manual Restock Alert:**
    - From `Trigger Restock Webhook` webhook, connect to `Send SMS Alert (Restock)` Twilio node.
    - Connect to `Basic LLM Chain` Langchain chain node.
    - Connect to `Send restock Request Email1` email send node.

12. **Post-Purchase Review Requests:**
    - From `Order Delivered Trigger` Shopify trigger, connect to `Wait 3 Days` wait node.
    - Connect to `Fetch Customer Data (for Review)` HTTP Request node.
    - Connect to `AI Agent4 (Review Request Email)` Langchain agent node.
    - Add `OpenAI Chat Model for Agent4` GPT-4o linked.
    - Add `DraftReviewRequestEmail` Langchain tool.
    - Connect to `Send Review Request Email` email send node.

13. **Review Monitoring:**
    - From `Listen for Review Webhook`, connect to `OpenAI` chat node for sentiment analysis.
    - Connect to `Flag Negative Review` IF node.
    - True branch: Connect to `Notify Support Team (Negative Review)` Slack node.
    - False branch and true branch both connect to `Add Review to Database` Google Sheets node.

14. **Email Campaign Automation:**
    - Add `Campaign Schedule` cron node for scheduled triggers.
    - Connect to `Fetch Target Audience API` HTTP Request node.
    - Connect to `Segment Audience Logic` function node.
    - Connect to `AI Agent5 (Campaign Content Generation)` Langchain agent node.
    - Add `OpenAI Chat Model for Agent5` GPT-4o linked.
    - Add `GenerateCampaignEmailVariant` Langchain tool.
    - Connect to `Prepare Campaign Email` function node.
    - Connect to `Send Campaign Email` email send node.
    - Connect to `Track Campaign Metrics API` HTTP Request node.
    - Connect to `AI Agent6 (Campaign Adjustment)` Langchain agent node.
    - Add `OpenAI Chat Model for Agent6` GPT-4o linked.
    - Add `AnalyzeCampaignPerformance` and `SuggestCampaignAdjustments` Langchain tools.
    - Connect to `Adjust Campaign (Implement Suggestions)` function node.
    - Connect to `Log Campaign Outcome & Adjustments` Google Sheets node.

15. **Credentials Setup:**
    - Set up Shopify API credentials with read/write access.
    - Configure Google Sheets API credentials with access to relevant sheets.
    - Configure Slack OAuth2 credentials for messaging.
    - Configure Twilio credentials for SMS.
    - Set Notion integration credentials.
    - Configure OpenAI API key with GPT-4o access.
    - Configure email SMTP or service credentials for email sending.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow leverages Langchain agents tightly integrated with Shopify and multiple communication channels for automation.  | Core conceptual design for AI-driven e-commerce automation                                        |
| The AI agents use GPT-4o chat models for advanced natural language understanding and generation.                              | Requires OpenAI API access with GPT-4o enabled                                                    |
| Slack integration is used for human escalation and operational alerts such as low stock and negative reviews.                  | Slack OAuth2 app setup with appropriate scopes                                                    |
| Twilio SMS is used for customer reminders and restock alerts.                                                                   | Twilio account and phone number setup required                                                   |
| Google Sheets and Notion are used for logging and data tracking to maintain audit trails and analytics.                        | Google API and Notion API credentials setup                                                       |
| Shopify nodes use OAuth or private app credentials with read/write scopes on orders, products, and discounts.                   | Shopify app credentials configuration                                                            |
| Campaign automation uses scheduled cron triggers and AI-generated content with feedback loops from performance tracking APIs. | Supports dynamic marketing optimization                                                          |
| Sticky notes in the original workflow provide contextual descriptions for each functional block for easier understanding.      | Located near relevant node groups in the n8n editor                                              |

---

This document fully outlines the structure, logic, and configuration of the "Shopify AI Agent Suite: Full E-commerce Automation" workflow, enabling detailed understanding, reproduction, and troubleshooting.