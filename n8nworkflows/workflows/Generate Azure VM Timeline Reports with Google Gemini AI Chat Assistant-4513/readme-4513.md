Generate Azure VM Timeline Reports with Google Gemini AI Chat Assistant

https://n8nworkflows.xyz/workflows/generate-azure-vm-timeline-reports-with-google-gemini-ai-chat-assistant-4513


# Generate Azure VM Timeline Reports with Google Gemini AI Chat Assistant

### 1. Workflow Overview

This workflow provides an AI-driven chat assistant that generates detailed timeline reports about Azure Virtual Machines (VMs) for users. It leverages Google Gemini AI chat models combined with Azure Monitor APIs to gather VM inventory, performance statistics, event logs, and runtime state data. The assistant responds to chat input requesting VM operational histories over a configurable timeframe (defaulting to the last 90 days).

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures user chat messages triggering the workflow.
- **1.2 Variable Initialization:** Sets common variables such as the Azure subscription ID.
- **1.3 AI Chat and Memory Management:** Uses Google Gemini for chat language understanding and maintains conversational context.
- **1.4 Azure Resource Discovery:** Retrieves Azure resource groups and virtual machines metadata.
- **1.5 Azure VM Runtime Information:** Queries instance views for VM states.
- **1.6 Azure VM Performance Metrics:** Collects time-series performance data from Azure Monitor.
- **1.7 Azure VM Activity Logs:** Gathers control-plane event logs related to VM changes.
- **1.8 AI Agent Orchestration:** Integrates all gathered data and responds with a timeline report.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for incoming chat messages from users. Acts as the workflow’s entry point.
- **Nodes Involved:** `When chat message received`
- **Node Details:**
  - **Type:** Chat trigger node (`@n8n/n8n-nodes-langchain.chatTrigger`)
  - **Configuration:** Listens via webhook for chat input; no additional options set.
  - **Expressions/Variables:** Outputs `chatInput` (user's message) and `sessionId`.
  - **Input:** External webhook trigger.
  - **Output:** Passes data to `Set Common Variables`.
  - **Edge Cases:** Missing or malformed chat input; webhook connectivity issues.
  - **Version:** 1.1

#### 2.2 Variable Initialization

- **Overview:** Defines common workflow variables like Azure subscription ID for use in API calls.
- **Nodes Involved:** `Set Common Variables`
- **Node Details:**
  - **Type:** Set node
  - **Configuration:** Assigns string variable `azure_subscription_id` with placeholder `<your azure subscription id here>`.
  - **Input:** Receives chat message data.
  - **Output:** Sends variables and input data to `AI Agent`.
  - **Edge Cases:** Subscription ID not set or incorrect leads to API authorization failures.
  - **Version:** 3.4

#### 2.3 AI Chat and Memory Management

- **Overview:** Handles the natural language understanding and conversational context.
- **Nodes Involved:** `Google Gemini Chat Model`, `Simple Memory`
- **Node Details:**
  - **Google Gemini Chat Model:**
    - **Type:** Language model node using Google Gemini (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`)
    - **Configuration:** Uses model `models/gemini-2.5-pro-preview-05-06` with credentials for Google Palm API.
    - **Input:** None directly connected; triggered internally by the AI Agent.
    - **Output:** Provides AI-generated chat responses.
    - **Edge Cases:** API rate limits, invalid credentials, model unavailability.
    - **Version:** 1
  - **Simple Memory:**
    - **Type:** Memory buffer node (`@n8n/n8n-nodes-langchain.memoryBufferWindow`)
    - **Configuration:** Uses session key derived from chat session ID to maintain context across messages.
    - **Input:** Chat session context.
    - **Output:** Feeds conversational memory to the AI Agent.
    - **Edge Cases:** Loss of session ID or memory corruption can degrade chat continuity.
    - **Version:** 1.3

#### 2.4 Azure Resource Discovery

- **Overview:** Retrieves Azure resource groups and lists all VMs in specified groups.
- **Nodes Involved:** `Get Azure Resource Groups`, `Get VM Information`
- **Node Details:**
  - **Get Azure Resource Groups:**
    - **Type:** HTTP request tool node (`@n8n/n8n-nodes-langchain.toolHttpRequest`)
    - **Configuration:** Calls Azure Management API to list resource groups using the subscription ID.
    - **Parameters:** API version `2021-04-01`, fields limited to `name` and `type`.
    - **Authentication:** OAuth2 with Microsoft Azure Monitor credentials.
    - **Output:** List of resource groups to downstream nodes.
    - **Edge Cases:** Auth errors, empty subscription, API throttling.
    - **Version:** 1.1
  - **Get VM Information:**
    - **Type:** HTTP request tool node
    - **Configuration:** Queries VMs within a resource group, retrieving metadata like VM name, location, tags, and size.
    - **Parameters:** API version `2023-03-01`.
    - **Authentication:** Same as above.
    - **Placeholders:** `azure_resource_group_name` dynamically set.
    - **Edge Cases:** Empty groups, permission issues, malformed group names.
    - **Version:** 1.1

#### 2.5 Azure VM Runtime Information

- **Overview:** Fetches current operational state details of VMs.
- **Nodes Involved:** `Get VM Instance View`
- **Node Details:**
  - **Type:** HTTP request tool node
  - **Configuration:** Retrieves instance view data, including power state and health status.
  - **Parameters:** API version `2023-09-01`.
  - **Authentication:** Microsoft Azure Monitor OAuth2.
  - **Placeholders:** `azure_resource_group_name`, `vm_name`.
  - **Edge Cases:** VM not found, API limits, transient network errors.
  - **Version:** 1.1

#### 2.6 Azure VM Performance Metrics

- **Overview:** Obtains performance metrics (CPU, network, disk) over specified intervals.
- **Nodes Involved:** `Get VM Performance Stats`
- **Node Details:**
  - **Type:** HTTP request tool node
  - **Configuration:** Calls Azure Monitor metrics endpoint for VM, requesting metrics like Percentage CPU, Network In/Out, Disk Read/Write Bytes.
  - **Parameters:** API version `2018-01-01`, timespan and interval placeholders.
  - **Authentication:** OAuth2 credentials.
  - **Placeholders:** `azure_resource_group_name`, `vm_name`, `from_time`, `to_time`, `interval`.
  - **Edge Cases:** Invalid time ranges, missing metrics, API throttling.
  - **Version:** 1.1

#### 2.7 Azure VM Activity Logs

- **Overview:** Retrieves management operation events for VMs useful for audit trails and timeline construction.
- **Nodes Involved:** `Get VM Events`
- **Node Details:**
  - **Type:** HTTP request tool node
  - **Configuration:** Queries Azure Activity Logs filtered by event timestamp (default 90 days) and optionally by VM/resource group.
  - **Parameters:** API version `2015-04-01`, uses `$filter` expression to limit timeframe and resources.
  - **Authentication:** OAuth2.
  - **Placeholders:** `resource_group_name`, `vm_name` (optional), `start_timestamp` (optional).
  - **Edge Cases:** Date parsing errors, missing permissions, large result sets.
  - **Version:** 1.1

#### 2.8 AI Agent Orchestration

- **Overview:** Central AI agent node orchestrates data retrieval and processes user input to generate timeline reports.
- **Nodes Involved:** `AI Agent`
- **Node Details:**
  - **Type:** Langchain AI agent node (`@n8n/n8n-nodes-langchain.agent`)
  - **Configuration:**
    - Text input is the user’s chat message.
    - System message defines the assistant’s role: querying Azure subscription for VM data and generating timeline reports.
    - Includes instructions on default timeframes and decision autonomy.
  - **Inputs:** Receives data and triggers from all other tools (performance, events, instance view, resource groups, VM info, current date, memory, language model).
  - **Outputs:** Produces the final timeline report response for the user.
  - **Edge Cases:** Expression evaluation failures, AI model latency, API call failures cascading into incomplete reports.
  - **Version:** 1.8

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role                         | Input Node(s)                   | Output Node(s)                | Sticky Note                                  |
|-------------------------|-------------------------------------|---------------------------------------|-------------------------------|------------------------------|----------------------------------------------|
| When chat message received | Chat trigger (`chatTrigger`)         | Entry point, listens for user messages | External webhook               | Set Common Variables          |                                              |
| Set Common Variables      | Set node                            | Defines Azure subscription ID          | When chat message received     | AI Agent                     |                                              |
| Google Gemini Chat Model  | Language model (Google Gemini)      | Processes natural language chat input  | AI Agent (ai_languageModel)    | AI Agent                     |                                              |
| Simple Memory             | Memory buffer                      | Maintains chat session context          | AI Agent (ai_memory)           | AI Agent                     |                                              |
| Get Azure Resource Groups | HTTP Request (Azure Management API) | Lists resource groups in subscription  | AI Agent (ai_tool)             | AI Agent                     |                                              |
| Get VM Information        | HTTP Request                       | Lists VMs within resource group         | AI Agent (ai_tool)             | AI Agent                     |                                              |
| Get VM Instance View      | HTTP Request                       | Gets runtime VM state and health        | AI Agent (ai_tool)             | AI Agent                     |                                              |
| Get VM Performance Stats  | HTTP Request                       | Retrieves VM performance metrics        | AI Agent (ai_tool)             | AI Agent                     |                                              |
| Get VM Events             | HTTP Request                       | Gathers Azure Activity Log events       | AI Agent (ai_tool)             | AI Agent                     |                                              |
| Get Current Date          | Code node (JavaScript)             | Provides current ISO date/time          | AI Agent (ai_tool)             | AI Agent                     |                                              |
| AI Agent                  | Langchain AI agent                 | Orchestrates all data, generates report | Set Common Variables, all tools| Final output to chat          |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Add `When chat message received` node.
   - Configure webhook to receive chat messages.
   - No additional settings needed.

2. **Set Common Variables:**
   - Add a `Set` node named `Set Common Variables`.
   - Define string variable `azure_subscription_id` with your Azure subscription ID.
   - Connect output of chat trigger node to this node.

3. **Add AI Agent Node:**
   - Add `AI Agent` node (`@n8n/n8n-nodes-langchain.agent`).
   - Set text expression to: `={{ $('When chat message received').item.json.chatInput }}`
   - Configure system message:
     ```
     You are an assistant that queries the user's Azure subscription for various Azure virtual machine information.

     Through your tools, you have the ability to query information in Azure such as resource groups and virtual machines.

     Your task is to carefully analyze all information about one or more VMs and provide a timeline report to the user including VM state changes, CPU, network, disk activity, etc. The report will be broken down by VM and provide a historical timeline of what happened to a VM during that time.

     Do not ask the user questions unless absolutely necessary. He relies on you to make your own decisions.

     By default, the user needs all events starting 90 days ago and newer but can override that with a request.
     ```
   - Connect `Set Common Variables` output to this node.

4. **Add Google Gemini Chat Model:**
   - Add `Google Gemini Chat Model` node.
   - Select model `models/gemini-2.5-pro-preview-05-06`.
   - Configure credentials for Google Palm API.
   - Connect this node to `AI Agent` as `ai_languageModel` input.

5. **Add Simple Memory Node:**
   - Add `Simple Memory` node.
   - Set session key: `={{ $('When chat message received').item.json.sessionId }}`
   - Choose session ID type: custom key.
   - Connect this node to `AI Agent` as `ai_memory` input.

6. **Add Get Azure Resource Groups Node:**
   - Add HTTP Request tool node named `Get Azure Resource Groups`.
   - URL: `https://management.azure.com/subscriptions/{{ $json.azure_subscription_id }}/resourcegroups?api-version=2021-04-01`
   - Authentication: Microsoft Azure Monitor OAuth2 credentials.
   - Fields to include: `name, type`
   - Connect this node to `AI Agent` as `ai_tool` input.

7. **Add Get VM Information Node:**
   - Add HTTP Request tool node named `Get VM Information`.
   - URL: `https://management.azure.com/subscriptions/{{ $json.azure_subscription_id }}/resourceGroups/{azure_resource_group_name}/providers/Microsoft.Compute/virtualMachines?api-version=2023-03-01`
   - Authentication: Microsoft Azure Monitor OAuth2.
   - Query params: `api-version=2023-03-01`
   - Fields: `name,location,tags,properties.hardwareProfile.vmSize`
   - Connect to `AI Agent` as `ai_tool`.

8. **Add Get VM Instance View Node:**
   - Add HTTP Request tool node named `Get VM Instance View`.
   - URL: `https://management.azure.com/subscriptions/{{ $json.azure_subscription_id }}/resourceGroups/{azure_resource_group_name}/providers/Microsoft.Compute/virtualMachines/{vm_name}/instanceView?api-version=2023-09-01`
   - Authentication: Microsoft Azure Monitor OAuth2.
   - Connect to `AI Agent` as `ai_tool`.

9. **Add Get VM Performance Stats Node:**
   - Add HTTP Request tool node named `Get VM Performance Stats`.
   - URL: `https://management.azure.com/subscriptions/{{ $json.azure_subscription_id }}/resourceGroups/{azure_resource_group_name}/providers/Microsoft.Compute/virtualMachines/{vm_name}/providers/microsoft.insights/metrics`
   - Query parameters:
     - `api-version=2018-01-01`
     - `metricnames=Percentage CPU,Network In,Network Out,Disk Read Bytes,Disk Write Bytes`
     - `timespan={from_time}/{to_time}`
     - `interval={interval}` (e.g., PT1H)
     - `aggregation=average,maximum`
   - Authentication: Microsoft Azure Monitor OAuth2.
   - Connect to `AI Agent` as `ai_tool`.

10. **Add Get VM Events Node:**
    - Add HTTP Request tool node named `Get VM Events`.
    - URL: `https://management.azure.com/subscriptions/{{ $json.azure_subscription_id }}/providers/microsoft.insights/eventtypes/management/values?api-version=2015-04-01`
    - Query parameter `$filter` (example):
      ```
      eventTimestamp ge '{{ $if('{start_timestamp}', '{start_timestamp}', DateTime.utc().minus({ days: 90 }).startOf('second').toISO()) }}' and resourceGroupName eq '{resource_group_name}'
      ```
      - Optionally add filter by vm_name.
    - Authentication: Microsoft Azure Monitor OAuth2.
    - Connect to `AI Agent` as `ai_tool`.

11. **Add Get Current Date Node:**
    - Add Code node named `Get Current Date`.
    - JavaScript code:
      ```javascript
      const curDate = new Date().toISOString();
      return curDate;
      ```
    - Connect to `AI Agent` as `ai_tool`.

12. **Finalize Connections:**
    - Ensure all `ai_tool` inputs connect their respective nodes to `AI Agent`.
    - Connect `Set Common Variables` main output to `AI Agent` main input.
    - Connect `When chat message received` main output to `Set Common Variables`.

13. **Credentials Setup:**
    - Configure Microsoft Azure Monitor OAuth2 credentials with proper permissions to access subscription data.
    - Configure Google Palm API credentials for Google Gemini chat model.

14. **Testing and Validation:**
    - Test chat webhook reception.
    - Validate API calls return expected data.
    - Confirm AI agent generates timeline reports.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| The AI assistant is designed to autonomously decide which VM data to query and how to structure the timeline report without user prompts unless necessary.                                                                                      | System message in AI Agent node                                                                                      |
| The workflow requires OAuth2 credentials for Microsoft Azure Monitor API with subscription-level read access.                                                                                                                                   | Azure portal authentication setup                                                                                    |
| Google Gemini model used is `models/gemini-2.5-pro-preview-05-06` requiring Google Palm API credentials.                                                                                                                                         | https://developers.generativeai.google/api/python/gemini                                                             |
| Azure Monitor API calls use specific API versions per endpoint to ensure compatibility and access to latest features.                                                                                                                           | Microsoft Azure REST API Reference                                                                                    |
| Time intervals for metrics follow ISO 8601 duration format (e.g., PT1H for 1 hour).                                                                                                                                                              | ISO 8601 duration standard                                                                                            |
| The workflow defaults to analyzing events from the last 90 days unless overridden in user input.                                                                                                                                                 | System message and `$filter` in Get VM Events node                                                                   |

---

**Disclaimer:** The text provided stems exclusively from an automated workflow built using n8n, an integration and automation tool. This process strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.