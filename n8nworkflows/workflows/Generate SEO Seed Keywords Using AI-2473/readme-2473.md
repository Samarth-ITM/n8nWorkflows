Generate SEO Seed Keywords Using AI

https://n8nworkflows.xyz/workflows/generate-seo-seed-keywords-using-ai-2473


# Generate SEO Seed Keywords Using AI

### 1. Workflow Overview

This workflow, titled **"Generate SEO Seed Keywords Using AI"**, is designed to assist SEO strategists and marketers in generating a focused list of seed keywords tailored to an Ideal Customer Profile (ICP). By leveraging AI language models, it produces 15-20 seed keywords that are relevant to the ICP’s needs, pain points, goals, and search behavior. These keywords form a foundational element of an SEO strategy, particularly for B2B SaaS companies.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the ICP details via manual input.
- **1.2 Data Aggregation:** Prepares and aggregates ICP data for AI processing.
- **1.3 AI Processing:** Uses an AI Agent (Anthropic Chat Model) to generate seed keywords based on the ICP.
- **1.4 Output Handling:** Parses AI output and routes it to an external database or storage.
- **1.5 Workflow Control & Information:** Provides user guidance, notes, and reminders embedded as sticky notes for setup and usage.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block receives the Ideal Customer Profile (ICP) details from the user manually and initiates the workflow execution.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Set Ideal Customer Profile (ICP)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Entry point for manual testing or running the workflow on demand.  
    - *Configuration:* Default manual trigger with no parameters.  
    - *Inputs:* None  
    - *Outputs:* Connects to "Set Ideal Customer Profile (ICP)"  
    - *Failure Modes:* None expected here.

  - **Set Ideal Customer Profile (ICP)**  
    - *Type:* Set Node  
    - *Role:* Collects user-defined ICP parameters as strings detailing product, pain points, goals, current solutions, and expertise level.  
    - *Configuration:* Contains placeholders to be replaced by users with their ICP data for the following fields:  
      - product  
      - pain points  
      - goals  
      - current solutions  
      - expertise level  
    - *Key Expressions:* Static string assignments with bold placeholders for user replacement.  
    - *Inputs:* Trigger from "When clicking ‘Test workflow’"  
    - *Outputs:* Passes structured ICP data downstream  
    - *Failure Modes:* If left unchanged, the placeholder text may confuse the AI or generate irrelevant keywords.

---

#### 1.2 Data Aggregation

- **Overview:** Aggregates all ICP data items into a single unified data structure to prepare for AI processing.
- **Nodes Involved:**  
  - Aggregate for AI node

- **Node Details:**

  - **Aggregate for AI node**  
    - *Type:* Aggregate  
    - *Role:* Combines all incoming ICP data items into one aggregated dataset.  
    - *Configuration:* Set to aggregate all item data into a single composite output.  
    - *Inputs:* Receives data from "Set Ideal Customer Profile (ICP)"  
    - *Outputs:* Feeds aggregated data into the AI Agent node  
    - *Failure Modes:* Data aggregation failure is unlikely but could occur if input schema changes or is empty.

---

#### 1.3 AI Processing

- **Overview:** Uses an AI agent powered by the Anthropic Chat Model to generate a list of SEO seed keywords based on the ICP data.
- **Nodes Involved:**  
  - Anthropic Chat Model  
  - AI Agent  
  - Split Out

- **Node Details:**

  - **Anthropic Chat Model**  
    - *Type:* Anthropic Chat Language Model Node (Langchain integration)  
    - *Role:* Provides AI language generation capabilities as the backend for the AI agent.  
    - *Configuration:* Default options, no custom parameters set.  
    - *Inputs:* Connected as the language model for the "AI Agent" node.  
    - *Outputs:* Provides raw AI completions to "AI Agent".  
    - *Version:* 1.2  
    - *Failure Modes:* API authentication failure, rate limiting, network errors, or model errors.

  - **AI Agent**  
    - *Type:* Langchain Agent Node  
    - *Role:* Implements complex AI prompt logic, including instructions and input integration, to generate seed keywords from ICP data.  
    - *Configuration:*  
      - System message defines the AI as an expert SEO strategist.  
      - Instructions include detailed rules for keyword generation (e.g., number of keywords, keyword format, search intent, specificity).  
      - Input is dynamically injected ICP data aggregated from prior nodes.  
      - Output is a JSON object with two keys: `{thoughts:}`, `{answer:}` containing an array of seed keywords.  
    - *Key Expressions:* Uses n8n expression syntax to inject ICP data from aggregated JSON.  
    - *Inputs:* Aggregated ICP data from "Aggregate for AI node"  
    - *Outputs:* Passes AI-generated response to "Split Out" node  
    - *Version:* 1.6  
    - *Failure Modes:* Expression evaluation errors, API call failures, malformed AI output, or empty responses.  
    - *Notes:* Critical node requiring valid AI API credentials (Anthropic/OpenAI).

  - **Split Out**  
    - *Type:* Split Out  
    - *Role:* Extracts the `output.answer` field from the AI Agent’s JSON response array to isolate the generated seed keywords.  
    - *Configuration:* Configured to split out the field `output.answer`.  
    - *Inputs:* From "AI Agent"  
    - *Outputs:* Routes keywords to the database connector node  
    - *Failure Modes:* If the AI output JSON structure changes or is invalid, this node may fail.

---

#### 1.4 Output Handling

- **Overview:** Routes the final list of seed keywords to a user-configured data sink such as a database, Google Sheet, or Airtable.
- **Nodes Involved:**  
  - Connect to your own database

- **Node Details:**

  - **Connect to your own database**  
    - *Type:* No Operation (NoOp) placeholder node  
    - *Role:* Indicates where users should connect their own database or data sink to store or further process the keyword list.  
    - *Configuration:* None, serves as a placeholder for custom integration.  
    - *Inputs:* Receives keyword array from "Split Out"  
    - *Outputs:* None  
    - *Failure Modes:* None internally; depends on user-implemented downstream integration.

---

#### 1.5 Workflow Control & Information

- **Overview:** Provides user instructions, notes on usage, costs, and setup reminders embedded as sticky notes.
- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note11  
  - Sticky Note14  
  - Sticky Note15  
  - Sticky Note16  
  - Sticky Note13 (disabled)  
  - Sticky Note12 (disabled)  
  - Sticky Note17 (disabled)

- **Node Details:**

  - **Sticky Note**  
    - *Content:* Overview of workflow purpose, outputs, and prerequisites.  
    - *Position:* Top-left area to provide initial context.

  - **Sticky Note11**  
    - *Content:* Cost estimate for running the workflow using Claude Sonnet 3.5 model: Approx. $0.02-0.05 per run.  
    - *Position:* Near AI processing nodes.

  - **Sticky Note14**  
    - *Content:* Reminder to connect to your own AI API credentials above.  
    - *Position:* Near AI nodes.

  - **Sticky Note15**  
    - *Content:* Reminder to connect to your own database or storage solution to output generated keywords.  
    - *Position:* Near the database connector.

  - **Sticky Note16**  
    - *Content:* Reminder to set the Ideal Customer Profile before running the workflow.  
    - *Position:* Near the ICP input nodes.

  - **Sticky Note13, Sticky Note12, Sticky Note17**  
    - *Status:* Disabled and not active in current workflow execution. They contain brief notes on draft keyword generation, Airtable data formatting, and adding data to the database.

---

### 3. Summary Table

| Node Name                      | Node Type                        | Functional Role                        | Input Node(s)                 | Output Node(s)               | Sticky Note                                         |
|--------------------------------|---------------------------------|-------------------------------------|------------------------------|------------------------------|-----------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger                  | Entry point to start workflow       | None                         | Set Ideal Customer Profile (ICP) |                                                     |
| Set Ideal Customer Profile (ICP) | Set                            | User input for ICP data              | When clicking ‘Test workflow’ | Aggregate for AI node          | **REQUIRED** Set your Ideal Customer Profile before proceeding |
| Aggregate for AI node          | Aggregate                       | Combines ICP data into one payload  | Set Ideal Customer Profile (ICP) | AI Agent                    |                                                     |
| Anthropic Chat Model           | Anthropic Chat Model (Langchain) | AI language model backend            | AI Agent (language model input) | AI Agent                    | **REQUIRED** Connect to your own AI API above       |
| AI Agent                      | Langchain Agent                 | Generates seed keywords using AI    | Aggregate for AI node          | Split Out                    | **REQUIRED** Connect to your own AI API above       |
| Split Out                     | Split Out                      | Extracts seed keywords from AI output | AI Agent                     | Connect to your own database |                                                     |
| Connect to your own database   | No Operation                   | Placeholder for database integration | Split Out                    | None                        | **REQUIRED** Connect to your own database / GSheet / Airtable base to output these |
| Sticky Note                   | Sticky Note                    | Workflow purpose and prerequisite notes | None                         | None                        | ## Generate SEO Seed Keywords Using AI<br>Pre-requisites: ICP, AI API account |
| Sticky Note11                 | Sticky Note                    | Cost estimate note                   | None                         | None                        | Costs to run Approx. $0.02-0.05 for a run using Claude Sonnet 3.5 |
| Sticky Note14                 | Sticky Note                    | AI API connection reminder          | None                         | None                        | **REQUIRED** Connect to your own AI API above       |
| Sticky Note15                 | Sticky Note                    | Database connection reminder        | None                         | None                        | **REQUIRED** Connect to your own database / GSheet / Airtable base to output these |
| Sticky Note16                 | Sticky Note                    | ICP setup reminder                  | None                         | None                        | **REQUIRED** Set your Ideal Customer Profile before proceeding |
| Sticky Note12 (disabled)      | Sticky Note                    | Disabled - Airtable data formatting note | None                         | None                        |                                                     |
| Sticky Note13 (disabled)      | Sticky Note                    | Disabled - Draft seed keyword generation note | None                         | None                        |                                                     |
| Sticky Note17 (disabled)      | Sticky Note                    | Disabled - Add data to database note | None                         | None                        |                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Serve as the workflow start trigger for manual execution.  
   - No special configuration required.

2. **Create Set Node: Set Ideal Customer Profile (ICP)**  
   - Type: Set  
   - Configure fields (all type string) with placeholder text for:  
     - product: "**Replace this with a string detailing your intended product (if you have one)**"  
     - pain points: "**Replace this with a string list of customer pain points**"  
     - goals: "**Replace this with a string list of your customers key goals/objectives**"  
     - current solutions: "**Replace this with a string detailing how your ideal customer currently solves their pain points**"  
     - expertise level: "**Replace this with a string detailing customer level of expertise**"  
   - Connect input from Manual Trigger.

3. **Create Aggregate Node: Aggregate for AI node**  
   - Type: Aggregate  
   - Set to aggregate all incoming items into one combined dataset.  
   - Connect input from Set Ideal Customer Profile node.

4. **Create Langchain Anthropic Chat Model Node**  
   - Type: Anthropic Chat Model (Langchain integration)  
   - Default options; no modifications necessary.  
   - No direct input connections; configured as language model for AI Agent node.

5. **Create Langchain Agent Node: AI Agent**  
   - Type: Langchain Agent  
   - Configure system message:  
     `"System: You are an expert SEO strategist tasked with generating 15-20 key head search terms (seed keywords) for a B2B SaaS company. Your goal is to create a comprehensive list of keywords that will attract and engage the ideal customer profile (ICP) described."`  
   - Configure prompt text with detailed instructions and rules for generating keywords, including JSON format for output with `{thoughts:}`, `{answer:}` keys.  
   - Use n8n expression to inject ICP data from the aggregated input:  
     `{{ $json.data[0].product }}` (and other ICP fields if necessary)  
   - Connect input from Aggregate for AI node.  
   - Set the Anthropic Chat Model node as the language model backend for this node.  
   - Ensure AI API credentials (Anthropic or OpenAI) are configured in n8n credentials.

6. **Create Split Out Node: Split Out**  
   - Type: Split Out  
   - Configure to extract the field: `output.answer` from the AI Agent output JSON.  
   - Connect input from AI Agent node.

7. **Create No Operation Node: Connect to your own database**  
   - Type: No Operation (NoOp)  
   - Placeholder node where users will connect their own database, Google Sheets, Airtable, or other data sinks.  
   - Connect input from Split Out node.

8. **Create Sticky Notes (Optional but Recommended):**  
   - Add sticky notes with the following contents positioned near relevant nodes:  
     - Workflow purpose and prerequisites.  
     - Reminder to set ICP before running.  
     - Reminder to connect AI API credentials.  
     - Reminder to connect the output to a database or storage.  
     - Cost estimate for running AI models.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                             |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------|
| Made by Simon @ automake.io                                                                                   | [https://automake.io](https://automake.io) |
| Workflow costs approx. $0.02-0.05 per run on Claude Sonnet 3.5 model                                          | Cost estimate sticky note in workflow      |
| Recommended AI API accounts: OpenAI or Anthropic                                                             | Setup prerequisites                         |
| Replace the NoOp node with your own database, Google Sheet, or Airtable integration to store generated keywords | User customization requirement              |

---

This documentation enables users and automation agents to fully understand, reproduce, and modify the "Generate SEO Seed Keywords Using AI" workflow while anticipating configuration needs and potential error points.