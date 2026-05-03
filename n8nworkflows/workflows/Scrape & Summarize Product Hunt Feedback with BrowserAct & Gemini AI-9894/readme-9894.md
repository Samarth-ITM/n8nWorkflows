Scrape & Summarize Product Hunt Feedback with BrowserAct & Gemini AI

https://n8nworkflows.xyz/workflows/scrape---summarize-product-hunt-feedback-with-browseract---gemini-ai-9894


# Scrape & Summarize Product Hunt Feedback with BrowserAct & Gemini AI

### 1. Workflow Overview

This workflow automates competitive intelligence gathering by scraping, analyzing, and summarizing user feedback on Product Hunt product launches. It is designed to extract actionable insights from community comments, helping product teams monitor competitor launches and gain strategic feedback.

The workflow is structured into four main logical blocks:

- **1.1 Input & Loop:** Fetches a list of product names from a Google Sheet and loops through each product to process it individually.
- **1.2 Data Collection:** Uses BrowserAct nodes to scrape all comments from the specified Product Hunt launch page.
- **1.3 AI Processing:** An AI Agent powered by Google Gemini analyzes the scraped comments, producing a structured summary highlighting positive, negative, and overall feedback with recommendations.
- **1.4 Reporting & Notification:** Stores the AI-generated analysis in a Google Sheet and sends a Slack notification to alert the team.

Each block is functionally cohesive and connected linearly, supporting automation from data ingestion to insight delivery.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Loop

**Overview:**  
This block initiates the workflow manually, retrieves the list of products to analyze from a Google Sheet, and loops over each product to process them one at a time.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get row(s) in sheet (Google Sheets)  
- Loop Over Items (SplitInBatches)  

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow when user clicks ‘Execute workflow’ in n8n UI.  
  - Config: No parameters; manual activation only.  
  - Inputs: None  
  - Outputs: Connects to “Get row(s) in sheet”  
  - Edge cases: No automation unless replaced with a schedule trigger; manual process only.

- **Get row(s) in sheet**  
  - Type: Google Sheets  
  - Role: Retrieves rows from a specific sheet containing product names to analyze.  
  - Config:  
    - Document ID and Sheet ID specify spreadsheet and sheet with product list.  
  - Inputs: From manual trigger  
  - Outputs: List of rows (products) to process  
  - Edge cases: Sheet access errors, empty rows, or incorrect sheet ID may cause failure.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each product row individually to avoid mixing data between items.  
  - Config: Default batching behavior (process one item at a time)  
  - Inputs: List of products from Google Sheets  
  - Outputs: Each product passed individually downstream  
  - Edge cases: Empty input list results in no loop iterations.

---

#### 1.2 Data Collection

**Overview:**  
This block uses BrowserAct to scrape all user comments from the Product Hunt launch page of the specified product, based on parameters passed from the loop.

**Nodes Involved:**  
- Run a workflow task (BrowserAct)  
- Get details of a workflow task (BrowserAct)

**Node Details:**  

- **Run a workflow task**  
  - Type: BrowserAct Workflow Node  
  - Role: Starts the scraping task using a predefined BrowserAct workflow template.  
  - Config:  
    - Workflow ID: “Product Hunt Launch Monitor” BrowserAct template ID  
    - Input Parameters:  
      - ProductName: dynamically set from current product name in loop (`{{$json["Product name"]}}`)  
      - Total_review: fixed at 30 (number of reviews to scrape)  
    - Additional: Saves browser data for task continuity  
  - Credentials: BrowserAct API account  
  - Inputs: Product info from loop  
  - Outputs: Task ID for scraping job  
  - Edge cases: API failures, invalid product names, scraping timeouts.

- **Get details of a workflow task**  
  - Type: BrowserAct Workflow Node  
  - Role: Polls BrowserAct for scraping job completion and fetches results.  
  - Config:  
    - Task ID: dynamically set from previous node output (`{{$json.id}}`)  
    - Operation: getTask  
    - Max Wait Time: 600 seconds (10 minutes)  
    - Wait For Finish: true (waits synchronously until task completes)  
  - Credentials: BrowserAct API account  
  - Inputs: Task ID from previous node  
  - Outputs: Scraped comments data  
  - Edge cases: Task failure, timeout, incomplete data if scraping fails.

---

#### 1.3 AI Processing

**Overview:**  
The AI Agent uses Google Gemini to analyze the scraped comments and generate a structured summary with prioritized insights categorized as positive, negative, overall summary, product name, and actionable recommendations.

**Nodes Involved:**  
- AI Agent (LangChain Agent)  
- Structured Output Parser (LangChain Output Parser)  
- Gemini (Google Gemini LM Chat)  

**Node Details:**  

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Processes raw comment text using a custom prompt to extract competitive intelligence insights.  
  - Config:  
    - Text input: concatenated scraped comments (`{{$json.output.string}}`)  
    - System Message: instructs agent as competitive intelligence analyst  
    - Output: JSON structured with fields Positive, Negative, Summary, Product, Recommendation  
  - Input: Scraped comments from BrowserAct  
  - Output: JSON string with categorized insights  
  - Edge cases: Parsing errors if input text malformed, API quota limits, response delays.

- **Structured Output Parser**  
  - Type: LangChain Output Parser (Structured)  
  - Role: Parses AI Agent's JSON string output into usable JSON object for downstream nodes.  
  - Config: JSON schema example with keys: Positive, Negative, Summary, Product, Recommendation  
  - Input: AI Agent output  
  - Output: Parsed JSON object  
  - Edge cases: Parsing failures if AI output malformed or unexpected format.

- **Gemini**  
  - Type: Google Gemini LM Chat Node  
  - Role: Provides underlying language model support for AI Agent (indirectly linked)  
  - Config: Credentials for Google Gemini API  
  - Input: Connected to AI Agent as language model provider  
  - Edge cases: API failures, quota limits, latency.

---

#### 1.4 Reporting & Notification

**Overview:**  
This block appends the structured competitive intelligence summary to a Google Sheet and sends a Slack message notification to inform the team that new analysis is available.

**Nodes Involved:**  
- Append row in sheet (Google Sheets)  
- Send a message (Slack)  

**Node Details:**  

- **Append row in sheet**  
  - Type: Google Sheets  
  - Role: Adds a new row to a target Google Sheet with AI-generated insights.  
  - Config:  
    - Document ID and Sheet Name specify target spreadsheet and worksheet  
    - Columns mapped to AI output fields: Product, Summary, Negative, Positive, Recommendation  
  - Inputs: Parsed JSON output from AI Agent  
  - Outputs: Loop continues or ends  
  - Edge cases: Google Sheets API limits, incorrect mapping, authentication failures.

- **Send a message**  
  - Type: Slack  
  - Role: Sends a Slack channel message notifying that new Product Hunt analysis is appended.  
  - Config:  
    - Channel ID: preconfigured Slack channel for notifications  
    - Message Text: static text indicating update completion with key highlights  
    - Authentication: OAuth2 Slack credentials  
  - Inputs: Triggered in parallel with scraping task per loop iteration  
  - Edge cases: Slack API failures, channel ID invalid, OAuth token expiration.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                          | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                                              |
|----------------------------|----------------------------------|----------------------------------------|--------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Manual start of workflow                | None                           | Get row(s) in sheet             | ## Try It Out! This template provides automated competitive intelligence by scraping and summarizing Product Hunt launch feedback with a specialized AI analyst. … |
| Get row(s) in sheet        | Google Sheets                     | Fetch list of products to process      | When clicking ‘Execute workflow’ | Loop Over Items                 | ### 📋 1. Input & Loop * Schedule Trigger kicks off process * Google Sheets fetches product list * Loop processes each individually |
| Loop Over Items             | SplitInBatches                   | Process each product individually      | Get row(s) in sheet            | Send a message, Run a workflow task | ### 📋 1. Input & Loop * Schedule Trigger kicks off process * Google Sheets fetches product list * Loop processes each individually |
| Send a message              | Slack                            | Notify team via Slack                   | Loop Over Items                | None                           | ### ⏰ 4. Storing the Analysis * Slack node sends actionable alerts to your team                                         |
| Run a workflow task         | BrowserAct Workflow              | Start scraping task on Product Hunt    | Loop Over Items                | Get details of a workflow task | ### 📊 1. Data Collection * BrowserAct nodes scrape comments from Product Hunt                                           |
| Get details of a workflow task | BrowserAct Workflow              | Poll and retrieve scraped comments     | Run a workflow task            | AI Agent                      | ### 📊 1. Data Collection * BrowserAct nodes scrape comments from Product Hunt                                           |
| AI Agent                   | LangChain Agent                  | Analyze scraped comments with Gemini AI | Get details of a workflow task | Append row in sheet             | ### 🧠 2. The AI Analyst * AI Agent synthesizes comments into structured insights                                        |
| Structured Output Parser   | LangChain Output Parser Structured | Parse AI Agent JSON output               | AI Agent                      | AI Agent                      | ### 🧠 2. The AI Analyst * AI Agent synthesizes comments into structured insights                                        |
| Gemini                     | LangChain LM Chat Google Gemini | Language model for AI Agent             | None                         | AI Agent                      | ### 🧠 2. The AI Analyst * AI Agent synthesizes comments into structured insights                                        |
| Append row in sheet        | Google Sheets                   | Save AI summary to Google Sheet          | AI Agent                      | Loop Over Items               | ### 📈 3. Storing the Analysis * Google Sheets node logs competitor feedback for review and tracking                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow on demand.

2. **Add Google Sheets Node to Get Rows**  
   - Type: Google Sheets (Read operation)  
   - Configure:  
     - Document ID: Your Google Sheet ID containing product list  
     - Sheet Name or GID: Target sheet with product names  
   - Connect input from Manual Trigger.

3. **Add SplitInBatches Node (Loop Over Items)**  
   - Type: SplitInBatches  
   - Purpose: Process each product row one at a time  
   - Connect input from Google Sheets node.

4. **Add Slack Node (Send a message)**  
   - Type: Slack  
   - Configure:  
     - OAuth2 credentials for Slack account  
     - Channel ID: Target Slack channel for notifications  
     - Message Text: “The Product Hunt result for has been updated - Positive and negative node analyzed the recommendation added for the product”  
   - Connect input from SplitInBatches node (main output 1).

5. **Add BrowserAct Node to Run Workflow Task**  
   - Type: BrowserAct Workflow  
   - Configure:  
     - Credentials: BrowserAct API credentials  
     - Workflow ID: Use BrowserAct template “Product Hunt Launch Monitor”  
     - Input Parameters:  
       - ProductName: Set to `={{ $json["Product name"] }}` (from loop)  
       - Total_review: 30 (number of comments to scrape)  
     - Additional Fields: Enable “Save Browser Data”  
   - Connect input from SplitInBatches node (main output 0).

6. **Add BrowserAct Node to Get Details of Workflow Task**  
   - Type: BrowserAct Workflow  
   - Configure:  
     - Credentials: BrowserAct API credentials  
     - Operation: getTask  
     - Task ID: `={{ $json.id }}` (from previous node)  
     - Max Wait Time: 600 seconds  
     - Wait for Finish: true  
   - Connect input from “Run a workflow task”.

7. **Add LangChain Agent Node (AI Agent)**  
   - Type: LangChain Agent  
   - Configure:  
     - Text input: `={{ $json.output.string }}` (scraped comments)  
     - System Message: Special prompt instructing competitive intelligence analyst role  
     - Define output JSON structure with keys: Positive, Negative, Summary, Product, Recommendation  
   - Connect input from “Get details of a workflow task”.

8. **Add LangChain Structured Output Parser Node**  
   - Type: LangChain Output Parser Structured  
   - Configure: JSON schema example matching AI Agent output structure  
   - Connect input from AI Agent (ai_outputParser input).

9. **Add Google Sheets Node to Append Row**  
   - Type: Google Sheets (Append operation)  
   - Configure:  
     - Document ID and Sheet Name: Target spreadsheet for storing analysis  
     - Columns mapped: Product, Summary, Negative, Positive, Recommendation (from AI output)  
   - Connect input from AI Agent main output.

10. **Set Credentials**  
    - BrowserAct API account for BrowserAct nodes  
    - Google Gemini (PaLM) API account for LangChain AI Agent  
    - Google Sheets OAuth2 API account for reading and appending sheets  
    - Slack OAuth2 API account for Slack messaging

11. **Final Connections**  
    - Loop Over Items node connects to both Slack message and BrowserAct Run Task nodes (parallel branches)  
    - Append row node connects back to Loop Over Items for continuous processing if needed.

12. **Optional Enhancements**  
    - Replace Manual Trigger with Schedule Trigger for automated periodic monitoring  
    - Adjust “Total_review” parameter to change number of comments scraped  
    - Customize AI Agent prompt for different analytical insights  
    - Add error handling nodes for API failures or empty data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                       | Context or Link                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This template provides automated competitive intelligence by scraping and summarizing Product Hunt launch feedback with a specialized AI analyst.                                                                                                 | Sticky Note - Intro                                                                            |
| Setup instructions: Add credentials for BrowserAct, Google Gemini, and Google Sheets; use BrowserAct template “Product Hunt Launch Monitor”; customize product input parameters; trigger manually or schedule.                                    | Sticky Note - How to Use                                                                       |
| Need help resources with detailed video guides on BrowserAct API key, workflow setup, BrowserAct templates customization, and combined use with n8n for competitor analysis.                                                                        | Sticky Note - Need Help                                                                        |
| Data collection uses BrowserAct nodes to scrape all comments from Product Hunt launch pages, waiting for scraping to complete before handing off data to AI.                                                                                       | Sticky Note - Data Collection                                                                 |
| AI Agent node is configured with a system message to act as a competitive intelligence analyst, producing a structured JSON output summarizing positive, negative, and overall feedback.                                                              | Sticky Note - AI Analyst                                                                       |
| Google Sheets node appends analyzed data to a spreadsheet, creating a running log of competitor feedback. Slack node sends notification messages to keep the team informed.                                                                         | Sticky Notes - Reporting and Reporting1                                                       |
| For more information and community support, join the BrowserAct Discord: [https://discord.com/invite/UpnCKd7GaU](https://discord.com/invite/UpnCKd7GaU) or visit the BrowserAct Blog: [https://www.browseract.com/blog](https://www.browseract.com/blog) |                                                                                                |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly follows applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.