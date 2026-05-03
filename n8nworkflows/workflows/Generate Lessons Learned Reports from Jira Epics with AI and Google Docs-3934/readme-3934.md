Generate Lessons Learned Reports from Jira Epics with AI and Google Docs

https://n8nworkflows.xyz/workflows/generate-lessons-learned-reports-from-jira-epics-with-ai-and-google-docs-3934


# Generate Lessons Learned Reports from Jira Epics with AI and Google Docs

### 1. Workflow Overview

This workflow automates the generation of **Lessons Learned** or **Retrospective** reports from Jira Epics when they reach the "Done" status. It is designed for Agile teams seeking to capture valuable insights from completed Epics without manual report writing effort.

The workflow logically divides into these blocks:

- **1.1 Epic Completion Trigger:** Detects when a Jira Epic’s status changes to "Done" to trigger the automation.
- **1.2 Data Collection:** Retrieves all issues under the Epic and fetches all comments associated with those issues.
- **1.3 Data Preparation:** Extracts and formats relevant fields (Epic name, status, task title, description, comments) for AI processing.
- **1.4 AI Processing:** Uses an LLM (OpenAI GPT-4o-mini) via Langchain to generate a structured Lessons Learned report in Markdown, following a detailed system prompt.
- **1.5 Report Delivery:** Updates a specified Google Docs document with the generated report content.

---

### 2. Block-by-Block Analysis

#### 2.1 Epic Completion Trigger

- **Overview:**  
  This block listens for any update on Jira issues and filters events to trigger the workflow only when an Epic’s status changes to "Done". It ensures the workflow activates exclusively on relevant Epic completions.

- **Nodes Involved:**  
  - Jira Trigger  
  - If

- **Node Details:**

  - **Jira Trigger**  
    - *Type & Role:* Webhook node listening to Jira issue updates.  
    - *Configuration:* Subscribed to "jira:issue_updated" events with no additional filters set here (filter is empty).  
    - *Input/Output:* Output triggers on any issue update.  
    - *Potential Failures:* Webhook connectivity issues, Jira API rate limits, webhook registration errors.

  - **If**  
    - *Type & Role:* Conditional filter node.  
    - *Configuration:* Checks if the first change item in the webhook payload’s changelog has the "toString" value equal to "Done". This targets status changes to "Done".  
    - *Key Expression:* `{{$json.changelog.items[0].toString}} == "Done"`  
    - *Input:* Receives webhook event data from Jira Trigger.  
    - *Output:* Only passes execution forward if the condition is met.  
    - *Edge Cases:* If the changelog is empty or differently structured, this may fail; strict type validation is enabled.  

- **Sticky Note:**  
  > "Epic Done? This Node is Triggered on any issue change in Jira. However it only triggers the automation when the Epic status is changed to **Done**"

---

#### 2.2 Data Collection

- **Overview:**  
  Once an Epic is marked Done, this block collects all issues related to that Epic and fetches all comments associated with each issue for comprehensive context.

- **Nodes Involved:**  
  - Jira Get All Issues  
  - Jira Get All Comments

- **Node Details:**

  - **Jira Get All Issues**  
    - *Type & Role:* Jira API node to fetch all issues.  
    - *Configuration:* Operation "getAll" retrieves all issues, expecting to filter or handle relevant Epic’s issues downstream. Uses Jira Software Cloud API credentials.  
    - *Input:* Triggered from the If node confirming Epic Done.  
    - *Output:* Issues data including parent Epic fields.  
    - *Potential Failures:* API authentication errors, pagination or rate limits, incomplete data if permissions are insufficient.

  - **Jira Get All Comments**  
    - *Type & Role:* Jira API node to fetch all comments for a specific issue.  
    - *Configuration:* Operation "getAll" on "issueComment" resource, uses dynamic issueKey from previous node’s JSON (`={{ $json.key }}`).  
    - *Input:* Receives each issue from previous node to get comments.  
    - *Output:* List of comments per issue.  
    - *Edge Cases:* Issues with no comments, API throttling, missing permissions.  

- **Sticky Note:**  
  > "Fetch issue Description and Comments. Once the Epic is Done, these nodes fetch issues and comments that fall under the Epic. For further processing the output is bundled."

---

#### 2.3 Data Preparation

- **Overview:**  
  This block extracts and formats fields needed for the AI agent, including Epic name, status, task title, description, and individual comments. It then concatenates comments for summarization.

- **Nodes Involved:**  
  - Edit Fields  
  - Summarize

- **Node Details:**

  - **Edit Fields**  
    - *Type & Role:* Set node to assign and rename key fields for clarity.  
    - *Configuration:*  
      - Sets EpicName from the parent Epic summary.  
      - Sets EpicStatus from the parent Epic status category name.  
      - Sets Title from current issue summary.  
      - Sets Description from issue description.  
      - Extracts the actual comment text from complex JSON path `$json.body.content[0].content[0].text`.  
    - *Input:* Receives issue and comment data.  
    - *Output:* Structured JSON with relevant fields for summarization.  
    - *Edge Cases:* Complex comment structures may not always fit expected JSON path; missing or empty fields must be handled gracefully.

  - **Summarize**  
    - *Type & Role:* Summarization node concatenating multiple comments.  
    - *Configuration:*  
      - Fields to split by: EpicName, EpicStatus, Title, Description (to keep context).  
      - Field to summarize: Comment field, concatenated with newline separators.  
    - *Input:* Receives edited fields with comments.  
    - *Output:* A single concatenated and summarized comment string.  
    - *Potential Failures:* Large number of comments causing size limits; improper concatenation may lose context.

---

#### 2.4 AI Processing

- **Overview:**  
  Uses a Langchain AI Agent node configured with a detailed system message and the OpenAI GPT-4o-mini model to transform the collected and summarized data into a polished Lessons Learned report in Markdown format.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Simple Memory

- **Node Details:**

  - **AI Agent**  
    - *Type & Role:* Langchain agent node handling AI prompt engineering and interaction.  
    - *Configuration:*  
      - Text prompt includes variables for comments, description, title, status, and epic name.  
      - System Message is a detailed instruction set for generating a structured, Markdown-formatted Lessons Learned report. It specifies input format, output instructions, formatting guidelines, metadata handling, and date format.  
      - Uses session key "47" for memory management.  
    - *Input:* Receives summarized comments and task data.  
    - *Output:* Markdown formatted Lessons Learned report.  
    - *Edge Cases:* AI API errors, prompt formatting mistakes, session memory overflow.

  - **OpenAI Chat Model**  
    - *Type & Role:* Language model node providing GPT-4o-mini capabilities to the AI Agent.  
    - *Configuration:* Model selected as "gpt-4o-mini".  
    - *Input:* Connected as the language model for the AI Agent.  
    - *Potential Failures:* API quota limits, latency, network errors.

  - **Simple Memory**  
    - *Type & Role:* Memory buffer node storing AI session data.  
    - *Configuration:* Uses custom session key "47" to maintain context.  
    - *Input:* Connected to AI Agent for memory management.  
    - *Potential Failures:* Memory mismanagement, session key conflicts.

- **Sticky Note:**  
  > "Summarize and send to Google Docs. The LLM is summarizing the description / comments and generates a report with a layout defined in the System Message. Finally the output is sent to Google Docs."

---

#### 2.5 Report Delivery

- **Overview:**  
  Final block updates a specific Google Docs document with the AI-generated Lessons Learned report content.

- **Nodes Involved:**  
  - Google Docs

- **Node Details:**

  - **Google Docs**  
    - *Type & Role:* Google Docs node to update document content.  
    - *Configuration:*  
      - Operation set to "update".  
      - Document identified by Google Docs document ID `"14X5gcowEprmL6ORyoo9tIrWWEB1HlhkixXUelesCLXs"`.  
      - Inserts the AI Agent’s output (`{{$json.output}}`) into the document.  
    - *Input:* Receives Markdown report from AI Agent.  
    - *Output:* Updates the Google Docs file with the report.  
    - *Credential:* Uses Google Docs OAuth2 credentials.  
    - *Potential Failures:* OAuth token expiration, Google API quota exceeded, invalid document ID, formatting issues in inserted content.

---

### 3. Summary Table

| Node Name           | Node Type                                       | Functional Role                    | Input Node(s)      | Output Node(s)      | Sticky Note                                                                                                                             |
|---------------------|------------------------------------------------|----------------------------------|--------------------|---------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Jira Trigger        | n8n-nodes-base.jiraTrigger                      | Trigger workflow on Jira updates | -                  | If                  | Epic Done? This Node is Triggered on any issue change in Jira. However it only triggers the automation when the Epic status is changed to **Done** |
| If                  | n8n-nodes-base.if                               | Filter for Epic status "Done"    | Jira Trigger       | Jira Get All Issues  | Epic Done? This Node is Triggered on any issue change in Jira. However it only triggers the automation when the Epic status is changed to **Done** |
| Jira Get All Issues | n8n-nodes-base.jira                             | Fetch all issues under Epic      | If                 | Jira Get All Comments | Fetch issue Description and Comments. Once the Epic is Done, these nodes fetch issues and comments that fall under the Epic.             |
| Jira Get All Comments| n8n-nodes-base.jira                             | Fetch comments for each issue    | Jira Get All Issues | Edit Fields          | Fetch issue Description and Comments. Once the Epic is Done, these nodes fetch issues and comments that fall under the Epic.             |
| Edit Fields         | n8n-nodes-base.set                              | Extract relevant fields          | Jira Get All Comments | Summarize           | Fetch issue Description and Comments. Once the Epic is Done, these nodes fetch issues and comments that fall under the Epic.             |
| Summarize           | n8n-nodes-base.summarize                        | Concatenate & summarize comments | Edit Fields        | AI Agent             | Summarize and send to Google Docs. The LLM is summarizing the description / comments and generates a report with a layout defined in the System Message. Finally the output is sent to Google Docs. |
| AI Agent            | @n8n/n8n-nodes-langchain.agent                  | Generate Lessons Learned report  | Summarize, Simple Memory, OpenAI Chat Model | Google Docs          | Summarize and send to Google Docs. The LLM is summarizing the description / comments and generates a report with a layout defined in the System Message. Finally the output is sent to Google Docs. |
| OpenAI Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenAi           | Provide GPT-4o-mini LLM          | -                  | AI Agent             | Summarize and send to Google Docs. The LLM is summarizing the description / comments and generates a report with a layout defined in the System Message. Finally the output is sent to Google Docs. |
| Simple Memory       | @n8n/n8n-nodes-langchain.memoryBufferWindow     | Maintain AI session context      | -                  | AI Agent             | Summarize and send to Google Docs. The LLM is summarizing the description / comments and generates a report with a layout defined in the System Message. Finally the output is sent to Google Docs. |
| Google Docs         | n8n-nodes-base.googleDocs                        | Update Google Docs document      | AI Agent           | -                    | Summarize and send to Google Docs. The LLM is summarizing the description / comments and generates a report with a layout defined in the System Message. Finally the output is sent to Google Docs. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Credentials:**  
   - Configure Jira Software Cloud API credentials with your Jira API key.  
   - Configure Google Docs OAuth2 credentials with access to your target document.  
   - Configure OpenAI API credentials.

2. **Create Jira Trigger Node:**  
   - Type: `jiraTrigger`  
   - Parameters:  
     - Events: Select "jira:issue_updated".  
     - No filter at this node (empty).  
   - Credentials: Jira Software Cloud API credentials.

3. **Create If Node:**  
   - Type: `if`  
   - Configure condition:  
     - Left value: Expression `{{$json.changelog.items[0].toString}}`  
     - Operator: equals  
     - Right value: `"Done"`  
   - Connect Jira Trigger → If.

4. **Create Jira Get All Issues Node:**  
   - Type: `jira`  
   - Operation: `getAll`  
   - Credentials: Jira Software Cloud API credentials.  
   - Connect If (true output) → Jira Get All Issues.

5. **Create Jira Get All Comments Node:**  
   - Type: `jira`  
   - Resource: `issueComment`  
   - Operation: `getAll`  
   - Parameter `issueKey`: Expression `={{ $json.key }}` (dynamic per issue).  
   - Credentials: Jira Software Cloud API credentials.  
   - Connect Jira Get All Issues → Jira Get All Comments.

6. **Create Edit Fields (Set) Node:**  
   - Type: `set`  
   - Assign fields:  
     - `EpicName`: `={{ $('Jira Get All Issues').item.json.fields.parent.fields.summary }}`  
     - `EpicStatus`: `={{ $('Jira Get All Issues').item.json.fields.parent.fields.status.statusCategory.name }}`  
     - `Title`: `={{ $('Jira Get All Issues').item.json.fields.summary }}`  
     - `Description`: `={{ $('Jira Get All Issues').item.json.fields.description }}`  
     - `Comment`: `={{ $json.body.content[0].content[0].text }}`  
   - Connect Jira Get All Comments → Edit Fields.

7. **Create Summarize Node:**  
   - Type: `summarize`  
   - Fields to split by: `EpicName, EpicStatus, Title, Description`  
   - Fields to summarize: Comment (concatenate with newline separator)  
   - Connect Edit Fields → Summarize.

8. **Create OpenAI Chat Model Node:**  
   - Type: `lmChatOpenAi`  
   - Model: `gpt-4o-mini`  
   - Credentials: OpenAI API credentials.  
   - No connections as input; it will link to AI Agent.

9. **Create Simple Memory Node:**  
   - Type: `memoryBufferWindow` (Langchain memory node)  
   - Session Key: `"47"`  
   - Session ID Type: `customKey`  
   - No input connection; links to AI Agent.

10. **Create AI Agent Node:**  
    - Type: `langchain.agent`  
    - Text prompt:  
      ```
      =comments = {{ $json.concatenated_Comment }}
      description = {{ $json.Description }}
      title = {{ $json.Title }}
      status = {{ $json.EpicStatus }}
      epic_name = {{ $json.EpicName }}
      ```
    - System Message: Use the detailed system prompt text provided in the original workflow, which instructs the AI to create a Markdown Lessons Learned report with specified headings, formatting instructions, and metadata handling.  
    - Prompt Type: `define`  
    - Connect Summarize → AI Agent (main)  
    - Connect OpenAI Chat Model → AI Agent (ai_languageModel)  
    - Connect Simple Memory → AI Agent (ai_memory)

11. **Create Google Docs Node:**  
    - Type: `googleDocs`  
    - Operation: `update`  
    - Document URL: Insert your Google Docs document ID (e.g., `"14X5gcowEprmL6ORyoo9tIrWWEB1HlhkixXUelesCLXs"`)  
    - Action: Insert text with expression `{{$json.output}}` (AI Agent’s output)  
    - Credentials: Google Docs OAuth2 credentials.  
    - Connect AI Agent → Google Docs.

12. **Save and Activate Workflow:**  
    - Test by updating a Jira Epic status to "Done".  
    - Monitor executions for errors.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                               | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The system message used in the AI Agent node is key for tailoring the Lessons Learned report. Customize it to adjust report style or focus.                                                  | Within AI Agent node parameters.                                                                  |
| Jira API key and permissions must allow reading issues, comments, and webhook registration.                                                                                                  | Jira credentials setup.                                                                            |
| Google Docs OAuth2 credentials require edit access to the specified document.                                                                                                                | Google Docs credentials setup.                                                                     |
| The workflow assumes comments have a specific JSON structure; complex or rich text comments may require adjustments in the JSON path in the "Edit Fields" node.                             | Data Preparation block.                                                                            |
| Uses GPT-4o-mini model for cost-effective AI generation; can be upgraded to other OpenAI models if desired.                                                                                   | OpenAI Chat Model node configuration.                                                            |
| Agile teams often skip retrospective documentation due to time constraints; this workflow automates that process to improve continuous improvement culture.                                | Workflow description.                                                                             |
| See official Jira webhook documentation for webhook filters and event details: https://developer.atlassian.com/server/jira/platform/webhooks/                                            | Jira webhook info.                                                                                |
| n8n Docs on Jira nodes and Google Docs integration provide further configuration details: https://docs.n8n.io/integrations/built-in/nodes/n8n-nodes-base.jira/ and https://docs.n8n.io/integrations/built-in/nodes/n8n-nodes-base.googleDocs/ | n8n node docs.                                                                                   |

---

This structured documentation provides a clear understanding of the workflow, its node-by-node logic, key configurations, potential failure points, and instructions to recreate it fully within n8n. It supports both technical users and automation agents in analyzing, modifying, or reproducing the workflow.