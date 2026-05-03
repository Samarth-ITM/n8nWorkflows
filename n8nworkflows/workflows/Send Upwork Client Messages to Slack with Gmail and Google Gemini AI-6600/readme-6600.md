Send Upwork Client Messages to Slack with Gmail and Google Gemini AI

https://n8nworkflows.xyz/workflows/send-upwork-client-messages-to-slack-with-gmail-and-google-gemini-ai-6600


# Send Upwork Client Messages to Slack with Gmail and Google Gemini AI

### 1. Workflow Overview

This n8n workflow automates the process of monitoring unread Gmail messages from Upwork clients and sending notifications to Slack. It includes two parallel logical blocks that differ in complexity and AI integration:

- **Block 1: Basic Upwork Message Notification without AI**  
  This simpler branch uses a Gmail trigger to detect new messages from Upwork clients, extracts client name and message content via JavaScript code, and sends formatted notifications to Slack.

- **Block 2: Enhanced Notification with Google Gemini AI and Structured Information Extraction**  
  This branch also triggers on new Upwork messages but uses LangChain’s Information Extractor node to parse client name and message content. It integrates the Google Gemini Chat Model for AI processing before sending the structured notification to Slack. This block provides better accuracy and richer processing.

Both blocks poll Gmail every 3 minutes for new unread messages from "via Upwork" and send Slack messages to a specified user via OAuth2 authentication.

Logical blocks defined:

- **1.1 Gmail Message Reception (two triggers)**
- **1.2 Basic Client & Message Parsing (Code node)**
- **1.3 Structured Information Extraction (Information Extractor node)**
- **1.4 AI Processing (Google Gemini Chat Model)**
- **1.5 Slack Notification Sending**

---

### 2. Block-by-Block Analysis

#### 1.1 Gmail Message Reception

- **Overview:**  
  These two nodes independently poll Gmail for unread messages from senders containing "via Upwork". They act as entry points for the two notification paths.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Gmail Trigger X

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Gmail Trigger  
    - Role: Poll Gmail inbox every 3 minutes for unread emails from senders containing "via Upwork".  
    - Configuration:  
      - `simple` mode disabled for full message data.  
      - Filters: sender includes "via Upwork", unread messages only.  
      - Poll interval: every 3 minutes.  
    - Input: none (trigger)  
    - Output: raw Gmail message JSON objects.  
    - Failures: Gmail API auth errors, network timeouts, Gmail quota limits.  
    - Notes: Used in basic notification branch.

  - **Gmail Trigger X**  
    - Same configuration and role as Gmail Trigger but used in enhanced AI branch.  
    - Same potential failure modes.

#### 1.2 Basic Client & Message Parsing

- **Overview:**  
  Parses the raw Gmail message JSON to extract client name and message text using JavaScript code expressions.

- **Nodes Involved:**  
  - Code

- **Node Details:**

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Extracts client name and message snippet from the Gmail message JSON.  
    - Configuration:  
      - Extracts client name by splitting the `from.value[0].name` string at " via".  
      - Extracts message content by splitting the `text` field using year and "Reply" keywords.  
    - Key Expressions:  
      ```js
      client: $input.first().json.from.value[0].name.split(" via")[0]
      messgae: $input.first().json.text.split($today.year)[1].split("Reply")[0]
      ```  
      (Note: "messgae" is misspelled in code and used downstream as is.)  
    - Input: Gmail Trigger output  
    - Output: JSON with `client` and `messgae` fields.  
    - Failures: If message text parsing does not match expected format, may throw errors or produce incorrect values.  
    - Version: Uses typeVersion 2 (latest Code node features).

#### 1.3 Structured Information Extraction

- **Overview:**  
  Uses LangChain’s Information Extractor to parse the client name and message from the raw email text, enforcing required attributes with descriptions.

- **Nodes Involved:**  
  - Information Extractor

- **Node Details:**

  - **Information Extractor**  
    - Type: LangChain Information Extractor  
    - Role: Extracts structured attributes `message` and `client` from email text.  
    - Configuration:  
      - Input text: `{{$json.text}}` (raw email text).  
      - Attributes:  
        - `message`: required, described as "Client's Message which was send to me".  
        - `client`: required, described as "Client name who sent this message".  
      - No additional options set.  
    - Input: Gmail Trigger X output  
    - Output: JSON containing extracted `message` and `client` fields under `output`.  
    - Failures: Possible extraction failure if text format is unexpected or if AI service is inaccessible.  
    - Version: typeVersion 1.2.

#### 1.4 AI Processing (Google Gemini Chat Model)

- **Overview:**  
  Sends the raw message text to Google Gemini Chat Model for AI processing, feeding the output into the Information Extractor node.

- **Nodes Involved:**  
  - Google Gemini Chat Model

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Provides AI language model processing on the input text.  
    - Configuration: Default options, no custom prompt or parameters supplied.  
    - Input: Not connected directly to trigger; no explicit input in this workflow (no upstream node connected).  
    - Output: Connected to Information Extractor node for further processing.  
    - Failures: Google API errors, authentication issues, rate limits.  
    - Version: typeVersion 1.

    **Note:** The connection from Gemini Chat Model to Information Extractor implies AI-enhanced extraction, but no input is connected to Gemini in this workflow JSON, which might be a logical gap or incomplete connection.

#### 1.5 Slack Notification Sending

- **Overview:**  
  Sends formatted Slack messages to a specific user, incorporating extracted client name and message content.

- **Nodes Involved:**  
  - Send to Slack  
  - Send to Slack X

- **Node Details:**

  - **Send to Slack**  
    - Type: Slack node  
    - Role: Sends notification messages for the basic parsing branch.  
    - Configuration:  
      - Message text template:  
        ```
        Message From: {{ $json.client }}

        {{ $json.messgae }}
        ```  
      - Sends to user selected via OAuth2 authentication. User ID left empty, must be configured.  
    - Input: Output of Code node.  
    - Output: Slack send confirmation.  
    - Failures: Slack API auth errors, user ID missing, network issues.  
    - Version: 2.3.  
    - Note: Uses misspelled field `messgae` as is.

  - **Send to Slack X**  
    - Role: Sends notification messages for the AI-enhanced branch.  
    - Configuration:  
      - Message text template:  
        ```
        Message From: {{ $json.output.client }}

        {{ $json.output.message }}
        ```  
      - Sends to Slack user via OAuth2. User ID must be configured.  
    - Input: Output from Information Extractor node.  
    - Failures: Same as above.  
    - Version: 2.3.

---

### 3. Summary Table

| Node Name           | Node Type                       | Functional Role                          | Input Node(s)         | Output Node(s)        | Sticky Note                                                   |
|---------------------|--------------------------------|----------------------------------------|-----------------------|-----------------------|---------------------------------------------------------------|
| Gmail Trigger       | Gmail Trigger                  | Poll Gmail for unread Upwork messages  | -                     | Code                  | **Upwork Client Messages Notification Without the requirement of an AI Model** |
| Code                | Code (JavaScript)              | Parse client name & message from email | Gmail Trigger         | Send to Slack          |                                                               |
| Send to Slack       | Slack                         | Send basic notification to Slack user  | Code                  | -                     |                                                               |
| Gmail Trigger X     | Gmail Trigger                  | Poll Gmail for unread Upwork messages  | -                     | Information Extractor  | **MUCH BETTER - Upwork Client Messages Notifier with Google Gemini** |
| Information Extractor| LangChain Information Extractor| Extract structured client & message    | Gmail Trigger X       | Send to Slack X        |                                                               |
| Google Gemini Chat Model | LangChain Google Gemini Chat Model | AI processing of message text          | - (no input connected) | Information Extractor  |                                                               |
| Send to Slack X     | Slack                         | Send AI-enhanced notification to Slack | Information Extractor | -                     |                                                               |
| Sticky Note         | Sticky Note                   | Visual note for basic branch            | -                     | -                     | **Upwork Client Messages Notification Without the requirement of an AI Model** |
| Sticky Note1        | Sticky Note                   | Visual note for AI-enhanced branch      | -                     | -                     | **MUCH BETTER - Upwork Client Messages Notifier with Google Gemini** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node (Basic branch):**  
   - Type: Gmail Trigger  
   - Set `simple` mode to false to get full message details.  
   - Add filter: Sender contains "via Upwork".  
   - Filter unread messages only.  
   - Set polling interval to every 3 minutes.  
   - Credentials: Configure Gmail OAuth2 with required scopes.

2. **Create Code node:**  
   - Connect input from Gmail Trigger.  
   - Paste JavaScript code:  
     ```js
     return { 
       client: $input.first().json.from.value[0].name.split(" via")[0], 
       messgae: $input.first().json.text.split($today.year)[1].split("Reply")[0]
     }
     ```  
   - Note the misspelling of `messgae` must be consistent downstream.

3. **Create Send to Slack node (Basic branch):**  
   - Connect input from Code node.  
   - Set Slack node to OAuth2 authentication.  
   - In message text, use:  
     ```
     Message From: {{ $json.client }}

     {{ $json.messgae }}
     ```  
   - Select or input Slack user ID to send notification.

4. **Create Gmail Trigger X node (AI-enhanced branch):**  
   - Duplicate Gmail Trigger configuration (same filters and polling).  
   - Connect to Information Extractor node.

5. **Create Google Gemini Chat Model node:**  
   - Set type to LangChain Google Gemini Chat Model.  
   - Default options (no prompt changes).  
   - No input connection needed (per JSON), but consider connecting Gmail Trigger X output if desired.  
   - Connect output to Information Extractor node.

6. **Create Information Extractor node:**  
   - Connect input from Gmail Trigger X (and Google Gemini Chat Model output if you link AI).  
   - Configure attributes:  
     - `message`: required, description "Client's Message which was send to me".  
     - `client`: required, description "Client name who sent this message".  
   - Input text: set to `{{$json.text}}`.

7. **Create Send to Slack X node:**  
   - Connect input from Information Extractor node.  
   - Slack OAuth2 authentication configured.  
   - Message text template:  
     ```
     Message From: {{ $json.output.client }}

     {{ $json.output.message }}
     ```  
   - Set Slack user to notify.

8. **Add Sticky Notes for clarity (optional):**  
   - Add one describing the basic notification branch:  
     "**Upwork Client Messages Notification Without the requirement of an AI Model**"  
   - Add one describing the AI-enhanced branch:  
     "**MUCH BETTER - Upwork Client Messages Notifier with Google Gemini**"

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow splits into two parallel branches: a basic notification and an AI-enhanced notification pipeline. | Highlights options for users with or without access to AI models like Google Gemini.             |
| Google Gemini Chat Model integration is experimental and requires Google API access with proper credentials.   | Google Cloud documentation: https://cloud.google.com/vertex-ai/docs/chat/introduction            |
| Slack user IDs must be configured manually in Slack nodes with OAuth2 credentials that have chat:write scope.  | Slack OAuth2 setup guide: https://api.slack.com/authentication/oauth-v2                          |
| The Code node contains a misspelling in the output field `messgae`—keep consistent or correct both references. | Potential source of bugs if corrected only partially.                                            |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.