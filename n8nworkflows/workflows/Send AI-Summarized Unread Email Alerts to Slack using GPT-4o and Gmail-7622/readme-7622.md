Send AI-Summarized Unread Email Alerts to Slack using GPT-4o and Gmail

https://n8nworkflows.xyz/workflows/send-ai-summarized-unread-email-alerts-to-slack-using-gpt-4o-and-gmail-7622


# Send AI-Summarized Unread Email Alerts to Slack using GPT-4o and Gmail

### 1. Workflow Overview

This workflow automates the process of monitoring unread Gmail messages and sending AI-generated concise email summaries as Slack notifications. It targets users who want timely, summarized alerts for unread emails directly in Slack, improving email management efficiency without opening the inbox.

Logical blocks in the workflow:

- **1.1 Input Reception:** Detects unread emails in Gmail using a polling trigger.
- **1.2 Email Retrieval:** Fetches full email content based on the detected unread email.
- **1.3 AI Processing:** Uses an AI agent powered by OpenAI’s GPT-4o-mini to summarize the email content into a structured JSON output with sender and summary.
- **1.4 Output Parsing:** Validates and structures the AI-generated JSON summary.
- **1.5 Notification Dispatch:** Sends the formatted summary to a selected Slack user via Slack’s API.
- **1.6 User Guidance:** Sticky notes provide configuration and usage tips.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Watches Gmail inbox for unread emails and triggers the workflow once per minute if any are found.
- **Nodes Involved:**  
  - Trigger on Unread Email

- **Node Details:**

  - **Trigger on Unread Email**  
    - Type: Gmail Trigger node (poll trigger)  
    - Configuration: Filters unread emails only (`readStatus: unread`), polling every minute.  
    - Inputs: None (trigger node)  
    - Outputs: Emits IDs of unread emails detected.  
    - Version requirements: Gmail API access with appropriate OAuth2 credentials.  
    - Edge cases:  
      - Gmail API quota or authentication failures.  
      - No emails found results in no trigger.  
      - Potential delays in Gmail’s unread status sync.  
    - Sticky notes attached:  
      - "🔁 Adjust Frequency" — advises on modifying poll interval in this node.

#### 1.2 Email Retrieval

- **Overview:** Retrieves the full content of the detected unread email by its message ID.
- **Nodes Involved:**  
  - Get Email

- **Node Details:**

  - **Get Email**  
    - Type: Gmail node (operation: get)  
    - Configuration: Fetches email by `messageId` from the trigger node; returns full message details (`simple: false`).  
    - Inputs: From “Trigger on Unread Email” node (message ID).  
    - Outputs: Full email JSON including text content.  
    - Version requirements: Gmail node version 2.1 or higher recommended for full message retrieval.  
    - Edge cases:  
      - Message ID invalid or deleted.  
      - API rate limits or auth errors.  
      - Emails with complex MIME structures may affect content extraction.  
    - Sticky notes attached:  
      - "🛠️ Connect Credentials First" — Gmail credentials required.

#### 1.3 AI Processing

- **Overview:** Summarizes the email content into a concise JSON format containing sender and summary fields using an AI agent powered by GPT-4o-mini.
- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Structured Output Parser

- **Node Details:**

  - **AI Agent**  
    - Type: Langchain Agent node  
    - Configuration:  
      - Prompt instructs summarization into JSON with two fields: sender and summary (max 250 characters).  
      - The email text is inserted dynamically (`{{ $json["text"] }}`) from the “Get Email” output.  
      - Has output parser enabled to enforce JSON output.  
    - Inputs: Email content JSON from “Get Email” node.  
    - Outputs: AI-generated text output (raw JSON string).  
    - Edge cases:  
      - AI output not conforming to JSON schema (malformed JSON).  
      - Rate limits or API errors from OpenAI.  
      - Timeout or slow response from AI.  
    - Sticky notes attached:  
      - "🧪 Tweak the AI Summary" — instructions on modifying prompt or summary length.

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat model node  
    - Configuration: Uses model `gpt-4o-mini` for efficient summarization.  
    - Inputs: Connected internally as language model for AI Agent node.  
    - Outputs: AI textual response.  
    - Edge cases:  
      - Model-specific token limits or API errors.  

  - **Structured Output Parser**  
    - Type: Langchain Output Parser (structured)  
    - Configuration: Manual JSON schema validation requiring exactly two fields: `sender` (string) and `summary` (string, max 250 characters).  
    - Inputs: AI Agent output (raw JSON string).  
    - Outputs: Parsed and validated JSON object to be used downstream.  
    - Edge cases:  
      - AI output failing to match schema leads to parsing errors.  
      - Additional properties or missing required fields cause failures.

#### 1.4 Notification Dispatch

- **Overview:** Sends the AI-generated summary as a Slack message to a selected user.
- **Nodes Involved:**  
  - Send Notification to Slack

- **Node Details:**

  - **Send Notification to Slack**  
    - Type: Slack node  
    - Configuration:  
      - Sends message text combining sender and summary (`={{ $json.output.sender }}: {{ $json.output.summary }}`).  
      - Target user selected manually in node config (`select: user`).  
      - OAuth2 authentication configured for Slack API access.  
      - No link to workflow included in message options.  
    - Inputs: Parsed JSON summary from “AI Agent” node main output.  
    - Outputs: Slack API response (message sent confirmation).  
    - Edge cases:  
      - Slack API auth token expired or revoked.  
      - User ID invalid or user not found.  
      - Message format issues causing send failure.  
    - Sticky notes attached:  
      - "📣 Slack Notification Tips" — tips on customizing Slack message format and routing.

#### 1.5 User Guidance

- **Overview:** Provides visual notes within the editor to guide users on credential setup, polling adjustments, AI prompt customization, and Slack notification formatting.
- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3

- **Node Details:**

  - Sticky notes do not process data but contain important setup information:  
    - Connect Gmail, Slack, and OpenAI credentials before running.  
    - Adjust Gmail trigger polling frequency as needed.  
    - Customize AI summarization prompt to modify summary style or length.  
    - Customize Slack message format, include emojis, prefixes, or routing logic.

---

### 3. Summary Table

| Node Name               | Node Type                             | Functional Role                   | Input Node(s)          | Output Node(s)            | Sticky Note                                                          |
|-------------------------|-------------------------------------|---------------------------------|-----------------------|---------------------------|----------------------------------------------------------------------|
| Trigger on Unread Email | Gmail Trigger                       | Detect unread emails             | None                  | Get Email                 | 🔁 Adjust Frequency: Modify polling interval here                   |
| Get Email               | Gmail                              | Retrieve full email content      | Trigger on Unread Email | AI Agent                  | 🛠️ Connect Credentials First: Gmail credentials required            |
| AI Agent                | Langchain Agent                    | Generate AI summary JSON         | Get Email              | Send Notification to Slack (main), Structured Output Parser (ai_outputParser), OpenAI Chat Model (ai_languageModel) | 🧪 Tweak the AI Summary: Adjust prompt or summary length            |
| OpenAI Chat Model        | Langchain OpenAI Chat Model        | Provide GPT-4o-mini model output | AI Agent (ai_languageModel) | AI Agent (ai_languageModel) |                                                                      |
| Structured Output Parser | Langchain Output Parser Structured | Validate and parse AI output     | AI Agent (ai_outputParser) | AI Agent (ai_outputParser) |                                                                      |
| Send Notification to Slack | Slack                            | Send summarized message to Slack | AI Agent (main)        | None                      | 📣 Slack Notification Tips: Customize message formatting and routing |
| Sticky Note             | Sticky Note                       | User instruction                 | None                  | None                      | 🛠️ Connect Credentials First: Gmail, Slack, and OpenAI must be connected |
| Sticky Note1            | Sticky Note                       | User instruction                 | None                  | None                      | 🔁 Adjust Frequency: Modify Gmail polling frequency                  |
| Sticky Note2            | Sticky Note                       | User instruction                 | None                  | None                      | 🧪 Tweak the AI Summary: Modify prompt or summary length            |
| Sticky Note3            | Sticky Note                       | User instruction                 | None                  | None                      | 📣 Slack Notification Tips: Format and customize Slack messages     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: Gmail Trigger**  
   - Set node name: "Trigger on Unread Email"  
   - Configure credentials: Connect Gmail OAuth2 credentials.  
   - Set filter: `readStatus` = `unread`  
   - Set polling frequency: every 1 minute (adjustable).  
   - Save.

2. **Create Gmail Node to Get Email**  
   - Set node name: "Get Email"  
   - Connect input to "Trigger on Unread Email" node output.  
   - Set operation: `get`  
   - Set parameter: `messageId` = `={{ $json.id }}` (from trigger data)  
   - Disable simple mode to get full email content.  
   - Ensure Gmail credentials connected.  
   - Save.

3. **Create Langchain AI Agent Node**  
   - Set node name: "AI Agent"  
   - Connect input to "Get Email" node output.  
   - Configure prompt (define type) with the following text (replace if desired):  
     ```
     You are an assistant that summarises emails.  
     Summarise the following email into a concise message (max 250 characters).  
     Your output must be valid JSON with exactly two fields:  
     - "sender": the sender’s name  
     - "summary": the 250-character summary  
     Here is the required output format:  
     {  
       "sender": "Example Sender",  
       "summary": "Example summary of the email, no more than 250 characters."  
     }  
     Email content:  
     {{ $json["text"] }}
     ```  
   - Enable output parser.  
   - Save.

4. **Create Langchain OpenAI Chat Model Node**  
   - Set node name: "OpenAI Chat Model"  
   - Configure credentials: connect OpenAI API key.  
   - Set model: `gpt-4o-mini`  
   - No additional options required.  
   - Connect this node to the “AI Agent” node’s `ai_languageModel` input.  
   - Save.

5. **Create Langchain Structured Output Parser Node**  
   - Set node name: "Structured Output Parser"  
   - Define JSON schema (manual):  
     ```json
     {
       "type": "object",
       "properties": {
         "sender": {
           "type": "string",
           "description": "The sender’s name from the email"
         },
         "summary": {
           "type": "string",
           "maxLength": 250,
           "description": "A concise summary of the email content"
         }
       },
       "required": ["sender", "summary"],
       "additionalProperties": false
     }
     ```  
   - Connect to “AI Agent” node’s `ai_outputParser` input.  
   - Save.

6. **Create Slack Node to Send Notification**  
   - Set node name: "Send Notification to Slack"  
   - Connect input from “AI Agent” node main output.  
   - Configure credentials: connect Slack OAuth2 credentials with permission to send messages.  
   - Set message text: `={{ $json.output.sender }}: {{ $json.output.summary }}`  
   - Select recipient user manually from Slack users list in node configuration.  
   - Disable including workflow link in message.  
   - Save.

7. **Connect Nodes**  
   - Trigger on Unread Email → Get Email  
   - Get Email → AI Agent  
   - AI Agent → Send Notification to Slack (main output)  
   - AI Agent (ai_languageModel) → OpenAI Chat Model  
   - AI Agent (ai_outputParser) → Structured Output Parser

8. **Optional: Add Sticky Notes**  
   - Add sticky notes for credential setup instructions, polling frequency tips, AI prompt customization, and Slack message formatting as per your preference for maintainability.

9. **Activate Workflow**  
   - Ensure all credentials are connected and tested.  
   - Activate the workflow to begin polling and sending Slack notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                      |
|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Connect Gmail, Slack, and OpenAI credentials before activating the workflow to ensure seamless operation.      | Credential setup instructions in sticky notes       |
| Adjust the Gmail Trigger polling interval to balance between timely alerts and API quota consumption.          | Sticky note "Adjust Frequency"                        |
| Customize the AI Agent prompt to tailor summaries: add urgency, action items, or remove character limits.      | Sticky note "Tweak the AI Summary"                    |
| Customize Slack message formatting: add emojis, prefixes, or route notifications differently per sender.       | Sticky note "Slack Notification Tips"                 |
| The workflow uses GPT-4o-mini, a cost-effective OpenAI model variant, but can be replaced with other models.   | Node configuration details                            |
| Ensure Slack OAuth2 credentials have permissions to send messages to the targeted users or channels.           | Slack node credential and permission requirements    |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.