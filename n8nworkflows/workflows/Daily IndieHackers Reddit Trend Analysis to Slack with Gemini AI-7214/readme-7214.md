Daily IndieHackers Reddit Trend Analysis to Slack with Gemini AI

https://n8nworkflows.xyz/workflows/daily-indiehackers-reddit-trend-analysis-to-slack-with-gemini-ai-7214


# Daily IndieHackers Reddit Trend Analysis to Slack with Gemini AI

### 1. Workflow Overview

This workflow automates a daily analysis of trending posts from the r/indiehackers subreddit and delivers actionable startup intelligence to a designated Slack channel using AI-powered insights. It is designed for startup founders, product managers, and growth teams interested in identifying hot topics, traction signals, and thematic trends within the indie hacker community.

The workflow logically divides into these blocks:

- **1.1 Scheduled Trigger:** Runs the workflow daily at a configured time.
- **1.2 Data Acquisition:** Fetches recent Reddit posts from r/indiehackers.
- **1.3 Data Preparation:** Prepares and lightly processes raw Reddit posts for AI analysis.
- **1.4 AI Analysis:** Uses Google Gemini AI to analyze posts for hotness scoring, topic extraction, theme clustering, traction signals, and actionable insights.
- **1.5 AI Post-Processing:** Parses or enhances AI output if needed.
- **1.6 Slack Notification:** Formats and sends the analysis results as a Slack Block Kit message to a selected channel.
- **1.7 Supplementary AI Module (Optional):** A Groq-powered AI agent node configured but not linked in the main flow, potentially for alternative or extended AI processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:** Initiates the workflow every day at 8:00 AM.
- **Nodes Involved:**  
  - *Daily Schedule*
- **Node Details:**
  - **Daily Schedule**  
    - Type: Cron Trigger  
    - Configured to trigger once daily at hour 8 (8:00 AM).  
    - No input nodes; outputs trigger event to "Get many posts".  
    - Potential failure: Cron misconfiguration or n8n scheduler issues.  
    - No special version constraints.

#### 2.2 Data Acquisition

- **Overview:** Retrieves the latest Reddit posts from r/indiehackers, limited to 5 posts by default.
- **Nodes Involved:**  
  - *Get many posts*
- **Node Details:**
  - **Get many posts**  
    - Type: Reddit OAuth2 node  
    - Operation: Get all (latest posts)  
    - Subreddit: indiehackers  
    - Limit: 5 posts  
    - OAuth2 credentials required for Reddit API access.  
    - Inputs: Triggered by "Daily Schedule"  
    - Outputs: Raw post data JSON array to "Extract Posts Data"  
    - Edge cases: API rate limits, OAuth token expiration, subreddit restrictions, empty results.

#### 2.3 Data Preparation

- **Overview:** Applies minimal processing to each Reddit post; currently a placeholder for potential JSON transformations.
- **Nodes Involved:**  
  - *Extract Posts Data*
- **Node Details:**
  - **Extract Posts Data**  
    - Type: Code node (JavaScript)  
    - Operation: Adds a new field `myNewField` with value 1 to each item (likely a placeholder or for debugging).  
    - Input: Reddit posts from "Get many posts"  
    - Output: Modified posts to "AI Intent Analysis"  
    - Edge cases: Code errors, empty input, malformed JSON.

#### 2.4 AI Analysis

- **Overview:** Performs a detailed, structured AI-driven analysis of the Reddit posts using Google Gemini (PaLM) AI. Tasks include sanity checks, hotness scoring, topic extraction, theme clustering, traction signal extraction, and actionable insight generation.
- **Nodes Involved:**  
  - *AI Intent Analysis*
- **Node Details:**
  - **AI Intent Analysis**  
    - Type: Google Gemini AI node (via LangChain integration)  
    - Model: `models/gemini-2.5-flash-lite-preview-06-17`  
    - Temperature: 0.3 (low randomness for consistent results)  
    - Input: JSON array of Reddit posts, passed from "Extract Posts Data"  
    - Prompt: Extensive instructions for structured analysis, including formulas and output format.  
    - Credential: Google Palm API key required.  
    - Output: AI-generated structured analysis text (JSON string with sections: hot posts, themes, insights, etc.)  
    - Edge cases: API throttling, malformed input, output truncation, unexpected AI responses.

#### 2.5 AI Post-Processing

- **Overview:** Intended to parse or further prepare the AI response, but currently contains placeholder code.
- **Nodes Involved:**  
  - *Parse AI Response*
- **Node Details:**
  - **Parse AI Response**  
    - Type: Code node (JavaScript)  
    - Operation: Adds `myNewField` = 1 to every item, placeholder functionality.  
    - Input: AI response from "AI Intent Analysis"  
    - Output: Passed to "AI Agent" (optional flow)  
    - Edge cases: Code failures, empty input.

#### 2.6 Slack Notification

- **Overview:** Sends the AI-generated analysis summary to a specified Slack channel using Block Kit formatting, enabling team members to quickly grasp current indie hacker trends.
- **Nodes Involved:**  
  - *AI Agent*  
  - *Send to Slack*
- **Node Details:**
  - **AI Agent**  
    - Type: LangChain AI agent node  
    - Inputs: Receives data from "Parse AI Response" and optionally from "Groq Chat Model" (though "Groq Chat Model" is not connected in main flow)  
    - Prompt: Detailed instructions to generate a compact, well-structured Slack Block Kit JSON message summarizing Reddit trends.  
    - Parameters include context (channel type, audience, call-to-action link, timeframe label, thread invitation text).  
    - Output: JSON object formatted for Slack Block Kit  
    - Edge cases: AI output errors, JSON formatting failures.  
    - Credential: Groq API credentials for optional language model invocation.  
  - **Send to Slack**  
    - Type: Slack node (OAuth2)  
    - Channel: Configurable; default ID `C098Y3YJC3C` (named "product-ai")  
    - Message Text: Extracted from AI Agent's JSON output (`content.parts[0].text`), expected to be the Slack Block Kit JSON string  
    - Options: Markdown enabled for rich formatting  
    - On error: Continue workflow to avoid blocking on Slack API issues  
    - Credential: Slack OAuth2 API required  
    - Edge cases: Slack API rate limits, invalid token, channel not found.

#### 2.7 Supplementary AI Module (Not in main flow)

- **Overview:** A Groq Chat Model node is configured, possibly as an alternative AI processor or for extended analysis, but it is not integrated into the main workflow chain except connected as a language model input to the "AI Agent".
- **Nodes Involved:**  
  - *Groq Chat Model*
- **Node Details:**
  - **Groq Chat Model**  
    - Type: LangChain model node using Groq’s GPT-OSS-120B model  
    - No special parameterization, uses default options  
    - Credential: Groq API  
    - Input: Not triggered by main flow but linked as a language model provider to "AI Agent"  
    - Edge cases: API availability, model response variability.

#### 2.8 Documentation and Setup Notes

- **Overview:** A large sticky note node documents the workflow purpose, requirements, setup instructions, customization tips, and links.
- **Nodes Involved:**  
  - *Sticky Note*
- **Node Details:**
  - **Sticky Note**
    - Contains markdown-formatted text with branding, usage instructions, and example configurations.  
    - No inputs or outputs; informational only.  
    - Important for user onboarding and credential setup.

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                             | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                                                    |
|---------------------|----------------------------------|--------------------------------------------|-------------------------|-----------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Daily Schedule      | Cron Trigger                     | Triggers workflow daily at 8:00 AM          | —                       | Get many posts        | See overall workflow setup and scheduling instructions in Sticky Note                                                          |
| Get many posts      | Reddit OAuth2                   | Fetches latest 5 posts from r/indiehackers | Daily Schedule           | Extract Posts Data     | Requires Reddit OAuth2 credentials; configurable post limit                                                                     |
| Extract Posts Data  | Code (JavaScript)                | Adds placeholder field to posts JSON        | Get many posts           | AI Intent Analysis    | Placeholder code; potential area for custom preprocessing                                                                        |
| AI Intent Analysis  | Google Gemini AI (LangChain)     | Performs structured Reddit post analysis    | Extract Posts Data       | Parse AI Response      | Requires Google Palm API credentials; prompt includes detailed analysis logic                                                   |
| Parse AI Response   | Code (JavaScript)                | Placeholder for processing AI response      | AI Intent Analysis       | AI Agent               | Placeholder code; currently no transformation                                                                                    |
| AI Agent            | LangChain AI Agent               | Formats AI output into Slack Block Kit JSON | Parse AI Response, Groq Chat Model (LM input) | Send to Slack       | Uses Groq API for language model; generates Slack message JSON per instructions                                                  |
| Send to Slack       | Slack OAuth2                    | Sends analysis summary message to Slack     | AI Agent                 | —                     | Requires Slack OAuth2; posts to channel “product-ai” by default                                                                  |
| Groq Chat Model     | LangChain LM (Groq GPT-OSS-120B)| Optional LM for AI Agent usage               | —                       | AI Agent (LM input)    | Not triggered in main flow; configured as LM for AI Agent                                                                       |
| Sticky Note         | Sticky Note                     | Documentation and setup instructions         | —                       | —                     | Contains detailed workflow purpose, requirements, and setup guidance                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron node named "Daily Schedule":**  
   - Type: Cron Trigger  
   - Set to trigger daily at hour 8 (8:00 AM)  
   - No further parameters needed.

2. **Create a Reddit node named "Get many posts":**  
   - Type: Reddit OAuth2  
   - Operation: Get All (latest posts)  
   - Subreddit: `indiehackers`  
   - Limit: 5 posts (adjustable)  
   - Connect input from "Daily Schedule" node.  
   - Configure Reddit OAuth2 credentials.

3. **Create a Code node named "Extract Posts Data":**  
   - Type: Code (JavaScript)  
   - Code snippet: Loop over all input items and add a field `myNewField = 1` to each JSON object (placeholder).  
   - Connect input from "Get many posts" node.

4. **Create a Google Gemini AI node named "AI Intent Analysis":**  
   - Type: Google Gemini via LangChain  
   - Model ID: `models/gemini-2.5-flash-lite-preview-06-17`  
   - Temperature: 0.3  
   - Paste the detailed prompt instructing the AI to analyze Reddit posts, compute hotness scores, extract topics, cluster themes, identify traction signals, and output a concise summary.  
   - Input connected from "Extract Posts Data".  
   - Configure Google Palm API credentials.

5. **Create a Code node named "Parse AI Response":**  
   - Type: Code (JavaScript)  
   - Same placeholder code adding `myNewField = 1` to each item.  
   - Connect input from "AI Intent Analysis".

6. **Create a LangChain AI Agent node named "AI Agent":**  
   - Type: LangChain Agent  
   - Provide a prompt that instructs generating a Slack Block Kit JSON message summarizing the Reddit trends with sections for hot posts, themes, actionable insights, and call-to-action links.  
   - Context inputs include channel type, audience, dashboard URL, timeframe label, and thread invitation note.  
   - Connect input from "Parse AI Response".  
   - Configure Groq API credentials.  
   - Configure Groq Chat Model node (see next step) and connect it as the language model input for this node.

7. **Create a LangChain LM node named "Groq Chat Model":**  
   - Type: LangChain LM using Groq GPT-OSS-120B  
   - No special parameters needed.  
   - Connect output as AI language model input to "AI Agent".

8. **Create a Slack node named "Send to Slack":**  
   - Type: Slack OAuth2  
   - Select target channel (default: `C098Y3YJC3C` named "product-ai")  
   - Message text expression: `={{ $json.content.parts[0].text }}` (extracts Slack Block Kit JSON string from AI Agent output)  
   - Enable Markdown formatting  
   - On error: Continue workflow  
   - Connect input from "AI Agent".  
   - Configure Slack OAuth2 credentials.

9. **Add a Sticky Note node for documentation:**  
   - Add markdown content describing workflow purpose, credential setup, scheduling, Slack channel configuration, and analysis customization tips.

**Connect the nodes in this order:**  
`Daily Schedule` → `Get many posts` → `Extract Posts Data` → `AI Intent Analysis` → `Parse AI Response` → `AI Agent` → `Send to Slack`

**Ensure credentials are configured for:**  
- Reddit OAuth2 API  
- Google Palm API (Gemini)  
- Groq API  
- Slack OAuth2 API

**Adjust default parameters as needed:**  
- Post limit in Reddit node (recommended 3-10)  
- Slack channel selection  
- Scheduling time in Cron node  
- Context parameters in AI Agent prompt (audience, timeframe, CTA link)

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow provides daily AI-powered trend analysis of r/indiehackers posts with actionable startup insights delivered to Slack. | Purpose summary                                                                                   |
| Requires OAuth2 credentials for Reddit, Slack, Google Gemini (PaLM), and Groq APIs. All offer free tiers. | Credential requirements                                                                           |
| Default schedule is daily at 8:00 AM; can be customized in "Daily Schedule" node’s cron parameters.  | Scheduling customization example provided in Sticky Note                                         |
| Slack messages use Block Kit JSON for rich formatting, including sections, context, and buttons.     | Slack formatting guidelines embedded in AI Agent prompt                                          |
| AI prompt includes detailed hotness scoring formula and constraints to avoid hallucination.          | Prompt design best practices                                                                      |
| Example Slack channel ID: `C098Y3YJC3C` (named "product-ai"); change per your workspace setup.        | Slack channel configuration                                                                       |
| Links to dashboard or reports can be embedded as CTAs in Slack messages via context parameters.       | Customizing call-to-action links in AI Agent node                                                |
| Sticky Note node contains comprehensive setup and usage instructions, including credential setup and context customization JSON snippet. | User onboarding and maintenance guide in Sticky Note node                                         |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow and strictly complies with content policies. It contains no illegal, offensive, or protected material. All processed data are legal and public.