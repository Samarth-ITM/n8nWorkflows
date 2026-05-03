Sentiment Analysis Tracking on Support Issues with Linear and Slack

https://n8nworkflows.xyz/workflows/sentiment-analysis-tracking-on-support-issues-with-linear-and-slack-2647


# Sentiment Analysis Tracking on Support Issues with Linear and Slack

### 1. Workflow Overview

This workflow continuously monitors active support issues managed in Linear.app and tracks the sentiment of ongoing conversations between reporters and assignees. Its primary use case is early detection of negative sentiment in support issue threads, enabling proactive team notifications to address potential customer dissatisfaction or escalation risks.

The workflow is logically divided into four main blocks:

- **1.1 Issue Retrieval and Preparation**  
  Scheduled polling of Linear’s GraphQL API to fetch recently updated issues, splitting them for individual processing.

- **1.2 Sentiment Analysis on Issue Comments**  
  Extracts and analyzes the sentiment of comments per issue using an AI-powered information extractor based on OpenAI.

- **1.3 Sentiment Tracking and Data Management in Airtable**  
  Stores sentiment data alongside issue details in Airtable, updating previous and current sentiment states for trend tracking.

- **1.4 Notification on Sentiment Changes via Slack**  
  Watches Airtable for sentiment transitions from non-negative to negative and sends deduplicated alerts to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Issue Retrieval and Preparation

**Overview:**  
This block fetches support issues updated in the last 30 minutes from Linear’s GraphQL API. It then splits the batch of issues into individual items for downstream processing.

**Nodes Involved:**  
- Schedule Trigger  
- Fetch Active Linear Issues (GraphQL)  
- Issues to List (SplitOut)

**Node Details:**

- **Schedule Trigger**  
  - Type: Trigger  
  - Role: Initiates workflow every 30 minutes for continuous monitoring.  
  - Config: Interval set to 30 minutes.  
  - Inputs: None (trigger node).  
  - Outputs: Triggers GraphQL query node.  
  - Failure modes: Workflow won’t trigger if n8n service is down; no retries configured here.

- **Fetch Active Linear Issues (GraphQL)**  
  - Type: GraphQL  
  - Role: Queries Linear’s GraphQL API for issues updated in the last 30 minutes.  
  - Config:  
    - Endpoint set to Linear GraphQL API.  
    - Query fetches issue id, identifier, title, description, url, timestamps, assignee name, and comments with user and body.  
    - Variables filter issues by updatedAt ≥ now minus 30 minutes.  
    - Authentication: Header-based (Linear API token).  
  - Inputs: Trigger output.  
  - Outputs: JSON with array of issues under `data.issues.nodes`.  
  - Failures: HTTP errors, auth token expiry, API rate limits.

- **Issues to List (SplitOut)**  
  - Type: SplitOut  
  - Role: Splits the array of issues into individual items for per-issue processing.  
  - Config: Splits on `data.issues.nodes`.  
  - Inputs: GraphQL node output.  
  - Outputs: One issue per item.  
  - Edge cases: Empty issue list results in no further processing.

---

#### 1.2 Sentiment Analysis on Issue Comments

**Overview:**  
Processes each issue’s comments thread using OpenAI-based Information Extractor to classify overall sentiment as positive, neutral, or negative, along with a sentiment summary description.

**Nodes Involved:**  
- OpenAI Chat Model (LangChain)  
- Sentiment over Issue Comments (Information Extractor)  
- Combine Sentiment Analysis (Set)

**Node Details:**

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides AI language model capability to downstream information extractor.  
  - Config: Uses OpenAI API credentials. No explicit prompt parameters here; used as language model provider.  
  - Inputs: None directly; linked as AI model for the Information Extractor.  
  - Outputs: Passes model results to Information Extractor.  
  - Failures: API key invalid, quota exceeded, timeouts.

- **Sentiment over Issue Comments (Information Extractor)**  
  - Type: LangChain Information Extractor  
  - Role: Extracts sentiment attributes from concatenated issue comments.  
  - Config:  
    - Text input: Joins all comments’ user display names, timestamps, and bodies separated by ‘---’.  
    - Attributes requested:  
      - `sentiment` (required): one of positive, negative, neutral.  
      - `sentimentSummary`: descriptive text about conversation mood.  
  - Inputs: Individual issue item from SplitOut node, plus OpenAI Chat Model as AI provider.  
  - Outputs: JSON with extracted sentiment and summary fields.  
  - Edge cases: Empty comment list may yield no sentiment; AI misclassification possible.

- **Combine Sentiment Analysis (Set)**  
  - Type: Set  
  - Role: Merges the original issue data with sentiment analysis results into a single JSON object.  
  - Config: Merges input JSON from ‘Issues to List’ with AI output using expression.  
  - Inputs: Sentiment node output.  
  - Outputs: Combined JSON for next block.  
  - Failures: Expression evaluation failures if expected fields missing.

---

#### 1.3 Sentiment Tracking and Data Management in Airtable

**Overview:**  
Iterates over each analyzed issue to upsert sentiment data into Airtable. Updates existing rows by shifting current sentiment to previous and saving new sentiment as current.

**Nodes Involved:**  
- For Each Issue... (SplitInBatches)  
- Copy of Issue (Set)  
- Get Existing Sentiment (Airtable - Search)  
- Update Row (Airtable - Upsert)

**Node Details:**

- **For Each Issue... (SplitInBatches)**  
  - Type: SplitInBatches  
  - Role: Processes each issue sequentially for Airtable updating.  
  - Config: Default batch size (1).  
  - Inputs: Combined sentiment JSON.  
  - Outputs: Single issue per batch.  
  - Edge cases: Large number of issues slows processing.

- **Copy of Issue (Set)**  
  - Type: Set  
  - Role: Creates a copy of the current item JSON for reference.  
  - Config: Outputs exact copy of input JSON.  
  - Inputs: For Each Issue (second output).  
  - Outputs: Used for later Airtable update values.  
  - Failures: Minimal risk.

- **Get Existing Sentiment (Airtable Search)**  
  - Type: Airtable (Search operation)  
  - Role: Searches Airtable for existing record matching current issue’s identifier.  
  - Config:  
    - Base and table set to the sentiment tracking Airtable.  
    - Fields fetched: Issue ID, Current Sentiment.  
    - Filter formula matches `Issue ID` to current issue’s identifier.  
    - Authentication: Airtable Personal Access Token.  
  - Inputs: Current issue JSON.  
  - Outputs: Existing Airtable row if found; empty if none.  
  - Failures: API errors, incorrect base/table IDs.

- **Update Row (Airtable Upsert)**  
  - Type: Airtable (Upsert operation)  
  - Role: Updates or inserts row with sentiment data.  
  - Config:  
    - Columns mapped:  
      - Summary: from sentimentSummary  
      - Assigned: assignee name  
      - Issue ID, Title, CreatedAt, UpdatedAt from issue data  
      - Current Sentiment: new sentiment converted to sentence case  
      - Previous Sentiment: set to old current sentiment from Airtable if exists, else 'N/A'  
    - Matching column: Issue ID  
  - Inputs: Existing sentiment (search) output and Copy of Issue data.  
  - Outputs: Updated Airtable row info.  
  - Edge cases: Concurrent updates may cause race conditions.

---

#### 1.4 Notification on Sentiment Changes via Slack

**Overview:**  
Detects changes in sentiment from non-negative to negative via Airtable trigger, filters for such transitions, deduplicates notifications, and sends Slack alerts to a designated channel.

**Nodes Involved:**  
- Airtable Trigger  
- Sentiment Transition (Switch)  
- Deduplicate Notifications (Remove Duplicates)  
- Report Issue Negative Transition (Slack)

**Node Details:**

- **Airtable Trigger**  
  - Type: Airtable Trigger  
  - Role: Watches Airtable table for updates on the ‘Current Sentiment’ field.  
  - Config:  
    - Base and table IDs configured.  
    - Poll interval: every hour.  
    - Trigger field: Current Sentiment.  
    - Authentication: Airtable Personal Access Token.  
  - Inputs: None (trigger).  
  - Outputs: Updated Airtable rows.  
  - Failures: Long poll errors, API limits.

- **Sentiment Transition (Switch)**  
  - Type: Switch  
  - Role: Filters rows where sentiment changed from non-negative to negative.  
  - Config:  
    - Rule: Previous Sentiment ≠ 'Negative' AND Current Sentiment = 'Negative'.  
  - Inputs: Airtable Trigger output.  
  - Outputs: Routes matching rows to next node; others discarded.  
  - Edge cases: Missing previous sentiment field, case sensitivity.

- **Deduplicate Notifications (Remove Duplicates)**  
  - Type: Remove Duplicates  
  - Role: Ensures unique notifications based on issue ID and last modified timestamp to prevent repeated alerts.  
  - Config:  
    - Deduplication value: concatenation of Issue ID and Last Modified date from Airtable fields.  
    - Operation: Remove items seen in previous executions.  
  - Inputs: Switch output.  
  - Outputs: Unique list of issues with negative sentiment transition.  
  - Edge cases: Clock skew causing wrong Last Modified timestamps.

- **Report Issue Negative Transition (Slack)**  
  - Type: Slack  
  - Role: Sends formatted Slack message alerting team about issues with newly negative sentiment.  
  - Config:  
    - Channel: specified Slack channel ID.  
    - Message type: block kit with sections and dividers.  
    - Message text includes total count and per-issue links to Linear with summaries.  
    - Webhook authentication for Slack API.  
  - Inputs: Deduplicated list of issues.  
  - Outputs: None.  
  - Failures: Slack API errors, invalid webhook, channel permission issues.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                   | Input Node(s)               | Output Node(s)                     | Sticky Note                                                                                                                                    |
|-----------------------------|----------------------------------|-------------------------------------------------|-----------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Trigger                          | Initiates workflow every 30 minutes              | -                           | Fetch Active Linear Issues        | ## 1. Continuously Monitor Active Linear Issues [Learn more about the GraphQL node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.graphql) To keep up with the latest changes in our active Linear tickets, we'll need to use Linear's GraphQL endpoint because filtering is currently unavailable in the official Linear.app node. For this demonstration, we'll check for updated tickets every 30mins. |
| Fetch Active Linear Issues  | GraphQL                         | Fetches recently updated Linear issues           | Schedule Trigger             | Issues to List                    | See above                                                                                                                                       |
| Issues to List              | SplitOut                        | Splits fetched issues into individual items      | Fetch Active Linear Issues   | Sentiment over Issue Comments     |                                                                                                                                                 |
| OpenAI Chat Model           | LangChain OpenAI Chat Model     | Provides AI model for sentiment extraction       | -                           | Sentiment over Issue Comments (AI provider) | ## 2. Sentiment Analysis on Current Issue Activity [Learn more about the Information Extractor node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.information-extractor) With our recently updated posts, we can use our AI to perform a quick sentiment analysis on the ongoing conversation to check the overall mood of the support issue. This is a great way to check how things are generally going in the support queue; positive should be normal but negative could indicate some uncomfortableness or even frustration. |
| Sentiment over Issue Comments | Information Extractor           | Extracts sentiment and summary from issue comments | Issues to List; OpenAI Chat Model | Combine Sentiment Analysis        | See above                                                                                                                                       |
| Combine Sentiment Analysis  | Set                            | Merges issue data with sentiment analysis results | Sentiment over Issue Comments | For Each Issue...                 |                                                                                                                                                 |
| For Each Issue...           | SplitInBatches                 | Processes each issue sequentially for Airtable   | Combine Sentiment Analysis   | Copy of Issue; Update Row          | ## 3. Capture and Track Results in Airtable [Learn more about the Airtable node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable) Next, we can capture this analysis in our insights database as means for human review. When the issue is new, we can create a new row but if the issue exists, we will update it's existing row instead. When updating an existing row, we move its previous "current sentiment" value into the "previous sentiment" column and replace with our new current sentiment. This gives us a "sentiment transition" which will be useful in the next step. Check out the Airtable here: https://airtable.com/appViDaeaFw4qv9La/shrq6HgeYzpW6uwXL |
| Copy of Issue               | Set                            | Copies current issue JSON for update reference   | For Each Issue...            | Get Existing Sentiment             | See above                                                                                                                                       |
| Get Existing Sentiment      | Airtable (Search)              | Searches for existing sentiment record in Airtable| Copy of Issue               | Update Row                       | See above                                                                                                                                       |
| Update Row                  | Airtable (Upsert)              | Updates or inserts sentiment data in Airtable    | Get Existing Sentiment       | For Each Issue...                 | See above                                                                                                                                       |
| Airtable Trigger            | Airtable Trigger               | Watches Airtable for updated sentiment rows      | -                           | Sentiment Transition              | ## 4. Get Notified when Sentiment becomes Negative [Learn more about the Slack node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/) A good use-case for tracking sentiment transitions could be to be alerted if ever an issue moves from a non-negative sentiment to a negative one. This could be a signal of issue handling troubles which may require attention before it escalates. In this demonstration, we use the Airtable trigger to catch rows which have their sentiment column updated and check for the non-negative-to-negative sentiment transition using the switch node. For those matching rows, we combine add send a notification via slack. A cool trick is to use the "remove duplication" node to prevent repeat notifications for the same updates - here we combine the Linear issue key and the row's last modified date. |
| Sentiment Transition        | Switch                         | Filters for sentiment changes from non-negative to negative | Airtable Trigger            | Deduplicate Notifications          | See above                                                                                                                                       |
| Deduplicate Notifications   | Remove Duplicates             | Prevents duplicate notifications for same issue update | Sentiment Transition        | Report Issue Negative Transition   | See above                                                                                                                                       |
| Report Issue Negative Transition | Slack                       | Sends Slack notification about negative sentiment transitions | Deduplicate Notifications   | -                                | See above                                                                                                                                       |
| Sticky Note (multiple)      | Sticky Note                   | Provides documentation and usage guidance         | -                           | -                                | Multiple sticky notes provide tutorial, documentation, and links as summarized above.                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to trigger every 30 minutes.

2. **Add a GraphQL node** (Fetch Active Linear Issues)  
   - Connect Schedule Trigger output to this node.  
   - Set endpoint to: `https://api.linear.app/graphql`  
   - Query:  
     ```
     query ($filter: IssueFilter) {
       issues(filter: $filter) {
         nodes {
           id
           identifier
           title
           description
           url
           createdAt
           updatedAt
           assignee { name }
           comments {
             nodes {
               id
               createdAt
               user { displayName }
               body
             }
           }
         }
       }
     }
     ```  
   - Variables:  
     ```json
     {
       "filter": {
         "updatedAt": { "gte": "={{ $now.minus(30, 'minutes').toISO() }}" }
       }
     }
     ```  
   - Authentication: Create and assign a Header Auth credential with Linear API token.

3. **Add a SplitOut node** (Issues to List)  
   - Connect GraphQL node output to this.  
   - Set to split on the field: `data.issues.nodes`.

4. **Add an OpenAI Chat Model node** (LangChain AI provider)  
   - Configure with OpenAI API credentials.

5. **Add an Information Extractor node** (Sentiment over Issue Comments)  
   - Connect SplitOut node output to this node.  
   - Set input text expression:  
     ```
     {{$json.comments.nodes.map(node => `${node.user.displayName} commented on ${node.createdAt}:\n${node.body}`).join('---\n')}}
     ```  
   - Attributes:  
     - `sentiment` (required): expects one of "positive", "negative", "neutral"  
     - `sentimentSummary`: textual description of sentiment  
   - Use OpenAI Chat Model node as the AI language model provider.

6. **Add a Set node** (Combine Sentiment Analysis)  
   - Connect Information Extractor output here.  
   - Set mode to raw, output JSON expression:  
     ```
     {
       ...$('Issues to List').item.json,
       ...$json.output
     }
     ```

7. **Add a SplitInBatches node** (For Each Issue...)  
   - Connect Set node output here.  
   - Default batch size (1).

8. **Add a Set node** (Copy of Issue)  
   - Connect second output of SplitInBatches to this node.  
   - Mode raw, output exactly the input JSON.

9. **Add an Airtable Search node** (Get Existing Sentiment)  
   - Connect Copy of Issue output here.  
   - Set Base and Table to your Airtable sentiment tracking base and table.  
   - Operation: Search  
   - Fields: Issue ID, Current Sentiment  
   - Filter formula:  
     ```
     ={Issue ID} = '{{ $json.identifier || "XYZ" }}'
     ```  
   - Use Airtable Personal Access Token credential.

10. **Add an Airtable Upsert node** (Update Row)  
    - Connect Airtable Search output here.  
    - Same Base and Table.  
    - Map columns:  
      - Summary: `={{ $('Copy of Issue').item.json.sentimentSummary || '' }}`  
      - Assigned: `={{ $('Copy of Issue').item.json.assignee.name }}`  
      - Issue ID: `={{ $('Copy of Issue').item.json.identifier }}`  
      - Issue Title: `={{ $('Copy of Issue').item.json.title }}`  
      - Issue Created: `={{ $('Copy of Issue').item.json.createdAt }}`  
      - Issue Updated: `={{ $('Copy of Issue').item.json.updatedAt }}`  
      - Current Sentiment: `={{ $('Copy of Issue').item.json.sentiment.toSentenceCase() }}`  
      - Previous Sentiment: `={{ !$json.isEmpty() ? $json['Current Sentiment'] : 'N/A' }}`  
    - Matching columns: Issue ID  
    - Use Airtable Personal Access Token credential.

11. **Add an Airtable Trigger node**  
    - Monitor the same Airtable base and table.  
    - Trigger on changes to “Current Sentiment” field.  
    - Poll interval: every hour.  
    - Use Airtable Personal Access Token credential.

12. **Add a Switch node** (Sentiment Transition)  
    - Connect Airtable Trigger output here.  
    - Add a rule:  
      - Output label: "NON-NEGATIVE to NEGATIVE"  
      - Condition:  
        ```
        {{$json.fields["Previous Sentiment"] !== 'Negative' && $json.fields["Current Sentiment"] === 'Negative'}}
        ```  
    - Fallback output: none.

13. **Add a Remove Duplicates node** (Deduplicate Notifications)  
    - Connect Switch node output here.  
    - Deduplication value:  
      ```
      {{$json.fields["Issue ID"]}}:{{$json.fields['Last Modified']}}
      ```  
    - Operation: Remove items seen in previous executions.

14. **Add a Slack node** (Report Issue Negative Transition)  
    - Connect Remove Duplicates output here.  
    - Set Slack channel (use correct channel ID).  
    - Message type: Block kit  
    - Message blocks:  
      ```json
      {
        "blocks": [
          {
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": ":rotating_light: The following Issues transitioned to Negative Sentiment"
            }
          },
          { "type": "divider" },
          ...($('Deduplicate Notifications').all().map(item => ({
            "type": "section",
            "text": {
              "type": "mrkdwn",
              "text": `*<https://linear.app/myOrg/issue/${$json.fields['Issue ID']}|${$json.fields['Issue ID']} ${$json.fields['Issue Title']}>*\n${$json.fields.Summary}`
            }
          })))
        ]
      }
      ```  
    - Authenticate with Slack API credentials.  
    - Set to execute once per run.

15. **Test the workflow** fully with real data and credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Check out the sample Airtable used in this workflow to understand data structure and for testing: https://airtable.com/appViDaeaFw4qv9La/shrq6HgeYzpW6uwXL                                                                                                                                                                                                                                | Sample Airtable                                                                                             |
| Learn more about using the GraphQL node in n8n here: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.graphql                                                                                                                                                                                                                                                              | n8n GraphQL node documentation                                                                              |
| Information Extractor node documentation and usage examples: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.information-extractor                                                                                                                                                                                                                    | Information Extractor node                                                                                   |
| Airtable node official docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable                                                                                                                                                                                                                                                                                      | Airtable node documentation                                                                                  |
| Slack node documentation for sending messages and blocks: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/                                                                                                                                                                                                                                                         | Slack node documentation                                                                                      |
| Join the n8n Discord or Community Forum for help: Discord (https://discord.com/invite/XPKeKXeB7d), Forum (https://community.n8n.io/)                                                                                                                                                                                                                                                          | Community support                                                                                            |
| This workflow demonstrates how sentiment monitoring can enhance customer support by early detection of negative trends and proactive team alerts. Consider customizing sentiment granularity or expanding to other issue types or teams to scale insights.                                                                                                                                 | Workflow usage suggestions                                                                                   |

---

This documentation provides a full understanding of the workflow’s purpose, node roles, data flow, and configuration details needed to build, maintain, or extend it efficiently.