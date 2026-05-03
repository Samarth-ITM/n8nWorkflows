Generate AI Summaries from Hacker News to Slack with OpenRouter

https://n8nworkflows.xyz/workflows/generate-ai-summaries-from-hacker-news-to-slack-with-openrouter-9774


# Generate AI Summaries from Hacker News to Slack with OpenRouter

### 1. Workflow Overview

This workflow automates the process of fetching recent news articles from Hacker News based on specified keywords, generating concise AI-powered summaries for each article, and posting the summaries to a designated Slack channel. It is designed for users who want to stay updated on particular topics with minimal manual effort by leveraging AI summarization and Slack notifications.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger**: Initiates the workflow automatically on a daily schedule.
- **1.2 Configuration Setup**: Sets user-defined keywords and Slack channel parameters.
- **1.3 News Retrieval**: Fetches recent news articles from Hacker News filtered by the configured keywords.
- **1.4 AI Summarization**: Uses an AI language model (via OpenRouter) to generate a concise summary of each article.
- **1.5 Slack Message Formatting and Delivery**: Prepares the formatted message and sends it to the Slack channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

**Overview:**  
This block automatically triggers the workflow daily at 9 AM, ensuring news summaries are generated and shared consistently without manual intervention.

**Nodes Involved:**  
- Trigger Daily at 9 AM

**Node Details:**  
- **Trigger Daily at 9 AM**  
  - Type: Schedule Trigger  
  - Configuration: Set to trigger on a daily interval at 9:00 AM by default; the schedule can be customized by editing the node.  
  - Input: None (start node)  
  - Output: Connects to "Configure Your Settings" node to start the workflow process.  
  - Version: 1.2  
  - Edge Cases: Misconfiguration of schedule can lead to workflow not triggering; no authentication required.  
  - Sticky Note Reference: Provides instructions on adjusting the schedule.

---

#### 1.2 Configuration Setup

**Overview:**  
Sets up the parameters necessary for the workflow, such as the keywords to filter news articles and the Slack channel where summaries will be posted.

**Nodes Involved:**  
- Configure Your Settings

**Node Details:**  
- **Configure Your Settings**  
  - Type: Set  
  - Configuration: Defines two key variables —  
    - `keywords`: JSON array, default set to `["AI"]` but can be customized to track multiple topics.  
    - `slack_channel`: String, default set to `"news-updates"`, representing the Slack channel name.  
  - Input: From schedule trigger  
  - Output: Connects to "Get many items" to fetch news articles.  
  - Version: 3.4  
  - Edge Cases: Invalid or empty keywords array may cause empty news fetch results; slack_channel must exist in Slack workspace or posting will fail.  
  - Sticky Note Reference: Guidance on modifying keywords and Slack channel.

---

#### 1.3 News Retrieval

**Overview:**  
Queries Hacker News to obtain recent news articles that match the specified keywords.

**Nodes Involved:**  
- Get many items

**Node Details:**  
- **Get many items**  
  - Type: HackerNews (n8n built-in node)  
  - Configuration:  
    - `limit`: 3 (fetches up to 3 articles per run)  
    - `resource`: "all" (fetch from all Hacker News categories)  
    - `keyword`: Dynamically set to the first element of the `keywords` array from previous node.  
  - Input: From "Configure Your Settings"  
  - Output: Connects to "Summarize Article with AI" for summarization.  
  - Version: 1  
  - Edge Cases: If no articles match the keyword, downstream nodes receive no data; rate limiting or API errors possible; keyword filtering limited to first keyword only.  
  - Sticky Note Reference: None.

---

#### 1.4 AI Summarization

**Overview:**  
Uses an AI language model integrated via OpenRouter to summarize each news article URL into a concise, professional, bullet-point summary in Japanese.

**Nodes Involved:**  
- Summarize Article with AI  
- OpenRouter Chat Model

**Node Details:**  
- **Summarize Article with AI**  
  - Type: LangChain Agent (AI Agent Node)  
  - Configuration:  
    - Input Text: The URL of the article from "Get many items".  
    - System Message: Instructs the AI to produce a 3-bullet summary in Japanese, focusing on key info with a neutral tone and no introductory phrases.  
    - passthroughBinaryImages: Enabled to allow image data if present.  
    - Prompt Type: Defined prompt style.  
  - Input: From "Get many items" (main data) and "OpenRouter Chat Model" (language model integration).  
  - Output: Connects to "Format Message for Slack".  
  - Version: 2.2  
  - Edge Cases: AI API authentication errors, timeout, malformed input URLs, or connectivity issues could cause failures; incomplete or ambiguous article content may yield poor summaries.  
  - Sub-Workflow: Uses the "OpenRouter Chat Model" as an AI language model backend.  

- **OpenRouter Chat Model**  
  - Type: LangChain LM Chat OpenRouter (AI Language Model Node)  
  - Configuration: Default options, connected as the language model for the summarization agent.  
  - Input: None directly from nodes (configured as AI model reference).  
  - Output: Used internally by "Summarize Article with AI".  
  - Version: 1  
  - Edge Cases: Requires valid OpenRouter API credentials; API rate limits or downtime affect summarization.  
  - Sticky Note Reference: Instructions on setting up OpenRouter credentials and swapping AI providers.

---

#### 1.5 Slack Message Formatting and Delivery

**Overview:**  
Transforms the AI-generated summary into a human-readable Slack message format and posts it to the specified Slack channel.

**Nodes Involved:**  
- Format Message for Slack  
- Send Summary to Slack

**Node Details:**  
- **Format Message for Slack**  
  - Type: Set  
  - Configuration:  
    - Creates a new string field `slackMessage` combining the article title in bold and the AI summary under a heading "AI Summary", formatted with line breaks for Slack readability.  
    - Keeps other fields intact for potential future use.  
  - Input: From "Summarize Article with AI" (summary output)  
  - Output: Connects to "Send Summary to Slack"  
  - Version: 3.4  
  - Edge Cases: Missing title or summary fields could produce incomplete messages.  

- **Send Summary to Slack**  
  - Type: Slack  
  - Configuration:  
    - Text: Uses the `slackMessage` field from the previous node.  
    - Channel: Uses channel ID for `"C09L12N8F45"` (hardcoded channel, overrides configured slack_channel)  
    - Authentication: OAuth2 credential required for Slack workspace access.  
  - Input: From "Format Message for Slack"  
  - Output: None (end node)  
  - Version: 2  
  - Edge Cases: Slack API errors such as invalid OAuth tokens, channel permissions, or rate limits; mismatch between configured channel name and hardcoded channel ID may cause confusion.  
  - Sticky Note Reference: Instructions for setting up Slack OAuth2 credentials.

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                            | Input Node(s)             | Output Node(s)             | Sticky Note                                                                                              |
|---------------------------|---------------------------------|--------------------------------------------|---------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| Trigger Daily at 9 AM      | Schedule Trigger                 | Starts workflow daily at 9 AM               | None                      | Configure Your Settings     | ## Set Your Schedule: Default daily 9 AM trigger; editable schedule node.                              |
| Configure Your Settings    | Set                             | Defines keywords array and Slack channel    | Trigger Daily at 9 AM      | Get many items             | ## Configure Keywords & Slack Channel: How to customize keywords and Slack channel.                    |
| Get many items            | HackerNews                      | Fetches recent news articles by keyword    | Configure Your Settings    | Summarize Article with AI  |                                                                                                        |
| Summarize Article with AI  | LangChain Agent (AI Agent)      | Generates AI summary of each article URL   | Get many items, OpenRouter Chat Model | Format Message for Slack | ## Connect Your AI Model: Use OpenRouter API key; can swap AI provider by replacing this node.         |
| OpenRouter Chat Model      | LangChain LM Chat OpenRouter    | Provides AI language model service          | None                      | Summarize Article with AI  | ## Connect Your AI Model: Use OpenRouter API key; can swap AI provider by replacing this node.         |
| Format Message for Slack   | Set                             | Formats summary into Slack message string  | Summarize Article with AI  | Send Summary to Slack      |                                                                                                        |
| Send Summary to Slack      | Slack                           | Posts formatted summary to Slack channel   | Format Message for Slack   | None                      | ## Connect Slack: Use OAuth2 credentials to connect Slack account.                                     |
| Sticky Note                | Sticky Note                     | Documentation and explanation               | None                      | None                      | ## Summarize news articles for specific keywords and post to Slack.                                   |
| Sticky Note1               | Sticky Note                     | Schedule adjustment instructions            | None                      | None                      | ## Set Your Schedule: Default daily 9 AM trigger; editable schedule node.                              |
| Sticky Note2               | Sticky Note                     | Configuration instructions for keywords and Slack channel | None              | None                      | ## Configure Keywords & Slack Channel: How to customize keywords and Slack channel.                    |
| Sticky Note3               | Sticky Note                     | AI model credential setup instructions      | None                      | None                      | ## Connect Your AI Model: Use OpenRouter API key; can swap AI provider by replacing this node.         |
| Sticky Note4               | Sticky Note                     | Slack OAuth2 credential setup instructions  | None                      | None                      | ## Connect Slack: Use OAuth2 credentials to connect Slack account.                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 9:00 AM (adjust as needed).

2. **Add a Set node named "Configure Your Settings"**  
   - Define two fields:  
     - `keywords` (JSON): default value `["AI"]` (editable to track different or multiple keywords).  
     - `slack_channel` (String): default value `"news-updates"` (replace with your Slack channel name).  
   - Connect Schedule Trigger output to this node.

3. **Add a HackerNews node named "Get many items"**  
   - Resource: "all"  
   - Limit: 3  
   - Keyword: Use expression `{{$json.keywords[0]}}` from "Configure Your Settings" node.  
   - Connect "Configure Your Settings" output to this node.

4. **Add a LangChain Agent node named "Summarize Article with AI"**  
   - Text input: Use expression `{{$json.url}}` from "Get many items".  
   - System Message (AI prompt):  
     ```
     You are a highly skilled news summarization AI. Your task is to analyze the provided news article snippet and create a concise, easy-to-understand summary. The summary must be in Japanese, presented as 3 bullet points. Focus on the key information and maintain a neutral, professional tone. Do not add any introductory phrases.
     ```  
   - Enable passthroughBinaryImages if needed.  
   - Prompt Type: define.  
   - Connect "Get many items" output to this node.

5. **Add a LangChain LM Chat OpenRouter node named "OpenRouter Chat Model"**  
   - Default parameters.  
   - This node acts as the AI language model backend for the summarization agent.  
   - Connect this node in the AI Model field of "Summarize Article with AI".  
   - Configure OpenRouter API credentials for this node.

6. **Add a Set node named "Format Message for Slack"**  
   - Create a new string field `slackMessage` with value:  
     ```
     *{{$json.title}}*\n\n*AI Summary:*\n{{$json.output}}
     ```  
   - Include other fields as needed.  
   - Connect "Summarize Article with AI" output to this node.

7. **Add a Slack node named "Send Summary to Slack"**  
   - Text: Use expression `{{$json.slackMessage}}` from "Format Message for Slack".  
   - Channel: Use the channel ID or name (example uses a specific channel ID; replace this with your Slack channel).  
   - Authentication: Configure OAuth2 credentials with Slack app permissions to post messages.  
   - Connect "Format Message for Slack" output to this node.

8. **Add Sticky Notes at appropriate places to document configuration instructions and usage tips.**

9. **Test the workflow by running manually or waiting for the scheduled trigger.**

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                           |
|---------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| This workflow automatically fetches and summarizes news articles daily, helping users stay informed on specific topics.  | Workflow purpose                                          |
| Adjust the schedule by editing the "Trigger Daily at 9 AM" node to fit your preferred notification time.                   | Scheduling instructions                                   |
| Customize keywords and Slack channel in the "Configure Your Settings" node to target different news topics and channels. | Keyword and Slack channel configuration                   |
| Use OpenRouter API credentials in the AI model node; you can swap this AI provider by replacing the node accordingly.     | AI model integration                                      |
| Slack node requires OAuth2 credentials with permission to post messages in the target channel.                            | Slack integration setup                                   |

---

**Disclaimer:**  
This document is generated from an n8n workflow designed for legal, public data processing and respects all current content policies.