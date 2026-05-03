Weekly SEO Keyword Research & Competitor Analysis with GPT-4 and Bright Data

https://n8nworkflows.xyz/workflows/weekly-seo-keyword-research---competitor-analysis-with-gpt-4-and-bright-data-5945


# Weekly SEO Keyword Research & Competitor Analysis with GPT-4 and Bright Data

### 1. Workflow Overview

This workflow automates weekly SEO keyword research and competitor analysis centered on a given topic, leveraging GPT-4 for AI-driven keyword discovery and Bright Data’s MCP search tool for real-time data. It processes and structures trending keywords and descriptions, then appends them into a Google Sheet for ongoing tracking.

The workflow is logically divided into three main blocks:

- **1.1 Trigger & Input Setup:** Scheduled weekly trigger initiates the workflow and sets the research topic.
- **1.2 AI Agent & Search Processing:** An AI agent uses GPT-4 and the MCP search tool to generate trending keywords and descriptions, applying output parsing to structure the data.
- **1.3 Data Persistence:** The structured keyword data is appended to a Google Sheet for record-keeping and further analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Setup

- **Overview:**  
  This block schedules the workflow to run weekly and defines the core input — the topic or competitor to research keywords for.

- **Nodes Involved:**  
  - 🕒 Run Weekly (Schedule Trigger)  
  - ✏️ Define Topic or competitor (Set)

- **Node Details:**

  1. **🕒 Run Weekly**  
     - Type: Schedule Trigger  
     - Role: Initiates the workflow automatically every Monday at 9 AM.  
     - Configuration: Interval set to weekly, triggering on day 1 (Monday) at hour 9.  
     - Inputs: None (trigger node)  
     - Outputs: Sends trigger signal to the next node.  
     - Edge cases: Timezone mismatches may cause unexpected trigger times; verify server timezone settings.  
     - Version-specific notes: Uses n8n version 1.2 for schedule trigger type.

  2. **✏️ Define Topic or competitor**  
     - Type: Set  
     - Role: Defines the research topic as a workflow variable.  
     - Configuration: Assigns a string field `Topic` with default value `"AI blogging"`. This can be changed manually or dynamically.  
     - Inputs: Trigger from schedule node.  
     - Outputs: Passes the topic data to the AI agent.  
     - Edge cases: If Topic is empty or null, downstream AI requests may fail or return irrelevant data. Validate input before running.  
     - Version-specific notes: Uses n8n version 3.4 for Set node.

---

#### 2.2 AI Agent & Search Processing

- **Overview:**  
  This block uses an AI agent combining GPT-4 chat and Bright Data’s MCP search tool to generate trending keywords and descriptions related to the topic. Output parsers ensure the AI returns clean, structured JSON data for downstream use.

- **Nodes Involved:**  
  - 🤖 AI Agent (Keyword Finder) (Langchain Agent)  
  - 💬 GPT Brain (OpenAI Chat Model)  
  - 🔍 MCP Keyword Search (MCP Client Tool)  
  - Auto-fixing Output Parser (Langchain Output Parser Autofixing)  
  - Structured Output Parser (Langchain Output Parser Structured)  
  - OpenAI Chat Model (Sub-node inside AI Agent)  
  - Structured Output Parser (Sub-node inside AI Agent)  

- **Node Details:**

  1. **🤖 AI Agent (Keyword Finder)**  
     - Type: Langchain Agent  
     - Role: Core AI orchestrator that sends prompts to GPT-4 and MCP search tool, receives results, and ensures output parsing.  
     - Configuration:  
       - Prompt text dynamically incorporates the Topic field (`Provide me trending keywords for the topic below.\n{{ $json.Topic }}`).  
       - Uses sub-nodes for language model, tool execution, and output parsing.  
     - Inputs: Receives Topic from Set node.  
     - Outputs: Structured JSON with topic and trending keywords to Google Sheets node.  
     - Edge cases:  
       - OpenAI API rate limits or downtime can cause failures.  
       - MCP tool authentication or quota issues may disrupt data retrieval.  
       - Parsing errors if AI response deviates from expected JSON format.  
     - Version-specific notes: Version 2 of Langchain Agent node; requires compatible Langchain integration in n8n.

  2. **💬 GPT Brain**  
     - Type: Langchain OpenAI Chat Model  
     - Role: Executes GPT-4o-mini model calls to generate or refine keyword queries.  
     - Configuration:  
       - Model set to `"gpt-4o-mini"`.  
       - Uses OpenAI API credentials.  
     - Inputs: Connected as language model sub-node of AI Agent.  
     - Outputs: Raw AI text responses.  
     - Edge cases: Possible timeout or API errors; ensure valid OpenAI API key and usage limits.  
     - Version-specific notes: Uses n8n v1.2 Langchain integration.

  3. **🔍 MCP Keyword Search**  
     - Type: MCP Client Tool  
     - Role: Executes "search_engine" tool operations to retrieve live search data related to keywords.  
     - Configuration: Tool parameters are dynamically generated by AI output (indicated by expression placeholder).  
     - Inputs: Connected as AI tool sub-node of AI Agent.  
     - Outputs: Search result data back to AI Agent.  
     - Edge cases: Authentication failures with MCP API; invalid tool parameters causing no results; network timeouts.  
     - Version-specific notes: Version 1.

  4. **Auto-fixing Output Parser**  
     - Type: Langchain Output Parser Autofixing  
     - Role: Automatically attempts to fix malformed AI output JSON before final parsing.  
     - Configuration: Default options; no user modifications.  
     - Inputs: Receives raw AI chat output from OpenAI Chat Model.  
     - Outputs: Cleaned output for structured parser.  
     - Edge cases: Complex AI outputs that are too corrupted may not be recoverable.  
     - Version-specific notes: Version 1.

  5. **Structured Output Parser**  
     - Type: Langchain Output Parser Structured  
     - Role: Parses JSON output from AI ensuring it matches expected schema.  
     - Configuration: Example JSON schema used for validation includes fields: `topic`, `keyword`, and `description`.  
     - Inputs: Receives cleaned output from Auto-fixing Output Parser.  
     - Outputs: Structured JSON for downstream processing.  
     - Edge cases: Outputs not conforming to schema cause parsing failures; user should ensure prompt leads to well-formed JSON.  
     - Version-specific notes: Version 1.2.

---

#### 2.3 Data Persistence

- **Overview:**  
  This block saves the structured keyword data into a Google Sheet, appending new rows each week for ongoing tracking of trends.

- **Nodes Involved:**  
  - 📄 Save to Google Sheets (Google Sheets node)

- **Node Details:**

  1. **📄 Save to Google Sheets**  
     - Type: Google Sheets  
     - Role: Appends rows containing Topic and Trending Keyword description into a specified Google Sheet.  
     - Configuration:  
       - Operation: Append  
       - Sheet Name: `gid=0` (Sheet1)  
       - Document ID: Specified Google Sheet document ID  
       - Columns mapped:  
         - `Topic` ← `$json.output.topic`  
         - `Trending Keyword` ← `$json.output.description`  
     - Inputs: Receives structured JSON output from AI Agent.  
     - Outputs: None (end node).  
     - Edge cases:  
       - Authentication failures with Google Sheets OAuth2.  
       - API quota limits or permission errors.  
       - Data type mismatches (although conversion is disabled, so careful mapping required).  
     - Version-specific notes: Version 4.6 Google Sheets node.

---

### 3. Summary Table

| Node Name                    | Node Type                          | Functional Role                      | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                   |
|------------------------------|----------------------------------|------------------------------------|-----------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| 🕒 Run Weekly                 | Schedule Trigger                 | Starts workflow weekly             | None                  | ✏️ Define Topic or competitor | See Section 1 notes: Weekly trigger setup                                                     |
| ✏️ Define Topic or competitor | Set                              | Defines research topic             | 🕒 Run Weekly          | 🤖 AI Agent (Keyword Finder) | See Section 1 notes: Topic input configuration                                                |
| 🤖 AI Agent (Keyword Finder)   | Langchain Agent                  | AI keyword generation & search    | ✏️ Define Topic        | 📄 Save to Google Sheets      | See Section 2 notes: AI Agent with GPT + MCP search + output parsing                         |
| 💬 GPT Brain                  | Langchain OpenAI Chat Model      | GPT-4 calls inside AI Agent        | 🤖 AI Agent            | 🤖 AI Agent                  | Sub-node within AI Agent                                                                    |
| 🔍 MCP Keyword Search         | MCP Client Tool                  | Live search for trending keywords | 🤖 AI Agent            | 🤖 AI Agent                  | Sub-node within AI Agent                                                                    |
| Auto-fixing Output Parser     | Langchain Output Parser Autofixing| Cleans AI output                  | OpenAI Chat Model      | Structured Output Parser     | Sub-node within AI Agent                                                                    |
| Structured Output Parser      | Langchain Output Parser Structured| Parses AI JSON output             | Auto-fixing Output Parser| 🤖 AI Agent                  | Sub-node within AI Agent                                                                    |
| 📄 Save to Google Sheets      | Google Sheets                   | Appends data to Google Sheet       | 🤖 AI Agent            | None                        | See Section 3 notes: Data persistence                                                       |
| Sticky Note                  | Sticky Note                     | Documentation                     | None                  | None                        | Covers Section 1: Trigger & Input Setup                                                    |
| Sticky Note1                 | Sticky Note                     | Documentation                     | None                  | None                        | Covers Section 2: AI Agent & Search Tool                                                  |
| Sticky Note2                 | Sticky Note                     | Documentation                     | None                  | None                        | Covers Section 3: Save to Google Sheets                                                   |
| Sticky Note3                 | Sticky Note                     | Affiliate note                    | None                  | None                        | Bright Data affiliate link                                                                 |
| Sticky Note4                 | Sticky Note                     | Full workflow explanation         | None                  | None                        | Comprehensive workflow overview and beginner instructions                                  |
| Sticky Note9                 | Sticky Note                     | Workflow assistance contact info  | None                  | None                        | Support and social media links                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Configure:  
     - Interval: Weekly  
     - Trigger day: Monday (day 1)  
     - Trigger hour: 9 AM  
   - No credentials needed.

2. **Add Set Node to Define Topic:**  
   - Type: Set  
   - Add field: `Topic` (string)  
   - Set default value to `"AI blogging"` (modifiable by user)  
   - Connect output of Schedule Trigger to this node.

3. **Add Langchain Agent Node (AI Agent):**  
   - Type: Langchain Agent  
   - Set prompt text to:  
     ```
     Provide me trending keywords for the topic below.
     {{ $json.Topic }}
     ```  
   - No additional options required.  
   - Connect output of Set node to AI Agent node.

4. **Within the AI Agent Node, configure sub-nodes:**  
   - **OpenAI Chat Model:**  
     - Type: Langchain OpenAI Chat Model  
     - Model: `gpt-4o-mini`  
     - Credentials: Connect valid OpenAI API credentials.  
   - **MCP Keyword Search Tool:**  
     - Type: MCP Client Tool  
     - Tool name: `search_engine`  
     - Operation: `executeTool`  
     - Tool parameters: leave dynamic (to be generated by AI output).  
     - Credentials: Add MCP Client API credentials with appropriate permissions.  
   - **Auto-fixing Output Parser:**  
     - Type: Langchain Output Parser Autofixing  
     - Default settings.  
   - **Structured Output Parser:**  
     - Type: Langchain Output Parser Structured  
     - Provide example JSON schema:  
       ```json
       {
         "topic": "AI Blogging",
         "keyword": "Artificial Intelligence",
         "description": "General discussions about AI technologies."
       }
       ```  
   - Connect these sub-nodes accordingly:  
     - OpenAI Chat Model output → Auto-fixing Output Parser input  
     - Auto-fixing Output Parser output → Structured Output Parser input  
     - Structured Output Parser output → AI Agent output

5. **Add Google Sheets Node:**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Use your Google Sheets document ID  
   - Sheet Name: Use `gid=0` or your target sheet name  
   - Columns mapping:  
     - `Topic` ← `{{$json.output.topic}}`  
     - `Trending Keyword` ← `{{$json.output.description}}`  
   - Set Google Sheets OAuth2 credentials with appropriate permissions.  
   - Connect AI Agent node output to this node.

6. **Set Active and Save Workflow:**  
   - Ensure all credentials are valid and tested.  
   - Activate the workflow to run automatically every Monday at 9 AM.

---

### 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow assistance and support contact: Yaron@nofluff.online                                                                           | Contact for help and questions                                                                     |
| YouTube channel with additional tips and tutorials: https://www.youtube.com/@YaronBeen/videos                                            | Additional learning resources                                                                       |
| LinkedIn profile for professional updates: https://www.linkedin.com/in/yaronbeen/                                                       | Networking and support                                                                              |
| Bright Data affiliate link (commission support): https://get.brightdata.com/1tndi4600b25                                                  | Affiliate link to support workflow creator                                                         |
| This workflow requires valid OpenAI and MCP Client credentials configured in n8n credentials section                                     | Credential setup necessary for AI and live data search                                             |
| Google Sheets OAuth2 credentials must have permission to append rows in the target spreadsheet                                          | Ensure Google Sheets API access is granted                                                         |
| Prompt design must ensure AI returns well-formed JSON matching the schema to avoid parsing errors                                         | Critical for successful output parsing                                                             |
| The workflow is beginner-friendly: no code needed, only configuration of topic and credentials                                           | Suitable for users with minimal programming experience                                             |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.