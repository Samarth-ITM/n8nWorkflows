Automated Email Digest from Gmail to Slack with GPT-4o Summary

https://n8nworkflows.xyz/workflows/automated-email-digest-from-gmail-to-slack-with-gpt-4o-summary-9638


# Automated Email Digest from Gmail to Slack with GPT-4o Summary

### 1. Workflow Overview

This workflow automates the creation and distribution of a daily email digest summary from Gmail to a Slack channel using GPT-4o for intelligent summarization. It targets users who need a concise yet comprehensive overview of all emails received the previous day, highlighting urgent matters and actionable items.

The workflow is logically grouped into the following blocks:

- **1.1 Scheduled Execution:** Triggers the workflow daily at 8:00 AM to process the previous day’s emails.
- **1.2 Email Retrieval:** Connects to Gmail to fetch all emails received from midnight to midnight of the previous day.
- **1.3 Email Existence Check:** Determines if any emails were found; if none, sends a no-email notification to Slack.
- **1.4 AI Analysis and Summarization:** Uses a LangChain AI agent with GPT-4o-mini via OpenRouter to analyze emails and generate a structured daily digest summary.
- **1.5 Summary Formatting:** Formats the AI-generated summary message suitable for Slack display.
- **1.6 Slack Notification:** Sends the formatted summary or no-email notification to a configured Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Execution

- **Overview:**  
  Initiates the workflow daily at exactly 8 AM to process the previous day's emails.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Sticky Note (Overview and setup instructions)

- **Node Details:**  
  - **Schedule Trigger**  
    - *Type:* Schedule Trigger  
    - *Configuration:* Cron expression set to `0 0 8 * * *` (8:00 AM daily)  
    - *Inputs:* None (trigger node)  
    - *Outputs:* Connects to "Gmail - Get Yesterday's Emails"  
    - *Edge Cases:* If the workflow is paused or the server is down at trigger time, the digest may be skipped.  
  - **Sticky Note**  
    - *Purpose:* Provides a high-level overview, required credentials, and schedule adjustment instructions.  
    - *Content:* Explains daily run at 8 AM, setup for Gmail and Slack credentials.

---

#### 2.2 Email Retrieval

- **Overview:**  
  Fetches all emails from the Gmail account received between midnight and midnight of the previous day.

- **Nodes Involved:**  
  - Gmail - Get Yesterday's Emails  
  - Sticky Note1 (Gmail configuration info)  
  - If (conditional node)  

- **Node Details:**  
  - **Gmail - Get Yesterday's Emails**  
    - *Type:* Gmail Node (v2.1)  
    - *Configuration:*  
      - Operation: Get All emails  
      - Filters:  
        - Received after: start of yesterday (`{{ $now.minus({days: 1}).startOf('day').toISO() }}`)  
        - Received before: start of today (`{{ $now.startOf('day').toISO() }}`)  
    - *Inputs:* Trigger from Schedule Trigger  
    - *Outputs:* Connects to "If" node for email count check  
    - *Credentials:* Requires Gmail OAuth2 with read permissions  
    - *Edge Cases:*  
      - API rate limits or auth token expiration could cause failures.  
      - If no emails found, output is empty array.  
  - **Sticky Note1**  
    - Explains Gmail OAuth2 requirement and date filtering logic.  
  - **If**  
    - *Type:* Conditional Node (v2.2)  
    - *Configuration:* Checks if the number of emails retrieved is greater than 0 (`{{ $items().length > 0 }}`)  
    - *Inputs:* From Gmail node  
    - *Outputs:*  
      - True path: to "Item Lists" aggregation node  
      - False path: to "Slack - No Emails" node  
    - *Edge Cases:*  
      - Empty or malformed email data could affect condition.

---

#### 2.3 Email Existence Check and No-Email Notification

- **Overview:**  
  If no emails were retrieved, sends a Slack message indicating no emails to report for that day.

- **Nodes Involved:**  
  - Slack - No Emails  

- **Node Details:**  
  - **Slack - No Emails**  
    - *Type:* Slack Node (v2.3)  
    - *Configuration:*  
      - Text: Static message indicating no emails found, including the target date in JST timezone.  
      - Channel: Configured Slack channel (`C09L12N8F45`)  
      - Authentication: OAuth2 with Slack bot credentials  
    - *Inputs:* From "If" node false branch  
    - *Outputs:* None  
    - *Edge Cases:*  
      - Slack API rate limits or expired credentials could cause failure.  
      - Channel ID must be valid and accessible by the bot.

---

#### 2.4 AI Analysis and Summarization

- **Overview:**  
  Aggregates all retrieved emails and passes them to an AI agent which analyzes and creates a structured, concise daily digest summary.

- **Nodes Involved:**  
  - Item Lists (aggregation)  
  - AI Agent - Analyze Emails  
  - OpenRouter Chat Model  
  - Structured Output Parser  
  - Sticky Note2 (AI processing info)

- **Node Details:**  
  - **Item Lists**  
    - *Type:* Aggregate Node (v1)  
    - *Configuration:* Aggregates all emails into a single dataset for AI processing  
    - *Inputs:* From "If" node true branch  
    - *Outputs:* To "AI Agent - Analyze Emails"  
  - **AI Agent - Analyze Emails**  
    - *Type:* LangChain Agent Node (v2)  
    - *Configuration:*  
      - Text prompt dynamically constructed using input emails serialized as JSON.  
      - Instruction to analyze emails with details including total count, urgent emails, grouped key messages, action items, and a brief topic overview.  
      - System message sets role as intelligent email assistant focusing on concise, comprehensive summary.  
      - Output parser enabled for structured output.  
    - *Inputs:* From "Item Lists"  
    - *Outputs:* To "Format for Slack"  
    - *Edge Cases:*  
      - Large email datasets might exceed token limits.  
      - API errors or timeouts from OpenRouter.  
  - **OpenRouter Chat Model**  
    - *Type:* LangChain OpenRouter Chat Model (v1)  
    - *Configuration:*  
      - Model: `openai/gpt-4o-mini` (cost-efficient GPT-4 variant)  
      - Max tokens: 2000  
      - Temperature: 0.3 (low randomness)  
    - *Inputs:* From AI Agent as language model  
    - *Outputs:* To AI Agent as model response  
    - *Credentials:* OpenRouter API key required  
  - **Structured Output Parser**  
    - *Type:* LangChain Output Parser (v1.2)  
    - *Configuration:* Manual JSON schema defining expected output fields:  
      - `summary` (string), `emailCount` (number), `urgentItems` (array of strings), `actionItems` (array of strings)  
    - *Inputs:* From OpenRouter Chat Model  
    - *Outputs:* Back to AI Agent to finalize output  
  - **Sticky Note2**  
    - Explains AI processing details, model choice, and credential requirements.

---

#### 2.5 Summary Formatting

- **Overview:**  
  Formats the structured AI summary output into a human-readable Slack message with appropriate emojis and sections.

- **Nodes Involved:**  
  - Format for Slack

- **Node Details:**  
  - **Format for Slack**  
    - *Type:* Code Node (JavaScript, v2)  
    - *Configuration:*  
      - Extracts AI output fields: summary, emailCount, urgentItems, actionItems.  
      - Formats the message with:  
        - Date header (localized US English)  
        - Total emails count  
        - Urgent items list, if any, with warning emoji  
        - Action items list, if any, with clipboard emoji  
        - Main summary text with memo emoji  
      - Returns a JSON object containing the formatted message text and email count.  
    - *Inputs:* From AI Agent output  
    - *Outputs:* To "Slack - Send Summary"  
    - *Edge Cases:*  
      - Missing fields from AI output could cause formatting errors.  
      - Timezone assumptions for date display fixed to ‘en-US’.

---

#### 2.6 Slack Notification

- **Overview:**  
  Sends the formatted daily digest summary message to the configured Slack channel.

- **Nodes Involved:**  
  - Slack - Send Summary  
  - Sticky Note3 (Slack configuration info)

- **Node Details:**  
  - **Slack - Send Summary**  
    - *Type:* Slack Node (v2.2)  
    - *Configuration:*  
      - Text: Dynamic expression `{{$json.message}}` from formatting node  
      - Channel: Slack channel ID `C09L12N8F45` (default set to #general or another channel)  
      - Authentication: OAuth2 credentials for Slack bot  
    - *Inputs:* From "Format for Slack"  
    - *Outputs:* None  
    - *Edge Cases:*  
      - Slack API errors or permission issues  
      - Channel must be accessible by the bot user  
  - **Sticky Note3**  
    - Details Slack OAuth setup, channel selection, and permissions required.

---

### 3. Summary Table

| Node Name                      | Node Type                        | Functional Role                         | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                                                                                                |
|--------------------------------|---------------------------------|---------------------------------------|------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger                 | Starts workflow daily at 8 AM         | None                         | Gmail - Get Yesterday's Emails | ## Daily Email Digest Workflow: Runs daily at 8 AM to analyze yesterday's emails and send to Slack. Setup Gmail & Slack credentials, adjust schedule if needed.             |
| Sticky Note                   | Sticky Note                     | Overview and setup instructions       | None                         | None                          | ## Daily Email Digest Workflow: Runs daily at 8 AM to analyze yesterday's emails and send to Slack. Setup Gmail & Slack credentials, adjust schedule if needed.             |
| Gmail - Get Yesterday's Emails| Gmail Node                      | Retrieve emails from yesterday        | Schedule Trigger             | If                            | ## Gmail Configuration: Retrieves all emails from previous day midnight to midnight. Requires Gmail OAuth2 with read permissions.                                          |
| Sticky Note1                  | Sticky Note                     | Gmail configuration details           | None                         | None                          | ## Gmail Configuration: Retrieves all emails from previous day midnight to midnight. Requires Gmail OAuth2 with read permissions.                                          |
| If                           | Conditional Node                | Checks if any emails retrieved        | Gmail - Get Yesterday's Emails | Item Lists / Slack - No Emails |                                                                                                                                                                            |
| Slack - No Emails             | Slack Node                     | Sends no-email notification to Slack | If (false branch)            | None                          |                                                                                                                                                                            |
| Item Lists                   | Aggregate Node                 | Aggregates all emails into dataset    | If (true branch)             | AI Agent - Analyze Emails      |                                                                                                                                                                            |
| AI Agent - Analyze Emails     | LangChain AI Agent             | Analyzes emails and creates summary   | Item Lists                   | Format for Slack               | ## AI Processing: Analyzes emails and creates structured summary using GPT-4o-mini. Requires OpenRouter API key.                                                           |
| OpenRouter Chat Model         | LangChain Model                | GPT-4o-mini chat model for AI agent   | AI Agent - Analyze Emails    | Structured Output Parser       | ## AI Processing: Analyzes emails and creates structured summary using GPT-4o-mini. Requires OpenRouter API key.                                                           |
| Structured Output Parser      | LangChain Output Parser        | Parses AI response into structured data| OpenRouter Chat Model        | AI Agent - Analyze Emails      | ## AI Processing: Analyzes emails and creates structured summary using GPT-4o-mini. Requires OpenRouter API key.                                                           |
| Sticky Note2                  | Sticky Note                     | AI processing notes                   | None                         | None                          | ## AI Processing: Uses GPT-4o-mini model via OpenRouter with API key.                                                                                                         |
| Format for Slack              | Code Node                     | Formats AI summary for Slack message  | AI Agent - Analyze Emails    | Slack - Send Summary           |                                                                                                                                                                            |
| Slack - Send Summary          | Slack Node                    | Sends formatted summary to Slack      | Format for Slack             | None                          | ## Slack Configuration: Setup OAuth2, select channel, grant bot permissions. Adjust channel name as needed.                                                                |
| Sticky Note3                  | Sticky Note                     | Slack configuration notes             | None                         | None                          | ## Slack Configuration: Setup OAuth2, select channel, grant bot permissions. Adjust channel name as needed.                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it "Daily Email Digest and Summary Bot".**

2. **Add a Schedule Trigger node:**
   - Set type to "Schedule Trigger".
   - Configure cron expression to `0 0 8 * * *` to run daily at 8:00 AM.
   - Connect its main output to the Gmail node.

3. **Add a Gmail node:**
   - Set operation to "Get All".
   - Set filters:  
     - `receivedAfter` = `={{ $now.minus({days: 1}).startOf('day').toISO() }}`  
     - `receivedBefore` = `={{ $now.startOf('day').toISO() }}`  
   - Connect Schedule Trigger output to this node.
   - Authenticate with Gmail OAuth2 credentials with email reading scope.

4. **Add an If node:**
   - Condition: Check if number of items returned from Gmail node is greater than 0. Use expression: `{{ $items().length > 0 }}`.
   - Connect Gmail node output to this If node.

5. **Add a Slack node named "Slack - No Emails":**
   - Configure static text message informing "No emails were found for yesterday" with the date in JST timezone:  
     `*:email: Daily Email Digest — No emails*\nTarget date (JST): {{$now.setZone('Asia/Tokyo').minus({days:1}).toFormat('yyyy-LL-dd (ccc)')}}\nNo emails were found for yesterday. The next digest will run as scheduled.`
   - Set channel to your Slack channel ID (e.g., `C09L12N8F45`).
   - Authenticate with Slack OAuth2 credentials with message posting permission.
   - Connect If node's false output to this Slack node.

6. **Add an Aggregate node "Item Lists":**
   - Use default aggregate settings to combine all Gmail emails into one array.
   - Connect If node's true output to this node.

7. **Add an AI Agent node (LangChain Agent):**
   - Use "AI Agent - Analyze Emails" node from LangChain.
   - Configure prompt:  
     ```
     Analyze these emails from yesterday and create a concise daily digest summary:

     Emails to analyze:
     {{ JSON.stringify($json) }}

     Create a structured summary that includes:
     1. Total number of emails received
     2. Important/urgent emails (if any)
     3. Key messages grouped by sender or topic
     4. Action items or requests that need attention
     5. Brief overview of main topics discussed

     Format the output as a clean, readable summary suitable for Slack.
     ```
   - Set system message to define role as an intelligent email assistant focusing on concise summaries.
   - Enable output parser with manual JSON schema (see step 8).
   - Connect Aggregate node output to AI Agent input.

8. **Add a LangChain OpenRouter Chat Model node:**
   - Select model `openai/gpt-4o-mini`.
   - Set max tokens to 2000 and temperature to 0.3.
   - Authenticate with OpenRouter API key.
   - Connect AI Agent node's ai_languageModel input to this node.

9. **Add a LangChain Structured Output Parser node:**
   - Use manual schema:  
     ```json
     {
       "type": "object",
       "properties": {
         "summary": { "type": "string", "description": "The complete daily email digest summary formatted for Slack" },
         "emailCount": { "type": "number", "description": "Total number of emails analyzed" },
         "urgentItems": { "type": "array", "items": { "type": "string" }, "description": "List of urgent or important items" },
         "actionItems": { "type": "array", "items": { "type": "string" }, "description": "List of action items or tasks" }
       },
       "required": ["summary", "emailCount"]
     }
     ```
   - Connect OpenRouter Chat Model output to this parser.
   - Connect parser output back to AI Agent node's ai_outputParser input.

10. **Add a Code node "Format for Slack":**
    - Use JavaScript code to format the AI output as follows:
      ```javascript
      const output = $input.first().json.output;
      const today = new Date().toLocaleDateString('en-US', { 
        weekday: 'long', 
        year: 'numeric', 
        month: 'long', 
        day: 'numeric' 
      });

      let slackMessage = `:email: *Daily Email Digest for ${today}*\n\n`;
      slackMessage += `*Total Emails Analyzed:* ${output.emailCount}\n\n`;

      if (output.urgentItems && output.urgentItems.length > 0) {
        slackMessage += `:warning: *Urgent Items:*\n`;
        output.urgentItems.forEach(item => {
          slackMessage += `• ${item}\n`;
        });
        slackMessage += `\n`;
      }

      if (output.actionItems && output.actionItems.length > 0) {
        slackMessage += `:clipboard: *Action Items:*\n`;
        output.actionItems.forEach(item => {
          slackMessage += `• ${item}\n`;
        });
        slackMessage += `\n`;
      }

      slackMessage += `:memo: *Summary:*\n${output.summary}`;

      return {
        message: slackMessage,
        emailCount: output.emailCount
      };
      ```
    - Connect AI Agent node output to this Code node.

11. **Add a Slack node "Slack - Send Summary":**
    - Set text to expression: `{{$json.message}}`.
    - Set channel to the Slack channel ID (same as in step 5).
    - Authenticate with Slack OAuth2 credentials.
    - Connect Code node output to this Slack node.

12. **Add Sticky Notes at appropriate places for documentation:**
    - Workflow overview and setup instructions at the start.
    - Gmail configuration details near Gmail node.
    - AI processing notes near AI Agent and related nodes.
    - Slack configuration notes near Slack nodes.

13. **Activate the workflow and test:**
    - Ensure all credentials are properly configured and valid.  
    - Adjust Slack channel ID if necessary.  
    - Verify cron schedule fits your timezone and requirements.

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow uses OpenRouter integration with GPT-4o-mini for cost-efficient AI summarization.                                              | https://openrouter.ai/                                                                             |
| Slack OAuth2 credentials require bot permissions to post messages and access the selected channel.                                      | https://api.slack.com/authentication/oauth-v2                                                     |
| Gmail OAuth2 credentials must grant read access to the email inbox for the specified account.                                           | https://developers.google.com/gmail/api/auth/scopes                                               |
| Date formatting in Slack message uses US English locale; adjust as needed for other locales or timezones.                              | https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/toLocaleDateString |
| This workflow is designed for daily digest but can be adapted for other intervals by adjusting the Schedule Trigger cron expression.   |                                                                                                   |

---

This documentation enables both human users and automated agents to fully understand, reproduce, and maintain the "Daily Email Digest and Summary Bot" workflow with clear consideration of integration points, error scenarios, and configuration dependencies.

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.