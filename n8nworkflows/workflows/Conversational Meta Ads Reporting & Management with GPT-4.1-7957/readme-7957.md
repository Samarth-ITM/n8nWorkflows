Conversational Meta Ads Reporting & Management with GPT-4.1

https://n8nworkflows.xyz/workflows/conversational-meta-ads-reporting---management-with-gpt-4-1-7957


# Conversational Meta Ads Reporting & Management with GPT-4.1

### 1. Workflow Overview

This workflow, titled **"Conversational Meta Ads Reporting & Management with GPT-4.1,"** is designed to facilitate interactive management and reporting of Meta (Facebook) advertising accounts through conversational AI powered by GPT-4.1. It targets use cases where users want to query and manage their Facebook ad accounts, campaigns, ad sets, and ads via chat messages or commands, enabling natural language interaction for marketing analytics and campaign management.

The workflow is logically divided into two main functional blocks:

- **1.1 Conversational AI Interface**  
  Handles incoming chat messages, manages conversational context memory, and invokes the AI agent powered by OpenAI’s GPT-4.1 model for understanding and responding to user inputs.

- **1.2 Meta Ads Data Retrieval and Processing**  
  Processes commands related to Meta ad accounts by querying the Facebook Graph API for accounts, campaigns, ad sets, ads, and insights. It handles different command types, branches logic accordingly, merges data from multiple API calls, and formats the results.

---

### 2. Block-by-Block Analysis

#### 2.1 Conversational AI Interface

**Overview:**  
This block receives chat messages from users, manages conversational memory to maintain context, and uses GPT-4.1 via Langchain to process queries and generate responses.

**Nodes Involved:**  
- When chat message received  
- Simple Memory  
- OpenAI Chat Model  
- AI Agent  

**Node Details:**  

- **When chat message received**  
  - *Type*: Chat trigger node (Langchain)  
  - *Role*: Webhook that triggers the workflow upon receiving a chat message.  
  - *Config*: Default options, no filters.  
  - *Input*: External chat message (e.g., from a front-end or chat client).  
  - *Output*: Passes chat message data to AI Agent.  
  - *Failures*: Network/webhook availability issues, malformed messages.

- **Simple Memory**  
  - *Type*: Memory buffer window (Langchain)  
  - *Role*: Maintains short-term conversational memory to provide context during AI processing.  
  - *Config*: Default settings, no size limits indicated.  
  - *Input*: Previous chat messages and AI responses.  
  - *Output*: Supplies memory context to AI Agent.  
  - *Failures*: Memory overflow if too large, potential data loss on reset.

- **OpenAI Chat Model**  
  - *Type*: Language model node (Langchain)  
  - *Role*: Executes chat completions using GPT-4.1.  
  - *Config*: Model set explicitly to "gpt-4.1", uses OpenAI API credentials.  
  - *Input*: Prompts and conversational memory from AI Agent.  
  - *Output*: AI-generated chat responses.  
  - *Failures*: API auth errors, rate limits, timeouts, model unavailability.

- **AI Agent**  
  - *Type*: Langchain agent node  
  - *Role*: Central orchestrator that receives input, manages memory, calls the language model, and invokes tools/workflows as needed based on intent.  
  - *Config*: Default options; connects to memory, language model, and tool workflows.  
  - *Input*: Chat message from trigger node, memory buffer, tool workflows.  
  - *Output*: Final AI response to be sent back to user.  
  - *Failures*: Expression errors, tool invocation failures, memory sync issues.

---

#### 2.2 Meta Ads Data Retrieval and Processing

**Overview:**  
This block handles commands related to Meta ad accounts. It branches based on the command type (e.g., list accounts, fetch specific account details), calls Facebook Graph API nodes to retrieve related ad data, merges and processes data, and can be triggered either by the conversational AI or directly by another workflow.

**Nodes Involved:**  
- When Executed by Another Workflow  
- Switch  
- list accounts (Tool Workflow)  
- account details (Tool Workflow)  
- graph: adaccounts  
- Edit Fields1  
- No Operation, do nothing  
- account: campaigns  
- account: adsets  
- account: ads  
- account: ads1  
- Merge  
- Code  

**Node Details:**  

- **When Executed by Another Workflow**  
  - *Type*: Execute Workflow Trigger  
  - *Role*: Allows this workflow to be triggered programmatically with inputs (command, id, since, until).  
  - *Config*: Defines input parameters for command execution.  
  - *Input*: External workflow inputs.  
  - *Output*: Passes inputs to Switch for routing.  
  - *Failures*: Input validation issues, invalid command format.

- **Switch**  
  - *Type*: Decision node  
  - *Role*: Routes workflow based on the "command" input value.  
  - *Config*: Two outputs - "list_accounts" if command equals "list_accounts", "account" if command equals "account". Case sensitive, strict string equality.  
  - *Input*: Workflow inputs or AI Agent commands.  
  - *Output*: Routes to respective tool workflows or further processing.  
  - *Failures*: Unmatched commands lead to no output; no default route defined.

- **list accounts (Tool Workflow)**  
  - *Type*: Tool Workflow (Langchain)  
  - *Role*: Invokes a sub-workflow "MCP: Meta Ads" to list ad accounts.  
  - *Config*: Passes command: "list_accounts" as input to sub-workflow.  
  - *Input*: Command string.  
  - *Output*: List of ad accounts.  
  - *Failures*: Sub-workflow failure, input schema mismatch.

- **account details (Tool Workflow)**  
  - *Type*: Tool Workflow (Langchain)  
  - *Role*: Invokes the same sub-workflow to fetch details for a specific account with date filters.  
  - *Config*: Passes parameters "id", "since", "until", and command "account".  
  - *Input*: Account ID and date range.  
  - *Output*: Detailed data for the account.  
  - *Failures*: Invalid IDs, date format issues, sub-workflow errors.

- **graph: adaccounts**  
  - *Type*: Facebook Graph API node  
  - *Role*: Fetches ad accounts connected to the current user ("me") from Facebook Graph API v23.0.  
  - *Config*: Requests the "name" field of adaccounts.  
  - *Input*: Node "me" with Facebook Graph credentials.  
  - *Output*: List of ad accounts.  
  - *Failures*: Facebook API auth errors, permissions, rate limits.

- **Edit Fields1**  
  - *Type*: Set node  
  - *Role*: Normalizes and prepares input fields for subsequent API calls.  
  - *Config*:  
    - Ensures account ID starts with "act_" prefix.  
    - Sets "since" and "until" dates, defaulting to current month start and end if missing.  
  - *Input*: JSON from switch or workflow inputs.  
  - *Output*: Modified JSON with sanitized fields.  
  - *Failures*: Date parsing errors, invalid input data.

- **No Operation, do nothing**  
  - *Type*: No-Op node  
  - *Role*: Placeholder that triggers parallel calls downstream without transformation.  
  - *Config*: No parameters.  
  - *Input*: Output from Edit Fields1.  
  - *Output*: Passes data to campaign, adset, ads, and ads1 nodes.  
  - *Failures*: None, safe no-op.

- **account: campaigns**  
  - *Type*: Facebook Graph API node  
  - *Role*: Retrieves active campaigns for a given ad account ID.  
  - *Config*:  
    - Edge: "campaigns"  
    - Node: Account ID from input JSON  
    - Fields: "id", "name", "status"  
    - Filter: Only "ACTIVE" campaigns  
  - *Input*: Account ID from No Operation node output.  
  - *Output*: List of active campaigns.  
  - *Failures*: API auth, invalid account ID.

- **account: adsets**  
  - *Type*: Facebook Graph API node  
  - *Role*: Retrieves active ad sets for a given ad account ID.  
  - *Config*:  
    - Edge: "adsets"  
    - Node: Account ID  
    - Fields: "id", "name", "status", "campaign"  
    - Filter: Only "ACTIVE" ad sets  
  - *Input*: Account ID.  
  - *Output*: List of active ad sets.  
  - *Failures*: Same as campaigns.

- **account: ads**  
  - *Type*: Facebook Graph API node  
  - *Role*: Retrieves active ads for a given ad account ID.  
  - *Config*:  
    - Edge: "ads"  
    - Node: Account ID  
    - Fields: "id", "name", "status", "campaign{id,name}", "adset{id,name}"  
    - Filter: Only "ACTIVE" ads  
  - *Input*: Account ID.  
  - *Output*: List of active ads.  
  - *Failures*: Same as campaigns.

- **account: ads1**  
  - *Type*: Facebook Graph API node  
  - *Role*: Fetches insights data for ads within a specified date range.  
  - *Config*:  
    - Edge: "insights"  
    - Node: Account ID  
    - Fields: Various metrics like "impressions", "spend", "ctr", "reach", etc.  
    - Query parameters: "time_range[since]" and "time_range[until]" from sanitized dates  
  - *Input*: Account ID and date range.  
  - *Output*: Ads performance metrics.  
  - *Failures*: Date format errors, permissions, API limits.

- **Merge**  
  - *Type*: Merge node  
  - *Role*: Combines data streams from campaigns, adsets, ads, and insights nodes into a single output.  
  - *Config*: Combine mode "combineByPosition", includes unpaired items.  
  - *Input*: Four inputs from Facebook API nodes.  
  - *Output*: Unified dataset.  
  - *Failures*: Mismatched data arrays, incomplete inputs.

- **Code**  
  - *Type*: Code node (JavaScript)  
  - *Role*: Aggregates and formats the merged data into a structured object for downstream consumption.  
  - *Config*: Runs once per item; extracts data arrays from merged nodes into keys: campaigns, adsets, ads, account_insights.  
  - *Input*: Merged data from previous node.  
  - *Output*: Single JSON object with organized account data.  
  - *Failures*: JavaScript errors if expected data missing.

---

### 3. Summary Table

| Node Name                    | Node Type                            | Functional Role                                    | Input Node(s)                      | Output Node(s)                   | Sticky Note                                               |
|------------------------------|------------------------------------|---------------------------------------------------|----------------------------------|---------------------------------|-----------------------------------------------------------|
| When chat message received    | Langchain Chat Trigger             | Entry point for chat messages                      | (external)                       | AI Agent                        |                                                           |
| AI Agent                     | Langchain Agent                    | Processes chat input, manages memory and tools    | When chat message received, Simple Memory, OpenAI Chat Model, Tool Workflows | (final AI response)             |                                                           |
| Simple Memory                | Langchain Memory Buffer Window    | Maintains conversational context                   | AI Agent                         | AI Agent                        |                                                           |
| OpenAI Chat Model            | Langchain Language Model (OpenAI) | Provides GPT-4.1 responses                          | AI Agent                         | AI Agent                        |                                                           |
| When Executed by Another Workflow | Execute Workflow Trigger          | Trigger workflow with external inputs              | (external)                      | Switch                         |                                                           |
| Switch                      | Switch                            | Routes commands to appropriate processing paths   | When Executed by Another Workflow | graph: adaccounts, Edit Fields1 |                                                           |
| list accounts               | Tool Workflow (Langchain)         | Calls sub-workflow to list ad accounts             | AI Agent, Switch                 | AI Agent                       |                                                           |
| account details             | Tool Workflow (Langchain)         | Calls sub-workflow for specific account details    | AI Agent, Switch                 | AI Agent                       |                                                           |
| graph: adaccounts           | Facebook Graph API                 | Retrieves user's ad accounts                        | Switch                          | (none)                        |                                                           |
| Edit Fields1                | Set                              | Normalizes account ID and date fields              | Switch                         | No Operation, do nothing       |                                                           |
| No Operation, do nothing    | No-Op                            | Passes data through to multiple downstream nodes   | Edit Fields1                   | account: campaigns, adsets, ads, ads1 |                                                           |
| account: campaigns          | Facebook Graph API                 | Fetches active campaigns for account                | No Operation                   | Merge                         |                                                           |
| account: adsets             | Facebook Graph API                 | Fetches active ad sets for account                   | No Operation                   | Merge                         |                                                           |
| account: ads                | Facebook Graph API                 | Fetches active ads for account                        | No Operation                   | Merge                         |                                                           |
| account: ads1               | Facebook Graph API                 | Fetches ad insights for account within date range   | No Operation                   | Merge                         |                                                           |
| Merge                      | Merge                            | Combines campaigns, adsets, ads, and insights data | account: campaigns, adsets, ads, ads1 | Code                         |                                                           |
| Code                       | Code (JavaScript)                 | Formats merged data into structured JSON             | Merge                         | (final output)                |                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" node**  
   - Type: Langchain Chat Trigger  
   - Configure webhook with default settings to receive chat messages.

2. **Create "Simple Memory" node**  
   - Type: Langchain Memory Buffer Window  
   - Use default settings for conversational memory.

3. **Create "OpenAI Chat Model" node**  
   - Type: Langchain LM Chat OpenAI  
   - Set model to "gpt-4.1"  
   - Configure OpenAI API credentials with valid account.

4. **Create "AI Agent" node**  
   - Type: Langchain Agent  
   - Connect inputs: from "When chat message received" (main), "Simple Memory" (ai_memory), "OpenAI Chat Model" (ai_languageModel)  
   - Configure to use tool workflows (to be created) for specific commands.

5. **Connect "When chat message received" → "AI Agent"** (main connection)  
   Connect "Simple Memory" → "AI Agent" (ai_memory)  
   Connect "OpenAI Chat Model" → "AI Agent" (ai_languageModel)

6. **Create "When Executed by Another Workflow" node**  
   - Type: Execute Workflow Trigger  
   - Define workflow input parameters: command (string), id (string), since (string), until (string).

7. **Create "Switch" node**  
   - Type: Switch  
   - Configure rules:  
     - Output "list_accounts" if command equals "list_accounts"  
     - Output "account" if command equals "account"

8. **Connect "When Executed by Another Workflow" → "Switch"**

9. **Create "list accounts" Tool Workflow node**  
   - Type: Langchain Tool Workflow  
   - Set workflow to sub-workflow ID (e.g., "MCP: Meta Ads")  
   - Input: { command: "list_accounts" }  
   - Connect "Switch" output "list_accounts" → "list accounts" (ai_tool)

10. **Create "account details" Tool Workflow node**  
    - Type: Langchain Tool Workflow  
    - Set same sub-workflow ID as above  
    - Input: { command: "account", id, since, until } (map from incoming parameters)  
    - Connect "Switch" output "account" → "account details" (ai_tool)

11. **Create "graph: adaccounts" node**  
    - Type: Facebook Graph API  
    - Set node to "me" and edge to "adaccounts"  
    - Request field "name"  
    - Set Facebook Graph API credentials.

12. **Create "Edit Fields1" node**  
    - Type: Set  
    - Assignments:  
      - id: prepend "act_" if not present  
      - since: input since or default to first day of current month  
      - until: input until or default to last day of current month

13. **Create "No Operation, do nothing" node**  
    - Type: No-Op  
    - No parameters

14. **Create "account: campaigns" node**  
    - Type: Facebook Graph API  
    - Node: account id from input JSON  
    - Edge: "campaigns"  
    - Fields: "id", "name", "status"  
    - Query parameter: effective_status = ["ACTIVE"]  
    - Facebook Graph API credentials.

15. **Create "account: adsets" node**  
    - Same as campaigns, edge "adsets", fields: "id", "name", "status", "campaign"  
    - Filter for "ACTIVE"

16. **Create "account: ads" node**  
    - Edge: "ads"  
    - Fields: "id", "name", "status", "campaign{id,name}", "adset{id,name}"  
    - Filter for "ACTIVE"

17. **Create "account: ads1" node**  
    - Edge: "insights"  
    - Fields: "attribution_setting", "conversion_values", "conversions", "cost_per_conversion", "cpc", "cpm", "ctr", "frequency", "impressions", "inline_post_engagement", "purchase_roas", "reach", "result_rate", "results", "spend"  
    - Query parameters: time_range[since] and time_range[until] (from sanitized dates)  
    - Facebook Graph API credentials.

18. **Create "Merge" node**  
    - Type: Merge  
    - Mode: Combine  
    - Combine by position  
    - Number of inputs: 4  
    - Include unpaired items.

19. **Create "Code" node**  
    - Type: Code (JavaScript)  
    - Mode: Run once for each item  
    - Code: Extract campaigns, adsets, ads, and account_insights data arrays from merged inputs and return as a single JSON object.

20. **Connect nodes as follows:**  
    - "Switch" output "list_accounts" → "list accounts"  
    - "Switch" output "account" → "Edit Fields1" → "No Operation" → parallel to:  
      - "account: campaigns" → "Merge" input 1  
      - "account: adsets" → "Merge" input 2  
      - "account: ads" → "Merge" input 3  
      - "account: ads1" → "Merge" input 4  
    - "Merge" → "Code" (final output)

21. **Credential setup:**  
    - Configure OpenAI API with valid key for GPT-4.1  
    - Configure Facebook Graph API with OAuth2 credentials that have permissions to access ad accounts, campaigns, ads, and insights.

22. **Sub-workflow "MCP: Meta Ads"**  
    - Must accept inputs command, id, since, until  
    - Provide functionality to list accounts or return account details based on command  
    - Ensure outputs are compatible with this workflow’s expected input/output schemas.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow uses GPT-4.1 model via Langchain for advanced conversational AI                              | Requires OpenAI API access with GPT-4.1 availability                                                  |
| Facebook Graph API version v23.0 is used for all Meta Ads data queries                               | Ensure Facebook App is properly configured with required permissions                                  |
| Sub-workflow "MCP: Meta Ads" is referenced for account listing and details retrieval via Langchain tool workflows | Sub-workflow ID: UXdblREvbkvy3WSs (must be available and configured in n8n instance)                   |
| For date fields, defaults to current month’s start and end if not provided                           | Uses n8n expression to get `$today.startOf('month')` and `$today.endOf('month')`                       |
| The workflow supports both direct chat interaction and programmatic triggering via workflow execution| Enables integration flexibility                                                                       |
| This setup allows scaling and modular extension by adding more tools or data sources to AI agent     | Design pattern supports conversational AI with external tool integrations                             |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.