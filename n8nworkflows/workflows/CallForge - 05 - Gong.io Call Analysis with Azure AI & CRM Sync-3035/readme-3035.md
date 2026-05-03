CallForge - 05 - Gong.io Call Analysis with Azure AI & CRM Sync

https://n8nworkflows.xyz/workflows/callforge---05---gong-io-call-analysis-with-azure-ai---crm-sync-3035


# CallForge - 05 - Gong.io Call Analysis with Azure AI & CRM Sync

### 1. Workflow Overview

This workflow, **CallForge - 05 - Gong.io Call Analysis with Azure AI & CRM Sync**, automates the extraction and processing of structured insights from Gong.io sales call transcripts using Azure AI models. It targets sales, marketing, and product teams by providing tailored AI-driven analyses that identify use cases, objections, competitor mentions, customer pain points, and next steps. The workflow also synchronizes processed insights with CRM and collaboration tools like Notion, Salesforce, and Slack, ensuring actionable data is accessible to relevant teams.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Prompt Preparation**: Receives Gong call data, normalizes and standardizes the transcript, and creates a unified user prompt for AI agents.
- **1.2 AI Processing Agents**: Three specialized AI agents analyze the transcript for Sales, Marketing, and Product insights using Azure OpenAI GPT-4o-mini models.
- **1.3 Structured Output Parsing & Data Recall**: Parses AI outputs into structured JSON and prepares data for downstream processing.
- **1.4 Department-Specific Data Processing**: Executes sub-workflows to process and store insights for Sales, Marketing, and Product teams.
- **1.5 Data Merging & Finalization**: Merges all processed data streams and aggregates results.
- **1.6 Status Reporting & Resilience Handling**: Generates success status messages and supports rerun logic for resilience against API limits or failures.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Prompt Preparation

**Overview:**  
This block receives the Gong call transcript and metadata, standardizes the data (e.g., correcting company name mispronunciations), and constructs a comprehensive user prompt that is sent to all AI agents for consistent analysis.

**Nodes Involved:**  
- Execute Workflow Trigger  
- Create User Prompt  
- Sticky Note (Receive Call Data and standardize User Prompt)

**Node Details:**

- **Execute Workflow Trigger**  
  - *Type:* `Execute Workflow Trigger`  
  - *Role:* Entry point that receives the Gong call data payload including transcript, metadata, attendees, and integration references.  
  - *Configuration:* No parameters; triggered externally.  
  - *Connections:* Outputs to `Create User Prompt`.  
  - *Edge Cases:* Missing or malformed call data could cause downstream failures.

- **Create User Prompt**  
  - *Type:* `Set`  
  - *Role:* Constructs a detailed prompt string embedding call context, attendee names, competitor and integration lists, and the transcript itself.  
  - *Configuration:* Uses expressions to insert dynamic data from the trigger node, including company domain, call title, and lists of competitors/integrations for disambiguation.  
  - *Key Expressions:*  
    - `prompt.transcript` includes a detailed instruction set for AI agents, emphasizing correction of mispronunciations and context awareness.  
  - *Connections:* Outputs to three AI agent processors (`Marketing AI Agent Processor`, `Product AI Agent Processor`, `AI Agent`).  
  - *Edge Cases:* Incorrect or incomplete metadata could reduce AI accuracy.

- **Sticky Note (Receive Call Data and standardize User Prompt)**  
  - *Type:* `Sticky Note`  
  - *Role:* Documentation node explaining the purpose of this block: standardizing input data and prompt for AI agents.  
  - *Connections:* None.

---

#### 2.2 AI Processing Agents

**Overview:**  
This block runs three specialized AI agents in parallel, each analyzing the call transcript for insights relevant to Sales, Marketing, and Product teams. Each agent uses the Azure OpenAI GPT-4o-mini model with tailored system messages and prompt instructions.

**Nodes Involved:**  
- Azure OpenAI Chat Model (3 instances)  
- Marketing AI Agent Processor  
- Product AI Agent Processor  
- AI Agent (Sales AI Agent)  
- Structured Output Parser1 (Marketing)  
- Structured Output Parser2 (Product)  
- Structured Output Parser3 (Sales)  
- Sticky Notes (CallForge branding and AI Agent Processor explanations)

**Node Details:**

- **Azure OpenAI Chat Model (3 nodes)**  
  - *Type:* `lmChatAzureOpenAi`  
  - *Role:* Provides GPT-4o-mini language model inference for each AI agent.  
  - *Configuration:* Model set to `gpt-4o-mini`, credentials linked to Azure OpenAI API with PII-approved key.  
  - *Connections:* Each feeds into its respective AI Agent Processor node.  
  - *Edge Cases:* API rate limits, authentication failures, or model unavailability.

- **Marketing AI Agent Processor**  
  - *Type:* `Langchain Agent`  
  - *Role:* Extracts marketing-specific insights such as landing page opportunities, workflow template requests, recurring topics, and actionable content ideas.  
  - *Configuration:* Uses a detailed system message defining marketing tags and output JSON schema.  
  - *Input:* Receives standardized prompt text from `Create User Prompt`.  
  - *Output:* Structured JSON parsed by `Structured Output Parser1`.  
  - *Edge Cases:* Misinterpretation of transcript context or missing marketing-related data.

- **Product AI Agent Processor**  
  - *Type:* `Langchain Agent`  
  - *Role:* Extracts product feedback, AI/ML references, and feature requests from the transcript.  
  - *Configuration:* System message instructs extraction of positive/negative feedback and AI/ML usage details.  
  - *Input:* Receives standardized prompt text from `Create User Prompt`.  
  - *Output:* Structured JSON parsed by `Structured Output Parser2`.  
  - *Edge Cases:* Ambiguous feedback or incomplete AI/ML context.

- **AI Agent (Sales AI Agent)**  
  - *Type:* `Langchain Agent`  
  - *Role:* Extracts sales use cases, objections, call summaries, customer pain points, next steps, competitor mentions, integrations, sentiment, budget, authority, timeline, and decision process.  
  - *Configuration:* Complex system message with detailed instructions for multiple output sections in JSON.  
  - *Input:* Receives standardized prompt text from `Create User Prompt`.  
  - *Output:* Structured JSON parsed by `Structured Output Parser3`.  
  - *Edge Cases:* Complex multi-part output may cause parsing errors if AI output deviates from expected format.

- **Structured Output Parser Nodes (1, 2, 3)**  
  - *Type:* `Langchain Output Parser Structured`  
  - *Role:* Parses AI agent raw JSON outputs into structured data for downstream processing.  
  - *Configuration:* Each has a JSON schema example matching expected AI output structure.  
  - *Input:* Connected to respective AI Agent Processor nodes.  
  - *Output:* Parsed structured JSON forwarded to data recall nodes.  
  - *Edge Cases:* Parsing failures if AI output is malformed or incomplete.

- **Sticky Notes (CallForge branding and AI Agent Processor explanations)**  
  - *Type:* `Sticky Note`  
  - *Role:* Provide visual documentation and explanation of the AI processing block and its purpose.  
  - *Connections:* None.

---

#### 2.3 Structured Output Parsing & Data Recall

**Overview:**  
This block captures the parsed AI outputs and assigns them to variables alongside call metadata and attendee information, preparing the data for department-specific processing.

**Nodes Involved:**  
- Data Recall Sales  
- Data Recall Marketing  
- Data Recall Product

**Node Details:**

- **Data Recall Sales**  
  - *Type:* `Set`  
  - *Role:* Assigns sales AI output and call metadata to variables for the sales data processor.  
  - *Configuration:* Copies AI output (`AIoutput`), metadata (`metaData`), attendees, PeopleDataLabs data, Salesforce opportunities (`sfOpp`), Pipedrive data, and Notion data from the trigger node.  
  - *Input:* From `AI Agent` output parser.  
  - *Output:* To `Sales Data Processor`.  
  - *Edge Cases:* Missing or inconsistent metadata could affect downstream processing.

- **Data Recall Marketing**  
  - *Type:* `Set`  
  - *Role:* Similar to Data Recall Sales but for marketing AI output and metadata.  
  - *Input:* From `Marketing AI Agent Processor` output parser.  
  - *Output:* To `Marketing Data Processor`.  
  - *Edge Cases:* Same as above.

- **Data Recall Product**  
  - *Type:* `Set`  
  - *Role:* Similar to above but for product AI output and metadata.  
  - *Input:* From `Product AI Agent Processor` output parser.  
  - *Output:* To `Product AI Data Processor`.  
  - *Edge Cases:* Same as above.

---

#### 2.4 Department-Specific Data Processing

**Overview:**  
This block executes sub-workflows dedicated to processing and storing insights for Sales, Marketing, and Product teams. These sub-workflows handle integration with Notion, Salesforce, or other CRMs as configured.

**Nodes Involved:**  
- Sales Data Processor  
- Marketing Data Processor  
- Product AI Data Processor  
- SF Sales Data Processor (disabled)

**Node Details:**

- **Sales Data Processor**  
  - *Type:* `Execute Workflow`  
  - *Role:* Processes sales insights, likely storing data in Notion and Salesforce for sales team use.  
  - *Configuration:* Calls sub-workflow with ID `I6lNpYOK5i8SXhPU` (Sales AI Data Processor Demo).  
  - *Input:* Receives data from `Data Recall Sales`.  
  - *Output:* To `Merge all processed data`.  
  - *Edge Cases:* Sub-workflow failures, API limits, or credential issues.

- **Marketing Data Processor**  
  - *Type:* `Execute Workflow`  
  - *Role:* Processes marketing insights, storing them in Notion or other marketing tools.  
  - *Configuration:* Calls sub-workflow with ID `enqv6mILqxzIW5TV` (Marketing AI Data Processor Demo).  
  - *Input:* Receives data from `Data Recall Marketing`.  
  - *Output:* To `Merge all processed data`.  
  - *Edge Cases:* Same as above.

- **Product AI Data Processor**  
  - *Type:* `Execute Workflow`  
  - *Role:* Processes product feedback and AI/ML references, storing in Notion or product management tools.  
  - *Configuration:* Calls sub-workflow with ID `sn0DvsN0Wqpkrxjv` (Product AI Data Processor Demo).  
  - *Input:* Receives data from `Data Recall Product`.  
  - *Output:* To `Merge all processed data`.  
  - *Edge Cases:* Same as above.

- **SF Sales Data Processor**  
  - *Type:* `Execute Workflow`  
  - *Role:* Disabled node intended for Salesforce-specific sales data processing.  
  - *Configuration:* Calls workflow ID `22QS6tCywKY2LN2K`.  
  - *Note:* Disabled, possibly replaced by integrated processing in `Sales Data Processor`.

---

#### 2.5 Data Merging & Finalization

**Overview:**  
This block merges the outputs from the three department-specific processors into a single aggregated data bundle for final reporting and notification.

**Nodes Involved:**  
- Merge all processed data  
- Bundle processed Data

**Node Details:**

- **Merge all processed data**  
  - *Type:* `Merge`  
  - *Role:* Combines outputs from Sales, Marketing, and Product processors into a single stream.  
  - *Configuration:* Set to accept 3 inputs.  
  - *Input:* From Sales Data Processor, Marketing Data Processor, Product AI Data Processor.  
  - *Output:* To `Bundle processed Data`.  
  - *Edge Cases:* Data mismatch or missing inputs could cause incomplete merges.

- **Bundle processed Data**  
  - *Type:* `Aggregate`  
  - *Role:* Aggregates all merged items into a single item containing all processed data.  
  - *Configuration:* Aggregates all item data into one.  
  - *Input:* From `Merge all processed data`.  
  - *Output:* To `Success Status Generated`.  
  - *Edge Cases:* Large data volumes could impact performance.

---

#### 2.6 Status Reporting & Resilience Handling

**Overview:**  
This block generates a success status message summarizing the workflow run and supports resilience by enabling reruns on failed calls, particularly to handle Notion API rate limits.

**Nodes Involved:**  
- Success Status Generated  
- Sticky Note (Process Queue Logic)  

**Node Details:**

- **Success Status Generated**  
  - *Type:* `Set`  
  - *Role:* Creates a human-readable status message indicating successful AI processing of the call, including call title and Gong call ID.  
  - *Configuration:* Uses expressions to insert dynamic call metadata from the trigger node.  
  - *Input:* From `Bundle processed Data`.  
  - *Output:* None (end node).  
  - *Edge Cases:* Missing metadata could cause incomplete status messages.

- **Sticky Note (Process Queue Logic)**  
  - *Type:* `Sticky Note`  
  - *Role:* Documents the workflow's resilience strategy, explaining that failed runs can be retried on remaining calls to handle API rate limits, especially with Notion.  
  - *Connections:* None.

---

### 3. Summary Table

| Node Name                 | Node Type                                | Functional Role                                  | Input Node(s)                     | Output Node(s)                                   | Sticky Note                                                                                              |
|---------------------------|-----------------------------------------|-------------------------------------------------|----------------------------------|-------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Execute Workflow Trigger   | n8n-nodes-base.executeWorkflowTrigger   | Entry point receiving Gong call data            | -                                | Create User Prompt                              |                                                                                                        |
| Create User Prompt         | n8n-nodes-base.set                      | Constructs unified AI prompt with call context  | Execute Workflow Trigger          | Marketing AI Agent Processor, Product AI Agent Processor, AI Agent | Sticky Note: Receive Call Data and standardize User Prompt                                              |
| Azure OpenAI Chat Model    | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Provides GPT-4o-mini model inference             | Marketing AI Agent Processor      | Marketing AI Agent Processor                     |                                                                                                        |
| Azure OpenAI Chat Model1   | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Provides GPT-4o-mini model inference             | Product AI Agent Processor        | Product AI Agent Processor                       |                                                                                                        |
| Azure OpenAI Chat Model2   | @n8n/n8n-nodes-langchain.lmChatAzureOpenAi | Provides GPT-4o-mini model inference             | AI Agent                        | AI Agent                                        |                                                                                                        |
| Marketing AI Agent Processor | @n8n/n8n-nodes-langchain.agent          | Extracts marketing insights from transcript      | Create User Prompt, Azure OpenAI Chat Model1 | Structured Output Parser1                        | Sticky Note: Process Marketing Agent                                                                   |
| Product AI Agent Processor | @n8n/n8n-nodes-langchain.agent          | Extracts product feedback and AI/ML references   | Create User Prompt, Azure OpenAI Chat Model2 | Structured Output Parser2                        | Sticky Note: Process Product Agent                                                                     |
| AI Agent                  | @n8n/n8n-nodes-langchain.agent          | Extracts sales use cases, objections, and more   | Create User Prompt, Azure OpenAI Chat Model | Structured Output Parser3                        | Sticky Note: Process Sales Agent                                                                       |
| Structured Output Parser1  | @n8n/n8n-nodes-langchain.outputParserStructured | Parses marketing AI output JSON                   | Marketing AI Agent Processor      | Data Recall Marketing                           |                                                                                                        |
| Structured Output Parser2  | @n8n/n8n-nodes-langchain.outputParserStructured | Parses product AI output JSON                     | Product AI Agent Processor        | Data Recall Product                             |                                                                                                        |
| Structured Output Parser3  | @n8n/n8n-nodes-langchain.outputParserStructured | Parses sales AI output JSON                       | AI Agent                        | Data Recall Sales                               |                                                                                                        |
| Data Recall Sales          | n8n-nodes-base.set                      | Assigns sales AI output and metadata              | Structured Output Parser3          | Sales Data Processor, SF Sales Data Processor (disabled) | Sticky Note: Process Sales Agent                                                                       |
| Data Recall Marketing      | n8n-nodes-base.set                      | Assigns marketing AI output and metadata          | Structured Output Parser1          | Marketing Data Processor                         | Sticky Note: Process Marketing Agent                                                                   |
| Data Recall Product        | n8n-nodes-base.set                      | Assigns product AI output and metadata            | Structured Output Parser2          | Product AI Data Processor                        | Sticky Note: Process Product Agent                                                                     |
| Sales Data Processor       | n8n-nodes-base.executeWorkflow          | Processes and stores sales insights               | Data Recall Sales                 | Merge all processed data                         | Sticky Note: Process Sales Agent                                                                       |
| Marketing Data Processor   | n8n-nodes-base.executeWorkflow          | Processes and stores marketing insights           | Data Recall Marketing             | Merge all processed data                         | Sticky Note: Process Marketing Agent                                                                   |
| Product AI Data Processor  | n8n-nodes-base.executeWorkflow          | Processes and stores product feedback              | Data Recall Product               | Merge all processed data                         | Sticky Note: Process Product Agent                                                                     |
| SF Sales Data Processor    | n8n-nodes-base.executeWorkflow (disabled) | Salesforce-specific sales data processing (disabled) | Data Recall Sales                 | Merge all processed data                         |                                                                                                        |
| Merge all processed data   | n8n-nodes-base.merge                    | Merges outputs from Sales, Marketing, Product     | Sales Data Processor, Marketing Data Processor, Product AI Data Processor | Bundle processed Data                            |                                                                                                        |
| Bundle processed Data      | n8n-nodes-base.aggregate                | Aggregates merged data into single bundle          | Merge all processed data          | Success Status Generated                         |                                                                                                        |
| Success Status Generated   | n8n-nodes-base.set                      | Generates success message with call metadata       | Bundle processed Data            | -                                               |                                                                                                        |
| Sticky Note (CallForge branding) | n8n-nodes-base.stickyNote              | Branding and workflow explanation                   | -                                | -                                               | Sticky Note: CallForge - The AI Gong Sales Call Processor; AI Agent Processor explanation               |
| Sticky Note (Process Queue Logic) | n8n-nodes-base.stickyNote              | Explains rerun and resilience logic for API limits | -                                | -                                               | Sticky Note: If the run fails, it can be rerun on remaining calls to handle Notion rate limiting        |
| Sticky Note (Receive Call Data and standardize User Prompt) | n8n-nodes-base.stickyNote              | Explains prompt standardization                      | -                                | -                                               | Sticky Note: Standardizes user prompt for all AI agents                                                |
| Sticky Note1 (Process Sales Agent) | n8n-nodes-base.stickyNote              | Explains sales agent processing                      | -                                | -                                               | Sticky Note: Sales agent output feeds Notion and Salesforce processors                                 |
| Sticky Note2 (Process Marketing Agent) | n8n-nodes-base.stickyNote              | Explains marketing agent processing                  | -                                | -                                               | Sticky Note: Marketing agent outputs to subworkflow feeding Notion                                    |
| Sticky Note3 (Process Product Agent) | n8n-nodes-base.stickyNote              | Explains product agent processing                    | -                                | -                                               | Sticky Note: Product agent outputs to subworkflow feeding Notion                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add `Execute Workflow Trigger` node to receive Gong call transcript and metadata.

2. **Create User Prompt Node**  
   - Add `Set` node named `Create User Prompt`.  
   - Configure to assign a string field `prompt.transcript` with a detailed prompt embedding:  
     - Call context (company domain, call title)  
     - Attendee names (internal and external)  
     - Competitor and integration lists for disambiguation  
     - Full call transcript  
   - Use expressions to pull data from the trigger node.

3. **Add Azure OpenAI Chat Model Nodes**  
   - Add three `lmChatAzureOpenAi` nodes, each configured with model `gpt-4o-mini` and linked to Azure OpenAI credentials.  
   - Name them for clarity: one each for Sales, Marketing, and Product AI agents.

4. **Add AI Agent Nodes**  
   - Add three `Langchain Agent` nodes:  
     - `Marketing AI Agent Processor` with system message focused on marketing insights extraction.  
     - `Product AI Agent Processor` with system message focused on product feedback and AI/ML references.  
     - `AI Agent` (Sales AI Agent) with detailed system message for sales use cases, objections, competitors, etc.  
   - Connect each `Create User Prompt` output to the respective AI Agent node’s input text.  
   - Connect each AI Agent node to its corresponding Azure OpenAI Chat Model node for inference.

5. **Add Structured Output Parser Nodes**  
   - Add three `Langchain Output Parser Structured` nodes, each configured with JSON schema examples matching expected AI outputs for Sales, Marketing, and Product.  
   - Connect each AI Agent node’s output to its respective parser node.

6. **Add Data Recall Set Nodes**  
   - Add three `Set` nodes named `Data Recall Sales`, `Data Recall Marketing`, and `Data Recall Product`.  
   - Configure each to assign:  
     - AI output from the respective parser node to `AIoutput`  
     - Metadata, attendees, PeopleDataLabs, Salesforce opportunities, Pipedrive data, and Notion data from the trigger node.  
   - Connect each parser node to its respective Data Recall node.

7. **Add Department-Specific Data Processor Nodes**  
   - Add three `Execute Workflow` nodes named `Sales Data Processor`, `Marketing Data Processor`, and `Product AI Data Processor`.  
   - Configure each to call the corresponding sub-workflow by workflow ID (replace with your own workflows if needed).  
   - Connect each Data Recall node to its respective processor node.

8. **Add Merge Node**  
   - Add a `Merge` node named `Merge all processed data`.  
   - Configure to accept 3 inputs.  
   - Connect outputs of the three data processor nodes to this merge node.

9. **Add Aggregate Node**  
   - Add an `Aggregate` node named `Bundle processed Data`.  
   - Configure to aggregate all item data into a single bundle.  
   - Connect the merge node output to this aggregate node.

10. **Add Success Status Node**  
    - Add a `Set` node named `Success Status Generated`.  
    - Configure to assign a `status` string with a message including call title and Gong call ID from the trigger node.  
    - Connect aggregate node output to this node.

11. **Add Sticky Notes for Documentation**  
    - Add sticky notes at appropriate positions to document:  
      - Workflow branding and AI agent explanation  
      - Input reception and prompt standardization  
      - Sales, Marketing, and Product agent processing explanations  
      - Process queue logic and rerun resilience notes

12. **Configure Credentials**  
    - Set up Azure OpenAI API credentials with appropriate keys and permissions.  
    - Configure any CRM or Notion credentials required by sub-workflows.

13. **Test Workflow**  
    - Trigger with sample Gong call data.  
    - Verify AI agents produce structured JSON outputs.  
    - Confirm sub-workflows process and store data correctly.  
    - Check final status message generation.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| CallForge branding image and workflow overview                                                        | ![Callforge](https://uploads.n8n.io/templates/callforgeshadow.png)                                |
| Workflow automates Gong.io sales call transcript analysis with AI-powered insights for multiple teams | Workflow description provided in the initial prompt                                               |
| Resilience strategy handles Notion API rate limits by enabling reruns on failed calls                  | Sticky Note: Process Queue Logic                                                                  |
| AI agents use Azure OpenAI GPT-4o-mini model with PII-approved credentials                              | Azure OpenAI Chat Model nodes configuration                                                      |
| Sub-workflows for Sales, Marketing, and Product data processing are modular and can be customized      | Workflow uses Execute Workflow nodes with specific workflow IDs for department processing          |
| SEO-optimized structured insights include use cases, objections, competitor analysis, and next steps   | Detailed in AI Agent system messages and output schemas                                           |
| Integration with Notion, Salesforce, and Slack for data storage and team notifications                  | Mentioned in workflow description and sub-workflow roles                                         |
| Workflow supports customization for data storage, notification channels, AI models, and CRM integrations | See "How to Customize This Workflow" section in the description                                   |

---

This structured documentation provides a comprehensive understanding of the CallForge workflow, enabling advanced users and AI agents to reproduce, modify, and anticipate operational considerations effectively.