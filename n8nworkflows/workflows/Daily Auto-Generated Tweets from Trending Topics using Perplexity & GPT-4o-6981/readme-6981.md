Daily Auto-Generated Tweets from Trending Topics using Perplexity & GPT-4o

https://n8nworkflows.xyz/workflows/daily-auto-generated-tweets-from-trending-topics-using-perplexity---gpt-4o-6981


# Daily Auto-Generated Tweets from Trending Topics using Perplexity & GPT-4o

---

### 1. Workflow Overview

This workflow, named **"Daily Auto-Generated Tweets from Trending Topics using Perplexity & GPT-4o"**, automates the process of generating and posting daily tweets based on current trending topics within a specified industry or subject area.

The primary use case is to provide an intelligent Twitter bot that:

- Automatically fetches the latest high-value developments from a selected domain
- Uses AI to craft human-like, engaging tweets from raw data
- Publishes these tweets daily on a Twitter (X) account

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger**: Initiates the workflow once per day at a defined hour.
- **1.2 Data Collection (Perplexity API)**: Calls the Perplexity AI API to retrieve concise, high-signal summaries of recent important developments in the target industry.
- **1.3 AI Processing (OpenAI GPT-4o)**: Uses OpenAI’s GPT-4o model to rewrite the Perplexity summaries into a single compelling, human-sounding tweet.
- **1.4 Posting to Twitter**: Sends the generated tweet text to Twitter’s API to publish it.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  This block starts the entire workflow automatically every day at 9 AM, ensuring the bot runs on a consistent daily schedule.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* n8n-nodes-base.scheduleTrigger  
    - *Role:* Time-based workflow initiator  
    - *Configuration:* Set to trigger once daily at 9:00 AM (hour-based interval)  
    - *Input:* None (start node)  
    - *Output:* Triggers "HTTP Request" node downstream  
    - *Version:* 1.2  
    - *Potential Failures:* Rare; possible system clock issues or n8n scheduler malfunctions  
    - *Notes:* Critical for automation timing; ensure server timezone aligns with intended schedule

#### 2.2 Data Collection (Perplexity API)

- **Overview:**  
  This block calls the Perplexity AI API to receive a concise, high-value summary of the latest developments in a user-defined topic or industry, emphasizing novelty and recency.

- **Nodes Involved:**  
  - HTTP Request (named "HTTP Request")

- **Node Details:**

  - **HTTP Request ("HTTP Request")**  
    - *Type:* n8n-nodes-base.httpRequest  
    - *Role:* Query Perplexity AI’s chat completions endpoint to fetch trending insights  
    - *Configuration:*  
      - POST request to `https://api.perplexity.ai/chat/completions`  
      - JSON body includes a prompt requesting "most important or interesting developments from the last 24–48 hours" on a placeholder topic `[ENTER YOUR TOPIC OR INDUSTRY]`  
      - Headers include Authorization Bearer token (value not set in JSON) and Content-Type `application/json`  
    - *Key Expression:* Static JSON body with imperative prompt for Perplexity AI  
    - *Input:* Trigger from Schedule Trigger  
    - *Output:* Passes Perplexity’s response JSON to the OpenAI node  
    - *Version:* 4.2  
    - *Edge Cases / Failures:*  
      - Missing or invalid API key causing auth errors  
      - Network timeouts or slow API responses  
      - Malformed or empty API response resulting in downstream errors  
      - Placeholder `[ENTER YOUR TOPIC OR INDUSTRY]` must be replaced before running to get relevant data  
    - *Notes:* Requires valid Perplexity API credentials and correct prompt customization

#### 2.3 AI Processing (OpenAI GPT-4o)

- **Overview:**  
  This block transforms the raw Perplexity summary into a polished, human-like tweet designed to engage followers with insight, tone, and rhetorical devices.

- **Nodes Involved:**  
  - OpenAI (named "OpenAI")

- **Node Details:**

  - **OpenAI ("OpenAI")**  
    - *Type:* @n8n/n8n-nodes-langchain.openAi  
    - *Role:* Rewrite Perplexity content into a compelling tweet using GPT-4o  
    - *Configuration:*  
      - Model: `chatgpt-4o-latest`  
      - System prompt instructs the AI to write in a sharp, human voice, avoiding AI clichés and generic phrases  
      - User prompt includes the Perplexity summary inserted dynamically via expression `{{ $json.choices[0].message.content }}`  
      - Output is JSON with the tweet content  
    - *Key Expressions:*  
      - System prompt provides detailed style and rhetorical instructions  
      - User message dynamically injects data from the previous node’s response  
    - *Input:* Perplexity API response JSON  
    - *Output:* JSON containing a tweet string passed to the Twitter posting node  
    - *Version:* 1.8  
    - *Edge Cases / Failures:*  
      - Invalid or expired OpenAI API credentials causing auth errors  
      - Rate limiting or timeouts from OpenAI API  
      - Improper JSON parsing if Perplexity response unexpected  
      - Model not available or deprecated  
    - *Credentials:* Requires OpenAI API credentials configured in n8n

#### 2.4 Posting to Twitter

- **Overview:**  
  This block publishes the generated tweet text on Twitter (now called X) using OAuth2 authentication.

- **Nodes Involved:**  
  - HTTP Request1 (named "HTTP Request1")

- **Node Details:**

  - **HTTP Request ("HTTP Request1")**  
    - *Type:* n8n-nodes-base.httpRequest  
    - *Role:* Post the crafted tweet to Twitter API v2 endpoint  
    - *Configuration:*  
      - POST request to `https://api.twitter.com/2/tweets`  
      - JSON body includes `"text": "{{ $json.message.content.tweet }}"` — dynamically injecting the tweet from OpenAI output  
      - OAuth2 authentication preset with Twitter credentials  
    - *Input:* JSON from OpenAI node  
    - *Output:* Twitter API response (confirmation or error)  
    - *Version:* 4.2  
    - *Edge Cases / Failures:*  
      - OAuth token expired or invalid, causing auth failures  
      - Twitter API rate limits or temporary outages  
      - Exceeding tweet character limits (should be handled upstream but possible)  
      - API changes or endpoint deprecation  
    - *Credentials:* Twitter OAuth2 API credentials must be set up in n8n

---

### 3. Summary Table

| Node Name        | Node Type                     | Functional Role                        | Input Node(s)     | Output Node(s)   | Sticky Note                                                                                          |
|------------------|-------------------------------|-------------------------------------|-------------------|------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Initiate workflow daily at 9 AM      | None              | HTTP Request     | ✅ Automated Daily Tweet Bot This workflow pulls trending insights from Perplexity, summarizes them using OpenAI, and tweets the result. 🧠 Workflow Steps: Schedule Trigger – Runs once per day. HTTP Request (Perplexity) – Pulls real-time insights from the web. OpenAI (GPT) – Summarizes & formats the insight as a tweet. HTTP Request (Twitter API) – Posts the tweet to X/Twitter. |
| HTTP Request     | n8n-nodes-base.httpRequest     | Fetch trending summaries from Perplexity AI | Schedule Trigger  | OpenAI           | See above                                                                                        |
| OpenAI           | @n8n/n8n-nodes-langchain.openAi | Rewrite summaries into engaging tweets | HTTP Request      | HTTP Request1    | See above                                                                                        |
| HTTP Request1    | n8n-nodes-base.httpRequest     | Post tweet to Twitter/X API         | OpenAI            | None             | See above                                                                                        |
| Sticky Note      | n8n-nodes-base.stickyNote      | Documentation and explanation       | None              | None             | ✅ Automated Daily Tweet Bot This workflow pulls trending insights from Perplexity, summarizes them using OpenAI, and tweets the result. 🧠 Workflow Steps: Schedule Trigger – Runs once per day. HTTP Request (Perplexity) – Pulls real-time insights from the web. OpenAI (GPT) – Summarizes & formats the insight as a tweet. HTTP Request (Twitter API) – Posts the tweet to X/Twitter. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow and name it** (e.g., "Daily Auto-Generated Tweets from Trending Topics").

2. **Add a Schedule Trigger node**:  
   - Node Type: Schedule Trigger  
   - Configure to trigger once daily at 9:00 AM (set interval to daily, triggerAtHour: 9)  
   - No credentials needed  
   - Connect output to next node

3. **Add an HTTP Request node for Perplexity API**:  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.perplexity.ai/chat/completions`  
   - Authentication: None (token sent in header)  
   - Headers:  
     - Authorization: `Bearer YOUR_PERPLEXITY_API_KEY` (replace with valid token)  
     - Content-Type: `application/json`  
   - Body Parameters (raw JSON mode):  
   ```json
   {
     "model": "sonar-pro",
     "messages": [
       {
         "role": "user",
         "content": "What are the most important or interesting developments from the last 24–48 hours in the [ENTER YOUR TOPIC OR INDUSTRY] space? Include key names, links, and short, high-signal summaries. Prioritize novelty, usefulness, and recency. Keep it concise and skip the fluff."
       }
     ]
   }
   ```  
   - Replace `[ENTER YOUR TOPIC OR INDUSTRY]` with your real target topic  
   - Connect Schedule Trigger output to this node’s input

4. **Add an OpenAI node for tweet generation**:  
   - Node Type: OpenAI (LangChain integration)  
   - Model: select `chatgpt-4o-latest`  
   - Credentials: Configure OpenAI API credentials in n8n (API Key)  
   - Messages configuration:  
     - System message: detailed prompt instructing GPT-4o to write a human-like tweet with rhetorical devices, avoid AI clichés, and end with a hook  
     - User message: inject Perplexity API’s response content dynamically using expression syntax like:  
       ```{{ $json.choices[0].message.content }}```  
   - Set output format as JSON, returning the generated tweet text  
   - Connect Perplexity HTTP Request output to OpenAI input

5. **Add an HTTP Request node to post to Twitter (X)**:  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.twitter.com/2/tweets`  
   - Authentication: OAuth2 with Twitter credentials configured in n8n  
   - Body Parameters (raw JSON mode or expression):  
     ```json
     {
       "text": "{{ $json.message.content.tweet }}"
     }
     ```  
   - Connect OpenAI node output to this node’s input

6. **Verify all credentials:**  
   - Perplexity API token in HTTP Request headers  
   - OpenAI API key in OpenAI node credentials  
   - Twitter OAuth2 credentials properly configured in n8n, with posting permissions

7. **Test workflow manually before activating:**  
   - Trigger the workflow to ensure each step executes correctly  
   - Confirm a tweet is posted on the target Twitter account

8. **Activate workflow** for daily automation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The placeholder `[ENTER YOUR TOPIC OR INDUSTRY]` in the Perplexity API prompt must be replaced with a real topic for relevant data. | Essential for meaningful trending topic summaries                                               |
| Twitter API v2 requires OAuth2 authentication with appropriate scopes to post tweets. Ensure your credentials have write access.   | Twitter Developer Portal: https://developer.twitter.com/en/docs/authentication/oauth-2-0        |
| The OpenAI node uses GPT-4o (chatgpt-4o-latest), a powerful model designed for human-like content creation.                       | OpenAI API documentation: https://platform.openai.com/docs/models/gpt-4o                        |
| To avoid character limit issues, the OpenAI prompt instructs tweets to stay under 280 characters, the max tweet length on Twitter. | Twitter tweet length rules: https://help.twitter.com/en/rules-and-policies/twitter-limits        |
| Sticky Note in the workflow documents the automation logic clearly for new maintainers or auditors.                                | Included directly in the workflow as a visual guide                                              |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---