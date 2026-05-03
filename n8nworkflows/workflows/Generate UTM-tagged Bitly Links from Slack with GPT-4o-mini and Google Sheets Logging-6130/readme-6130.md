Generate UTM-tagged Bitly Links from Slack with GPT-4o-mini and Google Sheets Logging

https://n8nworkflows.xyz/workflows/generate-utm-tagged-bitly-links-from-slack-with-gpt-4o-mini-and-google-sheets-logging-6130


# Generate UTM-tagged Bitly Links from Slack with GPT-4o-mini and Google Sheets Logging

### 1. Workflow Overview

This n8n workflow automates the creation of UTM-tagged Bitly short links initiated by Slack messages. The primary use case targets marketing teams and content creators who need to generate campaign-tracked links quickly and consistently. The workflow listens for Slack mentions, uses AI to extract and standardize UTM parameters, generates Bitly short URLs, logs all details to Google Sheets for campaign tracking, and responds back in Slack with the generated link and metadata.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception (Slack Trigger and User Info Retrieval):** Captures Slack messages where the bot is mentioned and fetches user profile information.
- **1.2 AI Processing (LangChain AI Agent and Memory):** Uses GPT-4o-mini to extract and normalize UTM parameters and target URLs from Slack messages, leveraging a session-based memory buffer.
- **1.3 Bitly Link Generation:** Creates a shortened URL with Bitly API using the AI-inferred target URL and UTM parameters.
- **1.4 Data Extraction and Validation:** Extracts structured UTM data and Bitly link from the AI response and validates the Bitly URL presence.
- **1.5 Logging and Response:** Appends the collected data into a Google Sheets document and replies in Slack thread with the shortened link and campaign details.
- **1.6 Error Handling:** Stops the workflow if the Bitly URL was not generated successfully.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures Slack mentions of the bot in a specific channel and retrieves the user’s profile to obtain the user’s real name for logging.

- **Nodes Involved:**  
  - Slack Trigger  
  - Slack Response  
  - Slack - Get User Name  

- **Node Details:**

  - **Slack Trigger**  
    - Type: Slack Trigger (event-driven)  
    - Role: Listens for Slack app mentions in a predefined channel (ID redacted for privacy).  
    - Configuration: Trigger set to "app_mention" with channel ID scoped. Resolves user and channel IDs for downstream use.  
    - Inputs: None (external event)  
    - Outputs: Slack message JSON including text, user, channel, and timestamps.  
    - Edge cases: Missing or invalid channel ID; permissions errors; Slack API rate limits.  

  - **Slack Response**  
    - Type: Slack node (message posting)  
    - Role: Posts a threaded reply in Slack with the AI-generated response text.  
    - Configuration: Replies in the same channel and thread as the trigger message using thread timestamp.  
    - Key Expression: Text content is set dynamically from AI Agent output.  
    - Inputs: AI Agent output (message text), Slack Trigger context.  
    - Outputs: Slack message confirmation.  
    - Edge cases: Slack API errors, permission issues replying in thread.

  - **Slack - Get User Name**  
    - Type: Slack API user profile fetch  
    - Role: Retrieves the real name of Slack user who triggered the workflow, to personalize logs.  
    - Configuration: Uses user ID from Slack Trigger node’s JSON.  
    - Inputs: Slack Trigger user ID  
    - Outputs: User profile information including real name.  
    - Edge cases: User deleted or deactivated, API permission errors, rate limits.

---

#### 2.2 AI Processing

- **Overview:**  
  Uses LangChain AI Agent powered by GPT-4o-mini to parse Slack message text, extract and normalize UTM parameters and target URL, and generate a Bitly short link. Maintains conversation context with a simple session memory keyed by Slack channel.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Simple Memory  

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain AI Agent node  
    - Role: Processes user Slack text input, extracts UTM parameters, normalizes them, and triggers Bitly link generation.  
    - Configuration:  
      - System message instructs the AI to: extract target URL and UTM parameters, normalize naming conventions (lowercase, underscores), convert common short forms (e.g., "IG" to "instagram"), restrict utm_medium to a specific whitelist, and generate a Bitly link via the Bitly tool node.  
      - Text input dynamically set from Slack message text.  
      - AI Agent uses OpenAI GPT-4o-mini as language model.  
    - Inputs: Slack Trigger text, Simple Memory context, OpenAI Chat Model.  
    - Outputs: Formatted response including Bitly link and UTM details.  
    - Edge cases: AI inference errors, incomplete information requiring follow-up, API timeouts or token limits, hallucinations.  
    - Notes: Prioritizes accuracy and asks clarifying questions if needed.

  - **OpenAI Chat Model**  
    - Type: OpenAI GPT-4o-mini language model node  
    - Role: Provides the language model backend for the AI Agent.  
    - Configuration: Model set explicitly to "gpt-4o-mini".  
    - Inputs: Text prompts from AI Agent.  
    - Outputs: AI-generated completions for UTM inference and link generation.  
    - Edge cases: API key invalid, rate limits, network issues.

  - **Simple Memory**  
    - Type: LangChain memory buffer  
    - Role: Maintains conversational context limited to last 10 messages per Slack channel session.  
    - Configuration: Session key based on Slack channel ID for scoped memory.  
    - Inputs: Slack Trigger channel info.  
    - Outputs: Context to AI Agent.  
    - Edge cases: Memory overflow or reset, session key misconfiguration.

---

#### 2.3 Bitly Link Generation

- **Overview:**  
  Creates the Bitly short link using the target URL and UTM parameters extracted by AI Agent.

- **Nodes Involved:**  
  - Bitly  

- **Node Details:**

  - **Bitly**  
    - Type: Bitly Tool node  
    - Role: Generates a shortened URL on Bitly platform.  
    - Configuration:  
      - Uses Bitly credentials (token, domain redacted).  
      - Long URL dynamically set from AI Agent output variable 'Long_URL'.  
    - Inputs: AI Agent output (Long_URL from AI-inferred data).  
    - Outputs: Shortened Bitly URL.  
    - Edge cases: Invalid token, API limits, malformed URL, Bitly service downtime.

---

#### 2.4 Data Extraction and Validation

- **Overview:**  
  Extracts structured UTM parameters, Bitly link, and user info from AI Agent response and Slack user data; validates presence of Bitly URL.

- **Nodes Involved:**  
  - Slack - Get User Name  
  - Information Extractor  
  - If  

- **Node Details:**

  - **Information Extractor**  
    - Type: LangChain information extractor  
    - Role: Parses AI Agent’s response text (from Slack Response) to extract structured fields: Bitly URL Link, utm_campaign, utm_source, utm_medium, utm_term, utm_content, Target URL, and User (from Slack profile).  
    - Configuration: Custom attributes defined with required flags and descriptions. User attribute dynamically set from Slack user real_name.  
    - Inputs: Slack Response message text, Slack - Get User Name output.  
    - Outputs: JSON object with extracted UTM and link data.  
    - Edge cases: Extraction failures if AI response format changes, missing fields, malformed text.

  - **If**  
    - Type: Conditional logic node  
    - Role: Checks if the 'Bitly URL Link' field is not empty to decide workflow continuation.  
    - Configuration: Condition set to string notEmpty on Bitly URL Link output from Information Extractor.  
    - Inputs: Information Extractor output.  
    - Outputs:  
      - True branch: continues to Google Sheets logging.  
      - False branch: triggers error handling.  
    - Edge cases: Unexpected missing Bitly URL, null values.

---

#### 2.5 Logging and Response

- **Overview:**  
  Logs all relevant campaign and URL data into Google Sheets and finalizes the Slack response with the generated information.

- **Nodes Involved:**  
  - Google Sheets  

- **Node Details:**

  - **Google Sheets**  
    - Type: Google Sheets append row  
    - Role: Appends a new row to a predefined Google Sheet with UTM parameters, Bitly URL, date, owner, status, and notes.  
    - Configuration:  
      - Document ID and Sheet name set to specific spreadsheet (redacted).  
      - Columns mapped explicitly from Information Extractor output fields and metadata: URL, Date (current date), Notes (fixed string), Owner (Slack user), Status (Done), UTM parameters, Bitly URL, and Title on Bitly (campaign name).  
      - Append operation mode.  
    - Inputs: Extracted structured data from Information Extractor, including user info and Bitly URL.  
    - Outputs: Confirmation of row append.  
    - Edge cases: Google API quota exceeded, permission errors, incorrect spreadsheet schema or missing columns.

---

#### 2.6 Error Handling

- **Overview:**  
  Stops workflow execution intentionally if Bitly URL is missing, signaling failure to generate the shortened link.

- **Nodes Involved:**  
  - Stop and Error  

- **Node Details:**

  - **Stop and Error**  
    - Type: Stop and Error node  
    - Role: Terminates workflow with an error message if Bitly URL Link is empty.  
    - Configuration: Static error message: "Workflow has stopped intentionally because Bitly URL Link does not exist or not generated"  
    - Inputs: If node false branch.  
    - Outputs: Stops workflow execution.  
    - Edge cases: None beyond logic correctness.

---

### 3. Summary Table

| Node Name           | Node Type                              | Functional Role                           | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                     |
|---------------------|--------------------------------------|-----------------------------------------|------------------------|------------------------|------------------------------------------------------------------------------------------------|
| Slack Trigger       | Slack Trigger                        | Captures Slack app mentions              | —                      | AI Agent               | Slack Trigger                                                                                   |
| AI Agent            | LangChain AI Agent                   | Extracts UTM params, normalizes, triggers Bitly link generation | Slack Trigger, Simple Memory, OpenAI Chat Model | Slack Response, Bitly (ai_tool) | Bitly AI Agent                                                                                  |
| OpenAI Chat Model   | OpenAI GPT-4o-mini                   | Provides language model engine for AI Agent | AI Agent (ai_languageModel) | AI Agent               |                                                                                                |
| Simple Memory       | LangChain Memory Buffer              | Maintains session context for AI Agent  | Slack Trigger           | AI Agent               |                                                                                                |
| Bitly               | Bitly Tool                          | Generates Bitly short link               | AI Agent (ai_tool)      | AI Agent               |                                                                                                |
| Slack Response      | Slack message posting                | Posts AI response in Slack thread        | AI Agent                | Slack - Get User Name   | Slack Response & Fetch User Name                                                               |
| Slack - Get User Name | Slack API user profile fetch         | Retrieves Slack user real name           | Slack Response          | Information Extractor   | Slack Response & Fetch User Name                                                               |
| Information Extractor | LangChain information extractor      | Parses AI response to extract UTM/link data | Slack - Get User Name, Slack Response | If                    | Information Extractor                                                                           |
| If                  | Conditional logic                   | Validates presence of Bitly URL          | Information Extractor   | Google Sheets, Stop and Error | If Node                                                                                       |
| Google Sheets        | Google Sheets append row             | Logs campaign and link data               | If (true branch)        | —                      | Update Log                                                                                    |
| Stop and Error       | Stop and Error                      | Stops workflow if Bitly URL is missing   | If (false branch)       | —                      |                                                                                                |
| Sticky Note          | Sticky Note                        | Documentation notes                       | —                      | —                      | Multiple notes across workflow: Slack Trigger, Bitly AI Agent, Slack Response, Information Extractor, If Node, Update Log, and workflow summary |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger Node**  
   - Type: Slack Trigger  
   - Trigger Event: `app_mention`  
   - Channel: Specify target Slack channel ID where the bot listens.  
   - Enable resolving IDs (user, channel) for downstream use.

2. **Add Simple Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Session Key: `={{ $('Slack Trigger').item.json.channel }}` to scope conversations by Slack channel.  
   - Context Window Length: 10 messages.

3. **Add OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4o-mini`  
   - Use default options.

4. **Add AI Agent Node**  
   - Type: LangChain AI Agent  
   - Text Input: `={{ $json.text }}` from Slack Trigger message text.  
   - System Message (configure prompt):  
     ```
     You are a bitly URL creator agent. Your job is to:
     1. Extract, infer and categorize accurately:
        a. Target URL
        b. utm_source
        c. utm_medium
        d. utm_campaign
        e. utm_term (if applicable)
        f. utm_content (if applicable)
     2. Normalize UTM naming: lowercase, underscore-separated, map short forms like "IG" to "instagram", restrict utm_medium to a whitelist.
     3. Create Bitly short link using Bitly Tool.
     Ask questions if info is missing. Prioritize accuracy.
     On success, respond with Bitly link and UTM details.
     ```
   - Connect AI Agent to OpenAI Chat Model (ai_languageModel) and Simple Memory (ai_memory).  
   - Configure AI Agent’s Bitly Tool integration.

5. **Add Bitly Tool Node**  
   - Type: Bitly Tool  
   - Long URL: `={{ $fromAI('Long_URL', '', 'string') }}` (override from AI Agent output)  
   - Domain: Set your Bitly domain (e.g., `bit.ly`).  
   - Set Bitly credentials with Bitly API token.

6. **Add Slack Response Node**  
   - Type: Slack (Send Message)  
   - Channel: same as Slack Trigger channel  
   - Text: AI Agent output text `={{ $json.output }}`  
   - Options: Post as threaded reply using `thread_ts` from Slack Trigger message timestamp.  
   - Connect AI Agent (main output) to Slack Response.

7. **Add Slack - Get User Name Node**  
   - Type: Slack (Get User Profile)  
   - User ID: `={{ $('Slack Trigger').item.json.user }}`  
   - Connect Slack Response main output to this node.

8. **Add Information Extractor Node**  
   - Type: LangChain Information Extractor  
   - Text Input: Slack Response message text `={{ $('Slack Response').item.json.message.text }}`  
   - Attributes to extract:  
     - Bitly URL Link (required)  
     - utm_campaign (required)  
     - utm_source (required)  
     - utm_medium (required)  
     - utm_term (required)  
     - utm_content (required)  
     - Target URL (required)  
     - User (required; set to Slack user real_name from Slack - Get User Name node)  
   - Connect Slack - Get User Name and Slack Response to this node.

9. **Add If Node**  
   - Type: If (Condition)  
   - Condition: Check if `Bitly URL Link` from Information Extractor output is not empty.  
   - True branch leads to Google Sheets node.  
   - False branch leads to Stop and Error node.

10. **Add Google Sheets Node**  
    - Type: Google Sheets (Append Row)  
    - Document ID: Your Google Sheets document ID.  
    - Sheet Name: Your target sheet name or gid (e.g., `gid=0` or `Sheet1`).  
    - Columns mapping:  
      - URL: `={{ $json.output['Target URL'] }}`  
      - Date: `={{ $today }}` (current date in ISO format)  
      - Notes: `"Executed by n8n workflow"`  
      - Owner: `={{ $json.output.User }}`  
      - Status: `"Done"`  
      - UTM ID: `={{ $json.output.utm_campaign }}`  
      - UTM Term: `={{ $json.output.utm_term }}`  
      - Bitly URL: `={{ $json.output['Bitly URL Link'] }}`  
      - UTM Medium: `={{ $json.output.utm_medium }}`  
      - UTM Source: `={{ $json.output.utm_source }}`  
      - UTM Content: `={{ $json.output.utm_content }}`  
      - UTM Campaign: `={{ $json.output.utm_campaign }}`  
      - Title on Bitly (Campaign): `={{ $json.output.utm_campaign }}`  
    - Connect If node's true branch to Google Sheets.

11. **Add Stop and Error Node**  
    - Type: Stop and Error  
    - Error Message: `"Workflow has stopped intentionally because Bitly URL Link does not exist or not generated"`  
    - Connect If node's false branch to this node.

12. **Set Credentials**  
    - Slack: OAuth2 credentials allowing app mention triggers and message posting.  
    - Bitly: API token with permission to shorten URLs.  
    - OpenAI: API key supporting GPT-4o-mini or equivalent.  
    - Google Sheets: OAuth2 credentials with access to target spreadsheet for appending rows.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| 🔗 Slack + Bitly UTM Generator — Powered by OpenAI. This no-code n8n workflow streamlines marketing link generation by combining Slack triggers, AI-driven UTM extraction, Bitly short-link creation, and Google Sheets logging. Perfect for growth marketers and teams needing fast, error-free campaign link creation. For detailed workflow build videos, visit https://www.youtube.com/@Automatewithmarc                                                                                                                                                                                                                                                                                                                                      | Workflow description and video resource link                        |
| UTM naming conventions enforced: lowercase, underscore separators, short forms normalized (e.g., IG → instagram). utm_medium restricted to known valid values (social, community, labs, video, cpc, email, referral, organic, banner, affiliate, tools).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | AI Agent prompt instructions                                       |
| Google Sheets schema requires matching columns for UTM parameters, Bitly URLs, user/owner, date, and status to ensure smooth logging and tracking.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Google Sheets node configuration                                   |
| Slack bot requires permissions for reading app mentions, posting threaded replies, and reading user profiles. Ensure proper OAuth scopes are assigned during Slack app setup.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Slack OAuth setup notes                                             |
| Bitly API token must have link creation privileges and domain configured as needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Bitly API setup                                                    |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow respecting all current content policies. It does not contain any illegal, offensive, or protected material. All data processed is legal and public.