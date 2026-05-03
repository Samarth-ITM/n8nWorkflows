Customer Support Channel and Ticketing System with Slack and Linear

https://n8nworkflows.xyz/workflows/customer-support-channel-and-ticketing-system-with-slack-and-linear-2323


# Customer Support Channel and Ticketing System with Slack and Linear

### 1. Workflow Overview

This workflow implements a streamlined customer support ticketing system integrating Slack, Linear, and AI capabilities. It targets small support teams seeking to leverage their existing tools without complex onboarding. The workflow automatically detects tagged Slack messages in a dedicated support channel, processes them with AI to generate structured ticket information, and manages ticket creation in Linear, facilitating efficient issue tracking and resolution.

Logical blocks:

- **1.1 Scheduled Slack Message Query:** Periodically searches a specific Slack channel for messages marked with a ticket emoji.
- **1.2 Message Data Extraction and Deduplication:** Extracts essential message details and checks if a ticket already exists to avoid duplicates.
- **1.3 AI Processing for Ticket Generation:** Sends message content to an AI model to create ticket title, summary, priority, and suggestions.
- **1.4 Ticket Creation in Linear:** Uses AI-generated data to create a new ticket in Linear with appropriate metadata and priority.
- **1.5 Data Aggregation for Deduplication:** Aggregates existing tickets’ metadata to identify already processed messages.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Slack Message Query

**Overview:**  
This block triggers the workflow on a schedule and queries Slack for recent messages tagged with a ticket emoji in a designated channel.

**Nodes Involved:**  
- Schedule Trigger  
- Slack  
- Sticky Note (explaining Slack search)

**Node Details:**

- **Schedule Trigger**  
  - *Type:* scheduleTrigger  
  - *Role:* Initiates the workflow every few minutes (configured in minutes interval).  
  - *Configuration:* Set to run at a defined interval in minutes to periodically start the workflow.  
  - *Connections:* Output to Slack node.  
  - *Potential Failures:* Incorrect interval may cause API rate limits or missed messages.

- **Slack**  
  - *Type:* slack (search operation)  
  - *Role:* Queries Slack messages in the specific channel `#n8n-tickets` that have the ticket emoji reaction (`:ticket:`).  
  - *Configuration:* Uses the Slack API search syntax `"in:#n8n-tickets has::ticket:"` with a limit of 10 messages.  
  - *Credentials:* Slack API credentials are required.  
  - *Connections:* Outputs to "Get Values" node.  
  - *Edge Cases:* Slack API rate limits; channel or emoji misconfiguration; no matching messages available.

- **Sticky Note (Slack Search Explanation)**  
  - Provides context and a link to Slack API search usage.  
  - *Content Highlights:* Emphasizes filtering latest messages with the ticket emoji in the monitored channel.

---

#### 2.2 Message Data Extraction and Deduplication

**Overview:**  
Extracts key fields from Slack messages and queries Linear for existing tickets to prevent duplicate ticket creation.

**Nodes Involved:**  
- Get Values (Set node)  
- Get Existing Issues (Linear node)  
- Collect Descriptions (Aggregate node)  
- Get Hashes Only (Set node)  
- Merge  
- Create New Ticket? (If node)  
- Sticky Notes (explaining deduplication and Slack channel config)

**Node Details:**

- **Get Values**  
  - *Type:* set  
  - *Role:* Extracts and formats message fields (id, type, title placeholder, channel, user, timestamp, permalink, text) from Slack message JSON.  
  - *Key Expressions:*  
    - `id` derived from the last segment of the permalink URL.  
    - `message` text is sanitized for quotes and newlines.  
  - *Connections:* Outputs to "Merge" and "Get Existing Issues" nodes.  
  - *Edge Cases:* Missing or malformed message data, permalink parsing errors.

- **Get Existing Issues**  
  - *Type:* linear (getAll operation)  
  - *Role:* Retrieves all existing tickets from Linear to compare against Slack messages.  
  - *Credentials:* Linear API credentials required.  
  - *Connections:* Outputs to "Collect Descriptions".  
  - *Failures:* API errors, authentication issues, large data sets causing timeouts.

- **Collect Descriptions**  
  - *Type:* aggregate  
  - *Role:* Aggregates the "description" fields from Linear tickets into an array.  
  - *Connections:* Outputs to "Get Hashes Only".  
  - *Edge Cases:* Empty ticket list, malformed descriptions.

- **Get Hashes Only**  
  - *Type:* set  
  - *Role:* Extracts hash identifiers from aggregated ticket descriptions using regex matching `"hash: <id>"`.  
  - *Connections:* Outputs to "Merge".  
  - *Potential Failures:* Regex mismatches if description format changes.

- **Merge**  
  - *Type:* merge (combine multiplex mode)  
  - *Role:* Combines Slack message data and existing Linear ticket hashes for comparison.  
  - *Connections:* Outputs to "Create New Ticket?" node.  
  - *Edge Cases:* Missing inputs or uneven data arrays.

- **Create New Ticket?**  
  - *Type:* if  
  - *Role:* Determines whether to create a new ticket by verifying if the Slack message id is absent in the hashes extracted from existing tickets.  
  - *Key Expression:* Checks if `$json.hashes` includes current `$json.id`. Creates new ticket only if `false`.  
  - *Connections:* If true (new ticket needed), proceeds to AI processing.  
  - *Edge Cases:* False negatives/positives in hash matching; case sensitivity issues.

- **Sticky Notes**  
  - Explain deduplication logic and emphasize required configuration of the Slack channel to monitor.

---

#### 2.3 AI Processing for Ticket Generation

**Overview:**  
Uses OpenAI's chat model via n8n Langchain nodes to generate a ticket title, summary, priority, and suggestions based on the Slack message content.

**Nodes Involved:**  
- OpenAI Chat Model  
- Generate Ticket Using ChatGPT (Chain LLM node with structured output parser)  
- Structured Output Parser  
- Sticky Note (explaining AI processing)

**Node Details:**

- **OpenAI Chat Model**  
  - *Type:* lmChatOpenAi (Langchain)  
  - *Role:* Performs the AI language model interaction for text generation.  
  - *Credentials:* OpenAI API key required.  
  - *Connections:* AI output connected to "Generate Ticket Using ChatGPT" node.  
  - *Failures:* API limits, network issues, prompt errors.

- **Generate Ticket Using ChatGPT**  
  - *Type:* chainLlm (Langchain LLM Chain with output parser)  
  - *Role:* Sends a prompt instructing the AI to generate:  
    1. A descriptive ticket title (<=10 words)  
    2. A summary of the issue  
    3. Up to 3 debugging suggestions  
    4. A priority label (low, medium, high, urgent) based on urgency  
  - *Prompt:* Contains the Slack message encapsulated in triple backticks.  
  - *Output Parser:* Uses "Structured Output Parser" for JSON schema validation.  
  - *Connections:* Outputs parsed results to "Create Ticket".  
  - *Edge Cases:* AI output format deviations; misinterpretation of priority; rate limits.

- **Structured Output Parser**  
  - *Type:* outputParserStructured  
  - *Role:* Parses AI response into JSON object with fields: title, summary, ideas (array), priority.  
  - *Schema:* Validates expected fields and types.  
  - *Connections:* Feeds parsed output back into the chain node.  
  - *Failures:* Parsing errors if AI outputs unexpected format.

- **Sticky Note**  
  - Explains AI's role and the tasks assigned to it for ticket content generation.

---

#### 2.4 Ticket Creation in Linear

**Overview:**  
Creates a new ticket in Linear using the AI-generated ticket details and Slack message metadata.

**Nodes Involved:**  
- Create Ticket (Linear node)  
- Sticky Note (explaining Linear ticket creation)

**Node Details:**

- **Create Ticket**  
  - *Type:* linear (create operation)  
  - *Role:* Creates a new issue in Linear with:  
    - Title from AI output  
    - Priority mapped to Linear priority IDs (urgent=1, high=2, medium=3, low=4)  
    - Description combining AI summary, suggestions, original Slack message user and text, plus metadata including channel, timestamp, permalink, and hash.  
  - *Parameters:*  
    - Team ID and State ID must be configured for the target Linear project and ticket state.  
  - *Credentials:* Linear API credentials required.  
  - *Connections:* Terminal node for new ticket creation.  
  - *Edge Cases:* Incorrect team or state IDs, API failures, mismatched priority mappings.

- **Sticky Note**  
  - Highlights the need to configure Linear Team ID and ticket state parameters.

---

#### 2.5 Summary and User Instructions

**Nodes Involved:**  
- Sticky Note (general instructions and summary)

**Content:**  
- Summarizes workflow capabilities and encourages users to join the n8n community channels for support.

---

### 3. Summary Table

| Node Name               | Node Type                    | Functional Role                              | Input Node(s)           | Output Node(s)               | Sticky Note                                                                                                    |
|-------------------------|------------------------------|----------------------------------------------|-------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | scheduleTrigger              | Initiates workflow on scheduled interval     |                         | Slack                       |                                                                                                                |
| Slack                  | slack (search)               | Queries Slack for messages with ticket emoji | Schedule Trigger        | Get Values                  | Explains Slack search usage and ticket emoji filter. [Slack docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack)  |
| Get Values             | set                         | Extracts and formats Slack message data      | Slack                   | Merge, Get Existing Issues   |                                                                                                                |
| Get Existing Issues    | linear (getAll)              | Retrieves all Linear tickets for deduplication | Get Values              | Collect Descriptions         | Explains deduplication logic and Slack channel config.                                                        |
| Collect Descriptions   | aggregate                   | Aggregates ticket descriptions from Linear   | Get Existing Issues      | Get Hashes Only              |                                                                                                                |
| Get Hashes Only        | set                         | Extracts hashes from ticket descriptions      | Collect Descriptions     | Merge                       |                                                                                                                |
| Merge                  | merge (combine multiplex)   | Combines message and ticket hash data         | Get Values, Get Hashes Only | Create New Ticket?         |                                                                                                                |
| Create New Ticket?     | if                          | Decides whether to create a new ticket        | Merge                   | Generate Ticket Using ChatGPT |                                                                                                                |
| OpenAI Chat Model      | lmChatOpenAi (Langchain)    | Sends message to AI model for ticket content  |                         | Generate Ticket Using ChatGPT (ai_languageModel) | Explains AI processing role and tasks. [Basic LLM Chain](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm) |
| Generate Ticket Using ChatGPT | chainLlm (Langchain)      | Generates ticket title, summary, priority     | Create New Ticket?       | Create Ticket               |                                                                                                                |
| Structured Output Parser | outputParserStructured      | Parses AI output into structured JSON         | OpenAI Chat Model output | Generate Ticket Using ChatGPT (ai_outputParser) |                                                                                                                |
| Create Ticket          | linear (create)             | Creates a new ticket in Linear                 | Generate Ticket Using ChatGPT |                          | Requires Linear Team ID and State ID configuration. [Linear docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.linear)          |
| Sticky Note            | stickyNote                  | Various explanatory notes                       |                         |                             | See individual notes above for content and links.                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: `scheduleTrigger`  
   - Configure interval to run every few minutes (e.g., every 5 minutes).  

2. **Create Slack Search Node**  
   - Type: `slack`  
   - Operation: `search`  
   - Query: `in:#n8n-tickets has::ticket:` (adjust channel as needed)  
   - Limit: 10 messages  
   - Connect Schedule Trigger output to Slack node input.  
   - Set up Slack API credentials with appropriate permissions to read messages and reactions.

3. **Create Set Node "Get Values"**  
   - Type: `set`  
   - Extract fields from Slack message JSON:  
     - `id`: last segment of `permalink` URL  
     - `type`: message type  
     - `title`: placeholder string `__NOT_SET__`  
     - `channel`: channel name  
     - `user`: username and user ID combined  
     - `ts`: timestamp  
     - `permalink`: message permalink  
     - `message`: sanitized text (escape quotes, newlines)  
   - Connect Slack node output to "Get Values".  

4. **Create Linear Node "Get Existing Issues"**  
   - Type: `linear`  
   - Operation: `getAll` to retrieve all tickets in target project  
   - Set credentials for Linear API with required scopes.  
   - Connect "Get Values" output to this node's input.

5. **Create Aggregate Node "Collect Descriptions"**  
   - Type: `aggregate`  
   - Aggregate field: `description` (rename output field to `descriptions`)  
   - Connect "Get Existing Issues" output to this node.

6. **Create Set Node "Get Hashes Only"**  
   - Type: `set`  
   - Use expression to extract hashes from descriptions using regex:  
     - `={{ $json.descriptions.map(desc => desc.match(/hash\:\s([\w#]+)/i)[1]) }}`  
   - Connect "Collect Descriptions" output to this node.

7. **Create Merge Node**  
   - Type: `merge`  
   - Mode: combine multiplex  
   - Connect "Get Values" output to first input, "Get Hashes Only" output to second input.

8. **Create If Node "Create New Ticket?"**  
   - Type: `if`  
   - Condition: Check if current Slack message id is NOT in hashes array:  
     - Expression: `={{ Boolean(($json.hashes ?? []).includes($json.id)) }} == false`  
   - Connect "Merge" node output to this node.  

9. **Create OpenAI Chat Model Node**  
   - Type: `lmChatOpenAi` (Langchain)  
   - No special options needed by default.  
   - Configure OpenAI API credentials with valid key.  
   - Connect "Create New Ticket?" true output to this node's input for AI processing.

10. **Create Structured Output Parser Node**  
    - Type: `outputParserStructured`  
    - JSON Schema:  
      ```json
      {
        "type": "object",
        "properties": {
          "title": { "type": "string" },
          "summary": { "type": "string" },
          "ideas": {
            "type": "array",
            "items": { "type": "string" }
          },
          "priority": { "type": "string" }
        }
      }
      ```  
    - Connect OpenAI Chat Model output(ai_languageModel) to this node's ai_outputParser input.

11. **Create Chain LLM Node "Generate Ticket Using ChatGPT"**  
    - Type: `chainLlm`  
    - Prompt Type: `define`  
    - Text prompt (use expression referencing "Get Values" message field):  
      ```
      The "user issue" is enclosed by 3 backticks:
      ```
      {{ $('Get Values').item.json.message }}
      ```
      You will complete the following 4 tasks:
      1. Generate a title intended for a support ticket based on the user issue only. Be descriptive but use no more than 10 words.
      2. Summarise the user issue only by identifying the key expectations and steps that were taken to reach the conclusion.
      3. Offer at most 3 suggestions to debug or resolve the user issue only. ignore the previous issues for this task.
      4. Identify the urgency of the user issue only and denote the priority as one of "low", "medium", "high" or "urgent". If you cannot determine the urgency of the issue, then assign the "low" priority. Also consider that requests which require action either today or tomorrow should be prioritised as "high".
      ```  
    - Set "Has Output Parser" to true, referencing the Structured Output Parser node.  
    - Connect "Create New Ticket?" node true output to this node's input.  
    - Connect OpenAI Chat Model output to this node’s ai_languageModel input.  
    - Connect Structured Output Parser output to this node's ai_outputParser input.

12. **Create Linear Node "Create Ticket"**  
    - Type: `linear`  
    - Operation: `create`  
    - Parameters:  
      - Title: `={{ $json.output.title }}`  
      - Team ID: Set to your Linear Team's GUID (must be configured)  
      - Additional fields:  
        - State ID: Set to the desired ticket state GUID  
        - Priority ID: Map priority string to Linear priority number using expression:  
          ```
          ={{ { 'urgent': 1, 'high': 2, 'medium': 3, 'low': 4 }[$json.output.priority.toLowerCase()] ?? 0 }}
          ```  
        - Description: Markdown formatted combining:  
          - Summary and suggestions from AI output  
          - Original user message with user info  
          - Metadata block with Slack channel, timestamp, permalink, and unique hash (message id)  
    - Connect "Generate Ticket Using ChatGPT" main output to this node.  
    - Configure Linear API credentials.

13. **Add Sticky Notes**  
    - Add notes explaining each block's purpose, required configurations (Slack channel, Linear team/state IDs), and useful links such as:  
      - Slack API docs  
      - Linear node docs  
      - Langchain node docs  
      - Workflow blog post: [https://blog.n8n.io/automated-customer-support-tickets-with-n8n-slack-linear-and-ai/](https://blog.n8n.io/automated-customer-support-tickets-with-n8n-slack-linear-and-ai/)  
      - Community support links

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| This workflow demonstrates a simple, effective customer support pipeline integrating Slack, Linear, and AI.                    | Blog post: https://blog.n8n.io/automated-customer-support-tickets-with-n8n-slack-linear-and-ai/                       |
| Slack API search uses the same syntax as in the Slack app; filtering messages with specific emoji reactions is key.             | Slack API Docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack                               |
| Linear integration enables automated ticket management for small teams; team and state IDs must be configured.                 | Linear Docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.linear                                 |
| AI prompt instructs ChatGPT to generate concise title, summary, suggestions, and priority for each support ticket.             | Langchain Docs: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm         |
| Join the n8n Discord or Forum for help and community support.                                                                    | Discord: https://discord.com/invite/XPKeKXeB7d, Forum: https://community.n8n.io/                                     |
| Workflow can be adapted for other ticketing systems like JIRA with minor modifications.                                          |                                                                                                                      |

---

This detailed reference documents all components, logic flows, configurations, and edge considerations to understand, reproduce, and adapt the workflow effectively.