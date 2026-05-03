Restaurant Daily Insights with Gemini AI - Sales, Waste & Feedback Email Summary

https://n8nworkflows.xyz/workflows/restaurant-daily-insights-with-gemini-ai---sales--waste---feedback-email-summary-5991


# Restaurant Daily Insights with Gemini AI - Sales, Waste & Feedback Email Summary

### 1. Workflow Overview

This workflow automates the generation and delivery of a **daily performance report for a restaurant** by integrating data from three main sources: **Sales**, **Food Waste**, and **Customer Feedback**. Each data source is fetched from Google Sheets, normalized, and analyzed by specialized AI agents to extract actionable insights. These insights are then merged, refined into a comprehensive email summary via AI, and sent to designated recipients through Gmail.

The workflow is logically organized into the following functional blocks:

- **1.1 Data Collection & Normalization**: Fetch raw data daily from Google Sheets for sales, food waste, and customer feedback; normalize data into JSON payloads suitable for AI processing.
- **1.2 AI Data Analysis**: Separate AI agents process each data category to validate, compute metrics, analyze trends, and generate structured JSON insights with recommendations.
- **1.3 Data Merging & Email Preparation**: Merge all AI-generated insights into a unified dataset, prepare the input for a final AI-powered email summary generator.
- **1.4 Email Generation & Delivery**: Use an AI model to create a detailed plain-text email summary, clean and normalize the email content, then send it via Gmail to stakeholders.
- **1.5 Scheduling & Control**: A scheduler triggers the workflow daily at 22:00 to ensure timely report generation.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Collection & Normalization

**Overview:**  
This block retrieves daily raw data from Google Sheets for Sales, Food Waste, and Customer Feedback. It then normalizes this data into structured JSON objects, bundling all rows for AI consumption.

**Nodes Involved:**  
- Daily Report Scheduler  
- Fetch Daily Sales Data  
- Normalize Sales Records  
- Fetch Daily Food Waste Records  
- Normalize Waste Data  
- Fetch Customer Feedback  
- Normalize Feedback Entries  

**Node Details:**

- **Daily Report Scheduler**  
  - Type: Schedule Trigger  
  - Role: Triggers the workflow every day at 22:00  
  - Config: Interval set to trigger at hour 22  
  - Inputs: None (trigger node)  
  - Outputs: Triggers the three Google Sheets fetch nodes simultaneously  
  - Edge cases: Missed triggers if n8n is offline; time zone considerations  

- **Fetch Daily Sales Data**  
  - Type: Google Sheets  
  - Role: Fetches daily sales records from a specified sheet/tab  
  - Config: Uses a service account for authentication; document and sheet IDs set to relevant sales data  
  - Inputs: Trigger from scheduler  
  - Outputs: Raw sales rows, one per item  
  - Edge cases: Access denied, missing sheet, network errors  

- **Normalize Sales Records**  
  - Type: Code (JavaScript)  
  - Role: Bundles all fetched sales rows into a single JSON object under `data.rows`  
  - Key Expression: Collects all input items and maps `.json` to an array  
  - Inputs: Sales data rows  
  - Outputs: Single JSON item with all sales rows bundled  
  - Edge cases: Empty input array; malformed rows  

- **Fetch Daily Food Waste Records**  
  - Type: Google Sheets  
  - Role: Fetches daily food waste data  
  - Config: Similar to sales node, uses service account authentication  
  - Inputs: Trigger from scheduler  
  - Outputs: Raw waste data rows  
  - Edge cases: Same as sales fetch node  

- **Normalize Waste Data**  
  - Type: Code  
  - Role: Bundles all food waste rows into a single JSON object under `data.rows`  
  - Inputs/Outputs: As sales normalization node  

- **Fetch Customer Feedback**  
  - Type: Google Sheets  
  - Role: Retrieves daily customer feedback entries  
  - Config: Service account and sheet details set accordingly  
  - Inputs: Trigger from scheduler  
  - Outputs: Raw feedback rows  
  - Edge cases: Same as above  

- **Normalize Feedback Entries**  
  - Type: Code  
  - Role: Bundles all feedback rows into single JSON object for AI input  
  - Inputs/Outputs: Same pattern as other normalization nodes  

---

#### 2.2 AI Data Analysis

**Overview:**  
Each normalized dataset is sent to a dedicated AI agent that validates, analyzes, and generates structured insights and recommendations as JSON outputs.

**Nodes Involved:**  
- AI Sales Insights Generator  
- Format Sales AI Output  
- AI Waste Reduction Insights Generator  
- Format Waste AI Output  
- AI Feedback Summary Generator  
- Format Feedback AI Output  

**Node Details:**

- **AI Sales Insights Generator**  
  - Type: LangChain Agent (AI Agent)  
  - Role: Processes sales JSON data to validate, calculate metrics (e.g., profit margin), identify top/bottom performers, analyze trends (weather, peak hours), and recommend actions  
  - Configuration: Receives sales data JSON (`$json.data`); system prompt defines expected tasks and output JSON structure  
  - Inputs: Normalized sales data  
  - Outputs: Raw AI JSON response as string in `output` field  
  - Edge cases: AI timeout, invalid JSON output, incomplete data  

- **Format Sales AI Output**  
  - Type: Code  
  - Role: Cleans AI JSON output, strips markdown/code fences, parses JSON, normalizes numeric fields, ensures consistent structure  
  - Inputs: AI raw output string  
  - Outputs: Parsed and normalized JSON object representing sales insights  
  - Edge cases: JSON parse errors, missing fields, non-string outputs  

- **AI Waste Reduction Insights Generator**  
  - Type: LangChain Agent  
  - Role: Analyzes food waste data for validation, cost calculations, categorization, reason/action analysis, and recommendations  
  - Configuration: Similar to sales AI with detailed system prompt for food waste specifics  
  - Inputs: Normalized food waste data  
  - Outputs: Raw AI JSON response string  
  - Edge cases: Same as sales AI  

- **Format Waste AI Output**  
  - Type: Code  
  - Role: Parses and normalizes AI food waste JSON output; ensures numeric fields and arrays are correctly handled  
  - Inputs/Outputs: Same pattern as Format Sales AI Output  

- **AI Feedback Summary Generator**  
  - Type: LangChain Agent  
  - Role: Processes customer feedback JSON for validation, aggregate ratings, sentiment analysis, dish-level insights, themes, and recommendations  
  - Configuration: System prompt tailored to restaurant feedback analysis  
  - Inputs: Normalized feedback JSON  
  - Outputs: Raw AI JSON response string  
  - Edge cases: Same as other AI nodes  

- **Format Feedback AI Output**  
  - Type: Code  
  - Role: Cleans and parses AI feedback JSON output; normalizes numeric values and arrays of insights and recommendations  
  - Inputs/Outputs: Same pattern as other format nodes  

---

#### 2.3 Data Merging & Email Preparation

**Overview:**  
This block merges the three AI insight datasets into one combined item, prepares a unified JSON payload for the final AI email generator.

**Nodes Involved:**  
- Combine All Insights (Merge)  
- Wait for All Data Processing  
- Prepare Final Email Input  

**Node Details:**

- **Combine All Insights**  
  - Type: Merge  
  - Role: Merges three inputs (sales insights, waste insights, feedback insights) into one output stream  
  - Config: Number of inputs set to 3; merges items by index (one-to-one)  
  - Inputs: Formatted AI outputs for sales, waste, and feedback  
  - Outputs: Combined items array  
  - Edge cases: Missing inputs, timing issues  

- **Wait for All Data Processing**  
  - Type: Wait  
  - Role: Ensures that all three AI insights are available before proceeding  
  - Inputs: Output from merge node  
  - Outputs: Passes on combined data  
  - Edge cases: Timeout if any branch fails or delays  

- **Prepare Final Email Input**  
  - Type: Code  
  - Role: Bundles merged data into a single JSON object with `data.rows` key for final AI summary generation  
  - Inputs: Combined insights  
  - Outputs: Single JSON item with all insights bundled  
  - Edge cases: Empty input, malformed merges  

---

#### 2.4 Email Generation & Delivery

**Overview:**  
A final AI agent composes a professional, detailed plain-text email summary from the combined insights. The email content is cleaned and sent via Gmail.

**Nodes Involved:**  
- AI-Generated Daily Summary  
- Format Final Email Content  
- Email Final Report via Gmail  

**Node Details:**

- **AI-Generated Daily Summary**  
  - Type: LangChain Agent  
  - Role: Receives combined sales, waste, and feedback JSON; generates a structured plain-text email body including greeting, key metrics, detailed insights, cross-analysis, next-day suggestions, and sign-off  
  - Configuration: System message instructs formatting rules and content requirements; output is plain text only  
  - Inputs: Bundled combined insights JSON  
  - Outputs: Raw email text (may include fences or prefixes) in `output` field  
  - Edge cases: AI failures, incomplete output, formatting errors  

- **Format Final Email Content**  
  - Type: Code  
  - Role: Cleans AI email output from fences and extraneous prefixes; trims whitespace; outputs clean plain-text email body under `email_body`  
  - Inputs: Raw AI email output  
  - Outputs: Clean email text ready for sending  
  - Edge cases: Missing output, non-string data  

- **Email Final Report via Gmail**  
  - Type: Gmail node  
  - Role: Sends the clean plain-text email to specified recipient  
  - Config: OAuth2 credentials for Gmail; recipient email set to `abc@gmail.com`; subject "Next monday prediction"; emailType set to text  
  - Inputs: Clean email body text  
  - Outputs: Email send status  
  - Edge cases: Auth errors, quota limits, invalid recipient address  

---

### 3. Summary Table

| Node Name                      | Node Type                       | Functional Role                             | Input Node(s)                           | Output Node(s)                         | Sticky Note                                                                                                              |
|--------------------------------|--------------------------------|---------------------------------------------|---------------------------------------|--------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview              | Sticky Note                    | Describes overall workflow purpose          | —                                     | —                                    | ## Restaurant Daily Report Workflow 🍽️ This workflow automates the generation and delivery of a daily performance report for a restaurant. It collects data from three sources: **Sales**, **Food Waste**, and **Customer Feedback**, processes them using AI, merges the insights, generates an email summary, and sends it out. |
| Sales Data Analysis            | Sticky Note                    | Describes sales data analysis logic          | —                                     | —                                    | ## Sales Data Analysis 📈 1. Fetches daily sales records from Google Sheets. 2. Bundles raw sales rows into JSON for AI. 3. AI processes sales data with calculations and recommendations. 4. Normalizes AI output. |
| Food Waste Analysis            | Sticky Note                    | Describes food waste data analysis logic     | —                                     | —                                    | ## Food Waste Analysis 🗑️ 1. Retrieves daily food waste entries from Google Sheets. 2. Formats raw waste data for AI. 3. AI analyzes waste, calculates costs, and suggests prevention actions. 4. Normalizes AI output. |
| Customer Feedback Analysis     | Sticky Note                    | Describes customer feedback data analysis    | —                                     | —                                    | ## Customer Feedback Analysis 💬 1. Reads daily feedback from Google Sheets. 2. Bundles feedback into JSON for AI. 3. AI aggregates ratings, sentiment, themes, and provides recommendations. 4. Normalizes AI output. |
| Merge & Email Creation         | Sticky Note                    | Describes merging insights and email creation | —                                     | —                                    | ## Merge & Email Creation 📧 1. Combines processed sales, waste, and feedback data. 2. Code node structures merged data for email AI. 3. AI compiles comprehensive email summary. |
| Send Daily Report             | Sticky Note                    | Describes final email cleaning and sending   | —                                     | —                                    | ## Send Daily Report 🚀 1. Code node cleans AI-generated email body. 2. Sends summary email via Gmail. |
| Daily Report Scheduler         | Schedule Trigger               | Triggers workflow daily at 22:00              | —                                     | Fetch Daily Sales Data, Fetch Daily Food Waste Records, Fetch Customer Feedback |                                                                                                                          |
| Fetch Daily Sales Data         | Google Sheets                 | Fetches sales data from Google Sheets         | Daily Report Scheduler                | Normalize Sales Records              |                                                                                                                          |
| Normalize Sales Records        | Code                          | Bundles sales rows into single JSON payload  | Fetch Daily Sales Data                | AI Sales Insights Generator          |                                                                                                                          |
| AI Sales Insights Generator    | LangChain Agent               | AI analyzes sales data, validates, calculates, and recommends | Normalize Sales Records              | Format Sales AI Output               |                                                                                                                          |
| Format Sales AI Output         | Code                          | Parses and normalizes AI sales JSON output    | AI Sales Insights Generator           | Combine All Insights                 |                                                                                                                          |
| Fetch Daily Food Waste Records | Google Sheets                 | Fetches food waste data from Google Sheets    | Daily Report Scheduler                | Normalize Waste Data                 |                                                                                                                          |
| Normalize Waste Data           | Code                          | Bundles waste rows into single JSON payload   | Fetch Daily Food Waste Records        | AI Waste Reduction Insights Generator |                                                                                                                          |
| AI Waste Reduction Insights Generator | LangChain Agent       | AI analyzes waste data, validates, calculates, and recommends | Normalize Waste Data                 | Format Waste AI Output               |                                                                                                                          |
| Format Waste AI Output         | Code                          | Parses and normalizes AI waste JSON output    | AI Waste Reduction Insights Generator | Combine All Insights                 |                                                                                                                          |
| Fetch Customer Feedback        | Google Sheets                 | Fetches customer feedback data from Google Sheets | Daily Report Scheduler                | Normalize Feedback Entries          |                                                                                                                          |
| Normalize Feedback Entries     | Code                          | Bundles feedback rows into single JSON payload | Fetch Customer Feedback              | AI Feedback Summary Generator       |                                                                                                                          |
| AI Feedback Summary Generator  | LangChain Agent               | AI analyzes feedback data, aggregates, and recommends | Normalize Feedback Entries           | Format Feedback AI Output            |                                                                                                                          |
| Format Feedback AI Output      | Code                          | Parses and normalizes AI feedback JSON output | AI Feedback Summary Generator        | Combine All Insights                 |                                                                                                                          |
| Combine All Insights           | Merge                         | Merges sales, waste, and feedback AI outputs  | Format Sales AI Output, Format Waste AI Output, Format Feedback AI Output | Wait for All Data Processing        |                                                                                                                          |
| Wait for All Data Processing   | Wait                          | Waits for all merged insights to be ready      | Combine All Insights                 | Prepare Final Email Input            |                                                                                                                          |
| Prepare Final Email Input      | Code                          | Bundles merged insights into JSON for email AI | Wait for All Data Processing          | AI-Generated Daily Summary           |                                                                                                                          |
| AI-Generated Daily Summary     | LangChain Agent               | Generates plain-text email summary from combined data | Prepare Final Email Input            | Format Final Email Content           |                                                                                                                          |
| Format Final Email Content     | Code                          | Cleans AI-generated email text for sending    | AI-Generated Daily Summary            | Email Final Report via Gmail         |                                                                                                                          |
| Email Final Report via Gmail   | Gmail                         | Sends the final daily report email              | Format Final Email Content           | —                                    |                                                                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configuration: Set interval to daily at 22:00 (triggerAtHour=22)  
   - Name it: `Daily Report Scheduler`  

2. **Create Google Sheets Nodes to Fetch Data**  
   - Node 1: `Fetch Daily Sales Data`  
     - Authentication: Google Service Account  
     - Document ID and Sheet Name set to sales data sheet/tab  
   - Node 2: `Fetch Daily Food Waste Records`  
     - Same Google credentials  
     - Document ID and Sheet Name set to food waste sheet/tab  
   - Node 3: `Fetch Customer Feedback`  
     - Same Google credentials  
     - Document ID and Sheet Name set to feedback sheet/tab  
   - Connect trigger outputs to all three nodes  

3. **Create Code Nodes to Normalize Data**  
   - Node 4: `Normalize Sales Records`  
     - JavaScript: bundle all input items `.json` into `{ rows: [...] }` under `data` key  
     - Connect from `Fetch Daily Sales Data`  
   - Node 5: `Normalize Waste Data`  
     - Same JavaScript logic as sales normalization  
     - Connect from `Fetch Daily Food Waste Records`  
   - Node 6: `Normalize Feedback Entries`  
     - Same JavaScript logic  
     - Connect from `Fetch Customer Feedback`  

4. **Create AI Agent Nodes to Analyze Each Data Source**  
   - Node 7: `AI Sales Insights Generator`  
     - Type: LangChain Agent  
     - Input: `{{$json.data}}` from `Normalize Sales Records`  
     - System prompt: detailed instructions for sales analysis, validation, calculations, aggregation, recommendations  
   - Node 8: `AI Waste Reduction Insights Generator`  
     - Same LangChain Agent type  
     - Input: `{{$json.data}}` from `Normalize Waste Data`  
     - System prompt: detailed instructions for waste analysis and recommendations  
   - Node 9: `AI Feedback Summary Generator`  
     - Same LangChain Agent type  
     - Input: `{{$json.data}}` from `Normalize Feedback Entries`  
     - System prompt: detailed instructions for feedback analysis, sentiment, and recommendations  

5. **Create Code Nodes to Format AI Outputs**  
   - Node 10: `Format Sales AI Output`  
     - JavaScript to clean and parse AI JSON string output, normalize numbers, arrays  
     - Connect from `AI Sales Insights Generator`  
   - Node 11: `Format Waste AI Output`  
     - Similar code to parse and normalize waste AI output  
     - Connect from `AI Waste Reduction Insights Generator`  
   - Node 12: `Format Feedback AI Output`  
     - Similar code for feedback AI output  
     - Connect from `AI Feedback Summary Generator`  

6. **Merge Insights**  
   - Node 13: `Combine All Insights`  
     - Type: Merge  
     - Number of inputs: 3  
     - Connect outputs of Nodes 10, 11, and 12 as inputs  
   - Node 14: `Wait for All Data Processing`  
     - Type: Wait  
     - Connect from `Combine All Insights`  

7. **Prepare Final Email Input**  
   - Node 15: `Prepare Final Email Input`  
     - Code Node: Bundle merged insights into `{ data: { rows: [...] } }` format  
     - Connect from `Wait for All Data Processing`  

8. **Generate Email Summary via AI**  
   - Node 16: `AI-Generated Daily Summary`  
     - LangChain Agent node  
     - Input: `{{$json.data}}` from `Prepare Final Email Input`  
     - System prompt: instruct generation of detailed plain-text email with greeting, metrics, insights, suggestions, sign-off  
     - Set `executeOnce` to true  

9. **Clean Email Content**  
   - Node 17: `Format Final Email Content`  
     - Code node to strip markdown fences and unwanted prefixes, trim whitespace  
     - Connect from `AI-Generated Daily Summary`  

10. **Send Email via Gmail**  
    - Node 18: `Email Final Report via Gmail`  
    - Configure OAuth2 Gmail credentials  
    - To: `abc@gmail.com` (replace as needed)  
    - Subject: "Next monday prediction"  
    - Message body: `{{$json.email_body}}`  
    - Connect from `Format Final Email Content`  

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow uses Google Gemini (PaLM) AI models integrated via LangChain nodes for advanced natural language processing.        | Requires Google Palm API credentials configured in n8n                                         |
| AI agents are tailored with comprehensive system prompts to ensure structured JSON outputs and professional email formatting.    | Custom system messages embedded in AI nodes                                                    |
| The workflow relies on Google Sheets with structured tabs for Sales, Food Waste, and Customer Feedback data.                       | Ensure Google Sheets have correct schema and accessible via service account credentials         |
| The email sending uses Gmail OAuth2 credentials to securely send emails without exposing passwords.                               | Gmail OAuth2 must be properly configured with necessary scopes                                 |
| Refer to n8n documentation for best practices on error handling with Google Sheets and Gmail nodes to handle quota or auth errors.| https://docs.n8n.io/integrations/builtin/                                                      |

---

**Disclaimer:**  
The text and data processed within this workflow are fully compliant with applicable content policies and legal guidelines. All data handled is public or authorized for use in this context.