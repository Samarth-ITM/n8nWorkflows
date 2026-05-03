Generate Personalized Startup Ideas with Claude Analysis and Gmail Reports

https://n8nworkflows.xyz/workflows/generate-personalized-startup-ideas-with-claude-analysis-and-gmail-reports-7480


# Generate Personalized Startup Ideas with Claude Analysis and Gmail Reports

### 1. Workflow Overview

This workflow is designed to generate personalized tech startup ideas specifically tailored for developers and entrepreneurs. It collects user input on technical skills and preferences, leverages advanced AI agents to generate, critique, and analyze startup concepts, and finally delivers a comprehensive, actionable report via email.

Logical blocks:

- **1.1 Input Reception:** Collects developer profile data through either a public form, scheduled daily runs, or manual trigger.
- **1.2 Data Preparation:** Normalizes and structures raw input data for AI consumption.
- **1.3 AI Analysis Pipeline:** Three sequential AI agents (Idea Generator, Idea Critic, Sentiment Analysis) generate a startup idea, critically evaluate it, and provide a sentiment-based recommendation.
- **1.4 Data Aggregation & Decision Logic:** Merges AI outputs, computes composite scores, and determines recommendations and routing flags.
- **1.5 Email Preparation & Delivery:** Formats the aggregated insights into a detailed HTML/email report and sends it to the developer via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block gathers the developer’s profile and preferences through multiple trigger options for flexibility: a web form, scheduled daily runs, or manual execution.

**Nodes Involved:**  
- Form (Form Trigger)  
- Scheduled trigger once a day (Schedule Trigger)  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- My Information (Set node with static personal data)

**Node Details:**

- **Form (Form Trigger)**  
  - *Type:* `n8n-nodes-base.formTrigger`  
  - *Role:* Receives input from external users via a web form.  
  - *Configuration:* Customized form with fields for name, email, programming languages, tech stack, skills, target market, time commitment, experience level, and budget.  
  - *Input:* User form submissions.  
  - *Output:* Raw form data JSON.  
  - *Edge Cases:* Missing required fields, malformed inputs, or unsubmitted forms may cause downstream errors.

- **Scheduled trigger once a day**  
  - *Type:* `n8n-nodes-base.scheduleTrigger`  
  - *Role:* Automatically triggers the workflow daily at 9 AM.  
  - *Configuration:* Cron-based schedule for daily execution.  
  - *Input:* None externally; triggers internal data processing.  
  - *Output:* Initiates workflow using static “My Information” data.  
  - *Edge Cases:* Timezone adjustments, missed runs due to downtime.

- **When clicking ‘Execute workflow’ (Manual Trigger)**  
  - *Type:* `n8n-nodes-base.manualTrigger`  
  - *Role:* Allows on-demand execution for testing or immediate runs.  
  - *Input:* None externally; manual user action.  
  - *Output:* Starts workflow using “My Information” data.  
  - *Edge Cases:* None significant; manual control.

- **My Information (Set node)**  
  - *Type:* `n8n-nodes-base.set`  
  - *Role:* Contains static developer profile data used for scheduled and manual triggers.  
  - *Configuration:* Fields for all personal and technical attributes, initially empty for user to fill.  
  - *Output:* Structured JSON with developer profile.  
  - *Edge Cases:* If left empty or outdated, generated ideas may be irrelevant.

---

#### 2.2 Data Preparation

**Overview:**  
Transforms raw input from either the form or the static personal data node into a normalized JSON object representing the developer profile for AI agents.

**Nodes Involved:**  
- Prepare Data (Set node)

**Node Details:**

- **Prepare Data**  
  - *Type:* `n8n-nodes-base.set`  
  - *Role:* Normalizes and consolidates input data into a consistent developer profile object.  
  - *Configuration:* Extracts fields like name, email, programming languages, tech stack, skills, target market, time commitment, experience level, budget, and current timestamp.  
  - *Input:* Form data or “My Information” node output.  
  - *Output:* Clean JSON object `developer_profile` used by AI agents.  
  - *Edge Cases:* Missing input data fields may cause incomplete profiles; timestamp ensures freshness.

---

#### 2.3 AI Analysis Pipeline

**Overview:**  
A three-stage AI pipeline sequentially generates a startup idea, critiques it critically, and performs sentiment analysis on the critique to produce a balanced final recommendation.

**Nodes Involved:**  
- Idea Generator Agent  
  - Idea Generator Model (Langchain Anthropic Claude 4)  
  - Idea Generator Parser (Output Parser Structured)  
- Idea Critic Agent  
  - Idea Critic Model (Langchain Anthropic Claude 4)  
  - Idea Critic Parser (Output Parser Structured)  
- Sentiment Analysis  
  - Sentiment Analysis Model (Langchain Anthropic Claude 4)  
  - Sentiment Analysis Parser (Output Parser Structured)  
- Merge Analysis (Merge node combining outputs)

**Node Details:**

- **Idea Generator Model**  
  - *Type:* `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  - *Role:* Uses Claude 4 with temperature 0.7 for creative generation of startup ideas.  
  - *Prompt:* Detailed instructions specifying startup components (core concept, problem, market, business model, innovation score, etc.) formatted as JSON.  
  - *Credentials:* Anthropic API required.  
  - *Edge Cases:* API rate limits, malformed output handled by parser.

- **Idea Generator Parser**  
  - *Type:* `@n8n/n8n-nodes-langchain.outputParserStructured`  
  - *Role:* Validates and parses the JSON output from the Idea Generator model.  
  - *Schema:* Matches prompt output structure to ensure consistent data.  
  - *Edge Cases:* Parsing failures if output deviates from JSON schema.

- **Idea Generator Agent**  
  - *Type:* `@n8n/n8n-nodes-langchain.agent`  
  - *Role:* Wraps model and parser to produce structured startup ideas.  
  - *Input:* Prepared developer profile data.  
  - *Output:* Structured idea JSON.

- **Idea Critic Model**  
  - *Type:* `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  - *Role:* Uses Claude 4 with temperature 0.3 for analytical critique.  
  - *Prompt:* Analyzes the startup idea for market validation, competition, execution challenges, business model, risks, and improvement recommendations, outputting a detailed JSON analysis.  
  - *Credentials:* Anthropic API required.  
  - *Edge Cases:* May produce overly negative or vague critiques; parser enforces structure.

- **Idea Critic Parser**  
  - *Type:* `@n8n/n8n-nodes-langchain.outputParserStructured`  
  - *Role:* Parses the critical analysis JSON to ensure data integrity.  
  - *Edge Cases:* Parsing errors handled gracefully.

- **Idea Critic Agent**  
  - *Type:* `@n8n/n8n-nodes-langchain.agent`  
  - *Role:* Combines model and parser for reliable critique output.  
  - *Input:* Idea Generator output.  
  - *Output:* Structured critique JSON.

- **Sentiment Analysis Model**  
  - *Type:* `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  - *Role:* Uses Claude 4 with temperature 0.2 for precise sentiment and psychological analysis of the critique.  
  - *Prompt:* Evaluates tone, bias, critique quality, psychological factors, meta-analysis, and synthesizes a final recommendation.  
  - *Credentials:* Anthropic API required.  
  - *Edge Cases:* May be limited if critique data is incomplete.

- **Sentiment Analysis Parser**  
  - *Type:* `@n8n/n8n-nodes-langchain.outputParserStructured`  
  - *Role:* Parses sentiment analysis output JSON.  
  - *Edge Cases:* Parsing errors mitigated.

- **Sentiment Analysis Agent**  
  - *Type:* `@n8n/n8n-nodes-langchain.agent`  
  - *Role:* Combines model and parser for final sentiment evaluation.  
  - *Input:* Idea Critic output and original idea data.  
  - *Output:* Sentiment and recommendation JSON.

- **Merge Analysis (Merge node)**  
  - *Type:* `n8n-nodes-base.merge`  
  - *Role:* Combines outputs from Idea Generator Agent, Idea Critic Agent, and Sentiment Analysis into a single dataset for aggregation.  
  - *Input:* Three AI agent outputs.  
  - *Output:* Unified JSON for processing.

---

#### 2.4 Data Aggregation & Decision Logic

**Overview:**  
Processes the merged AI outputs to compute composite innovation/viability/success scores, determine routing flags, and prepare a summary for email reporting.

**Nodes Involved:**  
- Data Aggregation (Code node)

**Node Details:**

- **Data Aggregation**  
  - *Type:* `n8n-nodes-base.code`  
  - *Role:* Parses outputs from all AI agents, calculates composite scores, assesses recommendation priority, and compiles final structured data.  
  - *Key Logic:*  
    - Composite score = average of innovation score, viability score, and success probability (scaled).  
    - Routing flags include whether to email, save data, human review necessity, and report generation based on score thresholds.  
  - *Input:* Merged AI outputs.  
  - *Output:* Aggregated JSON including metadata, raw AI outputs, processed insights, executive summary, and flags.  
  - *Edge Cases:* Missing or incomplete AI data handled with fallback defaults.

---

#### 2.5 Email Preparation & Delivery

**Overview:**  
Constructs a formatted email report from aggregated analysis data and sends it via Gmail to the developer.

**Nodes Involved:**  
- Email Creator (Code node)  
- Send Startup Idea via Email (Gmail node)

**Node Details:**

- **Email Creator**  
  - *Type:* `n8n-nodes-base.code`  
  - *Role:* Converts aggregated JSON data into a human-readable HTML and plain-text email body, including scores, risks, opportunities, verdict, and next steps.  
  - *Input:* Aggregated data from Data Aggregation node.  
  - *Output:* JSON containing email subject, HTML body, and text body.  
  - *Edge Cases:* If some data arrays (risks, opportunities) are empty, list items render empty.

- **Send Startup Idea via Email (Gmail)**  
  - *Type:* `n8n-nodes-base.gmail`  
  - *Role:* Sends the email report using Gmail OAuth2 credentials.  
  - *Configuration:* Recipient email must be set; subject and message dynamically populated.  
  - *Input:* Email content JSON from Email Creator.  
  - *Output:* Email delivery status.  
  - *Edge Cases:* OAuth token expiration, SMTP errors, invalid recipient address.

---

### 3. Summary Table

| Node Name                  | Node Type                                 | Functional Role                              | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                                  |
|----------------------------|-------------------------------------------|----------------------------------------------|------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Form                       | n8n-nodes-base.formTrigger                 | Collects developer data via web form         | —                            | Prepare Data                   | 🎯 Input Triggers: Three ways to run this workflow; form trigger is for external users                        |
| Scheduled trigger once a day| n8n-nodes-base.scheduleTrigger             | Daily scheduled run trigger                   | —                            | My Information                 | 🎯 Input Triggers: Daily runs use static personal data                                                      |
| When clicking ‘Execute workflow’ | n8n-nodes-base.manualTrigger            | On-demand manual trigger                       | —                            | My Information                 | 🎯 Input Triggers: For testing and manual runs                                                              |
| My Information             | n8n-nodes-base.set                         | Static personal/developer profile data        | Scheduled trigger, Manual     | Prepare Data                   | 🔧 Data Preparation Zone: Fill all fields; update periodically                                               |
| Prepare Data               | n8n-nodes-base.set                         | Normalize input to structured developer profile | Form, My Information          | Idea Generator Agent           | 🔧 Data Preparation Zone: Converts raw input into structured data for AI                                     |
| Idea Generator Model       | @n8n/n8n-nodes-langchain.lmChatAnthropic  | Creative AI model generating startup ideas    | Idea Generator Agent          | Idea Generator Agent           | 🤖 AI Analysis Pipeline: Creative generation with temperature 0.7                                           |
| Idea Generator Parser      | @n8n/n8n-nodes-langchain.outputParserStructured | Validates and parses AI JSON output            | Idea Generator Model          | Idea Generator Agent           | 🤖 AI Analysis Pipeline: Ensures structured JSON output                                                     |
| Idea Generator Agent       | @n8n/n8n-nodes-langchain.agent             | Encapsulates model + parser to produce ideas  | Prepare Data                  | Idea Critic Agent, Merge Analysis | 🤖 AI Analysis Pipeline: First AI agent generating startup ideas                                             |
| Idea Critic Model          | @n8n/n8n-nodes-langchain.lmChatAnthropic  | Analytical AI model critiquing startup idea   | Idea Critic Agent             | Idea Critic Agent              | 🤖 AI Analysis Pipeline: Critical market analysis with temperature 0.3                                      |
| Idea Critic Parser         | @n8n/n8n-nodes-langchain.outputParserStructured | Parses critical analysis JSON                   | Idea Critic Model             | Idea Critic Agent              | 🤖 AI Analysis Pipeline: Validates critique output                                                          |
| Idea Critic Agent          | @n8n/n8n-nodes-langchain.agent             | Agent wrapping critique model + parser         | Idea Generator Agent          | Sentiment Analysis, Merge Analysis | 🤖 AI Analysis Pipeline: Second AI agent providing critical market insights                                  |
| Sentiment Analysis Model   | @n8n/n8n-nodes-langchain.lmChatAnthropic  | AI model analyzing sentiment and recommendations | Sentiment Analysis            | Sentiment Analysis            | 🤖 AI Analysis Pipeline: Final sentiment and decision analysis with temperature 0.2                          |
| Sentiment Analysis Parser  | @n8n/n8n-nodes-langchain.outputParserStructured | Parses sentiment analysis JSON                   | Sentiment Analysis Model      | Sentiment Analysis            | 🤖 AI Analysis Pipeline: Ensures structured sentiment output                                                |
| Sentiment Analysis         | @n8n/n8n-nodes-langchain.agent             | Combines model + parser for sentiment analysis | Idea Critic Agent             | Merge Analysis                | 🤖 AI Analysis Pipeline: Third AI agent producing balanced recommendations                                   |
| Merge Analysis             | n8n-nodes-base.merge                       | Merges outputs of the three AI agents          | Idea Generator Agent, Idea Critic Agent, Sentiment Analysis | Data Aggregation             | 📊 Output Processing: Combines all AI outputs for aggregation                                               |
| Data Aggregation           | n8n-nodes-base.code                        | Aggregates AI data, calculates scores, flags  | Merge Analysis               | Email Creator                 | 📊 Output Processing: Calculates composite scores and routing flags                                         |
| Email Creator              | n8n-nodes-base.code                        | Formats aggregated data into HTML/text email  | Data Aggregation             | Send Startup Idea via Email   | 📊 Output Processing: Creates detailed email report                                                         |
| Send Startup Idea via Email| n8n-nodes-base.gmail                       | Sends email report via Gmail                    | Email Creator                | —                            | 📊 Output Processing: Requires Gmail OAuth2 credentials; sends startup analysis email                        |
| 📋 Workflow Overview      | n8n-nodes-base.stickyNote                  | Documentation sticky note                       | —                            | —                            | Explains workflow purpose and logical blocks                                                                |
| 📋 Input Sources          | n8n-nodes-base.stickyNote                  | Documentation sticky note                       | —                            | —                            | Details input trigger options                                                                                |
| 📋 Data Preparation       | n8n-nodes-base.stickyNote                  | Documentation sticky note                       | —                            | —                            | Explains data normalization step                                                                             |
| 📋 AI Analysis Pipeline   | n8n-nodes-base.stickyNote                  | Documentation sticky note                       | —                            | —                            | Describes the 3 AI agent pipeline                                                                             |
| 📋 Data Processing & Output| n8n-nodes-base.stickyNote                  | Documentation sticky note                       | —                            | —                            | Describes merging, aggregation, scoring, email prep                                                         |
| 📋 Setup Checklist        | n8n-nodes-base.stickyNote                  | Documentation sticky note                       | —                            | —                            | Lists critical setup steps                                                                                    |
| 📋 Technical Details      | n8n-nodes-base.stickyNote                  | Documentation sticky note                       | —                            | —                            | Notes on parsing, temperatures, data flow, common issues                                                     |
| 📋 Future Enhancements    | n8n-nodes-base.stickyNote                  | Documentation sticky note                       | —                            | —                            | Suggests future workflow improvements                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Input Nodes:**  
   - Add a **Form Trigger** node:  
     - Configure with fields: Name (text, required), Email (email, required), Programming Languages (textarea, required), Tech Stack & Frameworks (textarea, required), Specialized Skills (multi-select dropdown with predefined options), Target Market (dropdown), Time Commitment (dropdown, required), Experience Level (dropdown, required), Budget Range (dropdown, required).  
     - Set webhook path and enable.

   - Add a **Scheduled Trigger** node:  
     - Set to run daily at 9:00 AM.

   - Add a **Manual Trigger** node for testing.

2. **Create Static Data Node:**  
   - Add a **Set** node named “My Information”.  
   - Add fields for all developer profile attributes (matching form) with empty default values.  
   - This node will be the source for scheduled and manual triggers.

3. **Add Data Preparation Node:**  
   - Add a **Set** node named “Prepare Data”.  
   - Map all input fields from Form or “My Information” into a single JSON object property named `developer_profile`.  
   - Include a timestamp field with current ISO date.

4. **Configure AI Agents:**

   - For **Idea Generator Agent:**  
     - Add a **Langchain Model Anthropic** node named “Idea Generator Model”.  
     - Use `claude-sonnet-4-20250514` model with temperature 0.7.  
     - Connect to an **Output Parser Structured** node with the JSON schema matching the startup idea output.  
     - Wrap these in a Langchain **Agent** node “Idea Generator Agent” with a detailed prompt specifying startup idea generation instructions and JSON formatting.  
     - Input is `developer_profile` from “Prepare Data”.

   - For **Idea Critic Agent:**  
     - Add a **Langchain Model Anthropic** node named “Idea Critic Model”.  
     - Use `claude-sonnet-4-20250514` with temperature 0.3.  
     - Connect to an **Output Parser Structured** node with JSON schema for critical analysis.  
     - Wrap in an **Agent** node “Idea Critic Agent”.  
     - Input is output from “Idea Generator Agent” (startup idea JSON).

   - For **Sentiment Analysis Agent:**  
     - Add a **Langchain Model Anthropic** node named “Sentiment Analysis Model”.  
     - Use `claude-sonnet-4-20250514` with temperature 0.2.  
     - Connect to an **Output Parser Structured** node with JSON schema matching sentiment analysis output.  
     - Wrap in an **Agent** node “Sentiment Analysis”.  
     - Input is the critique JSON plus original idea JSON.

5. **Merge AI Outputs:**  
   - Add a **Merge** node named “Merge Analysis”.  
   - Configure to accept 3 inputs and combine the outputs of the three AI agents.

6. **Add Data Aggregation Node:**  
   - Add a **Code** node named “Data Aggregation”.  
   - Implement logic to extract innovation, viability, success scores; calculate composite score; set routing flags; compile executive summary.  
   - Input from “Merge Analysis”.

7. **Email Preparation:**  
   - Add a **Code** node named “Email Creator”.  
   - Format aggregated data into a well-structured HTML and plain text email.  
   - Input from “Data Aggregation”.

8. **Email Sending:**  
   - Add a **Gmail** node named “Send Startup Idea via Email”.  
   - Set recipient email address.  
   - Use Gmail OAuth2 credentials.  
   - Input message and subject from “Email Creator” output.

9. **Connect Triggers to Data Preparation:**  
   - Connect Form, Scheduled Trigger, and Manual Trigger nodes to “My Information” or directly to “Prepare Data” depending on source.  
   - “My Information” connects to “Prepare Data”.

10. **Connect AI pipeline:**  
    - “Prepare Data” → “Idea Generator Agent” → “Idea Critic Agent” → “Sentiment Analysis” → “Merge Analysis” → “Data Aggregation” → “Email Creator” → “Send Startup Idea via Email”.

11. **Credential Setup:**  
    - Add Anthropic API credentials for all Langchain model nodes.  
    - Add Gmail OAuth2 credentials for email node.

12. **Testing:**  
    - Run manual trigger to test end-to-end.  
    - Verify AI JSON outputs parse correctly.  
    - Verify email sending functionality.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow generates personalized startup ideas integrating AI creativity, critique, and sentiment analysis.     | Overview and purpose of the workflow.                                                          |
| Anthropic Claude 4 models are used with different temperature settings for creativity (0.7), critique (0.3), and sentiment (0.2). | Model configuration details.                                                                   |
| Gmail OAuth2 credentials are required for email sending.                                                      | Credential setup instruction.                                                                  |
| Form trigger webhook URL can be shared publicly to collect external inputs.                                   | Input reception details.                                                                        |
| Output parsers ensure robust JSON validation to prevent workflow failures due to malformed AI outputs.        | Technical detail on AI output reliability.                                                    |
| Composite score calculation balances innovation, market viability, and success probability for routing logic. | Decision logic in Data Aggregation node.                                                      |
| Consider future enhancements like database storage, Slack integration, analytics dashboard, and conditional email sending. | Ideas for scaling and improving workflow.                                                     |
| Testing tips: run manual trigger first, verify email delivery, and check logs for any parsing or API errors.  | Practical advice for deployment.                                                               |

---

This structured documentation enables users and automation agents to fully comprehend, replicate, and maintain the "Generate Personalized Startup Ideas with Claude Analysis and Gmail Reports" workflow efficiently and reliably.