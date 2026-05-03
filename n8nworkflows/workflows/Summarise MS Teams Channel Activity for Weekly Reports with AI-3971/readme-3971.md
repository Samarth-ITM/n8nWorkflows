Summarise MS Teams Channel Activity for Weekly Reports with AI

https://n8nworkflows.xyz/workflows/summarise-ms-teams-channel-activity-for-weekly-reports-with-ai-3971


# Summarise MS Teams Channel Activity for Weekly Reports with AI

### 1. Workflow Overview

This workflow automates the generation of a weekly summary report of Microsoft Teams channel activity, focusing on individual team member contributions and aggregating these into a comprehensive team-wide report. It targets remote teams using MS Teams as their main communication platform and leverages OpenAI’s language models for natural language summarization.

The workflow is logically divided into four main blocks:

- **1.1 Scheduled Trigger and Data Collection:** Automatically triggers every Monday morning to fetch all messages from the specified MS Teams channel for the previous week.

- **1.2 Data Grouping and Individual Member Report Generation:** Groups messages by user, enriches message context with reply information, then uses AI to generate personalized activity summaries per team member.

- **1.3 Team-wide Report Aggregation:** Aggregates all individual reports into a single concise team report, again utilizing AI to capture group-level insights.

- **1.4 Report Formatting and Posting:** Converts the final report from markdown to HTML and posts it back into the MS Teams channel as the first message of the week.

This structure ensures detailed attention to individual contributions while providing an accessible overview for the entire team, facilitating improved communication and motivation.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Scheduled Trigger and Data Collection

**Overview:**  
This block initiates the workflow every Monday at 6 AM, fetching all channel messages from the past week. It prepares raw data for subsequent analysis.

**Nodes Involved:**  
- Schedule Trigger  
- Fetch Latest Channel Messages  
- Group Messages By UserId  
- Groups to Items  

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the workflow on a defined schedule.  
  - *Configuration:* Runs weekly (implied Monday 6 AM by description, actual schedule interval is set in parameters).  
  - *Input/Output:* No input; outputs trigger signal to next node.  
  - *Failure Modes:* Scheduling misconfiguration could prevent runs; no direct failures expected.

- **Fetch Latest Channel Messages**  
  - *Type:* Microsoft Teams  
  - *Role:* Retrieves all channel messages from a specified MS Teams channel.  
  - *Configuration:*  
    - Team ID and Channel ID hardcoded to specific values (must be replaced for other teams/channels).  
    - Operation: “getAll” to fetch all messages.  
    - Credentials: Uses Microsoft Teams OAuth2.  
  - *Input/Output:* Trigger input from Schedule Trigger; outputs array of channel message objects.  
  - *Failure Modes:* Authentication errors; network timeouts; API rate limits; invalid team or channel IDs.

- **Group Messages By UserId**  
  - *Type:* Code (JavaScript)  
  - *Role:* Groups fetched messages by the user who posted them, adding parent message context for replies.  
  - *Configuration:*  
    - Custom JS code:  
      - Groups messages by `from.user.id`.  
      - For messages that are replies, finds and attaches the parent message.  
      - Outputs an array of objects each containing `user` info and their messages.  
  - *Input/Output:* Takes all messages from previous node; outputs grouped user-message objects.  
  - *Failure Modes:* JS execution errors if message structure unexpected; missing or malformed user data.

- **Groups to Items**  
  - *Type:* Split Out  
  - *Role:* Splits the array of grouped user data into individual items for further processing.  
  - *Configuration:* Splits on the `output` field from previous node.  
  - *Input/Output:* Input: grouped user-message objects; Output: individual user-message datasets.  
  - *Failure Modes:* None significant; if input empty, downstream nodes receive nothing.

---

#### 1.2 Data Grouping and Individual Member Report Generation

**Overview:**  
Processes each user’s messages using AI to generate a mini weekly report focusing on achievements, challenges, and interactions.

**Nodes Involved:**  
- Team Member Weekly Report Agent  
- OpenAI Chat Model  
- Merge Report With User Data  

**Node Details:**

- **Team Member Weekly Report Agent**  
  - *Type:* Chain LLM (LangChain)  
  - *Role:* Generates a personalized mini-report for each user based on their messages.  
  - *Configuration:*  
    - Input text formatted with user display name and messages, including replies.  
    - Chat prompt instructs AI to produce an energetic weekly summary focusing on wins, challenges, and team dynamics.  
    - Prompt includes context about the role of this summary (motivational, casual tone).  
  - *Input/Output:* Input: individual user-message data; Output: generated text report.  
  - *Failure Modes:* AI API errors, prompt formatting issues, incomplete user/message data.

- **OpenAI Chat Model**  
  - *Type:* Language Model node (gpt-4.1-mini)  
  - *Role:* Supports the Chain LLM by performing the actual language generation based on the provided prompt.  
  - *Configuration:* Uses OpenAI API credentials; model preset to “gpt-4.1-mini”.  
  - *Input/Output:* Receives prompt and messages; outputs text completion.  
  - *Failure Modes:* API rate limits, authentication problems, network errors.

- **Merge Report With User Data**  
  - *Type:* Set node  
  - *Role:* Combines original user data with the generated report text for downstream aggregation.  
  - *Configuration:* Merges grouped user data with AI-generated report in a single JSON object.  
  - *Input/Output:* Input: AI-generated report and user data; Output: merged JSON object.  
  - *Failure Modes:* Expression evaluation errors if input structure changes.

---

#### 1.3 Team-wide Report Aggregation

**Overview:**  
Aggregates all individual mini reports into a single team-wide weekly report using AI summarization.

**Nodes Involved:**  
- Reports to Single List  
- Team Weekly Report Agent  
- OpenAI Chat Model1  

**Node Details:**

- **Reports to Single List**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all individual user reports into a single list for team-level summarization.  
  - *Configuration:* Aggregates all incoming data items into one array.  
  - *Input/Output:* Inputs multiple merged user reports; outputs a single aggregated item.  
  - *Failure Modes:* Empty input; aggregation errors.

- **Team Weekly Report Agent**  
  - *Type:* Chain LLM (LangChain)  
  - *Role:* Generates a team-wide summary report from all individual reports.  
  - *Configuration:*  
    - Input text constructed by iterating over all user reports, formatting with user name, message count, and individual report.  
    - Prompt instructs AI to create an energetic team report highlighting wins, challenges, and connections between members.  
    - Output formatted as markdown without sign-off.  
  - *Input/Output:* Input: aggregated list of user reports; Output: markdown text report.  
  - *Failure Modes:* AI API failures; malformed input; prompt interpretation issues.

- **OpenAI Chat Model1**  
  - *Type:* Language Model node (gpt-4.1-mini)  
  - *Role:* Executes the language generation for the team-wide report.  
  - *Configuration:* Uses OpenAI API credentials; model preset.  
  - *Input/Output:* Receives prompt/messages; outputs markdown text.  
  - *Failure Modes:* Similar to other OpenAI nodes.

---

#### 1.4 Report Formatting and Posting

**Overview:**  
Converts the markdown report to HTML and posts it to the MS Teams channel as an HTML message.

**Nodes Involved:**  
- Markdown to HTML  
- Send Report to Channel  

**Node Details:**

- **Markdown to HTML**  
  - *Type:* Markdown  
  - *Role:* Converts the AI-generated markdown report to HTML for rich message formatting.  
  - *Configuration:* Mode set to “markdownToHtml”; input is the markdown text from previous node; output key is `html`.  
  - *Input/Output:* Input: markdown report string; Output: HTML formatted string.  
  - *Failure Modes:* Conversion errors if input malformed; generally reliable.

- **Send Report to Channel**  
  - *Type:* Microsoft Teams  
  - *Role:* Posts the final HTML report message to the specified MS Teams channel.  
  - *Configuration:*  
    - Team ID and Channel ID same as initial fetch node.  
    - Message content set to HTML output.  
    - Content type explicitly set to HTML.  
    - Credentials: Microsoft Teams OAuth2.  
  - *Input/Output:* Input: HTML report; Output: posted message confirmation.  
  - *Failure Modes:* Auth errors, permission issues, invalid IDs, message size limits.

---

### 3. Summary Table

| Node Name                   | Node Type                            | Functional Role                                      | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                                                                    |
|-----------------------------|------------------------------------|-----------------------------------------------------|---------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger                   | Initiates workflow every Monday                      | —                               | Fetch Latest Channel Messages    |                                                                                                                                                 |
| Fetch Latest Channel Messages| Microsoft Teams                   | Fetches all channel messages from last week         | Schedule Trigger                | Group Messages By UserId         | ## 1. Fetch All Channel Messages from Last Week [Learn more about the MS Teams node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.microsoftteams) |
| Group Messages By UserId    | Code                              | Groups messages by user and attaches reply context  | Fetch Latest Channel Messages   | Groups to Items                  |                                                                                                                                                 |
| Groups to Items             | Split Out                        | Splits grouped user data into individual items       | Group Messages By UserId        | Team Member Weekly Report Agent  |                                                                                                                                                 |
| Team Member Weekly Report Agent | Chain LLM (LangChain)          | Generates individual weekly reports per team member | Groups to Items                 | Merge Report With User Data      | ## 2. Generate Activity Reports for Each Team Member [Learn more about the Basic LLM node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm) |
| OpenAI Chat Model           | Language Model (OpenAI GPT-4)     | Supports individual report generation                 | Team Member Weekly Report Agent | Team Member Weekly Report Agent  |                                                                                                                                                 |
| Merge Report With User Data | Set                              | Merges user data with generated report text           | Team Member Weekly Report Agent | Reports to Single List           |                                                                                                                                                 |
| Reports to Single List      | Aggregate                        | Aggregates all individual reports into one list      | Merge Report With User Data     | Team Weekly Report Agent         |                                                                                                                                                 |
| Team Weekly Report Agent    | Chain LLM (LangChain)             | Generates team-wide summary report                     | Reports to Single List          | Markdown to HTML                | ## 3. Generate Final Report for Whole Team [Learn more about the Basic LLM node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm) |
| OpenAI Chat Model1          | Language Model (OpenAI GPT-4)     | Supports team report generation                        | Team Weekly Report Agent        | Team Weekly Report Agent         |                                                                                                                                                 |
| Markdown to HTML            | Markdown                         | Converts markdown report to HTML                       | Team Weekly Report Agent        | Send Report to Channel           | ## 4. Post Report on Team Channel (on Monday Morning!) [Learn more about the Markdown node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.markdown)  |
| Send Report to Channel      | Microsoft Teams                  | Posts final report message to MS Teams channel        | Markdown to HTML               | —                               |                                                                                                                                                 |
| Sticky Note                | Sticky Note                      | Documentation note for message fetching block         | —                               | —                               | See details in “1.1 Scheduled Trigger and Data Collection” block.                                                                              |
| Sticky Note1               | Sticky Note                      | Documentation note for individual report generation   | —                               | —                               | See details in “1.2 Data Grouping and Individual Member Report Generation” block.                                                             |
| Sticky Note3               | Sticky Note                      | Documentation note for team report aggregation        | —                               | —                               | See details in “1.3 Team-wide Report Aggregation” block.                                                                                       |
| Sticky Note4               | Sticky Note                      | Documentation note for report posting                  | —                               | —                               | See details in “1.4 Report Formatting and Posting” block.                                                                                      |
| Sticky Note6               | Sticky Note                      | General overview and instructions                      | —                               | —                               | General project description and usage instructions including links to community resources.                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to run weekly on Monday at 6:00 AM (adjust schedule interval accordingly).

2. **Add Microsoft Teams Node to Fetch Messages**  
   - Type: Microsoft Teams  
   - Operation: `getAll` on `channelMessage` resource  
   - Set Team ID to your team’s GUID  
   - Set Channel ID to your desired channel’s ID  
   - Connect credentials for Microsoft Teams OAuth2 API

3. **Create Code Node “Group Messages By UserId”**  
   - Paste provided JavaScript code that:  
     - Groups messages by `from.user.id`  
     - For each message that is a reply, attaches parent message data  
   - Connect output of Fetch Messages node to this code node.

4. **Add Split Out Node “Groups to Items”**  
   - Field to split out: `output` (from previous code output)  
   - Connect from Group Messages By UserId node.

5. **Add Chain LLM Node “Team Member Weekly Report Agent”**  
   - Use LangChain Chain LLM node  
   - Set text input with the template:  
     ```
     ## User
     DisplayName: {{ $json.user.displayName }}

     ## Messages
     {{ Array.from($json.messages).map(msg => {
       return [
         `Type: Message`,
         `Posted: ${msg.createdDateTime}`,
         `Message: ${msg.body.content.replaceAll('\n', ' ')}`,
         msg.parent ? `In Reply To: ${msg.parent.from.user.displayName} said "${msg.parent.body.content.replace('\n', ' ')}"` : ''
       ].join('\n')
     }).join('---\n') }}
     ```
   - Add system prompt message to instruct AI to summarize user’s weekly activity focusing on wins, challenges, motivation, and notable team dynamics.  
   - Connect from Groups to Items node.

6. **Add OpenAI Chat Model Node**  
   - Model: GPT-4.1-mini  
   - Connect to Chain LLM node as AI language model executor  
   - Provide OpenAI API credentials.

7. **Add Set Node “Merge Report With User Data”**  
   - Combine original user data with AI-generated report text in JSON output.  
   - Connect from Team Member Weekly Report Agent.

8. **Add Aggregate Node “Reports to Single List”**  
   - Aggregate all merged user reports into a single array.  
   - Connect from Merge Report With User Data.

9. **Add Chain LLM Node “Team Weekly Report Agent”**  
   - Input text: iterate over aggregated reports to format user display name, message count, and individual report.  
   - Prompt instructs AI to create an energetic, motivational team-wide weekly report highlighting wins, challenges, connections, welcomes, and farewells.  
   - Connect from Reports to Single List.

10. **Add OpenAI Chat Model Node (Team Level)**  
    - Same model and credentials as before.  
    - Connect as AI language model for Team Weekly Report Agent.

11. **Add Markdown Node “Markdown to HTML”**  
    - Convert markdown report output from Team Weekly Report Agent to HTML.  
    - Connect from Team Weekly Report Agent.

12. **Add Microsoft Teams Node “Send Report to Channel”**  
    - Operation: Post new message to channel  
    - Use same Team ID and Channel ID as fetching node  
    - Set message content to HTML from Markdown node  
    - Set content type to HTML  
    - Connect from Markdown to HTML node  
    - Use Microsoft Teams OAuth2 credentials.

13. **Connect all nodes as per above sequence** ensuring data flows properly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                 | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow is ideal for projects or teams where most communication happens in a single MS Teams channel. To cover multiple channels, duplicate and adjust per channel.                                                      | Usage instruction from workflow description                                                   |
| You can customize tone and style of AI reports by editing the prompt messages in the Chain LLM nodes.                                                                                                                       | Workflow customization advice                                                                 |
| For busy channels, consider sending the final report by email instead of posting to Teams.                                                                                                                                  | Workflow customization suggestion                                                             |
| Incorporate external project metrics or AI knowledgebase queries to enrich reports with performance data or ticket links.                                                                                                  | Workflow customization suggestion                                                             |
| Join the n8n community on [Discord](https://discord.com/invite/XPKeKXeB7d) or the [Forum](https://community.n8n.io/) for help and collaboration.                                                                             | Community support links                                                                        |
| Documentation links for key nodes:  
- MS Teams Node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.microsoftteams  
- Basic LLM Node: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm  
- Markdown Node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.markdown | Official n8n documentation resources                                                          |

---

This detailed reference document provides a complete understanding of the workflow’s structure, node roles, configurations, and how to rebuild or customize it effectively. It anticipates common failure points and integration considerations for smooth operation.