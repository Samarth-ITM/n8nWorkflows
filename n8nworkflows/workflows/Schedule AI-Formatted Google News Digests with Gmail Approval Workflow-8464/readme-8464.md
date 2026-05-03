Schedule AI-Formatted Google News Digests with Gmail Approval Workflow

https://n8nworkflows.xyz/workflows/schedule-ai-formatted-google-news-digests-with-gmail-approval-workflow-8464


# Schedule AI-Formatted Google News Digests with Gmail Approval Workflow

### 1. Workflow Overview

This workflow automates the process of collecting the latest Google News articles on a specified topic ("AI"), formatting these news items into a clean, modern HTML email digest using AI, and sending the digest via Gmail for manual approval. If the batch of news articles is declined, the workflow automatically fetches the next batch, formats it again, and sends another approval email. Upon approval, the workflow cleans up its state by deleting the batch counter.

**Target Use Cases:**  
- Regularly curated news digest generation for topics of interest.  
- Human-in-the-loop content approval before reposting or sharing news summaries.  
- Automating batch processing and iterative review of large news datasets.

**Logical Blocks:**  
- **1.1 Scheduled Trigger & News Fetching:** Initiates workflow periodically, queries Google News via SerpApi, and manages batch counters stored in Airtable.  
- **1.2 Batch Extraction & AI Formatting:** Extracts a batch of 10 news articles from results, then uses an AI agent to generate a responsive HTML email digest.  
- **1.3 Approval Email & Decision Handling:** Sends the formatted email via Gmail for approval, waits for user input, and branches based on approval or decline.  
- **1.4 Counter Management & Cleanup:** Updates or deletes the batch counter in Airtable to manage state and iterations over news batches.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & News Fetching

- **Overview:**  
  This block triggers the workflow on a schedule (default daily), creates a counter record in Airtable to track batch progress, performs a Google News search on the topic "AI" using SerpApi, and retrieves the current batch counter for processing.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Create Counter (Airtable)  
  - Google_news search (SerpApi)  
  - Get Counter (Airtable)  
  - Sticky Notes: Sticky Note9 (overview), Sticky Note3, Sticky Note1, Sticky Note4, Sticky Note7, Sticky Note9

- **Node Details:**

  - **Schedule Trigger**  
    - Type: `n8n-nodes-base.scheduleTrigger`  
    - Role: Initiates the workflow periodically; default interval unspecified but typically daily.  
    - Config: Basic schedule with interval set (empty object implies default daily).  
    - Inputs: None (trigger node)  
    - Outputs: Connects to Create Counter.  
    - Failures: Scheduler misconfiguration or node downtime could prevent runs.

  - **Create Counter (Airtable)**  
    - Type: `n8n-nodes-base.airtable`  
    - Role: Creates a new record in an Airtable base/table to maintain a numeric batch counter initialized to 0, linked to this workflow ID.  
    - Config: Airtable base and table selected; fields set to Counter=0 and Workflow ID=current workflow.  
    - Inputs: From Schedule Trigger  
    - Outputs: To Google_news search  
    - Failures: Airtable API authentication, rate limits, or schema mismatch.

  - **Google_news search (SerpApi)**  
    - Type: `n8n-nodes-serpapi.serpApi`  
    - Role: Queries Google News for the fixed search term "AI". Returns up to ~100 news results.  
    - Config: Query "AI", operation "google_news", no additional request options.  
    - Inputs: From Create Counter  
    - Outputs: To Get Counter  
    - Credentials: SerpApi API key required.  
    - Failures: API limits, invalid key, network errors.

  - **Get Counter (Airtable)**  
    - Type: `n8n-nodes-base.airtable`  
    - Role: Retrieves the current batch counter record for this workflow from Airtable.  
    - Config: Searches by Workflow ID equal to current workflow ID, returns only the Counter field.  
    - Inputs: From Google_news search  
    - Outputs: To Extract Details  
    - Failures: Airtable API issues or missing record.

- **Sticky Notes:**  
  - Sticky Note9: Explains overall workflow purpose and setup instructions.  
  - Sticky Note3: Notes that the workflow is scheduled to run once a day.  
  - Sticky Note1: Describes SerpApi node use to fetch ~100 news results.  
  - Sticky Note4: Mentions creating the Airtable counter record.  
  - Sticky Note7: Describes the “Get Counter” node role.

---

#### 2.2 Batch Extraction & AI Formatting

- **Overview:**  
  Extracts the next batch of 10 news articles starting from the current counter, compiles key details into simplified JSON, then passes this to an AI agent that generates a polished HTML email using a custom template.

- **Nodes Involved:**  
  - Extract Details (Code)  
  - Update Counter (Airtable)  
  - gpt-4o-mini (LangChain OpenAI Model)  
  - Prepare Content Review Email (LangChain Agent)  
  - Sticky Notes: Sticky Note13, Sticky Note17, Sticky Note18, Sticky Note14, Sticky Note6, Sticky Note8

- **Node Details:**

  - **Extract Details (Code)**  
    - Type: `n8n-nodes-base.code`  
    - Role: Extracts a batch of 10 news articles from the SerpApi results using the current Airtable counter as an index. Compiles simplified JSON objects with position, title, source, authors, and two link variants. Sorts the batch by position.  
    - Key Expression: Uses JavaScript and `$input.first().json.Counter` to get batch start index.  
    - Inputs: From Get Counter (provides current counter and news results)  
    - Outputs: JSON array `out` with simplified news + updated counter (incremented by batch size)  
    - Failures: Empty results, unexpected JSON structure, index out of bounds.

  - **Update Counter (Airtable)**  
    - Type: `n8n-nodes-base.airtable`  
    - Role: Updates the Airtable counter record with incremented counter value after batch extraction.  
    - Config: Matches record by Workflow ID, sets Counter to new value from code node.  
    - Inputs: From Extract Details  
    - Outputs: To Prepare Content Review Email  
    - Failures: Airtable write failures or conflicts.

  - **gpt-4o-mini (LangChain OpenAI Model)**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: AI language model node to format or assist in content generation. In this workflow, it is chained but not directly connected to the critical path (no output connections).  
    - Config: Uses model "gpt-4o-mini", output as plain text.  
    - Inputs: None explicitly connected for main flow.  
    - Outputs: To Prepare Content Review Email (AI agent).  
    - Credentials: OpenAI API key required.  
    - Failures: API errors, rate limits, or network issues.

  - **Prepare Content Review Email (LangChain Agent)**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Receives the extracted batch JSON and generates a clean, modern HTML email content using a detailed prompt specifying table-based layout, styling, and email compatibility.  
    - Key Expression: Passes extracted JSON as input data via expression referencing Extract Details output.  
    - Inputs: From Update Counter (with updated counter), and gpt-4o-mini (AI language model).  
    - Outputs: To Gmail Approval News node for sending.  
    - Failures: AI response failures, prompt parsing issues.

- **Sticky Notes:**  
  - Sticky Note13: Explains code node’s role to extract headers and details.  
  - Sticky Note17: Shows example extracted headers output.  
  - Sticky Note18: Shows example HTML email generated by AI agent.  
  - Sticky Note14: Describes the AI agent’s function to produce the HTML email.  
  - Sticky Note6: Highlights this entire block is managing the batch iteration counter.  
  - Sticky Note8: Notes the counter increment step.

---

#### 2.3 Approval Email & Decision Handling

- **Overview:**  
  Sends the AI-generated HTML digest email to the configured Gmail address with approval options enabled. Waits for user interaction to either approve or decline the batch. Based on the decision, it either deletes the counter record (end workflow) or fetches the next batch for processing.

- **Nodes Involved:**  
  - Gmail Approval News (Gmail node with send-and-wait)  
  - Approved? (If node to branch logic)  
  - Delete Counter (Airtable)  
  - Get Counter (Airtable) [for decline path]  
  - Sticky Notes: Sticky Note2, Sticky Note5, Sticky Note11, Sticky Note12, Sticky Note15, Sticky Note10, Sticky Note19

- **Node Details:**

  - **Gmail Approval News**  
    - Type: `n8n-nodes-base.gmail`  
    - Role: Sends an email containing the formatted news digest and waits for explicit user approval or decline. Uses Gmail OAuth2 credentials.  
    - Config:  
      - To address from environment variable EMAIL_ADDRESS_ME.  
      - Subject: "🔥FOR APPROVAL🔥Latest 10 Google News"  
      - Message body: Injects HTML from Prepare Content Review Email node output.  
      - Approval options: Double approval required (e.g., explicit approve/decline buttons).  
      - Timeout: 45 minutes wait before timeout.  
    - Inputs: From Prepare Content Review Email  
    - Outputs: To Approved? node for decision branching  
    - Failures: Gmail API auth errors, timeout, user non-response.

  - **Approved? (If node)**  
    - Type: `n8n-nodes-base.if`  
    - Role: Checks if the approval input equals "true" (case insensitive).  
    - Config: Compares `{{$json.data.approved}}` with "true".  
    - Inputs: From Gmail Approval News  
    - Outputs:  
      - True branch: To Delete Counter  
      - False branch: To Get Counter (to fetch next batch)  
    - Failures: Missing or malformed approval input.

  - **Delete Counter (Airtable)**  
    - Type: `n8n-nodes-base.airtable`  
    - Role: Deletes the Airtable batch counter record to reset state after approval completion.  
    - Config: Uses the record ID from Get Counter node.  
    - Inputs: From Approved? (true branch)  
    - Outputs: None (workflow ends)  
    - Failures: Airtable API errors or record missing.

  - **Get Counter (Airtable) [Decline branch]**  
    - Same as previously described, used here to restart batch extraction with next counter.  
    - Inputs: From Approved? (false branch)  
    - Outputs: To Extract Details (loop back)  
    - Failures: Airtable access issues.

- **Sticky Notes:**  
  - Sticky Note2: Marks the "Send Approval Email" step.  
  - Sticky Note5: Marks "Prepare Approval Email".  
  - Sticky Note11: Annotates the approval condition branch.  
  - Sticky Note12: Describes behavior on Decline to send next batch email.  
  - Sticky Note15: Notes "Send email and wait for Approve/Decline input".  
  - Sticky Note10: Notes deletion of Airtable record after approval.  
  - Sticky Note19: Shows example approval email screenshot.

---

### 3. Summary Table

| Node Name                  | Node Type                            | Functional Role                          | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                             |
|----------------------------|------------------------------------|----------------------------------------|------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | n8n-nodes-base.scheduleTrigger     | Starts workflow on schedule             | None                         | Create Counter              | The workflow is triggered based on its schedule (ie: once a day).                                    |
| Create Counter             | n8n-nodes-base.airtable            | Creates Airtable record as batch counter| Schedule Trigger             | Google_news search          | Create a AirTable record to use as a counter.                                                        |
| Google_news search          | n8n-nodes-serpapi.serpApi          | Fetches Google News results for "AI"   | Create Counter               | Get Counter                 | Use the SerpApi to obtain the latest Google News about a topic of interest. About 100 news_results are returned. |
| Get Counter                | n8n-nodes-base.airtable            | Retrieves current batch counter         | Google_news search, Approved? (decline path) | Extract Details, Delete Counter (approved path) | Get the current counter.                                                                               |
| Extract Details             | n8n-nodes-base.code                | Extracts next 10 news items from results| Get Counter                  | Update Counter              | Some code to extract only the "headers" and a few details of each news.                              |
| Update Counter             | n8n-nodes-base.airtable            | Updates batch counter in Airtable       | Extract Details              | Prepare Content Review Email | Increment the counter.                                                                                 |
| gpt-4o-mini                | @n8n/n8n-nodes-langchain.lmChatOpenAi | AI language model (optional assist)    | None (standalone)             | Prepare Content Review Email |                                                                                                       |
| Prepare Content Review Email| @n8n/n8n-nodes-langchain.agent    | Generates responsive HTML email content | Update Counter, gpt-4o-mini   | Gmail Approval News         | Use an AI Agent to generate the HTML email with the 10 news, based on our template.                   |
| Gmail Approval News         | n8n-nodes-base.gmail               | Sends approval email and waits response | Prepare Content Review Email  | Approved?                   | Send email and wait for Approve/Decline input.                                                       |
| Approved?                  | n8n-nodes-base.if                  | Branches workflow based on approval     | Gmail Approval News          | Delete Counter (true), Get Counter (false) | If Approved: / If Declined: prepare next 10 news and send another Approval email.                     |
| Delete Counter             | n8n-nodes-base.airtable            | Deletes batch counter after approval    | Approved? (true branch)      | None                       | Delete the AirTable record.                                                                            |
| Sticky Notes (various)      | n8n-nodes-base.stickyNote          | Workflow documentation and explanations | Various                     | None                       | See respective sticky notes for detailed explanations and screenshots.                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a "Schedule Trigger" node:**  
   - Set the interval to desired frequency (e.g., daily).  
   - This node triggers the workflow.

3. **Add an Airtable node "Create Counter":**  
   - Operation: Create  
   - Configure Airtable credentials (Personal Access Token with access to your base).  
   - Select the base and table for counters.  
   - Map fields:  
     - Counter: 0  
     - Workflow ID: `{{$workflow.id}}` (expression)  
   - Connect "Schedule Trigger" output to this node.

4. **Add "Google_news search" node (SerpApi):**  
   - Use SerpApi credentials (API key).  
   - Operation: google_news  
   - Query: "AI" (or desired topic)  
   - Connect "Create Counter" output to this node.

5. **Add "Get Counter" Airtable node:**  
   - Operation: Search  
   - Base/Table same as above.  
   - Filter formula: `={Workflow ID} = '{{$workflow.id}}'`  
   - Return field: Counter  
   - Connect "Google_news search" output to this node.

6. **Add "Extract Details" Code node:**  
   - Paste JavaScript code to:  
     - Read news_results from SerpApi output  
     - Use current counter as start index  
     - Extract batch of 10 news items with position, title, source, authors, and links  
     - Increment counter by 10  
     - Return array `out` and updated counter.  
   - Connect "Get Counter" output to this node.

7. **Add Airtable "Update Counter" node:**  
   - Operation: Update  
   - Match by Workflow ID  
   - Set Counter to `{{$json.counter}}` from previous node output  
   - Connect "Extract Details" output to this node.

8. **Add LangChain OpenAI "gpt-4o-mini" node:**  
   - Model: gpt-4o-mini  
   - Response format: text  
   - Use OpenAI credentials.  
   - (Optional node, connect if desired for enhanced AI formatting assistance.)

9. **Add LangChain Agent node "Prepare Content Review Email":**  
   - Prompt: Provide detailed instructions to generate HTML email from JSON batch (see overview for example template).  
   - Input data: Pass JSON output from "Extract Details" node using expression.  
   - Connect "Update Counter" output and "gpt-4o-mini" output to this node.

10. **Add Gmail node "Gmail Approval News":**  
    - Operation: Send and wait  
    - Send to your email address (set via environment variable `EMAIL_ADDRESS_ME`)  
    - Subject: "🔥FOR APPROVAL🔥Latest 10 Google News"  
    - Message body: Use HTML output from "Prepare Content Review Email" node.  
    - Approval options: Enable double approval.  
    - Set wait time limit to 45 minutes.  
    - Connect "Prepare Content Review Email" output to this node.  
    - Configure Gmail OAuth2 credentials.

11. **Add "Approved?" If node:**  
    - Condition: Check if `{{$json.data.approved}}` equals "true" (case insensitive).  
    - Connect "Gmail Approval News" output to this node.

12. **Add Airtable "Delete Counter" node:**  
    - Operation: Delete record  
    - Use record ID from "Get Counter" node.  
    - Connect "Approved?" true branch to this node.

13. **Connect "Approved?" false branch back to "Get Counter" node:**  
    - This loop fetches the next batch counter and restarts extraction.

14. **Add Sticky Notes throughout the workflow to document steps and add screenshots or explanations as needed.**

15. **Set workflow timezone to "Europe/Riga" or your preferred timezone.**

16. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                             | Context or Link                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Keep up to date with the latest news about your favorite topics. Save time in selecting the next news to repost on social media or blog about, with this scheduled workflow. The workflow uses a global counter in Airtable to manage batches. You can swap AI models or approval nodes as needed.          | Sticky Note9 content in the workflow.                                                              |
| Example screenshots for Google News results output, extracted headers, generated HTML email, and approval email visuals are embedded as sticky notes for reference. These aid understanding of data shapes and UI output.                                                                                 | Sticky Notes 16, 17, 18, 19                                                                         |
| The workflow requires credentials for SerpApi, Airtable (Personal Access Token), OpenAI, and Gmail OAuth2. Ensure these are properly configured before activation.                                                                                                                                      | Credential references in relevant nodes.                                                           |
| The AI agent prompt is carefully crafted to produce mobile-responsive, email-safe HTML with inline CSS and table layouts for compatibility across email clients.                                                                                                                                       | See Prepare Content Review Email node prompt details.                                              |
| Gmail approval uses send-and-wait with double approval option for explicit user confirmation. Timeout is configured to 45 minutes to avoid indefinite waits.                                                                                                                                             | Gmail Approval News node parameters.                                                               |
| The batch size is fixed at 10 news articles per email batch. The counter increments by 10 after each batch to paginate through results.                                                                                                                                                                  | Code node logic in Extract Details.                                                                |
| Airtable is used as a simple persistent store for batch counters, identified by workflow ID to allow multiple workflows or parallel runs without conflict.                                                                                                                                              | Airtable nodes and formula filters.                                                                |

---

This documentation enables full comprehension and reimplementation of the "Schedule AI-Formatted Google News Digests with Gmail Approval Workflow" in n8n. It provides detailed node roles, configurations, data flows, error considerations, and setup instructions for developers and automation agents alike.

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.