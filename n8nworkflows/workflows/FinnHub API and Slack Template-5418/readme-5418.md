FinnHub API and Slack Template

https://n8nworkflows.xyz/workflows/finnhub-api-and-slack-template-5418


# FinnHub API and Slack Template

### 1. Workflow Overview

This workflow, titled **Market Watcher Bot**, is designed to automate the retrieval and posting of daily company news for a predefined list of stock tickers. It leverages the **FinnHub API** to fetch the latest news articles about specific companies and then posts formatted summaries to a designated Slack channel. The process is scheduled to run every weekday morning at 9:15 AM.

The workflow is logically divided into the following blocks:

- **1.1 Scheduler Trigger:** Initiates the workflow on a set schedule (weekday mornings).
- **1.2 Prepare Tickers:** Defines and outputs the list of stock tickers to process.
- **1.3 Loop and Fetch News:** Iterates over each ticker, performs API calls to FinnHub to retrieve company news for the current date.
- **1.4 Format News:** Converts the raw news data into a Slack-friendly markdown message.
- **1.5 Post to Slack:** Sends the formatted news message to a specified Slack channel.
- **1.6 Wait:** Pauses briefly to respect API rate limits before processing the next ticker.
- **1.7 No Operation:** Acts as an endpoint to properly terminate the loop for items without data.

The workflow includes informative sticky notes that explain usage, configuration, and external resource links to aid users in setup and customization.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduler Trigger

- **Overview:**  
  This block triggers the entire workflow automatically every weekday at 9:15 AM.

- **Nodes Involved:**  
  - Daily Market News Trigger

- **Node Details:**  
  - **Daily Market News Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Cron expression "15 9 * * 1-5" to run at 9:15 AM Monday through Friday  
    - Inputs: None (trigger node)  
    - Outputs: Starts the flow by outputting empty data to the next node  
    - Potential Failures: Cron expression misconfiguration or n8n scheduler service downtime  
    - Version-specific: Uses typeVersion 1.2 (compatible with n8n scheduling features)  

#### 1.2 Prepare Tickers

- **Overview:**  
  Defines a static list of stock tickers to be processed.

- **Nodes Involved:**  
  - Prep Tickers  
  - Sticky Note (comment explaining this step)

- **Node Details:**  
  - **Prep Tickers**  
    - Type: Code (JavaScript)  
    - Configuration: Hardcoded list of tickers: `["AAPL", "META", "NVDA", "TSLA", "MSFT", "AMZN", "GOOG", "IAU", "IBIT", "QQQ", "SPY"]`  
    - Output: Array of JSON objects each containing a ticker symbol `{ ticker: "AAPL" }` etc.  
    - Inputs: Receives trigger from Schedule Trigger  
    - Outputs: Passes tickers array to SplitInBatches node  
    - Edge Cases: Empty or malformed ticker list would cause no API calls  
    - Notes: Users can customize ticker list here  
  - **Sticky Note** (near Prep Tickers)  
    - Content: Explains the tickers list is customizable  

#### 1.3 Loop and Fetch News

- **Overview:**  
  Iterates over each ticker to retrieve the latest company news from FinnHub API for the current date.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)  
  - HTTP Request (FinnHub API call)  
  - No Operation, do nothing (handles empty or end of loop)  
  - Sticky Note (instructions on API setup)  

- **Node Details:**  
  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Configuration: Default batch size (1 item per batch) to process tickers sequentially  
    - Inputs: Receives array of tickers from Prep Tickers  
    - Outputs: Two outputs:  
      - Output 1: to No Operation (presumably for empty batches or end flow)  
      - Output 2: to HTTP Request node for API calls  
    - Edge Cases: Batch size misconfiguration could cause rate limiting or concurrency issues  
  - **HTTP Request**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://finnhub.io/api/v1/company-news`  
      - Query Parameters:  
        - `symbol`: dynamically set to current ticker  
        - `from` and `to`: both set to current date (ISO format)  
      - Authentication: Header Auth with FinnHub API Key  
    - Inputs: Single ticker item from SplitInBatches  
    - Outputs: JSON response with news articles  
    - Edge Cases: API auth failure, rate limit exceeded, no news found for ticker, network errors  
  - **No Operation, do nothing**  
    - Type: NoOp  
    - Role: Acts as a sink for one of SplitInBatches outputs (likely end of branches)  
    - Inputs: From SplitInBatches output 1  
    - Outputs: None  
  - **Sticky Note1** (near HTTP Request)  
    - Content: Explains how to get FinnHub API key and setup header auth in n8n  

#### 1.4 Format News

- **Overview:**  
  Formats the retrieved news articles into a markdown message suitable for Slack posting.

- **Nodes Involved:**  
  - Format (Code node)  
  - Sticky Note (explains formatting logic)  

- **Node Details:**  
  - **Format**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Checks if news items have a `related` field (ticker symbol)  
      - Builds a markdown message starting with ticker and "Daily Market News" headline  
      - Loops through up to 5 news items, appending each as a clickable markdown link (`- <url|headline>`)  
      - If no news found, writes "No news found for this ticker"  
      - Returns an object with `output` key containing the message string  
    - Inputs: JSON array of news items from HTTP Request  
    - Outputs: Formatted message string to Slack node  
    - Edge Cases: Empty news array, missing fields, unexpected data format  
  - **Sticky Note2** (near Format node)  
    - Content: Describes posting to Slack and Slack webhook setup  

#### 1.5 Post to Slack

- **Overview:**  
  Sends the formatted news message to a Slack channel using a webhook.

- **Nodes Involved:**  
  - Slack node  
  - Sticky Note (Slack webhook explanation)  

- **Node Details:**  
  - **Slack**  
    - Type: Slack node  
    - Configuration:  
      - Text: Uses expression to send formatted news message (`{{$json.output}}`)  
      - Channel: `#stock-market` (by name)  
      - Options: Disables link to workflow in message  
      - Credentials: Slack API token configured with webhook permissions  
    - Input: Receives formatted message from Format node  
    - Output: Passes data to Wait node  
    - Edge Cases: Slack auth failure, invalid channel, message length limits, rate limiting  
  - **Sticky Note2** (near Slack node)  
    - Content: Provides instructions and link for creating Slack webhook  

#### 1.6 Wait

- **Overview:**  
  Introduces a delay of 5 seconds between processing tickers to prevent hitting FinnHub API rate limits.

- **Nodes Involved:**  
  - Wait node  
  - Sticky Note (explains purpose of wait)  

- **Node Details:**  
  - **Wait**  
    - Type: Wait  
    - Configuration: Default (5 seconds implied by sticky note)  
    - Input: From Slack node  
    - Output: Loops back to SplitInBatches node to process next ticker  
    - Edge Cases: Delay misconfiguration could cause workflow slowdowns or API throttling  
  - **Sticky Note3**  
    - Content: Explains wait duration is customizable to avoid API rate limits  

---

### 3. Summary Table

| Node Name                 | Node Type            | Functional Role                          | Input Node(s)              | Output Node(s)          | Sticky Note                                                                                      |
|---------------------------|----------------------|----------------------------------------|----------------------------|-------------------------|-------------------------------------------------------------------------------------------------|
| Daily Market News Trigger | Schedule Trigger      | Initiates workflow on schedule         | -                          | Prep Tickers            | Cron scheduler: currently runs weekdays at 9:15 AM. Customizable.                               |
| Prep Tickers              | Code                 | Prepares list of stock tickers          | Daily Market News Trigger   | Loop Over Items         | Prepare the list of tickers. Feel free to update the list.                                     |
| Loop Over Items           | SplitInBatches       | Loops over tickers to batch process     | Prep Tickers                | No Operation, HTTP Request |                                                                                                 |
| No Operation, do nothing  | NoOp                 | Terminal node for one batch output      | Loop Over Items             | -                       |                                                                                                 |
| HTTP Request              | HTTP Request         | Calls FinnHub API for company news      | Loop Over Items             | Format                  | Retrieve company news via FinnHub API. See FinnHub API site for free API key and setup.         |
| Format                   | Code                 | Formats news into Slack markdown message | HTTP Request                | Slack                   | Post to Slack Channel via webhook. See Slack API docs for incoming webhook setup.               |
| Slack                    | Slack                | Posts formatted message to Slack channel| Format                     | Wait                    | Slack channel webhook setup instructions with link to Slack messaging webhooks documentation.  |
| Wait                     | Wait                 | Waits between ticker processing         | Slack                      | Loop Over Items         | Wait for 5 seconds to avoid FinnHub API rate limits; customizable wait time.                    |
| Sticky Note7             | Sticky Note          | Explains entire workflow and usage      | -                          | -                       | Daily Company News Bot explanation with blog and forum links.                                  |
| Sticky Note              | Sticky Note          | Explains ticker preparation step        | -                          | -                       | Prepare the list of the tickers. Feel free to update the list.                                 |
| Sticky Note1             | Sticky Note          | Explains FinnHub API setup               | -                          | -                       | FinnHub FREE API key acquisition and header auth setup instructions.                           |
| Sticky Note2             | Sticky Note          | Explains Slack webhook setup             | -                          | -                       | Slack webhook creation instructions and link to official Slack webhook docs.                   |
| Sticky Note3             | Sticky Note          | Explains wait node purpose                | -                          | -                       | Wait 5 seconds to avoid sending too many requests to FinnHub at once.                          |
| Sticky Note4             | Sticky Note          | Explains scheduler customization          | -                          | -                       | Cron scheduler currently set to 9:15 AM weekdays; customizable.                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow and name it "Market Watcher Bot".**

2. **Add a Schedule Trigger node:**  
   - Set trigger type to "Cron"  
   - Use cron expression: `15 9 * * 1-5` (runs at 9:15 AM Monday to Friday)  
   - Connect output to next node  

3. **Add a Code node named "Prep Tickers":**  
   - Paste the following JavaScript in the code editor:  
     ```javascript
     const tickers = ["AAPL", "META", "NVDA", "TSLA", "MSFT", "AMZN", "GOOG", "IAU", "IBIT", "QQQ", "SPY"];
     let output = [];
     for(const item of tickers) {
       output.push({ticker: item});
     }
     return output;
     ```  
   - Connect Schedule Trigger output to this node  

4. **Add a SplitInBatches node named "Loop Over Items":**  
   - Default batch size is fine (process one ticker at a time)  
   - Connect "Prep Tickers" output to this node  

5. **Add an HTTP Request node named "HTTP Request":**  
   - Set Method: GET  
   - URL: `https://finnhub.io/api/v1/company-news`  
   - Query Parameters:  
     - `symbol`: Expression: `{{$json["ticker"]}}`  
     - `from`: Expression: `{{ $now.toLocal().toISO().split('T')[0] }}`  
     - `to`: Expression: `{{ $now.toLocal().toISO().split('T')[0] }}`  
   - Authentication: HTTP Header Auth  
   - Create FinnHub API Key credential in n8n with your API key, select it here  
   - Connect from "Loop Over Items" output (main output for API calls) to this node  

6. **Add a Code node named "Format":**  
   - Paste the following JS code:  
     ```javascript
     let msg = '*' + ($input.first().json.related ? $input.first().json.related : $('Loop Over Items').first().json.ticker) + '* - Daily Market News :newspaper: \n\n';

     let count = 0;

     if ($input.first().json.related) {
       for (const item of $input.all()) {
         count++;
         if (count <= 5) {
           msg += '- <' + item.json.url + '|' + item.json.headline + '>\n';
         } else {
           break;
         }
       }
     } else {
       msg += 'No news found for this ticker';
     }

     return {
       output: msg
     }
     ```  
   - Connect HTTP Request output to this node  

7. **Add a Slack node named "Slack":**  
   - Set "Text" field to expression: `{{$json.output}}`  
   - Select channel by name: enter `#stock-market` or your preferred channel  
   - Disable "Include Link To Workflow" option  
   - Create and select Slack credential with API token that has webhook permissions  
   - Connect "Format" node output to Slack node  

8. **Add a Wait node named "Wait":**  
   - Configure wait time to 5 seconds (or desired delay)  
   - Connect Slack node output to Wait node output  

9. **Connect Wait node output back to "Loop Over Items" node to continue processing next batch.**

10. **Add a No Operation node named "No Operation, do nothing":**  
    - Connect the other output of "Loop Over Items" (the first output) to this NoOp node  
    - This node acts as an endpoint for empty batches or flow completion  

11. **Add explanatory Sticky Notes at appropriate places:**  
    - Near Prep Tickers: "Prepare the list of the tickers. Feel free to update the list."  
    - Near HTTP Request: Instructions for obtaining FinnHub API key and setting header auth  
    - Near Slack: Instructions and link for Slack webhook setup: https://api.slack.com/messaging/webhooks  
    - Near Wait: Explanation for wait purpose to avoid API rate limits  
    - Near Schedule Trigger: Note on cron schedule customization  
    - Add a large Sticky Note summarizing the workflow purpose, usage, and help links:  
      - Blog contact: https://fans-ai-lab.com/contact  
      - n8n Community forum: https://community.n8n.io/  

12. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                              |
|----------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------|
| This workflow demonstrates usage of free FinnHub API to retrieve company news and post to Slack channel automatically on schedule. | Workflow overview and usage guidance                                        |
| For FinnHub API key registration and documentation, visit: https://finnhub.io/                                                  | FinnHub API official site                                                   |
| Slack webhook setup instructions and best practices: https://api.slack.com/messaging/webhooks                                     | Slack API documentation                                                     |
| Contact and help available via blog: https://fans-ai-lab.com/contact and n8n Forum: https://community.n8n.io/                    | Support and community                                                      |
| Cron expression "15 9 * * 1-5" schedules workflow at 9:15 AM on weekdays; customize as needed.                                     | Scheduling customization                                                    |
| Wait node delay is set to 5 seconds to avoid FinnHub API rate limiting; adjust delay based on your API plan and throughput needs.| API rate limit management                                                   |

---

**Disclaimer:**  
The provided content is generated exclusively from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.