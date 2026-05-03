🎯 Precision Prospecting: Automate LinkedIn Lead Gen with Bright Data

https://n8nworkflows.xyz/workflows/---precision-prospecting--automate-linkedin-lead-gen-with-bright-data-4873


# 🎯 Precision Prospecting: Automate LinkedIn Lead Gen with Bright Data

---

### 1. Workflow Overview

This workflow, titled **"🎯 Precision Prospecting: Automate LinkedIn Lead Gen with Bright Data"**, is designed to automate the generation of LinkedIn leads by integrating Google search, LinkedIn profile scraping, and AI-driven prospecting. It targets use cases such as sales prospecting, recruitment, and market research where users want to find detailed LinkedIn profiles based on company name, position, or direct LinkedIn URLs.

The workflow is logically structured into the following blocks:

- **1.1 Input Reception:** Handles incoming chat messages or external workflow triggers to start the prospecting process.
- **1.2 AI Prospector Agent:** Interprets user queries, generates Google search URLs, and orchestrates calls to other tools.
- **1.3 Google Search & Link Extraction:** Executes Google searches (via Bright Data), extracts URLs from search results, and filters LinkedIn profile links.
- **1.4 LinkedIn Profile Scraping:** Scrapes detailed LinkedIn profile data using Bright Data's web scraping capabilities.
- **1.5 AI Processing & Memory:** Uses OpenAI GPT-4o mini model with memory for context-aware responses and data summarization.
- **1.6 Workflow Utilities:** Includes nodes for limiting results and notes for user guidance.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures the initial trigger to start the prospecting process, either from chat messages or execution by another workflow.

- **Nodes Involved:**  
  - When chat message received  
  - When Executed by Another Workflow  

- **Node Details:**

  - **When chat message received**  
    - Type: LangChain chat trigger node  
    - Role: Entry point for chat-based user input to the workflow  
    - Configuration: Default options; listens for chat messages triggering the workflow  
    - Input/Output: No input; output to AI Prospector Agent node  
    - Edge Cases: Missing or malformed chat messages could cause no trigger; webhook must be publicly accessible  
    - Version: 1.1  

  - **When Executed by Another Workflow**  
    - Type: n8n native execute workflow trigger  
    - Role: Allows this workflow to be triggered programmatically from other workflows  
    - Configuration: Passes input data through unchanged  
    - Input/Output: No input (triggered externally); outputs to "Get 1 Google Result" node  
    - Edge Cases: Downstream nodes must handle input consistency  
    - Version: 1.1  

#### 2.2 AI Prospector Agent

- **Overview:**  
  This central AI agent interprets user queries, decides the prospecting method (by company & first name, company & position, or direct LinkedIn URI), generates Google search URLs, and orchestrates calls to LinkedIn-related tools.

- **Nodes Involved:**  
  - AI Prospector Agent  
  - Simple Memory1  
  - OpenAI Chat Model  
  - Search LinkedIn URI  
  - Get LinkedIn Profile Data  

- **Node Details:**

  - **AI Prospector Agent**  
    - Type: LangChain AI agent node  
    - Role: Core logic processor interpreting input and managing AI-driven prospecting flow  
    - Configuration:  
      - System message defines expert AI prospector behavior and step-by-step guidelines for handling different user query types  
      - Max 10 iterations limit ensures controlled AI processing  
      - Receives chat input from "When chat message received" or memory and outputs to tool nodes  
    - Key Expressions: Uses `{{$json.chatInput}}` as input text  
    - Connections: Receives chat input; calls "Search LinkedIn URI" or "Get LinkedIn Profile Data" as tools; uses OpenAI Chat Model and Simple Memory1 for context and language modeling  
    - Edge Cases: AI misinterpretation of queries, API rate limits, or failures in downstream tool nodes may impact results  
    - Version: 1.9  

  - **Simple Memory1**  
    - Type: LangChain memory buffer window  
    - Role: Maintains conversational context of last 20 interactions for AI agent  
    - Configuration: Context window length set to 20 messages  
    - Connections: Provides memory to AI Prospector Agent  
    - Edge Cases: Excessive context could exceed token limits; insufficient context may degrade AI understanding  
    - Version: 1.3  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI chat model node  
    - Role: Provides GPT-4o mini model for generating responses and reasoning for AI Prospector Agent  
    - Configuration: Model set to "gpt-4o-mini" (a smaller GPT-4 variant)  
    - Credentials: Requires OpenAI API credentials  
    - Connections: Provides language model to AI Prospector Agent  
    - Edge Cases: API rate limits, invalid credentials, or network issues  
    - Version: 1.2  

  - **Search LinkedIn URI**  
    - Type: LangChain tool workflow node  
    - Role: Calls a sub-workflow to perform Google search and extract the first LinkedIn profile URL  
    - Configuration: Calls workflow ID `fjEIEQ1L6n2IKqlx` (must be updated to user's actual workflow ID)  
    - Inputs: Receives query URL from AI Prospector Agent  
    - Outputs: Returns first LinkedIn URL to AI Prospector Agent  
    - Edge Cases: Workflow ID must be current; sub-workflow must be accessible; Google results may vary  
    - Version: 2.2  

  - **Get LinkedIn Profile Data**  
    - Type: Bright Data web scraper tool node  
    - Role: Scrapes detailed LinkedIn profile data synchronously by URL  
    - Configuration:  
      - Uses Bright Data dataset ID `gd_l1viktl72bvl7bjuj0` for scraping  
      - Input URLs expected as `[{"url":"https://www.linkedin.com/in/..."}]`  
    - Credentials: Requires Bright Data API key  
    - Inputs: URLs from AI Prospector Agent  
    - Outputs: Profile data back to AI Prospector Agent  
    - Edge Cases: Invalid or blocked URLs, Bright Data quota limits, network issues  
    - Version: 1  

#### 2.3 Google Search & Link Extraction

- **Overview:**  
  This block performs Google searches (via Bright Data), extracts all links from the HTML response, filters only LinkedIn profile URLs, and limits the number of results processed.

- **Nodes Involved:**  
  - Get 1 Google Result  
  - Get Links from Body  
  - Extract Links  
  - Filter only LinkedIn Profiles  
  - Limit  

- **Node Details:**

  - **Get 1 Google Result**  
    - Type: Bright Data scraping node  
    - Role: Executes Google search with given query URL, returning JSON results of 1 page  
    - Configuration:  
      - URL dynamically set as `={{ $json.query }}&num=1` to retrieve 1 search result  
      - Zone: `web_unlocker1` (Bright Data zone for scraping)  
      - Country: US  
      - Output format: JSON  
    - Credentials: Bright Data API credentials needed  
    - Inputs: Receives query from "When Executed by Another Workflow" or upstream calls  
    - Outputs: Passes results to HTML extraction node  
    - Edge Cases: Google blocking, Bright Data quota, invalid query format  
    - Version: 1  

  - **Get Links from Body**  
    - Type: HTML extraction node  
    - Role: Parses HTML content from Google search results body and extracts all anchor tag href attributes  
    - Configuration:  
      - Extracts `href` attribute from all `<a>` tags as array under `link` field  
      - Cleans and trims extracted text  
      - Operates on `body` property of input  
    - Inputs: Raw HTML from "Get 1 Google Result"  
    - Outputs: List of all links to "Extract Links" node  
    - Edge Cases: Missing or malformed HTML may yield no links  
    - Version: 1.2  

  - **Extract Links**  
    - Type: SplitOut node  
    - Role: Splits the array of links into individual items with `url` field for downstream processing  
    - Configuration: Extracts field named `link` into individual messages with field `url`  
    - Inputs: Links array from previous node  
    - Outputs: Single link per item to filter node  
    - Edge Cases: Empty or non-array input causes no output  
    - Version: 1  

  - **Filter only LinkedIn Profiles**  
    - Type: Filter node  
    - Role: Filters extracted URLs to keep only those starting with `https://` and containing `linkedin.com/`  
    - Configuration:  
      - Conditions: URL contains `linkedin.com/` AND starts with `https://`  
      - Case sensitive and strict type validation  
    - Inputs: Individual URLs from "Extract Links"  
    - Outputs: Valid LinkedIn URLs to "Limit" node  
    - Edge Cases: URLs missing https or malformed URLs are discarded  
    - Version: 2.2  

  - **Limit**  
    - Type: Limit node  
    - Role: Limits the number of LinkedIn URLs passed downstream  
    - Configuration: Default limit (no custom count specified) - effectively passes all or first N depending on setup  
    - Inputs: Filtered LinkedIn URLs  
    - Outputs: To AI Prospector Agent or further processing  
    - Edge Cases: No URLs results in empty output  
    - Version: 1  

#### 2.4 Workflow Utilities and Notes

- **Overview:**  
  Contains informational sticky notes for user guidance and instructions on credential setup.

- **Nodes Involved:**  
  - Sticky Note1  

- **Node Details:**

  - **Sticky Note1**  
    - Type: n8n sticky note  
    - Role: Provides instructions and TODOs for workflow setup  
    - Content Summary:  
      - Reminder to update "Search LinkedIn URI" node with current workflow ID after import  
      - Reminder to add Bright Data API keys to related nodes  
    - Position: Top-left of workspace for visibility  
    - Edge Cases: Misconfiguration if instructions not followed  
    - Version: 1  

---

### 3. Summary Table

| Node Name                 | Node Type                             | Functional Role                       | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                      |
|---------------------------|-------------------------------------|------------------------------------|--------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------|
| When chat message received | @n8n/n8n-nodes-langchain.chatTrigger | Entry point for chat message input | -                              | AI Prospector Agent             |                                                                                                 |
| When Executed by Another Workflow | n8n-nodes-base.executeWorkflowTrigger | Entry point for external workflow triggers | -                              | Get 1 Google Result             |                                                                                                 |
| AI Prospector Agent        | @n8n/n8n-nodes-langchain.agent      | Core AI logic and orchestration     | When chat message received, Simple Memory1, OpenAI Chat Model, Search LinkedIn URI, Get LinkedIn Profile Data | -                               |                                                                                                 |
| Simple Memory1             | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversational context    | -                              | AI Prospector Agent             |                                                                                                 |
| OpenAI Chat Model          | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o mini language model | -                              | AI Prospector Agent             |                                                                                                 |
| Search LinkedIn URI        | @n8n/n8n-nodes-langchain.toolWorkflow | Sub-workflow call to get first LinkedIn URL from Google search | AI Prospector Agent             | AI Prospector Agent             | TODO: Update this node with your current workflow ID after importing the workflow               |
| Get LinkedIn Profile Data  | n8n-nodes-brightdata.brightDataTool | Scrapes LinkedIn profile data       | AI Prospector Agent             | AI Prospector Agent             | Add your Bright Data API key to this node                                                       |
| Get 1 Google Result        | n8n-nodes-brightdata.brightData     | Performs Google search scraping     | When Executed by Another Workflow | Get Links from Body             | Add your Bright Data API key to this node                                                       |
| Get Links from Body        | n8n-nodes-base.html                  | Extracts all href links from Google search HTML body | Get 1 Google Result             | Extract Links                  |                                                                                                 |
| Extract Links              | n8n-nodes-base.splitOut              | Splits array of links into individual messages | Get Links from Body             | Filter only LinkedIn Profiles   |                                                                                                 |
| Filter only LinkedIn Profiles | n8n-nodes-base.filter               | Filters URLs to only LinkedIn profile links | Extract Links                  | Limit                         |                                                                                                 |
| Limit                     | n8n-nodes-base.limit                 | Limits number of LinkedIn URLs processed | Filter only LinkedIn Profiles   | AI Prospector Agent             |                                                                                                 |
| Sticky Note1              | n8n-nodes-base.stickyNote            | Workflow setup instructions         | -                              | -                               | # Precision Prospector with Bright Data - TODO: Update workflow ID in Search LinkedIn URI node; Add Bright Data API keys |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Add **"When chat message received"** (LangChain chat trigger) node: no special config needed, just default webhook trigger.
   - Add **"When Executed by Another Workflow"** (Execute Workflow Trigger) node: set `inputSource` to `passthrough`.

2. **Add AI Processing Nodes:**

   - Add **"Simple Memory1"** (LangChain memory buffer window) node: set context window length to 20.
   - Add **"OpenAI Chat Model"** (LangChain OpenAI chat model) node:  
     - Select model `gpt-4o-mini`.  
     - Configure OpenAI API credentials.
   - Add **"AI Prospector Agent"** (LangChain agent) node:  
     - Set input text expression to `{{$json.chatInput}}`.  
     - Set max iterations to 10.  
     - Paste the system message defining prospector AI behavior and guidelines (as per the original system message).  
     - Connect memory and language model inputs to "Simple Memory1" and "OpenAI Chat Model" respectively.  
     - Configure tool calls to "Search LinkedIn URI" and "Get LinkedIn Profile Data" nodes.

3. **Setup Google Search via Bright Data:**

   - Add **"Get 1 Google Result"** (Bright Data node):  
     - Set URL to `={{ $json.query }}&num=1`.  
     - Zone: `web_unlocker1`.  
     - Country: `us`.  
     - Configure Bright Data API credentials.
   - Add **"Get Links from Body"** (HTML node):  
     - Operation: Extract HTML content.  
     - Data property: `body`.  
     - Extract all `<a>` tag `href` attributes as array field named `link`.  
     - Enable trimming and text cleanup.
   - Add **"Extract Links"** (SplitOut node):  
     - Field to split: `link`.  
     - Destination field name: `url`.
   - Add **"Filter only LinkedIn Profiles"** (Filter node):  
     - Conditions:  
       - `url` contains `linkedin.com/`.  
       - `url` starts with `https://`.  
     - Use case-sensitive and strict validation.
   - Add **"Limit"** (Limit node): default configuration to limit outputs (custom limit can be applied as needed).

4. **Setup LinkedIn Profile Scraping:**

   - Add **"Get LinkedIn Profile Data"** (Bright Data tool node):  
     - Tool: Web Scraper.  
     - Dataset ID: use `gd_l1viktl72bvl7bjuj0` or your own dataset.  
     - URLs input format: array of objects with `"url"` keys, e.g., `[{"url":"https://www.linkedin.com/in/..."}]`.  
     - Configure Bright Data API credentials.

5. **Connect Nodes:**

   - Connect "When chat message received" main output to "AI Prospector Agent" main input.
   - Connect "When Executed by Another Workflow" main output to "Get 1 Google Result" main input.
   - Chain "Get 1 Google Result" → "Get Links from Body" → "Extract Links" → "Filter only LinkedIn Profiles" → "Limit".
   - Connect "Limit" output back to "AI Prospector Agent" as tool input.
   - Connect "Search LinkedIn URI" and "Get LinkedIn Profile Data" nodes as tools callable by "AI Prospector Agent".
   - Connect "Simple Memory1" and "OpenAI Chat Model" nodes as context and model inputs to "AI Prospector Agent".

6. **Final Setup:**

   - Update the **"Search LinkedIn URI"** node's workflow ID parameter to your current sub-workflow ID that performs Google search and returns LinkedIn URL.
   - Add your Bright Data API credentials to nodes "Get LinkedIn Profile Data" and "Get 1 Google Result".
   - Ensure OpenAI API credentials are configured for "OpenAI Chat Model".
   - Optionally add a sticky note with setup instructions for future reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| After importing this workflow, update the node "Search LinkedIn URI" tool to use your current workflow ID for sub-workflow calls.                                                                                                            | Sticky Note in workflow; critical for sub-workflow linkage                                                  |
| Add your Bright Data API key to both "Get LinkedIn Profile Data" and "Get 1 Google Result" nodes to enable scraping capabilities.                                                                                                            | Sticky Note in workflow                                                                                      |
| Bright Data web scraping relies on valid dataset IDs and quota; monitor your usage and consider proxy zones like `web_unlocker1` for scraping Google search results.                                                                        | Bright Data official docs                                                                                    |
| The AI Prospector Agent uses GPT-4o mini, a compact GPT-4 variant; ensure your OpenAI account supports this model.                                                                                                                           | OpenAI API documentation                                                                                     |
| The workflow is designed to handle three main user query types: company+first name, company+position, or direct LinkedIn URL; follow the AI system message guidelines for extending or customizing behavior.                                  | AI prompt inside "AI Prospector Agent"                                                                      |
| For troubleshooting, monitor webhook accessibility for chat message triggers and API limits for Bright Data and OpenAI nodes.                                                                                                              | n8n workflow monitoring best practices                                                                      |

---

**Disclaimer:** The provided text is extracted solely from an automated workflow created with n8n, a no-code automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---