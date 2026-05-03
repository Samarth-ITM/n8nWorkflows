Blog Writer & Editor with Google Sheets Memory (GPT-4)

https://n8nworkflows.xyz/workflows/blog-writer---editor-with-google-sheets-memory--gpt-4--7671


# Blog Writer & Editor with Google Sheets Memory (GPT-4)

### 1. Workflow Overview

This workflow titled **"Blog Writer & Editor with Google Sheets Memory (GPT-4)"** automates the process of generating or editing blog content using AI, while maintaining session-based contextual memory stored in Google Sheets. It allows users to either write a new blog post based on a topic or reword an existing blog post, tracks the number of interactions per session, and enables conditional branching based on usage frequency.

The workflow's core logic is divided into the following functional blocks:

- **1.1 Input Reception and Prompt Preparation:** Receives chat input for blog writing or editing; builds appropriate system and user prompts for the AI agent.
- **1.2 AI Processing and Memory Handling:** Invokes the AI language model with memory buffering to generate or reword blog content.
- **1.3 Google Sheets Integration and Session Counting (Sub-workflow “google”):** Retrieves past session data from Google Sheets to count prior blog interactions.
- **1.4 Output Parsing and Storage:** Parses AI structured JSON output and appends results to Google Sheets.
- **1.5 Control Logic and Conditional Branching:** Checks if the session has exceeded a threshold number of blog generations/edits to decide workflow branching.
- **1.6 Setup & Documentation Notes:** Provides setup instructions, usage notes, and contact information.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Prompt Preparation

**Overview:**  
This block captures user input from chat, determines whether to write a new blog or reword an existing one, and constructs system and user prompts accordingly. It prepares the input for the AI agent while embedding a call to a sub-workflow for fetching session row counts from Google Sheets.

**Nodes Involved:**  
- Ask about blog topic  
- Choose to Write or Edit Blog

**Node Details:**

- **Ask about blog topic**  
  - *Type:* Langchain Chat Trigger  
  - *Role:* Entry point capturing user chat input that triggers the workflow.  
  - *Configuration:* Uses default webhook-based trigger, no additional parameters.  
  - *Inputs:* External chat input via webhook.  
  - *Outputs:* Passes chat input to subsequent nodes.  
  - *Edge cases:* Missing or malformed chat input can cause empty prompts downstream.

- **Choose to Write or Edit Blog**  
  - *Type:* Code Node  
  - *Role:* Dynamically builds AI system and user prompts based on whether the input includes existing blog content.  
  - *Configuration:*  
    - Checks if an `output` field (existing blog) is present; if yes, sets system prompt to "Reword the blog..." and user prompt includes existing blog text and session id.  
    - If no existing output, system prompt instructs to "Write a blog..." using chat input and session id.  
    - Both prompt types instruct calling the "Google tool" sub-workflow to count rows, passing only the session id.  
    - Outputs `system_prompt` and `user_prompt` fields for AI consumption.  
  - *Expressions:* Utilizes JSON fields from trigger or current item; manages null/empty checks.  
  - *Inputs:* Chat input and optional existing blog output.  
  - *Outputs:* Prompts for the AI agent node.  
  - *Edge cases:* Missing session id or chat input could lead to incomplete prompts; improper JSON escaping could cause prompt formatting issues.

---

#### 1.2 AI Processing and Memory Handling

**Overview:**  
This block invokes the AI agent with memory buffering to generate or reword the blog content, using the prompts built previously. It supports context window memory to maintain session continuity and uses GPT model capabilities.

**Nodes Involved:**  
- OpenAI Chat Model2  
- Simple Memory  
- Structured Output Parser  
- Blog Writer & Editor

**Node Details:**

- **OpenAI Chat Model2**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Executes GPT-5 model calls to generate or edit blog content based on prompts.  
  - *Configuration:* Uses "gpt-5" model with default options; linked to OpenAI API credentials.  
  - *Inputs:* Receives prompt from "Blog Writer & Editor" agent.  
  - *Outputs:* AI-generated content for parsing.  
  - *Edge cases:* API key issues, rate limits, network timeouts, or invalid prompts could cause failures.

- **Simple Memory**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Maintains a sliding window of the last 10 conversational turns for context preservation.  
  - *Configuration:* `contextWindowLength` set to 10.  
  - *Inputs:* Chat history and prompts from prior nodes.  
  - *Outputs:* Enriched context for AI model.  
  - *Edge cases:* Memory overflow or context misalignment if session data is inconsistent.

- **Structured Output Parser**  
  - *Type:* Langchain Structured Output Parser  
  - *Role:* Parses AI responses to enforce a defined JSON schema containing `items`, `session`, and `blog` fields.  
  - *Configuration:* Uses a JSON schema example enforcing keys and data types.  
  - *Inputs:* Raw AI output from OpenAI Chat Model2.  
  - *Outputs:* Validated structured JSON for downstream processing.  
  - *Edge cases:* Parsing failures if AI returns malformed or unexpected JSON.

- **Blog Writer & Editor**  
  - *Type:* Langchain Agent Node  
  - *Role:* Central AI agent node coordinating prompt input, memory, language model, and output parser integration.  
  - *Configuration:* Uses dynamic system and user prompts; connects with the memory buffer and output parser; uses GPT-5 model.  
  - *Inputs:* Prompts from "Choose to Write or Edit Blog", memory from Simple Memory, model from OpenAI Chat Model2, parser from Structured Output Parser.  
  - *Outputs:* Structured blog content JSON.  
  - *Edge cases:* Failures in any linked component propagate here; missing or invalid inputs can cause agent errors.

---

#### 1.3 Google Sheets Integration and Session Counting (Sub-workflow “google”)

**Overview:**  
This sub-workflow tool receives a session id, queries Google Sheets to find rows matching that session, filters them, and summarizes the row count, effectively counting how many times the blog has been written or edited in that session.

**Nodes Involved (Sub-workflow “google”):**  
- When Executed by Another Workflow (Execute Workflow Trigger)  
- Get row(s) in sheet (Google Sheets node)  
- Filter  
- Summarize1

**Node Details:**

- **When Executed by Another Workflow**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Entry point for sub-workflow invoked as a tool by the main workflow, accepting session id as input.  
  - *Configuration:* Passthrough input source.  
  - *Inputs:* Called from main workflow with session id.  
  - *Outputs:* Passes session id downstream.

- **Get row(s) in sheet**  
  - *Type:* Google Sheets Node  
  - *Role:* Reads rows from a specified Google Sheet and worksheet by document ID and sheet name.  
  - *Configuration:* Points to Google Sheet ID `1NwnABaQIReMmG2sRGrC-lv-5kpmsKJkUlRm-KmvPsCE`, sheet `gid=0`. Uses OAuth2 credentials.  
  - *Inputs:* Session id from trigger node.  
  - *Outputs:* All rows from sheet for filtering.  
  - *Edge cases:* Auth failures, sheet access issues, or document ID misconfiguration.

- **Filter**  
  - *Type:* Filter Node  
  - *Role:* Filters rows where the `session` field matches the session id passed from the trigger node.  
  - *Configuration:* Case-sensitive strict string equality on `session` field matching `query` value from the trigger.  
  - *Inputs:* Rows from Google Sheets node.  
  - *Outputs:* Filtered rows corresponding to the session id.  
  - *Edge cases:* Missing or mismatched session ids can cause empty results.

- **Summarize1**  
  - *Type:* Summarize Node  
  - *Role:* Counts the number of filtered rows by summarizing the `row_number` field.  
  - *Configuration:* Summarizes the `row_number` field to get count.  
  - *Inputs:* Filtered rows.  
  - *Outputs:* Numeric count of rows for the session.  
  - *Edge cases:* Empty input leads to zero count; field missing or malformed rows cause errors.

---

#### 1.4 Output Parsing and Storage

**Overview:**  
This block appends the structured AI-generated blog content and session metadata into the Google Sheet to maintain session history.

**Nodes Involved:**  
- n8n History (Google Sheets append)  
- Check if Ran 4+ times (If node)

**Node Details:**

- **n8n History**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends a new row to the Google Sheet with session id, row count, and blog content.  
  - *Configuration:* Uses same Google Sheets document and sheet as the sub-workflow; mapping fields `session`, `Rows`, and `output` from AI structured JSON output.  
  - *Inputs:* AI-generated output JSON (with `items`, `session`, `blog`) from "Blog Writer & Editor" node.  
  - *Outputs:* Passes appended data downstream.  
  - *Edge cases:* Google Sheets API errors, credential issues, or sheet permission problems.

- **Check if Ran 4+ times**  
  - *Type:* If Node  
  - *Role:* Evaluates if the `Rows` count exceeds 3 to allow branching logic (e.g., limit usage or notify).  
  - *Configuration:* Condition: `Rows` > 3 (numeric greater than).  
  - *Inputs:* Output from n8n History node.  
  - *Outputs:*  
    - True branch: session has run 4 or more times.  
    - False branch: fewer than 4 runs.  
  - *Edge cases:* Missing or non-numeric `Rows` field could cause condition evaluation errors.

---

#### 1.5 Control Logic and Conditional Branching

**Overview:**  
Depending on the number of past blog interactions in a session, the workflow can branch to either proceed with generating/editing or take alternative actions (not fully defined here).

**Nodes Involved:**  
- Check if Ran 4+ times (If node)  
- Choose to Write or Edit Blog (re-trigger on False branch)

**Node Details:**

- **Check if Ran 4+ times**  
  - Described above; controls flow based on usage frequency.

- **Choose to Write or Edit Blog** (re-triggered)  
  - On the False branch (less than 4 runs), the workflow re-invokes prompt preparation to continue generating or editing blogs.

---

#### 1.6 Setup & Documentation Notes

**Overview:**  
Several sticky notes provide detailed setup instructions, usage guidance, and contact information.

**Nodes Involved:**  
- Sticky Note6  
- Sticky Note55  
- Sticky Note2  
- Sticky Note7  
- Sticky Note10

**Node Details:**

- **Sticky Note6** (Setup Instructions)  
  - Describes steps to set up OpenAI API keys, Google Sheets format, and usage notes.  
  - Provides links to OpenAI platform, billing, and example Google Sheets.

- **Sticky Note55** (Workflow Overview)  
  - High-level description of the AI blog writer/editor functionality and session counting logic.  

- **Sticky Note2** (Sub-workflow reminder)  
  - Notes that the Google sub-workflow counts blog writes/edits per session and to update the Google Sheet accordingly.

- **Sticky Note7** (OpenAI Setup)  
  - Step-by-step instructions to set up OpenAI credentials.

- **Sticky Note10** (Google Sheets Setup)  
  - Guidance on preparing Google Sheets with required columns and connecting via OAuth2.

---

### 3. Summary Table

| Node Name                    | Node Type                               | Functional Role                                 | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                           |
|------------------------------|---------------------------------------|------------------------------------------------|-----------------------------|-----------------------------|-----------------------------------------------------------------------------------------------------|
| Ask about blog topic          | Langchain Chat Trigger                 | Entry point for user chat input                 | (Webhook external)           | Choose to Write or Edit Blog |                                                                                                     |
| Choose to Write or Edit Blog  | Code                                  | Builds system and user prompts for AI           | Ask about blog topic         | Blog Writer & Editor         | Linked to Sticky Note6, Sticky Note55 for setup and overview                                        |
| OpenAI Chat Model2            | Langchain OpenAI Chat Model            | Calls GPT-5 model for blog generation/editing   | Blog Writer & Editor         | Blog Writer & Editor         | Linked to Sticky Note7 for OpenAI setup                                                             |
| Simple Memory                | Langchain Memory Buffer Window         | Maintains session context window                 | Blog Writer & Editor         | Blog Writer & Editor         |                                                                                                     |
| Structured Output Parser      | Langchain Structured Output Parser     | Parses AI output into structured JSON            | OpenAI Chat Model2           | Blog Writer & Editor         |                                                                                                     |
| Blog Writer & Editor          | Langchain Agent Node                   | Orchestrates AI prompt, memory, model, parser   | Choose to Write or Edit Blog, Simple Memory, OpenAI Chat Model2, Structured Output Parser | n8n History                 |                                                                                                     |
| n8n History                  | Google Sheets                         | Appends AI output and session data to sheet     | Blog Writer & Editor         | Check if Ran 4+ times        | Linked to Sticky Note10 for Google Sheets setup                                                     |
| Check if Ran 4+ times         | If                                    | Branches workflow if session run count > 3      | n8n History                  | Choose to Write or Edit Blog |                                                                                                     |
| When Executed by Another Workflow | Execute Workflow Trigger              | Sub-workflow entry point for session row count  | google (called as tool)      | Get History (Google Sheets)  |                                                                                                     |
| Get History                  | Google Sheets                         | Reads all rows from Google Sheets                 | When Executed by Another Workflow | Filter                      |                                                                                                     |
| Filter                       | Filter                                | Filters rows matching the session id             | Get History                 | Summarize1                  |                                                                                                     |
| Summarize1                   | Summarize                             | Counts filtered rows to determine session run count | Filter                      | When Executed by Another Workflow (response) |                                                                                                     |
| google (Tool Workflow)        | Tool Workflow (sub-workflow)          | Counts blog interactions per session via sheet   | Called by Blog Writer & Editor | Blog Writer & Editor (via ai_tool) | Sticky Note2 explains this tool workflow for counting rows                                         |
| Sticky Note6                 | Sticky Note                          | Setup instructions and usage notes                |                             |                             | https://platform.openai.com/api-keys, Google Sheets sample link, detailed setup instructions        |
| Sticky Note55                | Sticky Note                          | Workflow overview and usage summary               |                             |                             | Explains AI blog writer/editor with Google Sheets memory concept                                   |
| Sticky Note2                 | Sticky Note                          | Reminder about Google sub-workflow tool           |                             |                             | Highlights importance of updating Google Sheet accordingly                                         |
| Sticky Note7                 | Sticky Note                          | OpenAI connection setup instructions              |                             |                             | Step-by-step OpenAI credential setup instructions                                                  |
| Sticky Note10                | Sticky Note                          | Google Sheets preparation instructions            |                             |                             | Sample sheet link and OAuth2 connection details                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node**  
   - Type: Langchain Chat Trigger  
   - Configuration: Default webhook trigger to receive user chat input.

2. **Create Code Node "Choose to Write or Edit Blog"**  
   - Type: Code  
   - Configuration: Implement JavaScript logic to inspect incoming item for an existing blog (`output` field).  
   - If exists, build:  
     - `system_prompt`: instruct to reword the blog and call Google tool with session id only.  
     - `user_prompt`: includes session id, blog content to reword, and JSON response format example.  
   - Else, build:  
     - `system_prompt`: instruct to write a blog and call Google tool with session id only.  
     - `user_prompt`: includes session id, chat input topic, and JSON response format example.  
   - Output fields: `system_prompt`, `user_prompt`.

3. **Create Langchain Agent Node "Blog Writer & Editor"**  
   - Type: Langchain Agent  
   - Connect inputs:  
     - Prompts from "Choose to Write or Edit Blog"  
     - AI Memory from a Langchain Memory Buffer Window node  
     - AI Language Model using OpenAI Chat Model node  
     - AI Output Parser node for structured JSON  
   - Configuration: Enable output parser; assign fields appropriately.

4. **Create Langchain Memory Buffer Window Node "Simple Memory"**  
   - Type: Memory Buffer Window  
   - Configuration: `contextWindowLength` set to 10.

5. **Create Langchain OpenAI Chat Model Node "OpenAI Chat Model2"**  
   - Type: OpenAI Chat Model  
   - Configuration: Model set to `gpt-5` (or GPT-4 if preferred).  
   - Credentials: Configure with OpenAI API key.

6. **Create Langchain Structured Output Parser Node "Structured Output Parser"**  
   - Type: Structured Output Parser  
   - Configuration: JSON schema example specifying:  
     ```json
     {
       "items": 5,
       "session": "sessionid",
       "blog": "blog"
     }
     ```

7. **Connect Nodes:**  
   - Chat Trigger → Choose to Write or Edit Blog  
   - Choose to Write or Edit Blog → Blog Writer & Editor  
   - Simple Memory → Blog Writer & Editor (ai_memory input)  
   - OpenAI Chat Model2 → Blog Writer & Editor (ai_languageModel input)  
   - Structured Output Parser → Blog Writer & Editor (ai_outputParser input)  

8. **Create Google Sheets Node "n8n History"**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Use your Google Sheet document ID  
   - Sheet Name: Use the target sheet name (e.g., "Sheet1" or `gid=0`)  
   - Mapping: Map fields from AI output JSON:  
     - `session` → session id  
     - `Rows` → number of rows/items from output  
     - `output` → blog content  
   - Credentials: Google Sheets OAuth2 credentials

9. **Connect Blog Writer & Editor → n8n History**

10. **Create If Node "Check if Ran 4+ times"**  
    - Type: If  
    - Condition: Numeric `Rows` field > 3  
    - Connect n8n History → Check if Ran 4+ times

11. **Connect If Node**  
    - False branch → Choose to Write or Edit Blog (to re-trigger for new blog or edit)  
    - True branch → (Optional) Add notification, limit, or alternate flow

12. **Create Sub-workflow "google" (Tool Workflow):**  
    - Entry: Execute Workflow Trigger node (passthrough input)  
    - Google Sheets Node (Get row(s) in sheet): Reads all rows with same document and sheet as main workflow  
    - Filter Node: Filters rows where `session` field equals the passed session id  
    - Summarize Node: Counts filtered rows by summarizing `row_number`  
    - Output: Pass count back as tool output  
    - Credentials: Use same Google Sheets OAuth2 credentials  
    - Save this sub-workflow and reference it as a tool called "google" in the main workflow

13. **Configure Credentials:**  
    - OpenAI API key in n8n credentials for GPT-5/4  
    - Google Sheets OAuth2 credentials with access to your sheet

14. **Test workflow:**  
    - Trigger with chat input including `sessionId`  
    - Check Google Sheets for appended rows  
    - Verify conditional branching works after 3 runs

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow uses GPT-5 model (can be replaced by GPT-4 if unavailable). Ensure your OpenAI account has sufficient billing funds and API key configured in n8n.                                                                                                                                                                                                                                  | https://platform.openai.com/api-keys                                                                   |
| Google Sheet format must have columns named exactly: `session`, `Rows`, `output`. The first row is header. Data rows start from row 2.                                                                                                                                                                                                                                                         | Sample Sheet: https://docs.google.com/spreadsheets/d/1NwnABaQIReMmG2sRGrC-lv-5kpmsKJkUlRm-KmvPsCE/edit?gid=0 |
| The sub-workflow "google" is a reusable tool to count session rows from Google Sheets and is invoked by the AI prompt instructions.                                                                                                                                                                                                                                                            | Sub-workflow details included in the main workflow JSON                                                |
| To adapt this workflow for Airtable, Notion, or other databases, replace Google Sheets nodes with respective data source nodes, maintaining the same session filtering and counting logic.                                                                                                                                                                                                       | Contact for customization: robert@ynteractive.com                                                     |
| Workflow can be extended to auto-publish generated blogs or notify users when session limits are reached.                                                                                                                                                                                                                                                                                        | Contact and LinkedIn profile: https://www.linkedin.com/in/robert-breen-29429625/                       |
| The sticky notes nodes include comprehensive setup instructions, usage details, and links to resources to facilitate deployment and customization.                                                                                                                                                                                                                                            | See Sticky Note6, Sticky Note55, Sticky Note7, Sticky Note10                                          |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.