Generate Dream 100 Prospect Lists with Perplexity AI Research and Google Sheets

https://n8nworkflows.xyz/workflows/generate-dream-100-prospect-lists-with-perplexity-ai-research-and-google-sheets-8466


# Generate Dream 100 Prospect Lists with Perplexity AI Research and Google Sheets

### 1. Workflow Overview

This workflow, titled **"Generate Dream 100 Prospect Lists with Perplexity AI Research and Google Sheets,"** automates the generation of a curated list of 100 ideal prospects ("Dream 100") for a business owner using AI-driven research and Google Sheets for data storage. It is designed for business owners or marketers who want to identify their most valuable potential customers based on detailed criteria.

The workflow features the following logical blocks:

- **1.1 Input Reception and Message Handling:** Receiving user input via chat or Slack and preparing messages for AI processing.
- **1.2 AI Research Agent:** Using a Langchain AI agent combined with OpenRouter chat model and Perplexity AI to research and generate prospect data.
- **1.3 Google Sheets Data Management:** Reading and writing prospect data into a Google Sheets spreadsheet, including checks for data presence and completeness.
- **1.4 Control Logic for Data Completeness:** Conditional logic that determines whether to continue researching more prospects or finalize the process.
- **1.5 User Communication:** Sending messages back to the user (via chat or Slack) regarding progress or completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Message Handling

**Overview:**  
This block captures the initial user input, formats it for processing, and ensures that messages are correctly routed depending on their content.

**Nodes Involved:**  
- When chat message received  
- User Chat Message  
- Agent Re-trigger chat message  
- Send correct message  
- When Slack Received (Slack trigger)  

**Node Details:**  

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Entry point for user chat messages into workflow  
  - Configuration: Listens for chat messages, responds with nodes rather than raw data  
  - Input/Output: None input (trigger), outputs chat message JSON  
  - Edge cases: Missing or malformed chat input may cause empty messages  

- **User Chat Message**  
  - Type: Set node  
  - Role: Extracts and assigns user's chat input to variable "User Chat Message"  
  - Configuration: Assigns `={{ $json.chatInput }}` to `User Chat Message`  
  - Input: Output of chat trigger  
  - Output: JSON with `User Chat Message` string  
  - Edge cases: Empty or missing chat input leads to empty string assignment  

- **Agent Re-trigger chat message**  
  - Type: Set node  
  - Role: Prepares a message to ask the AI agent for the next batch of prospects (next 10 entries)  
  - Configuration: Sets the string instructing the agent to provide next 10 entries with incremented Prospect IDs and no duplicates, assigned to `Retrigger Chat`  
  - Input: None (triggered conditionally)  
  - Output: JSON with `Retrigger Chat` string  
  - Edge cases: Ensures no duplicate prospect IDs requested  

- **Send correct message**  
  - Type: Switch  
  - Role: Routes processing depending on presence of `User Chat Message` or `Retrigger Chat` content  
  - Configuration: Two conditions; if `Retrigger Chat` is not empty, or if `User Chat Message` is not empty, proceeds accordingly  
  - Input: JSON with both message variables  
  - Output: Routes to AI Agent node  
  - Edge cases: Both messages empty will cause no output path; strict string validation used  

- **When Slack Received**  
  - Type: Slack Trigger  
  - Role: Alternate entry point triggered by Slack message in a channel  
  - Configuration: Listens for Slack messages, channel ID unspecified (must be configured)  
  - Edge cases: Slack API errors, missing channel ID configuration  

---

#### 2.2 AI Research Agent

**Overview:**  
This block handles the AI-driven research to generate prospect lists based on user input. It uses Langchain’s AI agent with OpenRouter chat model and Perplexity research tool for deep research and outputs structured prospect data.

**Nodes Involved:**  
- AI Agent  
- OpenRouter Chat Model  
- Simple Memory  
- Research via Perplexity  
- Insert Data Into Sheet  

**Node Details:**  

- **AI Agent**  
  - Type: Langchain Agent Node  
  - Role: Core AI "brain" that processes user input, manages conversation context, and orchestrates research  
  - Configuration:  
    - Input text merges `User Chat Message` and `Retrigger Chat` for continuity  
    - System message instructs the agent on how to identify the Dream 100 prospects, specifying detailed customer profile requirements and output formatting for Google Sheets  
    - Uses OpenRouter Chat Model as language model backend  
    - Uses Perplexity AI for research tool calls  
  - Input: Text from message set nodes, memory buffer for conversation context  
  - Output: JSON containing AI-generated prospect data structured for Google Sheets insertion  
  - Edge cases: AI token limits, ambiguous user input, Perplexity API rate limits or incomplete data returns  
  - Version: Langchain agent v2.2  

- **OpenRouter Chat Model**  
  - Type: Language Model Node  
  - Role: Provides chat language model responses to the agent  
  - Configuration: Uses OpenRouter API credentials  
  - Input: Chat prompt from AI Agent  
  - Output: Chat completions for agent processing  
  - Edge cases: API authentication errors, network timeouts  

- **Simple Memory**  
  - Type: Memory Buffer Window  
  - Role: Maintains short-term conversational context for the AI Agent  
  - Configuration: Default buffer window, no special parameters  
  - Input: Feeds conversation history to AI Agent  
  - Edge cases: Memory overflow or context loss if conversation too long  

- **Research via Perplexity**  
  - Type: Perplexity AI tool node  
  - Role: Performs deep research calls to Perplexity AI to find prospects matching criteria  
  - Configuration:  
    - Model: sonar-deep-research  
    - Input: Message content generated by AI Agent's instructions  
  - Input: Research query from AI Agent  
  - Output: Research results parsed for prospect extraction  
  - Edge cases: Perplexity API limits, incomplete results (less than 100 prospects), retries necessary  

- **Insert Data Into Sheet**  
  - Type: Google Sheets Tool (append or update)  
  - Role: Inserts or updates prospect data rows in Google Sheets  
  - Configuration:  
    - Matches rows by "Prospect ID" column to avoid duplicates  
    - Columns mapped: Prospect ID, Prospect Name, Prospect Summary, Reason  
    - Document ID and Sheet Name explicitly set  
  - Input: Structured prospect data from AI Agent  
  - Output: Confirmation of data appended/updated  
  - Edge cases: Google Sheets API quotas, authentication errors, schema mismatches  

---

#### 2.3 Google Sheets Data Management

**Overview:**  
This block reads from the Google Sheet to determine existing data presence and to extract first and last prospect names for control logic.

**Nodes Involved:**  
- Check First Row  
- First Row Name  
- Check Last Row  
- Last Row Name  

**Node Details:**  

- **Check First Row**  
  - Type: Google Sheets node (read with filter)  
  - Role: Retrieves the first prospect row (Prospect ID = 1) to check if sheet has data  
  - Configuration: Filters on column "Prospect ID" equals "1"  
  - Input: Triggered by AI Agent output completion  
  - Output: JSON containing first prospect data if exists  
  - Edge cases: Sheet empty, filtering failure  

- **First Row Name**  
  - Type: Set node  
  - Role: Extracts "Prospect Name" from first row data and saves as `Prospect 1` variable  
  - Configuration: Assigns from previous node output field  
  - Input: Output of Check First Row  
  - Output: JSON with `Prospect 1` string  
  - Edge cases: Missing "Prospect Name" field  

- **Check Last Row**  
  - Type: Google Sheets node (read with filter)  
  - Role: Checks if the 100th prospect exists (Prospect ID = 100)  
  - Configuration: Filters on column "Prospect ID" equals "100"  
  - Input: Output of First Row Name node  
  - Output: JSON with last prospect data if present  
  - Edge cases: Sheet incomplete or missing last row  

- **Last Row Name**  
  - Type: Set node  
  - Role: Extracts "Prospect Name" from last row and saves as `Prospect 100` variable  
  - Configuration: Assigns from Check Last Row output  
  - Input: Output of Check Last Row  
  - Output: JSON with `Prospect 100` string  
  - Edge cases: Missing or empty last row data  

---

#### 2.4 Control Logic for Data Completeness

**Overview:**  
This block uses conditional logic nodes to determine whether the sheet is empty, partially filled, or complete, and controls whether more research is needed or the workflow is finished.

**Nodes Involved:**  
- Is there any data in the sheet?  
- Do we need MORE data in the sheet?  
- Agent Re-trigger chat message  
- Collect More Info  
- All Done!  

**Node Details:**  

- **Is there any data in the sheet?**  
  - Type: If node  
  - Role: Checks if both first and last prospect names are empty (sheet empty)  
  - Configuration: Condition tests if `Prospect 1` and `Prospect 100` are empty strings  
  - Input: Output of Last Row Name  
  - Output:  
    - True: No data present → triggers `Collect More Info` node  
    - False: Some data present → triggers next check node  
  - Edge cases: Data corruption or unexpected empty fields  

- **Do we need MORE data in the sheet?**  
  - Type: If node  
  - Role: Checks if first prospect exists but last prospect (100) is missing → sheet partially filled  
  - Configuration: Checks `Prospect 1` exists and `Prospect 100` empty  
  - Input: Output of Is there any data in the sheet? (False path)  
  - Output:  
    - True: Need more data → triggers agent message re-trigger for next batch  
    - False: Data complete → triggers `All Done!` node  
  - Edge cases: Partial data inconsistencies  

- **Agent Re-trigger chat message**  
  - (Previously detailed in 2.1)  
  - Role: Asks AI agent to provide next batch of prospects (increments Prospect ID)  

- **Collect More Info**  
  - Type: Langchain Chat node  
  - Role: Sends AI agent output message to user requesting additional info or signaling progress  
  - Configuration: Message content is output from AI Agent node  
  - Input: From "Is there any data in the sheet?" True branch  
  - Output: Sends message, no wait for user reply  

- **All Done!**  
  - Type: Langchain Chat node  
  - Role: Sends final message signaling completion of prospect list generation  
  - Configuration: Empty message (could be customized)  
  - Input: From "Do we need MORE data in the sheet?" False branch  
  - Output: Sends message, no wait for reply  

---

#### 2.5 User Communication (Slack Integration)

**Overview:**  
This block manages Slack messaging for user input reception and output delivery, providing an alternate interface to the chat trigger.

**Nodes Involved:**  
- When Slack Received (also in 2.1)  
- Collect More Info Slack  
- All Done Slack  

**Node Details:**  

- **Collect More Info Slack**  
  - Type: Slack node (message sending)  
  - Role: Sends AI agent's output (progress or info) to Slack channel/user  
  - Configuration: Sends message with content from AI Agent output  
  - Input: From "Collect More Info" node output  
  - Edge cases: Slack API errors, invalid channel IDs  

- **All Done Slack**  
  - Type: Slack node (message sending)  
  - Role: Sends final "All done" message to Slack  
  - Configuration: Sends message with content from "All Done!" node  
  - Input: From "All Done!" node output  

---

### 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                          | Input Node(s)                  | Output Node(s)                     | Sticky Note                                                                                                                       |
|----------------------------|-----------------------------------|----------------------------------------|-------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When chat message received  | Langchain Chat Trigger             | Entry point for user chat input        | -                             | User Chat Message                | Connect to Trigger via Slack Message. Change variable in "User Chat Message" from n8n chat -> slack message                      |
| User Chat Message           | Set                               | Assign user chat input to variable     | When chat message received     | Send correct message             | Pick a message to send to the agent                                                                                              |
| Agent Re-trigger chat message| Set                              | Prepare next batch request for AI      | Do we need MORE data in sheet? | Send correct message             | Pick a message to send to the agent                                                                                              |
| Send correct message        | Switch                           | Route based on which message is present| User Chat Message, Agent Re-trigger chat message| AI Agent                 | Pick a message to send to the agent                                                                                              |
| AI Agent                   | Langchain Agent                   | Core AI research and prospect generation| Send correct message           | Check First Row, Insert Data Into Sheet | The Brain - Takes your Prompt, Does Research                                                                                      |
| OpenRouter Chat Model       | Langchain LM Chat Model           | Provides chat completions for AI agent | AI Agent (language model input)| AI Agent                        | The Brain - Takes your Prompt, Does Research                                                                                      |
| Simple Memory               | Langchain Memory Buffer           | Maintains conversation context        | AI Agent                      | AI Agent                        | The Brain - Takes your Prompt, Does Research                                                                                      |
| Research via Perplexity     | Perplexity AI Tool                | Performs deep research for prospects   | AI Agent                      | AI Agent                        | The Brain - Takes your Prompt, Does Research                                                                                      |
| Insert Data Into Sheet      | Google Sheets Tool                | Append or update prospect data in Sheet| AI Agent                      | Check First Row                 | The Brain - Takes your Prompt, Does Research                                                                                      |
| Check First Row             | Google Sheets                    | Reads first prospect row (ID=1)        | AI Agent                      | First Row Name                  | Check how much data is in the sheet                                                                                              |
| First Row Name              | Set                              | Extracts first prospect name           | Check First Row               | Check Last Row                  | Check how much data is in the sheet                                                                                              |
| Check Last Row              | Google Sheets                    | Reads last prospect row (ID=100)       | First Row Name               | Last Row Name                  | Check how much data is in the sheet                                                                                              |
| Last Row Name               | Set                              | Extracts last prospect name            | Check Last Row               | Is there any data in the sheet? | Check how much data is in the sheet                                                                                              |
| Is there any data in the sheet? | If                         | Checks if sheet is empty                | Last Row Name                | Collect More Info, Do we need MORE data in the sheet? | Do more research or respond to user?                                                                                             |
| Do we need MORE data in the sheet? | If                     | Checks if more data needed (partial fill) | Is there any data in the sheet? | Agent Re-trigger chat message, All Done! | Do more research or respond to user?                                                                                             |
| Collect More Info           | Langchain Chat                   | Sends AI agent message to user         | Is there any data in the sheet? (True) | Collect More Info Slack        | Connect These to Respond via Slack Message. Connect AI agent's output as content of messages                                      |
| All Done!                  | Langchain Chat                   | Sends completion message to user       | Do we need MORE data in the sheet? (False) | All Done Slack               | Connect These to Respond via Slack Message. Connect AI agent's output as content of messages                                      |
| When Slack Received         | Slack Trigger                   | Entry point for Slack messages          | -                             | User Chat Message               | Connect to Trigger via Slack Message. Change variable in "User Chat Message" from n8n chat -> slack message                      |
| Collect More Info Slack     | Slack                          | Sends progress info to Slack            | Collect More Info             | -                              | Connect These to Respond via Slack Message. Connect AI agent's output as content of messages                                      |
| All Done Slack             | Slack                          | Sends completion message to Slack       | All Done!                    | -                              | Connect These to Respond via Slack Message. Connect AI agent's output as content of messages                                      |
| Sticky Note                 | Sticky Note                    | Instruction and documentation notes     | -                             | -                              | Pick a message to send to the agent                                                                                              |
| Sticky Note1                | Sticky Note                    | Instruction and documentation notes     | -                             | -                              | The Brain - Takes your Prompt, Does Research                                                                                      |
| Sticky Note2                | Sticky Note                    | Instruction and documentation notes     | -                             | -                              | Check how much data is in the sheet                                                                                              |
| Sticky Note3                | Sticky Note                    | Instruction and documentation notes     | -                             | -                              | Do more research or respond to user?                                                                                            |
| Sticky Note4                | Sticky Note                    | Instruction and documentation notes     | -                             | -                              | Connect to Trigger via Slack Message. Change variable in "User Chat Message" from n8n chat -> slack message                      |
| Sticky Note5                | Sticky Note                    | Instruction and documentation notes     | -                             | -                              | Connect These to Respond via Slack Message. Connect AI agent's output as content of messages                                      |
| Sticky Note6                | Sticky Note                    | Google Sheet template link               | -                             | -                              | Google Sheet Template (Make a copy): https://docs.google.com/spreadsheets/d/1WWRwi3bseuZd1ebaAdGQYzDhwW_zBEOYib20l2Z_coY/edit?usp=sharing |
| Sticky Note7                | Sticky Note                    | Setup tutorial video link                 | -                             | -                              | SETUP TUTORIAL LOOM VIDEO: https://www.loom.com/share/2a71542149054456b993826f56e5503f?sid=41a106b3-a16d-4686-8280-b7e56af8ce42 |
| Sticky Note8                | Sticky Note                    | Directions for usage                      | -                             | -                              | Directions: 1. Ensure Google sheet has NO NAMES; 2. Start chat; 3. Provide info; 4. Let agent research (~3-4 mins per 10 names)  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Langchain Chat Trigger** node named "When chat message received" to listen for user chat inputs.
   - Add a **Slack Trigger** node named "When Slack Received" to listen for Slack messages (configure the Slack channel).

2. **Set User Input Variables:**
   - Add a **Set** node called "User Chat Message" connected from "When chat message received" and "When Slack Received."
   - Configure it to assign `User Chat Message` = `={{ $json.chatInput }}` (chat trigger) or from Slack message text accordingly.

3. **Prepare Re-trigger Message:**
   - Add a **Set** node named "Agent Re-trigger chat message."
   - Configure it to set `Retrigger Chat` with instructions for the AI to fetch the next 10 prospects (incrementing Prospect IDs, no duplicates).

4. **Add Message Routing Logic:**
   - Add a **Switch** node named "Send correct message."
   - Configure two rules:
     - If `Retrigger Chat` not empty → output path 1.
     - Else if `User Chat Message` not empty → output path 2.
   - Connect both outputs to the AI Agent node.

5. **Configure AI Agent:**
   - Add a **Langchain Agent** node called "AI Agent."
   - Configure the `text` input as concatenation of `User Chat Message` and `Retrigger Chat`.
   - Add system message instructions detailing the Dream 100 prospect research criteria and output format for Google Sheets.
   - Set OpenRouter Chat Model as the language model.
   - Add Perplexity AI tool as the research tool.
   - Connect memory buffer node ("Simple Memory") as the agent's memory input.

6. **Setup Language Model:**
   - Add **Langchain OpenRouter Chat Model** node named "OpenRouter Chat Model."
   - Configure with OpenRouter API credentials.
   - Connect to AI Agent’s language model input.

7. **Setup Memory Buffer:**
   - Add **Langchain Memory Buffer Window** node named "Simple Memory."
   - Connect its output to AI Agent’s memory input.

8. **Configure Perplexity Research:**
   - Add **Perplexity AI Tool** node named "Research via Perplexity."
   - Set model to "sonar-deep-research."
   - Connect to AI Agent’s tool input.
   - Provide Perplexity API credentials.

9. **Insert Data Into Google Sheets:**
   - Add **Google Sheets Tool** node named "Insert Data Into Sheet."
   - Configure with your Google Sheets document ID and sheet name.
   - Map columns: Prospect ID, Prospect Name, Prospect Summary, Reason.
   - Enable append/update operation matching on "Prospect ID."
   - Provide Google Sheets OAuth2 credentials.
   - Connect AI Agent’s tool output to this node.

10. **Check Existing Data in Sheet:**
    - Add **Google Sheets** node "Check First Row" to filter where "Prospect ID" = 1.
    - Connect from "Insert Data Into Sheet."
    - Add a **Set** node "First Row Name" to extract "Prospect Name" as `Prospect 1`.
    - Add **Google Sheets** node "Check Last Row" to filter where "Prospect ID" = 100.
    - Connect from "First Row Name."
    - Add **Set** node "Last Row Name" to extract "Prospect Name" as `Prospect 100`.
    - Connect from "Check Last Row."

11. **Control Logic for Data Presence:**
    - Add **If** node "Is there any data in the sheet?" checking if both `Prospect 1` and `Prospect 100` are empty strings.
    - True branch connects to **Langchain Chat** node "Collect More Info" to inform user.
    - False branch connects to **If** node "Do we need MORE data in the sheet?" checking if `Prospect 1` exists and `Prospect 100` is empty.
    - True branch connects to "Agent Re-trigger chat message" (to fetch next batch).
    - False branch connects to **Langchain Chat** node "All Done!" to notify completion.

12. **Slack Message Outputs:**
    - Add **Slack** nodes "Collect More Info Slack" and "All Done Slack" to send AI agent messages to Slack.
    - Connect "Collect More Info" and "All Done!" Langchain Chat nodes to these Slack nodes respectively.

13. **Final Connections and Testing:**
    - Verify all nodes are correctly connected following the flow described.
    - Configure all credentials (OpenRouter API, Perplexity API, Google Sheets OAuth2, Slack OAuth).
    - Test with user input to ensure prospect list is generated, stored in Google Sheets, and that the workflow iteratively fetches data until 100 prospects are listed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Google Sheet Template (Make a copy before use)                                                                                                              | https://docs.google.com/spreadsheets/d/1WWRwi3bseuZd1ebaAdGQYzDhwW_zBEOYib20l2Z_coY/edit?usp=sharing                            |
| Setup tutorial Loom video for workflow setup and operation                                                                                                  | https://www.loom.com/share/2a71542149054456b993826f56e5503f?sid=41a106b3-a16d-4686-8280-b7e56af8ce42                             |
| Directions: 1. Ensure Google Sheet has NO NAMES at start. 2. Start chat with agent. 3. Provide required info. 4. Let agent research (~3-4 mins per 10 names) | General usage instructions                                                                                                         |
| Connect AI agent output as content for Slack messages to respond properly                                                                                    | Slack communication best practice                                                                                                |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. The process strictly complies with content policies without including any illegal, offensive, or protected material. All data processed is legal and publicly accessible.