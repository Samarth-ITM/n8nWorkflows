Personalized LinkedIn Responses with OpenAI GPT & Notion-based Routing

https://n8nworkflows.xyz/workflows/personalized-linkedin-responses-with-openai-gpt---notion-based-routing-4891


# Personalized LinkedIn Responses with OpenAI GPT & Notion-based Routing

### 1. Workflow Overview

This workflow automates the generation of personalized LinkedIn message responses using OpenAI's GPT models, enhanced with contextual routing information from a Notion database. It is designed for business professionals or teams handling inbound LinkedIn messages seeking to streamline and scale their engagement with contacts while maintaining a warm, personalized tone.

The workflow consists of three main logical blocks:

- **1.1 Input Reception and Data Isolation:** Receives input parameters from a parent workflow, isolating and preparing message and sender data for downstream processing.

- **1.2 Context Enrichment with Notion Database:** Queries a Notion database containing past request types and corresponding preferred responses, formats this data, and aggregates it into a single context object to inform AI-generated replies.

- **1.3 AI-Powered Response Generation:** Utilizes a LangChain AI Agent powered by OpenAI GPT-4o, supplemented with session-based memory, to generate tailored LinkedIn message replies that consider the sender’s LinkedIn profile data and the enriched context from the database.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Isolation

- **Overview:**  
  This block receives the LinkedIn message data from an external trigger (another workflow), isolates key fields (sender, message text, chat ID), and prepares them for enrichment and AI processing.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - Isolate parent workflow data for AI

- **Node Details:**

  - **When Executed by Another Workflow**  
    - *Type & Role:* Execute Workflow Trigger node; entry point that accepts input parameters from a parent workflow.  
    - *Configuration:* Expects inputs named `message`, `sender`, `chatid`, and `linkedinprofile` (the latter as an array).  
    - *Expressions/Variables:* Directly passes incoming JSON data.  
    - *Connections:* Outputs to "Isolate parent workflow data for AI".  
    - *Edge Cases:* Input missing required fields, or improperly formatted `linkedinprofile` array could cause downstream errors.

  - **Isolate parent workflow data for AI**  
    - *Type & Role:* Set node; extracts and renames relevant input fields for uniform downstream use.  
    - *Configuration:* Assigns `sender`, `message`, and `chatid` from incoming JSON to dedicated variables.  
    - *Expressions/Variables:* Uses JSON path expressions `={{ $json.<field> }}` to extract data.  
    - *Connections:* Outputs to "Get Request Router Directory Database".  
    - *Edge Cases:* Missing or null input fields could result in empty variables affecting AI context or session key generation.

#### 2.2 Context Enrichment with Notion Database

- **Overview:**  
  This block retrieves routing and response data from a Notion database, formats each database page into a structured text snippet, and aggregates all snippets into one composite object for AI context.

- **Nodes Involved:**  
  - Get Request Router Directory Database  
  - Format DB data for AI Context  
  - Aggregate DB objects into one item

- **Node Details:**

  - **Get Request Router Directory Database**  
    - *Type & Role:* Notion node; fetches all pages from a specified Notion database containing past request-response mappings.  
    - *Configuration:* Operation `getAll` on a fixed database ID.  
    - *Credentials:* Requires valid Notion API credentials.  
    - *Connections:* Outputs raw database pages to "Format DB data for AI Context".  
    - *Edge Cases:* API rate limits, invalid credentials, or database ID issues cause failure or incomplete data retrieval.

  - **Format DB data for AI Context**  
    - *Type & Role:* Set node; formats each Notion page into a string summarizing request examples, descriptions, and actions to create AI-readable context entries.  
    - *Configuration:* Uses template strings combining database page properties such as `name`, `property_request_description`, and `property_request_action`.  
    - *Expressions/Variables:* Uses expressions like `=Request Example: {{ $json.name }}` to format text.  
    - *Connections:* Outputs formatted text to "Aggregate DB objects into one item".  
    - *Edge Cases:* Missing or malformed database page properties could produce incomplete or confusing AI context snippets.

  - **Aggregate DB objects into one item**  
    - *Type & Role:* Aggregate node; concatenates all formatted database entries into a single JSON array field (`dbObject`) to feed into the AI Agent.  
    - *Configuration:* Aggregates over the `dbObject` field.  
    - *Connections:* Outputs aggregated context to "AI Agent".  
    - *Edge Cases:* Large datasets may exceed payload size limits or cause performance degradation.

#### 2.3 AI-Powered Response Generation

- **Overview:**  
  This block uses an AI Agent powered by LangChain and OpenAI GPT-4o to generate personalized LinkedIn responses based on the sender’s message, LinkedIn profile data, enriched context from the Notion database, and session-based memory to maintain conversational continuity.

- **Nodes Involved:**  
  - AI Agent  
  - Simple Memory  
  - OpenAI Chat Model  
  - Structured Output Parser

- **Node Details:**

  - **AI Agent**  
    - *Type & Role:* LangChain AI Agent node; central logic that composes the prompt and manages AI completion requests.  
    - *Configuration:*  
      - Prompt includes sender name, message text, and LinkedIn profile array serialized as JSON string.  
      - System message defines the assistant’s role, tone, and output format, emphasizing friendly, confident, professional responses that refer to the Notion database for existing templates.  
      - Output parser expects a JSON object with `output` (message text) and `found` (boolean indicating if a matched template was used).  
    - *Expressions/Variables:* Uses expressions to dynamically inject context from previous nodes such as `$('Isolate parent workflow data for AI').item.json.sender` and the aggregated `dbObject`.  
    - *Connections:*  
      - Input memory from "Simple Memory".  
      - Sends language model requests to "OpenAI Chat Model".  
      - Receives parsed outputs from "Structured Output Parser".  
    - *Version-Specific Requirements:* Requires LangChain integration and compatible n8n version supporting agent nodes (>= 1.9).  
    - *Edge Cases:*  
      - API quota or rate limits.  
      - Parsing errors if AI output does not conform to expected JSON.  
      - Missing or malformed input variables leading to incoherent prompts.

  - **Simple Memory**  
    - *Type & Role:* LangChain memory node; maintains a windowed conversational memory keyed by chat ID to provide context continuity across exchanges.  
    - *Configuration:* Session key sourced from `chatid` isolated earlier; uses custom key session ID type.  
    - *Connections:* Feeds memory into the AI Agent node.  
    - *Edge Cases:* Memory overload or session key collisions could affect context relevance.

  - **OpenAI Chat Model**  
    - *Type & Role:* LangChain OpenAI Chat model node; performs GPT-4o API calls to generate responses.  
    - *Configuration:* Uses GPT-4o model variant with default options.  
    - *Credentials:* Requires valid OpenAI API credentials scoped for marketing or business use.  
    - *Connections:* Receives prompt from AI Agent, outputs raw model response back to AI Agent.  
    - *Edge Cases:* API limits, network errors, or invalid credentials cause failures.

  - **Structured Output Parser**  
    - *Type & Role:* LangChain output parser; validates and extracts structured JSON from the raw AI model output.  
    - *Configuration:* Expects JSON matching schema with `output` string and `found` boolean.  
    - *Connections:* Outputs parsed result to AI Agent.  
    - *Edge Cases:* If AI output is malformed or incomplete, parser fails, causing downstream errors.

---

### 3. Summary Table

| Node Name                          | Node Type                            | Functional Role                                    | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                                                             |
|-----------------------------------|------------------------------------|---------------------------------------------------|----------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger            | Receives input from parent workflow                |                                  | Isolate parent workflow data for AI |                                                                                                                                         |
| Isolate parent workflow data for AI | Set                              | Extracts and renames key input fields               | When Executed by Another Workflow | Get Request Router Directory Database |                                                                                                                                         |
| Get Request Router Directory Database | Notion                           | Retrieves past request-response mappings from Notion | Isolate parent workflow data for AI | Format DB data for AI Context     | ![notion](https://uploads.n8n.io/templates/notionlogo.png)\n## Get routing Database data\nAn AI bot that does not get smarter over time is not a useful bot...|
| Format DB data for AI Context     | Set                               | Formats Notion DB pages into AI-readable strings    | Get Request Router Directory Database | Aggregate DB objects into one item |                                                                                                                                         |
| Aggregate DB objects into one item | Aggregate                        | Aggregates formatted DB entries into single context | Format DB data for AI Context     | AI Agent                        |                                                                                                                                         |
| AI Agent                         | LangChain AI Agent                 | Generates personalized LinkedIn replies using AI    | Aggregate DB objects into one item, Simple Memory, OpenAI Chat Model, Structured Output Parser | Simple Memory, OpenAI Chat Model, Structured Output Parser | ![hctiapi](https://uploads.n8n.io/templates/openai.png)\n## Generate Automated response\nThe AI bot receives not only the message from LinkedIn...  |
| Simple Memory                   | LangChain Memory Buffer Window     | Maintains conversational memory keyed by chat ID    | AI Agent                        | AI Agent                        |                                                                                                                                         |
| OpenAI Chat Model               | LangChain OpenAI Chat Model        | Performs GPT-4o API calls for text generation       | AI Agent                        | AI Agent                        |                                                                                                                                         |
| Structured Output Parser        | LangChain Output Parser Structured | Parses AI output JSON into structured data           | AI Agent                        | AI Agent                        |                                                                                                                                         |
| Sticky Note3                    | Sticky Note                       | Documentation note describing AI response logic      |                                  |                                  | ![hctiapi](https://uploads.n8n.io/templates/openai.png)\n## Generate Automated response\nThe AI bot receives not only the message from LinkedIn...  |
| Sticky Note9                    | Sticky Note                       | Documentation note describing Notion DB routing logic |                                  |                                  | ![notion](https://uploads.n8n.io/templates/notionlogo.png)\n## Get routing Database data\nAn AI bot that does not get smarter over time is not a useful bot...|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add an **Execute Workflow Trigger** node named "When Executed by Another Workflow".  
   - Configure inputs: `message` (string), `sender` (string), `chatid` (string), `linkedinprofile` (array).  

2. **Add Data Isolation Node:**  
   - Add a **Set** node named "Isolate parent workflow data for AI".  
   - Map incoming JSON fields to variables:  
     - `sender` = `{{$json.sender}}`  
     - `message` = `{{$json.message}}`  
     - `chatid` = `{{$json.chatid}}`  
   - Connect output from trigger node.

3. **Add Notion Database Retrieval Node:**  
   - Add a **Notion** node named "Get Request Router Directory Database".  
   - Set operation to `getAll` for resource `databasePage`.  
   - Enter the Notion database ID: `1da5b6e0-c94f-8086-80b6-eab8e340d60e`.  
   - Assign valid Notion API credentials (e.g., "Angelbot Notion").  
   - Connect output from data isolation node.

4. **Add Data Formatting Node:**  
   - Add a **Set** node named "Format DB data for AI Context".  
   - Add one assignment:  
     - Name: `dbObject`  
     - Type: string  
     - Value:  
       ```
       =Request Example: {{ $json.name }}
       --
       Request Description: {{ $json.property_request_description }}
       --
       Request Action: {{ $json.property_request_action }}
       ```  
   - Connect output from Notion node.

5. **Add Aggregation Node:**  
   - Add an **Aggregate** node named "Aggregate DB objects into one item".  
   - Configure to aggregate field `dbObject` into an array.  
   - Connect output from formatting node.

6. **Add AI Agent Node:**  
   - Add a **LangChain AI Agent** node named "AI Agent".  
   - Configure the prompt text to include:  
     ```
     Sender Name: {{ $('Isolate parent workflow data for AI').item.json.sender }}
     Sender Message: {{ $('Isolate parent workflow data for AI').item.json.message }}
     My Name: Your Name

     If found, the linkedin profile will be below, pay close attention to the number of followers and connections: 
     {{ $('When Executed by Another Workflow').item.json.linkedinprofile.toJsonString() }}
     ```  
   - Set the system message as a detailed instruction for professional, friendly LinkedIn responses referencing the `dbObject` context.  
   - Enable output parsing with expected JSON schema:  
     ```json
     {
       "output": "message response",
       "found": true
     }
     ```  
   - Connect input from aggregation node.

7. **Add Simple Memory Node:**  
   - Add a **LangChain memoryBufferWindow** node named "Simple Memory".  
   - Configure session key as `={{ $('Isolate parent workflow data for AI').item.json.chatid }}`  
   - Use `customKey` as session ID type.  
   - Connect memory input/output to AI Agent node (AI memory input).

8. **Add OpenAI Chat Model Node:**  
   - Add a **LangChain OpenAI Chat Model** node named "OpenAI Chat Model".  
   - Select model `gpt-4o`.  
   - Assign OpenAI API credentials (e.g., "Marketing OpenAI").  
   - Connect language model input/output to AI Agent node.

9. **Add Structured Output Parser Node:**  
   - Add a **LangChain Output Parser Structured** node named "Structured Output Parser".  
   - Provide example JSON schema for parsing AI output (as above).  
   - Connect output parser to AI Agent node.

10. **Link Connections:**  
    - Connect "When Executed by Another Workflow" → "Isolate parent workflow data for AI".  
    - "Isolate parent workflow data for AI" → "Get Request Router Directory Database".  
    - "Get Request Router Directory Database" → "Format DB data for AI Context".  
    - "Format DB data for AI Context" → "Aggregate DB objects into one item".  
    - "Aggregate DB objects into one item" → "AI Agent".  
    - "Simple Memory" → AI Agent (memory input).  
    - "OpenAI Chat Model" → AI Agent (language model input).  
    - "Structured Output Parser" → AI Agent (output parser input).  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The AI bot leverages LinkedIn user metadata such as number of followers and connections to tailor replies, e.g., prioritizing influencers with booking links and politely declining low-priority requests. This ensures efficient handling of inbound messages based on sender profile.                                                                                                                         | See Sticky Note3 content with image ![hctiapi](https://uploads.n8n.io/templates/openai.png) |
| The Notion database acts as a centralized “Request Router” that can be updated by multiple team members, allowing the AI to improve over time without workflow edits. This approach can be adapted for various platforms such as shared email inboxes and social media, beyond LinkedIn.                                                                                                                   | See Sticky Note9 content with image ![notion](https://uploads.n8n.io/templates/notionlogo.png) |
| The AI Agent’s system message carefully instructs the model to avoid robotic or sales-pitch tones, to acknowledge senders personally, and to avoid referencing URLs when none are available in the request database. This ensures more natural, human-like LinkedIn communications.                                                                                                                         | Embedded in AI Agent node configuration                         |

---

*Disclaimer: The provided text derives exclusively from an n8n automated workflow. It complies strictly with applicable content policies and contains no illegal, offensive, or protected materials. All processed data is legal and publicly accessible.*