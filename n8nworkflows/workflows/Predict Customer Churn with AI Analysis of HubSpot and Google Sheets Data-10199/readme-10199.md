Predict Customer Churn with AI Analysis of HubSpot and Google Sheets Data

https://n8nworkflows.xyz/workflows/predict-customer-churn-with-ai-analysis-of-hubspot-and-google-sheets-data-10199


# Predict Customer Churn with AI Analysis of HubSpot and Google Sheets Data

### 1. Workflow Overview

This workflow automates the process of predicting customer churn risk by analyzing combined data from HubSpot CRM and Google Sheets using AI-driven sentiment analysis and feature usage evaluation. It is designed for Customer Success teams needing early alerts on customers likely to churn by synthesizing deal data, customer support ticket sentiment, and product feature usage trends.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception and Ticket Sentiment Analysis:** Receives customer support ticket data via webhook, formats and analyzes sentiment per ticket, then aggregates sentiment scores.

- **1.2 Deal Iteration and Customer Data Gathering:** Iterates over all HubSpot deals to isolate each deal ID for detailed data enrichment, including deal metadata, contact email, tickets, and feature usage.

- **1.3 AI-Based Churn Risk Assessment:** Uses an AI agent to collect and structure comprehensive data per customer, then applies an AI chain to analyze churn signals based on deal age, sentiment score, and usage decline, generating actionable churn alert content if thresholds are exceeded.

- **1.4 Alert Formatting and Notification:** Converts AI-generated alert content to HTML and sends an email notification to Customer Success Managers, looping back to process all deals.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Ticket Sentiment Analysis

**Overview:**  
This block accepts incoming tickets (e.g., from HubSpot), formats them for analysis, splits them individually, analyzes sentiment using AI, converts sentiment categories to numeric scores, sums all scores, and returns the aggregated sentiment score.

**Nodes Involved:**  
- Trigger: Receive Tickets for Scoring  
- Code: Format Tickets for Analysis  
- Loop: For Each Ticket  
- AI: Analyze Ticket Sentiment  
- Merge: Consolidate Results  
- Code: Convert Sentiment to Score  
- Code: Sum All Ticket Scores  
- Respond: Return Total Score  Export to Sheets  

**Node Details:**

- **Trigger: Receive Tickets for Scoring**  
  - Type: Webhook  
  - Role: Receives POST requests with batches of tickets to analyze.  
  - Config: POST method, webhook path fixed, returns response from downstream node.  
  - Inputs: HTTP POST with tickets JSON.  
  - Outputs: Raw tickets JSON passed downstream.  
  - Failures: Invalid JSON, webhook not reachable, malformed ticket data.  

- **Code: Format Tickets for Analysis**  
  - Type: Code (JavaScript)  
  - Role: Parses incoming tickets, concatenates subject and content into text for sentiment analysis.  
  - Key Logic: Extracts ticket text lines for each ticket, handles presence/absence of properties wrapper.  
  - Inputs: Raw tickets JSON.  
  - Outputs: JSON array of ticket texts.  
  - Failures: JSON parse errors, missing fields.

- **Loop: For Each Ticket**  
  - Type: SplitOut  
  - Role: Splits array of ticket texts into individual items for separate sentiment analysis.  
  - Inputs: Array of ticket texts.  
  - Outputs: Single ticket text per item.

- **AI: Analyze Ticket Sentiment**  
  - Type: LangChain Sentiment Analysis  
  - Role: Classifies each ticket text as Positive, Neutral, or Negative sentiment.  
  - Inputs: Single ticket text string.  
  - Outputs: Sentiment category per ticket.  
  - Failures: AI service downtime, API limits, malformed input.

- **Merge: Consolidate Results**  
  - Type: Merge  
  - Role: Combines parallel sentiment analysis outputs back into a single stream for further processing.  
  - Inputs: Multiple sentiment outputs.  
  - Outputs: Single merged output.

- **Code: Convert Sentiment to Score**  
  - Type: Code (JavaScript)  
  - Role: Maps sentiment categories to numeric scores (Positive=10, Neutral=5, Negative=-10).  
  - Inputs: Sentiment category per ticket.  
  - Outputs: Numeric score per ticket.

- **Code: Sum All Ticket Scores**  
  - Type: Code (JavaScript)  
  - Role: Sums all individual ticket scores to produce an aggregate sentiment score.  
  - Inputs: Array of ticket scores.  
  - Outputs: Single totalScore number.

- **Respond: Return Total Score  Export to Sheets**  
  - Type: Respond to Webhook  
  - Role: Sends the total sentiment score back as webhook response to the caller.  
  - Inputs: Total score JSON.  
  - Outputs: HTTP response.  

---

#### 1.2 Deal Iteration and Customer Data Gathering

**Overview:**  
This block triggers on manual execution, retrieves all deals from HubSpot, loops over each deal, isolates the deal ID, and invokes an AI agent to gather enriched customer data including deal details, contacts, tickets, and feature usage from Google Sheets.

**Nodes Involved:**  
- Manual Trigger: Run Churn Analysis  
- HubSpot: Get All Deals  
- Start Loop: For Each Deal  
- Set: Isolate Current Deal ID  Export to Sheets  
- AI Agent: Gather Customer Data  
- Config: Set LLM for Agent & Chains  
- Tool: Get HubSpot Data  
- Tool: Calculate Sentiment Score  
- Tool: Get Feature Usage from Sheets  
- AI: Structure Agent's Findings  Export to Sheets  

**Node Details:**

- **Manual Trigger: Run Churn Analysis**  
  - Type: Manual Trigger  
  - Role: Entry point for the workflow to start processing all deals on-demand.  
  - Inputs: None.  
  - Outputs: Triggers next node.

- **HubSpot: Get All Deals**  
  - Type: HubSpot  
  - Role: Fetches all deals via HubSpot API using OAuth2.  
  - Config: No filters, resource 'deal', operation 'getAll'.  
  - Inputs: Trigger.  
  - Outputs: Array of deal objects.  
  - Failures: OAuth token expiration, API rate limits.

- **Start Loop: For Each Deal**  
  - Type: SplitInBatches  
  - Role: Iterates over each deal to process them individually.  
  - Inputs: Array of deals.  
  - Outputs: Single deal per iteration.

- **Set: Isolate Current Deal ID  Export to Sheets**  
  - Type: Set  
  - Role: Extracts dealId from current deal for downstream processing.  
  - Inputs: Single deal JSON.  
  - Outputs: JSON with 'dealId' field.

- **AI Agent: Gather Customer Data**  
  - Type: LangChain Agent  
  - Role: Orchestrates multi-step data collection for scoring via multiple tools: HubSpot data, contact emails, ticket sentiment, feature usage.  
  - Config: Prompt commands to retrieve deal info, contact email, tickets, and feature usage with tools.  
  - Inputs: dealId.  
  - Outputs: Structured JSON with comprehensive customer data including deal metadata, contact email, tickets, sentiment score, and feature usage.  
  - Failures: Tool endpoint failures, AI parsing errors, missing data.

- **Config: Set LLM for Agent & Chains**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Defines the OpenAI GPT model "gpt-5-mini" for all AI nodes requiring LLM capabilities.  
  - Inputs: None.  
  - Outputs: Model config reference for AI nodes.

- **Tool: Get HubSpot Data**  
  - Type: MCP Client Tool (custom integration)  
  - Role: Fetches detailed HubSpot data such as deal details, contacts, tickets as requested by AI Agent.  
  - Inputs: Structured queries from AI Agent.  
  - Outputs: HubSpot data JSON.

- **Tool: Calculate Sentiment Score**  
  - Type: HTTP Request Tool  
  - Role: Invokes the workflow’s own webhook to analyze all tickets at once and return a consolidated sentiment score.  
  - Inputs: Tickets JSON, POST body.  
  - Outputs: Sentiment score.  
  - Failures: Network errors, webhook unreachable.

- **Tool: Get Feature Usage from Sheets**  
  - Type: Google Sheets Tool  
  - Role: Retrieves feature usage data from a Google Sheet based on contact email.  
  - Config: Document ID and sheet name configured externally.  
  - Inputs: Contact email from AI Agent.  
  - Outputs: Feature usage data array.

- **AI: Structure Agent's Findings  Export to Sheets**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses AI Agent output into a structured JSON schema with fields including companyName, email, dealstage, score, tickets, and feature usage.  
  - Inputs: Raw AI Agent output.  
  - Outputs: Structured customer data JSON.

---

#### 1.3 AI-Based Churn Risk Assessment

**Overview:**  
This block groups feature usage data by feature name, then applies an AI chain to analyze churn risk signals based on deal age, sentiment score, and feature usage trends. If churn risk is detected, it generates a professional alert email content.

**Nodes Involved:**  
- Code: Group Feature Data by Name  
- AI Chain: Analyze for Churn Risk  
- AI: Structure Alert Email  

**Node Details:**

- **Code: Group Feature Data by Name**  
  - Type: Code (JavaScript)  
  - Role: Groups feature usage entries by their name into a dictionary for easier trend analysis.  
  - Inputs: Structured customer data JSON with feature usage array.  
  - Outputs: JSON stringified grouped feature data under key `res`.  
  - Failures: Missing or malformed feature data.

- **AI Chain: Analyze for Churn Risk**  
  - Type: LangChain Chain LLM  
  - Role: Analyzes multiple churn signals: deal age > 1 year, negative sentiment score, declining usage. Generates a clear, concise churn alert email body and subject if any threshold is exceeded.  
  - Inputs: Customer data including `hs_is_closed_timestamp`, sentiment `score`, and grouped feature usage data.  
  - Outputs: JSON with alert `subject` and `message`.  
  - Failures: AI model errors, prompt parsing errors.

- **AI: Structure Alert Email**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses the raw AI output email text into structured JSON with `subject` and `message` fields.  
  - Inputs: Raw AI email content.  
  - Outputs: Parsed email JSON.

---

#### 1.4 Alert Formatting and Notification

**Overview:**  
This final block converts the AI-generated alert message from markdown to HTML, sends the alert email, then loops to the next deal for processing.

**Nodes Involved:**  
- Format: Convert Alert to HTML  
- Email: Send Churn Alert  Export to Sheets  

**Node Details:**

- **Format: Convert Alert to HTML**  
  - Type: Markdown  
  - Role: Converts the markdown formatted alert message into HTML for email formatting.  
  - Inputs: AI structured alert message.  
  - Outputs: HTML message in `message` field.

- **Email: Send Churn Alert  Export to Sheets**  
  - Type: Email Send  
  - Role: Sends churn alert emails to Customer Success Managers with the formatted HTML message and subject line.  
  - Config: SMTP credentials configured with Postale account; `From` and `To` emails set here.  
  - Inputs: HTML message and email subject.  
  - Outputs: Email delivery confirmation.  
  - Failures: SMTP auth errors, invalid email addresses.

- After email send, the workflow loops back to process the next deal from the batch until all deals are processed.

---

### 3. Summary Table

| Node Name                            | Node Type                          | Functional Role                                         | Input Node(s)                     | Output Node(s)                              | Sticky Note                                                                                          |
|------------------------------------|----------------------------------|--------------------------------------------------------|----------------------------------|---------------------------------------------|---------------------------------------------------------------------------------------------------|
| Sticky Note                        | Sticky Note                      | Overview and setup instructions                         |                                  |                                             | # Customer Health Scoring & Churn Alert; Setup info including credential and URL configuration.    |
| Sticky Note1                      | Sticky Note                      | Contact info for workflow modification support         |                                  |                                             | ## Contact me; Email: thomas@pollup.net                                                           |
| Sticky Note2                      | Sticky Note                      | Explains sentiment scoring tool calls workflow webhook |                                  |                                             | Tool calls this workflow's own webhook to process all tickets at once and return a single score.  |
| Sticky Note3                      | Sticky Note                      | AI prompt explanation for churn signals                 |                                  |                                             | AI prompt checks three churn signals: deal age > 1 year, negative sentiment, declining usage.      |
| Manual Trigger: Run Churn Analysis| Manual Trigger                  | Starts the workflow on demand                           |                                  | HubSpot: Get All Deals                      |                                                                                                   |
| HubSpot: Get All Deals            | HubSpot                         | Retrieves all HubSpot deals                             | Manual Trigger                   | Start Loop: For Each Deal                    |                                                                                                   |
| Start Loop: For Each Deal         | SplitInBatches                  | Iterates over each deal                                 | HubSpot: Get All Deals           | Set: Isolate Current Deal ID  Export to Sheets |                                                                                                   |
| Set: Isolate Current Deal ID  Export to Sheets | Set                    | Extract dealId for further data gathering                | Start Loop: For Each Deal        | AI Agent: Gather Customer Data               |                                                                                                   |
| AI Agent: Gather Customer Data    | LangChain Agent                | Collects enriched deal, contact, tickets, features data | Set: Isolate Current Deal ID     | Code: Group Feature Data by Name             |                                                                                                   |
| Config: Set LLM for Agent & Chains| LangChain LM Chat OpenAI       | Sets GPT model for AI nodes                             |                                  | AI: Analyze Ticket Sentiment, AI Agent, AI Chain |                                                                                                   |
| Tool: Get HubSpot Data            | MCP Client Tool                | Fetches HubSpot deal/contact/ticket data               | AI Agent                        | AI Agent                                    |                                                                                                   |
| Tool: Calculate Sentiment Score   | HTTP Request Tool              | Calls webhook to analyze all tickets sentiment         | AI Agent                        | AI Agent                                    |                                                                                                   |
| Tool: Get Feature Usage from Sheets| Google Sheets Tool             | Retrieves feature usage data for a contact             | AI Agent                        | AI Agent                                    |                                                                                                   |
| AI: Structure Agent's Findings  Export to Sheets | LangChain Output Parser Structured | Parses AI agent output to structured JSON              | AI Agent                        | Code: Group Feature Data by Name             |                                                                                                   |
| Code: Group Feature Data by Name  | Code                          | Groups feature usage data by feature name               | AI: Structure Agent's Findings  | AI Chain: Analyze for Churn Risk             |                                                                                                   |
| AI Chain: Analyze for Churn Risk  | LangChain Chain LLM            | Detects churn signals and generates alert email content| Code: Group Feature Data by Name| AI: Structure Alert Email                     |                                                                                                   |
| AI: Structure Alert Email         | LangChain Output Parser Structured | Parses churn alert email subject and message           | AI Chain: Analyze for Churn Risk | Format: Convert Alert to HTML                 |                                                                                                   |
| Format: Convert Alert to HTML     | Markdown                      | Converts alert markdown message to HTML                 | AI: Structure Alert Email       | Email: Send Churn Alert  Export to Sheets    |                                                                                                   |
| Email: Send Churn Alert  Export to Sheets | Email Send                    | Sends churn alert email to Customer Success Manager     | Format: Convert Alert to HTML    | Start Loop: For Each Deal                      |                                                                                                   |
| Trigger: Receive Tickets for Scoring| Webhook                      | Receives tickets for sentiment analysis                 |                                  | Code: Format Tickets for Analysis             |                                                                                                   |
| Code: Format Tickets for Analysis | Code                          | Extracts ticket text for sentiment analysis             | Trigger: Receive Tickets         | Loop: For Each Ticket                         |                                                                                                   |
| Loop: For Each Ticket             | SplitOut                      | Splits tickets into individual items                     | Code: Format Tickets for Analysis| AI: Analyze Ticket Sentiment                  |                                                                                                   |
| AI: Analyze Ticket Sentiment      | LangChain Sentiment Analysis  | Analyzes sentiment of each ticket                        | Loop: For Each Ticket            | Merge: Consolidate Results                    |                                                                                                   |
| Merge: Consolidate Results        | Merge                         | Combines sentiment analysis results                      | AI: Analyze Ticket Sentiment     | Code: Convert Sentiment to Score              |                                                                                                   |
| Code: Convert Sentiment to Score  | Code                          | Converts sentiment categories to numeric scores         | Merge: Consolidate Results       | Code: Sum All Ticket Scores                   |                                                                                                   |
| Code: Sum All Ticket Scores       | Code                          | Aggregates all ticket scores into a total score         | Code: Convert Sentiment to Score | Respond: Return Total Score  Export to Sheets |                                                                                                   |
| Respond: Return Total Score  Export to Sheets | Respond to Webhook             | Returns aggregated sentiment score                       | Code: Sum All Ticket Scores      |                                             |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create a **Manual Trigger** node named `Manual Trigger: Run Churn Analysis`.

**Step 2:** Add a **HubSpot** node named `HubSpot: Get All Deals` to fetch all deals using OAuth2. Set resource to "deal" and operation to "getAll". Connect it to the manual trigger.

**Step 3:** Add a **SplitInBatches** node named `Start Loop: For Each Deal` connected to the HubSpot node to process deals one by one.

**Step 4:** Add a **Set** node named `Set: Isolate Current Deal ID  Export to Sheets` to extract the current deal’s `dealId`. Connect it to the loop node.

**Step 5:** Add a **LangChain LM Chat OpenAI** node named `Config: Set LLM for Agent & Chains`. Configure it to use the GPT model “gpt-5-mini”. This node will be referenced by all AI nodes needing LLM.

**Step 6:** Add a **LangChain Agent** node named `AI Agent: Gather Customer Data`. Configure its prompt to:

- Retrieve deal info for the current dealId using `Tool: Get HubSpot Data`.  
- Extract companyName, associatedVids, deal close timestamp.  
- Using associatedVids, fetch primary contact’s email via `Tool: Get HubSpot Data`.  
- Get all tickets for the contact and send them to `Tool: Calculate Sentiment Score`.  
- Retrieve feature usage data from Google Sheets for the contact email using `Tool: Get Feature Usage from Sheets`.

Connect the Set node to this agent node, and configure it to use the `Config: Set LLM for Agent & Chains`.

**Step 7:** Add the following tool nodes connected as AI tools to the agent:

- `Tool: Get HubSpot Data`: MCP Client Tool to fetch HubSpot details.  
- `Tool: Calculate Sentiment Score`: HTTP Request Tool that posts tickets to your own webhook for sentiment scoring.  
- `Tool: Get Feature Usage from Sheets`: Google Sheets Tool configured with your Sheet ID and sheet name to get feature usage.

**Step 8:** Add a **LangChain Output Parser Structured** node named `AI: Structure Agent's Findings  Export to Sheets` to parse the agent’s output to a structured JSON schema. Connect it to the agent node’s output parser.

**Step 9:** Add a **Code** node named `Code: Group Feature Data by Name` to group feature usage data by feature name for churn analysis.

**Step 10:** Add a **LangChain Chain LLM** node named `AI Chain: Analyze for Churn Risk`. Configure the prompt to:

- Trigger alerts when: deal is older than 1 year, sentiment score < 0, or usage/user count declined.  
- Generate a professional email subject and concise body with churn factors, risk level, and recommended actions.  
- Use a clear, proactive tone.

Connect the code node’s output to this node, and configure it to use the LLM configuration node.

**Step 11:** Add a **LangChain Output Parser Structured** node named `AI: Structure Alert Email` to parse the alert email subject and message.

**Step 12:** Add a **Markdown** node named `Format: Convert Alert to HTML`. Configure it to convert the AI alert message from markdown to HTML, outputting `message` field.

**Step 13:** Add an **Email Send** node named `Email: Send Churn Alert  Export to Sheets`. Configure SMTP credentials (e.g., Postale) and set From/To email addresses. Use the HTML message and subject from the previous node.

**Step 14:** Connect the email send node back to the `Start Loop: For Each Deal` node to process the next deal.

---

**Parallel Sub-Workflow: Ticket Sentiment Scoring**

**Step 15:** Create a **Webhook** node named `Trigger: Receive Tickets for Scoring`. Set POST method and appropriate path.

**Step 16:** Add a **Code** node `Code: Format Tickets for Analysis` that parses the incoming tickets JSON and concatenates subject and content into ticket text array.

**Step 17:** Add a **SplitOut** node `Loop: For Each Ticket` to process each ticket separately.

**Step 18:** Add a **LangChain Sentiment Analysis** node `AI: Analyze Ticket Sentiment` to classify sentiment per ticket.

**Step 19:** Add a **Merge** node `Merge: Consolidate Results` to combine individual sentiment results.

**Step 20:** Add a **Code** node `Code: Convert Sentiment to Score` mapping sentiment categories to scores (Positive=10, Neutral=5, Negative=-10).

**Step 21:** Add a **Code** node `Code: Sum All Ticket Scores` to sum all ticket scores into one total score.

**Step 22:** Add a **Respond to Webhook** node `Respond: Return Total Score  Export to Sheets` to send the total score back as webhook response.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow automatically calculates customer health scores by combining CRM, support ticket sentiment, and product usage data, then sends alerts for churn risks.                                                        | Sticky Note (top left)                                                                                             |
| Contact for modifications, help, or workflow development in n8n, Make, Langchain, or Langgraph: thomas@pollup.net                                                                                                            | Sticky Note1 (bottom left)                                                                                          |
| The sentiment scoring tool calls this workflow’s webhook to process all tickets together and return a single sentiment score.                                                                                                | Sticky Note2 (middle right)                                                                                         |
| AI prompt for churn risk analysis checks three signals: deal age > 1 year, negative sentiment score, and declining feature usage over time.                                                                                  | Sticky Note3 (middle right)                                                                                         |
| Model used for AI nodes is “gpt-5-mini” (requires n8n version supporting this model and LangChain integration).                                                                                                              | Config node                                                                                                        |
| Google Sheets document ID and sheet name must be configured in `Tool: Get Feature Usage from Sheets`.                                                                                                                         | Tool: Get Feature Usage from Sheets node                                                                           |
| Email node requires SMTP credentials (Postale account used here) and proper From/To addresses to send churn alerts.                                                                                                           | Email: Send Churn Alert  Export to Sheets node                                                                     |

---

This documentation fully covers the workflow’s structure, detailed node functions, and setup instructions to enable both manual reconstruction and programmatic understanding. It anticipates API failures, data parsing issues, and credential configurations.