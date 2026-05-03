Generate Marketing Campaign ROI Reports with Google Sheets, GPT-4o and Email

https://n8nworkflows.xyz/workflows/generate-marketing-campaign-roi-reports-with-google-sheets--gpt-4o-and-email-7586


# Generate Marketing Campaign ROI Reports with Google Sheets, GPT-4o and Email

### 1. Workflow Overview

This workflow automates the generation of marketing campaign ROI reports by integrating Google Sheets, OpenAI's GPT-4o model, and Outlook email. It is designed for marketing teams or analysts who want to summarize campaign performance data and receive insightful, actionable email reports regularly or on-demand.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Manual trigger to start the workflow.
- **1.2 Data Acquisition**: Fetch raw marketing campaign data from Google Sheets.
- **1.3 Data Aggregation and Preparation**: Summarize campaign and channel metrics, then combine them into a textual summary.
- **1.4 AI Processing**: Use GPT-4o to analyze the summarized data, generate a natural language report in JSON format.
- **1.5 Output Handling**: (Not explicitly shown here but implied) The AI-generated report can be sent via email (described in sticky notes and workflow purpose).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Initiates the workflow manually.
- **Nodes Involved:**  
  - Start Workflow (Manual Trigger)
- **Node Details:**

  - **Start Workflow**  
    - Type: Manual Trigger  
    - Technical Role: Entry point node to start the workflow manually.  
    - Configuration: No parameters, triggers execution on user demand.  
    - Input: None  
    - Output: Triggers "Get Data" node.  
    - Edge Cases: No inputs; failure unlikely unless n8n server down.

#### 2.2 Data Acquisition

- **Overview:** Retrieves raw marketing campaign data from a specific Google Sheets document and sheet.
- **Nodes Involved:**  
  - Get Data
- **Node Details:**

  - **Get Data**  
    - Type: Google Sheets  
    - Technical Role: Reads the campaign data spreadsheet.  
    - Configuration:  
      - Document ID: Google Sheets spreadsheet with campaign data.  
      - Sheet Name: Specific sheet ID within the spreadsheet.  
      - OAuth2 Credentials: Authenticated via Google Sheets OAuth2 API.  
      - Data expected: Columns in the first row, data rows 2–100.  
    - Input: Trigger from "Start Workflow"  
    - Output: Raw rows of campaign data to "Sum Campaigns" and "Sum Channels".  
    - Edge Cases:  
      - OAuth token expiry or invalid credentials causing auth errors.  
      - Sheet or document not found errors.  
      - Empty or malformed data rows.  
    - Version: 4.7

#### 2.3 Data Aggregation and Preparation

- **Overview:** Aggregates the campaign and channel data separately (sum of spend, clicks, conversions), combines the results, and converts them into a formatted textual summary.
- **Nodes Involved:**  
  - Sum Campaigns  
  - Sum Channels  
  - Combine (for campaigns)  
  - Combine  (for channels)  
  - Merge Results  
  - Convert to Text
- **Node Details:**

  - **Sum Campaigns**  
    - Type: Summarize  
    - Role: Aggregates data by "Campaign" field — sums Spend ($), Clicks, and Conversions.  
    - Input: Raw data from "Get Data"  
    - Output: Summarized campaign data for aggregation.  
    - Edge Cases: Missing campaign names, non-numeric fields.  
    - Version: 1.1

  - **Sum Channels**  
    - Type: Summarize  
    - Role: Aggregates data by "Channel" field with the same metrics.  
    - Input: Raw data from "Get Data"  
    - Output: Summarized channel data.  
    - Edge Cases: Missing channel names, inconsistent data.  
    - Version: 1.1

  - **Combine** (campaigns)  
    - Type: Aggregate  
    - Role: Aggregates all summarized campaign data into one field "campaign_performance".  
    - Input: Output from "Sum Campaigns"  
    - Output: Aggregated campaign data array.  
    - Version: 1

  - **Combine** (channels)  
    - Type: Aggregate  
    - Role: Aggregates all summarized channel data into "channel_performance".  
    - Input: Output from "Sum Channels"  
    - Output: Aggregated channel data array.  
    - Version: 1

  - **Merge Results**  
    - Type: Merge  
    - Role: Combines campaign and channel aggregated data into one item by position.  
    - Input: Outputs from both "Combine" nodes  
    - Output: Single item containing both "campaign_performance" and "channel_performance".  
    - Version: 3.2

  - **Convert to Text**  
    - Type: Code (JavaScript)  
    - Role: Converts aggregated data arrays into a human-readable multi-line string summary with bullet points.  
    - Key code details:  
      - Iterates over campaign and channel arrays.  
      - Formats spend with 2 decimals, includes clicks and conversions.  
      - Outputs combined summary string in field "output".  
    - Input: Merged data from "Merge Results"  
    - Output: JSON object with field "output" containing the textual summary.  
    - Edge Cases: Empty or missing data arrays may produce empty summaries or errors if not handled.  
    - Version: 2

#### 2.4 AI Processing

- **Overview:** Uses GPT-4o AI model to analyze the textual summary and generate a business-friendly marketing performance report in JSON format structured with an "output" field.
- **Nodes Involved:**  
  - OpenAI Chat Model1  
  - Structured Output Parser1  
  - Analyze Marketing Data
- **Node Details:**

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat  
    - Role: Calls GPT-4o model to process input text.  
    - Configuration:  
      - Model: gpt-4o (OpenAI's advanced GPT-4 model)  
      - Credentials: OpenAI API key credential configured.  
    - Input: From "Analyze Marketing Data" (as language model input node)  
    - Output: AI-generated response for parsing.  
    - Edge Cases: API key invalid, rate limits, network timeouts.  
    - Version: 1.2

  - **Structured Output Parser1**  
    - Type: LangChain Structured Output Parser  
    - Role: Defines expected JSON schema for AI output to ensure structured parsing.  
    - Configuration: Example JSON with field "output" containing the marketing summary text and bullet points.  
    - Input: From "Analyze Marketing Data" (as output parser)  
    - Output: Parsed structured JSON.  
    - Edge Cases: AI returns malformed JSON or unexpected format leading to parse errors.  
    - Version: 1.3

  - **Analyze Marketing Data**  
    - Type: LangChain Agent  
    - Role: Coordinates data input and AI model call with system prompt defining task.  
    - Configuration:  
      - System message defines the AI as "Marketing Performance Assistant".  
      - Input text template includes `=Data: {{ $json.output }}` (the textual summary).  
      - Output expected in JSON with one field "output".  
      - Includes business-friendly language and emojis as instructed.  
    - Input: From "Convert to Text" node output.  
    - Output: AI-generated marketing report JSON.  
    - Edge Cases: AI generating irrelevant or incorrect information; prompt misconfiguration.  
    - Version: 2.2

---

### 3. Summary Table

| Node Name            | Node Type                        | Functional Role                          | Input Node(s)              | Output Node(s)               | Sticky Note                                                                                      |
|----------------------|---------------------------------|----------------------------------------|----------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Sticky Note          | Sticky Note                     | Documentation and instructions         |                            |                             | 📈 Campaign ROI Report with Generative AI + Email… detailed setup and contact info              |
| Start Workflow       | Manual Trigger                  | Entry point to start the workflow      |                            | Get Data                    |                                                                                                |
| Get Data             | Google Sheets                   | Retrieves campaign data from Google Sheets | Start Workflow             | Sum Campaigns, Sum Channels  |                                                                                                |
| Sum Campaigns        | Summarize                      | Aggregates campaign metrics by Campaign | Get Data                   | Combine                     |                                                                                                |
| Sum Channels         | Summarize                      | Aggregates channel metrics by Channel  | Get Data                   | Combine                     |                                                                                                |
| Combine              | Aggregate                      | Aggregates all campaign summary data   | Sum Campaigns              | Merge Results               |                                                                                                |
| Combine              | Aggregate                      | Aggregates all channel summary data    | Sum Channels               | Merge Results               |                                                                                                |
| Merge Results        | Merge                          | Combines campaign and channel data     | Combine (campaign), Combine (channel) | Convert to Text         |                                                                                                |
| Convert to Text      | Code (JavaScript)              | Converts aggregated data to text summary | Merge Results              | Analyze Marketing Data      |                                                                                                |
| Analyze Marketing Data | LangChain Agent                | AI interprets and summarizes marketing data | Convert to Text            |                             |                                                                                                |
| OpenAI Chat Model1   | LangChain OpenAI Chat          | Calls GPT-4o model for analysis         | Analyze Marketing Data (lm input) | Analyze Marketing Data (lm output) |                                                                                                |
| Structured Output Parser1 | LangChain Structured Output Parser | Parses AI output JSON                    | Analyze Marketing Data (output parser) | Analyze Marketing Data (lm output) |                                                                                                |
| Sticky Note1         | Sticky Note                    | Labels data aggregation block          |                            |                             | Aggregate and Combine Data                                                                     |
| Sticky Note10        | Sticky Note                    | Instructions for Google Sheets setup   |                            |                             | 2. Prepare Your Google Sheet… sample data format and OAuth2 login instructions                 |
| Sticky Note2         | Sticky Note                    | Describes AI agent and daily email use |                            |                             | AI Agent analyzes data and sends daily email                                                  |
| Sticky Note9         | Sticky Note                    | OpenAI connection setup instructions   |                            |                             | 1. Set Up OpenAI Connection… links to API key and billing setup                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger**  
   - Add node: Manual Trigger (Start Workflow)  
   - No parameters needed.

2. **Configure Google Sheets Node**  
   - Add node: Google Sheets  
   - Operation: Read rows from a sheet.  
   - Document ID: Use your Google Sheets spreadsheet ID.  
   - Sheet Name: Use the sheet ID or name containing campaign data.  
   - Credentials: Create Google Sheets OAuth2 API credentials and authenticate with your Google account.  
   - Data Format: Ensure your sheet has column headers in row 1, data from rows 2 to 100.  
   - Connect Manual Trigger output to Google Sheets input.

3. **Add "Sum Campaigns" Node**  
   - Node type: Summarize  
   - Group by field: "Campaign"  
   - Summarize fields: Sum of "Spend ($)", "Clicks", "Conversions"  
   - Connect Google Sheets output to this node.

4. **Add "Sum Channels" Node**  
   - Node type: Summarize  
   - Group by field: "Channel"  
   - Summarize fields: Sum of "Spend ($)", "Clicks", "Conversions"  
   - Connect Google Sheets output to this node.

5. **Add "Combine" Node for Campaign Summary**  
   - Node type: Aggregate  
   - Aggregate all item data into a field named "campaign_performance".  
   - Connect "Sum Campaigns" output here.

6. **Add "Combine" Node for Channel Summary**  
   - Node type: Aggregate  
   - Aggregate all item data into a field named "channel_performance".  
   - Connect "Sum Channels" output here.

7. **Add "Merge" Node**  
   - Mode: Combine  
   - Combine by position (merge items from both inputs side by side).  
   - Connect both "Combine" nodes outputs (campaign and channel aggregates) as inputs.

8. **Add "Code" Node (JavaScript)**  
   - Purpose: Convert merged data arrays into a formatted text summary.  
   - Paste the provided JS code that formats campaign and channel summaries with bullet points and monetary formatting.  
   - Input: Connect output of "Merge" node.  
   - Output: JSON with field "output" (text summary).

9. **Set up OpenAI Credentials**  
   - In n8n, add OpenAI API credentials using your API key from https://platform.openai.com/api-keys.  
   - Ensure your account has billing set up.

10. **Add LangChain Agent Node ("Analyze Marketing Data")**  
    - Type: LangChain Agent  
    - Configure system prompt as per workflow with role "Marketing Performance Assistant".  
    - Input text set as `=Data: {{ $json.output }}` to feed previous node's text summary.  
    - Define output parser expecting JSON with one field "output" containing the marketing narrative.  
    - Connect "Convert to Text" node output here.

11. **Add LangChain OpenAI Chat Model Node**  
    - Model: gpt-4o  
    - Connect as language model input node for LangChain Agent node.

12. **Add LangChain Structured Output Parser Node**  
    - Provide example JSON schema for expected AI output.  
    - Connect as output parser node for LangChain Agent node.

13. **Connect nodes as specified:**  
    - Manual Trigger → Google Sheets  
    - Google Sheets → Sum Campaigns & Sum Channels  
    - Sum Campaigns → Combine (campaign)  
    - Sum Channels → Combine (channel)  
    - Both Combine nodes → Merge  
    - Merge → Code (Convert to Text)  
    - Code → LangChain Agent (Analyze Marketing Data)  
    - LangChain Agent connects internally to OpenAI Chat Model and Structured Output Parser nodes.

14. **Test the workflow:**  
    - Run manual trigger.  
    - Verify data fetch, aggregation, AI summary generation.  
    - Handle any errors such as API limits, parsing errors, or data inconsistencies.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Sample marketing data sheet format with column headers and example data rows for easy integration into Google Sheets.              | [Sample Marketing Sheet](https://docs.google.com/spreadsheets/d/1UDWt0-Z9fHqwnSNfU3vvhSoYCFG6EG3E-ZewJC_CLq4) |
| Instructions for setting up Google Sheets OAuth2 credentials in n8n for secure access to campaign data.                           | Sticky Note10 content in the workflow                                                                       |
| OpenAI API key creation and billing setup instructions to ensure uninterrupted AI service access.                                  | [OpenAI API Keys](https://platform.openai.com/api-keys), [OpenAI Billing](https://platform.openai.com/settings/organization/billing/overview) |
| Contact info for support or questions about the workflow provided by the author.                                                   | Email: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/                   |
| The AI prompt is designed to produce a professional marketing report with use of emojis and business-friendly language.           | See "Analyze Marketing Data" node system prompt                                                             |

---

This document fully covers all nodes, their roles, configurations, and dependencies, allowing a user or automation agent to understand, reproduce, or adapt the workflow confidently.  
Disclaimer: The text is derived exclusively from an n8n automated workflow, adhering to all applicable content policies and containing only legal and public data.